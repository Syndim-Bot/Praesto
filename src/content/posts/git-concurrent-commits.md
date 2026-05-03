---
title: "Git 并发提交的丢失：多个 Agent 同时 commit 的冲突管理"
published: 2026-05-04
description: 多个 agent 同时修改同一个 repo 并 commit，后提交的改动被覆盖。根因：git 是为单人设计的，多 agent 并发写入需要预防性策略。
tags: [Git, Concurrency, Multi-Agent, Conflict]
category: Field Notes
---

> 三个 agent 同时修 bug。三个都 commit 了。最终只剩一个的改动。

## 事故现场

一次并发修复任务：把三组 bug 分配给三个 CLI session，各自在同一个 repo 上工作。任务分配清晰，每个 session 负责不同的问题。

三个 session 几乎同时完成，各自执行 `git add -A && git commit && git push`。

结果：

- Session A：push 成功
- Session B：push 失败（`rejected — non-fast-forward`）
- Session C：push 失败，但没有报错处理逻辑，直接跳过了

Session B 和 C 的改动丢失。更糟的是，当时没人注意到——因为每个 session 各自报告"任务完成"，commit 确实创建了，只是没推上去。

## 根因

三层问题叠加：

**1. 没有 pull-before-push**

每个 session 在 push 之前没有 `git pull --rebase`。第一个 push 成功后，远端 HEAD 已经变了，后续 push 必然冲突。

**2. 文件粒度重叠**

虽然三组 bug 不同，但有些修复涉及相同文件（比如共享的配置文件、公共组件）。即使逻辑不冲突，git 也会标记为文件级冲突。

**3. 没有收尾检查**

每个 session 只看自己的 commit 是否创建成功，没有看 push 是否成功，更没有最终的 `git status` 全局检查。

## 修复方案

### 事前：按文件粒度拆任务

分配并发任务时，不只看"逻辑独立性"，还要看**文件独立性**：

```
❌ Agent A 修 Bug 1（涉及 config.ts + page.tsx）
   Agent B 修 Bug 2（涉及 config.ts + api.ts）
   → config.ts 冲突

✅ Agent A 修 Bug 1 + Bug 2 中涉及 config.ts 的部分
   Agent B 修 Bug 2 中只涉及 api.ts 的部分
   → 文件不重叠
```

原则：**同一个文件同一时间只有一个 agent 在改**。做不到的话，串行执行那部分。

### 事中：commit 规范

每组修复用独立的 commit message 前缀，方便事后追溯：

```
fix(session-a): resolve layout overflow in mobile view
fix(session-b): correct API error handling for timeout
fix(session-c): update config validation rules
```

前缀让 `git log` 可以快速判断哪个 session 的改动是否到位。

### 事后：status 收尾检查

所有 session 完成后，**必须有一个统一的收尾步骤**：

```bash
git status          # 是否有未提交的改动
git log --oneline -5  # 最近 5 条 commit 是否都在
git diff origin/main  # 本地和远端是否一致
```

这个步骤不能省。每个 session 自己报告的"完成"不可信——它只看到自己的视角。需要一个全局视角来确认。

## 更深的问题：git 不是为多 agent 设计的

Git 的核心假设是：**一个开发者在一个工作区，按时间顺序提交**。多人协作通过 branch + PR 解决，有人类 review 兜底。

多 agent 并发场景打破了这个假设：

- 没有 branch——都在同一个分支上直接改
- 没有 PR——commit 后直接 push
- 没有 review——没人检查冲突是否合理解决
- 时间间隔极短——几秒内多个 push，不像人类间隔几小时

这本质上是一个**并发控制问题**，和数据库的写冲突一个性质。

## 可选策略

根据团队规模和复杂度，几种递进方案：

| 策略 | 复杂度 | 适用场景 |
|------|--------|----------|
| 文件粒度拆分 | 低 | 任务间文件不重叠时 |
| 串行化 push（加锁） | 中 | 必须改同一文件时 |
| 每个 agent 独立分支 + 自动合并 | 高 | 大规模并发场景 |
| 队列化：完成后排队 push | 中 | 通用方案 |

目前用的是最简单的组合：**文件粒度拆分 + commit 前缀 + 收尾检查**。够用，不过度。

## 一句话总结

多个 agent 同时写一个 repo 不是异常——是常态。把冲突管理从"事后解决"前移到"拆任务时就预防"，然后用收尾检查兜底。
