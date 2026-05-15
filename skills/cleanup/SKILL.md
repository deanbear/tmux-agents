---
name: cleanup
description: >-
  Use this skill to shut down all tmux-agents sub-agents and clean up temp files.
  Trigger phrases: "cleanup agents", "清理 agent", "关闭 agent", "停止 agent",
  "/tmux-agents:cleanup".
user-invocable: true
---

# Cleanup

关闭所有子 agent pane，恢复 tmux 状态，清理临时文件。

---

## 步骤一：识别主控 Pane

先确定当前主控 pane（即正在执行 cleanup 的这个 pane），后续 kill 时必须排除它。

**获取主控 pane id**（不依赖 tmux 焦点，防止焦点漂移导致误杀）：

```bash
echo $TMUX_PANE
```

`$TMUX_PANE` 是 tmux 注入到每个 pane 进程的环境变量，标识当前进程所在的 pane，不受鼠标点击或 focus 切换影响。记录其值作为 **master_pane_id**。

然后获取所有 pane 状态：

```bash
tmux-cli status
```

---

## 步骤二：发现运行中的 Agent

按 setup-agents 中定义的配置查找顺序（`$TMUX_AGENTS_CONFIG` → `~/.claude/tmux-agents/agents.json` → 插件内 `config/agents.example.json`）读取配置文件，获取所有角色的 `pane_title`。

从步骤一的 `tmux-cli status` 输出中查找 pane title 匹配这些角色的 pane，**排除 master_pane_id**（即 `$TMUX_PANE` 的值）。

将所有待关闭的 pane id 收集到一个列表中，一次性确定后再开始 kill（避免逐个 kill 时 focus 漂移导致 pane id 变化）。

**安全校验**：在 kill 之前，逐一确认列表中没有 master_pane_id。这是最后一道防线。

如果没有找到任何 agent pane，告知用户当前没有运行中的 agent。

---

## 步骤三：关闭 Agent Pane

对收集到的 agent pane 列表，**从后往前逐个关闭**（逆序 kill，避免 tmux 自动切换 focus 到尚未处理的 pane）：

```bash
tmux-cli kill --pane={pane_id}
```

如果某个 pane kill 失败（例如已自行退出），跳过并继续处理下一个。

---

## 步骤四：恢复 tmux 状态

```bash
tmux set-option -w pane-border-status off
```

---

## 步骤五：清理临时文件

列出所有 session 目录：

```bash
ls /tmp/tmux-agents/
```

如果有内容，询问用户：
- 清理全部：`rm -rf /tmp/tmux-agents/`
- 清理指定 session：`rm -rf /tmp/tmux-agents/{session_id}/`
- 保留不清理

---

## 步骤六：报告

```
Cleanup 完成

  关闭的 agent：{列表}
  清理的 session：{列表或"未清理"}
```
