# 项目工作手册 (CLAUDE.md)

> 本文件是项目的"宪法"。Claude Code 在每次会话开始都会读取它。
> 本文件为可复用模板:落地到具体项目时,把 `CLAUDE.md` 放到项目根目录,替换全部〈...〉占位符,并按项目实际情况增删各小节。

## 1. 项目概览
- 名称: AI 目标管理平台
- 一句话目标: 面向传音(TECNO/itel/Infinix)全球多国渠道,把 SI/SO/ST 销售目标的**制定→拆解→确认→发布→执行追踪→复盘**做成 AI 辅助的全流程闭环,给国家/品牌经理、区域经理(RSM)、渠道执行层(Dealer/门店/Promoter)使用;一期孟加拉试点,品牌 TECNO/Infinix。详见 `docs/product-vision.md`。
- 架构形态:**单体应用 + 公司注册中心集成**(决策见 ADR-0003)。仓库仅保留**一个前端项目 + 一个后端项目**。业务按**包级模块(有界上下文)**划分;**模块间经应用层门面接口(facade)协作**;对**外部系统**经 **Eureka 注册发现 + OpenFeign** 调用。共享契约置于 **common 包**;**禁止模块间直连对方表,禁止用 Feign 调本应用内部模块**。模块划分见下方清单。
- 技术栈:
  - 后端:基于 **Spring Cloud Netflix** 的公司内部微服务框架 **Hulk**(`com.transsion.hulk`)。父 BOM:`com.transsion.hulk:hulk-parent:4.0.0.RELEASE`;能力以 `hulk-framework-starter-*` 形式引入(服务发现:`hulk-framework-starter-eureka`)。ORM 用 **MyBatis-Plus**。Java 版本以 Hulk 4.0 BOM 为准(待 architect 确认),构建用 **Maven**。后端应用 groupId 约定 `com.transsion.goal`,业务包按模块划分(如 `com.transsion.goal.<module>`)。
  - 数据/中间件:MySQL + Redis + ClickHouse(达成明细/大数据量查询)、Kafka。外部系统一律 **API 定时拉取,不做实时同步**。
  - **AI / Agent 框架**:**AgentScope Java 2.0.0-RC3**(`io.agentscope`,阿里通义;推荐入口 `agentscope-harness`,JDK 17+/Reactor,与 Hulk/Spring 以单例 Bean 集成)。速查手册见 `docs/reference/agentscope-java-guide.md`,仓库内参考实现 `com.transsion.fpm.rebate.agent.agentscope`。模型经 `ModelRegistry` 接 DashScope(通义千问)/OpenAI/Anthropic 等,**需 LLM API Key,无离线兜底**;工具做成 Spring `@Component` 再 `toolkit.registerTool`,每次 call 必传 `RuntimeContext(userId,sessionId)`。
  - **数值预测(决策 F2)**:预测**基准值实时调用大数据团队获取**,**三档(保守/推荐/挑战)与推荐区间由本系统按配置规则在基准上生成**;本系统**不自建 Prophet/SARIMA 等模型、不做特征工程/模型训练、不需本侧数据科学家**。大数据团队有自有数据湖(本系统无需开放数据出口),其实时预测接口契约与 SLA 待 ADR;无预测时降级人工设定。
  - **一期即引入 LLM(决策 F1,HITL)**:一期以 AgentScope LLM Agent 承载"业务依据/复盘/策略"文本能力与流程编排,数值预测为其工具,**决策全程人工确认,AI 不自动决策**。随之新增依赖:LLM API Key、调用成本、**海外数据出境合规**(渠道数据发往 LLM 需脱敏/合规评估)——详见 `docs/requirements/待澄清问题清单.md`。
  - 前端:基于 **Vben Admin**(Vue3 + Vite + TypeScript + Ant Design Vue,pnpm + monorepo + turbo)的公司内部前端框架。一期仅 **PC 端**;移动端沿用既有 **DCR APP(React Native,另有专门团队)**,本平台一期仅"触发通知",不建移动端。
