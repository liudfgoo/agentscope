# 最小使用示例：AgentScope

## 快速开始

### 环境准备

```bash
# 安装AgentScope
pip install agentscope

# 或从源码安装
git clone https://github.com/agentscope-ai/agentscope.git
cd agentscope
pip install -e .
```

### 设置API密钥

```bash
# 根据使用的模型设置对应的环境变量
export DASHSCOPE_API_KEY="your-dashscope-api-key"
export OPENAI_API_KEY="your-openai-api-key"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

---

## 示例 1：基本ReAct Agent

**功能说明**：创建一个具备工具使用能力的ReAct智能体，可以执行Python代码和Shell命令。

**前置条件**：需要设置`DASHSCOPE_API_KEY`环境变量。

```python
import asyncio
import os
from agentscope.agent import ReActAgent, UserAgent
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.model import DashScopeChatModel
from agentscope.tool import Toolkit, execute_python_code, execute_shell_command


async def main():
    # 创建工具集
    toolkit = Toolkit()
    toolkit.register_tool_function(execute_python_code)
    toolkit.register_tool_function(execute_shell_command)

    # 创建ReAct Agent
    agent = ReActAgent(
        name="Friday",
        sys_prompt="You are a helpful assistant named Friday.",
        model=DashScopeChatModel(
            api_key=os.environ.get("DASHSCOPE_API_KEY"),
            model_name="qwen-max",
            stream=True,
        ),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
        memory=InMemoryMemory(),
    )

    # 创建用户代理
    user = UserAgent("User")

    # 对话循环
    msg = None
    while True:
        msg = await user(msg)
        if msg.get_text_content() == "exit":
            break
        msg = await agent(msg)


asyncio.run(main())
```

**预期输出**：
```
User: 你好
Friday: 你好！我是Friday，有什么可以帮助你的吗？
User: 计算1+1
Friday: 我将使用Python代码来计算这个表达式。
Friday: [执行工具: execute_python_code]
Friday: 1+1 = 2
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `name` | str | Agent名称 | 必填 |
| `sys_prompt` | str | 系统提示词 | 必填 |
| `model` | ChatModelBase | 模型实例 | 必填 |
| `formatter` | FormatterBase | 格式化器 | 必填 |
| `toolkit` | Toolkit | 工具集 | None |
| `memory` | MemoryBase | 记忆模块 | InMemoryMemory() |
| `max_iters` | int | 最大推理轮数 | 10 |

---

## 示例 2：多智能体对话

**功能说明**：使用MsgHub管理多个智能体之间的对话，实现消息自动广播。

**前置条件**：需要设置`DASHSCOPE_API_KEY`环境变量。

```python
import asyncio
import os
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeMultiAgentFormatter
from agentscope.message import Msg
from agentscope.model import DashScopeChatModel
from agentscope.pipeline import MsgHub, sequential_pipeline


async def main():
    # 创建多个参与者智能体
    alice = ReActAgent(
        name="Alice",
        sys_prompt="You're Alice, a friendly teacher.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        ),
        formatter=DashScopeMultiAgentFormatter(),
    )

    bob = ReActAgent(
        name="Bob",
        sys_prompt="You're Bob, a curious student.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        ),
        formatter=DashScopeMultiAgentFormatter(),
    )

    # 在MsgHub中管理对话
    async with MsgHub(
        participants=[alice, bob],
        announcement=Msg(
            "system",
            "Introduce yourselves to each other.",
            "system",
        ),
    ) as hub:
        # 顺序执行对话
        await sequential_pipeline([alice, bob, alice, bob])
        
        # 动态添加参与者
        # hub.add(charlie)
        # hub.delete(bob)


asyncio.run(main())
```

**预期输出**：
```
[system]: Introduce yourselves to each other.
Alice: Hello Bob! I'm Alice, a teacher. Nice to meet you!
Bob: Hi Alice! I'm Bob, a student. I'm excited to learn from you!
Alice: That's great, Bob! What subjects are you most interested in?
Bob: I really enjoy math and science!
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `participants` | list[AgentBase] | 参与者列表 | 必填 |
| `announcement` | Msg \| list[Msg] | 开场消息 | None |
| `enable_auto_broadcast` | bool | 是否自动广播 | True |

---

## 示例 3：MCP工具集成

**功能说明**：使用MCP客户端接入外部工具服务。

**前置条件**：需要MCP服务URL和API密钥。

```python
import asyncio
import os
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.mcp import HttpStatelessClient
from agentscope.model import DashScopeChatModel
from agentscope.tool import Toolkit


async def main():
    # 初始化MCP客户端
    mcp_client = HttpStatelessClient(
        name="gaode_mcp",
        transport="streamable_http",
        url=f"https://mcp.amap.com/mcp?key={os.environ['GAODE_API_KEY']}",
    )

    # 创建工具集并注册MCP工具
    toolkit = Toolkit()
    
    # 获取MCP工具列表
    mcp_tools = await mcp_client.list_tool_functions()
    for tool in mcp_tools:
        toolkit.register_tool_function(tool)

    # 创建Agent
    agent = ReActAgent(
        name="MapAssistant",
        sys_prompt="You can use map tools to help users.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
        ),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
        memory=InMemoryMemory(),
    )

    # 使用Agent
    from agentscope.message import Msg
    msg = Msg("user", "What's the weather in Beijing?", "user")
    response = await agent(msg)
    print(response.get_text_content())


