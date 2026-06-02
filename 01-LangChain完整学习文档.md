# LangChain + LangGraph 完整学习文档

> 从零到生产级的 AI Agent 学习资料
>
> 适用版本：Python 3.14.5 · LangChain 1.3+ · LangGraph 1.2+ · LangSmith · PostgreSQL 16 + pgvector

---

## 1 环境与工具链

### 1.1 为什么要学习 LangChain？

大语言模型的能力再强，也只是"单次问答"。真实业务需要的是**能自主决策、调用工具、管理记忆、串联多个步骤**的程序。LangChain 提供了标准化的抽象层，让开发者用统一接口对接不同 LLM 提供商，并通过 LangGraph 构建有状态、可控制的多步骤 Agent 系统。

### 1.2 安装 uv 与 Python

```bash
# 检查 Python 版本（需 3.10+）
python3 --version

# 安装 uv（如果未安装）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 验证
uv --version
```

### 1.3 创建项目并安装依赖

```bash
# 创建项目目录
mkdir langchain-agent && cd langchain-agent

# 初始化 uv 项目
uv init

# 安装核心依赖
uv add "langchain>=1.3.0" "langchain-core>=0.3.0"
uv add "langchain-openai" "langchain-anthropic" "langchain-ollama"
uv add "langgraph>=1.2.0"
uv add "langchain-postgres>=0.0.17"
uv add "langchain-community"
uv add "httpx" "python-dotenv"

# 开发依赖
uv add --dev "pytest" "pytest-asyncio"

# 查看安装的版本
uv pip show langchain langchain-core langgraph
```

**关键说明：**
- `langchain-openai` / `langchain-anthropic` / `langchain-ollama` 是各提供商独立包，核心包不捆绑任何提供商
- `langgraph` 是底层运行时，`create_agent` 构建在其之上
- `langchain-postgres` 提供 pgvector 向量存储

### 1.4 设置环境变量

```bash
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_PROJECT=langchain-agent
```

### 1.5 验证安装

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4o-mini")
result = llm.invoke("Hello! Say 'OK' if you can hear me.")
print(result.content)
# 输出: OK
```

**Key Points:**
- `init_chat_model("provider:model_name")` 是 v1 的统一模型初始化入口
- 未设置 API key 会抛出 `AuthenticationError`

> **💡 Tip:** 如果只安装了一个提供商，可以简写 `init_chat_model("gpt-4o-mini")`，自动检测

### 1.6 Docker 环境（PostgreSQL + pgvector）

```bash
# 启动 PostgreSQL + pgvector
docker run --name pgvector-db \
  -e POSTGRES_USER=langchain \
  -e POSTGRES_PASSWORD=langchain \
  -e POSTGRES_DB=langchain \
  -p 5432:5432 \
  -d pgvector/pgvector:pg16

# 验证连接
docker exec -it pgvector-db psql -U langchain -d langchain -c "SELECT extname FROM pg_extension;"
# 输出应包含 pgvector
```

---

## 2 AI Agent 核心概念

### 2.1 什么是 AI Agent？

Agent 是一个**能自主感知环境、做出决策、执行行动**的智能程序。与传统程序不同，Agent 不是固定的 if-else 流程，而是：

```
用户输入 → 思考（LLM 推理）→ 行动（调用工具/生成回复）→ 观察结果 → 循环直到完成
```

### 2.2 Agent 的核心组件

| 组件 | 职责 | LangChain 中的实现 |
|------|------|-------------------|
| 大脑 (LLM) | 推理、决策、生成 | `BaseChatModel` |
| 工具 (Tools) | 与外部世界交互 | `@tool`, `BaseTool` |
| 记忆 (Memory) | 保存对话历史与状态 | `checkpointer`, `BaseStore` |
| 规划 (Planning) | 分解任务、决定步骤 | Agent loop + middleware |
| 执行 (Execution) | 调用工具、处理结果 | LangGraph graph runtime |

### 2.3 Agent 的工作循环

```python
# 这是 create_agent 内部的核心循环（伪代码）
def agent_loop(model, tools, messages):
    while True:
        # 1. 思考：调用 LLM
        response = model.invoke(messages)

        # 2. 判断：LLM 是否请求调用工具
        if not response.tool_calls:
            return response  # 完成

        # 3. 行动：执行所有工具调用
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call)
            messages.append(result)
```

### 2.4 LangChain 与 LangGraph 的关系

```
┌─────────────────────────────────────┐
│          LangChain (抽象层)           │
│  create_agent · init_chat_model     │
│  @tool · Middleware · Prompt        │
├─────────────────────────────────────┤
│          LangGraph (运行时)           │
│  StateGraph · Nodes · Edges         │
│  Checkpointer · Streaming · Interrupt│
├─────────────────────────────────────┤
│      LLM Providers (OpenAI, etc.)    │
└─────────────────────────────────────┘
```

**关键说明：** LangChain v1 的 `create_agent` 底层就是 LangGraph 的 `StateGraph`。你可以从 `create_agent` 开始，需要更精细控制时直接使用 `StateGraph`。

> **💡 Tip:** 初学者以 `create_agent` 为主，遇到复杂分支逻辑再切换到 `StateGraph`

### 2.5 第一个 Agent

```python
from langchain.agents import create_agent

def get_weather(location: str) -> str:
    """获取指定城市的天气"""
    return f"{location} 的天气是20°C，晴"

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather],
    system_prompt="你是一个天气助手，用中文回答。",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "北京今天天气怎么样？"}]
})
print(result["messages"][-1].content)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `str \| BaseChatModel` | 模型标识或实例 |
| `tools` | `list` | 工具列表 |
| `system_prompt` | `str \| SystemMessage` | 系统提示词 |
| `middleware` | `list[AgentMiddleware]` | 中间件链（v1 核心特性） |
| `checkpointer` | `Checkpointer` | 状态持久化 |
| `context_schema` | `type` | 运行时上下文类型 |

---

## 3 LLM 提供商抽象

### 3.1 统一模型初始化

LangChain v1 通过 `init_chat_model` 提供跨提供商的统一接口：

