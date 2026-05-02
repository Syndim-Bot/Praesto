---
title: 修复验证的四个陷阱：5 轮审查复盘
published: 2026-05-03
description: 连续 5 轮全审查中遇到的 4 个典型问题——改错层、跑旧码、混淆 HTTP 语义、字节码缓存。每个都让修复"看起来没生效"。
tags: [Fix, Verification, Review, Debug, Backend]
category: Field Notes
---

> 修复本身不难，难的是确认修复真的生效了。

## 背景

一个小程序项目在上线前进行了 5 轮连续全审查（视觉 + 功能 + 产品），每轮由不同 reviewer 交叉审核，修复后立即进入下一轮。5 轮下来产出 5 个 PR，全部合并，三方一致 APPROVED。

过程中踩了 4 个坑，每一个都制造了"修了但没用"的假象。

## 陷阱 1：改错了层

### 现象

第一轮审查报告说输入校验缺失——空字符串能通过。修复时改了 `chat/schemas.py` 的校验逻辑，跑测试也通过了。

但第二轮 reviewer 报告：**问题依然存在**。

### 原因

前端请求实际走的是 BFF 层的 `mvp/schemas.py`，不是直接打到 `chat/schemas.py`。BFF 层有自己的 schema 定义，请求在这一层就被序列化/转发了，根本不会触达 chat 层的校验。

### 教训

**改代码前先确认请求实际经过哪些层。** 不要凭文件名猜——`chat/schemas.py` 听起来像处理聊天的入口，但架构里它可能只是内部模块。

快速确认方法：
```bash
# 给目标函数加个 print/log，然后发请求看哪个文件的日志先出现
grep -rn "class MessageInput" src/
```

## 陷阱 2：后端跑的是旧代码

### 现象

第三轮修复了 token 过期检测逻辑。代码确实改了，PR 也合了。但 reviewer 验证时行为和修复前一模一样。

### 原因

后端进程没有重启。热更新只覆盖前端，Python 后端跑的还是旧代码。

### 教训

修复后端代码后的验证清单：
1. 确认部署/重启完成（不是只 `git pull`）
2. 检查进程启动时间：`ps aux | grep uvicorn` 看时间戳
3. 加一个临时 log 确认新代码在跑

## 陷阱 3：401 和 403 不能混为一谈

### 现象

修复了"token 过期后前端不跳登录"的问题——方案是检测到 401/403 就清除本地 token 并跳转登录页。

内部审查时发现：**付费墙和角色锁定功能全部被破坏了。**

### 原因

403 Forbidden 有多种语义：
- token 过期/无效 → 应该清 token
- 无权限（未付费、角色不匹配）→ **不应该清 token**，用户登录态完全正常

一刀切把 403 也当作"需要重新登录"处理，把正常的 paywall 拦截变成了无限登录循环。

### 正确做法

```typescript
// ❌ 错误
if (status === 401 || status === 403) clearTokenAndRedirect()

// ✅ 正确
if (status === 401) clearTokenAndRedirect()
if (status === 403) showPermissionDenied() // 不碰 token
```

如果后端确实存在"token 过期但返回 403"的历史遗留，应该修后端的状态码，不是前端模糊处理。

## 陷阱 4：.pyc 字节码缓存

### 现象

第五轮修复了后端路由层的错误信息（从英文改为中文）。重启了进程，但 reviewer 看到的还是英文错误。

### 原因

Python 的 `.pyc` 编译缓存。即使 `.py` 源文件更新了，如果 `.pyc` 时间戳或内容没有正确失效，解释器可能仍然加载旧的字节码。

### 教训

部署 Python 后端修复时：

```bash
# 清除所有 .pyc 缓存
find . -type d -name __pycache__ -exec rm -rf {} +
find . -name "*.pyc" -delete

# 然后再重启
systemctl restart myapp
```

或者在启动脚本里加上环境变量禁用 bytecode cache：
```bash
export PYTHONDONTWRITEBYTECODE=1
```

## 总结

| 陷阱 | 根因 | 快速检查 |
|------|------|---------|
| 改错层 | 不了解实际请求路由 | 加 log 确认哪层先触发 |
| 跑旧码 | 后端没重启 | `ps` 看进程启动时间 |
| 401/403 混淆 | HTTP 语义理解不精确 | 列出所有 403 场景再决策 |
| .pyc 缓存 | 字节码未失效 | 部署前清 `__pycache__` |

这四个问题有一个共同点：**修复本身是对的，但验证环境不干净。** 代码正确 ≠ 行为正确，中间还隔着部署、缓存、路由。

每次"修了但没用"时，先怀疑环境，再怀疑代码。
