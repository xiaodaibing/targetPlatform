---
name: ref-execution-tracking-module
description: 模块四执行追踪与复盘需求规格要点：异常类型/阈值、分级规则、实体清单、开放问题
metadata:
  type: project
---

需求规格已写入 `docs/requirements/执行追踪与复盘.md`，版本 v1.0（2026-07-02）。

**Why:** 记录一期核心规则，避免后续设计/实现阶段重复澄清或与原文口径漂移。

**How to apply:** architect 设计接口契约时参照实体字段和接口依赖；implementer 实现异常识别时核对阈值和幂等规则。

## 执行阶段异常类型（一期 4 类）

| 类型 | 触发条件 | 级别 |
|------|----------|------|
| LOW_ACHIEVEMENT（低达成） | 达成率 < 70% | 红色 |
| SLOW_PROGRESS（达成过慢） | 达成进度 < 时间进度 × 60% | 红色 |
| ABNORMAL_CHANGE（异常增长/下降） | 连续3天零产出 或 日环比突降≥50% | 红色 |
| HIGH_DOS（高库存） | DOS > 60天 | 黄色 |

串货/囤货一期不做交叉校验，人工举报时仅做标记记录。

## 分级处置阈值

- 红色：达成率 < 70% 或 偏差率 > 30%；自动上报区域经理
- 黄色：达成率 70%–85% 或 偏差率 15%–30%；RSM 跟进
- 绿色：达成率 ≥ 85%

## 关键 SLA

- 红色预警：异常识别到通知事件发出 ≤ 1 小时
- 高风险响应时效（端到端）：≤ 24 小时
- 数据延迟容忍：≤ 24 小时；超出则标记"数据异常"

## 核心实体（5 个）

ExecutionSnapshot / AnomalyRecord / NotificationEvent / ReviewReport / ModelFeedback

## 设计期遗留依赖

- D3（R4）：通知触发接口形态待 architect 与 DCR APP 团队对齐（阻塞实现）
- D4：ModelFeedback 回流格式待与预测模块对齐
- 偏差率定义：偏差率 = |目标量 - 达成量| ÷ 目标量，当达成率 < 100% 时与 (1 - 达成率) 等价

## 高优先级开放问题（产品/业务需确认）

- OQ-1：时间进度分母是自然日还是工作日（影响达成过慢判断）
- OQ-3：异常增长/下降的环比基准（前1日 vs 近N日均值）
- OQ-4：DOS 计算公式归属（本模块 or 数据接入模块）

关联：[[project-ai-target-platform]] [[ref-data-ingestion-module]]
