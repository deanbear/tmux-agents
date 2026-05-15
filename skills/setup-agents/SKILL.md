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

启动多 Agent 协作环境：根据配置文件中定义的所有角色，在 tmux 中为每个角色开启独立的 Claude Code session，锁定 pane title，验证就绪状态。

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

如果未安装，告知用户：`uv tool install claude-code-tools`，退出。

3. 确认在 tmux 环境中：

```bash
tmux-cli status
```

如果报错或显示不在 tmux 中，告知用户需要先启动 tmux session（`tmux new -s work`），退出。

4. 定位配置文件，按以下优先级用 Bash 检查文件是否存在，找到第一个后用 Read 工具读取：

```bash
test -f "$TMUX_AGENTS_CONFIG" && echo "$TMUX_AGENTS_CONFIG"
test -f ~/.claude/tmux-agents/agents.json && echo "~/.claude/tmux-agents/agents.json"
```

如果都不存在，使用 Read 工具读取 `${CLAUDE_SKILL_DIR}/../../config/agents.example.json`。

读取到 JSON 内容后，在对话中直接解析配置。

---

## 步骤二：检查已有 Agent

运行 `tmux-cli status`，解析输出，查找 pane title 与配置中各角色的 `pane_title` 匹配的 pane。

对每个角色：
- 如果已有匹配的 pane：报告 "✓ {role} 已在运行 (pane {id})"，跳过该角色的启动
- 如果没有：继续步骤三

---

## 步骤三：逐个启动 Agent

对每个需要启动的角色，执行以下子步骤：

### 3.0 启用 pane border 显示角色名

在启动第一个 agent 之前，执行：

```bash
tmux set-option -w pane-border-format "#{pane_title}"
tmux set-option -w pane-border-status top
```

### 3.1 启动 shell pane

```bash
tmux-cli launch "zsh"
```

记录返回的 pane id。

### 3.2 锁定 pane title

```bash
tmux-cli send "tmux select-pane -T '{pane_title}'" --pane={pane_id}
```

等待完成：

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=2 --timeout=10
```

### 3.3 组装并发送 claude 启动命令

根据角色配置组装命令。从配置中读取 `model`、`disallowed_tools`、`permission_mode`，如果角色没有指定则使用 `defaults` 中的值。

**配置值约束**：`pane_title` 和 `disallowed_tools` 的值不得包含单引号（`'`），因为它们会被单引号包裹后嵌入 tmux send 的双引号命令中。如果配置值包含单引号，命令会断裂导致启动失败。

角色定义文件路径：如果角色配置中有 `role_file` 字段，使用该路径；否则按 `type` 匹配插件内置的 role 文件（`${CLAUDE_SKILL_DIR}/../../roles/{type}.md`）。如果文件不存在则跳过 `--append-system-prompt-file` 参数。

**基础命令**（使用 Anthropic 原生 API）：

```bash
tmux-cli send "CLAUDE_CODE_DISABLE_AUTO_TITLE=1 claude --model {model} --permission-mode {permission_mode} --append-system-prompt-file ${CLAUDE_SKILL_DIR}/../../roles/{type}.md --disallowed-tools '{disallowed_tools}' --name '{pane_title}'" --pane={pane_id}
```

**自定义 API 命令**（角色配置中包含 `api_base` 和 `api_key`）：

将 API 地址和 key 内联到 claude 启动命令前，只对该进程生效，不污染 pane 的 shell 环境：

```bash
tmux-cli send "CLAUDE_CODE_DISABLE_AUTO_TITLE=1 ANTHROPIC_BASE_URL={api_base} ANTHROPIC_AUTH_TOKEN={api_key} claude --model {model} --permission-mode {permission_mode} --append-system-prompt-file ${CLAUDE_SKILL_DIR}/../../roles/{type}.md --disallowed-tools '{disallowed_tools}' --name '{pane_title}'" --pane={pane_id}
```

其中 `${CLAUDE_SKILL_DIR}` 由 Claude Code 自动展开为当前 SKILL.md 所在目录的绝对路径。

### 3.4 等待 Claude 启动就绪

```bash
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=60
```

### 3.5 验证 Agent 响应

```bash
tmux-cli send "回复 AGENT_READY 确认你已就绪" --pane={pane_id}
tmux-cli wait_idle --pane={pane_id} --idle-time=5 --timeout=30
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

- 子 agent 禁止执行 git commit、git push、git reset、git clean、gh pr create
- 只有总控（用户的主 session）有权执行 git 操作
- reviewer 的 role prompt 约束不得修改项目源代码，只能写 `/tmp/tmux-agents/` 下的 temp 文件
- 自定义 API 的 key 通过命令行内联环境变量传递，只对 claude 进程生效，不污染 pane shell 环境

### 已知限制

- 多个总控 session 并发使用同一套 agent 会产生竞争，当前设计为单用户场景

### 文本安全

- prompt 中避免裸 `#数字`（如 `PR #45`），改写为 `PR 45` 或完整 URL
- 某些 tmux 环境会截断 `#` 后的内容

### 故障处理

- `wait_idle` 超时：先 `tmux-cli capture` 检查部分输出
- agent 无响应：`tmux-cli interrupt --pane={id}` 后重试一次
- 仍无响应：报告给用户，建议手动检查对应 pane



清理环境请使用 `/tmux-agents:cleanup`。
