# 技术设计:AI 预测与拆分(forecast / M02)

> 版本:v1.0 | 日期:2026-07-03 | 作者:architect
> 上位约束:`docs/design/00-契约基线-总览与边界.md`、`-接口清单.md`、`-AgentScope与预测.md`;需求 `docs/requirements/AI预测与拆分.md`。
> 定位:forecast 模块细化设计。**门面签名/DTO/错误码沿用契约基线,本文不改契约**,只展开数据模型、算法、时序、非功能。
> 包路径:`com.transsion.goal.forecast`;共享 DTO/枚举/错误码只放 `com.transsion.goal.common`。

---

## 架构概览

### 模块定位与边界

forecast 是**无独立进程**的包级模块(有界上下文)。对上游只经**门面接口**读数、对外部大数据只经 **Feign**、经 **agent 层**生成文本,不直连别模块的表/Mapper,不用 Feign 调内部模块。

```
                        ┌───────────────────────── forecast(com.transsion.goal.forecast) ─────────────────────────┐
 workbench ──facade──►  │  facade 层:PredictionFacade / SplitFacade(契约基线锁定,应用内注入)                       │
 (转调,前端经它)        │        │                                                                                  │
                        │  service/编排层:                                                                          │
                        │   ┌ PredictionOrchestrator(预测编排:充分性检查→调基准→三档规则→LLM依据)                  │
                        │   ├ SplitEngine(逐层拆分)                                                                 │
                        │   ├ RebalanceEngine(自动平衡,四模式+锁定)                                                 │
                        │   ├ ValidationEngine(强校验 CV-1/2/3)                                                     │
                        │   └ AnomalyDetector(异常识别,一期3类)                                                     │
                        │  domain/规则:TierRuleEngine(三档) / WeightNormalizer(权重归一)                            │
                        │  mapper/表:t_target_prediction / t_prediction_rule_config / t_split_task /               │
                        │             t_split_item / t_split_anomaly / t_split_adjust_log / t_split_rule_config     │
                        └───────┬───────────────────┬─────────────────────────┬──────────────────────────────────┘
                                │facade             │Feign                    │agent Bean
                                ▼                    ▼                         ▼
              dataingest.HistoryDataQueryFacade   BigDataForecastClient   agent.forecastBasisAgent
                 (queryHistory/checkAvailability)   (baseline 实时基准)      (basisText 文本,HITL)
                                │                                              │
              workbench.TargetPlanFacade(读计划元信息)          经 agent 层中间件出境脱敏(PiiRedaction)
              对象/机型列表:复用 dataingest 已接入的 Territory/Product 结果(不新增重复 Feign)
```

### 依赖方向(全部单向向下,无环)

| 依赖 | 通道 | 用途 | 失败处置 |
|------|------|------|---------|
| dataingest `HistoryDataQueryFacade` | 门面 | 充分性检查(checkAvailability)、历史均值/销量(queryHistory) | HISTORY_INSUFFICIENT → 预测降 LIMITED;某权重源缺 → 归零重归一(AC-07) |
| workbench `TargetPlanFacade` | 门面 | 拆分前读计划元信息(国家/品牌/类型/周期/是否拆解/拆解层级) | 计划不存在 → SPLIT 前置校验失败 |
| 大数据 `BigDataForecastClient` | Feign | 实时预测基准 baseline | 返回 LIMITED → DEGRADED;超时/无返回/调用异常 → FAILED,均允许人工兜底(AC-04) |
| agent `forecastBasisAgent` | Bean(agent 层) | 生成 basisText 业务依据文本 | LLM 失败 → 规则模板文案,不阻塞数值链路 |
| 对象/机型列表 | 复用 dataingest Territory/Product 结果 | 拆分维度参照 | 缺失该层对象 → 该层拆分空,记录降级 |

> **数值与文本二分(契约基线底线)**:三档/区间/拆分/平衡/校验/异常全部是**本系统规则与数值计算**,不经 LLM;LLM 仅 `forecastBasisAgent` 生成文本,全程 HITL,失败可降级,与数值链路解耦。

---

## 数据模型

> ORM:MyBatis-Plus。乐观锁字段 `version` 加 `@Version`;逻辑主键 `id` 用 BIGINT 自增(或雪花,按 Hulk 约定)。金额统一 `DECIMAL(18,2)`,`≥0`(NF 目标值边界)。所有表带 `create_time/update_time`。forecast 独占本组表,不与别模块共享。

### 1. t_target_prediction — AI 预测结果(追加式留痕,AC-22)

| 字段 | 类型 | 约束/说明 |
|------|------|----------|
| id | BIGINT | PK,自增 |
| target_plan_id | BIGINT | 关联目标计划;联合索引 `idx_plan_current(target_plan_id, is_current)` |
| country_code | VARCHAR(16) | 国家 |
| brand_code | VARCHAR(16) | 品牌 |
| target_type | VARCHAR(8) | SI/SO/ST(枚举) |
| metric_code | VARCHAR(32) | 目标指标编码,一期恒 `SALES_VOLUME`(D-TAM 预留);预测按指标产出,未来多指标时同 plan 可有多指标预测行 |
| target_period | VARCHAR(16) | 目标周期 YYYY-MM |
| baseline_value | DECIMAL(18,2) | 大数据返回基准值(留痕) |
| conservative_value | DECIMAL(18,2) | 保守值 |
| recommend_value | DECIMAL(18,2) | 推荐值 |
| challenge_value | DECIMAL(18,2) | 挑战值 |
| interval_lower | DECIMAL(18,2) | 推荐区间下限 |
| interval_upper | DECIMAL(18,2) | 推荐区间上限 |
| confidence | INT | 置信度 0-100 |
| basis_text | TEXT | 业务语言依据(中/英,可空→仅当 LLM 降级模板时兜底) |
| prediction_source | VARCHAR(64) | 大数据来源标识 |
| rule_config_version | INT | 三档规则版本(AC-22 快照) |
| generated_at | DATETIME | 基准生成时间(回溯) |
| data_availability | VARCHAR(8) | FULL/PARTIAL/LIMITED |
| is_degraded | TINYINT | 是否降级运行 |
| status | VARCHAR(16) | PENDING/RUNNING/COMPLETED/DEGRADED/FAILED |
| fail_reason | VARCHAR(512) | FAILED 原因文本(AC-04 展示) |
| is_current | TINYINT | 1=当前有效行(getPrediction 命中),0=历史快照 |
| created_by | VARCHAR(64) | 触发人 |
| create_time | DATETIME | |

