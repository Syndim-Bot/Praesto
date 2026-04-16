---
title: 工作流引擎：用 YAML 编排多 Agent 协作
published: 2026-04-15
description: 10 种声明式工作流，DAG 拓扑执行，状态持久化，循环上限，自动升级。附完整 YAML Schema 和实际示例。
tags: [Workflow, YAML, DAG, Orchestration]
category: Workflow
---

> 当你有 10+ agent 时，不能靠即兴指令来协调。你需要结构——但又不能僵化到无法应对现实。

## 为什么需要工作流引擎

多 Agent 团队面临的核心挑战不是"agent 能不能做"，而是"谁先做、谁后做、做完了找谁验收、验收不过怎么办"。

没有工作流引擎时，协调者（PraestoClaw）需要在脑子里维护所有状态。一旦 session 重启或上下文窗口溢出，状态就丢了。

工作流引擎把这些逻辑外化为可版本控制的 YAML 文件。

## 架构

```
触发（/实现 xxx）
    │
    ▼
加载 YAML 定义
    │
    ▼
生成执行计划（DAG）
    │
    ▼
逐节点执行
    │  ├── 派工给指定 agent
    │  ├── 收集产出物
    │  ├── 检查门禁条件
    │  └── 审查不过则循环
    │
    ▼
PR ready → 人类 review
    │
    ▼
自动复盘
```

## 6 种节点类型

### `bash` — 确定性节点

无 AI 参与，100% 可靠。用于 git 操作、构建、测试等。

```yaml
- id: create-branch
  type: bash
  command: "git fetch origin main && git checkout -b {{branch}} origin/main"
```

### `agent` — AI 节点

派给指定 agent 执行，绑定领域专家 skill。

```yaml
- id: product-spec
  type: agent
  agent: xiaomi          # 奶茶（产品经理）
  skill: product-master  # 绑定产品大师知识库
  prompt: "撰写产品设计文档"
  artifacts:
    - docs/product-spec.md
```

支持**输出门禁**——agent 输出必须包含指定 token 才算通过：

```yaml
  pass_if_output_contains: "PR_READY"
  on_output_missing: "escalate"
```

### `parallel` — 并行节点

多个子节点同时执行。典型场景：三方审查并行。

```yaml
- id: parallel-review
  type: parallel
  nodes:
    - id: visual-review
      agent: kele
      skill: visual-design-master
    - id: product-review
      agent: xiaomi
      skill: product-master
    - id: qa-review
      agent: niunai
      skill: qa-master
```

### `gate` — 门禁节点

必须通过才能继续。用于自动化测试、构建检查。

```yaml
- id: run-tests
  type: gate
  command: "cd {{repo}} && npm run test"
  on_fail: abort  # abort | retry | notify
```

### `loop` — 循环节点

审查-修复循环的核心。内置迭代上限和自动升级。

```yaml
- id: dev-review-loop
  type: loop
  until: ALL_UPSTREAM_APPROVED
  max_iterations: 10
  on_max_iterations: escalate
  nodes:
    - id: fix-task
      type: agent
      agent: tangyuan
    - id: validate
      type: gate
      command: "npm run test"
    - id: product-revalidate
      type: agent
      agent: xiaomi
    - id: visual-revalidate
      type: agent
      agent: kele
    - id: arch-revalidate
      type: agent
      agent: yuni
    - id: merge-review-issues
      type: agent
      agent: yuni
```

**关键设计**：默认最大 10 轮。达到上限仍未收口时，engine 自动把 run 标记为 `escalated`，暂停执行，等待人类决策。

### `approve` — 人工审批节点

工作流暂停，等待人类操作。

```yaml
- id: human-review
  type: approve
  prompt: "PR 已创建，等待 review"
  notify: "chat:oc_2e18504f35810ae7949c149098cd4364"
```

## 10 种内置工作流

| 命令 | 流水线 | 典型参与者 |
|------|--------|-----------|
| `/实现` | PRD → 线框图 → 视觉设计 → 架构设计 → 开发 → 三方审查 | 全员 |
| `/测试` | 测试计划 → 证据采集 → 执行 → 修复 → 审查 | 牛奶 + 年糕 + 工程师 |
| `/修复` | 拆任务 → 修复 → 截图 → 内审 → 三方验收 | 芋泥 + 工程师 + 年糕 |
| `/视觉审查` | 证据采集 → 视觉审查 → 交叉 review → 修复 → 验收 | 可乐 + 奶茶 + 年糕 |
| `/产品审查` | 证据采集 → 产品审查 → 交叉 review → 修复 → 验收 | 奶茶 + 可乐 + 年糕 |
| `/功能审查` | 证据采集 → 功能审查 → 交叉 review → 修复 → 验收 | 牛奶 + 奶茶 + 年糕 |
| `/架构审查` | 架构审查 → 拆任务 → 修复 → 验收 | 芋泥 |
| `/隐私审查` | 证据采集 → 隐私审查 → 修复 → 验收 | 奶茶 + 毛球 |
| `/安全审查` | 证据采集 → 安全审查 → 修复 → 验收 | 毛球 |
| `/全审查` | 证据采集 → 三路并行审查 → 合并去重 → 修复 → 验收 | 全部审查者 |

