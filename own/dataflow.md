# 关键数据流：AgentScope

## 数据流总览

AgentScope的核心数据流包括：

1. **ReAct Agent回复流程**：用户输入 → 检索 → 推理 → 工具执行 → 输出
2. **多智能体消息广播**：智能体回复 → MsgHub广播 → 其他智能体观察
3. **工具调用流程**：模型生成tool_use → Toolkit执行 → 返回tool_result
4. **记忆管理流程**：消息存储 → 标记管理 → 检索 → 压缩
5. **流式输出流程**：模型流式生成 → 实时打印 → TTS合成

---

## 数据流 1：ReAct Agent回复流程

### 触发方式
用户调用 `await agent(msg)`，其中`msg`是用户输入的`Msg`对象。

### 流转路径

```text
[用户输入] → ReActAgent.reply() → _retrieve_from_long_term_memory() → 
_retrieve_from_knowledge() → [循环开始] → _compress_memory_if_needed() → 
_reasoning() → model() → [模型响应] → _acting() → toolkit.call_tool_function() → 
[工具执行] → [循环结束条件] → _summarizing() → [返回Msg]
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `ReActAgent.reply()` | `Msg \| list[Msg]` | 接收用户消息，开始回复流程 | - |
| 2 | `memory.add()` | 用户消息 | 将消息存入短期记忆 | - |
| 3 | `_retrieve_from_long_term_memory()` | 用户消息 | 从长期记忆检索相关信息 | 检索到的信息Msg |
| 4 | `_retrieve_from_knowledge()` | 用户消息 | 从知识库检索文档 | 检索到的文档Msg |
| 5 | `_compress_memory_if_needed()` | - | 检查并压缩过长的记忆 | - |
| 6 | `_reasoning()` | tool_choice | 格式化消息 → 调用模型 | 模型响应Msg |
| 7 | `_acting()` | ToolUseBlock列表 | 并行/串行执行工具 | ToolResultBlock列表 |
| 8 | `toolkit.call_tool_function()` | ToolUseBlock | 查找并执行工具函数 | ToolResponse流 |
| 9 | `_summarizing()` | - | 当达到最大迭代次数时生成总结 | 总结Msg |

### 数据格式变化

```
用户输入（Msg）
  ↓
系统提示 + 历史消息 + 检索信息（list[dict] - 格式化后）
  ↓
模型响应（ChatResponse）→ 转换为 Msg
  ↓
ToolUseBlock（如果有工具调用）→ 工具执行 → ToolResultBlock
  ↓
最终回复（Msg）
```

### 错误处理
- **模型调用失败**：抛出异常，由调用方处理
- **工具执行失败**：返回包含错误信息的ToolResultBlock，记录在记忆中
- **中断处理**：`asyncio.CancelledError`被捕获，生成中断提示消息

---

## 数据流 2：多智能体消息广播

### 触发方式
在`MsgHub`上下文中，任何参与者调用`reply()`生成回复。

### 流转路径

```text
[Agent调用reply()] → AgentBase.__call__() → _broadcast_to_subscribers() → 
[MsgHub订阅者] → subscriber.observe() → [各Agent的memory.add()]
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `AgentBase.__call__()` | 任意参数 | 包装reply调用，处理Hooks和中断 | Msg |
| 2 | `_broadcast_to_subscribers()` | 回复Msg | 移除thinking块，广播给所有订阅者 | - |
| 3 | `_strip_thinking_blocks()` | Msg | 移除内部的thinking内容块 | 过滤后的Msg |
| 4 | `subscriber.observe()` | 广播Msg | 接收并处理消息（默认存入记忆） | - |

### 数据格式变化

```
Agent回复Msg（包含thinking块）
  ↓
过滤后的Msg（移除thinking块）
  ↓
广播给所有订阅Agent
  ↓
各Agent存入自己的memory
```

---

## 数据流 3：工具调用流程

### 触发方式
模型在推理过程中生成`tool_use`类型的ContentBlock。

### 流转路径

```text
[模型生成tool_use] → ReActAgent._acting() → toolkit.call_tool_function() → 
_execute_tool() → [工具函数执行] → ToolResponse生成 → _print()显示结果
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `ReActAgent._acting()` | ToolUseBlock | 准备执行环境 | - |
| 2 | `toolkit.call_tool_function()` | ToolUseBlock | 查找工具，应用中间件 | AsyncGenerator[ToolResponse] |
| 3 | `_execute_tool()` | 工具参数 | 执行实际工具函数 | 原始结果 |
| 4 | `_async_generator_wrapper` | 原始结果 | 包装为ToolResponse流 | ToolResponse |
| 5 | `memory.add()` | ToolResultBlock | 记录工具结果到记忆 | - |

### 数据格式变化

```
ToolUseBlock: {"type": "tool_use", "id": "...", "name": "func", "input": {...}}
  ↓
工具函数调用参数（dict）
  ↓
工具函数执行结果（任意类型）
  ↓
ToolResponse: {content: list[ContentBlock], metadata: dict}
  ↓
ToolResultBlock: {"type": "tool_result", "id": "...", "name": "...", "output": "..."}
```

---

## 数据流 4：记忆管理流程

### 触发方式
Agent回复过程中调用`memory.add()`存入消息。

### 流转路径

```text
[消息产生] → memory.add() → [存储后端] → memory.get_memory() → 
[按标记筛选] → [返回消息列表]
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `MemoryBase.add()` | Msg + marks | 存储消息，关联标记 | - |
| 2 | `InMemoryMemory` | - | 存入内存列表 | - |
| 3 | `RedisMemory` | - | 序列化后存入Redis | - |
| 4 | `memory.get_memory()` | mark/exclude_mark | 根据标记筛选 | list[Msg] |
| 5 | `delete_by_mark()` | mark | 删除带指定标记的消息 | int（删除数量）|

### 数据格式变化

```
Msg对象
  ↓
序列化为JSON（RedisMemory）或保持对象（InMemoryMemory）
  ↓
存储到对应后端
  ↓
检索时反序列化/直接返回
  ↓
list[Msg]
```

---

## 数据流 5：流式输出流程

### 触发方式
模型配置`stream=True`时，模型调用返回异步生成器。

### 流转路径

```text
[模型流式生成] → _reasoning()循环 → AgentBase.print() → 
[流式打印] → [TTS处理] → [音频播放]
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `model()` | prompt | 流式调用模型API | AsyncGenerator[ChatResponse] |
| 2 | `_reasoning()循环` | ChatResponse块 | 逐块更新Msg内容 | 部分Msg |
| 3 | `AgentBase.print()` | Msg块 | 流式打印到控制台 | - |
| 4 | `tts_model.push()` | Msg块 | 推送到TTS模型（如有） | 音频数据 |
| 5 | `_process_audio_block()` | 音频数据 | 播放音频 | - |

### 数据格式变化

```
ChatResponse块（部分生成的内容）
  ↓
合并到当前Msg.content
  ↓
提取文本内容打印
  ↓
如有音频块，解码base64并播放
```

---

## 跨数据流的共享组件

### 1. Msg（消息对象）
所有数据流都以`Msg`对象作为核心数据传输格式。

### 2. Memory（记忆）
- 被ReAct Agent用于存储对话历史
- 被广播机制用于让其他Agent获取消息

### 3. Toolkit（工具集）
- 被ReAct Agent用于执行工具调用
- 支持MCP工具、本地函数、Agent Skills

### 4. Formatter（格式化器）
- 被Model用于格式化输入
- 被ReAct Agent用于准备prompt

### 5. MsgHub
- 管理多智能体之间的消息广播
- 维护订阅者关系
