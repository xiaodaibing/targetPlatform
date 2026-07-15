# 技术设计:契约基线（三）AgentScope 集成与数值预测

> 版本：v1.0 | 日期：2026-07-03 | 作者：architect
> 定位：gate #1 产物。落地 ADR-0001（AgentScope + HITL）、ADR-0002（预测来自大数据团队 + 本系统规则生成三档）、ADR-0004（LLM 出境合规）。
> 依据速查：`docs/reference/agentscope-java-guide.md`（AgentScope Java 2.0.0-RC3）。仓库参考实现 `com.transsion.fpm.rebate.agent.agentscope`。

---

## 1. AI 能力二分（务必先分清）

| 能力 | 归属 | 是否用 LLM | 一期 |
|------|------|-----------|------|
| **数值预测基准** | 大数据团队（Feign 实时调用） | 否（大数据侧建模） | 是 |
| **三档（保守/推荐/挑战）+ 推荐区间生成** | 本系统**规则引擎**（非 LLM，纯计算） | 否 | 是 |
| **业务语言依据文本**（预测） | 本系统 **AgentScope LLM Agent** | 是 | 是（HITL） |
| **复盘优化建议文本** | 本系统 **AgentScope LLM Agent** | 是 | 是（HITL） |
| **市场情报定性提示**（雨季/竞品/汇率/关税等，D-5 新增） | 本系统 **AgentScope LLM Agent + 搜索工具** | 是 | 是（HITL，**仅文本提示，不进任何数值计算**） |
| 达成率/异常判定/Gap/强校验 | 本系统规则/数值计算 | 否 | 是 |
| 策略建议 / 目标规划 / 对话助手 / 多 Agent 编排 | AgentScope | 是 | **二期** |

> 底线：**LLM 只生成文本、只做建议，不参与任何数值决策，全程 HITL 人工确认**（ADR-0001）。数值全部来自大数据基准 + 本系统规则，与 LLM 解耦——LLM 失败时数值链路不受影响，仅文本降级为规则模板/留空待人工。**市场情报工具查询到的汇率/关税等数字同样不进数值计算**（网搜结果不可审计，进数值链路会破坏"禁伪精确"红线，见 §3.2）。

---

## 2. AgentScope 集成架构（`com.transsion.goal.agent`）

### 2.1 单例 Bean 装配（Spring/Hulk）

```java
@Configuration
class GoalAgentConfig {
  // 模型：Provider 按 ADR-0004 合规选型（DashScope / 私有化 OpenAI 兼容网关），经 ModelRegistry 可切换
  @Bean Model llmModel(@Value("${goal.llm.provider}") String provider,
                       @Value("${goal.llm.api-key:}") String key,
                       @Value("${goal.llm.model-name}") String modelName) { /* build ChatModel */ }

  // Toolkit：注册各业务模块的 @Component 工具（forecast/tracking 提供）
  @Bean Toolkit goalToolkit(List<Object> agentTools) { /* registerTool each */ }

  // 生产用 Redis 状态存储（多副本共享，跨进程恢复）
  @Bean AgentStateStore stateStore(JedisPooled jedis) {
    return RedisAgentStateStore.builder().jedisClient(jedis).keyPrefix("goal:agent:").build();
  }

  // Agent：每类文本能力一个专用 Agent（无状态引擎，单实例服务全部用户）
  @Bean("forecastBasisAgent") ReActAgent forecastBasisAgent(Model m, Toolkit tk, AgentStateStore ss) {
    return ReActAgent.builder().name("forecast-basis").sysPrompt(FORECAST_BASIS_PROMPT)
        .model(m).toolkit(tk).stateStore(ss)
        .middlewares(List.of(new PiiRedactionMiddleware(), new LlmAuditMiddleware(), new OtelTracingMiddleware()))
        .maxIters(6).build();
  }
  @Bean("reviewAgent") ReActAgent reviewAgent(...) { /* 复盘优化建议 */ }
}
```

### 2.2 每次 call 必传 RuntimeContext（多租户隔离，踩坑 §15.7）

