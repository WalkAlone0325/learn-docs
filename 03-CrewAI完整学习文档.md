# CrewAI 完整学习文档

> 从零到生产级的 CrewAI 学习资料
>
> 适用版本：Python 3.13+ · CrewAI 1.14.6+ · PostgreSQL 16 + pgvector

---

## 1 环境与工具链

### 1.1 为什么要学习 CrewAI？

CrewAI 是一个**独立于 LangChain** 的 AI Agent 编排框架。与 LangChain 不同，CrewAI 从零构建，专注于**角色协作**——每个 Agent 有明确角色（Role）、目标（Goal）和背景故事（Backstory），通过任务（Task）协作完成复杂工作流。

CrewAI 的优势在于：
- **简洁直观**：基于角色的设计更符合人类团队协作模式
- **高性能**：框架轻量，独立于 LangChain
- **Flows 架构**：生产级的流程编排和状态管理
- **YAML 配置**：Agent 和 Task 配置与代码分离

### 1.2 Python 环境（⚠ CrewAI 需要 Python 3.10~3.13）

```bash
# 检查当前 Python 版本
python3 --version
# 输出: Python 3.14.5

# CrewAI 需要 Python < 3.14，安装 Python 3.13
uv python install 3.13

# 创建项目并指定 Python 3.13
mkdir crewai-project && cd crewai-project
uv init --python 3.13

# 验证
uv python list
# 输出应包含 3.13.x
```

### 1.3 安装 CrewAI

```bash
# 安装 CrewAI（自动包含 CLI）
uv add crewai

# 安装 CrewAI 工具包
uv add crewai-tools

# 验证安装
crewai version
# 输出: 1.14.6

# 查看所有命令
crewai --help
```

### 1.4 创建项目

```bash
# 方式一：使用 CLI 创建（推荐）
crewai create crew research-crew
cd research-crew

# 查看生成的结构
tree src/research_crew/
# src/research_crew/
# ├── __init__.py
# ├── crews/
# ├── config/
# │   ├── agents.yaml
# │   └── tasks.yaml
# ├── crew.py
# ├── main.py
# └── tools/

# 方式二：手动创建（灵活）
mkdir -p my_crew/src/my_crew/config
touch my_crew/src/my_crew/{__init__.py,crew.py,main.py}
touch my_crew/src/my_crew/config/{agents.yaml,tasks.yaml}
```

### 1.5 创建 Flow 项目

```bash
# CrewAI 推荐使用 Flow 架构
crewai create flow content-flow
```

### 1.6 设置环境变量

```bash
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
SERPER_API_KEY=...  # 网络搜索（可选）

# 验证
from crewai import Agent
agent = Agent(role="test", goal="test", backstory="test")
print("CrewAI 安装成功")
```

### 1.7 Docker 环境（PostgreSQL + pgvector）

```bash
docker run --name pgvector-db \
  -e POSTGRES_USER=crewai \
  -e POSTGRES_PASSWORD=crewai \
  -e POSTGRES_DB=crewai \
  -p 5432:5432 \
  -d pgvector/pgvector:pg16

# 验证
docker exec -it pgvector-db psql -U crewai -d crewai -c "SELECT extname FROM pg_extension;"
```

---

## 2 CrewAI 核心哲学

### 2.1 角色驱动设计

CrewAI 的核心思想是：**让 Agent 像真实团队成员一样工作**。

```
传统 Agent:     LLM + 工具 → 回答
CrewAI Agent:   角色 + 目标 + 背景故事 + 工具 → 协作完成任务
```

```python
from crewai import Agent

researcher = Agent(
    role="高级研究员",
    goal="发现并整理最新的行业趋势",
    backstory="""你是一名有 15 年经验的研究员，擅长从海量信息中
    提取关键洞察。你的报告以深度和准确性著称。""",
    verbose=True,
)
```

### 2.2 核心组件关系

```
┌─────────────────────────────────────┐
│             Crew (团队)               │
│  ┌──────────┐  ┌──────────┐         │
│  │  Agent 1  │  │  Agent 2  │  ...   │
│  │ (角色A)   │  │ (角色B)   │         │
│  └─────┬────┘  └─────┬────┘         │
│        │              │              │
│  ┌─────▼──────────────▼────┐         │
│  │      Tasks (任务)       │         │
│  │  Task1 → Task2 → Task3  │         │
│  └─────────────────────────┘         │
│        Process: sequential/hierarchical │
└─────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│          Flow (流程编排)              │
│  @start → @listen → @listen         │
│  状态管理 · 路由 · 持久化             │
└─────────────────────────────────────┘
```

### 2.3 CrewAI vs LangChain

