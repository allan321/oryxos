# Tasks: OryxOS 核心 ReAct Loop

**Input**: Design documents from `specs/001-react-loop-core/`

**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, quickstart.md

**Tests**: Not explicitly requested in spec; test tasks omitted. Add via `/speckit-tasks` re-run if needed.

**Organization**: Tasks grouped by user story for independent implementation and testing.

## Format: `- [ ] [ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Exact file paths in every description

## Path Conventions

- `oryxos-core/src/main/java/com/oryxos/core/` — core abstractions
- `oryxos-provider/src/main/java/com/oryxos/provider/` — LLM provider impl
- `oryxos-channel-cli/src/main/java/com/oryxos/channel/cli/` — CLI channel
- `oryxos-boot/src/main/java/com/oryxos/boot/` — Spring Boot entry point
- `oryxos-boot/src/main/resources/` — config files

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Maven 多模块骨架 + 基础配置

- [ ] T001 Create Maven parent POM (`pom.xml`) with modules: oryxos-core, oryxos-provider, oryxos-channel-cli, oryxos-boot; Java 21, Spring Boot 3.x, Picocli, SnakeYAML, SQLite dependencies
- [ ] T002 [P] Create `oryxos-core/pom.xml` with dependencies: Spring Boot Starter, Spring AI Alibaba, SnakeYAML, Jackson
- [ ] T003 [P] Create `oryxos-provider/pom.xml` with dependencies: Spring AI Alibaba, oryxos-core
- [ ] T004 [P] Create `oryxos-channel-cli/pom.xml` with dependencies: Picocli, JLine (readline), oryxos-core
- [ ] T005 [P] Create `oryxos-boot/pom.xml` with dependencies: Spring Boot Starter, Spring Data JPA, SQLite JDBC, oryxos-core, oryxos-provider, oryxos-channel-cli
- [ ] T006 Create `oryxos-boot/src/main/resources/application.yml` with: spring.threads.virtual.enabled=true, SQLite datasource config, logging config
- [ ] T007 [P] Create `oryxos-boot/src/main/resources/schema.sql` with DDL for sessions, tool_invocations, llm_calls tables (from data-model.md)
- [ ] T008 [P] Create `oryxos-boot/src/main/resources/logback-spring.xml` with structured JSON layout

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core abstractions shared by ALL user stories. MUST complete before Phase 3.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T009 Create `OryxTool` interface in `oryxos-core/src/main/java/com/oryxos/core/tool/OryxTool.java` — getName(), getDescription(), getParametersJsonSchema(), execute(JsonNode)
- [ ] T010 [P] Create `ToolResult` record in `oryxos-core/src/main/java/com/oryxos/core/tool/ToolResult.java` — toolCallId, success, content, errorMessage
- [ ] T011 [P] Create `Message` record in `oryxos-core/src/main/java/com/oryxos/core/model/Message.java` — role, content, toolCalls, toolCallId, timestamp
- [ ] T012 [P] Create `ToolCallData` record in `oryxos-core/src/main/java/com/oryxos/core/model/ToolCallData.java` — id, name, arguments
- [ ] T013 [P] Create `Profile` model in `oryxos-core/src/main/java/com/oryxos/core/model/Profile.java` — all fields from data-model.md Profile entity
- [ ] T014 Create `ToolRegistry` in `oryxos-core/src/main/java/com/oryxos/core/tool/ToolRegistry.java` — register/unregister OryxTool, getTool(name), listAll(), toFunctionCallingFormat()
- [ ] T015 Create `ProviderService` interface in `oryxos-core/src/main/java/com/oryxos/core/provider/ProviderService.java` — call(messages, tools, profile) returns ChatResponse
- [ ] T016 [P] Create `SessionEntity` JPA entity in `oryxos-core/src/main/java/com/oryxos/core/model/SessionEntity.java` — maps to sessions table
- [ ] T017 [P] Create `ToolInvocationEntity` JPA entity in `oryxos-core/src/main/java/com/oryxos/core/model/ToolInvocationEntity.java` — maps to tool_invocations table
- [ ] T018 [P] Create `LlmCallEntity` JPA entity in `oryxos-core/src/main/java/com/oryxos/core/model/LlmCallEntity.java` — maps to llm_calls table
- [ ] T019 Create `SessionRepository` interface in `oryxos-core/src/main/java/com/oryxos/core/persistence/SessionRepository.java` — extends JpaRepository<SessionEntity, String>
- [ ] T020 [P] Create `ToolInvocationRepository` in `oryxos-core/src/main/java/com/oryxos/core/persistence/ToolInvocationRepository.java` — extends JpaRepository<ToolInvocationEntity, Long>
- [ ] T021 [P] Create `LlmCallRepository` in `oryxos-core/src/main/java/com/oryxos/core/persistence/LlmCallRepository.java` — extends JpaRepository<LlmCallEntity, Long>