> 每次 `predict` 追加一行并置 `is_current=1`,同 plan 旧行改 0(单事务)。`getPrediction` 取 `is_current=1`。历史行即 AC-22 快照,支持回溯。**六字段全有或全空**在写入前由编排层保证(AC-05):校验不过则整行 status=FAILED、三档/区间字段留空。

### 2. t_prediction_rule_config — 三档规则配置(新增)

> 契约基线 §5.2 提到"`SplitRuleConfig` 同源思路"。为职责清晰,**三档生成规则单独建表**(与拆分权重解耦),不过度合并。按 `国家×品牌×目标类型` 生效。

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| country_code / brand_code / target_type | VARCHAR | 适用范围;唯一键 `uk_scope(country_code,brand_code,target_type,enabled)`;**P3 全球兜底包 country_code/brand_code 取通配值 `*`**(D-4) |
| rule_tier | VARCHAR(4) | P1(国家专属,一期落地)/ P3(全球兜底,一期落地);P0/P2 为匹配机制预留字段值,一期不建配置入口(D-4) |
| conservative_ratio | DECIMAL(5,4) | 保守下调幅度,如 0.1000 → 保守=基准×(1-0.10) |
| challenge_ratio | DECIMAL(5,4) | 挑战上调幅度,如 0.1500 → 挑战=基准×(1+0.15) |
| interval_ratio | DECIMAL(5,4) | 区间带宽 ±,如 0.0500 → [推荐×0.95, 推荐×1.05] |
| confidence_source | VARCHAR(16) | BIGDATA(随基准返回) / RULE_MAPPING(本系统按充分性映射)——**OQ-04 待定,二选一** |
| confidence_cap_full | INT | RULE_MAPPING 时 FULL 置信度上限(如 90) |
| confidence_cap_partial | INT | PARTIAL 上限(如 70) |
| confidence_cap_limited | INT | LIMITED 上限(如 40,禁伪精确 AC-02) |
| config_version | INT | 配置版本号(写入预测快照) |
| enabled | TINYINT | 是否启用 |
| updated_by / update_time | | 留痕 |

### 3. t_split_rule_config — 拆分规则配置(SplitRuleConfig)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| country_code / brand_code / target_type | VARCHAR | 适用范围;唯一键 `uk_scope`;**P3 全球兜底包 country_code/brand_code 取通配值 `*`**(D-4) |
| rule_tier | VARCHAR(4) | P1(国家专属,一期落地)/ P3(全球兜底,一期落地);P0/P2 预留,一期不建配置入口(D-4) |
| split_levels | JSON | 拆解层级列表,如 `["REGION","DEALER","SHOP","PROMOTER","MODEL"]`(枚举值统一用 SHOP,对齐基线 ObjectLevel;需求文旧稿曾写 STORE,以此为准) |
| weight_historical_rate | INT | 历史达成率权重占比 0-100 |
| weight_capital | INT | 资金盘权重占比 0-100 |
| weight_territory | INT | 阵地数权重占比 0-100;三项之和=100(入库校验) |
| historical_periods | INT | 历史达成率取近 N 期(可配置,4.2) |
| balance_mode | VARCHAR(16) | 默认平衡模式 AI_WEIGHT/HISTORY/RATE/AVERAGE |
| config_version | INT | 配置版本号 |
| enabled | TINYINT | |
| updated_by / update_time | | |

### 3b. 规则包优先级匹配机制(D-4,适用于 t_prediction_rule_config 与 t_split_rule_config)

> 对齐产品 Demo 的 P0(全球标准)→P1(国家专属)→P2(品牌/品类)→P3(全球兜底) 规则模板体系。**一期只落地 P1、P3 两级**,不新建"规则包"独立实体——沿用现有按 scope 配置的两张表,加 `rule_tier` 字段标注层级、`country_code`/`brand_code` 允许通配值 `*` 表示"全球兜底",匹配逻辑做优先级回退即可,成本增量小。

**匹配顺序**(`RuleConfigResolver.resolve(countryCode, brandCode, targetType)`,两张表复用同一套解析逻辑,仅表名不同):
```
1. 查 country_code=实际值 AND brand_code=实际值 AND target_type=实际值 AND rule_tier='P1' AND enabled=1
   命中 → 返回(P1 优先)
2. 未命中 → 查 country_code='*' AND brand_code='*' AND target_type=实际值 AND rule_tier='P3' AND enabled=1
   命中 → 返回(P3 兜底)
3. 均未命中 → 三档场景:预测按现有充分性/降级路径处理(不新增错误码);拆分场景:抛 SPLIT_NO_RULE(AC-08/AC-08c)
```
- `target_type`(SI/SO/ST)业务差异大,**恒精确匹配,不参与通配**——通配仅发生在 country_code/brand_code 维度。
- P3 兜底包为运营侧初始化数据(一期建库时预置孟加拉外的通用默认值),不提供 P0/P2 配置管理界面;`SplitRuleConfigDTO`/三档规则 DTO 预留 `ruleTier` 字段,便于二期直接开放 P0(临时指定)/P2(品牌/品类)插级,不改契约结构。
- 管理后台(见"拆分规则配置后台")新建 P3 包时,`country_code`/`brand_code` 入参校验放行通配值 `*`;新建 P1 包仍要求填具体国家/品牌。

