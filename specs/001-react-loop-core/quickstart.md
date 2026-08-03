# Quickstart: OryxOS 核心 ReAct Loop

**Feature**: 001-react-loop-core
**Date**: 2026-08-02

## Prerequisites

- JDK 21+
- Maven 3.9+
- LLM Provider API Key（如 DeepSeek）设为环境变量 `DEEPSEEK_API_KEY`
- 项目已编译: `mvn clean install -DskipTests` (from repo root)

## Validation Scenarios

### 场景 1: 初始化工作区

**验证目标**: `oryxos init` 正确创建 `.oryxos/` 目录结构。

```bash
cd /tmp/oryxos-test
java -jar oryxos-cli/target/oryxos-cli-*.jar init
```

**Expected**:
```
✓ .oryxos/ 工作区已初始化
  .oryxos/agents/
  .oryxos/memory/MEMORY.md
  .oryxos/sessions/
  .oryxos/logs/
  .oryxos/AGENTS.md
  .oryxos/SOUL.md
  .oryxos/USER.md
  .oryxos/oryxos.db
```

---

### 场景 2: 创建 Agent Profile

**验证目标**: 从 `AGENT.md` 加载 Profile。

手工创建 `.oryxos/agents/default/AGENT.md`:
```markdown
---
name: default
description: 默认助手
identity:
  agent_name: OryxOS
  prompt: 你是一个友好的 AI 助手。
provider:
  name: deepseek
  model: deepseek-chat
  temperature: 0.7
  api_key: ${DEEPSEEK_API_KEY}
tools: []
channels:
  - name: cli
settings:
  max_iterations: 5
  max_history_turns: 10
---

你是一个友好的 AI 助手。用中文回答。
```

```bash
java -jar oryxos-cli/target/oryxos-cli-*.jar status
```

**Expected**:
```
Profile 列表:
  - default (deepseek/deepseek-chat)

Provider 状态:
  - deepseek: ✓ 已连接
```

---

### 场景 3: 基础对话（无 Tool 调用）

**验证目标**: User Story 1 — CLI 交互式多轮对话。

```bash
java -jar oryxos-cli/target/oryxos-cli-*.jar chat
```

```
OryxOS> 你好，请用一句话介绍你自己
我是 OryxOS，一个运行在企业基础设施上的 AI Agent 操作系统。

OryxOS> 1+1等于几
1+1 等于 2。

OryxOS> /exit
再见!
```

**Expected**:
- Agent 回复不报错
- 第 3 轮对话能引用前文（如用"你"指代 Agent）
- `/exit` 正常退出

---

### 场景 4: Agent 调用 Tool 并基于结果回复

**验证目标**: User Story 2 — ReAct Loop 的 Tool Call 执行链路。

**前置**: 在 `.oryxos/` 目录下创建 `test.txt`:
```
Hello from OryxOS!
```

更新 `AGENT.md` 的 `tools: [read_file]`。

```bash
java -jar oryxos-cli/target/oryxos-cli-*.jar chat
```

```
OryxOS> 读一下 test.txt 的内容
（Agent 调用 read_file → 获取文件内容 → 返回解读）
test.txt 的内容是："Hello from OryxOS!"
```

**Expected**:
- Tool 被正确解析和执行
- Agent 回复引用了 Tool 执行结果
- 审计表 `tool_invocations` 和 `llm_calls` 中有对应记录

**验证审计**:
```bash
sqlite3 .oryxos/oryxos.db "SELECT tool_name, success, duration_ms FROM tool_invocations;"
# 应输出至少 1 行 read_file 调用记录

sqlite3 .oryxos/oryxos.db "SELECT provider, model, total_tokens FROM llm_calls;"
# 应输出至少 2 行（初始推理 + tool 结果后的二次推理）
```

---

### 场景 5: Session 持久化与恢复

**验证目标**: User Story 3 — Session 跨进程恢复。

```bash
# 第一轮对话
java -jar oryxos-cli/target/oryxos-cli-*.jar chat
Oryxos> 我叫张三，我喜欢蓝色
好的张三，我记住了你喜欢蓝色。
Oryxos> /exit
```

```bash
# 查看 Session ID
java -jar oryxos-cli/target/oryxos-cli-*.jar session list
# [active] sess-cli-default-xxxxx (3 messages, last active: ...)
```

```bash
# 恢复 Session
java -jar oryxos-cli/target/oryxos-cli-*.jar chat --session sess-cli-default-xxxxx
Oryxos> 我叫什么名字？
你叫张三。
Oryxos> 我喜欢什么颜色？
你提到喜欢蓝色。
```

**Expected**:
- 恢复后 Agent 记住前一轮对话内容
- 对话历史完整（丢失消息数 = 0）

---

### 场景 6: 多 Agent Profile 切换

**验证目标**: User Story 4 — 不同 Profile 不同行为。

创建第二个 Agent `.oryxos/agents/dev-agent/AGENT.md`:
```markdown
---
name: dev-agent
description: 开发助手
identity:
  agent_name: DevBot
  prompt: 你是一个编程助手，所有问题从代码角度回答。
provider:
  name: deepseek
  model: deepseek-chat
  api_key: ${DEEPSEEK_API_KEY}
tools: []
settings:
  max_iterations: 5
---

你是一个编程助手。所有问题从代码角度回答。你是 DevBot，不是 OryxOS。
```

```bash
# 用默认 Agent
java -jar oryxos-cli/target/oryxos-cli-*.jar chat --agent default
OryxOS> 你是谁
我是 OryxOS，一个友好的 AI 助手。

# 用开发 Agent
java -jar oryxos-cli/target/oryxos-cli-*.jar chat --agent dev-agent
OryxOS> 你是谁
我是 DevBot，一个编程助手。
```

**Expected**: 两个 Agent 使用不同 system prompt，人设不同。

---

### 场景 7: 错误处理（边界场景）

| 测试 | 命令/操作 | 预期结果 |
|------|---------|---------|
| 空白输入 | 在对话中直接回车 | 不发送给 LLM，重新显示提示符 |
| LLM 超时 | 设置极短超时配置模拟 | 报告"LLM 调用超时"，不自动重试 |
| max_iterations | 给 Agent 一个不可能单步完成的任务 + max_iterations=2 | 2 次迭代后终止，显示"已达到最大迭代次数" |
| 不存在的 Agent | `chat --agent nonexist` | 报错"Profile 'nonexist' 不存在" |

## Environment Variables

| 变量 | 说明 | 必填 |
|------|------|------|
| `DEEPSEEK_API_KEY` | DeepSeek API Key | Y (使用 DeepSeek 时) |
| `ORYXOS_HOME` | `.oryxos/` 工作区路径（默认当前目录） | N |

## Notes

- 首次运行前确保 `DEEPSEEK_API_KEY` 环境变量已设置
- 测试完成后可 `rm -rf /tmp/oryxos-test` 清理
- Tool 信任模式：本阶段不校验路径/命令白名单
