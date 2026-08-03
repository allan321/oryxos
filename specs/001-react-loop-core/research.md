# Research: OryxOS 核心 ReAct Loop

**Feature**: 001-react-loop-core
**Date**: 2026-08-02

## 1. Spring AI Alibaba Function Calling 适配

**Decision**: 使用 `SpringAiAlibabaChatModel.call(Prompt)` 进行 LLM 调用，手动解析
`ChatResponse.getOutput().getContent()` 和 `ChatResponse.getResults()` 中的 tool calls，
不依赖 `ChatClient.prompt().tools().call()` 的自动执行。

**Rationale**: Spring AI Alibaba 实现了 OpenAI 兼容的协议层，
`ChatModel.call()` 返回标准 `ChatResponse`，其中包含 `Generation` 列表，
每个 `Generation` 可能包含 `AssistantMessage`（含 `toolCalls` 字段）。
OryxOS 的 `ProviderService` 包装 `ChatModel.call()`，由 `ReActLoop`
自行检查 tool calls 并调度 `ToolExecutor`。

**Alternatives considered**:
- 直接用 OkHttp/HttpClient 调 LLM API: 放弃了协议适配层的价值，每个 Provider 要自己写请求构造和响应解析
- 用 Spring AI 的 `ChatClient`: 默认开启自动 tool 执行，需要额外配置禁用，增加了"踩坑"概率

**Key insight**: `ChatModel.call(Prompt)` 是最底层的 API，不触发任何自动行为。
返回的 `ChatResponse` 中 `getMetadata()` 可获取 token 用量，正好用于写入
`llm_calls` 审计表。

## 2. ReAct Loop 核心实现模式

**Decision**: 采用经典 while-loop 模式，伪代码:

```
while (iteration < maxIterations) {
    response = providerService.call(messages, tools)
    write llm_calls audit record

    if (response has no tool calls) {
        return response.content  // 最终回复
    }

    for (toolCall in response.toolCalls) {
        result = toolExecutor.execute(toolCall)
        write tool_invocations audit record
        messages.append(toolResult)
    }
}
return "达到最大迭代次数，未完成"  // 兜底
```

**Rationale**: 这是 Agent 领域的标准实现模式（参考 Anthropic、OpenAI 官方示例）。
约 50-80 行 Java 代码，完全掌握行为。关键设计点:
- Tool Call 失败时将错误信息传回 LLM 自修正（Constitution + clarify Q1）
- 空返回自动重试 1 次（clarify Q2）
- LLM 超时直接报错不重试（clarify Q3）
- 每次迭代写入审计记录（Constitution V）

**Alternatives considered**:
- 用 Spring AI 的 `ToolRunController`: 这正是要避免的——它会自动执行 tool，导致双次执行
- LangChain4j: 引入额外的 Agent 框架，同样有黑盒问题

## 3. Virtual Thread 配置

**Decision**: Spring Boot 3.x 中配置 `spring.threads.virtual.enabled=true` 即可启用
Virtual Thread。Tomcat 请求处理、`@Async` 方法、`TaskExecutor` 都自动使用 Virtual Thread。
对于纯粹同步阻塞代码无需特殊处理——`synchronized`/`Thread.sleep()` 在 Virtual Thread
上自动让出载体线程。

**Rationale**: Spring Boot 3.2+ 对 Virtual Thread 有正式支持。OryxOS 的核心循环
是同步阻塞的（LLM HTTP 调用、Tool 执行），Virtual Thread 让线程在 IO 等待时
自动释放 OS 线程，无需改代码。