asyncio.run(main())
```

**预期输出**：
```
MapAssistant: I'll check the weather in Beijing for you.
MapAssistant: [执行工具: maps_weather]
MapAssistant: The current weather in Beijing is sunny with a temperature of 25°C.
```

---

## 示例 4：带RAG的Agent

**功能说明**：创建具备知识库检索能力的智能体。

**前置条件**：需要准备文档和向量数据库。

```python
import asyncio
import os
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.model import DashScopeChatModel
from agentscope.rag import KnowledgeBase, Document


async def main():
    # 创建知识库
    docs = [
        Document(content="AgentScope is a multi-agent framework.", title="About"),
        Document(content="It supports ReAct agents and tool use.", title="Features"),
    ]
    
    knowledge = KnowledgeBase(
        documents=docs,
        embedding_model="dashscope",
    )

    # 创建带RAG的Agent
    agent = ReActAgent(
        name="KnowledgeBot",
        sys_prompt="You are a helpful assistant with access to a knowledge base.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
        ),
        formatter=DashScopeChatFormatter(),
        memory=InMemoryMemory(),
        knowledge=knowledge,  # 传入知识库
    )

    # 问答
    from agentscope.message import Msg
    msg = Msg("user", "What is AgentScope?", "user")
    response = await agent(msg)
    print(response.get_text_content())


asyncio.run(main())
```

**预期输出**：
```
KnowledgeBot: Based on the knowledge base, AgentScope is a multi-agent framework 
that supports ReAct agents and tool use.
```

---

## 示例 5：结构化输出

**功能说明**：让Agent生成符合特定结构的数据（如JSON）。

```python
import asyncio
import os
from pydantic import BaseModel, Field
from agentscope.agent import ReActAgent
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.model import DashScopeChatModel
from agentscope.message import Msg


# 定义输出结构
class WeatherInfo(BaseModel):
    city: str = Field(description="City name")
    temperature: float = Field(description="Temperature in Celsius")
    condition: str = Field(description="Weather condition")


async def main():
    agent = ReActAgent(
        name="WeatherBot",
        sys_prompt="Extract weather information from user input.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
        ),
        formatter=DashScopeChatFormatter(),
        memory=InMemoryMemory(),
    )

    msg = Msg("user", "It's sunny and 25 degrees in Beijing today.", "user")
    
    # 请求结构化输出
    response = await agent(msg, structured_model=WeatherInfo)
    
    # 结构化数据在metadata中
    print(response.metadata)


asyncio.run(main())
```

**预期输出**：
```python
{
    'city': 'Beijing',
    'temperature': 25.0,
    'condition': 'sunny'
}
```

---

## 常见使用模式

### 1. 自定义工具函数

```python
from agentscope.tool import ToolResponse
from agentscope.message import TextBlock

async def my_custom_tool(query: str) -> ToolResponse:
    """A custom tool function.
    
    Args:
        query: The search query
    """
    result = f"Result for: {query}"
    return ToolResponse(
        content=[TextBlock(type="text", text=result)],
        is_last=True,
    )

# 注册到Toolkit
toolkit.register_tool_function(my_custom_tool)
```

### 2. 使用长期记忆

```python
from agentscope.memory import ReMePersonalLongTermMemory

long_term_memory = ReMePersonalLongTermMemory(
    api_key="your-api-key",
)

agent = ReActAgent(
    # ...其他参数
    long_term_memory=long_term_memory,
    long_term_memory_mode="both",  # agent_control, static_control, both
)
```

### 3. 初始化配置

```python
import agentscope

agentscope.init(
    project="MyProject",
    name="Run-001",
    logging_path="./logs",
    logging_level="INFO",
    studio_url="http://localhost:5000",  # 可选，连接AgentScope Studio
)
```

---

## 常见问题

### 1. 模型调用失败
**问题**：`API key not found`或认证错误  
**解决**：检查对应模型API的环境变量是否设置正确。

### 2. 工具调用不生效
**问题**：Agent没有调用工具  
**解决**：
- 检查工具是否正确注册到Toolkit
- 检查系统提示词是否明确告知Agent可以使用工具
- 确保Formatter支持工具调用格式

### 3. 多智能体消息不共享
**问题**：智能体之间看不到对方的消息  
**解决**：
- 确保在`MsgHub`上下文中执行
- 检查是否使用了支持多智能体的Formatter（如`DashScopeMultiAgentFormatter`）

### 4. 流式输出不显示
**问题**：设置了`stream=True`但没有流式效果  
**解决**：
- 确保模型支持流式输出
- 检查是否在异步环境中正确调用
