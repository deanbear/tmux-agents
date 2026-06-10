---
name: setup-agents
description: >-
  Use this skill to start, discover, or clean up tmux-agents sub-agents running
  in tmux panes. Handles agent startup with locked pane titles, readiness
  verification, and defines the communication protocol used by all upper-layer
  skills. Trigger phrases: "启动 agent", "start agents", "setup agents",
  "初始化 agent", "/tmux-agents:setup-agents".
user-invocable: true
---

# Setup Agents

启动多 Agent 协作环境：根据配置文件中定义的所有角色，在 tmux 中为每个角色开启独立的 session，锁定 pane title，验证就绪状态。

**主控运行环境**：本 skill 支持 Claude Code 或 Codex 作为主控。子 agent 的 runtime 可以是 `claude`、`codewhale` 或 `codex`，由 `agents.json` 的 `runtime` 字段控制。

**插件根目录解析**：
- Claude Code 主控：使用 `${CLAUDE_SKILL_DIR}/../..`。
- Codex 主控：使用当前 `SKILL.md` 文件路径上两级的绝对路径。

后文用 `{plugin_root}` 表示解析后的插件根目录，例如内置 role 文件为 `{plugin_root}/roles/{type}.md`。

---

## 步骤一：前置检查

1. 确认 `tmux` 已安装：

```bash
which tmux
```

如果未安装，告知用户：`brew install tmux`（macOS）或 `apt install tmux`（Linux），退出。

2. 确认 `tmux-cli` 已安装：

```bash
which tmux-cli
```

如果未安装，告知用户：`uv tool install claude-code-tools`，退出。当前只依赖该包里的 `tmux-cli`，包名保留 Claude Code 是历史原因。

3. 确认在 tmux 环境中：

```bash
tmux-cli status
```

如果报错或显示不在 tmux 中，告知用户需要先启动 tmux session（`tmux new -s work`），退出。

4. 定位配置文件，按以下优先级用 Bash 检查文件是否存在，找到第一个后用 Read 工具读取：

```bash
test -f "$TMUX_AGENTS_CONFIG" && echo "$TMUX_AGENTS_CONFIG"
```

如果 `$TMUX_AGENTS_CONFIG` 未设置或文件不存在：
- Codex 主控读取 `~/.codex/tmux-agents/agents.json`
- Claude Code 主控读取 `~/.claude/tmux-agents/agents.json`

如果宿主默认配置也不存在，不要直接启动内置 example。告知用户 `{plugin_root}/config/agents.example.json` 是模板，需要先复制到当前宿主的默认配置路径，并删除不使用的 runtime 示例后再重新运行：

```bash
# Codex 主控
mkdir -p ~/.codex/tmux-agents
cp {plugin_root}/config/agents.example.json ~/.codex/tmux-agents/agents.json

# Claude Code 主控
mkdir -p ~/.claude/tmux-agents
cp {plugin_root}/config/agents.example.json ~/.claude/tmux-agents/agents.json
```

停止流程，等待用户编辑配置后重试。这样避免 clean install 时把模板里的 claude、codewhale、codex 多套示例角色全部启动。

读取到 JSON 内容后，在对话中直接解析配置。

5. 记录主控 pane id 和工作目录：

```bash
tmux-cli status
pwd
```

从 `tmux-cli status` 输出中找到标记为 `*`（当前活动）的 pane，记录其 pane id 作为 **master_pane_id**。将 `pwd` 输出记录为 **master_cwd**，后续 Codex runtime 用它作为 `codex --cd` 的工作目录。拼接命令时对 `master_cwd` 做 shell 参数安全引用，并将引用后的值记为 **master_cwd_quoted**。

后续步骤三启动子 agent 时，`tmux-cli launch` 会把焦点切走，需要切回主控 pane 才能让后续命令正确运转。

---

## 步骤二：检查已有 Agent

运行 `tmux-cli status`，解析输出，查找 pane title 与配置中各角色的 `pane_title` 匹配的 pane。

对每个角色：
- 如果已有匹配的 pane：报告 "✓ {role} 已在运行 (pane {id})"，跳过该角色的启动
- 如果没有：继续步骤三

---

## 步骤三：逐个启动 Agent

对每个需要启动的角色，读取其 `runtime` 字段（默认 `claude`），按对应分支执行启动序列。