**Configuration**:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```

## 4. Picocli + Spring Boot 集成

**Decision**: Picocli 提供 `@Command` 注解定义 CLI，通过 `CommandLine.run()` 在
`main()` 或 Spring Boot `CommandLineRunner` 中启动。不依赖 Picocli 的 Spring
集成 starter（其质量不稳定），直接手动调用。

**Rationale**: Picocli 的 Spring Boot Starter 历史上多次出现兼容性问题。
直接使用 Picocli 核心 JAR + 手动在 main 方法中构造 `CommandLine` 对象，从
Spring 容器中获取依赖注入到 Command。

**关键命令**:
- `oryxos init` — 初始化 `.oryxos/` 工作区
- `oryxos chat [--agent <name>]` — 启动交互式对话
- `oryxos status` — 查看运行状态

## 5. SQLite + Spring Data JPA 注意事项

**Decision**: 使用 SQLite JDBC 驱动 + Hibernate `SQLiteDialect`（社区提供）。
表创建用 `schema.sql` 显式维护，**禁用** `hibernate.ddl-auto=update`。

**Rationale**: SQLite 的 `ALTER TABLE` 支持极弱（不支持 DROP COLUMN、ALTER COLUMN 等）。
Spring Data JPA 的自动 DDL 在 SQLite 上不稳定。Constitution 明确要求手动维护
建表脚本或引入 Flyway。

**Dialect 方案**: 使用 `org.sqlite.hibernate.dialect.SQLiteDialect`（来自
`com.github.gwenn:sqlite-dialect` Maven 依赖）。

**初始表结构**（schema.sql）:
```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id VARCHAR(255) PRIMARY KEY,
    profile_name VARCHAR(255),
    channel VARCHAR(50),
    user_id VARCHAR(255),
    messages_json TEXT,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    archived_at TIMESTAMP
);

CREATE TABLE IF NOT EXISTS tool_invocations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id VARCHAR(255),
    tool_name VARCHAR(100),
    input_json TEXT,
    result_json TEXT,
    success BOOLEAN,
    error_message TEXT,
    duration_ms BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS llm_calls (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id VARCHAR(255),
    provider VARCHAR(100),
    model VARCHAR(100),
    prompt_tokens INTEGER,
    completion_tokens INTEGER,
    total_tokens INTEGER,
    duration_ms BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 6. OryxTool 接口设计

**Decision**: 采用 `org.springframework.ai.tool.annotation.Tool` 注解的 JsonSchema
生成 + OryxOS 自定义 `OryxTool` 接口包装，但不使用 Spring AI 的 Tool 执行机制。

**Tool 注册流程**:
1. Spring 容器扫描 `@Tool` 注解的 Bean
2. `ToolRegistry` 提取每个 Tool 的 name、description、parameter schema（用
   Spring AI 的 `ToolMethodMetadata` / `SchemaGenerator`）
3. 统一包装为 `OryxTool` 实现（包含 `execute(JsonNode input)` 方法）
4. `ToolExecutor.execute(toolName, params)` → 查 `ToolRegistry` → 调
   `OryxTool.execute()` → 返回 `ToolResult`

**Rationale**: 复用 Spring AI 的 Schema 生成（Principle II 允许的范围），但不让它
执行 Tool。`ToolRegistry` 是 OryxOS 自己的抽象，保持对 Tool 执行生命周期的完全控制。

## 7. Agent Profile 加载

**Decision**: 实现 `AgentLoader.deriveProfile(agentDir)` 读取 `AGENT.md` 的
YAML frontmatter，映射为 `Profile` 对象。Bootstrap 文件（AGENTS.md/SOUL.md/USER.md）
的从 `ContextLoader` 加载——本阶段简化为：将 `AGENT.md` 正文直接作为 system
prompt 注入。

**Rationale**: Constitution Principle IV 要求 profile 从 `AGENT.md` frontmatter
派生。核心阶段先实现最小加载链路：frontmatter → Profile，正文 → system prompt。
Bootstrap 文件的合并加载、渐进式披露的完整实现放后续 feature。

**Profile 字段映射**（from `AGENT.md` frontmatter → `Profile` POJO）:
```
name → profile.name
identity.agent_name → profile.agentName
identity.prompt → profile.systemPrompt (正文)
provider.name → profile.providerName
provider.model → profile.model
provider.temperature → profile.temperature
provider.api_key → 从环境变量解析 ${VAR}
tools → profile.toolNames
settings.max_iterations → profile.maxIterations
settings.max_history_turns → profile.maxHistoryTurns
```
