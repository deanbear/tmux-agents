---
name: code-review
description: >-
  Use this skill to get an independent code review from the code-reviewer agent.
  Reviews uncommitted changes on the current branch. Requires agents to be running
  (use /tmux-agents:setup-agents first). Trigger phrases: "review this code",
  "代码审查", "code review", "/tmux-agents:code-review".
user-invocable: true
---

# Code Review

让独立的 code-reviewer agent 审查当前分支的代码变更，总控裁决反馈，最多 2 轮。

---

## 步骤一：发现 agent

按 setup-agents 中定义的配置查找顺序读取配置文件（`$TMUX_AGENTS_CONFIG` → 当前宿主的默认配置目录 → 插件内 `config/agents.example.json`），找到 `type` 为 `code-reviewer` 和 `code-writer` 的角色，记录其 `pane_title`。

运行 `tmux-cli status`，查找：
- `type: code-reviewer` 对应的 pane（必须）
- `type: code-writer` 对应的 pane（可选，用于自动修复）

如果 code-reviewer 未找到：告知用户先运行 `/tmux-agents:setup-agents`，退出。

---

## 步骤二：准备 session

```bash
SESSION_ID=$(date +%Y%m%d-%H%M%S)-$$
mkdir -p /tmp/tmux-agents/$SESSION_ID
```

如果存在 plan 文件（用户指定路径或对话上下文中已有），写入 `/tmp/tmux-agents/{session_id}/plan.md`。

---

## 步骤三：发送 review 任务

遵循 setup-agents 中定义的通信协议：

1. 确认 code-reviewer pane id（重新运行 `tmux-cli status`）

2. 确认 agent 空闲：
```bash
tmux-cli wait_idle --pane={id} --idle-time=3 --timeout=10
```

3. 发送任务 prompt（单行文本）：

如果有 plan：
```bash
tmux-cli send "你有一个 code review 任务。请自行用 git diff 查看工作区变更（注意：code-writer 禁止 commit，所有改动在工作区），用 git ls-files --others --exclude-standard 检查新文件。读取 /tmp/tmux-agents/{session_id}/plan.md 理解设计意图。按照你的审查维度审查，将结果写入 /tmp/tmux-agents/{session_id}/code-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

如果没有 plan：
```bash
tmux-cli send "你有一个 code review 任务。请自行用 git diff 查看工作区变更（注意：code-writer 禁止 commit，所有改动在工作区），用 git ls-files --others --exclude-standard 检查新文件。按照你的审查维度审查，将结果写入 /tmp/tmux-agents/{session_id}/code-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

4. 等待完成：
```bash
tmux-cli wait_idle --pane={id} --idle-time={defaults.idle_time} --timeout={defaults.idle_timeout}
```

如果超时：先 capture 检查部分输出；无响应则 interrupt 后重试一次；仍无响应报告用户。

5. 检查完成：
```bash
tmux-cli capture --pane={id}
```
确认输出包含 `REVIEW_COMPLETE`。

---

## 步骤四：总控裁决

用 Read 工具读取 `/tmp/tmux-agents/{session_id}/code-review-r1.md`。

向用户展示 review 结果，附上总控的判断：

- 哪些意见有效
- 哪些意见过度或不适用

如果 Verdict 是 `APPROVE`：
- 报告代码通过审查，流程结束

如果 Verdict 是 `REQUEST_CHANGES`：
- 过滤出用户同意采纳的意见
- 如果 code-writer agent 在运行：进入步骤五（自动修复）
- 如果 code-writer agent 不在：展示需要修复的问题列表，用户自行修改后可重新触发 review

---

## 步骤五：自动修复 + 第 2 轮 review（如需）

### 5.1 发送修复任务给 code-writer

将过滤后的 review 意见写入 `/tmp/tmux-agents/{session_id}/review-feedback-r1.md`。

确认 code-writer 角色的 pane id，遵循通信协议发送：

```bash
tmux-cli send "你有一个代码修复任务。请用 Read 工具读取 /tmp/tmux-agents/{session_id}/review-feedback-r1.md 查看需要修复的 review 意见，逐一修复。完成后输出 REVISION_COMPLETE" --pane={code-writer-id}
```

等待 `REVISION_COMPLETE`。

### 5.2 第 2 轮 review

发送第 2 轮 review 给 code-reviewer：
- 输出路径改为 `code-review-r2.md`
- prompt 中说明这是修改后的重审，reviewer 自行获取最新变更

### 5.3 总控最终裁决

第 2 轮完成后，无论 Verdict 是什么，总控做最终裁决：
- 向用户展示剩余争议点（如有）
- 不再发起第 3 轮

---

## 步骤六：输出

展示最终结果：

```
Code Review 完成

状态：{APPROVE / 采纳修改后通过 / 总控裁决通过}
轮次：{1 / 2}
修改文件：{文件列表}

建议下一步：
  git add -p          # 逐块确认修改
  git commit -m "..."
```

不自动执行 git 操作。
