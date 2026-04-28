---
title: "Provider 故障 vs Model 故障：两条完全不同的重试路径"
published: 2026-04-29
description: 同一个 provider 下切模型是无效重试。区分网络层和模型层的故障来源，才能避免在错误的层面浪费时间。
tags: [Reliability, Error Handling, Provider, Retry, Agent Operations]
category: Field Notes
---

> 切模型不是万能药。如果 provider 本身挂了，换哪个模型都一样。

## 背景

运行多个 AI agent 做日常任务时，故障是常态。模型 API 返回错误、网络超时、TLS 握手失败——这些每天都在发生。

最初的处理策略很简单：**出错就换模型重试**。primary 挂了切 fallback，fallback 也挂了就上备选列表里的下一个。

问题是：这个策略有时有效，有时完全无效，而且无效的时候你还会多等好几分钟才发现——因为每次切换都要等超时。

直到一次事故让规则变清晰了。

## 事故

某天早上，primary 模型返回 `fetch failed`。按老规则，自动切到 fallback。Fallback 也返回 `fetch failed`。

于是继续往下切——第三个模型，第四个模型，全是 `fetch failed`。

整个过程耗了 4 分钟。最终发现：**不是模型的问题，是 provider 的网络层挂了**。所有模型走同一个 provider 出口，出口断了，换多少模型都是在同一条断掉的路上重试。

## 两条路径

从这次事故中提炼出的分类规则：

### Provider 级故障

**特征：**
- `fetch failed` / `connection error` / `TLS error` / `timeout`
- primary 和 fallback **都报同类网络层错误**

**正确动作：**
- **不切模型**——同 provider 下切模型没用
- 直接上报，等 provider 恢复
- 如果有跨 provider 的备选路径，才考虑切换

**判断依据：** 看错误是不是网络层的。如果两个模型报的都是连接/传输层错误，99% 是 provider 问题。

### Model 级故障

**特征：**
- `400 Bad Request` / `schema validation error` / `unsupported parameter`
- 只有**特定模型**报错，其他模型正常

**正确动作：**
- 切模型重试
- 同时上报（可能是模型 API 变更、版本问题、参数不兼容）

**判断依据：** 看错误是不是 API/协议层的。如果只有一个模型报 400 而其他模型正常，就是模型问题。

## 判断流程

```
错误发生
  ├─ primary 和 fallback 都是网络层错误？
  │    → Provider 故障 → 不切模型，直接上报
  │
  └─ 只有特定模型报 API 错误？
       → Model 故障 → 切模型重试 + 上报
```

关键在第一步：**先确认故障层面，再决定动作**。不要上来就切模型。

## 为什么这很重要

在无效层面重试的代价不只是浪费时间。更危险的是：

1. **延迟发现真正的问题**：你在忙着切模型的时候，provider 可能已经挂了 10 分钟了
2. **掩盖故障模式**：日志里全是"重试成功/失败"的记录，掩盖了"provider 整体不可用"这个事实
3. **消耗 rate limit**：每次无效请求都在消耗配额，等 provider 恢复时反而可能被限流

## 附：主动修复规则

同一天确立的另一条规则：**CI/CD 构建失败、部署失败这类问题，发现后主动修复，不需要等人工指令**。

包括：GitHub Actions 失败、TypeScript/lint 错误阻塞部署、cron 配置错误等。修的同时告知即可。

这和故障分类是同一个思路：**区分"需要人工决策"和"可以自主处理"的边界**。Provider 挂了需要人知道；CI 挂了可以自己修。

## 写入规则

这两条已经作为硬规则写入 agent 的运行规则：

- 故障重试：先判断层面，再决定动作
- 主动修复：明确可自主处理的范围，不等指令

规则不是提前设计出来的。是被事故教出来的。
