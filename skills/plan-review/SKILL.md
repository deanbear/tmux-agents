---
name: plan-review
description: >-
  Use this skill to get an independent plan review from the plan-reviewer agent.
  Requires agents to be running (use /tmux-agents:setup-agents first).
  Trigger phrases: "review this plan", "审查方案", "plan review",
  "/tmux-agents:plan-review".
user-invocable: true
---

# Plan Review

将实现方案发给独立的 plan-reviewer agent 审查，总控裁决反馈，最多 2 轮。

---

## 步骤一：发现 plan-reviewer agent

按 setup-agents 中定义的配置查找顺序（`$TMUX_AGENTS_CONFIG` → `~/.claude/tmux-agents/agents.json` → 插件内 `config/agents.example.json`）读取配置文件，找到 `type` 为 `plan-reviewer` 的角色，记录其 `pane_title`。

运行 `tmux-cli status`，在输出中查找匹配该 `pane_title` 的 pane。

- 找到：记录 pane id，继续
- 未找到：告知用户先运行 `/tmux-agents:setup-agents`，退出

---

## 步骤二：准备 plan 文档

创建 session 目录：

```bash
SESSION_ID=$(date +%Y%m%d-%H%M%S)-$$
mkdir -p /tmp/tmux-agents/$SESSION_ID
```

准备 plan 文件 `/tmp/tmux-agents/{session_id}/plan.md`：

- 如果用户提供了 plan 文件路径：读取内容，复制到 temp 目录
- 如果用户描述了需求但没有现成 plan：总控使用 `templates/plan-template.md` 模板结构，根据需求填写 plan，写入 temp 文件，展示给用户确认
- 如果当前对话上下文中已有 plan：提取并写入 temp 文件

---

## 步骤三：发送 review 任务

遵循 setup-agents 中定义的通信协议：

1. 确认 plan-reviewer pane id（重新运行 `tmux-cli status`）

2. 确认 agent 空闲：
```bash
tmux-cli wait_idle --pane={id} --idle-time=3 --timeout=10
```

3. 发送任务 prompt（单行文本）：
```bash
tmux-cli send "你有一个 plan review 任务。请用 Read 工具读取 /tmp/tmux-agents/{session_id}/plan.md，按照你的审查维度进行审查。将 review 结果写入 /tmp/tmux-agents/{session_id}/plan-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

4. 等待完成：
```bash
tmux-cli wait_idle --pane={id} --idle-time={defaults.idle_time} --timeout={defaults.idle_timeout}
```

如果超时：先 `tmux-cli capture` 检查部分输出。如果 agent 无响应，`tmux-cli interrupt --pane={id}` 后重试一次。仍无响应则报告用户。

5. 检查完成：
```bash
tmux-cli capture --pane={id}
```
确认输出包含 `REVIEW_COMPLETE`。

---

## 步骤四：总控裁决

用 Read 工具读取 `/tmp/tmux-agents/{session_id}/plan-review-r1.md`。

向用户展示 review 结果，附上总控的判断：

- 哪些意见有效，应当采纳
- 哪些意见总控认为过度或不适用（说明原因）

如果 Verdict 是 `APPROVE`：
- 报告 plan 通过审查，流程结束

如果 Verdict 是 `REQUEST_CHANGES`：
- 将用户同意采纳的修改应用到 plan
- 更新 `/tmp/tmux-agents/{session_id}/plan.md`
- 进入步骤五（第 2 轮 review）

---

## 步骤五：第 2 轮 review（如需）

重复步骤三的流程，但：
- 输出路径改为 `/tmp/tmux-agents/{session_id}/plan-review-r2.md`
- prompt 中说明这是修改后的重审

第 2 轮完成后，无论 Verdict 是什么，总控做最终裁决：
- 向用户展示剩余争议点（如有）
- 总控决定最终 plan，不再发起第 3 轮

---

## 步骤六：输出

展示最终结果：

```
Plan Review 完成

状态：{APPROVE / 采纳修改后通过 / 总控裁决通过}
轮次：{1 / 2}

最终 plan 路径：/tmp/tmux-agents/{session_id}/plan.md
```
