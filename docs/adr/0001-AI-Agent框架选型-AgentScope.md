# ADR-0001：AI / Agent 框架选型 AgentScope Java（一期即引入 LLM + HITL）

- 状态：已采纳
- 日期：2026-07-02
- 关联：`docs/requirements/待澄清问题清单.md`（决策 F1）、`docs/reference/agentscope-java-guide.md`

## 背景

AI 目标管理平台的 AI 能力包含两类：(1) 数值预测（见 ADR-0002，已外部化）；(2) LLM 类能力——业务语言依据、复盘报告、策略建议，以及未来的目标规划/风险预警等 Agent 编排。

PDF 原稿主张"一期不依赖大模型、Agent 放二期"（风控考虑）。但需要一个统一的 Agent 框架承载 LLM 能力与后续 Agent 编排，且须与内部 Hulk（Spring Cloud Netflix，Java）技术栈一致。团队规模小（1 PM + 2 前端 + 3 后端 + 1 测试），不宜自研 Agent 基座。

## 选项

1. **AgentScope Java 2.0.0-RC3（阿里通义）** — 优点：Java 原生（JDK 17+/Reactor），与 Hulk/Spring 以单例 Bean 集成无缝；内生 ReAct/工具/HITL/记忆/权限/流式；仓库内已有参考实现 `com.transsion.fpm.rebate.agent.agentscope`；模型层可切 DashScope/OpenAI/Anthropic 等。缺点：RC 版（需留意升稳定版）；需 LLM Key，无离线兜底。
2. **自研 / 直接裸调 LLM SDK** — 优点：轻。缺点：ReAct 循环、工具、HITL、记忆、权限全要自建；重复造轮子，团队负担重。
3. **一期完全不引入 LLM（沿用 PDF 原稿）** — 优点：无 LLM 成本/合规负担。缺点：业务依据/复盘/策略只能规则+模板硬编，价值弱；二期切换框架有返工。

## 决策

选 **选项 1：AgentScope Java 2.0.0-RC3**，且**一期即引入 LLM（方案 A）**：

- LLM Agent 承载"业务语言依据、复盘报告、策略建议"文本能力与流程编排。
- **全程人工确认（HITL）**：AI 只做建议，所有目标/审批决策由人工拍板，符合"AI 不替代决策"的风控底线。
- 入口用 `agentscope-harness`；Agent/Model/Toolkit 做成 Spring 单例 Bean，业务工具做 `@Component`；每次 `call` 必传 `RuntimeContext(userId, sessionId)` 做多租户隔离。
- 模型 Provider 一期选型与 LLM 数据出境合规绑定，见 ADR-0004。

此决策修订 PDF 原稿及澄清 D1"一期不依赖大模型"的表述。

## 后果

**正面**：AI 能力与 Java 技术栈统一；HITL + 权限系统天然满足风控；二期 Agent 编排（目标规划/风险预警等）在同一框架内平滑扩展，无返工。

**负面 / 代价**：
- 引入 LLM 依赖：需 LLM API Key，AgentScope 无离线兜底，**调用时才失败**——须在调用处（SSE/接口）兜异常并降级（无 LLM 输出时回退到规则/人工）。
- LLM 调用成本（token/并发）需评估与治理。
- RC 版本风险：锁定 `2.0.0-RC3`，关注升稳定版；Provider 模型已迁出 core，须引 `agentscope-extensions-model-*`。

**后续待办**：
- ADR-0004 定 LLM Provider 与数据出境合规方案。
- 设计期定 Agent 清单、工具边界、system prompt、StateStore（生产用 Redis）与并发模型。
- 阻塞调用（`block()/blockLast()`）放专用线程池，勿阻塞 Web 容器线程。
