# Feature Specification: OryxOS 核心 ReAct Loop

**Feature Branch**: `001-react-loop-core`

**Created**: 2026-08-02

**Status**: Draft

**Input**: User description: "实现 OryxOS 的核心 ReAct Loop，包括 PromptBuilder、ToolExecutor、Session 管理，支持通过 CLI 交互式多轮对话"

## Clarifications

### Session 2026-08-02

- Q: LLM 返回的 Tool Call 参数与 Schema 不匹配时，系统应该重试还是直接失败？ → A: 把格式错误信息传回 LLM 让它修正后重新生成 Tool Call（计为一次迭代，纳入 max_iterations 计数），尽可能在较少次数内修正参数格式。
- Q: LLM 返回内容为空（无 text 也无 tool call）或格式无法解析时，系统应该自动重试还是直接向用户报错？ → A: 自动重试 1 次（告知 LLM 上次返回为空请重新生成），仍失败则向用户报错。
- Q: LLM 调用超时（如超过 30 秒无响应）时，系统应该自动重试还是直接向用户报错？ → A: 直接向用户报错告知超时，不做自动重试，用户可手动重发消息。

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 启动 CLI 并开始交互式对话 (Priority: P1)

用户通过命令行启动 OryxOS 交互式对话，输入消息后，Agent 调用 LLM 进行
推理，如无需调用 Tool 则直接返回回复。用户可以看到 Agent 的回复并继续
下一轮对话，形成多轮交互。

**Why this priority**: 这是整个系统的最小可用单元——用户能跟 Agent 对话
是最基本的能力，没有它后续一切功能都无法验证。

**Independent Test**: 启动 CLI，输入"你好，请介绍一下你自己"，Agent 返回
一段文字回复，然后可以继续输入下一句话。完全不需要 Tool 调用即可验证。

**Acceptance Scenarios**:

1. **Given** 系统已配置好至少一个 LLM Provider（如 DeepSeek），**When** 用户在
   CLI 输入"你好"并回车，**Then** Agent 调用 LLM 并流式/同步返回一段文字回复，
   不报错。
2. **Given** 对话已进行 3 轮，**When** 用户输入第 4 条消息，**Then** Agent
   基于前 3 轮上下文给出连贯回复（不丢失上文）。
3. **Given** 用户正在对话中，**When** 用户输入 `/exit` 或按 Ctrl+C，**Then**
   对话正常结束，显示"再见"并退出。

---

### User Story 2 - Agent 自动调用 Tool 并返回结果 (Priority: P1)

用户提出的问题需要 Agent 调用 Tool（如读取文件、执行命令）才能回答。
Agent 识别 LLM 返回的 Tool Call，执行相应 Tool，将结果返回给 LLM 继续
推理，最终生成包含 Tool 执行结果信息的回复。

**Why this priority**: Tool 调用能力是 Agent 区别于普通 Chatbot 的核心。
与 US1 同为 P1 是因为没有 Tool 调用的 Agent 不能称为 Agent OS。

**Independent Test**: 在 `.oryxos/` 工作区放置一个测试文件，用户在 CLI 中
要求 Agent"读一下 test.txt 的内容"，Agent 调用 `read_file` Tool 读取文件，
并基于文件内容给出回复。

**Acceptance Scenarios**:

1. **Given** 对话进行中且用户请求需要调用 Tool（如"列出当前目录的文件"），
   **When** LLM 返回一个 Tool Call（如 `list_dir`），**Then** 系统执行该 Tool
   并将结果传回 LLM，LLM 基于结果生成最终回复。
2. **Given** LLM 在一次回复中返回了 2 个独立的 Tool Call，**When** 系统收到
   这 2 个 Tool Call，**Then** 按顺序执行它们，将所有结果一起传回 LLM。
3. **Given** Tool 执行失败（如文件不存在），**When** Tool 返回错误信息，
   **Then** 系统将错误信息传回 LLM，LLM 向用户解释发生了什么（而非静默失败）。

---

### User Story 3 - 会话持久化与恢复 (Priority: P2)

用户在对话过程中退出 CLI，之后重新启动并恢复之前的会话，Agent 记住之前的
对话内容，可以无缝继续对话。