### 4. t_split_task — 拆分任务头(计划级状态 + 并发闸)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| target_plan_id | BIGINT | **唯一键 `uk_plan`**(同计划仅一任务,NF 并发闸) |
| confirmed_total | DECIMAL(18,2) | 确认后的总目标 |
| balance_mode | VARCHAR(16) | 本次拆分选定平衡模式 |
| split_rule_version | INT | 使用的拆分规则版本(留痕) |
| status | VARCHAR(16) | SplitStatus:PENDING/SPLITTING/DRAFT/ADJUSTING/VALIDATION_PASS/VALIDATION_FAIL |
| version | INT | `@Version` 乐观锁(任务级状态流转防并发) |
| created_by / create_time / update_time | | |

### 5. t_split_item — 拆分结果明细(SplitResult 行,树形)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| target_plan_id | BIGINT | 索引 `idx_plan_level(target_plan_id,level)` |
| parent_item_id | BIGINT | 上级明细 id(逐层拆分/平衡定位);顶层为 NULL,索引 `idx_parent` |
| level | VARCHAR(12) | ObjectLevel:COUNTRY/REGION/DEALER/SHOP/PROMOTER/MODEL(SHOP=门店,统一枚举) |
| dimension_id | VARCHAR(64) | 层级对象 ID |
| dimension_name | VARCHAR(128) | 对象名(展示) |
| allocated_value | DECIMAL(18,2) | 分配值,≥0 |
| weight_snapshot | DECIMAL(9,6) | AI 初次拆分归一权重(AI_WEIGHT 平衡模式复用) |
| is_locked | TINYINT | 是否锁定(平衡时不动) |
| is_manual | TINYINT | 是否人工修改过 |
| adjust_reason | VARCHAR(512) | 调整原因(红色异常必填) |
| version | INT | `@Version` 乐观锁(同层对象并发修改,NF);每次平衡/调整递增 |
| created_by / create_time / update_time | | |

### 6. t_split_anomaly — 异常记录(SplitAnomaly)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| split_item_id | BIGINT | 关联明细,索引 |
| target_plan_id | BIGINT | 索引(按计划聚合展示) |
| anomaly_type | VARCHAR(24) | INTERVAL_DEVIATION/HISTORY_DEVIATION/TOTAL_IMBALANCE(一期仅此 3 类) |
| severity | VARCHAR(8) | 一期恒为 RED；YELLOW 为二期预留(一期拆分无黄色异常，见需求 6.1) |
| trigger_desc | VARCHAR(512) | 触发条件描述 |
| history_avg | DECIMAL(18,2) | 近 3 月历史均值(HISTORY_DEVIATION 展示,AC-16) |
| current_value | DECIMAL(18,2) | 当前值 |
| deviation_pct | DECIMAL(7,2) | 偏差百分比 |
| is_resolved | TINYINT | 是否已处理 |
| resolution_note | VARCHAR(512) | 处理说明 |
| bypass_confirmed | TINYINT | 黄色异常是否确认绕过(二期预留，一期恒 0) |
| confirmed_by | VARCHAR(64) | |
| confirmed_at | DATETIME | |
| create_time | DATETIME | |

### 7. t_split_adjust_log — 调整留痕(AC-19,审计只增不改)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| target_plan_id / split_item_id | BIGINT | 索引 |
| level / dimension_id | VARCHAR | 对象定位 |
| action_type | VARCHAR(16) | MANUAL_EDIT/REBALANCE/LOCK/UNLOCK/BYPASS_CONFIRM/SKIP_AI |
| before_value | DECIMAL(18,2) | 调整前值 |
| after_value | DECIMAL(18,2) | 调整后值 |
| adjust_reason | VARCHAR(512) | 原因(非异常场景可选填) |
| related_anomaly_type | VARCHAR(24) | 关联异常类型(可空) |
| operator | VARCHAR(64) | 操作人 |
| operate_time | DATETIME | 操作时间 |

### 状态机(对齐需求 §7.2)

**PredictionStatus**(`t_target_prediction.status`):
```
PENDING ──► RUNNING ──► COMPLETED     (FULL/PARTIAL:三档完整 + 真实置信度)
                    ├─► DEGRADED      (数据不足/LIMITED:禁伪精确,可人工设定)
                    └─► FAILED        (预测服务异常/大数据无返回/超时 或 AC-05 单调性违反:允许跳过AI人工输入,SKIP_AI 留痕)
```

**SplitStatus**(`t_split_task.status`):
```
PENDING ──► SPLITTING ──► DRAFT ──► ADJUSTING ──► VALIDATION_PASS  (强校验通过,移交 workbench 待发布)
                            ▲            │
                            │            └──► VALIDATION_FAIL ──► ADJUSTING  (校验失败,回退继续调整)
                            └───────(每次 rebalance/异常刷新回 DRAFT/ADJUSTING)
```
状态流转由 `t_split_task.version` 乐观锁保护;非法跃迁(如 PENDING 直接 validate)抛 `VALIDATION_FAILED` 前置拦截或 `LOCK_CONFLICT`。

---

## 接口契约