```python
from langchain.chat_models import init_chat_model

# 方式一：字符串标识（推荐）
llm = init_chat_model("openai:gpt-4o-mini")

# 方式二：直接实例化（需要更多控制时）
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 方式三：自动检测提供商
llm = init_chat_model("gpt-4o-mini")  # 自动检测已安装的包
```

### 3.2 字符串格式规范

```
provider:model_name
```

| 提供商 | 包名 | 示例 |
|--------|------|------|
| OpenAI | `langchain-openai` | `openai:gpt-4o` |
| Anthropic | `langchain-anthropic` | `anthropic:claude-sonnet-4-6` |
| Ollama | `langchain-ollama` | `ollama:llama3.2` |
| Google | `langchain-google-genai` | `google:gemini-2.0-flash` |

### 3.3 多提供商路由

生产环境中常需要根据任务类型路由到不同模型：

```python
from langchain.chat_models import init_chat_model
from langchain_core.languageModels import BaseChatModel
from langchain_core.messages import HumanMessage, SystemMessage

class ModelRouter:
    def __init__(self):
        self.models = {
            "fast": init_chat_model("openai:gpt-4o-mini"),
            "powerful": init_chat_model("anthropic:claude-sonnet-4-6"),
            "local": init_chat_model("ollama:llama3.2"),
        }

    def get_model(self, task_type: str) -> BaseChatModel:
        return self.models.get(task_type, self.models["fast"])

router = ModelRouter()

# 简单任务用快速模型
fast_llm = router.get_model("fast")
result = fast_llm.invoke([HumanMessage("1+1=?")])
print(result.content)  # 2

# 复杂推理用强模型
powerful_llm = router.get_model("powerful")
```

| 模型分级 | 适用场景 | 推荐模型 |
|---------|---------|---------|
| 快速/廉价 | 分类、提取、简单问答 | gpt-4o-mini, claude-haiku |
| 均衡 | 大部分任务 | gpt-4o, claude-sonnet |
| 强大/昂贵 | 复杂推理、代码生成 | gpt-5.4, claude-opus |
| 本地 | 离线、隐私敏感 | llama3.2, mistral |

### 3.4 模型参数配置

```python
llm = init_chat_model(
    "openai:gpt-4o-mini",
    temperature=0.7,        # 创造力 0-2
    max_tokens=4096,         # 最大输出 token
    timeout=30,              # 超时秒数
    max_retries=2,           # 失败重试次数
    model_kwargs={
        "frequency_penalty": 0.1,
        "presence_penalty": 0.1,
    },
)
```

> **💡 Tip:** `temperature=0` 用于确定性输出（分类、提取），`temperature=0.7+` 用于创意任务

---

## 4 Chat Models 与 Messages

### 4.1 消息协议

LLM 的输入输出都通过消息对象传递，LangChain 定义了标准消息类型：

```python
from langchain_core.messages import (
    HumanMessage,      # 用户消息
    AIMessage,         # AI 回复
    SystemMessage,     # 系统提示
    ToolMessage,       # 工具调用结果
    AIMessageChunk,    # 流式输出片段
)
```

### 4.2 基础对话

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4o-mini")

messages = [
    SystemMessage("你是一个专业的 Python 导师，用中文回答。"),
    HumanMessage("请解释什么是装饰器？"),
]

result = llm.invoke(messages)
print(result.content)
print(result.response_metadata)  # 包含 token 用量等元数据
```

### 4.3 标准内容块（Standard Content Blocks）

LangChain v1 引入了跨提供商的统一内容表示：

```python
# 传统方式（一直可用）
result = llm.invoke([HumanMessage("Hello")])
print(result.content)  # 字符串

# 新方式：标准内容块
content_blocks = result.content_blocks
for block in content_blocks:
    print(f"Type: {block.type}")     # text / image / tool_use / tool_result
    print(f"Content: {block.text}")   # 文本内容
```

| 内容块类型 | 说明 | 适用场景 |
|-----------|------|---------|
| `text` | 文本内容 | 普通回复 |
| `image` | 图片（base64/URL） | 多模态输入 |
| `tool_use` | 工具调用请求 | Agent 决策 |
| `tool_result` | 工具调用结果 | 工具执行反馈 |
| `thinking` | 思考过程 | Anthropic 的扩展思考 |

### 4.4 流式输出

```python
from langchain_core.messages import HumanMessage

llm = init_chat_model("openai:gpt-4o-mini")

for chunk in llm.stream([HumanMessage("讲个笑话")]):
    print(chunk.content, end="", flush=True)
```

### 4.5 多模态输入

```python
from langchain_core.messages import HumanMessage

llm = init_chat_model("openai:gpt-4o")

message = HumanMessage(
    content=[
        {"type": "text", "text": "描述这张图片"},
        {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}},
    ]
)

result = llm.invoke([message])
print(result.content)
```

---

## 5 Prompt Templates

### 5.1 基础模板

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import SystemMessage, HumanMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家。请用{language}回答。"),
    ("human", "{question}"),
])

messages = prompt.format_messages(
    role="Python",
    language="中文",
    question="什么是异步编程？",
)
print(messages)
# [SystemMessage('你是一个Python专家...'), HumanMessage('什么是异步编程？')]
```

### 5.2 Few-Shot 示例

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "2+2=?", "output": "4"},
    {"input": "3*5=?", "output": "15"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个数学助手"),
    few_shot,
    ("human", "{input}"),
])

messages = final_prompt.format_messages(input="12/4=?")
```

### 5.3 动态提示词（Middleware 方式）

在 Agent 中，动态提示词通过 middleware 实现：

```python
from langchain.agents.middleware import before_model, AgentState
from langgraph.runtime import Runtime

@before_model
def dynamic_prompt(state: AgentState, runtime: Runtime) -> dict:
    user_level = runtime.context.get("user_level", "beginner")
    time_info = f"当前时间：{runtime.context.get('current_time')}"

    return {
        "messages": [
            {"role": "system", "content": f"用户级别：{user_level}。{time_info}"},
            *state["messages"],
        ]
    }