**Why this priority**: 会话持久化是实际使用场景的必要能力——运维场景中
不可能一次完成所有工作。但它依赖于 US1/US2 的对话能力先跑通。

**Independent Test**: 进行 3 轮对话后退出，重新启动并指定恢复该 Session，
输入"继续我们刚才的话题"，Agent 能引用前 3 轮的内容。

**Acceptance Scenarios**:

1. **Given** 一个包含 5 轮对话的活跃 Session，**When** 用户退出 CLI，
   **Then** Session 状态、对话历史被持久化到存储中。
2. **Given** 之前有一个活跃 Session，**When** 用户启动 CLI 并指定恢复该
   Session，**Then** Agent 加载完整对话历史，可以基于上文继续对话。
3. **Given** 对话历史超过配置的最大轮数（如 20 轮），**When** 系统加载
   Session，**Then** 只保留最近的 N 轮，早期轮次被截断但不影响 Session 可用性。

---

### User Story 4 - 多 Profile 切换 (Priority: P3)

用户可以在启动 CLI 时指定不同的 Agent Profile（不同的人格、Provider、
模型参数），同一个底座支持运行不同配置的 Agent。

**Why this priority**: 多 Profile 是底座能力的体现，但单 Profile 已经可以
验证核心链路。放在 P3 确保核心先稳定。

**Independent Test**: 创建两个 Profile（如 "ops-agent" 和 "dev-agent"），
分别启动 CLI 对话，验证它们使用了不同的 system prompt 和模型配置。

**Acceptance Scenarios**:

1. **Given** `.oryxos/agents/` 下有两个 Agent 目录（各自的 `AGENT.md`），
   **When** 用户分别以 `--agent ops-agent` 和 `--agent dev-agent` 启动 CLI，
   **Then** 两个 Agent 展现不同的人设和知识领域。
2. **Given** 用户未指定 `--agent` 参数，**When** 启动 CLI，**Then** 使用
   默认 Agent Profile（如 `default`），不报错。

---

### Edge Cases

- 用户连续快速输入：CLI 同步逐条处理，无消息队列，不存在溢出问题。
- LLM 返回内容为空或格式异常时：自动重试 1 次（告知 LLM 上次返回为空），
  仍失败则向用户报错并说明原因。
- LLM 调用超时（默认 30 秒）时：直接向用户报错告知超时，不做自动重试，
  用户可手动重发消息。
- Tool Call 参数格式错误（与 Schema 不匹配）时：将错误信息传回 LLM 自修正，
  计为一次迭代（纳入 max_iterations），不直接向用户报错。
- ReAct Loop 达到 `max_iterations` 仍未产生最终回复时：终止循环，向用户返回
  已有信息和未完成原因（FR-010）。
- 用户输入为全空白字符时：忽略并重新显示输入提示，不发送给 LLM。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系统 MUST 支持从命令行启动交互式对话（`oryxos chat`），接收用户
  文本输入并打印 Agent 回复。
- **FR-002**: 系统 MUST 实现 ReAct（Reasoning + Acting）循环：用户消息 →
  LLM 推理 → [有 Tool Call → 执行 Tool → 回传结果 → LLM 继续推理] →
  最终回复用户。
- **FR-003**: ReAct Loop MUST 由项目自行实现，不得使用第三方框架的 Agent 自动
  执行能力（Constitution Principle I）。
- **FR-004**: 系统 MUST 支持多轮对话：每轮追加到对话历史，LLM 推理时可访问
  历史上下文。
- **FR-005**: 系统 MUST 在每次 LLM 调用前组装完整的 Prompt，包含：System Prompt
  （Agent 人设 + 引导指令）、对话历史（最近 N 轮）、可用 Tool 列表。
- **FR-006**: 系统 MUST 在收到 LLM 返回的 Tool Call 时，解析 Tool 名称和参数，
  通过 ToolRegistry 找到对应 Tool 并执行。
- **FR-006a**: 当 Tool Call 参数与 Schema 不匹配时，系统 MUST 将格式错误信息
  传回 LLM 让其自行修正，计为一次迭代（纳入 max_iterations 计数），而非直接
  向用户报错。
