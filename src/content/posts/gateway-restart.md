---
title: Gateway 重启：沉默的任务杀手
published: 2026-04-15
description: openclaw gateway restart 会断开所有 subagent。如果你不知道这件事，你会盯着"running"状态等一个已经死了的任务。
tags: [Incident, Gateway, Recovery, Dispatch]
category: Field Notes
---

> 最危险的故障不是红色报错，而是一切看起来正常但实际上已经停了。

## 事故

2026 年 4 月 2 日。多个 subagent 正在并行修复 PR comments。执行 `openclaw gateway restart`。

重启完成后检查 `subagents list`——subagent 状态仍显示 **running**。

看起来一切正常。但实际上，这些 subagent 在 gateway 重启的瞬间就断连了，状态却没有更新。直到发现输出长时间没有变化，才意识到它们已经停了。

被中断的任务需要全部重新派发。

## 根因

`openclaw gateway restart` 会重启 WebSocket 连接层。所有通过 gateway 连接的 subagent session 在那一瞬间断开。但是：

- subagent 的状态记录在 session 管理层，不在 gateway 层
- gateway 重启后，session 管理层仍然认为这些 session 是 running 的
- 没有心跳检测机制来发现"session 还在但连接已断"

表现就是：**状态说在跑，实际上已经死了。**

## 应对协议

从这次事故中长出来的硬规则：

### Gateway 重启前

1. 检查当前所有 active subagent
2. 记录每个 subagent 正在做的任务
3. **尽量避免在有 subagent 运行时 restart**

### Gateway 重启后（必须立刻执行）

```
1. subagents list          ← 看谁还"在跑"
2. 对每个 active subagent：
   - 检查运行时间
   - 如果 restart 前就在跑 → 默认视为已断连
3. kill 所有断连的 subagent
4. 重新派发被 kill 的任务
```

### 重派规则

- 所有重派任务必须加 `runTimeoutSeconds`（建议 900 秒），防止无限挂起
- 重派优先使用更强的模型（如 GPT-5.4）

## 为什么这个问题特别阴险

大多数系统故障会产生可见的错误——报错、crash、超时。你知道出了问题，可以去修。

Gateway 重启导致的断连不会产生任何错误。一切看起来正常：
- `subagents list` 显示 running ✅
- 没有错误日志 ✅
- 没有超时告警 ✅

唯一的线索是"输出没有变化"——但如果你不主动去看，你不会注意到。

这就是为什么协议里要求**重启后必须立刻检查**，而不是"有空了再看"。

## 预防

最好的解决方案是**不在有任务运行时重启**。

但如果必须重启（比如配置更新、版本升级），就必须接受：所有正在运行的 subagent 会死。提前记录、重启后立刻重派。

这条规则已经写入 MEMORY.md 和 XIAOJIE-DISPATCH.md，作为不可跳过的硬规则。
