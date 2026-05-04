---
title: "Agent Skill 的结构化表征：从文本文件到三层模型"
published: 2026-05-05
description: 当前 agent 的 skill 以 Markdown 文本存在，机器难以高效检索和风险评估。一篇新论文提出 SSL（调度-结构-逻辑）三层表征，将 Skill Discovery 的 MRR 从 0.573 提升至 0.707。
tags: [Agent, Skill, Research, Representation]
category: Skills
---

> Skill 不只是一段 prompt。它有什么时候该触发（调度）、执行步骤是什么（结构）、为什么这样做（逻辑）三个独立维度。

## 问题

大多数 agent 系统中，skill 以自然语言文档形式存在——比如一个 `SKILL.md` 文件，描述了何时触发、怎么做、注意事项。这对人类可读，但对机器来说：

- **检索效率低** —— 要在几十个 skill 中找到最匹配的，全文相似度搜索效果有限
- **风险评估难** —— 某个 skill 是否可能执行破坏性操作？从文本中很难结构化判断
- **组合困难** —— 两个 skill 能否串联？输入输出是否兼容？文本描述不提供这个信息

## SSL 三层表征

论文 [arXiv:2604.24026] 提出的 SSL（Scheduling-Structural-Logical）模型将一个 skill 分解为三层：

### Scheduling Layer（调度层）

回答"什么时候触发"：

- 触发条件（关键词、意图匹配规则）
- 优先级和互斥关系
- 上下文前提（需要哪些环境条件）

### Structural Layer（结构层）

回答"怎么执行"：

- 步骤序列或 DAG
- 工具调用清单
- 输入/输出 schema
- 超时和重试策略

### Logical Layer（逻辑层）

回答"为什么这样做"：

- 设计意图
- 约束条件（安全边界、不可越过的红线）
- 失败回退策略

## 效果

在 Skill Discovery 任务（给定用户意图，从 skill 库中检索最匹配的 skill）上：

| 方法 | MRR |
|------|-----|
| 纯文本相似度 | 0.573 |
| SSL 结构化表征 | **0.707** |

在 Risk Assessment 任务（判断某 skill 的风险等级）上：

| 方法 | Macro F1 |
|------|----------|
| 文本分类 | 0.744 |
| SSL 表征 | **0.787** |

## 启发

对于维护 skill 体系的 agent 系统，可以考虑：

1. **为每个 skill 建立结构化 registry** —— 不只是 Markdown 描述，而是一个 JSON schema 同时包含 scheduling/structural/logical 三层信息
2. **调度层独立索引** —— 用于快速匹配，不需要加载完整 skill 内容
3. **逻辑层用于安全审计** —— 自动检测 skill 是否包含高风险操作（文件删除、外部 API 调用、金钱相关）

当 skill 数量超过 20 个，纯文本的 `description` 字段匹配已经不够用了。结构化是规模化的前提。

## 参考

- [From Skill Text to Skill Structure: The SSL Representation for Agent Skills](https://arxiv.org/abs/2604.24026)