- **FR-006b**: 当 LLM 返回空内容（无 text 且无 tool call）时，系统 MUST 自动重试
  1 次（告知 LLM 重新生成），仍失败则向用户报错。
- **FR-006c**: LLM 调用超时（默认 30 秒可配置）时，系统 MUST 向用户报错告知超时，
  不做自动重试。
- **FR-007**: 系统 MUST 将每次 LLM 调用记录写入审计存储（`llm_calls`），包含
  Provider、模型、Token 用量、耗时（Constitution Principle V）。
- **FR-008**: 系统 MUST 将每次 Tool 调用记录写入审计存储（`tool_invocations`），
  包含 Tool 名称、输入参数、执行结果、成功/失败、耗时（Constitution Principle V）。
- **FR-009**: 系统 MUST 支持 Session 创建、持久化和恢复：退出时保存对话状态，
  重新启动时可恢复指定 Session。
- **FR-010**: 系统 MUST 在 ReAct Loop 达到最大迭代次数时终止循环，向用户返回
  已有信息并说明原因（而非无限循环）。
- **FR-011**: 系统 MUST 支持通过 `--agent` 参数指定 Agent Profile，不同 Profile
  使用独立的 system prompt、Provider、模型配置。
- **FR-012**: 系统 MUST 支持 `oryxos init` 命令初始化 `.oryxos/` 工作区目录结构。
- **FR-013**: 系统 MUST 在 CLI 中提供 `/exit` 命令正常退出对话。
- **FR-014**: 用户输入为全空白字符时，系统 MUST 忽略并重新显示输入提示，不发送给 LLM。

### Key Entities *(include if feature involves data)*

- **Session（会话）**: 代表一次或多次连续对话。属性包括：唯一标识、关联的 Agent
  Profile、对话历史（消息列表）、状态（active/archived）、创建时间、最后活跃时间。
- **Message（消息）**: 对话中的一条记录。包含：角色（user/assistant/tool）、内容、
  时间戳。Assistant 消息可能包含 Tool Call 信息。
- **Agent Profile**: 定义一个 Agent 的配置。包含：名称、描述、System Prompt、
  Provider 选择、模型名、温度参数、可用 Tool 列表、最大迭代次数、最大历史轮数。
- **Tool Call**: LLM 要求执行一个 Tool 的请求。包含：Tool 名称、调用参数（JSON）。
- **Tool Result**: Tool 执行的结果。包含：成功/失败标志、结果内容、错误信息（如有）。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 用户从输入消息到收到 Agent 最终回复（不含 Tool 调用）的端到端
  耗时不超过 5 秒（在典型网络条件下）。
- **SC-002**: 用户从输入消息到收到 Agent 最终回复（含 1 次 Tool 调用）的端到端
  耗时不超过 10 秒。
- **SC-003**: 多轮对话中，Agent 在第 5 轮时仍能正确引用第 1 轮的内容（上下文
  保持能力）。
- **SC-004**: Session 持久化后重新加载，对话历史完整恢复，丢失消息数为 0。
- **SC-005**: 单个 CLI 进程运行时内存占用不超过 512MB（不含 LLM Provider 侧）。
- **SC-006**: 100% 的 LLM 调用和 Tool 调用被记录到审计存储中（可审计性）。

## Assumptions

- LLM Provider（如 DeepSeek）的 API 端点是可访问的，网络延迟在正常范围（<200ms）。
- 用户已有 LLM Provider 的 API Key 并配置为环境变量。
- 核心阶段不涉及多用户并发——单个 CLI 进程只服务一个用户。
- 核心阶段不引入异步/响应式编程模型，全程同步阻塞 + Virtual Thread 处理 IO 等待。
- Tool 白名单和安全校验（SandboxChecker）的完整实现放在后续 Tool 模块 feature 中，
  本 feature 的 Tool 执行以"信任模式"运行（仅验证基本参数格式）。
- Bootstrap 文件（AGENTS.md、SOUL.md、USER.md）的加载逻辑放在后续 ContextLoader
  feature 中，本 feature 使用简化版 system prompt 注入。