- 业务模块清单(单体内包级模块,有界上下文;详见 `docs/requirements/<模块>.md`):
  1. **数据接入与达成计算** — 外部数据定时拉取、可用性分级与降级、SI/SO/ST 达成取数计算
  2. **AI 预测与拆分** — 三档预测、规则拆分、自动平衡、强校验、拆分阶段异常
  3. **目标协同管理(PC 双工作台)** — 目标创建/确认/发布、流程配置、版本留痕、四级权限隔离、通知触发
  4. **执行追踪与复盘** — 达成追踪、Gap 分析、执行异常与分级处置、基础复盘、结果回流

## 2. 文档地图(单人维持一致性的关键)
所有产物都在仓库里、由本节索引。开工前先读对应文档。

- 产品愿景:`docs/product-vision.md`
- 需求规格 + 验收标准:`docs/requirements/<模块>.md`
- 技术设计 + 接口契约:`docs/design/<模块>.md`
- 架构决策记录(ADR):`docs/adr/<编号>-<标题>.md`
- 验收清单:`docs/acceptance/<模块>.md`
- 上线清单:`docs/release-checklist.md`
- 一致性残留扫描清单:`docs/templates/consistency-scan-template.md`(需求/设计定稿前、口径变更后必跑)
- 文档模板:`docs/templates/`

## 3. 编码规范

### 3.1 后端(Hulk 内部框架)
- **优先用 Hulk 内部 starter,不直接引原生组件**:
  - 每个服务 pom 继承父 BOM:`com.transsion.hulk:hulk-parent:4.0.0.RELEASE`(版本由父 BOM 统一管理,业务 pom 不写 Spring/Spring Cloud 版本号)。
  - 按能力引入 `hulk-framework-starter-*`:服务发现用 `hulk-framework-starter-eureka`;其余 starter(web / feign / 配置中心 / 网关等)的确切名称以父 BOM 和**仓库里现有服务**为准,不要臆造包名。
- **调用边界二分**(单体应用):
  - **应用内模块间协作**走**应用层门面接口(facade)**——不用 Feign、不直连对方模块的表/Mapper。
  - **对外部系统**(DRP/DCR/激活/库存/产品库/假期日历/大数据团队/UAC 等)走 **OpenFeign 声明式客户端**(经 Eureka 发现),不手写 HTTP。
  - **共享 DTO/契约置于 `common` 包**,作为契约唯一来源(单体不拆 `*-api` 模块)。
- 服务发现走 Eureka;对外统一经网关;熔断/降级按内部框架约定〈Hystrix / 内部封装〉。
- 配置外部化(配置中心 Apollo),严禁把环境配置、连接串、密钥写进代码。本地开发可不连 Apollo,临时写在 `application.yml`,但不得提交真实密钥/生产配置。
- 构建/测试命令(在对应服务目录执行):
  - 编译打包:`mvn clean package`
  - 单元测试:`mvn test`
  - 集成 + 校验:`mvn verify`
- 测试:JUnit 5 + Mockito + Spring Boot Test;跨模块对门面接口做 mock;对外部系统对 Feign 客户端做 mock 或契约测试。
- 提交信息:如 Conventional Commits
- 禁止:密钥/连接串硬编码;吞异常;**跨模块直连对方表/Mapper**(必须经对方模块的门面接口);**用 Feign 调本应用内部模块**(Feign 仅对外部系统);对外部系统绕过 Feign 手写 HTTP。

### 3.2 前端(Vben Admin 内部框架)
- 包管理**强制 pnpm**(monorepo + turbo);Node 版本 〈如 ≥ 18〉。
- **组件 / 请求 / 权限一律用框架二次封装,不用原生**:
  - 用框架封装的请求 / useRequest(基于 axios)而非裸 fetch;接口地址走环境变量,不硬编码。
  - 用框架封装的组件与 hooks,而非直接堆 Ant Design Vue 原生组件。
  - 按钮级权限用框架的权限指令/组件,不要自己绕过权限直接渲染敏感操作。