```java
RuntimeContext rc = RuntimeContext.builder()
    .userId(currentUser.getId())            // 归属/权限隔离
    .sessionId("plan-" + targetPlanId)      // 同 (userId,sessionId) 自动加载/保存状态
    .put(GoalAgentContext.class, goalCtx)   // 类型化注入给工具（见 §3）
    .build();
```
- 相同 `(userId,sessionId)` 自动 FIFO 串行；不同的并行 —— 无需自加锁。
- **流式（SSE）需 per-call 上下文时用 `agent.stream(msgs, options, rc)`**，勿用 `streamEvents(msg)`（无 RuntimeContext 重载，会丢隔离，踩坑 §15.7）。
- `block()/blockLast()` 放**专用线程池**，勿阻塞 Web 容器线程（踩坑 §15.5）。

### 2.3 工具注入上下文（踩坑 §15.4：Reactor 线程 ThreadLocal 丢失）

业务 ThreadLocal（租户/请求头）不会传到工具线程，必经 `RuntimeContext.put(Class,..)` 注入，工具内重建、`finally` 清理：

```java
public record GoalAgentContext(String userId, String tenantHeader, String locale) {}

@Component @RequiredArgsConstructor
public class ForecastTools {
  private final HistoryDataQueryFacade historyFacade;   // 经门面读数据接入模块

  @Tool(name="history_stat", description="取对象近N期历史达成/销量统计", readOnly=true)
  public String historyStat(@ToolParam(name="objectId",description="对象ID") String objectId,
                            @ToolParam(name="periods",description="期数") int periods,
                            GoalAgentContext ctx) {          // 框架从 RuntimeContext 注入
    try { TenantHolder.set(ctx.tenantHeader());             // 重建 ThreadLocal
          return JsonUtil.toJson(historyFacade.queryHistory(/* by objectId */)); }
    finally { TenantHolder.clear(); }
  }
}
```
> 工具通过**门面接口**读数据（不直连别模块表）；数值预测基准工具见 §5.2。

### 2.4 市场情报搜索工具（D-5 新增，agent 层公共工具，不归属任何单一 facade）

> 对齐产品 Demo 的市场事件因子（竞品价格/活动、汇率、关税、雨季客流等）。**只产出定性文本、不进任何数值计算**——数值因子（汇率/关税等）若要参与计算，须待接入确定性数据源（公开汇率 API / 公司金融数据）后二期实现，一期网搜结果**不可审计，严禁进数值链路**。

```java
@Component @RequiredArgsConstructor
public class MarketIntelTools {
  private final WebSearchClient searchClient;   // 经统一出站网关调用搜索服务（Feign 或封装 HTTP，受 §4 出境合规约束）

  @Tool(name="market_intel_search", description="查询区域市场事件（雨季/竞品动态/宏观事件），仅供业务参考，不产生数值", readOnly=true)
  public String marketIntelSearch(@ToolParam(name="query",description="查询关键词，如'孟加拉 雨季 客流'") String query,
                                  GoalAgentContext ctx) {
    // 查询不含 PII、不含渠道/门店/Promoter 等业务数据，仅通用市场关键词
    return searchClient.search(query);
  }
}
```

- **谁能用**：`forecastBasisAgent`（创建期，补充节奏类提示，如"雨季客流走弱，建议调整月度节奏"）、`reviewAgent`（复盘期，补充市场归因）均可注册该工具；两者共用同一实现，不重复开发。
- **红线**：输出文本仅作**风险提示**随业务依据/复盘建议展示，不写入 `t_target_prediction`/`ReviewReport` 的任何数值字段；查询关键词本身不含 PII 或渠道业务数据（合规压力小于业务数据出境，但仍走统一出站网关，见 §4）。
- **失败降级**：搜索无返回/超时 → 该条提示留空，不影响 basisText/复盘建议其余内容生成，不阻塞主流程。

---

## 3. Agent 清单（一期）

| Agent Bean | 承载能力 | 输入（脱敏后） | 输出 | 工具 | HITL |
|-----------|---------|--------------|------|------|------|
| `forecastBasisAgent` | 预测**业务语言依据**文本（中/英）：解释影响预测的关键因子；含市场节奏类提示 | 品牌/类型/周期/基准值/三档/历史统计（对象以 ID 代名，无 PII） | `basisText` | `history_stat`（只读，经 HistoryDataQueryFacade）、`market_intel_search`（只读，D-5） | 结果随预测展示，人工确认总目标时把关 |
| `reviewAgent` | **复盘优化建议**文本：基于达成/偏差生成业务语言建议；含市场归因类提示 | 各层级最终达成率/偏差/超阈对象（ID 代名） | 建议摘要文本 | `review_stat`（只读，经 tracking 内部达成汇总）、`market_intel_search`（只读，D-5） | 生成后**人工审阅**再随报告展示（OQ-5 边界） |

