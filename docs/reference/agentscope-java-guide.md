# AgentScope Java 速查手册(2.0.0-RC3)

> 阿里通义实验室出品的 Java 版 Agent 框架。本文基于官网 v2 文档(<https://java.agentscope.io/v2/zh/docs/>)通读整理的**可直接照抄的速查手册**,版本锁定 **2.0.0-RC3**(2026-06-11 发布)。
> 2.0 相对 1.x 是一次大重构:消息/事件/Middleware 更轻更正交,HITL 与流式成为框架内生能力,新增 Harness 工程化与企业级分布式部署。从 1.x 升级见官网《V1 迁移指南》。
> 落地参考:本仓库 `src/main/java/com/transsion/fpm/rebate/agent/agentscope/` 即一个完整可用示例。

---

## 0. 一句话 & 选型

- AgentScope = 把"模型 + 工具(ReAct 循环)+ 多轮记忆 + 事件流 + 状态持久化 + 权限 + HITL"整合进一个统一 Agent 的框架。
- **`ReActAgent`**(`io.agentscope.core`):推理-行动循环引擎,核心抽象。**绝大多数场景(对话 + 工具 + 多轮记忆)用它就够。**
- **`HarnessAgent`**(`io.agentscope.harness.agent`):`ReActAgent` 的薄包装,**官网推荐的入口**,builder 接口大体一致,叠加 workspace/长期记忆/会话持久化/子 agent/沙箱/技能/上下文压缩等"长期自治 + 生产部署"能力。需要跨进程状态恢复、文件工作区、自动技能沉淀、子 Agent 编排、多副本部署时上它。
- **Agent 是无状态单例**:一个实例并发服务多用户/多会话,可变状态全在 `AgentState`(按 `(userId, sessionId)` 隔离)。→ 建议做成 Spring/Hulk 单例 Bean。

---

## 1. 版本与依赖

### 1.1 版本

`io.agentscope` 在 Maven 上有 1.x / 2.x 两条线,**API 不兼容**。本文对应 **2.0.x 线**,锁定 `2.0.0-RC3`。查可用版本:
```bash
curl -s "https://maven.aliyun.com/repository/public/io/agentscope/agentscope-core/maven-metadata.xml"
```
> RC 版,留意后续升稳定版。确认 API 可直接 `javap`:`javap -classpath <core.jar> io.agentscope.core.ReActAgent`。

### 1.2 环境要求

- **JDK 17+**(本项目用 Java 21,满足)。构建推荐 Maven 3.9+。
- 传递依赖含 **Reactor**(`Mono`/`Flux`)。与 Spring Cloud Netflix(Hulk)实测无冲突。

### 1.3 Maven 依赖

**推荐入口 `HarnessAgent`**——依赖 `agentscope-harness` 会自动把核心 `agentscope-core` 一并拉进来:
```xml
<properties>
    <agentscope.version>2.0.0-RC3</agentscope.version>
</properties>

<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-harness</artifactId>
    <version>${agentscope.version}</version>
</dependency>
```
只想跑裸 `ReActAgent`(不需要工作区/持久化/子 agent/沙箱),单独依赖 `agentscope-core` 即可。

### 1.4 ⚠️ Provider 模型已迁出 core(重大变化)

2.0 把 provider-specific 的模型实现从 `agentscope-core` **迁移到独立 extension module**。core 只保留共享契约(`Model`/`ChatModelBase`/`GenerateOptions`/`ChatResponse`)。**旧代码 `import io.agentscope.core.model.DashScopeChatModel` 会编译不过**,必须改:

| Provider | Maven artifact | 主包名 |
|---|---|---|
| DashScope(通义千问) | `agentscope-extensions-model-dashscope` | `io.agentscope.extensions.model.dashscope` |
| OpenAI / 兼容网关 | `agentscope-extensions-model-openai` | `io.agentscope.extensions.model.openai` |
| Anthropic | `agentscope-extensions-model-anthropic` | `io.agentscope.extensions.model.anthropic` |
| Gemini | `agentscope-extensions-model-gemini` | `io.agentscope.extensions.model.gemini` |
| Ollama | `agentscope-extensions-model-ollama` | `io.agentscope.extensions.model.ollama` |

```xml
<!-- 以 DashScope 为例 -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-extensions-model-dashscope</artifactId>
    <version>${agentscope.version}</version>
</dependency>
```
- formatter 也随之迁移:`io.agentscope.extensions.model.<provider>.formatter.*`。
- 其他扩展模块:`agentscope-extensions-redis` / `-mysql` / `-oss`(分布式 StateStore/BaseStore/快照)、`agentscope-extensions-sandbox-docker` 等沙箱后端、`agentscope-extensions-channel`(钉钉/飞书/企微 IM 接入)、`agentscope-spring-boot-starter` + provider-specific starter(Spring 自动装配)。
- MCP 集成需官方 MCP SDK,参考 `agentscope-examples/documentation/pom.xml`。

---

## 2. 模型(Model)

### 2.1 两层结构:Credential + ChatModel

模型层分两层:上层 **Credential**(`io.agentscope.core.credential`,承载某 provider 的 API 鉴权字段),下层 **ChatModel**(在该凭证上对接的具体推理模型)。

```
CredentialBase
└── ChatModelBase
    ├── OpenAIChatModel      ├── DashScopeChatModel
    ├── AnthropicChatModel   ├── GeminiChatModel
    └── OllamaChatModel
```
从一个凭证可通过 `listModels()` 获取该 provider 支持的模型列表(`List<ModelCard>`),`ModelCard` 含 `modelName/displayName/contextSize`,用于驱动前端模型选择器。

### 2.2 字符串 model id(推荐,最简)

`ReActAgent/HarnessAgent.Builder` 的 `.model(...)` 既接受 `ModelRegistry` 解析的字符串 id(自动读 env),也接受手动构造的 `Model` 实例。**字符串形式最常用**:
```java
HarnessAgent agent = HarnessAgent.builder()
        .name("my-agent")
        .sysPrompt("你是一个有帮助的助手。")
        // 由 ModelRegistry 解析,自动读取 DASHSCOPE_API_KEY
        // 切换厂商改成 "openai:gpt-4o-mini" / "anthropic:claude-sonnet-4-5"
        // / "gemini:gemini-2.0-flash" / "ollama:llama3"
        .model("dashscope:qwen-plus")
        .build();
```
`ModelRegistry` 支持 `dashscope` / `openai` / `anthropic` / `gemini` / `ollama`,自动从环境变量读 key(`DASHSCOPE_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GEMINI_API_KEY`)。

### 2.3 显式 Model builder(需精细控制时)

需要自定义超时/endpoint/参数时,显式构造 `Model` 实例:
```java
import io.agentscope.extensions.model.dashscope.DashScopeChatModel;
import io.agentscope.extensions.model.dashscope.formatter.DashScopeChatFormatter;
import io.agentscope.core.model.GenerateOptions;

// DashScope 流式
DashScopeChatModel model = DashScopeChatModel.builder()
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-plus")
        .stream(true)
        .formatter(new DashScopeChatFormatter())
        .build();

// OpenAI 兼容网关(vLLM / DeepSeek / Kimi 等)
OpenAIChatModel model = OpenAIChatModel.builder()
        .apiKey(apiKey)
        .modelName("gpt-4o-mini")
        .baseUrl("https://your-gateway/v1")   // 关键:兼容网关
        .stream(true)
        .build();

// 推理模型(思考模式)
DashScopeChatModel model = DashScopeChatModel.builder()
        .apiKey(key).modelName("qwen3-235b-a22b-thinking-2507")
        .stream(true).enableThinking(true)
        .defaultOptions(GenerateOptions.builder().thinkingBudget(2048).build())
        .build();
```
各 ChatModel builder 共享字段:`apiKey / modelName / stream / defaultOptions(GenerateOptions) / formatter / baseUrl`。`GenerateOptions` 含 `temperature / maxTokens / thinkingBudget / parallelToolCalls` 等。

### 2.4 Formatter

Formatter 把 `Msg` 转为各 provider API 期望的请求载荷,通过 builder 的 `.formatter(...)` 配置。每个 provider 内置两种:
- **ChatFormatter**(默认):单 agent 对话,每条 `Msg` 1:1 映射,保留原始角色。
- **MultiAgentFormatter**:多 agent 场景(辩论/moderator),连续 agent 消息聚合并标注发送者名。

| Provider | Chat | MultiAgent |
|---|---|---|
| DashScope | `DashScopeChatFormatter` | `DashScopeMultiAgentFormatter` |
| OpenAI | `OpenAIChatFormatter` | `OpenAIMultiAgentFormatter` |
| Anthropic | `AnthropicChatFormatter` | `AnthropicMultiAgentFormatter` |
| Gemini / Ollama | 同理 | 同理 |

### 2.5 结构化输出路径

框架按模型能力自动选路径,调用方代码完全相同:

| 路径 | 条件 | 机制 |
|---|---|---|
| **Native** | `supportsNativeStructuredOutput()=true` | `response_format`+`json_schema` 让模型直接输出合规 JSON |
| **Fallback**(默认) | `=false` | 注入 `generate_response` 合成工具,模型以 tool call 返回结构化数据 |

native 失败(如模型返回 400)会**自动降级**到 fallback。各 provider 默认行为:

| Provider | supportsNativeStructuredOutput | 说明 |
|---|---|---|
| OpenAI(GPT-4o 等) | `true` | 原生支持 `json_schema` |
| OpenAI(DeepSeek/GLM formatter) | `false` | 走 fallback |
| DashScope | `false` | 原生端点仅支持 `json_object`,默认走 fallback |
| Anthropic | `false` | — |

> DashScope 思考模式(`enableThinking(true)`)不支持结构化输出,强制走 fallback。
> 显式开启 native:`.nativeStructuredOutput(true)`。结构化输出与工具调用共存时,部分 OpenAI 兼容 API(Kimi/DeepSeek)会跳过工具调用,设 `.nativeStructuredOutputWithTools(false)` 解决。

### 2.6 自定义 Provider

最小路径:实现 `CredentialBase` 子类(`getChatModelClass()`)+ `ChatModelBase` 子类(实现 `doStream`),可选注册到 `ModelRegistry`:
```java
ModelRegistry.registerFactory("myprov:.*", modelId ->
    new MyProviderChatModel(new MyProviderCredential(System.getenv("MYPROV_API_KEY"), null),
        modelId.substring("myprov:".length())));
// 之后:ReActAgent.builder().model("myprov:my-model-v1")
```

---

## 3. 组装 Agent

### 3.1 ReActAgent

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.core.tool.Toolkit;

ReActAgent agent = ReActAgent.builder()
        .name("my-agent")                       // 必填
        .sysPrompt("你是一个有帮助的助手。")      // 必填
        .model("dashscope:qwen-plus")           // 必填,Model 或字符串
        .toolkit(new Toolkit())                 // 默认 new Toolkit()
        .stateStore(new JsonFileAgentStateStore(Paths.get(...)))  // 不配则不持久化
        .maxIters(10)                           // ReAct 最大迭代,默认 10
        .build();
```

### 3.2 HarnessAgent(推荐入口)

```java
import io.agentscope.harness.agent.HarnessAgent;
import io.agentscope.harness.agent.memory.compaction.CompactionConfig;
import java.nio.file.Paths;

HarnessAgent agent = HarnessAgent.builder()
        .name("note-taker")
        .sysPrompt("你是一个帮助用户做笔记的助手。")
        .model("dashscope:qwen-plus")
        .workspace(Paths.get(".agentscope/workspace"))   // 工作区
        .compaction(CompactionConfig.builder()            // 对话压缩
                .triggerMessages(30).keepMessages(10).build())
        .build();
```
HarnessAgent 默认用 `JsonFileAgentStateStore`,状态落在 `~/.agentscope/state/<agentId>/`(工作区**之外**,因为状态是恢复工作区的前提)。多聊几轮触发压缩后,事实落到 `workspace/memory/YYYY-MM-DD.md` 再合并到 `MEMORY.md`,下轮自动注入 system prompt。

### 3.3 Builder 参数说明

| 参数 | 类型 | 默认 | 描述 |
|---|---|---|---|
| `name` | `String` | 必填 | 标识符,用于消息/日志 |
| `sysPrompt` | `String` | 必填 | 基础系统提示词 |
| `model` | `Model`/`String` | 必填 | 推理 LLM |
| `toolkit` | `Toolkit` | `new Toolkit()` | 管理工具/MCP/skill/tool group |
| `middlewares` | `List<? extends MiddlewareBase>` | `List.of()` | 5 阶段钩子 |
| `stateStore` | `AgentStateStore` | `null`(ReAct)/JsonFile(Harness) | 配置后自动加载/保存 AgentState |
| `defaultSessionId` | `String` | agent `name` | RuntimeContext 没带 sessionId 时的兜底 |
| `permissionContext` | `PermissionContextState` | DEFAULT 模式 | 工具执行细粒度规则 |
| `modelConfig` | `ModelConfig` | 默认 | 重试次数/备用模型 |
| `reactConfig` | `ReactConfig` | 默认 | 最大迭代/拒绝处理 |
| `maxIters` | `int` | `10` | ReAct 主循环最大迭代 |
| `maxRetries` | `int` | — | 模型调用失败自动重试 |
| `fallbackModel` | `Model`/`String` | — | 主模型连续失败后切换 |
| `skillRepository` | `AgentSkillRepository` | — | 技能仓库,可重复调用追加 |
| `enableMetaTool(true)` | — | — | 注册 `reset_tools` 元工具(官网 Agent 页另称 `list_tools`/`activate_group`),让 LLM 切换工具组 |
| `enableTaskList()` | — | — | 注册任务列表工具 |

---

## 4. 工具(Tool)

三层概念:**Tool**(工具)、**Toolkit**(容器)、**Tool Group**(命名集合,可运行时启停)。

### 4.1 注解式(最常用)

```java
import io.agentscope.core.tool.Tool;
import io.agentscope.core.tool.ToolParam;
import io.agentscope.core.tool.Toolkit;

public class SimpleTools {
    @Tool(name = "get_current_time",
          description = "Returns the current time in a given IANA timezone.",
          readOnly = true, concurrencySafe = true)
    public String getCurrentTime(
            @ToolParam(name = "timezone", description = "IANA timezone, e.g. Asia/Shanghai")
            String timezone) {
        return LocalDateTime.now(ZoneId.of(timezone)).format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
    }
}

Toolkit toolkit = new Toolkit();
toolkit.registerTool(new SimpleTools());   // 注册整个 POJO,所有 @Tool 方法生效
```
- `@Tool` 属性:`name / description / readOnly / concurrencySafe / stateInjected / dangerousFiles / dangerousDirectories / converter`(除 name/description 外都有默认值,可只写关心的)。注意 `externalTool` 是 `ToolBase.builder()` 的方法、**非** `@Tool` 注解字段(见 §4.2)。
- `@ToolParam` 属性:`name / description / required`(默认 true;可空参数写 `required=false`)。
- 返回值作为 `ToolResultBlock` 回灌给模型,**返回"对模型有用的自然语言/JSON 文本"**。
- `registerTool(Object)` 注册的工具进入特殊的 `"basic"` 组,始终激活。

### 4.2 继承 `ToolBase`(自定义 schema/权限/外部执行)

需要自定义 `inputSchema` 或 `checkPermissions()` 时继承 `ToolBase`,重写 `callAsync(ToolCallParam)→Mono<ToolResultBlock>`:
```java
public class WebSearchTool extends ToolBase {
    public WebSearchTool() {
        super(ToolBase.builder()
                .name("WebSearch")
                .description("Search the web for information on a given query.")
                .inputSchema(Map.of("type","object",
                    "properties", Map.of("query", Map.of("type","string","description","The search query.")),
                    "required", List.of("query")))
                .readOnly(true).concurrencySafe(true));
    }
    @Override public Mono<PermissionDecision> checkPermissions(Map<String,Object> in, ToolExecutionContext ctx) {
        return Mono.just(PermissionDecision.allow("read-only"));
    }
    @Override public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        String q = (String) param.getInput().get("query");
        return doSearchAsync(q).map(text -> ToolResultBlock.builder()
                .id(param.getId()).name(getName())
                .output(List.of(TextBlock.builder().text(text).build())).build());
    }
}
```
`externalTool(true)` 则交外部执行(框架发 `RequireExternalExecutionEvent` 暂停,不必实现 `callAsync`)。

### 4.3 给工具传上下文(关键技巧)

`@Tool` 方法里**无 `@ToolParam` 的参数**,框架按类型从 `RuntimeContext` 自动注入:

| 参数类型 | 注入来源 |
|---|---|
| `ToolEmitter` | 流式中间产物 emitter |
| `Agent` | 当前 agent 实例 |
| `AgentState` | 当前 call 的 per-session 状态 |
| `RuntimeContext` | 当前 per-call 上下文 |
| `ToolExecutionContext` | 兼容层(`@Deprecated`,新代码用 RuntimeContext) |
| 其它用户自定义 POJO | `runtimeContext.get(ParamType.class)` |

```java
public record UserContext(String username, String locale) {}

