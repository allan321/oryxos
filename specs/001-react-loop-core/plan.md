# Implementation Plan: OryxOS 核心 ReAct Loop

**Branch**: `001-react-loop-core` | **Date**: 2026-08-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-react-loop-core/spec.md`

## Summary

实现 OryxOS 底座的核心运行时：自实现的 ReAct Loop（Reasoning + Acting 循环）、
PromptBuilder（System Prompt + 对话历史 + Tool 列表组装）、ToolExecutor（Tool
调度与执行）、Session 管理（创建、持久化、恢复），以及 Picocli 命令行入口
（`oryxos chat` 交互式多轮对话）。这是 OryxOS 的最小完整运行时内核。

## Technical Context

**Language/Version**: Java 21 (virtual thread 处理并发 IO)

**Primary Dependencies**:
- Spring Boot 3.x (DI 容器 + MVC)
- Spring AI Alibaba (仅 LLM 协议转换 + `@Tool` JSON Schema 生成)
- Picocli (命令行解析)
- SnakeYAML (配置解析)
- SQLite + Spring Data JPA (持久化)
- Logback + SLF4J (结构化 JSON 日志)

**Storage**: SQLite (sessions, tool_invocations, llm_calls 三张表)

**Testing**: JUnit 5 + Mockito (单元测试); Spring Boot Test (集成测试)

**Target Platform**: Linux/macOS/Windows 服务器, JDK 21+

**Project Type**: Java CLI + 库 (Spring Boot 多模块 Maven 项目)

**Performance Goals**:
- 无 Tool 调用的对话端到端 ≤5s
- 含 1 次 Tool 调用的对话端到端 ≤10s
- 单进程内存 ≤512MB

**Constraints**:
- 同步阻塞 + Virtual Thread (不引入 Reactor/WebFlux/CompletableFuture)
- ReAct Loop 必须自实现 (不用 Spring AI Agent 抽象)
- 审计表 (tool_invocations, llm_calls) Day One 写入 SQLite
- Provider 显式映射 (不靠类型扫描)
- Tool 本阶段以信任模式运行 (SandboxChecker 放后续 feature)

**Scale/Scope**: 单用户 CLI, 核心阶段, 约 4 个 Maven 模块

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. 自实现 ReAct Loop | ✅ PASS | `ReActLoop` 自行实现，不使用 Spring AI 自动 Tool 执行 |
| II. Spring AI 仅协议转换 + Schema 生成 | ✅ PASS | Spring AI 只用于 `ChatModel.call()` 和 `@Tool` Schema 生成；Tool 执行由 `ToolExecutor` 接管 |
| III. Provider 显式映射 | ✅ PASS | `ProviderService` 维护 `Map<String, ChatModel>` 显式映射 |
| IV. 一个目录 = 一个 Agent | ✅ PASS | Agent Profile 从 `AGENT.md` frontmatter 加载；不进 ToolRegistry |
| V. 审计表 Day One 写入 | ✅ PASS | `tool_invocations` + `llm_calls` 在每次操作后同步写入 |
| VI. 禁用 SecurityManager | ✅ PASS | 本阶段 Tool 信任模式；SandboxChecker 放后续 feature |
| VII. 同步执行模型 | ✅ PASS | 全程同步阻塞 + Virtual Thread |
| VIII. Tool 模块三合一 | ✅ PASS | 内置 Tool 的接口和基础实现放在 `oryxos-tool` 模块 |

**Gate Result**: ALL PASS — 无违规，无需 Complexity Tracking。

## Project Structure

### Documentation (this feature)

```text
specs/001-react-loop-core/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (API contracts)
└── tasks.md             # Phase 2 output (/speckit-tasks - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
oryxos/
├── oryxos-core/         # 核心抽象：OryxTool, Session, Profile,
│   │                    #   ReActLoop, PromptBuilder, ToolExecutor
│   ├── src/main/java/com/oryxos/core/
│   │   ├── model/       # Session, Message, Profile, ToolCall, ToolResult
│   │   ├── loop/        # ReActLoop (核心循环实现)
│   │   ├── prompt/      # PromptBuilder (组装 system prompt + 历史 + tools)
│   │   ├── tool/        # ToolExecutor, ToolRegistry, SandboxChecker 桩
│   │   ├── agent/       # AgentService, ContextLoader (简化版)
│   │   ├── session/     # SessionManager (create/save/load)
│   │   ├── persistence/ # JPA Repository 接口定义
│   │   └── config/      # ConfigLoader (YAML → Profile)
│   └── src/test/java/com/oryxos/core/
│       ├── loop/        # ReActLoop 单元测试
│       ├── prompt/      # PromptBuilder 单元测试
│       └── tool/        # ToolExecutor 单元测试
│
├── oryxos-provider/     # ProviderService, 显式 provider 映射, FC 适配
│   ├── src/main/java/com/oryxos/provider/
│   │   ├── ProviderService.java
│   │   └── FunctionCallingAdapter.java
│   └── src/test/java/com/oryxos/provider/
│
├── oryxos-channel-cli/  # CLI Channel: oryxos chat 实现
│   ├── src/main/java/com/oryxos/channel/cli/
│   │   ├── CliChannel.java         # 交互式命令行主循环
│   │   ├── CliCommand.java         # @Command 定义 (Picocli)
│   │   └── CliFormatter.java       # 输出格式化
│   └── src/test/java/com/oryxos/channel/cli/
│
└── oryxos-boot/         # Spring Boot 启动模块 + 自动配置
    ├── src/main/java/com/oryxos/boot/
    │   ├── OryxosApplication.java  # @SpringBootApplication 主类
    │   └── OryxosAutoConfiguration.java
    └── src/main/resources/
        └── application.yml
```

**Structure Decision**: Maven 多模块——4 个模块覆盖核心阶段。`oryxos-core`
放所有接口和核心实现，其他模块通过依赖倒置实现 core 的接口。后续 feature
可以按需添加模块（如 `oryxos-memory`、`oryxos-web`）。