> 门面签名/DTO/错误码**已在契约基线锁定**,本文不改。下方仅补充触发条件、幂等、状态语义。均为应用内 Java 注入接口(非 REST),异常经 `GoalBizException(GoalErrorCode)` 抛出。前端不直接调 forecast,统一经 workbench REST 转调(见"前端契约")。

### 接口:PredictionFacade

| 方法 | 语义 | 关键约束 |
|------|------|---------|
| `PredictionHandle predict(PredictionRequest)` | **异步**触发:充分性检查→调大数据基准→三档规则→LLM 依据;立即返回受理句柄(target_plan_id + 受理时刻),后台线程池执行 | 幂等:同 plan 若已有 RUNNING 任务,复用不重复触发(返回现有句柄);COMPLETED/DEGRADED/FAILED 可重跑(追加新 is_current 行) |
| `PredictionResultDTO getPrediction(Long targetPlanId)` | 前端**轮询**读结果,含 status(PENDING/RUNNING/COMPLETED/DEGRADED/FAILED) | 只读幂等;无记录返回 status=PENDING 空壳 |

- `PredictionRequest`:targetPlanId, countryCode, brand, targetType, metricCode, targetPeriod, objectType(可选机型范围)。
- `PredictionResultDTO`:conservativeValue/recommendValue/challengeValue/intervalLower/intervalUpper/confidence(0-100)/basisText/predictionSource/dataAvailability/status。
- **AC-05 强约束**:保守≤推荐≤挑战,下限≤推荐≤上限;违反→整体 FAILED,六字段全空或全有,禁止部分输出。
- **AC-04 降级/失败**:数据不足 → `status=DEGRADED`;大数据超时/无返回/调用异常或其他预测服务失败 → `status=FAILED`;两者均附提示信息并允许上层跳过 AI、直接人工输入(SKIP_AI 留痕)。

### 接口:SplitFacade

| 方法 | 语义 | 错误码触发条件 |
|------|------|--------------|
| `SplitResultDTO split(SplitRequest)` | 接确认总目标,逐层拆分(国家→区域→Dealer→门店→Promoter→机型),写初稿+跑异常,status→DRAFT | `SPLIT_NO_RULE`:按 §3b 匹配顺序,该 国家/品牌/目标类型 既无 P1 专属配置也无 P3 兜底配置(enabled)(AC-08/AC-08c) |
| `SplitResultDTO rebalance(RebalanceRequest)` | 人工改某层值后按模式重分配未锁定部分,锁定值不动,重跑异常,status→ADJUSTING | `LOCK_CONFLICT`:**仅表**锁定值之和≥上级目标、无可分配余量(AC-10/11);`@Version` 乐观锁并发冲突**内部重试或提示重载**,不复用 LOCK_CONFLICT |
| `ValidationResultDTO validate(Long targetPlanId)` | 强校验 CV-1/2/3(≥ 且 0 容忍);通过 status→VALIDATION_PASS,失败→VALIDATION_FAIL | `VALIDATION_FAILED`:任一 CV 不过,`failures[]` 附 level/name/subtotal/parentValue/gap(AC-12);**存在未处理 RED 异常时亦拦截**(AC-21) |
| `SplitResultDTO getSplitResult(Long targetPlanId)` | 工作台确认页展示当前树 + 异常 | 只读幂等 |

- `SplitRequest`:targetPlanId, confirmedTotal, balanceMode。
- `RebalanceRequest`:targetPlanId, changedLevel, changedDimensionId, newValue, balanceMode。
- `SplitResultDTO`:targetPlanId, status(SplitStatus), items[](level,dimensionId,dimensionName,allocatedValue,isLocked,isManual,adjustReason,version,anomalies[])。
- `Anomaly`:anomalyType, severity(一期恒 RED；YELLOW 二期预留), triggerDesc, isResolved, resolutionNote, bypassConfirmed(二期预留)。
- `ValidationResultDTO`:passed(bool), failures[](level,name,subtotal,parentValue,gap)。

- **幂等/版本**:`rebalance`/异常处理携带 `item.version`,MyBatis-Plus `@Version` CAS,冲突即刷新重试或提示重载,防同层并发覆盖(NF)。`split` 对已有 DRAFT/ADJUSTING 任务重跑=覆盖重建(单事务,清旧 item/anomaly 再写)。
- **异常处理入口**(AC-15/16/21,契约基线前端路由 `POST /goal/plans/{id}/split/{itemId}/resolve`):落在 `t_split_anomaly` 更新 + `t_split_adjust_log` 追加,一期仅 RED 需 adjust_reason/确认处理(YELLOW 记 bypass_confirmed 为二期预留，一期不触发)。此为 workbench REST → SplitFacade 内部方法(非新增门面方法,不改锁定契约)。

---

## 三档规则引擎(TierRuleEngine,非 LLM)

落地契约基线 §5.2 公式,规则按 `国家×品牌×目标类型` 取 `t_prediction_rule_config`,匹配顺序见 §3b(P1 国家专属优先,未命中回退 P3 全球兜底):

```
recommendValue    = baselineValue                              (大数据基准直接作推荐值)
conservativeValue = round(baselineValue × (1 - conservativeRatio))
challengeValue    = round(baselineValue × (1 + challengeRatio))
intervalLower     = round(recommendValue × (1 - intervalRatio))
intervalUpper     = round(recommendValue × (1 + intervalRatio))
confidence        = confidence_source==BIGDATA ? 大数据返回置信度
                                               : min(充分性映射cap, 大数据返回或默认)   (OQ-04)
```

### 充分性 → 置信度降级映射(禁伪精确,AC-02/03)

充分性来自 `HistoryDataQueryFacade.checkAvailability`(本系统前置)**与** 大数据 `baseline` 返回 `dataAvailability` 取**更保守者**:

