<!--
  Sync Impact Report
  ==================
  Version change: [template] → 1.0.0 (initial ratification)
  Modified principles: N/A (initial creation from CLAUDE.md)
  Added sections:
    - Core Principles: 8 principles (from CLAUDE.md §不可违背的原则)
    - Technical Stack Constraints (from CLAUDE.md §技术栈 + §模块结构)
    - Design Philosophy & Development Workflow (from CLAUDE.md §设计原则)
    - Governance
  Removed sections: None
  Templates requiring updates:
    - .specify/templates/plan-template.md ✅ (Constitution Check gate compatible, no changes needed)
    - .specify/templates/spec-template.md ✅ (no constitution-specific sections, no changes needed)
    - .specify/templates/tasks-template.md ✅ (no constitution-specific task types, no changes needed)
    - .specify/templates/checklist-template.md ⚠ pending (review for principle-aligned checklist items)
  Follow-up TODOs: None
-->

# OryxOS Constitution

## Core Principles

### I. 自实现 ReAct Loop

`ReActLoop` MUST 由项目自行实现，**不得**使用 Spring AI 的 Agent 抽象（如
`ChatClient.prompt().call()` 的自动工具执行）。核心循环约数十行 Java，完整掌握
Agent 工作机制，保留未来定制循环行为的空间。

**Rationale**: 第三方 Agent 抽象是黑盒——定制循环行为（如中间审计、提前终止、
动态工具集切换）时必然需要突破其限制。自实现 ReAct Loop 是 OryxOS 作为 Agent
OS 底座的基石。

### II. Spring AI 仅做协议转换与 Schema 生成

Spring AI 在 OryxOS 里 MUST 只承担两项职责：

1. LLM Provider 协议转换（吸收 OpenAI / Anthropic / Gemini 等各家格式差异）
2. `@Tool` 注解的 JSON Schema 生成

**MUST 禁用** Spring AI 的自动 tool 执行。Tool 的调度和执行完全由 `ReActLoop` +
`ToolExecutor` 控制。违反此原则会导致 tool 被调两次。

```java
// 禁止：不得用 Spring AI 自动执行 tool
chatClient.prompt(prompt).tools(tools).call().content();

// 正确：只用 Spring AI 做 LLM 调用，tool 调用结果自己处理
ChatResponse response = chatModel.call(new Prompt(messages, options));
```

### III. Provider 显式映射

多 Provider 并存时，**不得**靠扫描 Spring 容器里的 `ChatModel` Bean 类型来区分
Provider（因为所有 Provider 的 Bean 类型相同）。MUST 维护 `provider name → ChatModel`
的显式映射表：

```java
Map<String, ChatModel> providerMap = Map.of(
    "deepseek", deepseekChatModel,
    "qwen",     qwenChatModel,
    "kimi",     kimiChatModel
);
```

**Rationale**: 类型扫描在多个同类型 Bean 存在时不可靠，显式映射是唯一确定性的
Provider 路由方式。

### IV. 一个目录 = 一个 Agent；AGENT.md 归 ContextLoader

**一个目录 = 一个 Agent**：`.oryxos/agents/<name>/` 里 `AGENT.md` =
frontmatter（profile 配置）+ 正文（任务指令）；外加可选 `skills/*.md`（子指令）、
`scripts/`（脚本）、`REFERENCE.md`（参考）。

加载走**渐进式披露**：
- `AGENT.md` 正文由 `ContextLoader`/`PromptBuilder` 注入 system prompt
- 子指令 / 参考用 `read_file` 按需读
- 脚本用 `shell` 按需跑
- **没有**跨 Agent 的能力库、`use_skill`、全局能力索引

**一个 Agent 目录永远不是一个可执行 Tool**，不进 `ToolRegistry`、不放在
`oryxos-tool` 模块里。

### V. 审计表 Day One 写入

`tool_invocations` 和 `llm_calls` 两张审计表 MUST 在核心阶段就写入 SQLite。
不得以"日志够了"为由跳过落库。

**Rationale**: 可审计是 OryxOS 的核心差异化能力。事后从日志反解析审计数据的
成本远高于从一开始就结构化落库。

### VI. 禁用 SecurityManager，使用白名单沙箱

`SecurityManager` 在 JDK 17 起废弃、JDK 21 已不可用。Sandbox MUST 通过
`SandboxChecker` 的 Path / Pattern 白名单实现：

| 操作类型 | 白名单配置 | 说明 |
|---------|-----------|------|
| 文件操作 | `file.allowed_paths` | 路径白名单 |
| Shell | `shell.allowed_commands` | 命令首 token 白名单 |
| HTTP | `http.allowed_domains` | 域名通配符白名单 |

### VII. 同步执行模型

核心阶段 MUST 全程同步阻塞，配合 Java 21 Virtual Thread 处理并发。**不得引入**
Reactor / WebFlux / CompletableFuture 等异步编程模型。

SSE 流式响应等异步需求放扩展阶段实现。

**Rationale**: Virtual Thread 让同步代码获得异步 IO 的吞吐优势，同时保持代码的
可读性和可调试性。引入响应式编程会显著增加复杂度而无实际收益。

### VIII. Tool 模块三合一

内置 Tool、MCP Client MUST 合并在一个 `oryxos-tool` 模块，**不拆成多个模块**。
Agent 目录加载（`AGENT.md` 及子指令）归 `oryxos-core` 的 `ContextLoader`。

**Rationale**: Tool 的注册、发现、沙箱检查、执行是一个紧密耦合的流程。拆分模块
产生循环依赖风险，增加理解成本。

## Technical Stack Constraints

