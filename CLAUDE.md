# tmux-agents 开发指南

## 目录结构约定

```
.claude-plugin/     插件元数据，包含 plugin.json
skills/             各 skill 定义，每个子目录一个 SKILL.md
roles/              子 Agent 角色定义文件（系统提示词）
config/             配置示例文件
templates/          任务文件模板
```

## SKILL.md 编写规范

每个 skill 目录下的 SKILL.md 必须包含 frontmatter：

```yaml
---
name: skill-name
description: 一句话描述，供 Claude 判断何时触发
user-invocable: true   # 用户可通过 /tmux-agents:skill-name 直接触发
---
```

内容按步骤组织，每个步骤说明：
1. 做什么（操作）
2. 怎么做（调用哪个工具、发什么消息）
3. 等待什么（期望的输出标记）

skill 文件中引用 role 文件时使用相对路径：`${CLAUDE_SKILL_DIR}/../../roles/{type}.md`

## roles 文件约束

- **plan-reviewer** 和 **code-reviewer**：不得修改项目源代码，只输出审查意见
- **code-writer**：不得执行 `git commit`、`git push`、`gh pr create` 等操作，不得自行审查自己的代码
- 所有子 Agent：只与总控通信，不主动发起行为，等待总控指令

## 配置文件

`config/agents.example.json` 展示最简配置，示例配置只展示必要字段。