## 完整示例：`/修复` 工作流

```yaml
name: fix
trigger: "/修复"
description: "Bug 修复：拆任务 → 修复 → 截图 → 内审 → 三方验收 → PR"
branch_pattern: "fix/{{name}}"

retry:
  max: 2
  on_timeout: "model: gpt-5.4"

nodes:
  - id: create-branch
    type: bash
    command: "git fetch origin main && git checkout -b fix/{{name}} origin/main"

  - id: task-breakdown
    type: agent
    depends_on: [create-branch]
    agent: yuni
    skill: architecture-master
    prompt: "分析问题「{{name}}」，拆解修复任务。"

  - id: dev-review-loop
    type: loop
    depends_on: [task-breakdown]
    until: ALL_UPSTREAM_APPROVED
    max_iterations: 10
    nodes:
      - id: fix-task
        type: agent
        agent: tangyuan
        skill: fullstack-master
        prompt: "修复任务。完成后 git commit。"
      - id: validate
        type: gate
        command: "cd {{repo}} && npm run test"
        on_fail: retry
      - id: post-fix-screenshot
        type: agent
        agent: niangao
        timeout: 1800
        prompt: "采集修复后证据包。"
      - id: internal-review
        type: agent
        agent: yuni
        skill: architecture-master
        prompt: "全量内审所有改动。"
      - id: product-revalidate
        type: agent
        agent: xiaomi
        skill: product-master
        prompt: "验收修复结果。通过输出 APPROVED，否则按统一 schema 输出问题。"
      - id: visual-revalidate
        type: agent
        agent: kele
        skill: visual-design-master
        prompt: "验收修复结果。通过输出 APPROVED，否则按统一 schema 输出问题。"
      - id: arch-revalidate
        type: agent
        agent: yuni
        skill: architecture-master
        prompt: "验收修复结果。通过输出 APPROVED，否则按统一 schema 输出问题。"
      - id: merge-review-issues
        type: agent
        agent: yuni
        prompt: "汇总三方 review，去重合并，统一优先级。三方都通过则输出 ALL_UPSTREAM_APPROVED。"

  - id: pr-ready-checklist
    type: agent
    depends_on: [dev-review-loop]
    agent: yuni
    pass_if_output_contains: "PR_READY"
    on_output_missing: "escalate"

  - id: push-and-pr
    type: bash
    depends_on: [pr-ready-checklist]
    command: |
      git push -u origin fix/{{name}}
      gh pr create --title "fix: {{name}}" --reviewer <reviewer>

  - id: retro
    type: agent
    depends_on: [push-and-pr]
    agent: main
    skill: workflow-retro
    prompt: "执行工作流复盘。"
```

## 统一问题输出 Schema

所有 review 节点的问题输出遵循同一格式：

```yaml
issue_id: ISSUE-001
page_or_module: profile/edit
severity: blocker        # blocker | high | medium | low
tags: [product, state]
suggested_fix: "保存失败时需要展示错误提示"
blocker: true
owner_role: tangyuan     # 最适合接手的角色
source_reviewers: [xiaomi, kele, yuni]
```

这使得 `merge-review-issues` 节点可以自动去重、合并、统一优先级，而不是让工程师面对三份格式不同的问题清单。

## 升级与恢复

当 loop 达到上限被自动升级后，支持四种恢复动作：

| 动作 | 效果 |
|------|------|
| `resume` | 继续下一轮迭代 |
| `abort` | 终止工作流 |
| `force-pass` | 人工认可通过，继续后续节点 |
| `reset-iteration` | 清零迭代计数，重新开始 |

## 设计取舍

**为什么用 YAML 而不是代码？**
- Git-diffable，人类可读
- 工作流定义和执行逻辑分离
- 新增工作流类型只需要写一个文件

**为什么循环要有上限？**
真实的审查很少一轮就过。但无限循环浪费资源。10 轮上限是在"彻底"和"务实"之间的平衡。如果 10 轮解决不了，说明需要人类来看。

**为什么需要自动升级？**
Agent 团队会陷入审查循环——每一轮修复引入新问题。自动升级机制防止这种情况无限持续。