**Checkpoint**: Foundation ready — core abstractions, entities, repositories in place. User story implementation can begin.

---

## Phase 3: User Story 1 — 启动 CLI 并开始交互式对话 (Priority: P1) 🎯 MVP

**Goal**: 用户通过 `oryxos chat` 启动 CLI 交互式对话，Agent 调用 LLM 返回回复（无 Tool 调用），支持多轮对话。

**Independent Test**: `oryxos chat` → 输入"你好" → Agent 返回文字回复 → 继续输入第 2 条 → Agent 基于上文回复 → `/exit` 正常退出。

### Implementation for User Story 1

- [ ] T022 [US1] Implement `ProviderServiceImpl` in `oryxos-provider/src/main/java/com/oryxos/provider/ProviderServiceImpl.java` — maintain Map<String, ChatModel>, create ChatModel per provider config, implement call()
- [ ] T023 [P] [US1] Implement `ConfigLoader` in `oryxos-core/src/main/java/com/oryxos/core/config/ConfigLoader.java` — load oryxos-boot application.yml, parse env vars (${VAR} format), validation with clear error messages
- [ ] T024 [US1] Implement `AgentLoader.deriveProfile()` in `oryxos-core/src/main/java/com/oryxos/core/agent/AgentLoader.java` — read AGENT.md, parse YAML frontmatter, map to Profile object, resolve ${VAR} env placeholders
- [ ] T025 [US1] Implement `PromptBuilder` in `oryxos-core/src/main/java/com/oryxos/core/prompt/PromptBuilder.java` — assemble: [system prompt from Profile] + [conversation history] + [available tools list from ToolRegistry]
- [ ] T026 [US1] Implement `ReActLoop` (basic, no tool call handling) in `oryxos-core/src/main/java/com/oryxos/core/loop/ReActLoop.java` — while loop skeleton: call LLM → if no tool calls return response; stubs for tool handling (filled in US2)
- [ ] T027 [US1] Implement `OryxosAutoConfiguration` in `oryxos-boot/src/main/java/com/oryxos/boot/OryxosAutoConfiguration.java` — @Bean definitions for ProviderService, ToolRegistry, ReActLoop, PromptBuilder, ConfigLoader
- [ ] T028 [US1] Implement `OryxosApplication` in `oryxos-boot/src/main/java/com/oryxos/boot/OryxosApplication.java` — @SpringBootApplication main class, Picocli CommandLine.run() in main()
- [ ] T029 [US1] Implement `CliCommand` in `oryxos-channel-cli/src/main/java/com/oryxos/channel/cli/CliCommand.java` — Picocli @Command for chat/init/status/profile subcommands
- [ ] T030 [US1] Implement `CliChannel` in `oryxos-channel-cli/src/main/java/com/oryxos/channel/cli/CliChannel.java` — JLine readline loop: read input → call ReActLoop → print response → repeat; handle /exit and Ctrl+C
- [ ] T031 [US1] Implement `oryxos init` command logic — create .oryxos/ directory tree (agents/, memory/, sessions/, logs/), write AGENTS.md/SOUL.md/USER.md stubs
- [ ] T032 [US1] Implement `oryxos status` command logic — list profiles, show provider status, check DB connectivity
- [ ] T033 [US1] Implement max_iterations check in ReActLoop — terminate when iteration >= maxIterations, return partial result with reason (FR-010)

**Checkpoint**: `oryxos chat` 可启动、对话、退出。多轮上下文连贯。MVP 达成！

---

## Phase 4: User Story 2 — Agent 自动调用 Tool 并返回结果 (Priority: P1)

**Goal**: Agent 识别 LLM 的 Tool Call，执行 Tool，将结果传回 LLM，生成基于 Tool 结果的最终回复。

**Independent Test**: 在 `.oryxos/` 放置 test.txt → `oryxos chat` → "读一下 test.txt" → Agent 调用 read_file → 基于文件内容回复。

### Implementation for User Story 2

