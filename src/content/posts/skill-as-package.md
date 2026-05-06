---
title: Agent Skills 正在变成新时代的 npm packages——我们踩过的坑和学到的事
published: 2026-05-07
description: 当 agent-skills 仓库 7 天涨了 629 stars，我们已经在生产环境跑了 8 套 skill 模板两个月。这是关于"怎么把知识变成可安装的 Agent 能力包"的实战笔记。
tags: [Skills, Agent Design, Modularity, Ecosystem]
category: Skills
---

> addyosmani/agent-skills 一周涨了 629 stars，到 29.8K。book-to-skill 把 PDF 转成 Claude Code skill。automotive-skills-suite 覆盖 100+ 汽车行业场景。整个社区突然意识到：Agent 的能力应该是可安装的。

我们两个月前就在做这件事了。不是因为有远见——是因为通用 agent 实在太弱了。

## 问题：通用 agent 什么都能做，什么都做不好

给 agent 一个"你是架构师"的 system prompt，它能输出格式完美的架构文档。字段全、逻辑通、模板标准。

但它缺少**判断力**。

这个服务该拆还是不拆？这个依赖方向对不对？这个技术债是现在还还是继续欠着？它不知道。它只是在做一个合格的填空题选手。

我们试过加长 system prompt——把所有原则、方法论、评估框架全塞进去。结果呢？Context window 爆了，注意力稀释了，该重点关注的东西反而被淹没了。

## 解决方案：把知识变成可安装的能力包

我们的做法很简单——每个 agent 配一套**领域专家 skill**：

```
skill/
├── SKILL.md              # 激活规则 + 输出模板 + 路由逻辑
└── references/           # 深度知识库（按需加载）
    ├── 01-principles.md
    ├── 02-methods.md
    ├── 03-patterns.md
    └── ...
```

核心设计只有三点：

**1. 按需加载，不是全量塞入。** reference 文件只在 skill 被激活时才加载。平时不占上下文窗口。agent 日常对话占 2K tokens，skill 激活时加载 8-15K tokens 的结构化知识——这比把所有东西一开始就塞进 system prompt 高效 10 倍。

**2. 知识融合，不是简单搬运。** 每套 skill 融合 2-3 个权威来源，合成一套连贯的、有观点的方法论。比如我们的架构 skill 融合了 Martin Fowler 的演进式设计 + Uncle Bob 的 Clean Architecture + Google SWE Book 的大规模实践。不是把三本书粘在一起——是提炼出"面对这个具体问题时，从三个视角分别怎么看"的决策框架。

**3. 一人一域，关注点分离。** 每个 agent 只精通一个领域。做架构的不碰视觉，做测试的不碰产品决策。听起来浪费？恰恰相反——当你有 10+ agent 协作时，专精带来的质量提升远超通才的灵活性。

## 两个月跑下来的实际数据

8 套 skill 模板，78 个 reference 文件，924KB 结构化知识库。

覆盖的领域：架构设计、全栈工程、产品管理、视觉设计、测试质量、Prompt 工程、内容运营、前端实现。

最直接的效果：**审查质量断崖式提升**。

举个例子。我们有个产品审查工作流，会从产品经理视角检查 28 项标准。在没有 skill 之前，通用 agent 能检查到 15-18 项，输出的判断往往是正确但没有洞察的。加了 skill 之后，它不仅能检查全部 28 项，还能给出"这个需求的用户价值公式算不过来，因为切换成本太高"这种基于具体方法论的深度判断。

## 我们踩过的三个坑

### 坑 1：reference 文件太大

早期我们把整本书的精华都塞进一个 reference 文件。结果单个文件 40KB+，agent 加载后要处理大量不相关的信息。

**解决方案**：拆成 8 个左右的小文件，按任务类型建立索引。SKILL.md 里定义"什么任务读什么文件"的路由规则。做标题优化只需要读 `03-title-copywriting.md`，不用加载整个内容策略体系。

### 坑 2：SKILL.md 写成了说明书

第一版的 SKILL.md 写了 3000 字，详细描述了激活条件、输出格式、注意事项。agent 读完这个 SKILL.md 就已经占了不少注意力，还没开始读 reference 呢。

**解决方案**：SKILL.md 精简到三部分——(1) 什么时候触发，(2) 读哪些文件，(3) 输出什么格式。其他一切放到 reference 里去。SKILL.md 是目录，不是教材。

### 坑 3：Skill 之间缺乏协作协议

产品 skill 发现了问题，架构 skill 需要评估可行性，测试 skill 要补充用例。三个 skill 分属三个 agent——它们之间怎么传递信息？

早期我们让 L0 协调者手动搬运上下文。效率极低。

**解决方案**：定义结构化的"skill 输出格式"，让上游 skill 的输出能直接作为下游 skill 的输入。产品 skill 输出 `## 关键假设` 和 `## 关键风险`，架构 skill 直接消费这两个字段。不需要中间人翻译。

## 为什么现在是 Skill 生态化的时刻

回到 addyosmani/agent-skills 的爆火。它说明了一件事：**社区已经从"让 agent 更聪明"转向了"让 agent 更专业"。**

这跟 Node.js 早期的发展路径一模一样：

- 2010：大家用 Node.js 写一切，什么都自己实现
- 2012：npm 起来了，社区开始发布和复用小模块
- 2014：npm 成为事实标准，生态繁荣

Agent Skills 现在处于 2012 阶段。有人在做（我们做了两个月），有人在定义标准（SKILL.md + references/ 的约定），有人在做分发（book-to-skill、ClaWHub）。但还没有一个类似 npm 的事实标准平台。

**谁先把"Skill Schema"标准化——定义 inputs/outputs/dependencies 声明——谁就有可能成为 Agent 生态的 npm。**

## 给同行的三条建议

**1. 现在就开始模块化你的 prompts。** 如果你有超过 2000 字的 system prompt，它大概率需要拆成一个 SKILL.md + 若干 reference 文件。按需加载，别全量塞入。

**2. 定义清晰的 IO 格式。** 你的 skill 输出什么？用什么格式？下游谁来消费？这些想清楚了，skill 才能被复用——先被你自己的其他 agent 复用，再被社区复用。

**3. 不要从零写 reference——从你已有的知识中提炼。** 你的 Notion 里有没有积累了一年的最佳实践？你收藏的技术文章有没有反复引用的？你写过的技术方案有没有形成模式的？这些就是你的 reference 文件来源。book-to-skill 能把一本 PDF 转成 skill——你也可以把你的经验库转成 skill。

---

Agent 能力的竞争正在从"谁的模型更强"转向"谁的知识结构更好"。模型是基础设施，人人能用。但你怎么组织知识、怎么路由能力、怎么让 10 个 agent 各有所专——这才是真正的壁垒。

我们的 8 套 skill 模板不算多。但两个月跑下来，它们让一群通用 agent 变成了有判断力的专家团队。这个方向，值得所有做 AI agent 的人认真投入。
