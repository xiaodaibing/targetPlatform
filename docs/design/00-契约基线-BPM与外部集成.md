# 技术设计：契约基线（三）BPM 与外部集成

> 版本：v1.0 | 日期：2026-07-23 | 作者：architect
> **本文替代 `00-契约基线-AgentScope与预测.md`（整份作废，E-3）。** 原文档涉及的 AgentScope 集成、Agent 清单、System Prompt 边界、RuntimeContext、出境脱敏中间件、市场情报搜索工具、大数据实时预测接口、三档规则引擎、HITL/SSE，一期全部不实现。
> 本文承接一期真正的外部集成面：**公司 BPM 审批**、**DCR APP 通知与 APP 接口**、**大数据历史指标**，以及 ADR-0004 废弃后遗留的数据合规要求。

---

## 1. 一期外部集成全景

| 外部系统 | 方向 | 阻塞性 | 归属模块 | 关键风险 |
|---------|------|--------|---------|---------|
| **公司 BPM** | 出（发起）+ 入（回调）| **强阻塞** | workbench | 契约与模板未定；不可用时不能丢数据 |
| **DCR APP** | 出（通知）+ 入（APP 调本平台接口）| 非阻塞发布流程，但阻塞 APP 交付 | workbench / tracking | 形态未定；BP 与 APP 用户 ID 映射未知 |
| **大数据团队** | 出（拉指标）| 强阻塞 | dataingest | 出口归属未定（大数据 vs DCR）|
| **DCR** | 出（拉销售明细/季节参数/人员门店/调店）| 强阻塞 | dataingest | 调店流水是否存在未知 |
| **产品库** | 出（拉机型与标记）| 强阻塞 | dataingest | HSP/平板/重点/停产 标记是否现成 |
| **库存 / 激活(UAC)** | 出（拉库存、DOS、激活）| 强阻塞 | dataingest | — |
| **UAC 权限** | 出（拉授权）| 强阻塞 | workbench | — |

> **一期不存在的外部依赖**：LLM Provider（DashScope / OpenAI / Anthropic）、搜索引擎（市场情报）、大数据预测接口、大数据回流接口。任何设计或实现中出现这四类出站调用，均为 v1.0 残留。

---

## 2. BPM 审批集成（ADR-0005 落地）

### 2.1 职责边界（再次明确，实现期最容易越界的地方）

| 本平台做 | BPM 做 |
|---------|--------|
| 按 国家×品牌×目标类型 匹配流程模板 ID | 审批人路由与层级 |
| 发起流程实例并携带业务参数 | 加签 / 转交 / 退回 |
| 接收回调、流转本地状态、生成版本快照 | 催办 / 超时策略 |
| 展示审批状态、当前处理人、审批意见、跳转链接 | 审批意见收集与留存 |

**本平台明确不落库的东西**：审批人、审批层级、加签规则、转交规则、超时规则。这些一旦在本平台建了表，就会和 BPM 双写漂移——发现有人要加 `t_approval_user` 之类的表，先回来看这一节。

### 2.2 配置表 `goal_wb_approval_template`

| 字段 | 说明 |
|------|------|
| `id` | 主键 |
| `country_code` / `brand_code` / `target_type` | 匹配维度（三者组合唯一）|
| `process_template_id` | BPM 流程模板 ID |
| `business_key_prefix` | 业务标识前缀，默认 `goal-person` |
| `scope_type` | BATCH / BY_STORE（派-2 未定前恒 BATCH）|
| `enabled` | 是否启用 |
| `created_by` / `updated_by` / `updated_at` | 审计 |

未匹配到启用中的模板 → 抛 `APPROVAL_TEMPLATE_NOT_CONFIGURED`，阻止提交（AC-18）。

### 2.3 发起流程（出站）

```java
@FeignClient(name = "${goal.bpm.service-name}", path = "${goal.bpm.base-path}")
public interface BpmClient {
    BpmStartResponse startProcess(BpmStartRequest request);
    BpmProcessDTO queryProcess(String processInstanceId);
}
```