```

---

## 6 Output Parsers 与结构化输出

### 6.1 基础解析器

```python
from langchain_core.output_parsers import (
    CommaSeparatedListOutputParser,
    MarkdownListOutputParser,
    JsonOutputParser,
)
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4o-mini")

# 逗号分隔列表
parser = CommaSeparatedListOutputParser()
prompt = ChatPromptTemplate.from_messages([
    ("human", "列出 5 种编程语言。\n{format_instructions}"),
])
prompt = prompt.partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
result = chain.invoke({})
print(result)  # ['Python', 'JavaScript', 'Go', 'Rust', 'Java']
```

### 6.2 Pydantic 结构化输出（ToolStrategy）

LangChain v1 推荐使用 ToolStrategy 进行结构化输出：

```python
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage

class Person(BaseModel):
    """人物信息"""
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")
    occupation: str = Field(description="职业")

llm = init_chat_model("openai:gpt-4o-mini")

# 方式一：ToolStrategy（推荐，利用 tool calling）
llm_with_structure = llm.with_structured_output(Person)
result = llm_with_structure.invoke([
    HumanMessage("从这句话中提取人物信息：张三，28岁，软件工程师")
])
print(result)  # Person(name='张三', age=28, occupation='软件工程师')
```

### 6.3 ProviderStrategy

```python
# 方式二：ProviderStrategy（JSON 模式，不需要 tool calling）
llm_with_structure = llm.with_structured_output(
    Person,
    method="json_mode",  # 强制使用 JSON 模式
)
```

| 策略 | 要求 | 优点 | 缺点 |
|------|------|------|------|
| `ToolStrategy` | 模型支持 tool calling | 更可靠 | 消耗更多 token |
| `ProviderStrategy` | 模型支持 JSON mode | token 更少 | 可靠性略低 |

### 6.4 Agent 中的结构化输出

```python
from langchain.agents import create_agent

class ResearchReport(BaseModel):
    """研究报告"""
    title: str = Field(description="报告标题")
    summary: str = Field(description="摘要")
    key_findings: list[str] = Field(description="关键发现")
    confidence: float = Field(description="置信度 0-1")

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search_tool],
    system_prompt="你是一个研究员。",
    response_format=ResearchReport,  # 结构化输出
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "研究 AI Agent 的最新趋势"}]
})
```

---

## 7 工具定义与集成

### 7.1 @tool 装饰器

```python
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """获取指定城市的当前天气。

    Args:
        location: 城市名称，如"北京"
    """
    # 实际应该调用天气 API
    return f"{location}: 20°C，晴"

print(get_weather.name)       # "get_weather"
print(get_weather.description) # 函数 docstring
print(get_weather.args)       # {"location": {"title": "Location", "type": "string"}}
```

### 7.2 BaseTool 类

需要更复杂控制时使用 BaseTool：

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大结果数")

class WebSearchTool(BaseTool):
    name: str = "web_search"
    description: str = "搜索网络信息"
    args_schema: type = SearchInput

    def _run(self, query: str, max_results: int = 5) -> str:
        # 实际调用搜索 API
        return f"搜索 '{query}' 的结果（最多{max_results}条）..."

    async def _arun(self, query: str, max_results: int = 5) -> str:
        return await self._run_async(query, max_results)
```

### 7.3 工具错误处理

LangkChain v1 通过 middleware 处理工具错误：

```python
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[unreliable_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            retry_on=(TimeoutError, ConnectionError),
        ),
    ],
)
```

### 7.4 工具依赖注入

```python
from langchain_core.tools import tool
from langgraph.runtime import Runtime

@tool
def get_user_info(runtime: Runtime) -> str:
    """获取当前用户信息（从运行时上下文注入）"""
    user_id = runtime.context.get("user_id")
    return f"User {user_id} info..."

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_user_info],
    context_schema=type("Context", (), {"user_id": str}),
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "我的信息是什么？"}]},
    context={"user_id": "12345"},
)
```

| 工具定义方式 | 适用场景 |
|------------|---------|
| `@tool` 装饰器 | 简单函数，无需状态 |
| `BaseTool` 子类 | 需要配置、状态或异步支持 |
| `@tool` + `Runtime` 注入 | 需要访问运行时上下文 |

---

## 8 LCEL 与 Runnable 协议

### 8.1 Runnable 协议

LangChain 表达式语言（LCEL）的核心是 `Runnable` 协议：任何实现了 `invoke`、`stream`、`batch` 方法的对象都是 Runnable。

```python
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = init_chat_model("openai:gpt-4o-mini")
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家"),
    ("human", "{question}"),
])

# 管道组合：每个组件都是 Runnable
chain = prompt | llm | StrOutputParser()

# 调用
result = chain.invoke({"role": "Python", "question": "什么是闭包？"})
print(result)

# 流式
for chunk in chain.stream({"role": "Python", "question": "什么是闭包？"}):
    print(chunk, end="", flush=True)
```

### 8.2 Runnable 组合方式

```python
from langchain_core.runnables import (
    RunnableParallel,
    RunnablePassthrough,
    RunnableLambda,
)

# 并行执行
parallel = RunnableParallel(
    weather=lambda x: f"{x['city']}天气晴",
    time=lambda x: f"时间：2026年",
)
result = parallel.invoke({"city": "北京"})
# {"weather": "北京天气晴", "time": "时间：2026年"}

# 转换数据
chain = (
    RunnablePassthrough()
    | RunnableLambda(lambda x: {"question": x["text"], "role": "助手"})
    | prompt | llm | StrOutputParser()
)
```

### 8.3 Runnable 与 Agent

`create_agent` 返回的也是 Runnable：

```python
agent = create_agent(model="openai:gpt-4o-mini", tools=[])
print(isinstance(agent, Runnable))  # True

# 可以使用所有 Runnable 方法
agent.invoke(...)
agent.stream(...)
agent.batch([...])
```

---

## 9 LangGraph StateGraph

### 9.1 为什么需要 LangGraph？

当 Agent 逻辑复杂到需要**分支、循环、并行、人机交互**时，`create_agent` 的简单循环就不够了。LangGraph 允许你定义有向图，每个节点是一个处理步骤，边定义了数据流向。