### 3.0 启用 pane border 显示角色名

在启动第一个 agent 之前，执行：

```bash
tmux set-option -w pane-border-format "#{pane_title}"
tmux set-option -w pane-border-status top
```

### 3.1 启动 shell pane（所有 runtime 共用）

```bash
tmux-cli launch "zsh"
```

记录返回的 pane id。

**注意**：`tmux-cli launch` 会把焦点切换到新 pane。立即切回主控 pane，避免后续 send 误发：

```bash
tmux select-pane -t {master_pane_id}
```

`master_pane_id` 是步骤一记录的总控 pane id。如果不切回，后面所有 send 命令仍能用 `--pane=` 显式指定目标，不会发错，但 `tmux-cli status` 等命令的 "current location" 报告会指向子 agent pane，可能干扰判断。

### 3.2 锁定 pane title（所有 runtime 共用）

**背景**：tmux 默认 `allow-rename on` + `automatic-rename on`，子进程通过 OSC 转义序列（`\033]0;...\007` 等）或 tmux 根据 foreground 命令名都会改写 pane title。zsh 的 precmd hook（macOS Terminal 默认会写 `hostname` 当 title）、codewhale-tui 自己写的 window title，都会把 `select-pane -T` 设的 title 盖掉。`claude` runtime 因为有 `CLAUDE_CODE_DISABLE_AUTO_TITLE=1` + `--name` 两层保护，问题不显；但 `codewhale` 等其他 runtime 必须额外锁 title。

为保持所有 runtime 行为一致、且不污染同 window 内别的 pane，采用 **pane-level 锁**：

```bash
# 先关该 pane 的 rename（pane-level，-p）
tmux set-option -p -t {pane_id} allow-rename off
tmux set-option -p -t {pane_id} automatic-rename off

# 再设 title
tmux-cli send "tmux select-pane -T '{pane_title}'" --pane={pane_id}
```

`-p` 标志表示 pane-level option，只锁这一个 pane，同 window 里其他 pane（包括主控 pane）不受影响。`set-option` 由总控直接执行（不通过 `tmux-cli send`），避免把命令文本注入子 pane 的 shell。

等待完成：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=2 --timeout=10
```

**验证 title 是否生效**（推荐启动 agent 前确认一次）：

```bash
tmux display-message -p -t {pane_id} '#{pane_title}'
```

输出应为 `{pane_title}`。如果不是，说明 pane-level rename 锁未生效（tmux 版本过旧不支持 `-p` 的某些 option？），降级方案：临时用 `-w` window-level 锁，但要记住这会影响同 window 的其他 pane title。

### 3.3 按 runtime 启动 agent

**配置值约束**：`pane_title` 和 `disallowed_tools` 的值不得包含单引号（`'`），因为它们会被单引号包裹后嵌入 tmux send 的双引号命令中。如果配置值包含单引号，命令会断裂导致启动失败。

角色定义文件路径：如果角色配置中有 `role_file` 字段，使用该路径；否则按 `type` 匹配插件内置的 role 文件（`{plugin_root}/roles/{type}.md`）。

#### 3.3.A runtime = claude

**基础命令**（使用 Anthropic 原生 API）：

```bash
tmux-cli send "CLAUDE_CODE_DISABLE_AUTO_TITLE=1 claude --model {model} --permission-mode {permission_mode} --append-system-prompt-file {plugin_root}/roles/{type}.md --disallowed-tools '{disallowed_tools}' --name '{pane_title}'" --pane={pane_id}
```

如果 role 文件不存在则跳过 `--append-system-prompt-file` 参数。

**自定义 API 命令**（角色配置中包含 `api_base` 和 `api_key`）：

将 API 地址和 key 内联到 claude 启动命令前，只对该进程生效，不污染 pane 的 shell 环境：

```bash
tmux-cli send "CLAUDE_CODE_DISABLE_AUTO_TITLE=1 ANTHROPIC_BASE_URL={api_base} ANTHROPIC_AUTH_TOKEN={api_key} claude --model {model} --permission-mode {permission_mode} --append-system-prompt-file {plugin_root}/roles/{type}.md --disallowed-tools '{disallowed_tools}' --name '{pane_title}'" --pane={pane_id}
```

