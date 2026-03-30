# 代码结构：AgentScope

## 顶层目录一览

| 路径 | 类型 | 用途说明 |
|------|------|----------|
| `src/agentscope/` | 源码 | 框架核心源代码 |
| `tests/` | 测试 | 单元测试和集成测试 |
| `examples/` | 示例 | 各类使用示例 |
| `docs/` | 文档 | 项目文档 |
| `assets/` | 资源 | 图片等静态资源 |
| `pyproject.toml` | 配置 | Python项目配置 |
| `README.md` | 文档 | 项目说明 |
| `CONTRIBUTING.md` | 文档 | 贡献指南 |

## 核心源码目录详解

### `src/agentscope/`

框架根模块，定义了全局配置和初始化函数。

| 文件 | 用途 |
|------|------|
| `__init__.py` | 模块入口，导出主要子模块和`init()`函数 |
| `_version.py` | 版本号定义 |
| `_logging.py` | 日志配置 |
| `_run_config.py` | 运行时配置类 |

### `src/agentscope/agent/`

智能体实现，是框架的核心。

> 职责：定义智能体基类和各类智能体实现

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出所有Agent类 |
| `_agent_base.py` | `AgentBase` - 所有智能体的抽象基类，提供Hooks、消息广播、流式打印 |
| `_react_agent_base.py` | `ReActAgentBase` - ReAct模式智能体基类 |
| `_react_agent.py` | `ReActAgent` - 完整的ReAct实现，支持工具调用、记忆、RAG |
| `_user_agent.py` | `UserAgent` - 用户输入代理，处理人类输入 |
| `_user_input.py` | 用户输入方式定义（终端、Studio界面） |
| `_a2a_agent.py` | `A2AAgent` - A2A协议智能体 |
| `_realtime_agent.py` | `RealtimeAgent` - 实时语音智能体 |
| `_agent_meta.py` | Agent元类定义 |
| `_utils.py` | Agent模块工具函数 |

### `src/agentscope/model/`

模型API封装。

> 职责：封装各厂商大模型API调用

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出所有Model类 |
| `_model_base.py` | `ChatModelBase` - 模型基类 |
| `_model_response.py` | `ChatResponse` - 模型响应定义 |
| `_model_usage.py` | 模型用量统计 |
| `_dashscope_model.py` | 通义千问模型封装 |
| `_openai_model.py` | OpenAI模型封装 |
| `_anthropic_model.py` | Anthropic Claude模型封装 |
| `_gemini_model.py` | Google Gemini模型封装 |
| `_ollama_model.py` | Ollama本地模型封装 |
| `_trinity_model.py` | Trinity微调模型封装 |

### `src/agentscope/message/`

消息系统定义。

> 职责：定义统一的消息格式和内容块类型

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Msg和各类ContentBlock |
| `_message_base.py` | `Msg` - 消息基类，支持多模态内容 |
| `_message_block.py` | 各种ContentBlock定义（TextBlock, ImageBlock, AudioBlock, ToolUseBlock, ToolResultBlock等） |

### `src/agentscope/memory/`

记忆管理模块。

> 职责：管理智能体的短期和长期记忆

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出所有Memory类 |
| `_working_memory/_base.py` | `MemoryBase` - 工作记忆基类 |
| `_working_memory/_in_memory_memory.py` | `InMemoryMemory` - 内存存储 |
| `_working_memory/_redis_memory.py` | `RedisMemory` - Redis存储 |
| `_working_memory/_sqlalchemy_memory.py` | `AsyncSQLAlchemyMemory` - 数据库存储 |
| `_long_term_memory/_long_term_memory_base.py` | `LongTermMemoryBase` - 长期记忆基类 |
| `_long_term_memory/mem0/` | Mem0长期记忆实现 |
| `_long_term_memory/reme/` | ReMe长期记忆实现 |

### `src/agentscope/tool/`

工具集管理。

> 职责：工具函数注册、管理和执行

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Toolkit和内置工具函数 |
| `_toolkit.py` | `Toolkit` - 工具集核心类 |
| `_response.py` | `ToolResponse` - 工具响应定义 |
| `_types.py` | 工具相关类型定义 |
| `_async_wrapper.py` | 异步生成器包装器 |
| `_coding/_python.py` | 执行Python代码工具 |
| `_coding/_shell.py` | 执行Shell命令工具 |
| `_text_file/` | 文本文件查看/写入工具 |
| `_multi_modality/` | 多模态工具 |

### `src/agentscope/formatter/`

消息格式化器。

> 职责：将Msg格式化为不同模型API所需格式

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出所有Formatter |
| `_formatter_base.py` | `FormatterBase` - 格式化器基类 |
| `_dashscope_formatter.py` | DashScope API格式 |
| `_openai_formatter.py` | OpenAI API格式 |
| `_anthropic_formatter.py` | Anthropic API格式 |
| `_gemini_formatter.py` | Gemini API格式 |
| `_ollama_formatter.py` | Ollama API格式 |

### `src/agentscope/pipeline/`

工作流编排。

> 职责：提供多智能体工作流的语法糖

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Pipeline相关类 |
| `_msghub.py` | `MsgHub` - 消息中心，管理多智能体消息广播 |
| `_class.py` | `SequentialPipeline`, `FanoutPipeline` - 类式pipeline |
| `_functional.py` | `sequential_pipeline`, `fanout_pipeline` - 函数式pipeline |
| `_chat_room.py` | `ChatRoom` - 聊天室模式 |

### `src/agentscope/mcp/`

MCP协议支持。