| 充分性 | 展示提示 | 置信度约束 |
|--------|---------|-----------|
| FULL | 正常三档 + 真实置信度 | 正常(≤ cap_full) |
| PARTIAL | "数据部分可用,置信度受影响"(AC-03) | 降低(≤ cap_partial) |
| LIMITED | "数据不足,预测仅供参考,建议结合业务经验人工设定"(AC-02) | **≤ cap_limited,不得虚高** |

- **AC-04 数据不足降级**:大数据返回 `dataAvailability=LIMITED` 或本系统充分性为 LIMITED 时,可不生成完整三档或生成受限结果,`status=DEGRADED(is_degraded=1)`,提示人工设定,不阻塞创建流程。
- **AC-04 预测失败**:大数据无返回、超时或调用异常时,`status=FAILED`,三档字段留空,`fail_reason` 记录原因,允许人工设定。
- **AC-05 单调性守卫**:三档/区间计算后强校验 `保守≤推荐≤挑战 ∧ 下限≤推荐≤上限`;违反(如异常 ratio 配置)→ status=FAILED、三档字段留空、fail_reason 记录,降级人工。
- **AC-22 快照**:写入 `prediction_source + generated_at + rule_config_version + baseline_value + 三档输出`,append 行永久留存。

---

## 拆分 / 平衡 / 强校验算法

### 逐层拆分(SplitEngine,AC-06)

自顶向下遍历 `split_levels`,对每一层的每个父对象,把父分配值按子对象**综合权重**分配:

1. **权重占比归一(源级)**:取 `weight_historical_rate/capital/territory`。若某源数据不可用(AC-07),该占比归零,剩余源重新归一化到 100%。全源不可用则退化为平均分摊,记录降级。
2. **对象指标归一(对象级)**:对每个可用源 k,取各兄弟对象指标 `m_k`(历史达成率经 `HistoryDataQueryFacade`;资金盘/阵地数见 OQ-02/03 占位),兄弟间归一化使 `Σ m_k = 1`。
3. **综合权重**:`score(o) = Σ_k ( w_k × m_k(o) )`;因每个 `Σ_k m_k=1` 且 `Σ w_k=1`,故兄弟 `Σ score=1`。
4. **分配**:`allocated(o) = round(parentValue × score(o))`;`weight_snapshot=score(o)` 落库(供 AI_WEIGHT 平衡)。
5. **取整残差**:四舍五入残差补给 `score` 最大对象,保证 `Σ allocated ≥ parentValue`(满足 CV 的 ≥ 语义)。

> 权重源缺失重归一提示文案在返回 DTO/异常提示里带出(AC-07):"[XX 权重]数据不可用,已按剩余权重重新分配"。

### 自动平衡(RebalanceEngine,AC-09/10/11)

触发时机(一期):改总目标 / 改区域层 / 改机型层 / 月中调整后。四模式二次分配**未锁定**部分:

| 模式 | 分配基准 |
|------|---------|
| AI_WEIGHT | `weight_snapshot` 原始权重 |
| HISTORY | 近 3 月各对象历史销量占比(`HistoryDataQueryFacade`) |
| RATE | 近 3 期各对象平均达成率占比 |
| AVERAGE | 未锁定对象等分 |

算法:
```
distributable = parentValue − Σ(locked 子对象值)
if distributable ≤ 0:  抛 LOCK_CONFLICT("锁定值之和已超出或等于上级目标…")   // AC-10
对未锁定对象按模式权重 w'(归一) 分配:alloc = round(distributable × w')
锁定对象值保持不变(AC-11);残差补最大未锁定对象;version+1;写 t_split_adjust_log(REBALANCE)
平衡完成后重跑 AnomalyDetector,并保证该层满足对应 CV 约束(AC-09)
```

### 强校验(ValidationEngine,AC-12/13/14,0 容忍)

点击提交时执行,三级 `≥`,允许超额(AC-13):

| 规则 | 判定 | 失败信息 |
|------|------|---------|
| CV-1 | Σ 区域分配 ≥ 总目标 | level=REGION 汇总,gap=总目标−合计 |
| CV-2 | 每区域下 Σ(Dealer/门店/Promoter) ≥ 该区域目标 | 具体区域名 + gap |
| CV-3 | 每对象下 Σ 机型 ≥ 该对象目标 | 具体对象名 + gap |

- 任一不过 → 抛 `VALIDATION_FAILED`,`failures[]` 全量返回(不短路,一次列全,AC-12),status→VALIDATION_FAIL。
- **AC-21**:校验前先扫 `t_split_anomaly` 未处理 RED(is_resolved=0 且 severity=RED),存在即拦截并列出。
- **AC-14 0 误判**:纯数值比较、边界 `=` 视为通过(≥),用边界值用例回归。
- 通过 → status=VALIDATION_PASS,移交 workbench(见时序)。

---

## 异常识别(AnomalyDetector)

### 一期有效 3 类

| 类型 | 触发 | 级别 | 处置 |
|------|------|------|------|
| INTERVAL_DEVIATION(偏离 AI 区间) | 拆分值(含人工调整)超出该对象参考区间 `[初稿×(1−interval_ratio), 初稿×(1+interval_ratio)]` | RED | 标红、提交禁用,**填写调整原因**后可提交(AC-15) |
| HISTORY_DEVIATION(偏离历史过大) | `|当前值 − 近3月均值| / 近3月均值 > 50%` | RED | 标红,展示 均值/当前/偏差%,**人工确认处理**后可提交(AC-16) |
| TOTAL_IMBALANCE(合计不平衡) | 下级合计 < 上级(实时计算) | RED(自动提示) | 拆分页实时显示合计/差额,由强校验 CV-1/2/3 兜底禁止提交 |