- [ ] T034 [US2] Implement `ToolExecutor` in `oryxos-core/src/main/java/com/oryxos/core/tool/ToolExecutor.java` — lookup tool in ToolRegistry, call OryxTool.execute(), return ToolResult (success/failure)
- [ ] T035 [US2] Implement `SandboxChecker` stub in `oryxos-core/src/main/java/com/oryxos/core/tool/SandboxChecker.java` — trust mode: allow all operations, log warning "SandboxChecker not enforced"; real implementation in future feature
- [ ] T036 [US2] Implement `FileTools.readFile` in `oryxos-core/src/main/java/com/oryxos/core/tool/FileTools.java` — @Tool annotated class: read_file(name, description, schema), read file content, return content
- [ ] T037 [P] [US2] Implement `FileTools.writeFile` in `oryxos-core/src/main/java/com/oryxos/core/tool/FileTools.java` — write_file tool: write content to file path
- [ ] T038 [P] [US2] Implement `FileTools.listDir` in `oryxos-core/src/main/java/com/oryxos/core/tool/FileTools.java` — list_dir tool: list directory contents
- [ ] T039 [US2] Implement `@Tool` annotation scanning in `ToolRegistry` — scan Spring beans with @Tool, extract name/description/schema via Spring AI reflection, register as OryxTool wrapper
- [ ] T040 [US2] Extend `ReActLoop` tool call handling in `oryxos-core/src/main/java/com/oryxos/core/loop/ReActLoop.java` — parse toolCalls from ChatResponse → call ToolExecutor for each sequentially (not parallel) → convert ToolResult to ToolMessage → append to messages → loop back to LLM
- [ ] T041 [US2] Implement LLM audit writer — after each ChatModel.call(), write LlmCallEntity via LlmCallRepository (FR-007, Constitution V)
- [ ] T042 [US2] Implement Tool audit writer — after each OryxTool.execute(), write ToolInvocationEntity via ToolInvocationRepository (FR-008, Constitution V)
- [ ] T043 [US2] Implement Tool Call param mismatch handling (FR-006a) — catch schema validation errors → build error ToolResult → append to messages as tool feedback → LLM self-corrects in next iteration
- [ ] T044 [US2] Implement LLM empty return handling (FR-006b) — detect empty content + no toolCalls → retry 1x with "Previous response was empty, please generate a response" → if still empty, error to user
- [ ] T045 [US2] Implement LLM timeout handling (FR-006c) — set timeout on ChatModel call (configurable, default 30s) → on timeout, return error message to user, no auto-retry

**Checkpoint**: Agent 可以调用 Tool 并基于结果回复。审计表写入完整。错误处理就绪。

---

## Phase 5: User Story 3 — 会话持久化与恢复 (Priority: P2)

**Goal**: 退出 CLI 时保存对话历史，重新启动可恢复 Session 继续对话。

**Independent Test**: 3 轮对话 → 退出 → 重启指定 Session → Agent 记住前 3 轮内容。

### Implementation for User Story 3

- [ ] T046 [US3] Implement `SessionManager` in `oryxos-core/src/main/java/com/oryxos/core/session/SessionManager.java` — create(profileName, channel, userId) → SessionEntity, save(sessionId, messages), load(sessionId) → List<Message>
- [ ] T047 [US3] Implement Session auto-save on exit — in CliChannel cleanup/shutdown hook, serialize messages to JSON, call SessionManager.save()
- [ ] T048 [US3] Implement Session restore on startup — add `--session <id>` flag to CliCommand, call SessionManager.load(), inject messages as conversation history into ReActLoop
- [ ] T049 [US3] Implement `max_history_turns` truncation — in SessionManager.load(), if messages count > maxHistoryTurns*2 (user+assistant pairs), keep only most recent N turns
- [ ] T050 [US3] Implement `oryxos session list` command — query SessionRepository.findByStatus("active"), display session_id, profile_name, message count, last_active_at

**Checkpoint**: 跨进程恢复对话。历史截断生效。Session 列表可见。

---

## Phase 6: User Story 4 — 多 Agent Profile 切换 (Priority: P3)

**Goal**: 用户可以用 `--agent <name>` 启动不同 Agent Profile。

**Independent Test**: 创建 2 个 AGENT.md → 分别启动 → 验证人设/配置不同。

### Implementation for User Story 4

- [ ] T051 [US4] Extend `AgentLoader` to scan `.oryxos/agents/*/AGENT.md` — list all agent directories, call deriveProfile() for each, return Map<String, Profile>
- [ ] T052 [US4] Implement `--agent <name>` flag in `CliCommand` — parse argument, look up profile from AgentLoader, pass to ReActLoop/ProviderService
- [ ] T053 [US4] Implement default agent fallback — when --agent not specified, look for `.oryxos/agents/default/AGENT.md`, if not found, use built-in minimal default profile
- [ ] T054 [US4] Implement agent not found error — when --agent specifies nonexistent agent, print clear error: "Agent 'X' not found. Available agents: [list]" and exit
- [ ] T055 [US4] Implement `oryxos profile list` command — iterate AgentLoader scan results, display name, description, provider, model for each