public class PersonalizedTools {
    @Tool(name = "greet", description = "Greet the user")
    public String greet(
            @ToolParam(name = "greeting", description = "Greeting word") String greeting,  // 模型提供
            UserContext userCtx) {                                                          // 框架注入
        return greeting + ", " + (userCtx == null ? "unknown" : userCtx.username()) + "!";
    }
}

RuntimeContext ctx = RuntimeContext.builder()
        .put(UserContext.class, new UserContext("alice", "en"))
        .userId("alice").build();
agent.call(List.of(new UserMessage("Greet me.")), ctx).block();
```
`ToolBase.callAsync` 中用 `param.getRuntimeContext()` 取 context。`ToolCallParam` 还暴露 `getAgent()/getInput()/getEmitter()/getToolUseBlock()`。

> ⚠️ **工具运行在 Reactor 线程**,`ThreadLocal`(如自建租户/请求头上下文)不会自动传过去。把需要的值放进上述注入对象,在工具方法内重建 ThreadLocal 再用,`finally` 清理(见 §15 踩坑清单)。

### 4.4 MCP

集成 Model Context Protocol,支持 STDIO / SSE / Streamable HTTP 三种连接。MCP tool 命名为 `mcp__{server_name}__{tool_name}`:
```java
McpClientWrapper amap = McpClientBuilder.streamableHttp()
        .name("amap")
        .url("https://mcp.amap.com/mcp?key=" + System.getenv("AMAP_API_KEY"))
        .build();
toolkit.registerMcpClient(amap).block();   // 注意返回 Mono,需 block/订阅

// STDIO:  McpClientBuilder.stdio().name("fs").command("mcp-server-filesystem").args("--root","/my/project").build();
// SSE:    McpClientBuilder.sse().name("search").url("https://api.search.com/mcp/sse").build();
```

### 4.5 Skill(基于 markdown 的指令集)

Skill 不能被直接调用,agent 通过自动注册的查看器工具 `load_skill_through_path` 读取 `SKILL.md` 指令,再用现有 tool 执行。加载 skill 时其绑定的 tool group 自动激活。
```java
ReActAgent.builder()
        .skillRepository(new FileSystemSkillRepository(Paths.get("/path/to/skills"), false))
        .build();
```
`skillRepository(...)` 可重复调用追加(低→高优先级,同名后者覆盖)。Skill 涉及脚本执行时,ReActAgent 需注册 `ShellCommandTool`/`ReadFileTool`/`WriteFileTool`;HarnessAgent 自带 workspace 感知的 shell/文件工具。

### 4.6 Tool Group / Meta Tool

`ToolGroup` 是带名称的 tool/MCP/skill 集合,可整体激活/停用。`enableMetaTool(true)` 后自动注册 `reset_tools`,让 agent 运行时自管理激活哪些 group,保持上下文聚焦:
```java
ToolGroup database = new ToolGroup("database", "DB ops", ToolGroupScope.SESSION, false);
database.addTool("db_query");
toolkit.registerTool(new DatabaseTools());
toolkit.registerToolGroup(database);

