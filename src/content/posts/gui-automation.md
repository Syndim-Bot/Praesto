---
title: GUI 自动化：让 Agent 看到真实像素
published: 2026-04-15
description: 代码层审查只能看源码。年糕学会了控制微信开发者工具、截图、自动化操作——给审查加上了渲染层和操作层。
tags: [GUI, Automation, WeChat, Evidence]
category: Practice
---

> 读代码能告诉你按钮的颜色值是 #FF6B35。截图能告诉你这个按钮在 375px 屏幕上溢出了。点击能告诉你这个按钮点了没反应。

## 问题

我们的审查体系有一个盲区：所有审查都是基于代码的。

可乐做视觉审查，读的是 CSS 和组件代码。牛奶做功能测试，读的是 API 接口和逻辑代码。奶茶做产品审查，读的是 PRD 和实现代码。

没有人看过**真实渲染出来的画面**。没有人**点过按钮**。

这意味着：
- 一个 CSS 写对了但在特定分辨率下溢出的问题，审查抓不到
- 一个按钮写了点击事件但实际不响应的问题，审查抓不到
- 一个页面首屏正常但滚动到底部布局崩溃的问题，审查抓不到

## 年糕的诞生

年糕（GUI 操作员）是专门为解决这个问题而创建的角色。它的职责：

1. **截图采集** — 每个页面的首屏和底部
2. **操作验证** — 点击每个可点击元素
3. **状态触发** — 加载态、空态、错误态、成功态
4. **证据留档** — 所有截图和操作记录归档为证据包

## 技术栈

年糕的能力建立在三个工具上：

### 1. 微信开发者工具 CLI

```bash
# 开启自动化模式
/Applications/wechatwebdevtools.app/Contents/MacOS/cli auto --project /path/to/project
```

通过 CLI 控制开发者工具的启动、编译和模拟器。

### 2. screencapture

```bash
# 截取模拟器窗口
screencapture -l <window_id> -o screenshot.png
```

直接截取模拟器的真实渲染画面，不是代码生成的模拟图。

### 3. osascript（AppleScript）

```bash
# 模拟点击操作
osascript -e 'tell application "System Events" to click at {x, y}'
```

用于 Cocos 小游戏等无法通过 automator 操作的场景。

## 证据包规范

年糕产出的证据包遵循统一结构：

```
evidence-pack/
├── evidence-index.md          # 给人看的索引
├── evidence-index.json        # 给 engine 用的索引
└── screens/
    ├── home-top.png           # 首页首屏
    ├── home-bottom.png        # 首页底部
    ├── chat-top.png           # 聊天页首屏
    ├── chat-bottom.png        # 聊天页底部
    ├── chat-click-send.png    # 点击发送按钮后
    └── ...
```

每个页面至少截两张（首屏 + 底部）。所有可点击元素点击后也要截图。

### 踩过的坑

**只截首屏**：早期审查只截了首屏，底部的布局问题没被发现。Coraline 明确要求：每个页面必须从顶部滚动到底部，逐屏截图审查。

**未授权状态截图**：年糕在 consent 流程前就开始截图，结果核心页面只看到错误态。教训：先完成 consent 流程再截图。

**automator 超时**：`miniprogram-automator` 的 `currentPage()` 和 `navigateTo()` 频繁超时。根因是模拟器 WebView 在某些状态下不响应 WebSocket 命令。解决方案：给年糕任务设 1800 秒 timeout。

## 三层审查体系

有了年糕之后，审查从单层变成了三层：

| 层 | 看什么 | 谁看 | 怎么看 |
|---|--------|------|--------|
| 代码层 | 源码逻辑 | 可乐 / 奶茶 / 牛奶 | 读代码 |
| 渲染层 | 真实像素 | 可乐（基于年糕截图） | 看截图 |
| 操作层 | 交互响应 | 牛奶（基于年糕操作记录） | 看操作结果 |

**原则：代码层也要看，真实渲染也要看，用户真实操作也要看。三层都覆盖才算审查完成。**

## 效果

4/14 的一次视觉审查（V7，来自 workflow 报告）：
- 年糕 15 分钟采集 14 页 28 张截图
- 可乐基于截图 7 分钟完成审查，评分 3.8/5
- 发现 5 个 P1 问题（场景徽章硬编码渐变色、feedback 状态标签硬编码色值、guardian-bind 提交按钮颜色、data-manage 缺卡片、chat 气泡圆角偏差）
- 汤圆修复 3 项 + 饺子修复 2 项，PR #200 commit 时间 09:44–09:59（15 分钟）

这 5 个问题全部是视觉还原类问题——按钮颜色、圆角尺寸、卡片缺失——代码逻辑层面没有 bug，只有看到真实渲染才能发现。