| 特性 | CrewAI | LangChain |
|------|--------|-----------|
| 设计哲学 | 角色协作 | 组件编排 |
| 框架独立性 | 完全独立 | 需要 LangChain 生态 |
| 配置方式 | YAML + 装饰器 | Python 代码 |
| 流程控制 | Flows / Process | LangGraph |
| 学习曲线 | 较低 | 中等 |
| 适用场景 | 固定角色团队、内容生产 | 通用 Agent 构建 |

> **💡 Tip:** 如果需要灵活的 Agent 循环（自定义状态、条件分支），选 LangChain/LangGraph；如果业务是固定的角色协作流程（如内容生产、研究分析），CrewAI 更高效

---

## 3 LLM 提供商集成

### 3.1 默认模型配置

CrewAI 默认使用 OpenAI GPT-4，可以通过多种方式配置：

```python
import os

# 方式一：环境变量（自动读取）
os.environ["OPENAI_API_KEY"] = "sk-..."

# 方式二：Agent 级别指定
agent = Agent(
    role="研究员",
    goal="搜索信息",
    backstory="...",
    llm="gpt-4o-mini",  # 字符串方式
)
```

### 3.2 多提供商配置

```python
from crewai import Agent, LLM

# 方式三：使用 LLM 对象（推荐，支持多提供商）
openai_llm = LLM(
    model="gpt-4o-mini",
    temperature=0.7,
    max_tokens=4096,
)

anthropic_llm = LLM(
    model="anthropic/claude-sonnet-4-6",  # 注意格式不同
    temperature=0.3,
)

local_llm = LLM(
    model="ollama/llama3.2",
    base_url="http://localhost:11434",
)

# 不同 Agent 使用不同模型
fast_agent = Agent(
    role="快速分析员",
    goal="快速分类信息",
    backstory="...",
    llm=openai_llm,
)

deep_agent = Agent(
    role="深度分析员",
    goal="复杂推理",
    backstory="...",
    llm=anthropic_llm,
)
```

### 3.3 模型参数

```python
llm = LLM(
    model="gpt-4o-mini",
    temperature=0.7,        # 0-2，越高越有创意
    max_tokens=4096,         # 最大输出
    top_p=0.9,               # 核采样
    frequency_penalty=0.1,   # 频率惩罚
    presence_penalty=0.1,    # 存在惩罚
    stop=["STOP"],           # 停止词
    seed=42,                 # 随机种子（一致性）
)
```

| 提供商 | 字符串格式 | 环境变量 |
|--------|----------|---------|
| OpenAI | `gpt-4o` / `gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `anthropic/claude-sonnet-4-6` | `ANTHROPIC_API_KEY` |
| Ollama | `ollama/llama3.2` | - |
| Google | `google/gemini-2.0-flash` | `GOOGLE_API_KEY` |
| Azure | `azure/gpt-4o` | `AZURE_API_KEY` |

---

## 4 Agent 定义

### 4.1 基本结构

```python
from crewai import Agent

agent = Agent(
    role="数据分析师",
    goal="从数据中提取 actionable 的洞察",
    backstory="""你是一名资深数据分析师，在科技行业工作 10 年。
    你擅长发现数据模式并用通俗语言解释复杂发现。""",
    verbose=True,     # 打印详细日志
    allow_delegation=True,  # 允许委派任务给其他 Agent
)
```

### 4.2 Agent 参数详解

```python
agent = Agent(
    # 必需参数
    role="研究报告撰写人",
    goal="撰写清晰、准确的研究报告",
    backstory="你是一名专业的技术写手...",

    # 可选参数
    verbose=True,                    # 调试日志
    allow_delegation=True,           # 允许委派（hierarchical 模式下）
    max_retry_limit=3,               # 最大重试次数
    max_iter=25,                     # 最大迭代次数
    llm="gpt-4o-mini",              # 自定义模型
    function_calling_llm="gpt-4o-mini",  # 工具调用专用模型
    memory=False,                    # 是否启用记忆
    respect_context_window=True,     # 遵守上下文窗口限制
    code_execution_mode="safe",      # 代码执行模式
    tools=[],                        # Agent 可用工具
    step_callback=None,              # 每步回调
)
```

### 4.3 YAML 配置方式（推荐）

```yaml
# config/agents.yaml
researcher:
  role: >
    高级研究员
  goal: >
    对给定课题进行深入研究，收集全面的信息
  backstory: >
    你是一名资深研究员，擅长从多个来源收集和分析信息。
    你的研究笔记详尽且有条理。
  verbose: true
  allow_delegation: false

