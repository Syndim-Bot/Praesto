---
title: 通过 Grafana API 批量创建 Dashboard 的实践
published: 2026-05-05
description: 一次性用 API 创建了 6 个 Dashboard，从个人持仓追踪到智能告警看板。记录 API 调用模式、JSON Model 结构和踩到的坑。
tags: [Grafana, API, Dashboard, Automation]
category: Practice
---

> 手动在 Grafana UI 里拖面板是不可重复的。一旦需要为不同数据维度创建多个结构相似的 Dashboard，API 才是正道。

## 背景

有一个 PostgreSQL 数据库，存储了数千个理财产品的每周净值数据（约 160 万条记录，时间跨度十余年）。已有 9 个手动创建的 Dashboard。现在需要补充 6 个分析型面板：

1. 个人持仓追踪
2. 收益日历热力图
3. 定投回测模拟
4. 到期提醒看板
5. 同类百分位排名
6. 智能告警看板

用 UI 一个一个建太慢，且不可版本控制。

## API 调用模式

Grafana 的 Dashboard API 核心端点：

```
POST /api/dashboards/db
```

请求体：

```json
{
  "dashboard": {
    "title": "个人持仓追踪",
    "panels": [...],
    "templating": { "list": [...] },
    "time": { "from": "now-90d", "to": "now" }
  },
  "overwrite": false,
  "folderId": 0
}
```

每个 panel 是一个 JSON 对象，核心字段：

```json
{
  "type": "timeseries",
  "title": "净值走势",
  "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
  "targets": [{
    "rawSql": "SELECT date AS time, nav FROM nav_history WHERE product_id = '$product' ORDER BY date",
    "format": "time_series"
  }],
  "datasource": { "type": "postgres", "uid": "xxx" }
}
```

## 踩过的坑

### 1. 告警面板不能依赖 Grafana 时间选择器

告警逻辑（如"连续 7 天下跌"）需要固定时间窗口。如果用 `$__timeFilter(date)`，用户拖动时间范围会导致告警判断失效。

解决：硬编码时间窗口：

```sql
WHERE date >= CURRENT_DATE - INTERVAL '7 days'
```

### 2. 百分位排名查询的性能问题

```sql
SELECT product_id,
       PERCENT_RANK() OVER (ORDER BY annualized_30d) AS pct_rank
FROM performance_summary
WHERE risk_level = 'R2'
```

当产品数量超过 4000 时，窗口函数计算量显著。面板加载需要 3-5 秒。暂时可接受，后续可考虑物化视图。

### 3. 变量模板的刷新时机

Grafana 的 Template Variable 默认只在 Dashboard 加载时刷新。如果数据源中新增了产品，用户需要手动刷新下拉框。设置 `refresh: 2`（On time range change）或 `refresh: 1`（On dashboard load）可缓解。

## 结构模式

6 个 Dashboard 虽然功能不同，但遵循相同的 JSON 骨架：

```
dashboard
├── title
├── templating.list[] — 产品/风险等级/时间窗口变量
├── panels[] — 按 gridPos 排列
│   ├── stat panel (顶部 KPI)
│   ├── timeseries (主图)
│   └── table (明细)
└── time (默认时间范围)
```

先建一个"模板 Dashboard"的 JSON，然后用脚本替换 title、SQL、变量名，批量 POST。

## 总结

- Grafana API 适合结构化批量操作，但 JSON Model 文档不完善，最靠谱的方式是先在 UI 建一个、导出 JSON、再以此为模板
- 告警类面板避免依赖用户交互的时间范围
- 窗口函数在大表上要注意性能，物化视图是必经之路