启动后等待就绪：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=60
```

验证 agent 响应：

```bash
tmux-cli send "回复 AGENT_READY 确认你已就绪" --pane={pane_id}
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=30
tmux-cli capture --pane={pane_id}
```

检查 capture 输出中是否包含 `AGENT_READY`。

#### 3.3.B runtime = codewhale
**验证范围**：当前 codewhale runtime 仅在 deepseek provider + litellm 代理 + `deepseek/<name>` 路由前缀这条路径下验证过。其他 provider、原生 API endpoint（如 `api.deepseek.com`、`api.openai.com`）未验证。读者使用前请知悉。

**字段语义差异（与 claude runtime 对比）**：`runtime: codewhale` 时 `api_base` 指 litellm 兼容代理 URL、`api_key` 指 litellm 的 key，**不是** LLM provider 的原生凭证。同名字段在 `runtime: claude` 下指向 Anthropic 兼容 API 的 endpoint 和 key。这是 schema 设计上的认知陷阱，长期应该 split 字段名（见末尾"已知设计缺陷"）。

codewhale 没有 `--append-system-prompt-file` 等价参数，role prompt 通过启动后的首条消息让子 agent Read role 文件加载。

**调用真正的 TUI 二进制 `codewhale-tui`，不是包装器 `codewhale`**：`codewhale` 是 npm wrapper，它的 `--yolo` 只把审批策略改为 auto；只有 `codewhale-tui --yolo` 才会**同时启用 workspace trust mode**（状态栏显示 `yolo · <model>` 而非 `agent · <model>`），让 `read_file`/`write_file` 接受 workspace 外的路径——这是子 agent 能读 plugin 内置 role 文件、写 `/tmp/tmux-agents/` 通信目录的前提。

**字段语义须知**：切到 `runtime: codewhale` 后，`permission_mode` 和 `disallowed_tools` 字段被忽略（codewhale 没有对应机制）。如果你从 claude 角色 copy 配置过来，记得删掉 `disallowed_tools`，否则会让你误以为有工具层兜底。启动时主控会 warn 提示（见步骤三）。
- `--skip-onboarding`：必须加，否则 codewhale TUI 卡在新手引导
- `codewhale-tui` **不接受 `--model`/`--base-url`/`--api-key` flag**（只有包装器 `codewhale` 接受）。改用环境变量 `DEEPSEEK_MODEL`/`DEEPSEEK_BASE_URL`/`DEEPSEEK_API_KEY` 内联到启动命令前
- 不传 `--workspace`：codewhale-tui 默认用 cwd 作为 workspace

**文件系统访问模型（先了解再读启动命令）**：codewhale-tui `--yolo` 启动后 trust mode 默认 enabled，`read_file`/`write_file` 可直接读写任意路径（workspace 内、`/tmp/`、`~/.claude/plugins/cache/` 等），与 `~/.codewhale/workspace-trust.json` 的内容无关。这就是子 agent 能直接读 plugin role 文件、写 `/tmp/tmux-agents/` 通信目录的原因。trust list（`/trust add` 写入的）只在**非 yolo** 模式下有约束意义。

但要注意：trust mode 启用**不是新增的攻击面**——任何允许 shell 调用的子 agent 都已有 UID 级别的全机访问能力，trust mode 开关只影响"是否要绕一道 shell 才能读 workspace 外路径"的攻击成本（5 秒量级），不是 0/1 安全边界。详见末尾"通用安全说明"。

**配置 lint（启动前）**：codewhale runtime 不支持 `disallowed_tools` 字段，配置里有则 warn 后继续。在发送启动命令之前执行：

```bash
# 伪代码：从已解析的角色配置里检查
if [ -n "${role_disallowed_tools}" ]; then
  echo "WARNING: role '{role_name}' has 'disallowed_tools' but runtime=codewhale ignores it. Remove the field to avoid confusion." >&2
fi
```

**基础命令**：

```bash
tmux-cli send "DEEPSEEK_MODEL={model} codewhale-tui --skip-onboarding --yolo" --pane={pane_id}
```

**自定义 API 命令**（角色配置中包含 `api_base` 和 `api_key`）：

```bash
tmux-cli send "DEEPSEEK_BASE_URL={api_base} DEEPSEEK_API_KEY={api_key} DEEPSEEK_MODEL={model} codewhale-tui --skip-onboarding --yolo" --pane={pane_id}
```

启动后等待 TUI 就绪（codewhale TUI 启动需要 5-10s）：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=60
```