- `businessKey = {business_key_prefix}-{targetPlanId}`，在 BPM 侧唯一。
- 携带的业务参数见接口清单 §二 `BpmStartRequest`，其中**软校验结果四字段必须带上**（`softCheckSkipped` / `softCheckPassed` / `softCheckGapValue` / `softCheckGapRatio`）——审批人需要看到「这批目标合计比总目标少了多少」才能判断，否则审批就是走过场。
- 超时 ≥ 30s，重试 ≤ 3 次（指数退避）。**重试必须带同一 `businessKey`**，BPM 侧据此去重，避免网络抖动导致创建出多个流程实例。

### 2.4 回调（入站）

```
POST /api/bpm/callback
Body: { processInstanceId, businessKey, result(APPROVED/REJECTED),
        approverUserId, approverName, comment, finishTime }
```

**幂等实现要求（AC-21，硬性）**：
1. `goal_wb_approval_instance` 表以 `process_instance_id` 建**唯一索引**；
2. 处理前先按状态机前置条件判断（只有 `PENDING_APPROVAL` 才接受回调），非法状态直接返回成功但不做任何事；
3. 版本快照与通知触发放在同一事务边界内的后置动作，靠状态流转结果驱动，不靠回调次数驱动。

只做第 2 步不够——并发两次回调会同时通过状态判断。唯一索引是兜底，必须有。

### 2.5 降级：`SUBMIT_FAILED`

BPM 不可用时：
- 人员目标状态 → `SUBMIT_FAILED`，**目标数据完整保留**（这是降级设计的全部意义，不能因为提交失败就回滚掉用户辛苦调好的几百行目标）
- 记录失败原因与时间，页面提示「审批系统暂不可用，请稍后重试」
- 提供单条「重新提交」与管理员批量重试入口
- 重试时复用同一 `businessKey`

### 2.6 鉴权与可观测

- 出站鉴权方式待 BPM 团队确认（派-1）；入站回调需做**来源校验**（IP 白名单 / 签名 / 网关鉴权，三选一以上），不可裸开放。
- BPM 调用与回调**单独埋点**：调用成功率、P95 耗时、回调延迟、`SUBMIT_FAILED` 累计数。这是一期唯一的强阻塞外部依赖，失败率必须可观测。

---

## 3. DCR APP 集成

### 3.1 出站：通知触发

```java
@FeignClient(name = "${goal.dcrapp.service-name}", path = "${goal.dcrapp.base-path}")
public interface DcrAppNotifyClient {
    void notify(NotificationEventDTO event);
}
```

- 形态未定（REST 回调 / Kafka / SDK，派-6）。**契约先按 Feign 定义，`NotificationEventDTO` 保持稳定**；若最终定为 Kafka，只换实现类，DTO 与调用方代码不动。
- 异步触发，失败不阻塞主流程；结果写 `goal_wb_notify_log`，管理员可重试。
- 幂等键：
    - 目标类通知：(targetPlanId, promoterId, notificationType, 版本号)
    - 预警类通知：(promoterId, metricCode, alertType, 自然日)

### 3.2 入站：APP 调本平台接口

见接口清单 §3.2。三条硬约束再强调一次：

1. **接口不接受 `promoterId` 入参**，一律从鉴权上下文解析当前用户对应的 BP。暴露了这个入参，权限校验早晚被绕过。
2. 越权返回空集，不返回 403。
3. P95 ≤ 1 秒。

### 3.3 待对齐清单（派-6）

| 事项 | 说明 |
|------|------|
| 通知形态 | REST / Kafka / SDK |
| BP 与 APP 用户 ID 映射 | 由哪侧维护？本平台存映射表还是每次调对方接口换算？ |
| APP 调本平台的鉴权方案 | 沿用公司统一网关 token，还是单独签发？ |
| 页面交互设计交付物 | 本平台需产出哪些原型/字段说明给对方 |
| 联调窗口 | 双方排期 |

> 这五条任何一条未定，APP 端能力就无法在一期交付。**但 PC 端闭环独立可用**，APP 缺失不阻塞一期上线——这是当初选择「不建移动端工程」的直接收益，实现排期时要把 PC 与 APP 解耦，别让 APP 拖住整条线。

---

