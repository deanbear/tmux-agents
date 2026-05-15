# tmux-agents

基于 tmux 的多 Agent 协作编排：通过 tmux pane 管理独立的子 Agent，消除同一 Agent 自写自审的确认偏差。

## 工作原理

tmux-agents 使用总控 + 子 Agent 的架构：

- **总控**（你的 Claude Code 主 session）负责拆解任务、派发指令、整合结果
- **子 Agent**（各 tmux pane 中独立运行的 Claude Code 实例）分别承担 plan-reviewer、code-writer、code-reviewer 等角色

每个子 Agent 运行在独立的 tmux pane 中，拥有独立的上下文，不受总控影响。总控通过 tmux-cli 向子 Agent 发送消息、读取输出，实现跨 pane 的协作编排。

## 前置依赖

- [tmux](https://github.com/tmux/tmux)

  ```bash
  brew install tmux        # macOS
  apt install tmux         # Ubuntu/Debian
  ```

- [Claude Code CLI](https://docs.anthropic.com/claude-code)

- **tmux-cli**（由 `claude-code-tools` 提供）

  ```bash
  uv tool install claude-code-tools   # 推荐
  # 或者
  pip install claude-code-tools
  ```

## 安装

```bash
claude plugins add https://github.com/<your-github-username>/tmux-agents
```

（项目发布后替换为实际的 GitHub URL）

## 配置

安装后，将示例配置复制到 `~/.claude/tmux-agents/agents.json`：

```bash
mkdir -p ~/.claude/tmux-agents && cp config/agents.example.json ~/.claude/tmux-agents/agents.json
```

然后按需编辑，格式参考 `config/agents.example.json`：

```json
{
  "defaults": {
    "model": "sonnet",
    "permission_mode": "default",
    "idle_timeout": 120,
    "idle_time": 5
  },
  "roles": {
    "code-writer": {
      "type": "code-writer",
      "pane_title": "code-writer",
      "model": "sonnet",
      "permission_mode": "acceptEdits",
      "disallowed_tools": "..."
    },
    "plan-reviewer": { ... },
    "code-reviewer": { ... }
  }
}
```

`api_base` 和 `api_key` 是可选字段，只有在通过自定义代理访问 Claude API 时才需要配置，直接使用 Anthropic 官方 API 时不需要。

## 使用方式

在 Claude Code 主 session 中通过 `/` 触发各 skill：

| Skill | 触发短语 | 说明 |
|-------|----------|------|
| `setup-agents` | `/tmux-agents:setup-agents` | 启动子 Agent，或发现已有 Agent pane |
| `orchestrate` | `/tmux-agents:orchestrate` | 完整编排流程：plan -> review -> code -> review |
| `plan-review` | `/tmux-agents:plan-review` | 将当前方案发给 plan-reviewer 独立审查 |
| `code-review` | `/tmux-agents:code-review` | 将当前代码变更发给 code-reviewer 独立审查 |
| `cleanup` | `/tmux-agents:cleanup` | 关闭所有子 Agent pane，清理临时文件 |

## 目录结构

```
tmux-agents/
├── .claude-plugin/     # 插件元数据（plugin.json）
├── skills/             # 各 skill 定义，每个子目录包含一个 SKILL.md
│   ├── setup-agents/
│   ├── orchestrate/
│   ├── plan-review/
│   ├── code-review/
│   └── cleanup/
├── roles/              # 子 Agent 角色定义（系统提示词）
│   ├── code-writer.md
│   ├── code-reviewer.md
│   └── plan-reviewer.md
├── config/             # 配置示例
│   └── agents.example.json
└── templates/          # 任务文件模板
```