writer:
  role: >
    技术写手
  goal: >
    基于研究资料撰写清晰、专业的技术报告
  backstory: >
    你是一名技术文档专家，擅长将复杂概念转化为易懂的文字。
    你的报告结构清晰、语言精准。
  verbose: true
```

```python
# crew.py 中引用
from crewai import Agent
from crewai.project import CrewBase, agent

@CrewBase
class ResearchCrew:
    agents_config = "config/agents.yaml"

    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config["researcher"],
            tools=[search_tool],
        )

    @agent
    def writer(self) -> Agent:
        return Agent(
            config=self.agents_config["writer"],
        )
```

### 4.4 Agent 委派

```python
# 在 hierarchical 模式下，manager Agent 可以委派任务
manager = Agent(
    role="项目经理",
    goal="协调团队完成项目",
    backstory="你是一个有经验的项目经理...",
    allow_delegation=True,  # 允许委派
)

worker = Agent(
    role="执行者",
    goal="完成任务",
    backstory="...",
    allow_delegation=False,  # 不允许再委派
)

crew = Crew(
    agents=[manager, worker],
    tasks=[...],
    process=Process.hierarchical,
    manager_agent=manager,
)
```

---

## 5 Task 定义

### 5.1 基本结构

```python
from crewai import Task

research_task = Task(
    description="研究 AI Agent 在 2026 年的最新发展趋势",
    expected_output="""一份结构化的研究笔记，包含：
    1. 三大主要趋势
    2. 每个趋势的关键技术
    3. 代表性公司和产品
    4. 未来发展预测""",
    agent=researcher,  # 指定执行 Agent
)
```

### 5.2 Task 参数详解

```python
task = Task(
    # 必需参数
    description="对一个主题进行深入研究",  # 任务描述
    expected_output="一份 Markdown 格式的研究报告",  # 期望输出

    # 可选参数
    agent=researcher,              # 负责 Agent（不指定则由 Crew 分配）
    tools=[search_tool],           # 任务专用工具（覆盖 Agent 工具）
    context=[previous_task],       # 依赖的前置任务输出
    async_execution=False,         # 是否异步执行
    callback=my_callback_fn,       # 完成回调
    output_file="output/report.md", # 输出文件路径
    output_json=ReportModel,       # JSON 输出（Pydantic 模型）
    output_pydantic=ReportModel,   # Pydantic 输出
    guardrail=validation_fn,       # 输出验证函数
    max_retries=3,                 # 最大重试次数
)
```

### 5.3 任务依赖与上下文

```python
# Task A → Task B（B 需要 A 的输出）
research_task = Task(
    description="研究主题",
    expected_output="研究笔记",
    agent=researcher,
)

writing_task = Task(
    description="撰写报告",
    expected_output="完整报告",
    agent=writer,
    context=[research_task],  # 依赖 research_task 的输出
)

# 异步任务（并行执行）
data_task_1 = Task(
    description="收集数据源 A",
    async_execution=True,  # 异步执行
)

data_task_2 = Task(
    description="收集数据源 B",
    async_execution=True,  # 异步执行
)

summary_task = Task(
    description="汇总数据",
    context=[data_task_1, data_task_2],  # 等待两个任务完成
)
```

### 5.4 YAML 配置方式

```yaml
# config/tasks.yaml
research_task:
  description: >
    对{topic}进行深入研究。搜索最新的信息和数据，
    确保信息来源可靠。当前年份是 2026 年。
  expected_output: >
    结构化的研究笔记，包含关键发现、数据来源和参考文献。
  agent: researcher

writing_task:
  description: >
    基于研究笔记撰写完整报告。
    报告应包含摘要、正文、结论和参考文献。
  expected_output: >
    一篇完整的 Markdown 格式报告，不低于 2000 字。
  agent: writer
  context:
    - research_task
  output_file: output/report.md
```

### 5.5 结构化输出

```python
from pydantic import BaseModel

class ResearchReport(BaseModel):
    title: str
    summary: str
    key_findings: list[str]
    sources: list[str]
    conclusion: str

task = Task(
    description="研究 AI Agent 趋势",
    expected_output="结构化研究报告",
    agent=researcher,
    output_pydantic=ResearchReport,  # 输出自动解析为 Pydantic 模型
)

# 执行任务
result = crew.kickoff()
print(result.pydantic.title)
print(result.pydantic.key_findings)
```

---

## 6 Tool 创建与集成

### 6.1 @tool 装饰器

```python
from crewai.tools import tool

@tool("Web Search")
def web_search(query: str) -> str:
    """搜索网络获取信息。输入：搜索关键词。"""
    return f"关于'{query}'的搜索结果..."