> 二期新增：策略建议 Agent、目标规划 Agent、对话助手、多 Agent 编排——在同框架平滑扩展，本期不建。

### 3.1 System Prompt 边界（OQ-5，防跑偏/防敏感泄露）

- 明确"你只生成业务语言解释/建议，不输出任何目标数值决策，不虚构数据"；输入外无数据的字段一律不臆测。
- 复盘建议限定在"达成合理性评估、跟进建议"，不含具体人员绩效指控、不输出 PII。
- 输出语言随 `locale`；失败/空输出时上层降级为规则模板文案，不阻塞报告/预测。
- **市场情报提示的额外边界**（D-5）：`market_intel_search` 工具返回内容仅可用于生成"风险提示"类表述，Prompt 须明确禁止把搜索到的数字（汇率/关税等）当作目标数值依据输出——只能是"建议关注/建议调整节奏"类定性表述，不得出现"按最新汇率计算应调整为 X"这类把网搜数字包装成计算依据的表述。

---

## 4. 出境合规与脱敏（ADR-0004，落地到中间件）

一期试点孟加拉，渠道/门店/Promoter 数据可能含 PII，发往 LLM 存在出境风险。统一在 `agent` 层中间件处理，业务模块无感：

| 中间件 | Hook | 职责 |
|--------|------|------|
| `PiiRedactionMiddleware` | `onSystemPrompt` + `onModelCall` | **出站脱敏/最小化**：人名/手机/账号等 PII 以稳定 ID 代替；只送生成任务必需字段。是数据出境的唯一出口，统一红线管控。 |
| `LlmAuditMiddleware` | `onModelCall` | LLM 调用审计留痕：谁/何时/哪个 Agent/token 用量/是否降级；不落敏感原文（对齐 8.4：不输出密码/密钥/令牌）。 |

- **Provider 选型受合规结论约束**：`ModelRegistry` 支持后置切换；PII/敏感聚合数据优先走**区域化/私有化模型**，纯模板文本可评估公有云（混合方向，ADR-0004）。**上线前必须定 Provider**。
- 保留"不调用 LLM 也能降级运行"路径（数值链路与 LLM 解耦）。
- LLM 无离线兜底，Key 缺失/调用异常**调用时才失败** → 在 SSE/调用处兜异常，回退规则模板/人工（踩坑 §15.3）。

### 4.1 市场情报搜索工具的出站说明（D-5 新增，另一条独立出口）

`market_intel_search`（§2.4）是**区别于 LLM Provider 调用的另一条出站通道**——查询关键词发往搜索服务，而非发往 LLM。合规画像与业务数据出境不同：

- 查询内容仅为通用市场关键词（国家/品类/季节/宏观事件），**不含渠道/门店/Promoter/PII**，不适用 `PiiRedactionMiddleware` 脱敏对象范围，但仍须**统一经出站网关调用、留调用日志**（谁/何时/查询词/是否成功），并入 `LlmAuditMiddleware` 同类审计范畴，不额外新建审计表。
- ADR-0004 的评估范围**需扩展**覆盖该出口：确认搜索服务提供方是否受同等跨境合规约束（详见 ADR-0004 补充）。

---

## 5. 数值预测（ADR-0002）

### 5.1 大数据团队实时预测基准接口契约（`BigDataForecastClient`，Feign）

> **字段/SLA 待与大数据团队钉死（F2/OQ-04）**，此为契约结构占位。