ReActAgent agent = ReActAgent.builder()
        .toolkit(toolkit).enableMetaTool(true).build();
```
> `reset_tools` 输入表示所有 group 的**最终状态**而非增量,未显式置 true 的非 basic group 都会被停用。

---

## 5. 消息(Msg)

`Msg`(`io.agentscope.core.message`)代表对话一个轮次,内容以有序的类型化 `ContentBlock` 列表表示。按 role 固定子类:`UserMessage`/`AssistantMessage`/`SystemMessage`/`ToolResultMessage`。

### 5.1 内容块

| 块类型 | 说明 | 允许出现在 |
|---|---|---|
| `TextBlock` | 纯文本 | USER/ASSISTANT/SYSTEM |
| `DataBlock` | 二进制数据(图片/音频/视频),base64 或 URL;**统一替代旧 ImageBlock/AudioBlock/VideoBlock** | USER/ASSISTANT |
| `ImageBlock`/`AudioBlock`/`VideoBlock` | 旧版具体多媒体块(仍兼容,新代码建议用 DataBlock) | USER |
| `ThinkingBlock` | 模型推理过程(思维链) | ASSISTANT |
| `ToolUseBlock` | 工具调用,含 `id/name/input/state` | ASSISTANT |
| `ToolResultBlock` | 工具执行结果,含 `state` | ASSISTANT |
| `HintBlock` | 以用户上下文形式注入循环的指令 | ASSISTANT |

> 角色约束在构造时强制:`USER` 只能含 text/data/image/audio/video;`SYSTEM` 只能含 `TextBlock`;`ASSISTANT` 可含所有类型。

### 5.2 创建与访问

```java
new UserMessage("纯文本");                          // 最常用,字符串自动包 TextBlock
new UserMessage("user", TextBlock.builder().text("描述这张图片：").build(),
        DataBlock.builder().source(Base64Source.builder().data("...").mediaType("image/png").build()).build());

msg.getTextContent();                  // 所有 TextBlock 拼接(\n),无则空串
msg.getContentBlocks(ToolUseBlock.class);   // 按类型过滤
msg.getFirstContentBlock(Class<T>);
msg.hasContentBlocks(Class<T>);
```
`Msg` 核心字段:`getId/getName/getRole(MsgRole)/getContent(List<ContentBlock>)/getMetadata/getTimestamp/getUsage(ChatUsage)/getGenerateReason`。`GenerateReason`:`MODEL_STOP / TOOL_SUSPENDED / REASONING_STOP_REQUESTED / ACTING_STOP_REQUESTED / INTERRUPTED / MAX_ITERATIONS`。

---

## 6. 事件(AgentEvent)

事件是消息的流式对应物。`streamEvents(...)` 产出 `Flux<AgentEvent>`(`io.agentscope.core.event`),驱动实时界面与 HITL。单次 `call` 产生的事件序列最终汇聚成恰好一条 assistant `Msg`。

### 6.1 生命周期

事件遵循 **start → delta → end** 模式。同一次回复所有事件共享 `replyId`;内部用 `blockId` 关联文本/思考/数据块,用 `toolCallId` 关联工具调用与结果。`getSource()` 标识来源:顶层 Agent 为 `null`,子 Agent 为斜杠路径(如 `"main/researcher"`)。

### 6.2 事件类型(按类别)

| 类别 | 事件类 | 关键取值 |
|---|---|---|
| 生命周期 | `AgentStartEvent` / `AgentEndEvent` / `ExceedMaxItersEvent` / `RequestStopEvent` | `getReplyId()` |
| 文本流式 | `TextBlockStartEvent` / `TextBlockDeltaEvent` / `TextBlockEndEvent` | `getDelta()`(逐 token) |
| 思考流式 | `ThinkingBlockStart/Delta/EndEvent` | 思维链 |
| 数据流式 | `DataBlockStart/Delta/EndEvent` | `getMediaType()`/`getData()`(base64) |
| 工具调用 | `ToolCallStartEvent` / `ToolCallDeltaEvent` / `ToolCallEndEvent` | `getToolCallName()`/`getToolCallId()` |
| 工具结果 | `ToolResultStartEvent` / `ToolResultTextDeltaEvent` / `ToolResultDataDeltaEvent` / `ToolResultEndEvent` | `getState()`(SUCCESS/ERROR/INTERRUPTED/DENIED/RUNNING) |
| 模型调用 | `ModelCallStartEvent` / `ModelCallEndEvent` | `getModelName()`/`inputTokens`/`outputTokens` |
| 人工介入 | `RequireUserConfirmEvent` / `RequireExternalExecutionEvent` / `UserConfirmResultEvent` / `ExternalExecutionResultEvent` | `getToolCalls()` |
| 子 Agent | `SubagentExposedEvent` | `getSubagentId()/getAgentId()/getSessionId()/getLabel()` |

**RC3 新增事件**:
- **`AgentResultEvent`** —— agent 调用完成后、`AgentEndEvent` 之前发出,携带最终 `Msg` 结果。`streamEvents()` 消费方可直接从事件流拿最终结果,无需额外订阅 `Mono<Msg>`。
- **`CustomEvent`** —— 通用可扩展事件,中间件向前端推送应用级通知(状态变更/团队变更),无需为每种场景新增枚举。内置 well-known name:`state_updated`、`team_updated`。
- **`HintBlockEvent`** —— 一次性 hint block 事件,传递团队消息/后台工具结果/用户中断等完整内容。
- **工具事件携带 `toolCallName`**(RC3):`ToolCallDeltaEvent`/`ToolCallEndEvent`/`ToolResultDataDeltaEvent`/`ToolResultEndEvent`/`ToolResultTextDeltaEvent` 均新增 `toolCallName`,消费端不再需要缓存 start 事件的名称映射。

### 6.3 SSE 适配范式

```java
agent.streamEvents(new UserMessage("帮我做个返利政策"))
    .doOnNext(ev -> {
        if (ev instanceof TextBlockDeltaEvent d) {
            sink.send("message", d.getDelta());          // 推 token
        } else if (ev instanceof ToolCallStartEvent t) {
            sink.send("tool", Map.of("name", t.getToolCallName()));
        } else if (ev instanceof AgentResultEvent r) {
            sink.send("result", r.getMsg().getTextContent());  // RC3:直接拿最终结果
        }
    })
    .blockLast();
```
事件流可按 `replyId/blockId/toolCallId` 聚合还原成完整 `AssistantMessage`——后端 SSE 推给前端,前端客户端侧重建消息,连接中断也能从任意检查点重放恢复。

---

## 7. 运行:call / streamEvents / RuntimeContext

```java
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.UserMessage;
import io.agentscope.core.message.Msg;

RuntimeContext ctx = RuntimeContext.builder()
        .userId("alice")            // 归属/隔离,可空(匿名)
        .sessionId("session-001")   // 同一 (userId,sessionId) 自动加载/保存记忆
        .put("request_id", "req-1")          // 字符串属性层
        .put(MyCallContext.class, myCtx)     // 类型化属性层(给工具注入)
        .build();

// 同步拿最终结果(Reactor,记得 block)
Msg result = agent.call(List.of(new UserMessage("你好")), ctx).block();
String text = result.getTextContent();

// 流式(streamEvents 不接受 RuntimeContext 重载,见下方说明)
agent.streamEvents(new UserMessage("帮我做个返利政策"))
     .doOnNext(ev -> { /* 适配 SSE */ })
     .blockLast();
```
- 全部 API 返回 **Reactor 类型**(`Mono`/`Flux`),需 `block()/blockLast()` 或订阅。
- `call` 重载:`call(List<Msg>)` / `call(List<Msg>, RuntimeContext)` / `call(Msg, RuntimeContext)` / `call(msgs, structuredOutputClass, runtimeContext)`。`streamEvents(List<Msg>)` / `streamEvents(Msg)`。
- ⚠️ **`streamEvents` 没有 `RuntimeContext` 重载**。不传 ctx 时框架用 `RuntimeContext.empty()`(会话字段为 null,回退到 builder 的 `defaultSessionId`)。**流式且需 per-call 上下文(多用户隔离/工具注入)时,改用 `agent.stream(msgs, options, ctx)`**(返回 `Flux<AgentEvent>`),或在 builder 上配全局 `toolExecutionContext`。这是多用户场景下最容易踩的坑——直接 `streamEvents(msg)` 会丢失 userId/sessionId 隔离。
- `observe(msg)`:只注入上下文不回复(返回 `Mono<Void>`),多 agent 场景用。
- **并发模型**:不同 `(userId,sessionId)` 完全并行;**相同 `(userId,sessionId)` 自动 FIFO 串行**(保证对话一致,无需自己加锁)。
- **中断**:`agent.interrupt(userId, sessionId)` 精确打断单会话(可带消息);返回带 `GenerateReason.INTERRUPTED` 的 Msg,状态自动保存,下次同 session 从中断点恢复。

### RuntimeContext 三层槽位

| 槽位 | 设置 | 读取 |
|---|---|---|
| 会话字段 | `userId(String)` / `sessionId(String)` | `getUserId()` / `getSessionId()` |
| 字符串属性 | `put(String,Object)` | `<T> T get(String)` |
| 类型化属性 | `put(Class<T>,T)` | `<T> T get(Class<T>)` |

> `RuntimeContext` 是 per-call 瞬态元数据袋,**不持久化**;`sessionId/userId` 字段驱动持久化(决定读写哪个 AgentState 槽位)。call 入口框架会把 call-scoped `AgentState` 注入到 `RuntimeContext`,中间件/工具用 `ctx.getAgentState()` 或静态 `RuntimeContext.resolveAgentState(ctx, agent)` 取本次状态。`ToolExecutionContext` 已 `@Deprecated`,底层自动桥接到 `RuntimeContext.asToolExecutionContext()`。

---

## 8. 上下文与记忆(AgentState / StateStore)

### 8.1 无状态引擎

Agent 实例只持有不可变配置(sysPrompt/model/toolkit/middlewares);所有 per-session 可变数据在 `AgentState`(`io.agentscope.core.state`),以 `(userId, sessionId)` 索引。一个实例服务全部用户,每次请求传不同 `RuntimeContext`。

### 8.2 AgentState 字段

| 字段 | 内容 |
|---|---|
| `getContext()` / `contextMutable()` | 当前对话历史 |
| `getSummary()` | 压缩摘要 |
| `getPermissionContext()` | 工具权限规则 |
| `getPlanModeContext()` | Plan Mode 状态/计划文件路径 |
| `getTasksContext()` | `todo_write` 任务清单 |
| `getToolContext()` | 工具组激活状态 |

> 1.0 的 `Memory` 接口(`InMemoryMemory`/`LongTermMemory`)在 2.0 已 `@Deprecated(forRemoval=true)`,新代码用 `AgentState.getContext()` + `AgentStateStore`。

### 8.3 自动持久化与恢复

`call()` 入口从缓存/store 加载 `AgentState` 注入 RuntimeContext → 推理循环(中间件就地改 state)→ call 结束整体写回 store。**状态存储不在每条消息后落盘,而是 call 结束/shutdown 时整体写入**,对后端吞吐压力低。只要 `streamEvents/call` 时传同一 `(userId,sessionId)`,无需自己回放历史消息。

### 8.4 StateStore 实现

| 实现 | 模块 | 用途 |
|---|---|---|
| `InMemoryAgentStateStore` | core | 测试/单进程 demo,**重启即丢** |
| `JsonFileAgentStateStore` | core | 单机开发,**HarnessAgent 默认**,落 `~/.agentscope/state/<agentId>/` |
| `RedisAgentStateStore` | extensions-redis | **生产首选**,多副本共享;支持 Jedis/Lettuce/Redisson(Standalone/Cluster/Sentinel) |
| `MysqlAgentStateStore` | extensions-mysql | 审计/报表/联表 |
| `OssAgentStateStore` | extensions-oss | 对象存储 |

```java
ReActAgent.builder().stateStore(new JsonFileAgentStateStore(Paths.get(...)))...   // 多轮记忆开关