注入 role prompt（合并 readiness 验证为一步）：

```bash
tmux-cli send "你现在被纳入 tmux-agents 多 Agent 协作体系，担任 {type} 角色。请用 Read 工具读取 {plugin_root}/roles/{type}.md 加载你的角色定义。读取并理解后请严格遵守 role 文件里的所有约束，并在单独一行输出 AGENT_READY 表示你已就绪。" --pane={pane_id}
```

注意：此消息**不应**硬编码任何特定角色的行为约束（如"不修改源代码"），那些约束由各 role 文件自行声明。

等待并验证：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=120
tmux-cli capture --pane={pane_id}
```

检查 capture 输出中是否包含 `AGENT_READY`。如果不包含，报告该 agent 启动异常，继续启动下一个。

**首次工具审批**：

> ⚠️ 当前 codewhale-tui --yolo 是否仍弹首次审批待在干净机器或新 thread 上重新验证。现观察可能被既有 `~/.codewhale/sessions/*.json` 的 `auto_approve` 记录污染。

codewhale 子 agent 第一次调用未分类工具（如 `task_shell_start`）时会弹出审批对话框。codewhale 把"按 `2`（同类自动批准）"的决策**持久化到 thread record**（`~/.codewhale/sessions/<thread>.json` 的 `auto_approve` 字段），后续同 workspace 启动的 codewhale 进程从同一 thread 继续时自动继承此决策，无需再批。

实践上：一台机器上一旦人工批过一次 `2`，之后所有 tmux-agents 的 codewhale 子 agent 通常都免审。**真正卡审批的场景**是第一次在这台机器上用 codewhale runtime——如遇审批弹窗，总控代发按键 `2`（注意见末尾"通信协议 / tmux-cli send 使用须知"对纯数字按键的引号包裹要求）。

**常见启动失败**：

> 以下第一条有现场观察证据，后两条为预防性提示，实际触发条件待用户反馈。

- 状态栏显示 `agent` 而非 `yolo`：用错了 `codewhale` 包装器而不是 `codewhale-tui`。`yolo` 状态栏标识同时要求 approval=auto AND trust mode=on 才显示；wrapper 的 `--yolo` 只完成 approval 一半，所以显示 `agent`。
- `Path escapes workspace` 错误：启动前用 `which codewhale-tui` 确认指向 npm 真二进制（`/opt/homebrew/lib/node_modules/codewhale/bin/codewhale-tui.js` 或类似），而不是 PATH 上的 alias 或同名 wrapper 脚本。trust mode 启用后此错误不会再出现
- `error: unexpected argument '--model'`：用 `codewhale-tui` 时必须用环境变量传 API/model，不能用 flag

#### 3.3.C runtime = codex

Codex runtime 调用当前机器已安装、已登录、已配置的 Codex CLI。不要在 `agents.json` 中为 Codex runtime 配置新的 provider endpoint 或 API key。

**支持字段**：
- `runtime`: 必须为 `codex`
- `type`
- `pane_title`
- `model`（可选；只在当前 role 显式配置时传给 Codex；不填则不传 `--model`，使用当前 Codex 默认模型）
- `profile`（可选；对应 `codex --profile`）
- `sandbox`（可选；默认 `workspace-write`）
- `approval_policy`（可选；默认 `never`）
- `add_dirs`（可选；数组；默认包含 `/tmp/tmux-agents`）

**不支持字段（fail fast）**：

如果角色配置中出现 `api_base`、`api_key`、`permission_mode` 或 `disallowed_tools`，停止启动该 role，并提示用户删除这些字段。原因：
- Codex runtime 只使用当前 Codex 配置，不接受 tmux-agents 传入新的 provider key
- Codex CLI 没有与 Claude Code `--disallowed-tools` 等价的工具黑名单启动参数
- `permission_mode` 是 Claude Code 字段，Codex 使用 `sandbox` 和 `approval_policy`

**安全语义**：

Codex 子 agent 会继承用户当前 Codex 配置、AGENTS 指令、skills/plugins/rules。role prompt 是本协作协议的一部分，但不是 Codex 子 agent 的唯一输入。`approval_policy` 默认 `never`，这样子 agent 不会在子 pane 内弹审批；越过 sandbox 的动作会直接失败。

注意：Codex runtime 不继承 `defaults.model`。如果 role 自己没有配置 `model`，启动命令不要拼接 `--model`。

**基础命令**：

```bash
tmux-cli send "codex --cd {master_cwd_quoted} --sandbox {sandbox} --ask-for-approval {approval_policy} --add-dir /tmp/tmux-agents --no-alt-screen" --pane={pane_id}
```

如果配置了 `model`：

```bash
tmux-cli send "codex --model {model} --cd {master_cwd_quoted} --sandbox {sandbox} --ask-for-approval {approval_policy} --add-dir /tmp/tmux-agents --no-alt-screen" --pane={pane_id}
```

如果只配置了 `profile`：

```bash
tmux-cli send "codex --profile {profile} --cd {master_cwd_quoted} --sandbox {sandbox} --ask-for-approval {approval_policy} --add-dir /tmp/tmux-agents --no-alt-screen" --pane={pane_id}
```

如果同时配置了 `profile` 和 `model`：

```bash
tmux-cli send "codex --profile {profile} --model {model} --cd {master_cwd_quoted} --sandbox {sandbox} --ask-for-approval {approval_policy} --add-dir /tmp/tmux-agents --no-alt-screen" --pane={pane_id}
```

如果配置了额外 `add_dirs`，对每个目录追加一个 `--add-dir {dir}`。不要追加插件根目录；本机验证过 Codex workspace 权限下可读插件 cache 路径，role 文件直接使用 `{plugin_root}/roles/{type}.md`。

启动后等待 TUI 就绪：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=60
```

注入 role prompt（合并 readiness 验证为一步）：

```bash
tmux-cli send "你现在被纳入 tmux-agents 多 Agent 协作体系，担任 {type} 角色。请用 Read 工具读取 {plugin_root}/roles/{type}.md 加载你的角色定义。读取并理解后请严格遵守 role 文件里的所有约束，并在单独一行输出 AGENT_READY 表示你已就绪。" --pane={pane_id}
```

等待并验证：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=120
tmux-cli capture --pane={pane_id}
```

检查 capture 输出中是否包含 `AGENT_READY`。如果不包含，报告该 agent 启动异常，继续启动下一个。

---

## 步骤四：报告启动结果

展示所有角色的状态：

```
Tmux-Agents 环境就绪

  {role_name}:  pane {id} | type: {type} | model: {model}
  ...（列出配置中所有角色）

可用的工作流：
  /tmux-agents:plan-review   — 独立 plan 审查
  /tmux-agents:code-review   — 独立代码审查
  /tmux-agents:orchestrate           — 全流程编排
  /tmux-agents:cleanup       — 关闭 agent 并清理
```

---

## 通信协议

以下协议供上层 skill（plan-review、code-review、orchestrate）在与子 agent 通信时遵循。

### 发送任务

```
1. DISCOVER:  tmux-cli status → 按 pane title 找到目标 pane id
              每次通信前必须重新确认，不要缓存旧的 pane id
2. WAIT:      tmux-cli wait_idle --pane={id} --idle-time=3 --timeout=10
              确认 agent 空闲
3. CONTEXT:   将大上下文（plan、diff、代码文件内容）写入
              /tmp/tmux-agents/{session_id}/ 下的文件
4. SEND:      tmux-cli send "{prompt}" --pane={id}
              prompt 中引用上下文文件路径，指示 agent 用 Read 工具读取
              prompt 必须是单行文本（不含裸换行），避免 tmux 将换行解释为多次回车
5. WAIT:      tmux-cli wait_idle --pane={id} --idle-time={defaults.idle_time} --timeout={defaults.idle_timeout}
              其中 idle_time 和 idle_timeout 取自配置文件的 defaults 段
              如果超时：先 capture 检查部分输出；如果 agent 无响应，
              tmux-cli interrupt 后重试一次；仍无响应则报告用户
6. CAPTURE:   tmux-cli capture --pane={id}
              检查完成标记（TASK_COMPLETE / REVIEW_COMPLETE / REVISION_COMPLETE）
```

### tmux-cli send 使用须知

`tmux-cli send` 参数由 fire 库解析，下列场景需要注意：

- **向 TUI pane 发纯数字按键**（任何 runtime 都适用）：直接 `tmux-cli send 2 --pane=...` 会被 fire 当 int 处理，抛 `TypeError: expected str, bytes or os.PathLike object, not int`。必须用单引号包裹：`tmux-cli send "'2'" --pane=...`。常见场景：响应审批菜单、回应数字选项菜单、按错回退后重新输入数字键
- **二次确认**：codewhale 等 TUI 的审批菜单要求"两次按键确认"，发完第一个 `2` 后需 `sleep 1` 再发第二个：

  ```bash
  tmux-cli send "'2'" --pane={pane_id}
  sleep 1
  tmux-cli send "'2'" --pane={pane_id}
  ```

- **prompt 必须单行**：上文 SEND 步骤已强调，prompt 不能含裸换行（会被解释为多次回车）

### 获取结果

- **code-writer**：代码直接修改在项目文件中
- **reviewer**（plan-reviewer / code-reviewer）：reviewer 自行获取项目状态（git diff、Read 文件等），
  将 review 结果写入总控指定的 `/tmp/tmux-agents/{session_id}/` 下的文件，总控用 Read 工具读取。
  reviewer 的 role prompt 约束其不得修改项目源代码，只能写 temp 目录下的文件

### 完成标记

- `TASK_COMPLETE` — code-writer 完成编码任务
- `REVISION_COMPLETE` — code-writer 完成 review 反馈的修改
- `REVIEW_COMPLETE` — reviewer 完成审查
- `AGENT_READY` — agent 就绪确认

### 安全约束

以下约束**在 claude runtime 下由工具层硬保证**（`--disallowed-tools`）；**在 codewhale 和 codex runtime 下仅为 role prompt 软约束**（详见后文"Runtime 差异与安全说明"）：

- 子 agent 禁止执行 git commit、git push、git reset、git clean、gh pr create
- 只有总控（用户的主 session）有权执行 git 操作
- reviewer 的 role prompt 约束不得修改项目源代码，只能写 `/tmp/tmux-agents/` 下的 temp 文件
- **API key 暴露面（claude/codewhale）**：key 经 `tmux-cli send` 传递时会出现在：
  1. 主控 `tmux-cli send` 进程 cmdline（`ps aux` 可见）
  2. 子 pane 的 `~/.zsh_history`（除非用户配了 HISTCONTROL/HISTIGNORE 过滤）
  3. 启动那一帧的 `tmux-cli capture` 输出
  这是 `tmux-cli send` 设计的固有限制，不是 runtime 差异。原"环境变量前缀不进 cmdline"只对 LLM 子进程自身成立（变量在 `/proc/<pid>/environ` 而非 cmdline），对 `tmux-cli send` 这条路径无效
- **Codex runtime 不接受 API key**：`runtime: codex` 使用当前 Codex 已配置的认证和 profile，不支持在 `agents.json` 中传入 `api_base` 或 `api_key`。

### Runtime 差异与安全说明

不同 runtime 对"工具黑名单"机制的支持差异显著，本插件**有意保留这种不对称**，使用者需理解差异：

| 维度 | claude runtime | codewhale runtime | codex runtime |
|---|---|---|---|
| 工具黑名单 | `--disallowed-tools` **启动参数硬约束**，工具层拦截，任何路径都挡得住 | 无原生机制，仅依靠 **role prompt 软约束** | 无等价启动参数，仅依靠 **role prompt 软约束** |
| 权限字段 | `permission_mode` 直接使用 | `permission_mode` 被忽略，统一 `--yolo` | 使用 `sandbox` 和 `approval_policy`，不支持 `permission_mode` |
| disallowed_tools 字段 | 实际生效 | 不应配置（无等价机制，配置反而误导） | 不支持；出现时 fail fast |
| API 字段 | 可用 `api_base`/`api_key` 配置 Anthropic 兼容接口 | 可用 `api_base`/`api_key` 配置 litellm 兼容代理 | 不支持；使用当前 Codex 配置 |
| 工具直接读写 workspace 外路径 | claude 受 cwd 默认限制（但可被 shell 调用绕过） | codewhale-tui --yolo 下 trust mode 默认放行（无需绕 shell） | 受 Codex sandbox/profile/配置控制 |

**为什么不对称**：claude 的硬约束零成本可用，启用它没有理由不用；codewhale/codex 没有等价的工具黑名单启动参数，因此本插件不伪造硬约束，由 role prompt、sandbox 和总控人工把关兜底。

**为什么 codewhale 必须用 `codewhale-tui --yolo`**：codewhale 的 `read_file`/`write_file` 工具有"workspace 内才能读写"的应用层硬校验（独立于 OS 沙箱）。tmux-agents 通信协议需要子 agent 读 plugin 内置的 role 文件（cache 目录，workspace 外）、写 `/tmp/tmux-agents/` 通信目录。`codewhale --yolo`（npm 包装器）只放开审批不放开 workspace 校验，必须直接调 `codewhale-tui --yolo` 才会启用 trust mode 让任意路径都通——这是 codewhale 的设计取舍，不是本插件能收窄的。

**使用建议**：
- 默认 runtime 是 `claude`，有完整安全保护
- 用 codewhale runtime 时，安全约束由 role prompt 提供，工具层无兜底
- 用 codex runtime 时，安全约束由 role prompt 和 Codex sandbox/profile 提供，工具层无 `--disallowed-tools` 等价兜底
- 所有 git 写操作仍只在总控层执行，子 agent 任何 runtime 都不应触碰

### 通用安全说明

不要在不信任的项目目录（克隆别人代码后未审阅）下启动任何 runtime 的子 agent。子 agent 跟你在同一 UID 下运行，prompt 注入可让它执行任意 shell 命令——这跟 runtime 选择无关，是 "tmux pane as agent" 设计的固有限制。所有 git 写操作只在总控层执行，子 agent 任何 runtime 都不应触碰。

### Runtime 适配器接口（扩展指引）

如果未来要新增一个 runtime（例如 gemini、qwen），步骤三需要按以下四个维度定义一个新分支 3.3.X：

| 维度 | 说明 |
|---|---|
| 启动命令 | 启动该 runtime CLI 的命令模板，含模型、workspace、API 等参数；是否支持环境变量内联自定义 API |
| prompt 注入方式 | system prompt 怎么注入：启动参数？启动后 send 首条消息让其 Read 文件？是否需要单独的 readiness 验证步骤 |
| 就绪检测 | wait_idle 参数（idle-time / timeout）和验证 AGENT_READY 的策略 |
| 安全约束机制 | 是否有原生工具黑名单参数。如果没有，明示走 role prompt 软约束并在文档中说明 |

新增 runtime 不应改动通信协议（pane title 锁定、SEND/WAIT/CAPTURE 循环、完成标记），这些对 runtime 透明。

### 已知限制

- 多个总控 session 并发使用同一套 agent 会产生竞争，当前设计为单用户场景
- **schema 设计缺陷待修**：`api_base`/`api_key` 字段在 `runtime: claude` 和 `runtime: codewhale` 下语义不同——claude 指 LLM provider 原生 endpoint/key，codewhale 当前指 litellm 代理 endpoint/key。`runtime: codex` 明确不支持这两个字段。长期应 split 成 `claude_api_base`/`codewhale_api_base` 或引入 `provider_kind` 字段
- **example role 命名约定**：当前 `code-reviewer-codewhale` 这个角色名同时编码了 type 和 runtime，但 schema 字段已经有 type 和 runtime 单独表达，是否要约定/强制命名规范待讨论
- **pane title 易被 OSC 序列覆盖**：tmux 默认允许子进程通过 OSC 转义改写 pane title。`claude` runtime 有 `CLAUDE_CODE_DISABLE_AUTO_TITLE=1` 防护，但 zsh/codewhale-tui 等其他 runtime 没有等价机制。步骤 3.2 通过 `tmux set-option -p {pane} allow-rename off` + `automatic-rename off` 在 pane 级别锁死 title。如果未来发现某个 runtime 仍能改 title，说明它绕过了 tmux 的 rename 拦截（理论上不该发生），需要回到该 runtime 自身的禁用机制

### 文本安全

- prompt 中避免裸 `#数字`（如 `PR #45`），改写为 `PR 45` 或完整 URL
- 某些 tmux 环境会截断 `#` 后的内容

### 故障处理

- `wait_idle` 超时：先 `tmux-cli capture` 检查部分输出
- agent 无响应：`tmux-cli interrupt --pane={id}` 后重试一次
- 仍无响应：报告给用户，建议手动检查对应 pane



清理环境请使用 `/tmux-agents:cleanup`。