**Checkpoint**: 多 Agent Profile 并存，独立运行。

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Input validation, edge case hardening, documentation.

- [ ] T056 [P] Implement blank input handling (FR-014) in `CliChannel` — trim input, if empty: skip, re-display prompt, do NOT send to LLM
- [ ] T057 [P] Implement `/exit` command handling in `CliChannel` — detect "/exit", print goodbye message, save session, exit cleanly
- [ ] T058 Add structured logging to ReActLoop — log at INFO: iteration count, LLM call duration, tool execution results; DEBUG: full prompt contents
- [ ] T059 Run quickstart.md scenario 1-7 validation — verify all scenarios pass; also check SC-005 (memory ≤512MB via jconsole/htop) and SC-006 (audit coverage: run scenario 4 then `SELECT COUNT(*) FROM llm_calls` and `tool_invocations` to confirm every LLM/Tool call has a row)
- [ ] T060 [P] Add project README.md with build instructions (mvn clean install) and quickstart link

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 (Maven modules exist) — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational — MVP
- **User Story 2 (Phase 4)**: Depends on US1 (ReActLoop skeleton exists to extend) + Foundational
- **User Story 3 (Phase 5)**: Depends on US1 (Session data exists to persist)
- **User Story 4 (Phase 6)**: Depends on US1 (profile loading in place)
- **Polish (Phase 7)**: Depends on all desired user stories

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational — No dependencies on other stories
- **US2 (P1)**: Depends on US1 (ReActLoop, ToolRegistry in place)
- **US3 (P2)**: Depends on US1 (messages exist to persist); independent of US2
- **US4 (P3)**: Depends on US1 (AgentLoader, Profile in place); independent of US2/US3

### Within Each User Story

1. Core abstractions → Implementation classes
2. Provider/Config → ReActLoop skeleton → CLI wiring
3. Tools → ToolExecutor → ReActLoop extension
4. Session save/load → CLI integration → truncation
5. AgentLoader scan → CLI flag → fallback/error

---

## Parallel Opportunities

### Phase 1 (Setup)
```bash
# Launch all module pom.xml creation in parallel:
Task T002: "Create oryxos-core/pom.xml"
Task T003: "Create oryxos-provider/pom.xml"
Task T004: "Create oryxos-channel-cli/pom.xml"
Task T005: "Create oryxos-boot/pom.xml"
Task T007: "Create schema.sql"
Task T008: "Create logback-spring.xml"
```

### Phase 2 (Foundational)
```bash
# Launch all model/entity files in parallel:
Task T010: "Create ToolResult record"
Task T011: "Create Message record"
Task T012: "Create ToolCallData record"
Task T013: "Create Profile model"
Task T016: "Create SessionEntity"
Task T017: "Create ToolInvocationEntity"
Task T018: "Create LlmCallEntity"
Task T020: "Create ToolInvocationRepository"
Task T021: "Create LlmCallRepository"
```

### Phase 4 (US2)
```bash
# Launch all FileTools methods in parallel:
Task T036: "Implement FileTools.readFile"
Task T037: "Implement FileTools.writeFile"
Task T038: "Implement FileTools.listDir"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T008)
2. Complete Phase 2: Foundational (T009-T021)
3. Complete Phase 3: User Story 1 (T022-T033)
4. **STOP and VALIDATE**: `oryxos chat` works — can converse, multi-turn context OK, `/exit` clean
5. Demo/deploy MVP

### Incremental Delivery

1. MVP (US1) → Deploy/Demo: basic conversational Agent
2. +US2 → Deploy/Demo: Agent can use tools, full ReAct cycle, audit trail
3. +US3 → Deploy/Demo: sessions survive restarts
4. +US4 → Deploy/Demo: multiple Agents coexist
5. Polish → Final release

### Parallel Team Strategy (if 2+ developers)

1. Team completes Setup + Foundational together
2. Once Foundational done:
   - **Developer A**: User Story 1 (critical path — US2/US3/US4 all depend on it)
   - Once US1 skeleton ready: Developer B starts US2 (ToolExecutor, FileTools)
3. After US1 stable:
   - Developer A: User Story 3 (Session persistence)
   - Developer B: User Story 4 (Multi-profile) OR continue US2

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Stop at any checkpoint to validate story independently
- Commit after each task or logical group
- File paths are relative to repo root: `oryxos-<module>/src/...`
- All tasks use Constitution-compliant patterns (sync + Virtual Thread, explicit provider mapping, self-implemented ReAct Loop)