// Redis 生产
AgentStateStore stateStore = RedisAgentStateStore.builder()
        .jedisClient(new JedisPooled("redis://localhost:6379"))
        .keyPrefix("myapp:session:").build();
```
直接读写:`agent.getAgentState(userId, sessionId)` / `state.toJson()` / `AgentState.fromJsonString(json)`。

### 8.5 跨进程恢复

只要 store 是分布式(Redis),不同进程/物理机的 agent 实例用相同 `(userId,sessionId)` 的 `call()` 会自动拉到之前留下的 AgentState → **故障转移/滚动发布/跨场景接续**都自动。`InterruptControl` 是纯运行时信号不序列化(per-session 中断);`AgentState.shutdownInterrupted` 标志会持久化,记录是否被优雅停机中断。

---

## 9. 结构化输出

```java
public record WeatherResp(String location, String temperature, String condition) {}

Msg r = agent.call(List.of(new UserMessage("旧金山天气?")), WeatherResp.class, ctx).block();
WeatherResp data = r.getStructuredData(WeatherResp.class);
// 或:Map<String,Object> map = (Map) r.getMetadata().get("_structured_output");
```
也可传 `JsonNode` schema。框架按模型能力自动选 native(`response_format`)或 fallback(合成 `generate_response` 工具)路径,见 §2.5。结构化输出与工具可同时使用——先调工具完成任务,最后以 schema 输出。

---

## 10. 人机交互(HITL)

### 10.1 用户确认

权限系统返回 ASK 时,agent 发 `RequireUserConfirmEvent` 暂停。事件携带 `getReplyId()` 和 `getToolCalls()`(一组 `ToolUseBlock`,暴露 `getId/getName/getInput/getSuggestedRules`)。用 `streamEvents` 监听 → 构造 `ConfirmResult` → 通过 metadata 传给下一次 `call` 恢复:
```java
List<ConfirmResult> results = new ArrayList<>();
for (var tc : confirmEvent.getToolCalls()) {
    results.add(new ConfirmResult(true, tc, tc.getSuggestedRules()));  // true=确认,false=拒绝
}
UserMessage resumeMsg = UserMessage.builder()
        .metadata(Map.of(Msg.METADATA_CONFIRM_RESULTS, results)).build();
Msg result = agent.call(List.of(resumeMsg), RuntimeContext.empty()).block();
```
接受的规则会持久化到权限引擎,匹配的未来调用自动放行。

### 10.2 外部工具执行

`isExternalTool()=true` 的工具,agent 发 `RequireExternalExecutionEvent` 暂停。外部执行后把每个结果封装为 `ToolResultBlock` 回传:
```java
ToolResultBlock.builder().id(tc.getId()).name(tc.getName())
        .output(List.of(TextBlock.builder().text(output).build()))
        .state(ToolResultState.SUCCESS).build();
```

---

## 11. Middleware(中间件)

不修改 agent/model 代码,向执行流程关键位置注入自定义逻辑(日志/追踪/改写/限流/访问控制)。5 个 hook 覆盖从 reply 全流程到模型 API 调用的全链路:

| 位置 | 类型 | 说明 |
|---|---|---|
| `onAgent` | Onion | 包裹一次完整 reply(所有 ReAct 轮次/工具/最终输出) |
| `onReasoning` | Onion | 包裹一轮推理(输入组装→模型调用→流式解码) |
| `onActing` | Onion | 包裹一次工具调用执行(仅 agent 内部工具,external execution 不追踪) |
| `onModelCall` | Onion | 包裹一次底层 `ChatModel` API 调用,最贴近模型 |
| `onSystemPrompt` | Transformer | 组装 system prompt 时触发,多个 middleware 串行接力变换 |

- **Onion(洋葱)**:middleware 包裹下一层 handler,`next.apply(input)` 前后插入逻辑、观察 `Flux<AgentEvent>`。
- **Transformer(变换式)**:前一个输出作后一个输入,无"内层"概念(仅 `onSystemPrompt`)。

嵌套关系:`onAgent` → 每轮 `onReasoning`(内含 `onSystemPrompt` + `onModelCall`)→ `onActing`。

### 11.1 装备与内置

```java
import io.agentscope.core.middleware.MiddlewareBase;
import io.agentscope.core.tracing.OtelTracingMiddleware;

ReActAgent agent = ReActAgent.builder()
        .middlewares(List.of(new OtelTracingMiddleware()))   // .middleware(one) 单数追加
        .build();
```
未实现的 hook 自动跳过,零开销。内置:
- **`OtelTracingMiddleware`**(`io.agentscope.core.tracing`):OpenTelemetry 全链路追踪,在 `onAgent/onModelCall/onActing` 打 span(`invoke_agent <name>` / `chat <model>` / `execute_tool <name>`)。未配 OTel SDK 时短路到 no-op,近零开销。使用前先初始化 OTLP exporter + `SdkTracerProvider`。
- **`TaskReminderMiddleware`**(`io.agentscope.core.middleware`):配合 `enableTaskList(true)` + `TodoTools`,每轮 reasoning 前把 `tasksContext` 渲染成 `<system-reminder>` 注入,防长任务跑偏。
- **`ToolResultEvictionMiddleware`**:RC3 修正时机,从 `onActing` 迁到 `onReasoning`(确保工具结果持久化后再淘汰)。

### 11.2 自定义 Middleware

实现 `MiddlewareBase`(`io.agentscope.core.middleware`),只重写需要的 hook。每个 Onion hook 收到 `next` 函数,调 `next.apply(input)` 进内层;可用 Reactor 算子(`doOnNext/flatMap/map`)观察改写事件流。各 hook 的 input record(均位于 `io.agentscope.core.middleware`):

| Hook | Input record | 字段 |
|---|---|---|
| `onAgent` | `AgentInput` | `msgs: List<Msg>` |
| `onReasoning` | `ReasoningInput` | `messages, tools: List<ToolSchema>, options: GenerateOptions` |
| `onActing` | `ActingInput` | `toolCalls: List<ToolUseBlock>` |
| `onModelCall` | `ModelCallInput` | `messages, tools, options, model: Model` |
| `onSystemPrompt` | `String` | 当前 prompt |

```java
public class TimingMiddleware implements MiddlewareBase {
    @Override
    public Flux<AgentEvent> onModelCall(
            Agent agent, RuntimeContext ctx, ModelCallInput input,
            Function<ModelCallInput, Flux<AgentEvent>> next) {
        long start = System.nanoTime();
        return next.apply(input).doFinally(sig -> {
            System.out.println("[timing] " + agent.getName() + ": "
                    + (System.nanoTime() - start) / 1_000_000 + "ms");
        });
    }
}
```
- **读 RuntimeContext**:所有 hook 第二参数即本次 `call/stream` 绑定的 `RuntimeContext`,可读会话字段、按类型/key 取属性,也可 `put` 写入给下游 hook/tool 传值(内部线程安全 map)。
- **不要把请求级状态缓存到 middleware 实例字段**——一个实例被多 agent/call 复用;放 `RuntimeContext` 或用 Reactor `contextWrite`。
- **执行顺序**:Onion 列表**第一个最外层**(`mw1前→mw2前→内部→mw2后→mw1后`);Transformer(`onSystemPrompt`)**从左到右串行接力**。

### 11.3 实用配方

- **限速**:`onModelCall` 里 `Mono.delay(minInterval).thenMany(next.apply(input))`,两次模型调用间强制最小间隔。
- **动态 system prompt**:`onSystemPrompt` 追加实时上下文(`Mono.just(currentPrompt + "\n## Context\n" + ctxFn.get())`)。
- **模型回退**:`onModelCall` 用 `onErrorResume` 切到备用 model(简单主→备直接用 builder 的 `fallbackModel(...)`/`maxRetries(...)`,无需自写)。

---

## 12. 权限系统(Permission)

`io.agentscope.core.permission` 拦截每次工具调用,给三种决策:**ALLOW** 执行 / **DENY** 拒绝 / **ASK** 询问用户。三组件共同决定:

- **Rules**:针对 tool+调用模式的显式 allow/deny/ask,最高优先级。来源:静态预配置,或 ASK 时用户接受**建议规则**动态加入(接受后相同调用自动放行)。
- **Mode**:全局静态策略,决定不命中规则的调用的默认行为。
- **Built-in Checks**:tool 自身 `checkPermissions()` 基于真实输入的运行时分析,**不可绕过**(不受 mode/rules 覆盖)。

> Deny 规则与危险路径检查**不可绕过**——即使 `BYPASS` 模式也照常生效。

### 12.1 PermissionMode

| Mode | 行为 | 适用 |
|---|---|---|
| `DEFAULT` | 所有操作需显式规则或用户确认 | 最安全,推荐默认 |
| `ACCEPT_EDITS` | 自动放行工作目录内文件操作 | 用户在场的活跃开发 |
| `EXPLORE` | 只读:放行读、拒绝所有写与命令 | 代码探索/规划 |
| `BYPASS` | 放行一切(deny/ask 规则仍生效) | 完全可信沙箱 |
| `DONT_ASK` | 所有 ASK 转为 DENY | 无人值守/计划任务 |

### 12.2 配置 Rules / Mode

```java
import io.agentscope.core.permission.*;