### 9.2 StateGraph 基础

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage, AIMessage

# 1. 定义状态
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # 消息列表（自动追加）
    next_step: str                           # 下一步标识

# 2. 定义节点
llm = init_chat_model("openai:gpt-4o-mini")

def chatbot(state: AgentState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return "end"

# 3. 构建图
builder = StateGraph(AgentState)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_conditional_edges(
    "chatbot",
    should_continue,
    {"tools": "tools", "end": END},
)

# 4. 编译
graph = builder.compile()

# 5. 执行
result = graph.invoke({
    "messages": [HumanMessage("你好！")],
    "next_step": "",
})
print(result["messages"][-1].content)
```

### 9.3 状态管理详解

```python
from typing import Annotated, TypedDict
from operator import add

class WorkflowState(TypedDict):
    messages: Annotated[list, add_messages]  # 追加模式
    counter: Annotated[int, add]             # 累加模式
    metadata: dict                           # 替换模式（无 reducer）
```

| reducer 函数 | 行为 | 适用场景 |
|-------------|------|---------|
| `add_messages` | 追加到消息列表 | 对话历史 |
| `operator.add` | 累加数值 | 计数、总分 |
| 无 | 直接替换 | 单一值状态 |
| 自定义函数 | 自定义合并逻辑 | 复杂状态 |

### 9.4 条件分支

```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(AgentState)

def router(state: AgentState) -> str:
    last = state["messages"][-1].content.lower()
    if "天气" in last:
        return "weather"
    elif "计算" in last:
        return "calculator"
    return "fallback"

builder.add_conditional_edges(
    "chatbot",
    router,
    {
        "weather": "weather_node",
        "calculator": "calc_node",
        "fallback": "fallback_node",
    },
)
```

### 9.5 人机交互断点

```python
# 在编译时设置断点
graph = builder.compile(
    interrupt_before=["approval_node"],  # 在 approval_node 之前暂停
)

# 执行到断点
result = graph.invoke({
    "messages": [HumanMessage("发送邮件给老板")],
})

# 查看待审批
print(result.get("pending_approval"))

# 继续执行
result = graph.invoke(None, config={"configurable": {"thread_id": "thread-1"}})
```

---

## 10 create_agent（v1 核心）

### 10.1 create_agent 概述

`create_agent` 是 LangChain v1 构建 Agent 的标准方式。它内部使用 LangGraph 的 `StateGraph`，封装了完整的 agent 循环（LLM 调用 → 工具执行 → 循环）。

```python
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search_tool, calculator_tool],
    system_prompt="你是一个有用的助手。",
)
```

### 10.2 完整的 Agent 示例

```python
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """搜索网络信息"""
    return f"关于'{query}'的搜索结果..."