# 使用
tools = [web_search]
```

### 6.2 预置工具

```python
# crewai-tools 提供了丰富的预置工具
from crewai_tools import (
    SerperDevTool,      # Google 搜索
    ScrapeWebsiteTool,  # 网页抓取
    FileReadTool,       # 文件读取
    FileWriteTool,      # 文件写入
    CSVSearchTool,      # CSV 搜索
    MDXSearchTool,      # Markdown 搜索
    JSONSearchTool,     # JSON 搜索
    CodeDocsSearchTool, # 代码文档搜索
    YoutubeChannelSearchTool,
    WikipediaTool,
)

# 使用示例
search_tool = SerperDevTool()
scrape_tool = ScrapeWebsiteTool()

agent = Agent(
    role="研究员",
    goal="搜索和分析信息",
    backstory="...",
    tools=[search_tool, scrape_tool],
)
```

### 6.3 自定义 Tool 类

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

class DatabaseQueryInput(BaseModel):
    query: str = Field(description="SQL 查询语句")

class DatabaseQueryTool(BaseTool):
    name: str = "Database Query"
    description: str = "执行数据库查询并返回结果"
    args_schema: type = DatabaseQueryInput

    def __init__(self, connection_string: str):
        super().__init__()
        self.connection_string = connection_string

    def _run(self, query: str) -> str:
        # 执行查询
        return f"查询结果：{query}"

    async def _arun(self, query: str) -> str:
        # 异步版本
        return await self._run_async(query)
```

### 6.4 缓存工具结果

```python
from crewai.tools import tool
from functools import lru_cache

@tool("Cached Search")
@lru_cache(maxsize=100)
def cached_search(query: str) -> str:
    """带缓存的搜索"""
    return f"搜索'{query}'的结果..."
```

### 6.5 MCP 工具集成

CrewAI 支持 MCP（Model Context Protocol）工具：

```python
from crewai_tools import MCPTool

# 连接 MCP 服务器
mcp_tool = MCPTool(
    name="custom_service",
    description="调用自定义微服务",
    url="http://localhost:8000/mcp",
)
```

---

## 7 Crew 与 Process

### 7.1 基本 Crew

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,  # 顺序执行（默认）
    verbose=True,
)

# 执行
result = crew.kickoff(inputs={"topic": "AI Agent 趋势"})
```

### 7.2 两种流程模式

```python
# 顺序模式（Sequential）
# Task1 → Task2 → Task3（线性执行）
from crewai import Process

sequential_crew = Crew(
    agents=[agent1, agent2, agent3],
    tasks=[task1, task2, task3],
    process=Process.sequential,
)

# 层级模式（Hierarchical）
# Manager Agent 分配任务给 Worker Agent
from crewai import Process

hierarchical_crew = Crew(
    agents=[manager, worker1, worker2],
    tasks=[task1, task2],
    process=Process.hierarchical,
    manager_llm="gpt-4o",  # Manager 使用更强模型
)
```

| 模式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Sequential | 固定流程、流水线 | 简单可预测 | 缺乏灵活性 |
| Hierarchical | 需要任务分配和协调 | 灵活、可扩展 | 需要 Manager LLM |

### 7.3 传入参数

```python
result = crew.kickoff(inputs={
    "topic": "2026 AI Agent 趋势",
    "depth": "deep",
    "format": "markdown",
})

# 在 YAML 中使用变量
# tasks.yaml
# research_task:
#   description: "研究 {topic}，深度：{depth}"
```

### 7.4 Crew 回调

```python
def on_task_start(task):
    print(f"任务开始：{task.description[:50]}...")

def on_task_end(task):
    print(f"任务完成：{task.description[:50]}...")

def on_crew_end(crew):
    print("团队任务全部完成")

crew = Crew(
    agents=[...],
    tasks=[...],
    task_callback=on_task_end,
    step_callback=lambda agent, task: print(f"{agent.role}: {task.description[:30]}..."),
)
```

---

## 8 Flow 架构

### 8.1 什么是 Flow？

Flow 是 CrewAI 的**生产级流程编排架构**。它提供：
- **状态管理**：通过 `FlowState` 管理跨步骤的状态
- **事件驱动**：`@start()` / `@listen()` 装饰器定义执行顺序
- **路由**：`@router()` 实现条件分支
- **持久化**：Flow 状态可持久化

### 8.2 基础 Flow

```python
from crewai.flow import Flow, listen, start
from pydantic import BaseModel

class ContentState(BaseModel):
    topic: str = ""
    research: str = ""
    article: str = ""