PermissionContextState permCtx = PermissionContextState.builder()
        .mode(PermissionMode.DEFAULT)
        .addAllowRule("safe_read",
                new PermissionRule("safe_read", null, PermissionBehavior.ALLOW, "userSettings"))
        .addAskRule("dangerous_delete",
                new PermissionRule("dangerous_delete", null, PermissionBehavior.ASK, "userSettings"))
        .addDenyRule("drop_table",
                new PermissionRule("drop_table", null, PermissionBehavior.DENY, "userSettings"))
        .build();

ReActAgent agent = ReActAgent.builder()
        .name("guarded").sysPrompt("...").model(model)
        .permissionContext(permCtx).build();
```
`PermissionRule` 字段:`toolName`(必填)/ `ruleContent`(匹配模式,由 tool 的 `matchRule()` 解释;`null` 匹配所有)/ `behavior`(`ALLOW`/`DENY`/`ASK`/`PASSTHROUGH`)/ `source`(`"userSettings"`/`"session"`/`"suggested"` 等)。`ACCEPT_EDITS` 可配 `.addWorkingDirectory(path, AdditionalWorkingDirectory)`。

### 12.3 Built-in Checks 与危险路径

`ToolBase.checkPermissions(toolInput, context)` 返回 `Mono<PermissionDecision>`,四个静态构造:`allow(msg)`/`deny(msg)`/`ask(msg)`/`passthrough(msg)`(PASSTHROUGH=不强加判断,交引擎按 rules/mode 评估)。自定义 tool 重写它做运行时安全校验:
```java
@Override public Mono<PermissionDecision> checkPermissions(Map<String,Object> in, ToolExecutionContext ctx) {
    Object target = in.get("target");
    if (target instanceof String s && s.startsWith("prod-"))
        return Mono.just(PermissionDecision.ask("targets production: " + s));
    return Mono.just(PermissionDecision.passthrough("default"));
}
```
**危险路径保护**:`ToolBase` 内置 `ToolDangerousPathConstants`(shell 配置 `.bashrc`/`.zshrc`、git `.gitconfig`、ssh `.ssh/config`/`id_rsa`、凭证 `.env`/`.aws/credentials`、目录 `.git/`/`.ssh/`/`.aws/`/`.kube/` 等),命中后即使 BYPASS 也强制 ASK。`@Tool` 注解可追加 `dangerousFiles`/`dangerousDirectories`。

### 12.4 结合 HITL

权限返回 ASK 时,agent 暂停并返回 `GenerateReason.PERMISSION_ASKING` 的响应(携带待确认 `ToolUseBlock`,见 §10)。调用方检查 `getGenerateReason()` 展示给用户,收集决策后发新消息恢复;接受建议规则则附在 `ConfirmResult` 回传,自动写入引擎。**无人值守**(CI/定时任务)用 `DONT_ASK`,ASK 自动降级 DENY 不阻塞。

### 12.5 常见配方

- **只读探索**:`mode(EXPLORE)`,写工具自动拒。
- **无人值守自动化**:`mode(DONT_ASK)` + 显式 `addAllowRule` 放行白名单命令,其余静默拒。
- **阻止危险命令**:`mode(BYPASS)` + `addDenyRule` 拦 `drop_table`/`force_push` 等(deny 不可绕过)。
- RC2 新增 `HarnessAgent.setPermissionMode()/getPermissionMode()` 运行时按 session 动态切换。

---

## 13. Harness 高级能力

> `HarnessAgent` 是 `ReActAgent` 的薄包装,把长期运行 agent 必备的工程能力打包进单一 builder。裸 ReActAgent 只解决"一次请求→推理→工具→回复",Harness 回答的是:下一轮怎么接上一轮、上下文如何有界、多用户如何隔离、危险操作如何先 review、可复用能力如何沉淀。第一个示例见 §3.2,部署见 §14,本节讲各能力。

### 13.1 架构与核心原理

三件事记住:
1. **能力叠加在推理循环关键时机上,不改循环**——工作区注入/压缩/子agent/沙箱/PlanMode 都钩在 ReAct 循环关键时机,core 算法没动。
2. **能力间互不依赖,只通过三个共享对象通信**——`RuntimeContext`(本次 call 是谁)、工作区(谁读写哪些文件)、`AgentStateStore`(跨调用怎么恢复)。
3. **内置 middleware 注册顺序固定,你加的跑最前**——`.middleware(...)` 加的跑在 Harness 内置之前。

状态分三层,框架自动在层间搬数据:
- **调用内状态**:`AgentState`(对话上下文/权限/Plan/工具状态)+ `RuntimeContext`(sessionId/userId/沙箱句柄)。
- **跨调用状态**:每次 call 结束自动写盘、下次自动加载——`AgentStateStore` 里的 `AgentState` 快照、`sessions/<sid>.log.jsonl` 永不压缩对话日志、子任务记录、沙箱元数据。
- **长期记忆**:跨 session 累积——`memory/YYYY-MM-DD.md` 只追加,后台节流合并到 `MEMORY.md`,每轮注入 system prompt。

规律:system prompt 每轮重新拼(改 `AGENTS.md`/`MEMORY.md` 立即生效,无需重启);压缩/记忆/后台维护都被节流闸门管(不会每轮跑);`AgentState` 由 core 自动持久化,Harness 不额外做。

核心组件(按需在 builder 开):

| 能力 | Builder 入口 | 解决 |
|---|---|---|
| 工作区驱动人格 | `.workspace(path)` | 人格/知识/子agent/技能/MCP白名单以文件存在 |
| 状态持久化 | 默认开;`.stateStore(...)` | 同(userId,sessionId)跨请求/进程/副本恢复 |
| 双层长期记忆 | 默认开;`.memory(...)` | 长会话事实沉淀到 MEMORY.md |
| 对话压缩 | `.compaction(...)` | 上下文有界;溢出强制重试 |
| 大工具结果卸载 | `.toolResultEviction(...)` | 超80K结果落盘+占位符 |
| 子agent编排 | `.subagent(...)` 或 `workspace/subagents/` | 委派子agent,同步/后台,自动反向通知 |
| 可插拔文件系统 | `.filesystem(...)` | 本机/共享存储/沙箱,不改代码切换 |
| 沙箱隔离 | `.filesystem(new DockerFilesystemSpec()...)` | 文件命令隔离,跨调用恢复,多副本 |
| 计划模式 | `.enablePlanMode()` | 只读思考阶段+HITL退出 |
| 技能装配 | `.skillRepository(...)` | Git/Nacos/MySQL/classpath/工作区 |
| MCP+工具白名单 | `workspace/tools.json` | 声明式MCP server+工具allow/deny |
| Channel路由 | `agent.channel(...)`/`GatewayBootstrap` | 会话管理/per-session并发/多agent路由/流式 |

### 13.2 工作区(Workspace)

工作区是 agent 定义与进化的 source of truth,以目录+Markdown/JSON 组织。目录布局(**逻辑布局**,物理落点由 filesystem 决定,见 §14.2):
```
.agentscope/workspace/
├── AGENTS.md              ← 静态:人格+行为约定(唯一建议写,不写也能跑)
├── MEMORY.md              ← 长期记忆:策划后事实,每轮注入
├── tools.json             ← 静态:MCP server+工具白名单(可选)
├── memory/YYYY-MM-DD.md   ← 长期记忆:每天追加事实流水账
├── knowledge/             ← 静态:领域知识(KNOWLEDGE.md全文+其余只列路径)
├── skills/<name>/SKILL.md ← 静态:技能
├── subagents/<id>.md      ← 静态:子agent声明(文件名即agent_id)
├── plans/PLAN.md          ← 运行时:Plan Mode计划文件
└── agents/<agentId>/      ← 运行时:sessions/(永不压缩日志)+tasks/(子agent任务)
```
内容按生命周期分三类:静态资产(工程师编辑,每轮注入prompt)、运行时文件(框架call写回)、长期记忆(跨session累积)。

**system prompt 拼装**(`WorkspaceContextMiddleware`):每轮 reasoning 时把工作区文件拼成文本追加到 builder 的 sysPrompt 后。注入段:`<agents_context>`(AGENTS.md全文)、`<memory_context>`(MEMORY.md,受 `maxContextTokens` 预算默认8000,超出截断+提示用 memory_search)、`<domain_knowledge_context>`(KNOWLEDGE.md全文+其余文件路径清单)、`<x_md>`(additionalContextFile)。**每轮重新拼,改文件立即生效**。

**两层读架构**:对注入prompt的关键文件,`WorkspaceManager.readWithOverride()` 先问 filesystem 有没有→有则返回(覆盖层),没有再读本地磁盘(兜底层)。写入永远走 filesystem 后端。共享存储模式下:本地是团队git同步的只读模板,远端KV是运行时覆盖,所有副本下一轮读到最新。

**多租户覆盖**:`RuntimeContext.userId` 是切多用户的钥匙。运行时数据(sessions/tasks/memory)按userId进命名空间;静态资产(skills/subagents)可用户级覆盖——`<userId>/skills/code-reviewer/` 覆盖共用 `skills/code-reviewer/`(两层读,用户级为上层)。效果:单个 HarnessAgent 实例对每个租户表现得像不同 agent,不用fork代码/多套部署。

**tools.json**:
```json
{
  "allow": ["read_file","grep_files","execute"],
  "deny":  ["write_file"],
  "mcpServers": {
    "amap": {"transport":"streamableHttp","url":"https://mcp.amap.com/mcp?key=${AMAP_API_KEY}"},
    "local-py": {"transport":"stdio","command":"python","args":["mcp_servers/my_server.py"],"env":{"PYTHONUNBUFFERED":"1"}}
  }
}
```
MCP server 构建期一次性注册;allow/deny 在所有工具注册完后应用(**会过滤内置工具**,用allow白名单时务必把 `read_file`/`memory_search`/`agent_spawn` 等要保留的一并列出);`${ENV_VAR}` 环境变量替换。编程注入:`builder.toolsConfig(ToolsConfig.builder()...)`,关掉读取 `disableToolsConfig()`。

**写入安全**:写文件用 `harnessAgent.getWorkspaceManager()` 而非 `java.nio.Files`(后者在沙箱/远端模式写错地方)。例外:builder装配时初始化种子文件(那时没运行时上下文)用 java.nio.Files OK。框架做 path-traversal 校验(拒绝 `../../etc/passwd`)。

```java
HarnessAgent agent = HarnessAgent.builder()
    .name("MyAgent").model(model)
    .workspace(Paths.get(".agentscope/workspace"))   // 不传则 ${user.dir}/.agentscope/workspace
    .additionalContextFile("SOUL.md")                // 任意workspace相对路径,全文注入
    .maxContextTokens(8000)                          // MEMORY注入预算
    .build();