> **设计决策 D-1(偏离 AI 区间的对象级口径)**:AI 推荐区间原为**总目标级**。一期把每对象的参考区间派生为「该对象 AI 初稿分配值 × (1 ± interval_ratio)」,人工调整后超带即触发红色。此为模块内合理解释,不涉及锁定契约;若产品另有口径(如仅总目标层拦截),按产品结论微调,不影响门面签名。

### 明确不实现(界面不显示、无任何逻辑路径)

- **超市场容量**、**新品供应风险**:一期数据源缺失,整体降级二期(AC-18)。AnomalyDetector 无对应枚举/分支。
- **清尾风险**:一期不做(AC-17,OQ-01 已闭环)。无枚举、无推算历史清库速度逻辑。

### 红色分级处置与留痕(6.2/6.3,AC-19/21；一期仅红色)

- RED:强制拦截提交,直至 `is_resolved=1`(填原因/人工确认)。
- YELLOW(二期预留，一期不产生):标黄提示,`bypass_confirmed=1` + confirmed_by/at 记录"确认知晓"日志后可绕过。
- 所有人工调整(含无异常主动修改)写 `t_split_adjust_log`:操作人/时间/调整前值/调整后值/原因/关联异常类型(AC-19)。

---

## LLM 业务依据(forecastBasisAgent,HITL)

- 预测编排在三档数值就绪后,调 `agent` 层 `forecastBasisAgent` 生成 `basisText`(解释影响预测的关键因子,中/英随 locale)。输入经**脱敏**:品牌/类型/周期/基准/三档/历史统计(对象以 ID 代名,无 PII),经 `PiiRedactionMiddleware` 出境(ADR-0004 唯一出口)。
- 每 call 必传 `RuntimeContext(userId, sessionId="plan-"+targetPlanId)` + `GoalAgentContext`;工具 `history_stat` 经 `HistoryDataQueryFacade` 只读取数,`market_intel_search`(D-5)只读查市场情报,Reactor 线程内重建 ThreadLocal、`finally` 清理。
- **HITL**:依据文本随三档展示,人工确认总目标时把关;Agent 不自动决策。
- **失败降级(不阻塞数值链路)**:LLM Key 缺失/调用异常/空输出 → `basisText` 回退**规则模板文案**(如按充分性/三档拼装的固定话术),status 仍按数值链路走 COMPLETED/DEGRADED,不因文本失败标 FAILED。
- 文本生成走 `agent.stream(msgs, options, rc)` SSE 推前端;`doOnError`+外层兜底保证 `emitter.complete()`。

### 输出语言风格要求(D-6,对齐产品 Demo 的经营参谋语感)

Demo 展示的 AI 依据文本是"综合近 6 月 SO 趋势与新品 Spark 爬坡,高 DOS 机型已降权,达成率预估 92%"这类**可执行的经营语言**,而不是"基于统计模型生成推荐值"这类技术黑箱表述。Prompt 设计须遵循:

- **素材边界不变,只改表达**:basisText 仍只能引用编排层传入的规则计算结果(基准值/三档/历史统计/充分性等级/market_intel_search 提示),不得引入未传入的数据或数值,不改变 §3.1 System Prompt 边界。
- **句式对齐 Demo**:优先"综合 X 因子、Y 因子,某档预估达成率 N%"或"因 Z(如高 DOS/雨季/新品爬坡),建议关注/调整"的结构,避免"模型输出""统计显著"等技术措辞。
- **降级模板文案同步要求**:LLM 失败时的规则模板文案也按此风格预置(如"近 6 月均值 X,高 DOS 机型降权,推荐档预估达成率 Y%"),而非"AI 服务暂不可用"这类纯技术提示——保证降级路径下用户体验一致。

**示例(保守/推荐/挑战三档,对齐 Demo 措辞)**:
- 保守档:"兜底节奏,近 6 月均值为基准,高 DOS 机型已降权,预估达成率 96%。"
- 推荐档:"综合 SI/SO/DOS 与季节性,新品 Spark/Camon 已加权,预估达成率 92%。"
- 挑战档:"新机加权、区域上探、中高端投放放大,预估达成率 84%,建议结合渠道承接能力确认。"
- 市场情报提示(命中 market_intel_search 时追加一句):"达卡雨季(6–8 月)线下客流历史走弱,建议关注节奏是否需要调整,最终以人工确认为准。"

---

## 关键时序

### 创建期主流程(目标制定,异步预测 + 前端轮询)

```
workbench 建目标草稿(TargetPlanFacade)
        │
        ▼ ① workbench 转调 PredictionFacade.predict(异步)
   PredictionOrchestrator(后台线程池):
     a. HistoryDataQueryFacade.checkAvailability → 充分性等级(FULL/PARTIAL/LIMITED)
      b. BigDataForecastClient.baseline(返回 LIMITED → DEGRADED;超时/无返回/调用异常 → FAILED)
     c. TierRuleEngine 三档+区间(AC-05 单调守卫;违反→FAILED)
     d. forecastBasisAgent 生成 basisText(失败→模板,不阻塞)
     e. 写 t_target_prediction(is_current=1),status=COMPLETED/DEGRADED/FAILED
        │
        ▼ ② 前端轮询 workbench(GET /plans/{id}/prediction)→ getPrediction(status 驱动 UI)
        │     FAILED/DEGRADED:提示 + 允许跳过 AI 人工输入(SKIP_AI 留痕,AC-04)
        ▼ ③ 人工拍板总目标(workbench 模块 confirm-total,HITL,不在本模块)
        │
        ▼ ④ workbench 转调 SplitFacade.split(confirmedTotal, balanceMode)
   SplitEngine 逐层拆分 → 写 t_split_task(DRAFT)+t_split_item → AnomalyDetector 跑异常
        │     无规则配置 → SPLIT_NO_RULE(AC-08)
        ▼ ⑤ 前端展示拆分树+异常;人工调整某层值 → SplitFacade.rebalance(锁定不动)
        │     锁定和≥上级 → LOCK_CONFLICT(AC-10);平衡后重跑异常
        ▼ ⑥ 人工处理红/黄异常(resolve:填原因/确认处理/知晓绕过 → 留痕)
        ▼ ⑦ 提交 → SplitFacade.validate(CV-1/2/3 + 未处理RED拦截)
        │     不过 → VALIDATION_FAILED(层级名+差额)→ 回⑤
        ▼ ⑧ 通过 → status=VALIDATION_PASS,移交 workbench 处理确认与发布(本模块止步于"待发布")
```