- 一律 Composition API + `<script setup>` + TypeScript;状态用 Pinia;路由用 Vue Router;动态菜单/权限按框架约定配置。
- 命令(前端仓库根目录执行):
  - 安装:`pnpm install`
  - 开发:`pnpm dev`
  - 构建:`pnpm build`
  - 校验 + 修复:`pnpm lint`(ESLint / Prettier / Stylelint)
  - 类型检查:`pnpm type:check`
  - 单测:`pnpm test`(Vitest)〈若尚未接入则补一句"暂无前端单测"〉
  - 提交:`pnpm commit`;husky 会在提交时自动跑 lint
- 禁止:引入与框架重复的 UI/工具库;在组件里硬编码接口地址或绕过框架请求封装。

> 规范有任何变更,只改本节;所有子代理都会读到并遵守。

## 4. 工作流与角色

流程:产品 → PRD 评审 → 需求 → 技术设计 → 研发 → 测试 → 验收 → 上线。
- 产品(无子代理):在 chat / Claude Cowork 里思考发散,产出/更新 `docs/product-vision.md`,作为后续输入。
- PRD 评审 → `prd-review`(skill):对 `docs/reference/` 的 PRD 初稿做逻辑闭环体检,在主对话逐条追问产品并回填 `docs/requirements/待澄清问题清单.md`,可跨会话续跑;所有阻塞项闭环、产出"确认基线"后才放行需求。
- 需求 → `requirements-analyst`(消费 PRD 评审的确认基线)  | 技术设计 → `architect`
- 研发 → `implementer`  | 测试 → `test-engineer`  | 验收/审查 → `code-reviewer`  | 上线 → `release-engineer`

子代理定义见全局 `~/.claude/agents/`(跨项目共享);本项目已锁定的契约/决策/复审记录见 `.claude/agent-memory/<角色>/`,子代理动手前应先读自己的目录。

## 5. 必须遵守的关卡 (gates)
0. **PRD 评审先行**:产品给的 PRD 初稿先经 `prd-review` skill 做逻辑闭环体检;`docs/requirements/待澄清问题清单.md` 存在 🔴 阻塞项未闭环时,**不许进入 `requirements-analyst`**。阻塞项全部闭环、产出"确认基线"后方可拆需求。
1. **契约先行**:任何跨模块或对外部系统的改动,先让 `architect` 在 `docs/design/` 确认接口契约(模块间**门面接口** / 外部 **Feign 接口**)与 `common` 包 DTO;契约没定,不许写实现。
2. **验收标准可验证**:需求里每条验收标准都要能判真假,它就是后面的测试用例和验收清单。
2b. **一致性扫描**:需求/设计**定稿前**或任一决策口径变更后,按 `docs/templates/consistency-scan-template.md` 做一次关键词残留扫描(机械对照,不靠通读),把发现回填 `docs/requirements/需求审查遗留问题.md`。跨文档口径漂移靠通读几乎必残留。
3. **提交前必过审查**:涉及鉴权、支付、用户数据、跨模块/外部系统接口、权限控制的改动,提交前必须经 `code-reviewer`。
4. **成本意识**:并行(Agent Teams)约 15× token,子代理约 4–7×。包间/服务间耦合紧的工作默认串行;若工作天然独立(不同模块/服务、互不依赖),可考虑并行。

## 6. 给所有子代理的通用指令
- 动手前先读第 2 节指向的相关文档,不要凭空假设。
- 后端遵守 3.1、前端遵守 3.2;优先用内部框架封装。
- 产出的结论沉淀回对应文档,保持文档与代码同步。
- 不确定需求或契约时停下来提问,不要自行脑补。