```
可关掉的子系统(调试/自管时有用):`disableWorkspaceContext()`(关prompt注入)/`disableMemoryHooks()`(关记忆flush)/`disableMemoryTools()`(关memory_search等)/`disableSubagents()`/`disableDynamicSkills()`(技能build时一次合并)/`disableToolsConfig()`/`disableSessionPersistence()`。

### 13.3 上下文压缩(Compaction)

四套策略正交可组合,默认全不开:

| 策略 | 中间件 | 触发 | 解决 |
|---|---|---|---|
| 对话摘要压缩 | `CompactionMiddleware` | 每次模型推理前 | 上下文太"深"(消息/token太多) |
| 大工具结果卸载 | `ToolResultEvictionMiddleware` | 工具执行后 | 上下文太"宽"(单条结果体量大) |
| 上下文溢出兜底 | `recoverFromOverflow` | call()抛错时 | 撞到 context_length_exceeded |
| 预压缩参数截断 | `TruncateArgsConfig` | 摘要前轻量预处理 | 工具入参体量大但后期没人看 |

**对话摘要压缩**:按消息条数或估算token触发,把对话前缀用一次LLM压成结构化摘要(SESSION INTENT/SUMMARY/ARTIFACTS/NEXT STEPS),保留尾部N条原文,写回 `AgentState.contextMutable()`。
```java
.compaction(CompactionConfig.builder()
    .triggerMessages(30)     // 30条触发
    .keepMessages(10)        // 压缩后保留最近10条原文
    .model(fastModel)        // 可选:为压缩指定独立轻量模型,不设用主模型
    .truncateArgs(CompactionConfig.TruncateArgsConfig.builder()
        .maxArgLength(2000).truncationText("... [truncated] ...").build())
    .build())
```
`CompactionConfig` 字段:`triggerMessages/triggerTokens/keepMessages/keepTokens/flushBeforeCompact`(默认true,摘要前先抽事实到Memory)/`offloadBeforeCompact`(默认true,摘要前把原始消息写到log.jsonl)/`model`/`truncateArgs`。

**大工具结果卸载**:单条工具结果文本超阈值(默认80K字符≈20K tokens),全文写盘,上下文只留首尾各约2K字符+`read_file`路径提示。
```java
.toolResultEviction(ToolResultEvictionConfig.defaults())
```
默认排除 `read_file`/`write_file`/`edit_file`/`grep_files`/`glob_files`/`list_files`/`memory_*`/`session_search`(自带分页或返回值小);shell `execute` 默认不排除(输出可能很大)。

**溢出兜底**:模型返回 `context_length_exceeded` 等错误时,强制走一次 `triggerMessages=1` 的极端压缩再自动重试。前提是配了 `.compaction(...)`,否则错误原样抛。无需额外配置。

**压缩不触碰**:Plan Mode状态、子agent后台任务、`todo_write`任务清单、权限规则——各有独立状态机,压缩通路透明。

**session 查询工具**(默认开):`session_list agentId="..."`/`session_history agentId sessionId lastN=20`/`session_search query agentId`,读永不压缩的log.jsonl。

### 13.4 长期记忆(Memory)

两层结构:`MEMORY.md`(策划后长期事实,每轮注入system prompt)+ `memory/YYYY-MM-DD.md`(每天追加事实流水账,未去重)。
- **写入**:对话压缩前 `MemoryFlushMiddleware` 把对话前缀新事实抽到 `memory/YYYY-MM-DD.md`(追加);后台节流任务定期合并去重重写 `MEMORY.md`。
- **读取**:框架自动读 MEMORY.md(两层读);agent 可调 `memory_search`/`memory_get` 找老内容。
- **与压缩联动**:`flushBeforeCompact`(默认true)决定摘要前是否先抽事实到Memory——摘要丢前缀时信息不丢,agent仍可 `memory_search` 查回。

### 13.5 子 Agent(Subagent)

让主agent把"可独立处理、上下文重、可并行"的任务委派出去。每个子agent是临时实例(本地HarnessAgent或远程stub),跑自己会话,结果通过工具返回父。

**声明方式**(构建时合并):
- 工作区spec文件:`workspace/subagents/<id>.md`,文件名即agent_id(不要在frontmatter写name)。
- 编程式:`builder.subagent(SubagentDeclaration.builder()...)`。
- 内置`general-purpose`:总是有,通用兜底(镜像主agent能力,共享主工作区)。

spec文件 frontmatter:
```yaml
---
description: 代码评审专家     # 必填,agent选择是否委派的关键依据
workspace:
  mode: isolated              # 默认isolated;shared和父共享工作区
model: openai:gpt-4o-mini     # 可选;不写继承父
steps: 8                      # 可选;单次最大迭代
hidden: false                 # true不出现在可见列表(仍可程序化spawn)
expose_to_user: true          # 可选三态;强制/禁止向用户暴露
tools: [read_file, grep_files]  # 可选;继承工具白名单
---
你是一个专注代码评审的子 agent。
```
编程式:`.subagent(SubagentDeclaration.builder().name("reviewer").description(...).workspace(Path.of("./defs/reviewer")).workspaceMode(WorkspaceMode.ISOLATED).model("qwen3-max").steps(8).tools(List.of("read_file")).build())`。三种来源互斥:`workspace(...)`/`inlineAgentsBody(...)`/`url(...)`三选一。远程子agent填 `.url("http://...").headers(Map.of(...))`。

**同步/后台**:`agent_spawn agent_id="reviewer" task="..."`,`timeout_seconds`决定:
- `>0`(默认30,最大600):**同步**,主agent block等结果,结果作工具结果返回。
- `=0`:**后台**,立即返回task_id,子agent后台跑。

**后台任务自动反向通知**:后台任务完成后,主agent下一轮推理前框架把结果作 `<system-reminder>` 注入对话末尾,**无需轮询**。后台任务工具:`agent_spawn`/`agent_send`(向已存在子agent追加消息,用agent_key或label寻址)/`agent_list`/`task_output`(按task_id取结果)/`task_cancel`/`task_list`。

**持久会话**:`.persistSession(true)` 让同一子agent多次spawn间复用(按 parentSessionId×agentId×label 生成确定性key)。

**向用户暴露**:`agent_spawn ... expose_to_user=true` 把子agent暴露为用户可直接交互入口(在Gateway注册+发`SubagentExposedEvent`),用户可绕过父直接和子对话。需配 `agent.channel(ChatUiChannel.create())`(没绑Channel时静默忽略)。暴露优先级:RuntimeContext按调用覆盖 > SubagentDeclaration按类型策略 > LLM参数 > 默认false。跨重启/多副本需配 `.distributedStore(...)`。

**流式转发**:父 `streamEvents()` 时,同步本地子agent的中间事件实时流回父的Flux(带 `source` 路径如 `"main/researcher"`,父source为null)。call()非流式/后台任务/远程子agent不实时转发。子agent出错被捕获写成 TOOL_RESULT 给父,不传播 onError 到父流。

**行为细节**:`description` 要写好(模型据此决定委派);递归保护(子不能再spawn子,硬上限3层);userId透传;权限继承(父的DENY规则自动传子,`inheritParentPermissions(false)`关闭);Plan Mode下spawn的子agent自动继承只读限制。

### 13.6 技能(Skill)

skill是写好的能力包:目录里放 `SKILL.md`(YAML frontmatter 的 name+description + 给agent的指令),可带 `references/` 参考资料、`scripts/` 脚本。

**四层合成**(低→高优先级,重名上层覆盖,下层独有保留):
1. 项目全局:`projectGlobalSkillsDir(Path)`,如 `~/.agentscope/skills/`
2. 市场:`skillRepository(...)`,后注册覆盖先注册
3. 工作区共用:`workspace/skills/`
4. 用户隔离:`<userId>/skills/`

**市场后端**:`GitSkillRepository(url)`(HEAD变才pull)/`NacosSkillRepository(aiService,namespace)`(在线下发+订阅,AutoCloseable)/`MysqlSkillRepository.builder(dataSource).writeable(true/false)`(平台统一治理)/`ClasspathSkillRepository("skills")`(跟JAR发)。可重复 `.skillRepository(...)` 追加。

**agent读取执行**:每轮推理system prompt里有 `<available_skills>` 块(只列 name+description+skill-id+files-root)。agent觉得相关才调 `load_skill_through_path(skillId, path="SKILL.md")` 加载详情(也可传 `references/xxx.md`、`scripts/xxx.py`)。加载skill时其绑定的tool group自动激活。**SKILL.md和脚本中只用相对路径**(框架按filesystem模式生成正确 files-root 绝对前缀),不要硬编码绝对路径。

**自学习闭环**(可选,按顺序启用):
1. `.enableSkillManageTool(SkillManageConfig.defaults())` → agent获 `propose_skill`(写草稿到 `skills/_drafts/`)和 `skill_manage`(编辑已有)工具。`.enableSkillManageTool(true)`=autoPromote直接生效(生产不建议)。
2. `.enableSkillPromotionGate(gate, filter).environment("prod")` → 草稿变正式需经闸门(内置:直接拒绝/本地人工确认/推消息等);可见性过滤按环境/灰度/白名单。
3. `.enableSkillCurator(SkillCuratorConfig.builder().intervalHours(7*24).staleAfterDays(30).archiveAfterDays(90).build())` → 后台周期整理:30天没用标stale,90天归档到 `skills/.archive/`。
程序化:`agent.runCuratorOnce()`/`agent.promoteSkill(name,userId)`/`agent.queryAudit(date,filter)`。

**建议**:`description` 决定agent用不用(写"当用户要…时使用"而非"数据分析工具");SKILL.md控制在2k tokens,详细放 `references/`;通用能力放市场,项目特有写工作区;用户目录用来覆盖+补充不当主存放。

### 13.7 计划模式(Plan Mode)

让agent动手前先"想清楚+写下来"再执行。开启后进入只读阶段:只能调只读工具+4个白名单(`plan_enter`/`plan_write`/`plan_exit`/`todo_write`),其它工具调用被拒,退出走HITL确认(复用权限ASK)。

```java
HarnessAgent agent = HarnessAgent.builder()
    .name("planner").model(model).workspace(workspace)
    .enablePlanMode()              // 装 PlanMode 三件套
    .planFileDirectory("plans")    // 可选;默认"plans"
    .allowShellInPlanMode()        // 可选;按需放开plan阶段shell(execute)
    .build();