### 并发与原子性(NF)

- **单拆分任务闸**:`t_split_task` 对 `target_plan_id` 唯一键 + status 乐观锁,保证同计划同时刻仅一个拆分任务运行。
- **同层并发修改**:`t_split_item.version` `@Version` CAS,后提交者冲突→提示重载,防覆盖。
- **原子性**:`split`/`rebalance`/`validate` 各在单数据库事务内完成(task+item+anomaly+log 同事务),不出现部分写入。
- **异步预测隔离**:predict 在专用线程池执行,不阻塞 Web 容器线程;LLM `block()/blockLast()` 亦入专用池(踩坑 §15.5)。

---

## 非功能设计

- **性能**:单次预测 ≤ 30s(异步 + 前端轮询;超 60s 提示失败允许手填);多级拆分 ≤ 30s(孟加拉一期数据量,单事务批量写)。弱网(延迟≤3s)核心可用——预测异步化后前端只需轮询短响应。
- **安全**:国家/品牌/角色/经销商四级权限隔离。越权统一返回空集,不返 4xx、不暴露存在性(契约基线);forecast 门面被 workbench 调用,权限过滤在 workbench Controller 入口 + 本模块查询按授权维度过滤。LLM Key 走 Apollo 不硬编码;出境必经 PiiRedaction。
- **原子性/一致性**:见并发段;预测与拆分写入保证同计划不部分写入。
- **审计/留痕**:`t_split_adjust_log`(人工调整)+`t_target_prediction` 追加快照(预测+规则版本)+`LlmAuditMiddleware`(LLM 调用,不落敏感原文)。
- **可观测性**:OtelTracingMiddleware 贯穿 Agent 调用;predict/split 关键阶段(充分性/基准/三档/拆分/校验)打点耗时与降级标记,便于定位 30s 预算。
- **多语言**:basisText 与界面提示中/英,随 `Accept-Language`/locale;规则模板文案双语内置。
- **目标值边界**:所有分配/三档值 ≥0,不支持负数。

---

## 前端契约(Vben,经 workbench REST 转调)

> forecast 无独立 REST;前端经 workbench 的「创建/拆解工作台」资源(契约基线 §三)。字段沿用 PredictionResultDTO / SplitResultDTO。

| 前端能力 | REST(workbench) | 转调 | 关键字段/权限点 |
|---------|-----------------|------|----------------|
| 查看三档预测+依据 | `GET /goal/plans/{id}/prediction` | getPrediction | 六字段+confidence+basisText+status;权限 `goal:prediction:view` |
| 跳过 AI 人工输入 | `POST /goal/plans/{id}/confirm-total`(skip 标记) | — | `goal:prediction:skip`(留痕 SKIP_AI) |
| 触发/查看拆分 | `GET /goal/plans/{id}/split` | getSplitResult | items 树+anomalies;`goal:split:view` |
| 生成拆分 | (确认总目标后) → split | split | `goal:split:run` |
| 改值/平衡 | 前端改值提交 → rebalance | rebalance | `goal:split:edit`、锁定 `goal:split:lock` |
| 处理异常 | `POST /goal/plans/{id}/split/{itemId}/resolve` | 异常处理 | 一期 RED 填原因/确认(YELLOW 知晓为二期预留);`goal:anomaly:resolve` |
| 提交(强校验) | `POST /goal/plans/{id}/publish` 前置 validate | validate | `goal:split:submit` |
| 三档预测规则(保守/推荐/挑战档配置) | `GET/PUT /goal/config/split-flow` 扩展 | 规则配置 | 管理员 `goal:split:config` |
| 拆分权重规则(方案 B,本模块独立后台) | 见下「拆分规则配置后台(AC-08b)」独立路由 `/goal/config/split-rule` | — | 系统管理员 `goal:split-rule:manage` |

- UI 需固定标注"仅供参考,最终目标由您决定"(AC-20),无"AI 已自动确认"表述。
- LIMITED/PARTIAL/FAILED 分别按 §三档降级映射展示提示;RED 异常存在时提交按钮禁用并列未处理项(AC-21)。

---

## 拆分规则配置后台(AC-08b,方案 B)

> 拆分权重规则(`t_split_rule_config`)归本模块独立维护,**不并入** workbench 的流程配置 `/goal/config/split-flow`(那里只管"是否拆解/拆解至层级/是否拆到机型"的流程编排)。权重属拆分算法参数,与拆分逻辑内聚,故独立后台。仅**系统管理员**可增改。

### 接口(admin REST,workbench 网关路由 → 本模块 Controller)