> 职责：对接MCP服务器，将MCP工具集成到Toolkit

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出MCP相关类 |
| `_client_base.py` | `MCPClientBase` - MCP客户端基类 |
| `_mcp_function.py` | `MCPToolFunction` - MCP工具函数包装 |
| `_stateful_client_base.py` | `StatefulClientBase` - 有状态客户端基类 |
| `_stdio_stateful_client.py` | `StdIOStatefulClient` - stdio传输 |
| `_http_stateless_client.py` | `HttpStatelessClient` - HTTP无状态 |
| `_http_stateful_client.py` | `HttpStatefulClient` - HTTP有状态 |

### `src/agentscope/a2a/`

A2A协议支持。

> 职责：支持A2A（Agent-to-Agent）协议

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出A2A相关类 |
| `_base.py` | `AgentCardResolverBase` - Agent卡片解析器基类 |
| `_file_resolver.py` | `FileAgentCardResolver` - 文件解析 |
| `_well_known_resolver.py` | `WellKnownAgentCardResolver` - Well-Known解析 |
| `_nacos_resolver.py` | `NacosAgentCardResolver` - Nacos服务发现 |

### `src/agentscope/rag/`

RAG（检索增强生成）。

> 职责：知识库管理和文档检索

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出RAG相关类 |
| `_knowledge_base.py` | `KnowledgeBase` - 知识库 |
| `_document.py` | `Document` - 文档定义 |
| `_simple_knowledge.py` | 简单知识库实现 |
| `_reader/` | 各类文档读取器（PDF、Word、Excel等） |
| `_store/` | 向量存储后端（Qdrant、Milvus等） |

### `src/agentscope/realtime/`

实时语音。

> 职责：实时语音交互模型支持

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Realtime相关类 |
| `_base.py` | `RealtimeBase` - 实时模型基类 |
| `_dashscope_realtime_model.py` | DashScope实时语音模型 |
| `_openai_realtime_model.py` | OpenAI实时语音模型 |
| `_gemini_realtime_model.py` | Gemini实时语音模型 |
| `_events/` | 实时事件定义 |

### `src/agentscope/tuner/`

模型微调。

> 职责：支持Agent应用的模型微调

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Tuner相关类 |
| `_tune.py` | 微调主流程 |
| `_config.py` | 微调配置 |
| `_dataset.py` | 数据集处理 |
| `_judge.py` | 评判器 |
| `_model.py` | 微调模型相关 |
| `_workflow.py` | 微调工作流 |

### `src/agentscope/tracing/`

可观测性。

> 职责：OpenTelemetry追踪支持

| 文件 | 用途 |
|------|------|
| `__init__.py` | 导出Tracing相关类 |
| `_setup.py` | 追踪初始化 |
| `_trace.py` | 追踪装饰器 |
| `_attributes.py` | 追踪属性定义 |
| `_converter.py` | 数据转换 |
| `_extractor.py` | 数据提取 |
| `_utils.py` | 工具函数 |

### 其他模块

| 目录 | 用途 |
|------|------|
| `embedding/` | 文本嵌入模型封装 |
| `token/` | Token计数器 |
| `tts/` | 文本转语音 |
| `plan/` | 计划管理 |
| `session/` | 会话持久化 |
| `evaluate/` | 评估框架 |
| `module/` | 状态模块基类 |
| `types/` | 类型定义 |
| `exception/` | 异常定义 |
| `hooks/` | Studio Hooks |
| `_utils/` | 内部工具函数 |

## 关键文件说明

### `src/agentscope/agent/_agent_base.py`
- **作用**：所有智能体的抽象基类
- **主要类/函数**：`AgentBase`, `reply()`, `observe()`, `print()`, `interrupt()`
- **与其他文件的关系**：被所有具体Agent实现继承，依赖`message`模块

### `src/agentscope/agent/_react_agent.py`
- **作用**：ReAct智能体的完整实现
- **主要类/函数**：`ReActAgent`, `reply()`, `_reasoning()`, `_acting()`
- **与其他文件的关系**：继承`ReActAgentBase`，使用`model`, `memory`, `tool`, `formatter`模块

### `src/agentscope/tool/_toolkit.py`
- **作用**：工具集管理核心
- **主要类/函数**：`Toolkit`, `register_tool_function()`, `call_tool_function()`
- **与其他文件的关系**：被`ReActAgent`使用，支持`mcp`模块的工具

### `src/agentscope/pipeline/_msghub.py`
- **作用**：多智能体消息广播
- **主要类/函数**：`MsgHub`, `broadcast()`, `add()`, `delete()`
- **与其他文件的关系**：管理多个`AgentBase`实例之间的消息流转

### `src/agentscope/message/_message_base.py`
- **作用**：消息系统核心
- **主要类/函数**：`Msg`, `get_content_blocks()`, `get_text_content()`
- **与其他文件的关系**：被几乎所有模块依赖

## 入口点

- **程序主入口**：由用户代码调用`agentscope.init()`初始化后使用
- **库的公开API入口**：`src/agentscope/__init__.py`
- **CLI入口**：框架主要作为库使用，无内置CLI
- **Web入口**：通过`examples/`中的各类服务器示例

## 设计模式体现

| 文件 | 设计模式 |
|------|----------|
| `_agent_base.py` | 模板方法模式、观察者模式 |
| `_toolkit.py` | 策略模式、责任链模式、工厂模式 |
| `_msghub.py` | 观察者模式、上下文管理器模式 |
| `formatter/` | 策略模式 |
| `model/` | 策略模式 |