```
三工具:`plan_enter`(进入)/`plan_write(content)`(写计划到 `plans/PLAN.md`,专为PlanMode设计的写入入口,避开 `write_file` 安全风险)/`plan_exit(rationale?)`(退出→执行阶段,HITL确认)。

工作流:`plan_enter`→思考调只读工具→`plan_write` 写PLAN.md→`plan_exit` 弹HITL确认→`ConfirmResult(true)`→进入执行阶段所有工具解禁。非白名单工具被拒返回 `[Tool denied — plan mode is active]`。

**四种终态**(只看 `isPlanModeActive()==false` 有歧义):从未进入 / 进入→`plan_exit`(成功) / 仍在plan+有PLAN.md(起草未退出,发后续消息批准) / 仍在plan+无PLAN.md(只说不做)。代码里结合 `isPlanModeActive`、计划文件是否存在、是否调过 `plan_enter`/`write`(从 `ToolCallStartEvent` 捕获)区分。

**运行期切换权限模式**(危险开关逃生口):`agent.setPermissionMode(ctx, PermissionMode.BYPASS)` 全部放开不弹确认(建议配沙箱);`DONT_ASK` 无人值守不弹确认但ASK变DENY保留管控。`setPermissionMode` 保留该session的allow/deny/ask规则与工作目录,只改mode,下次call生效。

**程序化进出**:`agent.enterPlanMode(ctx)`/`agent.exitPlanMode(ctx)`(程序入口不触发HITL)/`agent.isPlanModeActive(ctx)`。配 `agentscope-admin-spring-boot-starter` 可用 admin HTTP接口。

**持久化**:Plan Mode是运行时状态随 `AgentState` 持久化,进程重启/节点切换/跨副本恢复后plan阶段一起恢复;计划文件在 `workspace/plans/`。

**与 `todo_write` 协作**:Plan Mode(阶段开关+计划文件+HITL退出)与 `todo_write`(执行阶段维护结构化清单,全量替换必须恰好一个 in_progress)独立但常一起用。典型:plan写PLAN.md→`plan_exit`→执行阶段 `todo_write` 把PLAN拆5-8条→逐条推进。不要和子agent后台任务(`task_output` 等)混淆。

### 13.8 Channel(IM接入)

`agentscope-extensions-channel` 系列模块实现IM平台接入(钉钉/飞书/企业微信/GitHub/GitLab),内置ChatUI提供开箱即用对话界面。`agent.channel(ChatUiChannel.create())` 创建内部gateway并自动接好bridge(`expose_to_user` 直接可用)。多agent场景用 `GatewayBootstrap`。Channel负责会话管理、per-session并发控制、多agent路由、流式SSE。详见官网 [Channel](https://java.agentscope.io/v2/zh/docs/harness/channel.html) 页。

---

## 14. 上生产要点

### 14.1 DistributedStore 一键配置

生产多副本要共享会话/隔离用户/支持不可信代码/pod 重启接续。**最快方式**用 `DistributedStore` 一键配置所有分布式组件(官网 Release Notes 称 `DistributedBackend`/`distributedBackend`,两处官网文档命名不一致,以 jar 实测为准):
```java
DistributedStore store = RedisDistributedStore.fromJedis(jedis);
// 或 MysqlDistributedStore.create(dataSource); / OssDistributedStore.create(ossClient, bucket, prefix);

HarnessAgent.builder()
    .distributedStore(store)   // 自动注入 stateStore + baseStore + snapshotSpec + executionGuard
    .filesystem(...)
    .build();
```
混合 store 也行(MySQL 管状态 + Redis 管 sandbox 锁)。能力矩阵:

| 能力 | Redis | OSS | MySQL |
|---|---|---|---|
| `AgentStateStore` | `RedisAgentStateStore` | `OssAgentStateStore` | `MysqlAgentStateStore` |
| `BaseStore`(共享 KV 文件) | `RedisStore` | `OssBaseStore` | `JdbcStore` |
| `SandboxSnapshotSpec` | `RedisSnapshotSpec` | `OssSnapshotSpec` | `JdbcSnapshotSpec` |
| `SandboxExecutionGuard` | `RedisSandboxExecutionGuard` | —(对象存储不适合做锁) | `JdbcSandboxExecutionGuard` |

> ⚠️ **不要把 OSS 当 BaseStore 用**——`MEMORY.md`/`memory/` 每秒可能写几次,OSS 延迟与 per-request 成本会失控。高频小 KV 用 Redis/MySQL,大对象(沙箱 workspace tar)才用 OSS。

### 14.2 Filesystem 模式 & IsolationScope

| 模式 | 配置 | shell | 适用 |
|---|---|---|---|
| 本机+shell | `LocalFilesystemSpec` 或不配 | ✅ 宿主 `sh -c` | 单进程/信任环境 |
| 共享存储 | `RemoteFilesystemSpec(store)` | ❌ | 多副本共享长期记忆 |
| 沙箱 | `DockerFilesystemSpec` 等 5 种 | ✅ 沙箱内 | 不可信代码/跨调用恢复/多用户隔离 |

`IsolationScope` 决定命名空间分桶:`SESSION`(沙箱默认,每 sessionId 独立)/`USER`(Remote 默认,同 userId 跨 session 共享)/`AGENT`/`GLOBAL`。

### 14.3 沙箱 + Snapshot

五种沙箱:`DockerFilesystemSpec`/`KubernetesFilesystemSpec`/`DaytonaFilesystemSpec`/`E2bFilesystemSpec`/`AgentRunFilesystemSpec`(阿里云 AgentRun,原生 NAS/OSS mount)。沙箱默认瞬时,`SandboxSnapshotSpec` 把工作区打 tar 持久化,下次 call 自动 hydrate 回新容器。`AGENT/GLOBAL` scope 多副本需 `SandboxExecutionGuard` 跨节点串行化。用 `distributedStore(...)` 后快照和执行锁自动注入。

### 14.4 Skill 集中管理

生产优先 `MysqlSkillRepository(writeable=false)` 或 `NacosSkillRepository`(在线下发+变更订阅),agent 端只读。开 `enableSkillManageTool` 让 agent 起草新 skill 时**必须**配 `enableSkillPromotionGate(...)`,生产严禁 `autoPromote=true`。`NacosSkillRepository` 是 `AutoCloseable`,Spring 用 `@PreDestroy` 关。

### 14.5 完整生产 builder 模板

```java
Path workspace = Paths.get("/var/agentscope/workspace");
JedisPooled jedis = new JedisPooled(System.getenv("REDIS_URI"));
DistributedStore store = RedisDistributedStore.fromJedis(jedis);

HarnessAgent agent = HarnessAgent.builder()
        .name("coding-assistant")
        .model("dashscope:qwen-plus")
        .workspace(workspace)
        .distributedStore(store)
        .filesystem(new DockerFilesystemSpec()
                .image("python:3.12-slim")
                .isolationScope(IsolationScope.USER))
        .compaction(CompactionConfig.builder().triggerMessages(50).keepMessages(20).build())
        .middlewares(List.of(new OtelTracingMiddleware()))
        .build();

// HTTP handler 中
agent.call(msg, RuntimeContext.builder()
        .userId(httpRequest.tenantUserId())
        .sessionId(httpRequest.sessionId()).build()).block();
