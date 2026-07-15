---
name: ref-data-ingestion-module
description: 模块一"数据接入与达成计算"的关键决策与待确认依赖，供后续设计阶段参考
metadata:
  type: reference
---

需求规格已写入：`docs/requirements/数据接入与达成计算.md`

**Why:** 该模块是整个平台的数据底座，其设计决策影响AI预测和执行追踪两个下游模块。

**How to apply:** architect设计该模块时先读需求规格，特别注意设计期依赖R2。

## 关键决策
- 数据接入方式：定时拉取（非实时），外部系统API对接
- 技术栈：MySQL（业务状态）+ ClickHouse（历史分析）+ Kafka（事件总线）
- 历史数据要求：至少12个月（建议存储24个月）
- 数据可用性分级：FULL / PARTIAL / LIMITED 三级

## 设计期依赖（R2）
ST与SO（Dealer维度）在PDF 7.5中均指向DCR IMEI Sales Query New同一菜单。
精确的API差异参数需在接口契约阶段与DCR团队确认。
需求层已标注⚠️，不阻塞需求但阻塞实现。

## DCR三个数据来源（菜单名，非API端点）
1. IMEI Sales Query New → SO/Dealer、ST/Dealer（待区分）、SO/门店
2. IMEI Purchase Query New → SI/Dealer、SI/门店
3. Shop Sales Query New → SO/Promoter、SO/Supervisor、SO/区域经理

## 无此规则场景
ST/门店/人员 → 返回 RULE_NOT_DEFINED，不计算

## Kafka事件清单（本模块发布）
- `data_source_anomaly`：数据源异常（延迟>24h或连续失败）
- `achievement_updated`：达成计算完成，下游订阅消费