class ContentFlow(Flow[ContentState]):
    @start()
    def set_topic(self):
        self.state.topic = "AI Agent 2026"

    @listen(set_topic)
    def do_research(self):
        result = ResearchCrew().crew().kickoff(
            inputs={"topic": self.state.topic}
        )
        self.state.research = result.raw

    @listen(do_research)
    def write_article(self):
        result = WriteCrew().crew().kickoff(
            inputs={"research": self.state.research}
        )
        self.state.article = result.raw

    @listen(write_article)
    def save_result(self):
        with open("output/article.md", "w") as f:
            f.write(self.state.article)

flow = ContentFlow()
flow.kickoff()
```

### 8.3 路由（条件分支）

```python
from crewai.flow import Flow, start, listen, router
import random

class ReviewFlow(Flow):
    @start()
    def generate_content(self):
        return "这是一篇需要审查的文章内容"

    @router(generate_content)
    def review_content(self):
        score = random.randint(1, 10)
        if score >= 7:
            return "approved"
        return "rejected"

    @listen("approved")
    def publish(self):
        print("文章已发布")

    @listen("rejected")
    def revise(self):
        print("需要修改")
        return self.revise()  # 重新提交审查（注意避免无限循环）
```

### 8.4 Flow 状态管理

```python
from pydantic import BaseModel
from crewai.flow import Flow, start, listen

class PipelineState(BaseModel):
    article_count: int = 0
    errors: list[str] = []
    current_phase: str = "init"

class PipelineFlow(Flow[PipelineState]):
    @start()
    def initialize(self):
        self.state.current_phase = "started"
        self.state.article_count = 0

    @listen(initialize)
    def check_errors(self):
        if self.state.errors:
            print(f"发现 {len(self.state.errors)} 个错误")

flow = PipelineFlow()
flow.kickoff()
```

### 8.5 Flow vs Crew

| 特性 | Crew | Flow |
|------|------|------|
| 职责 | 执行一组 Agent 任务 | 编排多个 Crew 或步骤 |
| 状态管理 | 无 | 有（FlowState） |
| 执行顺序 | Sequential / Hierarchical | 事件驱动（start/listen） |
| 条件分支 | 无 | 有（router） |
| 适用场景 | 单个工作单元 | 多步骤工作流 |
| 生产推荐 | 作为 Flow 的步骤 | 主架构 |

---

## 9 记忆系统

### 9.1 记忆类型

```python
from crewai import Agent

agent = Agent(
    role="客服代表",
    goal="提供持续的服务体验",
    backstory="...",
    memory=True,  # 启用记忆
)

# CrewAI 支持四种记忆：
# - ShortTermMemory（短期记忆）：当前对话
# - LongTermMemory（长期记忆）：跨会话
# - EntityMemory（实体记忆）：记住用户/事物信息
# - UserMemory（用户记忆）：用户偏好
```

### 9.2 配置记忆存储

```python
from crewai.memory.storage import SQLiteStorage

crew = Crew(
    agents=[agent],
    tasks=[task],
    memory=True,
    memory_config={
        "provider": "sqlite",
        "storage": SQLiteStorage(db_path="crew_memory.db"),
    },
)
```

### 9.3 实体记忆

```python
# 自动提取和存储实体信息
agent = Agent(
    role="销售代表",
    goal="了解客户需求",
    backstory="...",
    memory=True,
)

# Agent 会自动记住对话中提到的实体
# 如：客户公司名、联系人、偏好等
```

---

## 10 RAG 集成

### 10.1 知识源

CrewAI 支持知识源（Knowledge Sources）用于 RAG：

```python
from crewai.knowledge.source import (
    TextFileKnowledgeSource,
    PDFKnowledgeSource,
    CSVKnowledgeSource,
    JSONKnowledgeSource,
)

# 创建知识源
text_source = TextFileKnowledgeSource(
    file_path="docs/knowledge_base.txt",
    chunk_size=1000,
    chunk_overlap=200,
)

pdf_source = PDFKnowledgeSource(
    file_path="docs/manual.pdf",
)
```

### 10.2 将知识源加入 Crew

```python
crew = Crew(
    agents=[agent],
    tasks=[task],
    knowledge_sources=[text_source, pdf_source],
    embedder={
        "provider": "openai",
        "config": {
            "model": "text-embedding-3-small",
        },
    },
)
```

### 10.3 pgvector 集成

```python
import psycopg2
from pgvector.psycopg2 import register_vector

# 直接使用 pgvector 作为知识存储
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    user="crewai",
    password="crewai",
    dbname="crewai",
)
register_vector(conn)