@tool
def calculate(expression: str) -> str:
    """计算数学表达式"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"计算错误：{e}"

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search, calculate],
    system_prompt="你是一个智能助手。搜索信息后回答用户问题。",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "2024年奥运会金牌榜第一名是谁？"}]
})
print(result["messages"][-1].content)
```

### 10.3 状态持久化（Checkpointer）

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent

# 内存检查点（开发用）
checkpointer = InMemorySaver()

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    checkpointer=checkpointer,
)

# 第一次对话
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是张三"}]},
    config={"configurable": {"thread_id": "thread-1"}},
)

# 第二次对话（记住上下文）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么名字？"}]},
    config={"configurable": {"thread_id": "thread-1"}},
)
print(result["messages"][-1].content)  # 张三
```

### 10.4 运行时上下文

```python
from pydantic import BaseModel
from langchain.agents import create_agent

class RuntimeContext(BaseModel):
    user_id: str
    timezone: str = "Asia/Shanghai"

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_user_info],
    context_schema=RuntimeContext,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "我的信息是什么？"}]},
    context={"user_id": "u-001", "timezone": "Asia/Shanghai"},
)
```

### 10.5 create_agent vs create_react_agent

| 特性 | `create_agent` (v1) | `create_react_agent` (已废弃) |
|------|-------------------|----------------------------|
| 导入路径 | `langchain.agents` | `langgraph.prebuilt` |
| 定制方式 | Middleware | Pre/post hooks |
| 状态定义 | `state_schema` 或 middleware | `state_schema` |
| 模型选择 | Middleware 动态切换 | Callable |
| 系统提示 | `system_prompt` 参数 | `prompt` 参数 |
| 工具错误 | `ToolRetryMiddleware` | 需手动实现 |

---

## 11 内置 Middleware 实战

### 11.1 Middleware 架构

Middleware 是 LangChain v1 的核心创新，它让你在不修改 Agent 内部代码的前提下，插入自定义逻辑。

```
请求进入 → before_agent → before_model → wrap_model_call → model
                                                          ↓
请求返回 ← after_agent  ← after_model  ← wrap_tool_call ← tools
```

| Hook | 时机 | 用途 |
|------|------|------|
| `before_agent` | Agent 开始前 | 验证输入、加载记忆 |
| `before_model` | 每次 LLM 调用前 | 注入动态 prompt、修剪消息 |
| `wrap_model_call` | 包裹 LLM 调用 | 动态换模型、重试 |
| `wrap_tool_call` | 包裹工具调用 | 工具拦截、审计 |
| `after_model` | 每次 LLM 调用后 | 验证输出、计数 |
| `after_agent` | Agent 完成后 | 保存结果、清理 |

### 11.2 SummarizationMiddleware

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",   # 用于摘要的模型
            trigger=("tokens", 4000),      # 超过 4000 token 时触发
            keep=("messages", 20),         # 保留最近 20 条
        ),
    ],
    checkpointer=InMemorySaver(),
)
```

### 11.3 HumanInTheLoopMiddleware

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[send_email, search],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {
                    "description": "请审核此邮件",
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "search": False,  # False = 不需要审批
            }
        ),
    ],
)

# 执行（会在 send_email 前暂停）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "给老板发邮件说项目完成"}]},
    config={"configurable": {"thread_id": "t1"}},
)

# 查看中断信息
print(result.get("interrupts"))

# 批准继续
result = agent.invoke(
    None,
    config={"configurable": {"thread_id": "t1"}},
)
```

### 11.4 PIIMiddleware

```python
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[
        PIIMiddleware(
            "email",                     # PII 类型
            strategy="redact",           # redact/block/annotate
            apply_to_input=True,         # 处理输入
        ),
        PIIMiddleware(
            "phone_number",
            strategy="block",            # 阻止包含电话的请求
        ),
    ],
)
```

### 11.5 重试与回退

```python
from langchain.agents.middleware import (
    ToolRetryMiddleware,
    ModelRetryMiddleware,
    ModelFallbackMiddleware,
)

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[unreliable_api],
    middleware=[
        ToolRetryMiddleware(max_retries=3, retry_on=(TimeoutError,)),
        ModelRetryMiddleware(max_retries=2),
        ModelFallbackMiddleware(
            "openai:gpt-4o-mini",     # 主模型
            "anthropic:claude-haiku",  # 回退模型
        ),
    ],
)
```

### 11.6 速率限制

```python
from langchain.agents.middleware import (
    ModelCallLimitMiddleware,
    ToolCallLimitMiddleware,
)

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search, api_call],
    checkpointer=InMemorySaver(),
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=50,   # 每次对话最多 50 次 LLM 调用
            run_limit=20,      # 单次执行最多 20 次
            exit_behavior="end",
        ),
        ToolCallLimitMiddleware(
            tool_name="api_call",
            thread_limit=10,   # 每次对话最多调用 10 次
        ),
    ],
)
```

### 11.7 工具选择器

```python
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[tool1, tool2, tool3, tool4, tool5],
    middleware=[
        LLMToolSelectorMiddleware(
            model="openai:gpt-4o-mini",
            max_tools=3,          # 每次最多暴露 3 个工具给 LLM
            always_include=["search"],  # 总是包含的工具
        ),
    ],
)
```

---

## 12 自定义 Middleware

### 12.1 装饰器方式

```python
from langchain.agents.middleware import (
    before_model, after_model, before_agent,
    AgentState,
)
from langgraph.runtime import Runtime

@before_agent
def validate_input(state: AgentState, runtime: Runtime) -> dict | None:
    """验证用户输入"""
    messages = state["messages"]
    if messages and len(messages[0].content) > 10000:
        return {
            "messages": [AIMessage("输入过长，请精简后重试。")],
            "jump_to": "end",  # 跳过后续步骤
        }
    return None

@before_model
def inject_context(state: AgentState, runtime: Runtime) -> dict | None:
    """注入上下文信息"""
    user_level = runtime.context.get("user_level", "beginner")
    return {
        "messages": [
            SystemMessage(f"用户级别：{user_level}"),
            *state["messages"],
        ],
    }

@after_model
def log_usage(state: AgentState, runtime: Runtime) -> dict | None:
    """记录 token 使用"""
    last = state["messages"][-1]
    usage = last.response_metadata.get("token_usage", {})
    print(f"Token used: {usage}")
    return {"total_tokens": state.get("total_tokens", 0) + usage.get("total_tokens", 0)}

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[validate_input, inject_context, log_usage],
)
```

### 12.2 类方式

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langgraph.runtime import Runtime

class AuditMiddleware(AgentMiddleware):
    """审计中间件：记录所有操作"""

    def __init__(self, audit_log: list):
        super().__init__()
        self.audit_log = audit_log

    def before_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
        self.audit_log.append({
            "event": "agent_start",
            "user_id": runtime.context.get("user_id"),
            "message_count": len(state["messages"]),
        })
        return None

    @hook_config(can_jump_to=["end"])
    def before_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        """如果成本超限，终止对话"""
        cost = state.get("total_cost", 0)
        if cost > 1.0:  # 超过 $1
            return {
                "messages": [AIMessage("对话成本超限，已终止。")],
                "jump_to": "end",
            }
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        last = state["messages"][-1]
        usage = last.response_metadata.get("token_usage", {})
        cost = (usage.get("prompt_tokens", 0) * 0.00001 +
                usage.get("completion_tokens", 0) * 0.00003)
        return {"total_cost": state.get("total_cost", 0) + cost}

    def after_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
        self.audit_log.append({
            "event": "agent_end",
            "total_cost": state.get("total_cost"),
        })
        return None
```

### 12.3 Wrap 风格 Middleware

```python
from langchain.agents.middleware import (
    AgentMiddleware, ModelRequest, ModelResponse,
)

class LoggingModelMiddleware(AgentMiddleware):
    """记录模型请求和响应"""

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: callable,
    ) -> ModelResponse:
        print(f"模型请求：{len(request.state['messages'])} 条消息")
        start = time.time()

        response = handler(request)

        elapsed = time.time() - start
        print(f"响应耗时：{elapsed:.2f}s")
        print(f"Token 用量：{response.usage}")

        return response

    def wrap_tool_call(self, request, handler):
        print(f"工具调用：{request.tool_name}({request.args})")
        result = handler(request)
        print(f"工具结果：{str(result)[:100]}")
        return result
```

### 12.4 Middleware 执行顺序

```python
agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[m1, m2, m3],  # 顺序执行
)

# before hooks 顺序：m1 → m2 → m3
# after hooks 逆序：m3 → m2 → m1
```

| Hook | 执行方向 | 说明 |
|------|---------|------|
| `before_agent` | 顺序 | 按注册顺序执行 |
| `before_model` | 顺序 | 按注册顺序执行 |
| `wrap_model_call` | 洋葱圈 | 最外层先进、最后出 |
| `wrap_tool_call` | 洋葱圈 | 同上 |
| `after_model` | 逆序 | 最后注册的最先执行 |
| `after_agent` | 逆序 | 最后注册的最先执行 |

---

## 13 多 Agent 系统

### 13.1 Supervisor Agent 模式

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SubAgentMiddleware

# 定义子 Agent
researcher = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    system_prompt="你是一个研究员，负责搜索和分析信息。",
)

writer = create_agent(
    model="openai:gpt-4o-mini",
    tools=[],
    system_prompt="你是一个写手，负责将研究结果整理成文章。",
)

# Supervisor Agent
agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[],
    middleware=[
        SubAgentMiddleware(
            default_model="openai:gpt-4o-mini",
            subagents=[
                {
                    "name": "researcher",
                    "description": "搜索和分析信息",
                    "system_prompt": "使用搜索工具查找信息。",
                    "tools": [search],
                    "model": "openai:gpt-4o-mini",
                },
                {
                    "name": "writer",
                    "description": "撰写文章",
                    "system_prompt": "将研究结果写成文章。",
                    "tools": [],
                    "model": "openai:gpt-4o-mini",
                },
            ],
        ),
    ],
)
```

### 13.2 LangGraph 多 Agent 编排

```python
from langgraph.graph import StateGraph, START, END
from langchain.agents import create_agent

# 创建团队
researcher = create_agent(model="openai:gpt-4o-mini", tools=[search])
critic = create_agent(model="openai:gpt-4o-mini", tools=[])

class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    task: str
    research: str
    review: str
    iteration: int

def research_node(state: TeamState) -> dict:
    result = researcher.invoke({
        "messages": [{"role": "user", "content": state["task"]}]
    })
    return {"research": result["messages"][-1].content, "iteration": state["iteration"] + 1}

def review_node(state: TeamState) -> dict:
    result = critic.invoke({
        "messages": [
            {"role": "system", "content": "审查研究成果，提出改进意见。如果满意请回复 APPROVED。"},
            {"role": "user", "content": f"任务：{state['task']}\n研究结果：{state['research']}"},
        ]
    })
    return {"review": result["messages"][-1].content}

def decide(state: TeamState) -> str:
    if "APPROVED" in state["review"] or state["iteration"] >= 3:
        return "end"
    return "research"

builder = StateGraph(TeamState)
builder.add_node("research", research_node)
builder.add_node("review", review_node)
builder.add_edge(START, "research")
builder.add_edge("research", "review")
builder.add_conditional_edges("review", decide, {"research": "research", "end": END})
team = builder.compile()
```

### 13.3 Agent Team 通信模式

| 模式 | 描述 | 适用场景 |
|------|------|---------|
| Supervisor | 一个 Agent 协调多个子 Agent | 任务分发 |
| Debate | 多个 Agent 讨论达成共识 | 质量审查 |
| Pipeline | Agent 链式传递结果 | 内容生产 |
| Mesh | 所有 Agent 自由通信 | 复杂协作 |

---

## 14 RAG 与 pgvector

### 14.1 什么是 RAG？

检索增强生成（RAG）让 Agent 从外部知识库检索相关信息，作为上下文注入 LLM。解决 LLM 知识过期、幻觉问题。

```
用户问题 → 向量检索（pgvector）→ 相关文档 → LLM + 上下文 → 准确回答
```

### 14.2 配置 pgvector

```bash
# 前提：PostgreSQL 16 + pgvector 已运行（见 1.6）
```

### 14.3 PGEngine 连接池

```python
from langchain_postgres import PGEngine, PGVectorStore
from langchain_openai import OpenAIEmbeddings

# 连接字符串
CONNECTION_STRING = "postgresql+asyncpg://langchain:langchain@localhost:5432/langchain"

# 创建引擎（连接池）
engine = PGEngine.from_connection_string(url=CONNECTION_STRING)

# 初始化向量表
VECTOR_SIZE = 1536  # OpenAI text-embedding-3-small 的维度
TABLE_NAME = "documents"

await engine.ainit_vectorstore_table(
    table_name=TABLE_NAME,
    vector_size=VECTOR_SIZE,
)
```

### 14.4 创建向量存储

```python
from langchain_openai import OpenAIEmbeddings
from langchain_postgres import PGVectorStore
from langchain_core.documents import Document

embedding = OpenAIEmbeddings(model="text-embedding-3-small")

store = await PGVectorStore.create(
    engine=engine,
    table_name=TABLE_NAME,
    embedding_service=embedding,
)

# 添加文档
docs = [
    Document(
        page_content="LangChain v1 于 2025 年 10 月发布，引入了 Middleware 系统。",
        metadata={"source": "langchain-docs", "category": "release"},
    ),
    Document(
        page_content="create_agent 是 LangChain v1 构建 Agent 的标准方式。",
        metadata={"source": "langchain-docs", "category": "agents"},
    ),
]
await store.aadd_documents(docs)
```

### 14.5 语义搜索

```python
# 相似度搜索
results = await store.asimilarity_search(
    "如何创建 LangChain Agent？",
    k=3,
    filter={"source": "langchain-docs"},
)
for doc in results:
    print(f"{doc.page_content} ({doc.metadata})")

# 带分数的搜索
results_with_score = await store.asimilarity_search_with_score(
    "如何创建 LangChain Agent？",
    k=3,
)
for doc, score in results_with_score:
    print(f"{score:.3f}: {doc.page_content}")
```

### 14.6 混合搜索

```python
from langchain_postgres import HybridSearchConfig
from langchain_postgres.utils import reciprocal_rank_fusion

store = await PGVectorStore.create(
    engine=engine,
    table_name=TABLE_NAME,
    embedding_service=embedding,
    hybrid_search_config=HybridSearchConfig(
        fusion_function=reciprocal_rank_fusion,
    ),
)

# 混合搜索（向量 + 全文）
results = await store.asimilarity_search(
    "LangChain 版本发布",
    k=5,
)
```

### 14.7 Agent 中使用 RAG

```python
from langchain_core.tools import tool

class KnowledgeBase:
    def __init__(self):
        self.store = None  # PGVectorStore 实例

    @tool
    async def search_knowledge(self, query: str) -> str:
        """从知识库中搜索相关信息"""
        docs = await self.store.asimilarity_search(query, k=3)
        return "\n\n".join(d.page_content for d in docs)

kb = KnowledgeBase()

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[kb.search_knowledge, web_search],
    system_prompt="""你是一个内部知识助手。
    优先搜索内部知识库，找不到再用 Web 搜索。""",
)
```

### 14.8 元数据过滤

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `$eq` | 等于 | `{"source": "docs"}` |
| `$in` | 在列表中 | `{"category": {"$in": ["tech", "science"]}}` |
| `$gt`/`$gte` | 大于/大于等于 | `{"year": {"$gte": 2025}}` |
| `$like` | 模糊匹配 | `{"title": {"$like": "%Agent%"}}` |
| `$and`/`$or` | 逻辑组合 | `{"$and": [{"a": 1}, {"b": 2}]}` |

---

## 15 记忆系统

### 15.1 Checkpointer 持久化

LangGraph 通过 Checkpointer 实现会话持久化：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# 开发：内存
memory = InMemorySaver()

# 生产：SQLite
sqlite_memory = SqliteSaver.from_conn_string("checkpoints.db")

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    checkpointer=sqlite_memory,
)
```

### 15.2 thread_id 会话管理

```python
# 不同 thread_id = 不同会话
session_1 = agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是张三"}]},
    config={"configurable": {"thread_id": "session-A"}},
)

session_2 = agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是李四"}]},
    config={"configurable": {"thread_id": "session-B"}},
)

# session-A 记得张三
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么？"}]},
    config={"configurable": {"thread_id": "session-A"}},
)
# 输出：张三
```

### 15.3 Store（长期记忆）

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# 存储用户偏好
store.put(
    ("users", "user-123"),
    "preferences",
    {"language": "zh-CN", "tone": "professional"},
)

# Agent 中访问 Store
from langchain.agents.middleware import before_model

@before_model
def load_preferences(state, runtime):
    prefs = runtime.store.get(("users", "user-123"), "preferences")
    return {
        "messages": [
            SystemMessage(f"用户偏好：{prefs}"),
            *state["messages"],
        ],
    }
```

### 15.4 消息历史管理与摘要

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    checkpointer=InMemorySaver(),
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",
            trigger=("messages", 20),  # 超过 20 条时摘要
            keep=("messages", 10),      # 保留最近 10 条原始消息
        ),
    ],
)
```

---

## 16 可观测性（LangSmith）

### 16.1 配置 LangSmith

```bash
# .env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_PROJECT=my-agent-project
```

### 16.2 自动追踪

配置环境变量后，所有 `create_agent` 和 `StateGraph` 调用自动追踪：

```python
# 无需额外代码，所有调用自动上报
agent = create_agent(model="openai:gpt-4o-mini", tools=[search])
agent.invoke({"messages": [{"role": "user", "content": "你好"}]})
# 在 LangSmith 控制台可以看到：
# - 完整调用链
# - Token 用量
# - 延迟分析
# - 错误追踪
```

### 16.3 手动追踪

```python
from langchain_core.tracers import LangChainTracer
from langchain_core.callbacks import CallbackManager

tracer = LangChainTracer(
    project_name="custom-tracing",
)
manager = CallbackManager([tracer])

agent.invoke(
    {"messages": [{"role": "user", "content": "你好"}]},
    config={"callbacks": manager},
)
```

### 16.4 评估（Eval）

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# 定义数据集
dataset = client.create_dataset("agent-qa")
client.create_examples(
    dataset_id=dataset.id,
    inputs=[{"question": "1+1=?"}],
    outputs=[{"answer": "2"}],
)

# 评估
def target(inputs: dict) -> dict:
    result = agent.invoke({"messages": [{"role": "user", "content": inputs["question"]}]})
    return {"answer": result["messages"][-1].content}

experiment_results = evaluate(
    target,
    data=dataset.name,
    evaluators=[exact_match_evaluator],
)
```

---

## 17 错误处理与重试策略

### 17.1 LLM 调用错误处理

```python
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            retry_on=(RateLimitError, APIError),
            backoff_factor=2.0,  # 指数退避
        ),
    ],
)
```

### 17.2 工具错误处理

```python
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[unreliable_api],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            retry_on=(TimeoutError, ConnectionError, HTTPError),
            backoff_factor=1.5,
        ),
    ],
)
```

### 17.3 模型回退

```python
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[
        ModelFallbackMiddleware(
            "openai:gpt-4o-mini",      # 主要模型
            "anthropic:claude-haiku",   # 第一次回退
            "ollama:llama3.2",          # 最终回退（本地）
        ),
    ],
)
```

### 17.4 自定义错误处理 Middleware

```python
@wrap_model_call
def resilient_model_call(request, handler):
    for attempt in range(3):
        try:
            return handler(request)
        except RateLimitError as e:
            wait = min(2 ** attempt * 10, 60)
            print(f"限流，等待 {wait}s...")
            time.sleep(wait)
        except APIError as e:
            if attempt == 2:
                # 最后一次尝试：换模型
                request = request.override(
                    model="openai:gpt-4o-mini",
                )
            print(f"API 错误，重试 {attempt+1}/3")
    raise RuntimeError("LLM 调用全部失败")
```

---

## 18 安全最佳实践

### 18.1 PII 过滤

```python
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    middleware=[
        PIIMiddleware("email", strategy="redact"),
        PIIMiddleware("phone_number", strategy="block"),
        PIIMiddleware(
            "credit_card",
            strategy="redact",
            detector=r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b",
        ),
    ],
)
```

### 18.2 API Key 管理

```python
# .env 文件（永远不要提交到 Git）
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# 使用 pydantic-settings 管理
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str
    anthropic_api_key: str = ""
    langchain_tracing: bool = False
    langchain_api_key: str = ""

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()
assert settings.openai_api_key, "OPENAI_API_KEY 未设置"
```

### 18.3 Prompt 注入防护

```python
@before_model
def guard_against_injection(state: AgentState, runtime: Runtime) -> dict | None:
    last_user_msg = None
    for msg in reversed(state["messages"]):
        if msg.type == "human":
            last_user_msg = msg.content
            break

    if last_user_msg:
        injections = [
            "ignore previous instructions",
            "forget all previous",
            "you are now",
            "system prompt",
        ]
        for pattern in injections:
            if pattern in last_user_msg.lower():
                return {
                    "messages": [AIMessage("检测到可能的 Prompt 注入，已拦截。")],
                    "jump_to": "end",
                }
    return None
```

### 18.4 速率限制

```python
from langchain.agents.middleware import (
    ModelCallLimitMiddleware,
    ToolCallLimitMiddleware,
)

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search, expensive_api],
    checkpointer=InMemorySaver(),
    middleware=[
        ModelCallLimitMiddleware(
            user_limit=100,    # 每个用户最多 100 次
            thread_limit=20,   # 每次对话最多 20 次
            run_limit=10,      # 单次执行最多 10 次
            exit_behavior="end",
        ),
        ToolCallLimitMiddleware(
            tool_name="expensive_api",
            user_limit=5,     # 每个用户最多调用 5 次
        ),
    ],
)
```

### 18.5 Token 预算控制

```python
class TokenBudgetMiddleware(AgentMiddleware):
    def __init__(self, max_tokens: int = 100000):
        self.max_tokens = max_tokens

    @hook_config(can_jump_to=["end"])
    def before_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        used = state.get("total_tokens", 0)
        if used >= self.max_tokens:
            return {
                "messages": [AIMessage(f"Token 预算已超限（{used}/{self.max_tokens}）")],
                "jump_to": "end",
            }
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        last = state["messages"][-1]
        usage = last.response_metadata.get("token_usage", {})
        return {
            "total_tokens": state.get("total_tokens", 0) + usage.get("total_tokens", 0),
        }
```

---

## 19 部署与生产化

### 19.1 FastAPI 封装

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain.agents import create_agent
from langgraph.checkpoint.sqlite import SqliteSaver

app = FastAPI(title="Agent API")

class ChatRequest(BaseModel):
    message: str
    thread_id: str
    user_id: str

class ChatResponse(BaseModel):
    reply: str
    thread_id: str

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search],
    checkpointer=SqliteSaver.from_conn_string("checkpoints.db"),
)

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    try:
        result = await agent.ainvoke(
            {"messages": [{"role": "user", "content": request.message}]},
            config={
                "configurable": {"thread_id": request.thread_id},
                "context": {"user_id": request.user_id},
            },
        )
        return ChatResponse(
            reply=result["messages"][-1].content,
            thread_id=request.thread_id,
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### 19.2 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.14-slim

WORKDIR /app

# 安装 uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# 复制依赖文件
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# 复制源代码
COPY . .

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: langchain
      POSTGRES_PASSWORD: langchain
      POSTGRES_DB: langchain
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U langchain"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### 19.3 Health Check

```python
from fastapi import FastAPI
from langgraph.checkpoint.sqlite import SqliteSaver

app = FastAPI()

@app.get("/health")
async def health():
    """健康检查端点"""
    checks = {
        "status": "ok",
        "checks": {},
    }

    # 检查数据库连接
    try:
        db = SqliteSaver.from_conn_string("checkpoints.db")
        db.list(None, limit=1)
        checks["checks"]["database"] = "ok"
    except Exception as e:
        checks["status"] = "degraded"
        checks["checks"]["database"] = str(e)

    # 检查 LLM 连接
    try:
        llm = init_chat_model("openai:gpt-4o-mini")
        llm.invoke("test")
        checks["checks"]["llm"] = "ok"
    except Exception as e:
        checks["status"] = "degraded"
        checks["checks"]["llm"] = str(e)

    return checks
```

### 19.4 优雅关闭

```python
import signal
import asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动
    app.state.agent = create_agent(...)
    yield
    # 关闭：清理资源
    await app.state.agent.acheckpointer.aflush()

app = FastAPI(lifespan=lifespan)
```

### 19.5 CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy Agent
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run pytest

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t agent-app .
      - run: docker push registry.example.com/agent-app:latest
```

---

## 20 参考资料

### 官方文档

- LangChain v1 官方文档：https://docs.langchain.com/oss/python/
- LangChain v1 迁移指南：https://docs.langchain.com/oss/python/migrate/langchain-v1
- LangGraph 官方文档：https://docs.langchain.com/oss/python/langgraph/
- LangGraph API 参考：https://reference.langchain.com/python/langgraph
- LangSmith 文档：https://docs.smith.langchain.com/

### 关键包

- `langchain` - 核心包（包含 `create_agent`, `init_chat_model`）
- `langchain-core` - 基础抽象（`BaseChatModel`, `BaseTool`, `Runnable`）
- `langgraph` - Agent 运行时
- `langchain-openai` / `langchain-anthropic` / `langchain-ollama` - 提供商集成
- `langchain-postgres` - pgvector 向量存储
- `langchain-community` - 社区集成

### 版本信息

```ini
PYTHON_VERSION=3.14.5
LANGCHAIN_VERSION=1.3+
LANGGRAPH_VERSION=1.2+
LANGCHAIN_CORE_VERSION=0.3+
LANGCHAIN_POSTGRES_VERSION=0.0.17+
POSTGRESQL_VERSION=16
PGVECTOR_VERSION=0.8+
UV_VERSION=0.11+
```

### 推荐阅读

1. LangChain v1 新特性：https://docs.langchain.com/oss/python/releases/langchain-v1
2. Agent Middleware 介绍：https://www.langchain.com/blog/agent-middleware
3. LangGraph 概念指南：https://docs.langchain.com/oss/python/langgraph/concepts/
4. PGVectorStore 集成指南：https://docs.langchain.com/oss/python/integrations/vectorstores/pgvectorstore
5. LangSmith 评估指南：https://docs.smith.langchain.com/evaluation

### 常见问题快速索引

| 问题 | 章节 | 解决方案 |
|------|------|---------|
| API Key 未设置 | 1.4 | 检查 `.env` 文件和 `Settings` 类 |
| 模型返回格式不对 | 6.2 | 使用 `with_structured_output` |
| Agent 循环不退出 | 10.2 | 检查工具是否被正确调用 |
| 对话历史丢失 | 15.1 | 配置 Checkpointer |
| 上下文超长 | 11.2 | 使用 `SummarizationMiddleware` |
| 工具调用失败 | 11.5 | 使用 `ToolRetryMiddleware` |
| 向量检索不准确 | 14.5 | 调整分块大小和检索参数 |
| 端口冲突 | 1.6 | `lsof -i :5432` 检查端口 |
