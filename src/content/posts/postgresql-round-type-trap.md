---
title: "PostgreSQL ROUND() 的类型陷阱：double precision 不能直接取整"
published: 2026-05-05
description: 在 Grafana 面板中写 SQL 时遇到 ROUND(column, 2) 报错——PostgreSQL 的 ROUND 不接受 double precision 参数，必须先 cast 到 numeric。一个五分钟的坑。
tags: [PostgreSQL, SQL, Grafana, Type System]
category: Field Notes
---

> 报错信息："function round(double precision, integer) does not exist"。看起来像 bug，其实是 PostgreSQL 类型系统的设计决策。

## 场景

用 Grafana 连接 PostgreSQL 数据源，为理财产品净值数据构建分析面板。其中一个 panel 需要计算年化收益率并保留两位小数：

```sql
SELECT ROUND(annualized_return, 2) FROM performance_view;
```

直接报错。

## 原因

PostgreSQL 有两个 `ROUND()` 函数签名：

- `ROUND(numeric)` → numeric（四舍五入到整数）
- `ROUND(numeric, integer)` → numeric（保留 N 位小数）

注意：**没有** `ROUND(double precision, integer)` 这个签名。

`double precision`（即 `float8`）是浮点数，PostgreSQL 设计上不允许对浮点数做精确的小数位控制——因为浮点数本身就是近似值，"保留两位小数"在语义上不自洽。

## 修复

Cast 到 `numeric` 再取整：

```sql
SELECT ROUND(annualized_return::numeric, 2) FROM performance_view;
```

或者在复杂表达式中：

```sql
SELECT ROUND(CAST(
  (latest_nav - prev_nav) / prev_nav * 365.0 / holding_days * 100
  AS numeric), 2) AS annualized_pct
FROM nav_history;
```

## 为什么这个坑反复出现

1. **MySQL 不区分** —— MySQL 的 `ROUND()` 接受任何数值类型，用惯了 MySQL 的人不会意识到这是个问题
2. **隐式转换的假象** —— PostgreSQL 在很多场景下会隐式转换 `float8 → numeric`，但 `ROUND()` 恰好不做这个隐式转换
3. **Grafana 变量模板** —— Grafana 的 `$__timeFilter` 和自动列检测经常让你忘记底层列的真实类型

## 记住的规则

在 PostgreSQL 中，凡是需要 `ROUND(x, n)` 的地方：

```sql
ROUND(x::numeric, n)
```

无脑加 `::numeric`，不多想。