## 4. 大数据历史指标接口（ADR-0002 落地）

```java
@FeignClient(name = "${goal.bigdata.service-name}", path = "${goal.bigdata.base-path}")
public interface BigDataMetricClient {
    MetricPullResponse queryMetrics(MetricPullRequest request);
}
```

`MetricPullRequest`：countryCode, brandCode, periodMonth, `List<String> metricKeys`, dimensionType, `List<String> dimensionIds`
`MetricPullResponse`：`List<MetricItem>`（metricKey, dimensionType, dimensionId, value, unit, **statPeriod**, **generatedAt**）+ `List<String> missingKeys`

**与 v1.0 `BigDataForecastClient` 的本质差异**：

| | v1.0（作废）| v2.0 |
|---|---|---|
| 提供内容 | 预测基准值 + 置信度 | 历史统计指标 |
| 调用时机 | 目标创建主流程内**实时同步**调用 | 定时拉取，落本地表；测算时读本地 |
| SLA 敏感度 | **高**——接口慢/挂则目标创建卡死 | 低——本地有数据即可测算 |
| 失败影响 | 阻塞主流程 | 数据可用性降级为 PARTIAL/LIMITED，仍可人工设定 |

> 这个改动把一条实时强依赖降级成了一次批量拉取，是本轮架构风险下降最明显的一处。实现时**不要退回实时调用**——即便某个指标本地缺失，也应走 `METRIC_UNAVAILABLE` 提示人工设定，而不是临时同步去问大数据要。

**必须与大数据团队钉死的口径**（OQ-DI-01 / M02 OQ-04）：
- 每个 `metricKey` 的统计口径与时间窗（尤其「BP 近 3 月月均 SO」是否含当月、离职 BP 是否计入）
- 「去年同期增长幅度」由大数据提供成品指标，还是提供分子分母由我方算
- 指标出口归属：与 DCR 销售明细是否同一出口
- 数据延迟承诺（决定我方 T+1 调度时点）

---

## 5. 数据合规（承接 ADR-0004 废弃后的遗留项）

ADR-0004 废弃只消除了「数据发往 LLM 提供商」这一条出境通道，以下义务不因此免除，需在各模块设计的安全小节落实：

| 要求 | 落地位置 |
|------|---------|
| **人员级数据隔离**：BP 只能查看本人目标与达成 | workbench / tracking 的 APP Controller 入口；不接受 `promoterId` 入参 |
| **外部交互数据最小化**：发往 BPM / DCR APP 的负载只带流程与展示必需字段，不带全量明细 | `BpmStartRequest`、`NotificationEventDTO` 字段清单即为白名单，不得随意扩 |
| **敏感信息不落日志**：BP 姓名、门店等可标识信息在 DEBUG 日志中脱敏 | 全局日志脱敏配置 |
| **操作日志留存期** | 待确认（建议 ≥ 2 年，与公司审计要求对齐）|
| **权限数据来源单一** | 一律取自 UAC，不在本平台维护第二套授权 |

---

## 6. 生产要点清单

- [ ] BPM `businessKey` 全局唯一，重试复用同一 key
- [ ] BPM 回调唯一索引 + 状态机前置双重幂等
- [ ] BPM 回调入站做来源校验
- [ ] `SUBMIT_FAILED` 场景数据零丢失，可重试
- [ ] 通知触发异步化，失败不阻塞主流程，可重试
- [ ] APP 接口不暴露 `promoterId` 入参
- [ ] 大数据指标定时拉取落本地，**不做实时同步调用**
- [ ] 外部 Feign 全部配超时（≥30s）与重试上限（≤3）
- [ ] 无任何 LLM / 搜索引擎出站调用（ArchUnit 守卫）
- [ ] BPM 调用与回调单独埋点，`SUBMIT_FAILED` 计数纳入告警

---

## 7. 关联

- ADR-0005（人员目标审批接入公司 BPM）
- ADR-0002（总目标测算来源改为公式化本地测算）
- ADR-0001 / ADR-0004（已废弃，仅供二期参考）
- `00-契约基线-总览与边界.md` / `00-契约基线-接口清单.md`
