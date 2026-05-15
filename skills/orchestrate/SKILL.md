---
name: orchestrate
description: >-
  Full tmux-agents orchestration: plan, review plan, write code, review code.
  Supports --skip-plan (skip plan phase, go straight to coding) and --plan-only
  (stop after plan review). Requires agents to be running or will auto-start them.
  Trigger phrases: "tmux-agents workflow", "tmux agent 协作", "全流程",
  "/tmux-agents:orchestrate".
user-invocable: true
---

# Tmux-Agents 全流程编排

完整的多 Agent 协作流程：出方案 → 审查方案 → 写代码 → 审查代码。

支持参数：
- `--skip-plan`：跳过方案阶段，直接写代码 + 审查（适用于已有明确方案的场景）
- `--plan-only`：只做方案 + 审查方案，不写代码（适用于只想讨论设计的场景）

---

## 步骤零：解析参数

从用户输入中识别：
- `--skip-plan`：设置 SKIP_PLAN=true
- `--plan-only`：设置 PLAN_ONLY=true
- 其余内容作为需求描述

---

## 步骤一：确保 Agent 就绪

读取配置文件，按 `type` 字段识别各角色。运行 `tmux-cli status`，检查需要的 agent 是否在运行：

- 默认模式：需要 `type: code-writer`、`type: plan-reviewer`、`type: code-reviewer`
- `--skip-plan`：需要 `type: code-writer`、`type: code-reviewer`
- `--plan-only`：需要 `type: plan-reviewer`

对于未运行的 agent，按照 `/tmux-agents:setup-agents` 的步骤三（逐个启动 Agent）执行启动。不要在此重复启动细节，直接参照 setup-agents SKILL.md 中的步骤三。

如果有 agent 启动失败，明确报告哪个 agent 失败及原因，停止流程。不要在部分 agent 不可用的情况下继续。

---

## 步骤二：创建 Session

```bash
SESSION_ID=$(date +%Y%m%d-%H%M%S)-$$
mkdir -p /tmp/tmux-agents/$SESSION_ID
```

---

## Phase 1：出方案（SKIP_PLAN=true 时跳过）

1. 根据用户的需求描述，使用 `templates/plan-template.md` 的结构，填写完整的实现方案：
   - **目标**：一句话说清楚要做什么
   - **方案**：具体步骤，改哪些文件/模块
   - **影响范围**：会波及哪些现有功能
   - **风险点**：最容易出问题的地方

2. 将 plan 写入 `/tmp/tmux-agents/{session_id}/plan.md`

3. 展示 plan 给用户做快速确认，再发给 reviewer

---

## Phase 2：审查方案（SKIP_PLAN=true 时跳过）

遵循 setup-agents 中定义的通信协议。

### Round 1

1. 确认 plan-reviewer pane id（`tmux-cli status`）
2. 等待 agent 空闲
3. 发送 review 任务（单行 prompt）：

```bash
tmux-cli send "你有一个 plan review 任务。请用 Read 工具读取 /tmp/tmux-agents/{session_id}/plan.md，按照你的审查维度进行审查。将 review 结果写入 /tmp/tmux-agents/{session_id}/plan-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

4. 等待 `REVIEW_COMPLETE`（`tmux-cli wait_idle` + `tmux-cli capture` 检查标记）
5. 用 Read 工具读取 `/tmp/tmux-agents/{session_id}/plan-review-r1.md`

### 总控裁决

- 向用户展示 review 结果，说明哪些意见有效、哪些过度
- 如果 `APPROVE`：进入 Phase 3
- 如果 `REQUEST_CHANGES`：
  - 将用户同意的修改应用到 plan，更新 `plan.md`
  - 进入 Round 2

### Round 2（如需）

- 重复 Round 1 流程，持久化路径改为 `plan-review-r2.md`
- 完成后总控做最终裁决，不再发起 Round 3
- 向用户报告最终 plan 状态

如果 PLAN_ONLY=true：展示最终 plan，流程结束。

---

## Phase 3：写代码

1. 准备任务文件 `/tmp/tmux-agents/{session_id}/code-task.md`，内容包含：
   - 经过 review 的 plan（默认模式）或用户的需求描述（`--skip-plan` 模式）
   - 明确的实现指令
   - 允许修改的文件范围
   - 任何需要遵守的约束

2. 确认 code-writer 角色的 pane id（按配置中 `type: code-writer` 角色的 `pane_title` 在 `tmux-cli status` 中查找）

3. 发送任务：

```bash
tmux-cli send "你有一个编码任务。请用 Read 工具读取 /tmp/tmux-agents/{session_id}/code-task.md，按照任务要求实现。完成后输出 TASK_COMPLETE" --pane={id}
```

4. 等待 `TASK_COMPLETE`

---

## Phase 4：审查代码

### Round 1

1. 确认 code-reviewer pane id（`tmux-cli status`）
2. 等待 agent 空闲
3. 发送 review 任务（单行 prompt）：

如果有 plan：
```bash
tmux-cli send "你有一个 code review 任务。请自行用 git diff 查看工作区变更（注意：code-writer 禁止 commit，所有改动在工作区），用 git ls-files --others --exclude-standard 检查新文件。读取 /tmp/tmux-agents/{session_id}/plan.md 理解设计意图。按照你的审查维度审查，将结果写入 /tmp/tmux-agents/{session_id}/code-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

如果没有 plan（`--skip-plan` 模式）：
```bash
tmux-cli send "你有一个 code review 任务。请自行用 git diff 查看工作区变更（注意：code-writer 禁止 commit，所有改动在工作区），用 git ls-files --others --exclude-standard 检查新文件。按照你的审查维度审查，将结果写入 /tmp/tmux-agents/{session_id}/code-review-r1.md，完成后输出 REVIEW_COMPLETE" --pane={id}
```

4. 等待 `REVIEW_COMPLETE`（`tmux-cli wait_idle` + `tmux-cli capture` 检查标记）
5. 用 Read 工具读取 `/tmp/tmux-agents/{session_id}/code-review-r1.md`

### 总控裁决

- 向用户展示 review 结果
- 如果 `APPROVE`：进入汇总
- 如果 `REQUEST_CHANGES`：
  - 过滤出用户同意采纳的意见
  - 将过滤后的意见写入 `/tmp/tmux-agents/{session_id}/review-feedback-r1.md`
  - 发送修复任务给 code-writer 角色（prompt 引用 feedback 文件路径）
  - 等待 `REVISION_COMPLETE`
  - 进入 Round 2

### Round 2（如需）

- 重复 Round 1 流程，diff 和持久化路径使用 r2 版本
- 完成后总控做最终裁决，不再发起 Round 3

---

## 步骤末：汇总

展示最终结果：

```
Tmux-Agents 全流程完成

方案：
  状态：{APPROVE / 采纳修改后通过 / 总控裁决通过 / 跳过}
  Review 轮次：{0 / 1 / 2}

代码变更：
  修改文件：{文件列表}
  状态：{APPROVE / 采纳修改后通过 / 总控裁决通过}
  Review 轮次：{1 / 2}

Session 目录：/tmp/tmux-agents/{session_id}/

建议下一步：
  git diff                # 查看完整变更
  git add -p              # 逐块确认
  git commit -m "..."     # 提交
```

不自动执行 git 操作，由用户决定提交策略。

询问用户是否要保留 agent 继续下一个任务，还是清理环境。