```
POST /bigdata/forecast/baseline    （经 Eureka 发现，实际路径待定）
ForecastBaselineRequest:
  countryCode, brand, targetType, targetPeriod, objectType, modelScope?(机型)
ForecastBaselineResponse:
  baselineValue(BigDecimal)          # 预测基准值
  confidence?(0-100)                 # 置信度：随基准返回 or 本系统规则映射（OQ-04 二选一，待定）
  caliber?(String)                   # 口径标识
  dataAvailability?(FULL/PARTIAL/LIMITED)  # 大数据侧数据状态（可选）
  predictionSource, generatedAt      # 来源标识 + 生成时间（写入预测留痕，可回溯）
```
- SLA（延迟/可用性/限流）待定；调用超时兜底：**无预测/超时降级为人工设定**，不阻塞创建流程（AC-04）。
- 本系统**无需向大数据开放数据出口**（OQ-DI-01 闭环，大数据用自有数据湖）。

### 5.2 三档规则引擎（本系统能力，非 LLM）

在基准值上按**可配置规则**生成三档 + 推荐区间，规则按 `国家 × 品牌 × 目标类型` 配置（`SplitRuleConfig` 同源思路）：

```
recommendValue  = baselineValue
conservativeValue = baselineValue × (1 - conservativeRatio)   # 配置，如 0.9
challengeValue    = baselineValue × (1 + challengeRatio)      # 配置，如 1.15
intervalLower / intervalUpper = 按配置带宽（如 ±intervalRatio）
confidence     = 随基准返回 或 由数据充分性映射（OQ-04 定）
```
- **强约束（AC-05）**：保守 ≤ 推荐 ≤ 挑战，下限 ≤ 推荐 ≤ 上限；违反 → 预测整体标 FAILED，降级人工。
- 三档规则版本写入预测快照（`prediction_source` + 获取时间 + 规则版本 + 输出值，AC-22）。

### 5.3 数据充分性 → 置信度降级映射（禁伪精确，AC-02/03）

| 充分性（大数据返回 或 本系统 checkAvailability） | 展示 | 置信度约束 |
|--------|------|-----------|
| FULL（近12月无重大缺口） | 正常三档 + 真实置信度 | 正常 |
| PARTIAL（6–12月/字段有缺口） | "数据部分可用，置信度受影响" | 降低 |
| LIMITED（<6月/关键字段缺失） | "数据不足，建议结合业务经验人工设定" | **不得虚高**，禁伪精确 |

> 大数据标注数据不足 → 按 LIMITED 降级为"建议人工设定"（DEGRADED）；大数据未返回预测/超时/调用异常 → 记为 FAILED 并允许人工设定。两种情况都不阻塞创建流程（ADR-0002）。

---

## 6. HITL 与 SSE（预测/复盘文本生成）

- 文本生成经 `agent.stream(msgs, options, rc)` → SSE 推前端（token 流 `TextBlockDeltaEvent`，RC3 可从 `AgentResultEvent` 直取最终结果）。
- **决策卡点在业务流程，不在 Agent 自动确认**：预测三档 + 依据展示后由创建人拍板总目标；复盘建议人工审阅后随报告发布。AgentScope `RequireUserConfirmEvent` 一期文本生成场景可不启用工具级确认（工具均只读），HITL 体现在**业务确认步骤**。
- 异常兜底：`doOnError` + 外层 try 保证 `emitter.complete()`；LLM 失败回退规则模板文案（踩坑 §15.11）。

---

## 7. 生产要点清单（对齐踩坑 §15）

1. 版本锁 `2.0.0-RC3`；Provider 模型迁出 core，引 `agentscope-extensions-model-*` + formatter 同步。
2. Redis `AgentStateStore`（生产首选，跨进程恢复）；分布式文件系统模式勿配本地 StateStore（build 会失败，§15.9）。
3. 每次 call 必传 `RuntimeContext(userId,sessionId)`；流式用 `stream(...,rc)`。
4. 工具内 ThreadLocal 重建 + `finally` 清理；工具经**门面接口**读数据，不直连别模块表。
5. `block()/blockLast()` 入专用线程池；SSE 异常兜底。
6. LLM Key 走 Apollo，不硬编码；出境数据必经 `PiiRedactionMiddleware`。
7. 中间件/工具读 AgentState 用 `RuntimeContext.resolveAgentState(ctx, agent)`（并发安全，§15.8）。

---

## 8. 关联

- `00-契约基线-总览与边界.md`、`00-契约基线-接口清单.md`
- 后续：forecast 模块设计文档展开三档规则配置模型、拆分/平衡/强校验算法细节；tracking 模块设计文档展开复盘报告结构与回流。
