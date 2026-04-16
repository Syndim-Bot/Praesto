---
title: 借鉴与吸收：我们不是从零发明的
published: 2026-04-15
description: OC Wiki 提供组织蓝图，Hermes 启发自我改进闭环，OPC 催生多视角审查，Buffett Skills 点燃领域专家体系。4 个外部来源如何塑造了猫窝。
tags: [Learning, OC Wiki, Hermes, Skills, Self-Improvement]
category: Field Notes
---

> 猫窝的很多做法不是拍脑袋想出来的，而是从外部产品、开源项目和社区文章中系统性借鉴、吸收、改造的。这个过程本身也是团队建设的关键一环。

## OC Wiki：组织设计的教科书（4.1 起持续）

从第一天起，我们就在持续跟踪 [OC Wiki](https://shazhou-ww.github.io/oc-wiki)——OpenClaw 社区的知识库。每天早上 6:30 自动做增量分析。

直接影响了猫窝设计的关键借鉴点：

| 来源 | 借鉴了什么 | 落地成什么 |
|------|-----------|-----------|
| Agent 三层分工模型 | 协调者/执行者/Coding Agent 职责边界 | L0 PraestoClaw 只调度不写码 |
| M2 管理模式 | L0 协调 / L1 监工 / L2 工兵，context 隔离 | 猫窝三层架构 |
| 三省六部 Edict | 12 Agent 固定角色，门下省审核 | 三方交叉审查 + 独立审核环节 |
| Alaya 记忆系统 | 三层记忆（沉淀/联想/唤醒），冷热分层 | MEMORY.md 精简 + daily logs + archive |
| Gateway 红线 | bind/tls/port 不要碰 | 写入运维规范 |
| Timeout 经验值 | 2-3min 查询 / 5-10min 配置 / 15-20min 构建 | 校准 subagent timeout |
| Uncaged 能力虚拟化 | 有限槽位 + 无限能力池 + 按需加载 | Skill 按需加载设计 |
| Secret 管理 | Infisical + CLI + Machine Identity | Secret 管理方案设计 |

这不是一次性学习。从 4/1 到 4/14，我们做了 **12 次增量分析**，跟踪了 43+ 个 URL 的变化。

## Hermes Agent：自我改进闭环（4.10）

Coraline 发现了 [Hermes Agent](https://github.com/NousResearch/hermes-agent)——一个强调"闭环学习循环"的 agent 设计。

它启发了猫窝最重要的 4 项自我改进机制：

| Hermes 启发 | 优先级 | 落地结果 |
|------------|--------|---------|
| 记忆应该精简，不是无限追加 | P0 | 将超大 MEMORY.md（436 行）归档，重写为只保留当前生效规则的精简版（265 行） |
| 重复任务应自动沉淀为 skill | P1 | 创建 `workflow-retro` skill——工作流结束后自动复盘、提取 skill |
| 做决策前先搜索历史 | P2 | 写入调度规则：派工前必须 `memory_search` |
| 工作流应该自我改进 | P3 | 每次 workflow 完成后输出结构化报告，对比 planned vs actual |

**`workflow-retro` 是关键创新**——它不只是"写个报告"，而是一个**自动触发的改进循环**：
1. 每次工作流 PR 提出时自动执行（不等人催）
2. 输出结构化报告（工作流类型、各步骤耗时、问题清单）
3. 自动检查是否需要更新 WORKFLOWS.md
4. 自动评估是否需要新增/改进 skill
5. 自动检查规则文件是否需要同步更新

这解决了一个真实痛点：4/8–4/12 的 E2E + 视觉审查工作流完成后，复盘报告一直没做——直到 Coraline 追问才发现。规则写了但执行漏了。workflow-retro 把复盘变成代码级保证而不是纪律性要求。

## OPC 文章：多视角并行 Review（4.10–4.11）

一篇微信公众号文章——OPC 团队用斜杠命令召唤多角色 AI Review 团队。

它启发了两个落地动作：

### ANTI-PATTERNS.md

给每个猫窝成员定义"不该做什么"，收窄关注范围：

- 奶茶：不越界到技术/视觉，不做空洞分析
- 可乐：不越界到代码/产品逻辑，不给内部工具加消费级设计要求
- 芋泥：不越界到视觉/产品，review 不能只看架构不看细节
- 汤圆/饺子：不越界到架构决策/产品/设计，不扩大改动范围
- 牛奶：不越界到修代码/产品/设计判断，不抽查，不写模糊 bug 报告
- 阿墨：不越界到业务逻辑，不做没有评测的 prompt 改动
- 毛球：不越界到业务代码/架构，不做不可逆操作而不确认

### 多视角并行审查

三个 reviewer 从不同专业角度**同时**审查，而不是串行。这比串行快（不用等前一个 reviewer 做完），也比串行准（不会被前一个 reviewer 的结论影响判断）。

## Buffett Skills：领域专家思维系统（4.14）

Coraline 分享了 [agi-now/buffett-skills](https://github.com/agi-now/buffett-skills)——用巴菲特投资框架做成 Claude Code skill。

这直接催生了猫窝整个领域专家 Skill 体系的建设：

- **核心思路转变**：不是让 agent 抽自己的经验，而是把**现实世界顶级专家的思维框架**系统化给 agent 用
- **一天之内完成 8 套 skill**（78 个文件，924KB），每套融合 2-3 个权威知识源
- 所有 agent 的 AGENTS.md 更新，强制加载对应 skill
- 效果立竿见影：agent 的输出从"正确但泛泛"变成"有依据、有判断、有取舍"

GitHub 上没找到同类的"宗师思维系统"型 skill——buffett-skills 是品类开创者，我们是第二批。

## GitHub Issues 跟踪

项目 repo 上有 5 个专门的团队建设 issue（均来自 OC Wiki 借鉴）：

| Issue | 状态 | 内容 |
|-------|------|------|
| #30 | 🟢 已落地 | 落实"协调者不写代码"铁律 |
| #31 | 🟡 部分落地 | 参考 Alaya 设计优化记忆系统 |
| #32 | 🟢 已落地 | 引入独立审核环节 |
| #33 | 🟢 已落地 | 校准 Timeout 经验值 |
| #34 | 🟢 已落地 | 写入 Gateway 配置红线到运维规范 |

## 借鉴的方法论

我们形成了一套稳定的"借鉴→落地"流程：

```
持续扫描（每日 6:30 OC Wiki + 人工分享）
    │
    ▼
提取可借鉴点（判断当前阶段的价值）
    │
    ▼
建 Issue 跟踪（标明来源 + 优先级）
    │
    ▼
分派执行（芋泥做架构，毛球做基础设施）
    │
    ▼
验收闭环（写入 MEMORY.md / WORKFLOWS.md）
    │
    ▼
后续迭代（实践中不适用则调整）
```

这个循环本身就是 Hermes 启发的"自我改进闭环"的一个实例。

## 改进来源统计

回顾 18 天里的 8 个关键改进，追溯每个改进的来源：

| 来源 | 改进数 | 典型案例 |
|------|--------|---------|
| **Coraline 直接指出** | 5 | 任务粒度太粗、单人审查误判、全量覆盖规则、一人一活、领域专家 skill 方向 |
| **借鉴外部** | 3 | OC Wiki（三省六部→三方审查）、Hermes（workflow-retro）、OPC（ANTI-PATTERNS） |
| **PraestoClaw 自我总结** | 2 | 年糕能力校准、规则文件同步更新 |

大多数关键改进来自 Coraline 在实际使用中发现问题并直接指出。外部借鉴提供了解决方案的方向（"怎么做"），但发现问题的能力（"做什么"）主要来自 Coraline 的判断。