# 创建向量表
cur = conn.cursor()
cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
cur.execute("""
    CREATE TABLE IF NOT EXISTS knowledge (
        id SERIAL PRIMARY KEY,
        content TEXT,
        embedding vector(1536),
        metadata JSONB
    )
""")
conn.commit()
```

### 10.4 搜索知识库

```python
# Tool 封装知识库搜索
from crewai.tools import tool
from openai import OpenAI
import numpy as np

client = OpenAI()

@tool("Knowledge Search")
def search_knowledge(query: str) -> str:
    """从内部知识库搜索相关信息"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=query,
    )
    embedding = response.data[0].embedding

    cur.execute("""
        SELECT content, 1 - (embedding <=> %s::vector) as similarity
        FROM knowledge
        ORDER BY similarity DESC
        LIMIT 3
    """, (embedding,))

    results = cur.fetchall()
    if not results:
        return "未找到相关信息"
    return "\n\n".join(f"{r[0]}" for r in results)
```

---

## 11 多 Crew 协作

### 11.1 Crew 嵌套

```python
from crewai.flow import Flow, start, listen

# 子 Crew
class ResearchCrew:
    def run(self, inputs: dict) -> str:
        researcher = Agent(role="研究员", goal="研究", backstory="...")
        task = Task(description="研究{topic}", expected_output="笔记", agent=researcher)
        crew = Crew(agents=[researcher], tasks=[task])
        return crew.kickoff(inputs=inputs).raw

class ContentCrew:
    def run(self, inputs: dict) -> str:
        writer = Agent(role="写手", goal="写作", backstory="...")
        task = Task(description="基于{research}撰写文章", expected_output="文章", agent=writer)
        crew = Crew(agents=[writer], tasks=[task])
        return crew.kickoff(inputs=inputs).raw

# Flow 编排多个 Crew
class ProductionFlow(Flow):
    @start()
    def research_phase(self):
        return ResearchCrew().run({"topic": "AI Agents"})

    @listen(research_phase)
    def content_phase(self, research):
        return ContentCrew().run({"research": research})

    @listen(content_phase)
    def done(self, article):
        print(f"完成！文章长度：{len(article)} 字")
```

### 11.2 并行 Crew

```python
from crewai.flow import Flow, start, listen

class ParallelResearchFlow(Flow):
    @start()
    def research_trends(self):
        return ResearchCrew().run({"topic": "AI 趋势"})

    @start()
    def research_market(self):
        return ResearchCrew().run({"topic": "市场规模"})

    @start()
    def research_competitors(self):
        return ResearchCrew().run({"topic": "竞品分析"})

    @listen(research_trends, research_market, research_competitors)
    def synthesize(self, results):
        trends, market, competitors = results
        return f"{trends}\n{market}\n{competitors}"
```

---

## 12 回调与事件监听

### 12.1 任务回调

```python
def task_callback(task):
    """任务完成后触发"""
    print(f"[{task.agent.role}] 完成: {task.description[:50]}...")
    print(f"输出预览: {task.output.raw[:100]}...")

def step_callback(agent, task):
    """每个 Agent 步骤后触发"""
    print(f"[{agent.role}] 执行中...")

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    task_callback=task_callback,
    step_callback=step_callback,
)
```

### 12.2 Flow 事件

```python
from crewai.flow import Flow, start, listen

class ObservableFlow(Flow):
    @start()
    def step_one(self):
        print("Flow 开始")
        return "step one done"

    @listen(step_one)
    def step_two(self, result):
        print(f"收到步骤一并传入: {result}")
        return "step two done"

    @listen(step_two)
    def step_three(self, result):
        print(f"最终结果: {result}")
        return "complete"
```

### 12.3 Plot 可视化

```python
# 可视化 Flow
flow = ProductionFlow()
flow.plot()  # 生成 flow.html 可视化图表
```

---

## 13 错误处理与重试

### 13.1 任务重试

```python
# Task 级别的重试
task = Task(
    description="调用外部 API",
    expected_output="API 响应",
    agent=agent,
    max_retries=3,  # 最大重试次数
)

# 注意：max_retries 已弃用，改用 guardrail_max_retries
task = Task(
    description="...",
    agent=agent,
    guardrail=validate_output,
    guardrail_max_retries=3,  # guardrail 失败时重试
)
```

### 13.2 Guardrail 验证

```python
def validate_json_output(output):
    """验证输出是否为有效 JSON"""
    import json
    try:
        data = json.loads(output.raw)
        if "title" not in data:
            return False, "缺少 title 字段"
        return True, None
    except json.JSONDecodeError:
        return False, "无效的 JSON 格式"

task = Task(
    description="生成结构化数据",
    expected_output="JSON 格式的数据",
    agent=agent,
    guardrail=validate_json_output,
    guardrail_max_retries=3,
)
```

### 13.3 错误处理最佳实践

```python
# Crew 级别的错误处理
try:
    result = crew.kickoff(inputs={"topic": "AI"})
except Exception as e:
    print(f"Crew 执行失败：{e}")
    # 记录错误日志
    # 发送告警
    # 执行回退方案
```

---

## 14 安全与速率限制

### 14.1 API Key 管理

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str
    serper_api_key: str = ""
    model: str = "gpt-4o-mini"

    model_config = {"env_file": ".env"}

settings = Settings()
assert settings.openai_api_key, "OPENAI_API_KEY 未设置"
```

### 14.2 速率限制

```python
import time
from functools import wraps

def rate_limit(max_calls: int, period: int):
    def decorator(func):
        calls = []
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            calls[:] = [c for c in calls if now - c < period]
            if len(calls) >= max_calls:
                raise Exception("速率限制超限")
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@tool("Rate Limited API")
@rate_limit(max_calls=10, period=60)
def api_call(endpoint: str) -> str:
    """调用外部 API（每分钟最多 10 次）"""
    return f"API 响应：{endpoint}"
```

---

## 15 缓存与优化

### 15.1 工具缓存

```python
from crewai.tools import tool
from functools import lru_cache

@tool("Cached Knowledge")
@lru_cache(maxsize=200)
def cached_knowledge(query: str) -> str:
    """带缓存的知识库查询"""
    return search_database(query)

# 清除缓存
cached_knowledge.cache_clear()
```

### 15.2 Agent 缓存配置

```python
agent = Agent(
    role="高效分析员",
    goal="快速分析",
    backstory="...",
    cache=True,  # 启用 Agent 级别缓存
)
```

### 15.3 请求合并

```python
# 在 Task 层面，合并相似请求
# 相同输入的任务会自动复用缓存结果
task1 = Task(description="研究 AI", agent=agent)
task2 = Task(description="研究 AI", agent=agent)
# 第二次不会重复调用 LLM
```

---

## 16 文件处理

### 16.1 任务输出到文件

```python
task = Task(
    description="撰写报告",
    expected_output="完整报告",
    agent=writer,
    output_file="output/report.md",  # 自动写入文件
)
```

### 16.2 结构化输出

```python
from pydantic import BaseModel

class MarketReport(BaseModel):
    overview: str
    market_size: float
    growth_rate: float
    key_players: list[str]
    risks: list[str]

task = Task(
    description="生成市场分析报告",
    expected_output="结构化市场数据",
    agent=analyst,
    output_pydantic=MarketReport,  # Pydantic 模型输出
)

result = crew.kickoff()
report = result.pydantic  # 直接访问模型
print(f"市场规模：${report.market_size}B")
print(f"增长率：{report.growth_rate}%")
```

### 16.3 CSV 工具

```python
from crewai_tools import CSVSearchTool

csv_tool = CSVSearchTool(csv="data/sales.csv")

agent = Agent(
    role="数据分析师",
    goal="分析销售数据",
    backstory="...",
    tools=[csv_tool],
)
```

---

## 17 可观测性

### 17.1 OpenTelemetry

```bash
# 配置 OTEL
OTEL_SERVICE_NAME=crewai-app
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

```python
# CrewAI 自动集成 OpenTelemetry
# 设置环境变量后，所有 Agent 调用自动上报
import os
os.environ["OTEL_SERVICE_NAME"] = "my-crew-app"
```

### 17.2 日志配置

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)

crew = Crew(
    agents=[agent],
    tasks=[task],
    verbose=True,  # 详细日志
)
```

### 17.3 追踪执行

```python
# CrewAI 默认输出执行追踪
crew.kickoff(inputs={"topic": "AI"})
# 输出示例：
# [研究员] 任务开始：研究 AI
# [研究员] 使用工具：Web Search
# [研究员] 任务完成：研究 AI
# [写手] 任务开始：撰写报告
```

---

## 18 部署与生产化

### 18.1 FastAPI 封装

```python
# app.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from crewai.flow import Flow, start, listen

app = FastAPI(title="CrewAI API")

class ResearchRequest(BaseModel):
    topic: str
    depth: str = "medium"

class ResearchResponse(BaseModel):
    report: str
    status: str

class ResearchFlow(Flow):
    def __init__(self, request: ResearchRequest):
        super().__init__()
        self.request = request

    @start()
    def execute(self):
        crew = ResearchCrew()
        result = crew.crew().kickoff(
            inputs={"topic": self.request.topic}
        )
        return ResearchResponse(report=result.raw, status="completed")

@app.post("/research", response_model=ResearchResponse)
async def start_research(req: ResearchRequest):
    try:
        flow = ResearchFlow(req)
        result = flow.kickoff()
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.14.6"}
```

### 18.2 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./output:/app/output

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: crewai
      POSTGRES_PASSWORD: crewai
      POSTGRES_DB: crewai
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crewai"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### 18.3 CLI 运行

```bash
# 开发
crewai run

# 或直接 Python
python src/my_crew/main.py

# 交互模式
crewai chat
```

### 18.4 pyproject.toml 配置

```toml
[project]
name = "research-crew"
version = "1.0.0"
requires-python = ">=3.10,<3.14"
dependencies = [
    "crewai>=1.14.0",
    "crewai-tools>=1.14.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "python-dotenv>=1.0.0",
    "pydantic-settings>=2.10.0",
]

[project.scripts]
run = "my_crew.main:run"
```

---

## 19 常见问题排查

### 19.1 Python 版本问题

```bash
# CrewAI 不支持 Python 3.14
# 症状：安装失败
# 解决：使用 Python 3.13

uv python install 3.13
uv init --python 3.13
```

### 19.2 API Key 问题

```bash
# 症状：OpenAI API 返回 401
# 检查：
echo $OPENAI_API_KEY
grep OPENAI_API_KEY .env

# 解决：确保 .env 文件存在且格式正确
```

### 19.3 Crew 执行超时

```python
# 设置 Agent 最大迭代次数
agent = Agent(
    role="研究员",
    goal="...",
    backstory="...",
    max_iter=25,  # 增加迭代上限
    max_execution_time=300,  # 5 分钟超时
)
```

### 19.4 工具调用失败

```python
# 检查工具是否返回正确格式
@tool("Search")
def safe_search(query: str) -> str:
    try:
        result = search_api(query)
        return result
    except Exception as e:
        return f"搜索失败：{e}"  # 返回错误信息而非抛异常
```

### 19.5 Flow 状态丢失

```python
# 确保 FlowState 继承自 BaseModel
from pydantic import BaseModel

class AppState(BaseModel):
    data: str = ""  # 提供默认值

class AppFlow(Flow[AppState]):
    ...
```

| 问题 | 原因 | 解决 |
|------|------|------|
| 安装失败 | Python 版本超出限制 | 使用 Python 3.13 |
| Agent 不执行工具 | 工具未传入 | 确认 `tools=[...]` 参数 |
| 输出格式错误 | 未设置 structured output | 使用 `output_pydantic` |
| 任务顺序错误 | context 配置不当 | 检查 Task 的 context 参数 |
| 记忆不生效 | memory 未启用 | 设置 `memory=True` |

---

## 20 参考资料

### 官方文档

- CrewAI 官方文档：https://docs.crewai.com/
- CrewAI 快速开始：https://docs.crewai.com/en/quickstart
- CrewAI Tasks：https://docs.crewai.com/en/concepts/tasks
- CrewAI CLI：https://docs.crewai.com/en/concepts/cli
- CrewAI Changelog：https://docs.crewai.com/en/changelog

### 关键包

- `crewai` - 核心框架（Python >=3.10 <3.14）
- `crewai-tools` - 官方工具包
- `crewai-cli` - 命令行工具

### GitHub

- CrewAI GitHub：https://github.com/crewAIInc/crewAI

### 版本信息

```ini
PYTHON_VERSION=3.13.x
CREWAI_VERSION=1.14.6+
POSTGRESQL_VERSION=16
PGVECTOR_VERSION=0.8+
UV_VERSION=0.11+
```

### 推荐阅读

1. CrewAI Flows 指南：https://docs.crewai.com/en/concepts/flows
2. CrewAI Agents 概念：https://docs.crewai.com/en/concepts/agents
3. MCP 工具集成：https://docs.crewai.com/en/concepts/tools
4. CrewAI 记忆系统：https://docs.crewai.com/en/concepts/memory

### 快速索引

| 需要 | 章节 | 方案 |
|------|------|------|
| 安装 CrewAI | 1.3 | `uv add crewai` |
| 创建项目 | 1.4 | `crewai create crew` |
| 创建 Flow | 1.5 | `crewai create flow` |
| YAML 配置 Agent | 4.3 | `config/agents.yaml` |
| 结构化输出 | 5.5 | `output_pydantic=Model` |
| 启用记忆 | 9 | `memory=True` |
| 多 Crew 协作 | 11 | Flow 编排 |
| 部署 API | 18.1 | FastAPI 封装 |
| Docker 部署 | 18.2 | Dockerfile + Compose |