以下技术选型为项目级约束，所有 feature 的 plan MUST 在此框架内做技术决策：

| 组件 | 选型 | 约束级别 |
|------|------|---------|
| 语言 / 运行时 | Java 21（virtual thread） | MUST |
| 框架 | Spring Boot 3.x | MUST |
| LLM 调用 | Spring AI Alibaba（仅协议转换 + Schema 生成） | MUST |
| HTTP 服务 | Spring MVC + Java 21 Virtual Thread | MUST |
| 命令行 | Picocli | MUST |
| YAML 解析 | SnakeYAML | MUST |
| 持久化 | SQLite + Spring Data JPA | MUST |
| 日志 | Logback + SLF4J（结构化 JSON） | MUST |
| 构建 | Maven 多模块 | MUST |

**模块结构可按需演进**：模块划分跟随 Agent 的能力域，不锁死初始 9 个模块。
可以新建模块或调整模块边界，但 MUST 满足：

- 新建/改名在对应 feature 的 plan 里声明理由，并同步更新 CLAUDE.md 模块表与
  `docs/TechnicalSolution.md` §10
- 跨模块契约（接口 + 值对象）放 `oryxos-core`，由下游模块实现（依赖倒置）
- **禁止**模块间循环依赖

**敏感配置**：API key、MCP server 凭证 MUST 通过环境变量注入（如
`${DEEPSEEK_API_KEY}`），不得明文写在配置文件里。`ConfigLoader` 启动时 MUST
做必填项和格式校验，缺失或非法时给清晰报错，不静默失败。

**数据库迁移**：`hibernate.ddl-auto=update` 在 SQLite 上 `ALTER TABLE` 支持很弱。
表结构变更时 MUST 手动维护建表脚本或引入 Flyway，不得依赖 Hibernate 自动迁移。

## Design Philosophy & Development Workflow

### 设计原则

以下原则指导所有技术决策，plan 阶段的 "Constitution Check" MUST 逐条验证：

1. **底座优先于 Agent**：最重要的交付不是某个强大的 Agent，而是让任意 Agent 可靠
   运行的环境。新功能 MUST 先判断"这是底座能力还是单个 Agent 能力"——底座能力优先。

2. **自实现核心，复用管道**：ReAct 循环手写；LLM 协议适配委托给 Spring AI Alibaba。
   核心逻辑（循环调度、tool 执行、记忆管理）不进第三方框架的黑盒。

3. **一个目录 = 一个 Agent**：一个业务 Agent 由一个目录定义，不需要写 Java 代码。
   降低 Agent 创建门槛，让非 Java 开发者也能定义 Agent。

4. **对接开放标准**：工具用 MCP（Model Context Protocol），Agent 间协作用 A2A
   （Agent-to-Agent Protocol），Agent 目录借 Anthropic Agent Skills 的目录形态。

5. **无状态实例，状态外置**：Session 持久化到 SQLite，Memory 外置到文件系统。
   这是未来走向分布式架构而不需要大改设计的前提。

6. **安全是地基，不是补丁**：工具来源管控、最小权限、强制沙箱白名单、凭证走环境
   变量、完整审计记录从第一天就写入 SQLite。

7. **分阶段克制**：先构建最小完整的运行时内核（核心阶段）；治理和分布式基础设施
   在真实使用数据验证后再做（扩展阶段）。核心阶段不做认证、SSE、WebSocket、限流、
   RBAC。

### 常见陷阱（Plan & Implement 必查）

| 陷阱 | 症状 | 修复 |
|------|------|------|
| Spring AI 自动执行 tool | Tool 被调两次 | 禁用自动执行，由 `ToolExecutor` 接管 |
| Provider 靠类型扫描区分 | 多 Provider 路由错乱 | 改用显式 `Map<String, ChatModel>` |
| Agent 目录放进 Tool 模块 | 被当 Tool 注册，执行报错 | 归 `ContextLoader` |
| 审计只写日志不落库 | 扩展阶段需反解析日志 | 核心阶段就写 SQLite |
| 依赖 Hibernate 自动迁移 | SQLite ALTER TABLE 报错 | 手动建表脚本或 Flyway |
| ReAct Loop 用异步 | 复杂度激增 | 保持同步 + Virtual Thread |
| MEMORY.md 超 4000 字不截断 | 超 context window | `LongTermMemory.truncateIfNeeded()` |
| Tool 模块拆成多个 | 模块间循环依赖 | 合并为一个 `oryxos-tool` |

## Governance

### Amendment Procedure

1. 任何原则的修改、新增、删除 MUST 通过 PR 提交，并在 PR 描述中说明动机和影响
   范围。
2. 修改已 Ratified 的原则 MUST 获得至少一次 review 批准。
3. 修改 MUST 同步更新本文件的版本号，并在 Sync Impact Report 注释中记录变更。
4. 受影响的模板文件（plan、spec、tasks、checklist）MUST 在同一 PR 中更新。

### Versioning Policy

Constitution 版本遵循语义化版本规范：

- **MAJOR**: 原则删除、含义重新定义、不向后兼容的治理变更
- **MINOR**: 新增原则/章节、实质性扩展指导
- **PATCH**: 措辞澄清、拼写修正、非语义性调整

### Compliance Review

- 每个 feature 的 `plan.md` 中的 **Constitution Check** 章节 MUST 逐条验证
  Core Principles 的合规性
- 任何违规 MUST 在 plan 的 Complexity Tracking 表格中说明理由，否则 plan 不通过
- Code review MUST 对照常见陷阱表验证实现

**Version**: 1.0.0 | **Ratified**: 2026-08-02 | **Last Amended**: 2026-08-02
