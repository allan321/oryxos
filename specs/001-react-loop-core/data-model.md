# Data Model: OryxOS 核心 ReAct Loop

**Feature**: 001-react-loop-core
**Date**: 2026-08-02

## Entity Relationship

```
┌──────────┐        ┌──────────────┐        ┌───────────────┐
│  Profile  │◄───────│   Session    │────────►│ llm_calls     │
│ (内存对象) │        │  (SQLite)    │        │  (SQLite)     │
└──────────┘        └──────┬───────┘        └───────────────┘
                           │
                           │ (session 内嵌)
                           ▼
                    ┌──────────────┐        ┌───────────────┐
                    │   Message[]  │────────►│ tool_invoc..  │
                    │   (JSON)     │        │  (SQLite)     │
                    └──────────────┘        └───────────────┘
```

## Entities

### Profile (内存 + YAML 文件)

从 `AGENT.md` frontmatter 派生，运行时加载。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | String | Y | 唯一标识，用于 `--agent <name>` |
| `description` | String | N | Agent 用途描述 |
| `agentName` | String | N | 展示名称（如"运维小欧"） |
| `systemPrompt` | String | Y | 注入 system prompt 的正文 |
| `providerName` | String | Y | Provider 映射 key（如 "deepseek"） |
| `model` | String | Y | 模型名（如 "deepseek-chat"） |
| `temperature` | Double | N | 温度参数（0.0-2.0，默认 0.7） |
| `apiKey` | String | Y | 从环境变量解析（`${VAR}`格式） |
| `toolNames` | List\<String\> | N | 启用的 Tool 名称列表 |
| `maxIterations` | int | N | ReAct Loop 最大迭代次数（默认 10） |
| `maxHistoryTurns` | int | N | 保留最近 N 轮对话历史（默认 20） |

**来源**: `AgentLoader.deriveProfile(agentDir)` 读 `AGENT.md` YAML frontmatter。

**加载示例**:
```yaml
# .oryxos/agents/ops-agent/AGENT.md
---
name: ops-agent
provider:
  name: deepseek
  model: deepseek-chat
  temperature: 0.7
  api_key: ${DEEPSEEK_API_KEY}
tools:
  - read_file
  - shell
settings:
  max_iterations: 10
  max_history_turns: 20
---
你是一个专业的运维助手...
```

### Session (SQLite: `sessions`)

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `sessionId` | VARCHAR(255) | PK | 唯一标识，格式 `{channel}-{userId}-{profileName}-{timestamp}` |
| `profileName` | VARCHAR(255) | NOT NULL | 关联 Profile.name |
| `channel` | VARCHAR(50) | NOT NULL | 渠道标识（"cli"） |
| `userId` | VARCHAR(255) | NOT NULL | 用户标识（CLI 场景默认 "cli-user"） |
| `messagesJson` | TEXT | NOT NULL | JSON 序列化的 `List<Message>` |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active / archived |
| `createdAt` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `lastActiveAt` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 最后活跃时间 |
| `archivedAt` | TIMESTAMP | NULLABLE | 归档时间 |

**状态转移**:
```
active ──[archive]──► archived
```

**JPA Entity**: `com.oryxos.core.model.SessionEntity`
**JPA Repository**: `com.oryxos.storage.SessionRepository`

### Message (内嵌于 Session.messagesJson)

不在独立 SQLite 表，作为 JSON 数组存入 `sessions.messages_json`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `role` | String | `user` / `assistant` / `tool` |
| `content` | String | 消息文本内容 |
| `toolCalls` | List\<ToolCallData\> (nullable) | assistant 消息可能包含的 Tool Call 信息 |
| `toolCallId` | String (nullable) | tool 消息关联的 Tool Call ID |
| `timestamp` | Long | epoch millis |

**Java Record**: `com.oryxos.core.model.Message`

### ToolCallData (内嵌于 Message)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | LLM 返回的 Tool Call ID |
| `name` | String | Tool 名称（如 "read_file"） |
| `arguments` | String | JSON 参数字符串 |

### ToolResult (内存值对象)

| 字段 | 类型 | 说明 |
|------|------|------|
| `toolCallId` | String | 关联的 Tool Call ID |
| `success` | boolean | 执行是否成功 |
| `content` | String | 结果内容 |
| `errorMessage` | String (nullable) | 错误信息 |

**Java Record**: `com.oryxos.core.model.ToolResult`

### ToolInvocation (SQLite: `tool_invocations`)

审计表，Day One 写入。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTOINCREMENT | 主键 |
| `sessionId` | VARCHAR(255) | NOT NULL | 关联 Session |
| `toolName` | VARCHAR(100) | NOT NULL | Tool 名称 |
| `inputJson` | TEXT | NOT NULL | JSON 序列化的调用参数 |
| `resultJson` | TEXT | NULLABLE | JSON 序列化的执行结果 |
| `success` | BOOLEAN | NOT NULL | 是否成功 |
| `errorMessage` | TEXT | NULLABLE | 错误信息 |
| `durationMs` | BIGINT | NOT NULL | 执行耗时（毫秒） |
| `createdAt` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 调用时间 |

### LlmCall (SQLite: `llm_calls`)

审计表，Day One 写入。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTOINCREMENT | 主键 |
| `sessionId` | VARCHAR(255) | NOT NULL | 关联 Session |
| `provider` | VARCHAR(100) | NOT NULL | Provider 名称 |
| `model` | VARCHAR(100) | NOT NULL | 模型名 |
| `promptTokens` | INT | NULLABLE | 输入 token 数 |
| `completionTokens` | INT | NULLABLE | 输出 token 数 |
| `totalTokens` | INT | NULLABLE | 总 token 数 |
| `durationMs` | BIGINT | NOT NULL | 调用耗时（毫秒） |
| `createdAt` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 调用时间 |

## OryxTool Interface (核心抽象)

```java
package com.oryxos.core.tool;

interface OryxTool {
    String getName();
    String getDescription();
    String getParametersJsonSchema();  // JSON Schema 字符串
    ToolResult execute(JsonNode input);
}
```

`ToolRegistry` 的注册入口就是这个接口——无论是内置 Tool、MCP Tool 还是 Plugin
Tool，最终都适配为 `OryxTool`。