forecast 无独立进程,此后台走 workbench 前端调用、由 workbench Controller 转调本模块的 `SplitRuleConfigService`(应用内 Java 调用,非门面契约、非 Feign)。路由沿用 `/goal/config/*` 风格,资源名 `split-rule`:

| 能力 | REST | 说明 | 权限点 |
|------|------|------|--------|
| 列表/查询配置 | `GET /goal/config/split-rule?countryCode=&brandCode=&targetType=` | 按国家×品牌×目标类型过滤;不传维度返当前生效全集 | `goal:split-rule:manage` |
| 查看单条(含历史版本) | `GET /goal/config/split-rule/{id}` | 返当前+历史 config_version 快照 | `goal:split-rule:manage` |
| 新建配置 | `POST /goal/config/split-rule` | 唯一键 `uk_scope`(国家+品牌+目标类型)已存在则拒绝,改用更新;`ruleTier=P1` 要求 country_code/brand_code 为具体值,`ruleTier=P3` 要求两者均为通配值 `*`(D-4,校验放行) | `goal:split-rule:manage` |
| 更新配置 | `PUT /goal/config/split-rule/{id}` | 变更即递增 `config_version`;权重强校验 | `goal:split-rule:manage` |
| 启停配置 | `PUT /goal/config/split-rule/{id}/enabled` | 切换 `enabled`;停用后该场景触发拆分按 `SPLIT_NO_RULE` 阻止(AC-08) | `goal:split-rule:manage` |

请求/响应 DTO:`SplitRuleConfigDTO`(country_code/brand_code/target_type/**rule_tier**/split_levels/weight_historical_rate/weight_capital/weight_territory/historical_periods/balance_mode/config_version/enabled),置 **common** 包(契约唯一来源)。错误统一走 `GoalBizException(GoalErrorCode.VALIDATION_FAILED)`。

### 入库校验(强校验,对齐 AC-08b)

保存(POST/PUT)时,`weight_historical_rate + weight_capital + weight_territory = 100`,否则拒绝保存并返 `VALIDATION_FAILED`,提示"三项权重之和必须为 100%"。三项均须 0-100 整数。balance_mode 须为枚举值之一。

### 审计

配置变更写操作日志,复用现有留痕设计的"只增不改"约定,新增 `t_split_rule_config_log`(与 `t_split_adjust_log` 同风格,分表因维度不同):

| 字段 | 说明 |
|------|------|
| id / rule_config_id | 关联配置 |
| operator / operate_time | 操作人 / 时间 |
| action | CREATE / UPDATE / ENABLE / DISABLE |
| scope_snapshot | 配置维度(country_code/brand_code/target_type) |
| before_value / after_value | 变更前后权重+balance_mode(JSON;CREATE 时 before 为空) |
| config_version | 本次生效版本号 |

### 生效范围与版本

- `config_version` 每次变更(UPDATE/启停)递增,历史配置以日志快照留存可回溯。
- 新建目标触发拆分时,`SplitFacade.split` 读取当前生效(`enabled=1`)配置的 `config_version`,写入 `t_split_task.split_rule_version` 留痕(该字段已存在)。
- **已生成的拆分结果不追溯**:改配置只影响变更后新建目标的拆分,不回改历史 `t_split_item`。

---

## 与其他模块的依赖

- **dataingest**:`HistoryDataQueryFacade`(checkAvailability/queryHistory)——充分性、历史均值/销量/达成率。
- **workbench**:`TargetPlanFacade`(读计划元信息);workbench 转调本模块 Prediction/SplitFacade,并承接 confirm-total/publish。
- **大数据(外部)**:`BigDataForecastClient.baseline`(Feign)。
- **agent**:`forecastBasisAgent`(Bean),`ForecastTools.history_stat`(工具,经门面读数)。
- **对象/机型列表**:复用 dataingest 已接入的 Territory/Product 结果,**不新增重复 Feign**(契约基线约定)。

---

## 遗留标注与占位处置(待对齐)

| 编号 | 遗留 | 影响 | 一期占位处置 |
|------|------|------|-------------|
| OQ-02 | 资金盘数据来源接口(DRP/其他) | 拆分权重 weight_capital(AC-07) | 复用 dataingest `DrpClient.capitalPool` 结果经门面取;未通时该权重归零重归一,提示降级,拆分照跑 |
| OQ-03 | 阵地数数据来源接口(DCR 人员/门店) | 拆分权重 weight_territory(AC-07) | 复用 dataingest `TerritoryClient` 结果经门面取;未通同上归零重归一 |
| F2 / OQ-04 | 大数据 baseline 字段/SLA、置信度来源(随基准 or 规则映射) | 三档生成、confidence、失败/降级路径 | 契约结构占位(ForecastBaselineRequest/Response),字段标"待大数据团队确认";confidence_source 配置项二选一;数据不足走 DEGRADED,超时/无返回走 FAILED |
| OQ-5 | 复盘不在本模块,但 forecastBasisAgent prompt 边界与 reviewAgent 共用规范 | LLM 输出边界 | 沿用契约基线 §3.1 System Prompt 边界(不输出数值决策/不虚构/不含 PII);basisText 仅解释因子 |
| D-1 | 偏离 AI 区间对象级口径(总目标级区间派生到对象) | 异常 INTERVAL_DEVIATION 判定 | 一期按"对象初稿×(1±interval_ratio)"派生;待产品确认口径,不影响门面契约 |

---

## 关联

- `docs/design/00-契约基线-接口清单.md`(PredictionFacade/SplitFacade 签名、DTO 归属)
- `docs/design/00-契约基线-AgentScope与预测.md`(Agent/工具/三档公式/预测基准契约/降级映射)
- 下游 workbench 设计文档:confirm-total/publish、前端工作台 REST 展开