```

---

## 15. 实战踩坑清单(务必看)

1. **版本**:用 `2.0.0-RC3`(对应 v2 文档);`1.0.x` API 不同。RC 版,留意后续升稳定版。见 §1。
2. **Provider 模型迁出 core**:`DashScopeChatModel` 等已不在 `io.agentscope.core.model`,改引 `io.agentscope.extensions.model.<provider>.*`,并加对应 extension 依赖。formatter 同步迁移。见 §1.4/§2。
3. **必须有 LLM key**:没有离线兜底;`DashScope` 配 `DASHSCOPE_API_KEY`。Bean 装配阶段一般不校验 key(build 不报错),**调用时才失败** → 在 SSE/调用处兜异常回前端。
4. **工具里的 ThreadLocal**:工具跑在 Reactor 线程,自建租户/请求头 ThreadLocal 丢失。→ 用 `RuntimeContext.put(Ctx.class, ...)` 注入,在工具内重建并 `finally` 清理。
5. **阻塞**:`blockLast()/block()` 会阻塞当前线程;放进异步执行器(本仓库用 SSE 专用线程池),别阻塞 web 容器线程。
6. **Spring/Hulk 集成**:Agent/Model/Toolkit 做成单例 `@Bean`;工具 POJO 做 `@Component` 以便注入业务 Service,再 `toolkit.registerTool(toolsBean)`。或直接用 `agentscope-spring-boot-starter` + provider-specific starter(`agentscope-dashscope-spring-boot-starter` 等),配 `agentscope.openai.api-key=...`。
7. **每次 call 必传 RuntimeContext**:不传 `sessionId` 时所有请求共享 `defaultSessionId` 状态造成串台。多用户场景每次都 `RuntimeContext.builder().userId(...).sessionId(...).build()`。
8. **中间件/工具中读 AgentState**:用 `RuntimeContext.resolveAgentState(ctx, agent)` 而非 `agent.getAgentState()`——并发时后者返回最后一次活跃 session 状态,不确定。
9. **分布式文件系统 + 本地 StateStore 会 build 失败**:`filesystem(RemoteFilesystemSpec)` + 没换 stateStore/distributedStore → `build()` 抛 `IllegalStateException`(设计如此,别把状态留在某 pod 本地磁盘)。
10. **网络拉依赖**:沙箱无外网时 `mvn` 拉不到 agentscope;用可联网环境或预热本地仓库(`mvn dependency:get -Dartifact=io.agentscope:agentscope-harness:2.0.0-RC3`)。
11. **流里发 SSE**:`doOnNext` 可能在 Reactor 线程触发,`SseEmitter.send` 线程安全即可;异常用 `doOnError`/外层 try 包住,保证 `emitter.complete()`。RC3 后可从 `AgentResultEvent` 直接拿最终结果。
12. **OSS/NAS 走完 IAM 再上线**:`OssSnapshotSpec` 的 AK/SK 是平台凭证,用 RAM Role + STS 临时凭证更稳。

---

## 16. 最小可运行示例(Spring/Hulk,摘自本仓库)

```java
// 1) 模型 + 工具 + Agent 单例
@Configuration
class AgentConfig {
  @Bean Model model(@Value("${llm.api-key:}") String key) {
    return DashScopeChatModel.builder().apiKey(key).modelName("qwen-plus")
        .stream(true).formatter(new DashScopeChatFormatter()).build();
  }
  @Bean Toolkit toolkit(MyTools tools) { Toolkit t = new Toolkit(); t.registerTool(tools); return t; }
  @Bean ReActAgent agent(Model model, Toolkit toolkit) {
    return ReActAgent.builder().name("a").sysPrompt("……")
        .model(model).toolkit(toolkit)
        .stateStore(new JsonFileAgentStateStore(Paths.get(System.getProperty("user.home"), ".agentscope/sessions")))
        .maxIters(10).build();
  }
}

// 2) 工具(Spring Bean,可注入业务 Service)
@Component @RequiredArgsConstructor
class MyTools {
  private final BizService biz;
  @Tool(name="do_x", description="……", readOnly=false)
  public String doX(@ToolParam(name="p", description="…") String p, MyCtx ctx) {
    // ctx 经 RuntimeContext 注入;按需重建 ThreadLocal 后调 biz
    return biz.handle(p);
  }
}

// 3) 调用(流式 → SSE)。streamEvents 不接受 RuntimeContext;
//    流式 + per-call 上下文(多用户隔离/工具注入)用 stream(msgs, options, ctx)
RuntimeContext rc = RuntimeContext.builder().userId(uid).sessionId(sid).put(MyCtx.class, myCtx).build();
agent.stream(List.of(new UserMessage(userText)), streamOptions, rc)
     .doOnNext(ev -> { if (ev instanceof TextBlockDeltaEvent d) sseSend(d.getDelta()); })
     .blockLast();
```
> `streamOptions` 的确切类型以 jar 实测为准(`javap` harness builder);若暂不需 per-call 上下文,可直接用 `agent.streamEvents(new UserMessage(userText))`。

---

## 17. 关键类索引(2.0.0-RC3)

| 类/注解 | 包 |
|---|---|
| `Agent` / `ReActAgent` | `io.agentscope.core.agent` / `io.agentscope.core` |
| `HarnessAgent` / `DistributedStore` / `IsolationScope` | `io.agentscope.harness.agent` |
| `RuntimeContext` | `io.agentscope.core.agent` |
| `Msg` / `UserMessage` / `AssistantMessage` / `SystemMessage` / `ToolResultMessage` | `io.agentscope.core.message` |
| `ContentBlock` / `TextBlock` / `DataBlock` / `ThinkingBlock` / `ToolUseBlock` / `ToolResultBlock` / `HintBlock` | `io.agentscope.core.message` |
| `AgentEvent` / `AgentEventType` / `TextBlockDeltaEvent` / `ToolCallStartEvent` / `AgentResultEvent` / `CustomEvent` / `RequireUserConfirmEvent` | `io.agentscope.core.event` |
| `Toolkit` / `Tool` / `ToolParam` / `ToolBase` / `ToolCallParam` / `ToolGroup` | `io.agentscope.core.tool` |
| `McpClientBuilder` / `McpClientWrapper` | `io.agentscope.core.tool.mcp` |
| `Model` / `ChatModelBase` / `GenerateOptions` / `ChatResponse` / `ModelRegistry` | `io.agentscope.core.model` |
| `DashScopeChatModel` / `OpenAIChatModel` / `AnthropicChatModel` / `GeminiChatModel` / `OllamaChatModel` | `io.agentscope.extensions.model.<provider>` |
| `CredentialBase` / `ModelCard` | `io.agentscope.core.credential` |
| `AgentState` / `AgentStateStore` / `InMemoryAgentStateStore` / `JsonFileAgentStateStore` | `io.agentscope.core.state` |
| `RedisAgentStateStore` / `MysqlAgentStateStore` / `RedisDistributedStore` | `io.agentscope.extensions.redis` / `.mysql` |
| `PermissionMode` / `PermissionDecision` / `PermissionContextState` | `io.agentscope.core.permission` |
| `MiddlewareBase` / `OtelTracingMiddleware` | `io.agentscope.core.middleware` / `io.agentscope.core.tracing` |
| `CompactionConfig` | `io.agentscope.harness.agent.memory.compaction` |
| `DockerFilesystemSpec` 等沙箱 | `io.agentscope.harness.agent.sandbox.impl.*` |

> 验证某 API:`javap -classpath <…>/agentscope-core-2.0.0-RC3.jar io.agentscope.core.ReActAgent`(或 `...$Builder`)。

---

## 延伸阅读:集成层与参考

本手册已覆盖核心组件(§1–§12)、Harness 高级能力(§13)、上生产(§14)。以下为**未在本手册展开**的集成层与参考,按需查官网:

**Harness 深入**(§13 为精炼版,官网有完整细节)
- [架构](https://java.agentscope.io/v2/zh/docs/harness/architecture.html) / [工作区](https://java.agentscope.io/v2/zh/docs/harness/workspace.html) / [上下文压缩](https://java.agentscope.io/v2/zh/docs/harness/compaction.html) / [记忆](https://java.agentscope.io/v2/zh/docs/harness/memory.html) / [文件系统](https://java.agentscope.io/v2/zh/docs/harness/filesystem.html) / [沙箱](https://java.agentscope.io/v2/zh/docs/harness/sandbox.html) / [子 Agent](https://java.agentscope.io/v2/zh/docs/harness/subagent.html) / [技能](https://java.agentscope.io/v2/zh/docs/harness/skill.html) / [计划模式](https://java.agentscope.io/v2/zh/docs/harness/plan-mode.html) / [Channel](https://java.agentscope.io/v2/zh/docs/harness/channel.html)

**集成层**(未展开)
- [RAG 知识库](https://java.agentscope.io/v2/zh/integration/rag/index.html) — Simple/百炼/Dify/HayStack/RAGFlow
- [智能体协议](https://java.agentscope.io/v2/zh/integration/protocol/index.html) — A2A/AG-UI/Agent Protocol(对外暴露 agent)
- [记忆集成](https://java.agentscope.io/v2/zh/integration/memory/index.html) — Mem0/百炼记忆/ReMe
- [基础设施](https://java.agentscope.io/v2/zh/integration/infrastructure/index.html) — Higress 网关/Nacos 配置/Scheduler 调度
- [生态](https://java.agentscope.io/v2/zh/integration/ecosystem/index.html) — Studio 可视化/Chat Completions Web/在线训练

**参考**
- [V1 迁移指南](https://java.agentscope.io/v2/zh/docs/change-log.html) — 1.x→2.0 必须迁移/推荐迁移清单
- [FAQ](https://java.agentscope.io/v2/zh/docs/others/faq.html)

## 参考
- 官网 v2 文档:<https://java.agentscope.io/v2/zh/docs/>
- Release Notes:<https://java.agentscope.io/v2/zh/docs/others/release-notes.html>
- 上生产指南:<https://java.agentscope.io/v2/zh/docs/others/going-to-production.html>
- V1 迁移指南:<https://java.agentscope.io/v2/zh/docs/change-log.html>
- Maven 坐标:`io.agentscope:agentscope-harness` / `agentscope-core`(<https://central.sonatype.com/namespace/io.agentscope>)
