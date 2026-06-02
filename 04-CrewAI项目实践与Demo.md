# CrewAI 项目实践与 Demo：内容生产 Pipeline

> **版本:** 1.0.0
> **运行时:** Python 3.13+ · CrewAI 1.14.6+ · PostgreSQL 16 + pgvector

---

## 1 项目概述

### 1.1 项目架构

本项目实现了一个企业级内容生产 Pipeline。三个 Crew 通过 Flow 编排协作，完成从调研到发布的全流程。

```
用户请求 → ContentFlow
              │
  ┌───────────▼───────────┐
  │   Research Crew       │  Sequential
  │   ┌─────────────────┐ │
  │   │ 调研员 → 分析师  │ │
  │   │  → 数据验证员    │ │
  │   └──────────┬──────┘ │
  └──────────────┼────────┘
                 ▼
  ┌───────────▼───────────┐
  │   Content Crew        │  Hierarchical
  │   ┌─────────────────┐ │
  │   │ Manager         │ │
  │   │ ├ 撰稿人        │ │
  │   │ ├ 编辑          │ │
  │   │ └ SEO 优化师    │ │
  │   └──────────┬──────┘ │
  └──────────────┼────────┘
                 ▼
  ┌───────────▼───────────┐
  │   Review Crew         │  Parallel
  │   ┌─────────────────┐ │
  │   │ 质量审查员      │ │
  │   │ 合规审查员      │ │
  │   └──────────┬──────┘ │
  └──────────────┼────────┘
                 ▼
          最终发布
```

### 1.2 技术栈

| 层级 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 语言 | Python | 3.13+ | 应用开发语言 |
| Agent 框架 | CrewAI | 1.14.6+ | Agent 编排 |
| Web 框架 | FastAPI | 0.115+ | API 服务 |
| 向量存储 | pgvector | 0.8+ | 知识库 |
| 嵌入 | OpenAI | text-embedding-3-small | 向量化 |
| 数据库 | PostgreSQL | 16 | 主数据库 |
| 配置 | pydantic-settings | 2.10+ | 环境变量 |
| Auth | python-jose + passlib | - | JWT + 密码 |

### 1.3 API 端点总览

| 方法 | 路径 | 认证 | 说明 | 涉及概念 |
|------|------|------|------|---------|
| POST | `/api/v1/auth/register` | 无 | 注册 | JWT |
| POST | `/api/v1/auth/login` | 无 | 登录 | JWT |
| POST | `/api/v1/pipeline` | JWT | 启动内容生产 | Flow, Multi-Crew |
| GET | `/api/v1/pipeline/{id}` | JWT | 获取状态 | Flow state |
| GET | `/api/v1/pipeline/{id}/article` | JWT | 获取文章 | Task output |
| POST | `/api/v1/pipeline/{id}/approve` | JWT | 审查通过 | Guardrail |
| POST | `/api/v1/knowledge/documents` | JWT | 上传知识文档 | pgvector |
| GET | `/api/v1/knowledge/search` | JWT | 搜索知识库 | RAG |
| GET | `/api/v1/pipeline` | JWT | Pipeline 列表 | 分页查询 |
| GET | `/health` | 无 | 健康检查 | 健康检查 |

---

## 2 项目结构

```
content-pipeline/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI 入口
│   ├── config.py            # 配置
│   ├── database.py          # 数据库连接
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py          # 用户模型
│   │   └── pipeline.py      # Pipeline 模型
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── pipeline.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── pipeline.py
│   │   └── knowledge.py
│   ├── crews/
│   │   ├── __init__.py
│   │   ├── research_crew.py     # 调研 Crew
│   │   ├── content_crew.py      # 内容 Crew
│   │   └── review_crew.py       # 审查 Crew
│   ├── flows/
│   │   ├── __init__.py
│   │   └── content_flow.py      # 主 Flow
│   ├── config/
│   │   ├── research_agents.yaml
│   │   ├── research_tasks.yaml
│   │   ├── content_agents.yaml
│   │   ├── content_tasks.yaml
│   │   ├── review_agents.yaml
│   │   └── review_tasks.yaml
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── search_tools.py
│   │   └── knowledge_tools.py
│   └── middleware/
│       ├── __init__.py
│       └── auth.py
├── .env
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

---

## 3 完整代码实现

### 3.1 配置管理

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Content Pipeline API"

    # Auth
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 60

    # Database
    database_url: str = "sqlite+aiosqlite:///./pipeline.db"

    # pgvector
    pgvector_host: str = "localhost"
    pgvector_port: int = 5432
    pgvector_user: str = "crewai"
    pgvector_password: str = "crewai"
    pgvector_database: str = "crewai"

    @property
    def pgvector_connection_string(self) -> str:
        return (
            f"postgresql+asyncpg://{self.pgvector_user}:{self.pgvector_password}"
            f"@{self.pgvector_host}:{self.pgvector_port}/{self.pgvector_database}"
        )

    # LLM
    openai_api_key: str = ""
    anthropic_api_key: str = ""
    default_model: str = "gpt-4o-mini"
    powerful_model: str = "anthropic/claude-sonnet-4-6"

    model_config = {"env_file": ".env"}

settings = Settings()
```

### 3.2 数据库模型

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(settings.database_url)
async_session = async_sessionmaker(engine, class_=AsyncSession)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncSession:
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.sql import func
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, autoincrement=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(255), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

```python
# app/models/pipeline.py
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Enum as SAEnum
from sqlalchemy.sql import func
import enum
from app.database import Base

class PipelineStatus(str, enum.Enum):
    PENDING = "pending"
    RESEARCHING = "researching"
    WRITING = "writing"
    REVIEWING = "reviewing"
    APPROVED = "approved"
    REJECTED = "rejected"
    FAILED = "failed"

class Pipeline(Base):
    __tablename__ = "pipelines"

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False, index=True)
    topic = Column(String(255), nullable=False)
    keywords = Column(Text, default="")
    status = Column(SAEnum(PipelineStatus), default=PipelineStatus.PENDING)
    article = Column(Text, default="")
    research_data = Column(Text, default="")
    review_feedback = Column(Text, default="")
    error_message = Column(Text, default="")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### 3.3 Auth 中间件

```python
# app/middleware/auth.py
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.config import settings
from app.database import get_db
from app.models.user import User

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: int) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
    return jwt.encode(
        {"sub": str(user_id), "exp": expire},
        settings.jwt_secret,
        algorithm=settings.jwt_algorithm,
    )

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.jwt_secret,
            algorithms=[settings.jwt_algorithm],
        )
        user_id = int(payload.get("sub"))
    except (JWTError, ValueError, TypeError):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user or not user.is_active:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return user
```

```python
# app/api/auth.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, EmailStr
from app.database import get_db
from app.models.user import User
from app.middleware.auth import hash_password, verify_password, create_access_token, get_current_user

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])

class RegisterRequest(BaseModel):
    username: str
    email: EmailStr
    password: str

class LoginRequest(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"

@router.post("/register", response_model=TokenResponse)
async def register(req: RegisterRequest, db: AsyncSession = Depends(get_db)):
    existing = await db.execute(
        select(User).where((User.username == req.username) | (User.email == req.email))
    )
    if existing.scalar_one_or_none():
        raise HTTPException(status_code=409, detail="用户或邮箱已存在")

    user = User(
        username=req.username,
        email=req.email,
        hashed_password=hash_password(req.password),
    )
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return TokenResponse(access_token=create_access_token(user.id))

@router.post("/login", response_model=TokenResponse)
async def login(req: LoginRequest, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.username == req.username))
    user = result.scalar_one_or_none()
    if not user or not verify_password(req.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    return TokenResponse(access_token=create_access_token(user.id))
```

### 3.4 YAML 配置 — Research Crew

```yaml
# app/config/research_agents.yaml
researcher:
  role: >
    高级调研员
  goal: >
    对课题进行全面的信息收集和整理
  backstory: >
    你是一名资深调研员，擅长从多个渠道（网络、知识库、学术）收集信息。
    你的调研笔记详尽、有条理，包含数据来源和关键引用。
  verbose: true
  allow_delegation: false

analyst:
  role: >
    数据分析师
  goal: >
    从调研数据中提取洞察和趋势
  backstory: >
    你是一名数据分析专家，擅长从原始信息中发现模式和趋势。
    你能将复杂数据转化为清晰的洞察。
  verbose: true
  allow_delegation: false

validator:
  role: >
    数据验证员
  goal: >
    验证调研数据的准确性和可靠性
  backstory: >
    你是质量把关人，擅长交叉验证信息来源的可靠性。
    你会标记可疑数据和需要进一步确认的信息。
  verbose: true
  allow_delegation: false
```

```yaml
# app/config/research_tasks.yaml
research_task:
  description: >
    对{topic}进行全面的信息调研。关键词：{keywords}。
    使用网络搜索和知识库工具收集信息。
    关注最新数据、权威来源和不同观点。当前年份是 2026 年。
  expected_output: >
    详细的调研笔记，包含：关键发现、数据统计、信息来源列表、不同观点对比。
  agent: researcher

analysis_task:
  description: >
    基于调研笔记进行分析，提取核心洞察和趋势。
    识别数据中的模式、关联性和异常值。
  expected_output: >
    分析报告，包含：核心洞察（3-5 条）、趋势预测、数据可视化建议。
  agent: analyst
  context:
    - research_task

validation_task:
  description: >
    验证调研数据和分析结论的可靠性。
    检查信息来源的可信度，标记不确定的数据。
  expected_output: >
    验证报告，包含：可信度评分、确认可靠的信息、需要标记的问题、补充建议。
  agent: validator
  context:
    - analysis_task
  output_file: output/research_report.md
```

### 3.5 YAML 配置 — Content Crew

```yaml
# app/config/content_agents.yaml
manager:
  role: >
    内容项目经理
  goal: >
    协调内容创作流程，确保高质量产出
  backstory: >
    你是一名经验丰富的内容项目经理，擅长规划内容策略、
    分配任务和控制质量。你负责确保文章符合要求。
  verbose: true
  allow_delegation: true

writer:
  role: >
    内容撰稿人
  goal: >
    撰写高质量的专业文章
  backstory: >
    你是一名专业的内容创作者，擅长将复杂主题转化为易懂、
    有趣的文章。你的写作风格清晰、有说服力。
  verbose: true
  allow_delegation: false

editor:
  role: >
    内容编辑
  goal: >
    审阅和优化文章内容
  backstory: >
    你是一名资深编辑，对语言质量、逻辑结构和内容准确性
    有极高的要求。你善于在不改变原意的前提下优化表达。
  verbose: true
  allow_delegation: false

seo_specialist:
  role: >
    SEO 优化专家
  goal: >
    优化文章以提高搜索引擎排名
  backstory: >
    你是一名 SEO 专家，精通关键词优化、元数据编写
    和内容结构化。你确保文章既对读者友好又对搜索引擎友好。
  verbose: true
  allow_delegation: false
```

```yaml
# app/config/content_tasks.yaml
writing_task:
  description: >
    基于调研报告撰写一篇关于{topic}的专业文章。
    目标读者：技术决策者。
    文章要求：3000-5000 字，包含引言、正文（3-5 节）、结论。
    使用调研数据支撑论点。
  expected_output: >
    一篇完整的 Markdown 格式文章，结构完整，数据准确。
  agent: writer

editing_task:
  description: >
    审阅并优化文章。检查：
    1. 语言流畅度和专业性
    2. 逻辑结构和段落衔接
    3. 数据准确性和引用完整性
    4. 是否符合目标读者定位
  expected_output: >
    优化后的文章，附带编辑说明（修改了哪些地方及原因）。
  agent: editor
  context:
    - writing_task

seo_task:
  description: >
    对文章进行 SEO 优化：
    1. 优化标题和副标题（包含目标关键词）
    2. 编写 meta description（<160 字）
    3. 优化标题标签（H1/H2/H3）
    4. 添加内部链接建议
    5. 确保关键词密度合理
    关键词：{keywords}
  expected_output: >
    SEO 优化后的文章，附带 SEO 优化说明。
  agent: seo_specialist
  context:
    - editing_task
  output_file: output/final_article.md
```

### 3.6 YAML 配置 — Review Crew

```yaml
# app/config/review_agents.yaml
quality_reviewer:
  role: >
    质量审查员
  goal: >
    确保内容质量达到发布标准
  backstory: >
    你是内容质量的最后一道防线，对细节极其认真。
    你检查事实准确性、逻辑一致性和整体可读性。
  verbose: true
  allow_delegation: false

compliance_reviewer:
  role: >
    合规审查员
  goal: >
    确保内容符合法规和公司政策
  backstory: >
    你是一名合规专家，熟悉行业法规和公司内容政策。
    你检查是否存在法律风险、版权问题和政策违规。
  verbose: true
  allow_delegation: false
```

```yaml
# app/config/review_tasks.yaml
quality_task:
  description: >
    对文章进行质量审查。检查标准：
    1. 事实准确性：所有数据是否有可靠来源？
    2. 逻辑完整性：论证是否充分？
    3. 语言质量：是否清晰专业？
    4. 结构合理性：是否易于阅读？
    如果通过，回复 APPROVED 并附简要评价。
    如果不通过，列出具体问题。
  expected_output: >
    APPROVED + 评价，或问题列表。
  agent: quality_reviewer

compliance_task:
  description: >
    对文章进行合规审查。检查：
    1. 是否有法律/合规风险？
    2. 是否包含侵权内容？
    3. 是否符合行业规范？
    4. 是否有不当声明或承诺？
    如果通过，回复 COMPLIANT 并附简要说明。
    如果不通过，列出具体风险点。
  expected_output: >
    COMPLIANT + 说明，或风险列表。
  agent: compliance_reviewer
```

### 3.7 工具定义

```python
# app/tools/search_tools.py
from crewai_tools import tool

@tool("Web Search")
def web_search(query: str) -> str:
    """搜索网络获取最新信息"""
    return f"关于'{query}'的搜索发现..."

@tool("Fetch Webpage")
def fetch_webpage(url: str) -> str:
    """获取网页内容"""
    return f"URL {url} 的内容..."
```

```python
# app/tools/knowledge_tools.py
from crewai_tools import tool
import psycopg2
from pgvector.psycopg2 import register_vector
from openai import OpenAI

client = OpenAI()

def get_db_connection():
    conn = psycopg2.connect(
        host="localhost", port=5432,
        user="crewai", password="crewai",
        dbname="crewai",
    )
    register_vector(conn)
    return conn

@tool("Knowledge Search")
def knowledge_search(query: str) -> str:
    """从内部知识库搜索相关信息"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=query,
    )
    embedding = response.data[0].embedding

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        SELECT content, source, 1 - (embedding <=> %s::vector) as similarity
        FROM knowledge_docs
        ORDER BY similarity DESC
        LIMIT 3
    """, (embedding,))

    results = cur.fetchall()
    cur.close()
    conn.close()

    if not results:
        return "未找到相关信息"

    return "\n\n".join(
        f"[来源: {r[1]}] {r[0]}"
        for r in results
    )

@tool("Save to Knowledge")
def save_to_knowledge(content: str, source: str) -> str:
    """保存文档到知识库"""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=content,
    )
    embedding = response.data[0].embedding

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO knowledge_docs (content, embedding, source)
        VALUES (%s, %s, %s)
    """, (content, embedding, source))
    conn.commit()
    cur.close()
    conn.close()

    return f"已保存到知识库（来源：{source}）"
```

### 3.8 Crew 定义

```python
# app/crews/research_crew.py
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai.agents.agent_builder.base_agent import BaseAgent
from typing import List
from app.tools.search_tools import web_search, fetch_webpage
from app.tools.knowledge_tools import knowledge_search

@CrewBase
class ResearchCrew:
    """调研 Crew — Sequential 流程"""

    agents: List[BaseAgent]
    tasks: List[Task]

    agents_config = "app/config/research_agents.yaml"
    tasks_config = "app/config/research_tasks.yaml"

    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config["researcher"],
            tools=[web_search, fetch_webpage, knowledge_search],
        )

    @agent
    def analyst(self) -> Agent:
        return Agent(
            config=self.agents_config["analyst"],
        )

    @agent
    def validator(self) -> Agent:
        return Agent(
            config=self.agents_config["validator"],
        )

    @task
    def research_task(self) -> Task:
        return Task(config=self.tasks_config["research_task"])

    @task
    def analysis_task(self) -> Task:
        return Task(config=self.tasks_config["analysis_task"])

    @task
    def validation_task(self) -> Task:
        return Task(config=self.tasks_config["validation_task"])

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.sequential,
            verbose=True,
        )
```

```python
# app/crews/content_crew.py
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai.agents.agent_builder.base_agent import BaseAgent
from typing import List

@CrewBase
class ContentCrew:
    """内容 Crew — Hierarchical 流程"""

    agents: List[BaseAgent]
    tasks: List[Task]

    agents_config = "app/config/content_agents.yaml"
    tasks_config = "app/config/content_tasks.yaml"

    @agent
    def manager(self) -> Agent:
        return Agent(config=self.agents_config["manager"])

    @agent
    def writer(self) -> Agent:
        return Agent(
            config=self.agents_config["writer"],
            tools=[],
        )

    @agent
    def editor(self) -> Agent:
        return Agent(config=self.agents_config["editor"])

    @agent
    def seo_specialist(self) -> Agent:
        return Agent(config=self.agents_config["seo_specialist"])

    @task
    def writing_task(self) -> Task:
        return Task(config=self.tasks_config["writing_task"])

    @task
    def editing_task(self) -> Task:
        return Task(config=self.tasks_config["editing_task"])

    @task
    def seo_task(self) -> Task:
        return Task(config=self.tasks_config["seo_task"])

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.hierarchical,
            manager_agent=self.manager(),
            verbose=True,
        )
```

```python
# app/crews/review_crew.py
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai.agents.agent_builder.base_agent import BaseAgent
from typing import List

@CrewBase
class ReviewCrew:
    """审查 Crew — 并行流程"""

    agents: List[BaseAgent]
    tasks: List[Task]

    agents_config = "app/config/review_agents.yaml"
    tasks_config = "app/config/review_tasks.yaml"

    @agent
    def quality_reviewer(self) -> Agent:
        return Agent(config=self.agents_config["quality_reviewer"])

    @agent
    def compliance_reviewer(self) -> Agent:
        return Agent(config=self.agents_config["compliance_reviewer"])

    @task
    def quality_task(self) -> Task:
        return Task(config=self.tasks_config["quality_task"])

    @task
    def compliance_task(self) -> Task:
        return Task(config=self.tasks_config["compliance_task"])

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.sequential,
            verbose=True,
        )
```

### 3.9 Flow 定义

```python
# app/flows/content_flow.py
from crewai.flow import Flow, listen, start
from pydantic import BaseModel
from app.crews.research_crew import ResearchCrew
from app.crews.content_crew import ContentCrew
from app.crews.review_crew import ReviewCrew

class ContentFlowState(BaseModel):
    topic: str = ""
    keywords: str = ""
    research_data: str = ""
    article: str = ""
    quality_review: str = ""
    compliance_review: str = ""
    approved: bool = False
    iteration: int = 0

class ContentFlow(Flow[ContentFlowState]):
    """内容生产主 Flow"""

    max_iterations: int = 3

    @start()
    def research_phase(self):
        """阶段一：调研"""
        result = ResearchCrew().crew().kickoff(inputs={
            "topic": self.state.topic,
            "keywords": self.state.keywords,
        })
        self.state.research_data = result.raw
        print(f"[Flow] 调研完成，数据长度：{len(self.state.research_data)}")

    @listen(research_phase)
    def content_phase(self):
        """阶段二：内容创作"""
        result = ContentCrew().crew().kickoff(inputs={
            "topic": self.state.topic,
            "keywords": self.state.keywords,
        })
        self.state.article = result.raw
        print(f"[Flow] 文章完成，长度：{len(self.state.article)}")

    @listen(content_phase)
    def review_phase(self):
        """阶段三：审查"""
        quality_result = ReviewCrew().crew().kickoff(inputs={
            "article": self.state.article,
        })
        self.state.quality_review = quality_result.raw

        self.state.iteration += 1
        approved = "APPROVED" in self.state.quality_review
        compliant = "COMPLIANT" in self.state.compliance_review

        if approved and compliant:
            self.state.approved = True
            print("[Flow] 审查通过！")
        elif self.state.iteration >= self.max_iterations:
            self.state.approved = True
            print(f"[Flow] 达到最大迭代次数 {self.max_iterations}，强制通过")
        else:
            print(f"[Flow] 需要修改（第 {self.state.iteration} 次）")
            self.content_phase()  # 重新生成
```

### 3.10 API 路由

```python
# app/schemas/pipeline.py
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class PipelineCreate(BaseModel):
    topic: str
    keywords: str = ""

class PipelineResponse(BaseModel):
    id: int
    topic: str
    status: str
    article: str
    created_at: datetime
    updated_at: Optional[datetime] = None

class ApproveRequest(BaseModel):
    approved: bool
    feedback: str = ""

class PipelineListResponse(BaseModel):
    items: list[PipelineResponse]
    total: int
```

```python
# app/api/pipeline.py
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
import asyncio
from app.database import get_db
from app.models.pipeline import Pipeline, PipelineStatus
from app.models.user import User
from app.schemas.pipeline import PipelineCreate, PipelineResponse, ApproveRequest, PipelineListResponse
from app.middleware.auth import get_current_user
from app.flows.content_flow import ContentFlow, ContentFlowState

router = APIRouter(prefix="/api/v1/pipeline", tags=["pipeline"])

@router.post("", response_model=PipelineResponse)
async def create_pipeline(
    req: PipelineCreate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    pipeline = Pipeline(
        user_id=user.id,
        topic=req.topic,
        keywords=req.keywords,
        status=PipelineStatus.RESEARCHING,
    )
    db.add(pipeline)
    await db.commit()
    await db.refresh(pipeline)

    try:
        flow = ContentFlow()
        flow.state = ContentFlowState(
            topic=req.topic,
            keywords=req.keywords,
        )

        result = await asyncio.to_thread(flow.kickoff)

        pipeline.status = PipelineStatus.APPROVED if flow.state.approved else PipelineStatus.REVIEWING
        pipeline.article = flow.state.article
        pipeline.research_data = flow.state.research_data

    except Exception as e:
        pipeline.status = PipelineStatus.FAILED
        pipeline.error_message = str(e)

    await db.commit()
    await db.refresh(pipeline)
    return _pipeline_to_response(pipeline)

@router.get("/{pipeline_id}", response_model=PipelineResponse)
async def get_pipeline(
    pipeline_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Pipeline).where(Pipeline.id == pipeline_id, Pipeline.user_id == user.id)
    )
    pipeline = result.scalar_one_or_none()
    if not pipeline:
        raise HTTPException(status_code=404, detail="Pipeline 不存在")
    return _pipeline_to_response(pipeline)

@router.get("/{pipeline_id}/article", response_model=PipelineResponse)
async def get_article(
    pipeline_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Pipeline).where(Pipeline.id == pipeline_id, Pipeline.user_id == user.id)
    )
    pipeline = result.scalar_one_or_none()
    if not pipeline or not pipeline.article:
        raise HTTPException(status_code=404, detail="文章尚未生成")
    return _pipeline_to_response(pipeline)

@router.post("/{pipeline_id}/approve", response_model=PipelineResponse)
async def approve_pipeline(
    pipeline_id: int,
    req: ApproveRequest,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Pipeline).where(Pipeline.id == pipeline_id, Pipeline.user_id == user.id)
    )
    pipeline = result.scalar_one_or_none()
    if not pipeline:
        raise HTTPException(status_code=404, detail="Pipeline 不存在")

    pipeline.status = PipelineStatus.APPROVED if req.approved else PipelineStatus.REJECTED
    pipeline.review_feedback = req.feedback
    await db.commit()
    await db.refresh(pipeline)
    return _pipeline_to_response(pipeline)

@router.get("", response_model=PipelineListResponse)
async def list_pipelines(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size
    total = await db.scalar(
        select(func.count(Pipeline.id)).where(Pipeline.user_id == user.id)
    )
    result = await db.execute(
        select(Pipeline)
        .where(Pipeline.user_id == user.id)
        .order_by(Pipeline.created_at.desc())
        .offset(offset)
        .limit(page_size)
    )
    items = [_pipeline_to_response(r) for r in result.scalars().all()]
    return PipelineListResponse(items=items, total=total or 0)

def _pipeline_to_response(p: Pipeline) -> PipelineResponse:
    return PipelineResponse(
        id=p.id,
        topic=p.topic,
        status=p.status.value if hasattr(p.status, "value") else p.status,
        article=p.article or "",
        created_at=p.created_at,
        updated_at=p.updated_at,
    )
```

```python
# app/api/knowledge.py
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from app.middleware.auth import get_current_user
from app.models.user import User
from app.tools.knowledge_tools import knowledge_search, save_to_knowledge

router = APIRouter(prefix="/api/v1/knowledge", tags=["knowledge"])

class DocumentUpload(BaseModel):
    content: str
    source: str

class SearchRequest(BaseModel):
    query: str
    top_k: int = 3

class SearchResult(BaseModel):
    content: str
    source: str
    score: float

@router.post("/documents")
async def upload_document(
    req: DocumentUpload,
    user: User = Depends(get_current_user),
):
    result = save_to_knowledge(req.content, req.source)
    return {"message": result}

@router.post("/search")
async def search(
    req: SearchRequest,
    user: User = Depends(get_current_user),
):
    result = knowledge_search(req.query)
    return {"results": result}
```

### 3.11 FastAPI 入口

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.database import init_db
from app.api import auth, pipeline, knowledge

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield

app = FastAPI(title=settings.app_name, lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router)
app.include_router(pipeline.router)
app.include_router(knowledge.router)

@app.get("/health")
async def health():
    return {
        "status": "ok",
        "version": "1.0.0",
        "framework": "CrewAI",
        "crewai_version": "1.14.6",
    }
```

---

## 4 API 端点参考

### 4.1 认证

```bash
# 注册
curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "bob", "email": "bob@example.com", "password": "secret123"}'

# 登录并保存 Token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "bob", "password": "secret123"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token: $TOKEN"
```

### 4.2 Pipeline

```bash
# 启动内容生产 Pipeline
curl -X POST http://localhost:8000/api/v1/pipeline \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"topic": "2026年AI Agent在企业中的应用趋势", "keywords": "AI Agent, 企业自动化, 智能助手"}'

# 获取 Pipeline 状态
curl http://localhost:8000/api/v1/pipeline/1 \
  -H "Authorization: Bearer $TOKEN"

# 获取生成的文章
curl http://localhost:8000/api/v1/pipeline/1/article \
  -H "Authorization: Bearer $TOKEN"

# 审查通过
curl -X POST http://localhost:8000/api/v1/pipeline/1/approve \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"approved": true, "feedback": "内容完整，质量达标"}'

# 列表查询
curl "http://localhost:8000/api/v1/pipeline?page=1&page_size=10" \
  -H "Authorization: Bearer $TOKEN"
```

### 4.3 知识库

```bash
# 上传文档到知识库
curl -X POST http://localhost:8000/api/v1/knowledge/documents \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"content": "CrewAI 是一个独立于 LangChain 的 AI Agent 框架", "source": "官方文档"}'

# 搜索知识库
curl -X POST http://localhost:8000/api/v1/knowledge/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query": "CrewAI 特性", "top_k": 3}'
```

### 4.4 健康检查

```bash
curl http://localhost:8000/health
# {"status":"ok","version":"1.0.0","framework":"CrewAI","crewai_version":"1.14.6"}
```

---

## 5 业务模式库

### 5.1 软删除

```python
from sqlalchemy import Column, Boolean, DateTime
from sqlalchemy.sql import func

class SoftDeleteMixin:
    is_deleted = Column(Boolean, default=False)
    deleted_at = Column(DateTime, nullable=True)

    def soft_delete(self):
        self.is_deleted = True
        self.deleted_at = func.now()
```

### 5.2 批量操作

```python
@router.post("/pipeline/batch-delete")
async def batch_delete_pipelines(
    ids: list[int],
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Pipeline).where(
            Pipeline.id.in_(ids),
            Pipeline.user_id == user.id,
        )
    )
    for pipeline in result.scalars().all():
        await db.delete(pipeline)
    await db.commit()
    return {"deleted": len(ids)}
```

### 5.3 CSV 导出

```python
from fastapi.responses import StreamingResponse
import csv, io

@router.get("/pipeline/export")
async def export_pipelines(
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Pipeline).where(Pipeline.user_id == user.id)
    )
    pipelines = result.scalars().all()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "主题", "状态", "创建时间"])
    for p in pipelines:
        writer.writerow([p.id, p.topic, p.status.value, p.created_at])

    output.seek(0)
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=pipelines.csv"},
    )
```

### 5.4 审计日志

```python
from datetime import datetime, timezone

def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)

class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    action = Column(String(50), nullable=False)
    resource = Column(String(50), nullable=False)
    resource_id = Column(String(50))
    details = Column(Text)
    created_at = Column(DateTime, default=_utcnow)
```

### 5.5 速率限制

```python
from fastapi import HTTPException
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_requests: int = 30, window: int = 60):
        self.max_requests = max_requests
        self.window = window
        self.requests: dict[str, list[float]] = defaultdict(list)

    async def check(self, key: str):
        now = time.time()
        window_start = now - self.window
        self.requests[key] = [t for t in self.requests[key] if t > window_start]
        if len(self.requests[key]) >= self.max_requests:
            raise HTTPException(status_code=429, detail="请求过于频繁")
        self.requests[key].append(now)
```

### 5.6 缓存

```python
from functools import lru_cache
import time

class PipelineCache:
    def __init__(self, ttl: int = 300):
        self._cache: dict[str, tuple[float, str]] = {}
        self.ttl = ttl

    def get(self, key: str) -> str | None:
        if key in self._cache:
            ts, val = self._cache[key]
            if time.time() - ts < self.ttl:
                return val
            del self._cache[key]
        return None

    def set(self, key: str, value: str):
        self._cache[key] = (time.time(), value)

pipeline_cache = PipelineCache()
```

### 5.7 游标分页

```python
@router.get("/pipeline/cursor")
async def cursor_pagination(
    cursor: int | None = Query(None),
    limit: int = Query(20, le=100),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    query = select(Pipeline).where(Pipeline.user_id == user.id)
    if cursor:
        query = query.where(Pipeline.id < cursor)
    query = query.order_by(Pipeline.id.desc()).limit(limit + 1)

    result = await db.execute(query)
    items = result.scalars().all()
    has_more = len(items) > limit
    if has_more:
        items = items[:limit]

    return {
        "items": [_pipeline_to_response(r) for r in items],
        "next_cursor": items[-1].id if has_more else None,
        "has_more": has_more,
    }
```

### 5.8 异步 SSE 通知

```python
from fastapi.responses import StreamingResponse
import asyncio
import json

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[asyncio.Queue]] = {}

    def subscribe(self, topic: str) -> asyncio.Queue:
        if topic not in self._subscribers:
            self._subscribers[topic] = []
        q: asyncio.Queue = asyncio.Queue()
        self._subscribers[topic].append(q)
        return q

    async def publish(self, topic: str, message: dict):
        for q in self._subscribers.get(topic, []):
            await q.put(message)

event_bus = EventBus()

@router.get("/pipeline/{pipeline_id}/events")
async def stream_events(pipeline_id: int, user: User = Depends(get_current_user)):
    q = event_bus.subscribe(f"pipeline-{pipeline_id}")

    async def event_stream():
        try:
            while True:
                msg = await q.get()
                yield f"data: {json.dumps(msg)}\n\n"
        except asyncio.CancelledError:
            pass

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

---

## 6 构建与运行

### 6.1 环境配置

```bash
# 创建 .env
cat > .env << EOF
JWT_SECRET=change-this-in-production
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=sqlite+aiosqlite:///./pipeline.db
PGVECTOR_HOST=localhost
PGVECTOR_PORT=5432
PGVECTOR_USER=crewai
PGVECTOR_PASSWORD=crewai
PGVECTOR_DATABASE=crewai
EOF
```

### 6.2 安装和启动

```bash
# 确保使用 Python 3.13
uv python install 3.13
uv init --python 3.13

# 安装依赖
uv add "crewai>=1.14.0" "crewai-tools>=1.14.0"
uv add "fastapi" "uvicorn"
uv add "sqlalchemy[asyncio]" "aiosqlite"
uv add "python-jose[cryptography]" "passlib[bcrypt]" "bcrypt"
uv add "python-dotenv" "pydantic-settings"
uv add "psycopg2-binary" "pgvector" "openai"

# 初始化向量表
python3 -c "
import psycopg2
from pgvector.psycopg2 import register_vector
conn = psycopg2.connect(host='localhost', user='crewai', password='crewai', dbname='crewai')
conn.autocommit = True
cur = conn.cursor()
cur.execute('CREATE EXTENSION IF NOT EXISTS vector')
cur.execute('''
    CREATE TABLE IF NOT EXISTS knowledge_docs (
        id SERIAL PRIMARY KEY,
        content TEXT NOT NULL,
        embedding vector(1536),
        source TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    )
''')
cur.close()
conn.close()
print('向量表创建成功')
"

# 启动 PostgreSQL
docker run --name pgvector-db \
  -e POSTGRES_USER=crewai \
  -e POSTGRES_PASSWORD=crewai \
  -e POSTGRES_DB=crewai \
  -p 5432:5432 \
  -d pgvector/pgvector:pg16

# 启动应用
uv run uvicorn app.main:app --reload --port 8000
```

### 6.3 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.13-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
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
      - ./pipeline.db:/app/pipeline.db

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: crewai
      POSTGRES_PASSWORD: crewai
      POSTGRES_DB: crewai
    ports:
      - "5432:5432"
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

### 6.4 测试运行

```bash
# 健康检查
curl http://localhost:8000/health

# Pipeline 快速测试
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","email":"demo@test.com","password":"demo123"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -X POST http://localhost:8000/api/v1/pipeline \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"topic":"AI Agent 介绍","keywords":"AI, Agent, 自动化"}'
```

---

## 7 版本锁定

```ini
PYTHON_VERSION=3.13.x
CREWAI_VERSION=1.14.6+
CREWAI_TOOLS_VERSION=1.14.0+
FASTAPI_VERSION=0.115+
UVICORN_VERSION=0.30+
SQLALCHEMY_VERSION=2.0+
POSTGRESQL_VERSION=16
PGVECTOR_VERSION=0.8+
OPENAI_VERSION=1.0+
PSYCOPG2_VERSION=2.9+
```

```toml
# pyproject.toml
[project]
name = "content-pipeline"
version = "1.0.0"
requires-python = ">=3.10,<3.14"

dependencies = [
    "crewai>=1.14.0",
    "crewai-tools>=1.14.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "sqlalchemy[asyncio]>=2.0",
    "aiosqlite>=0.20.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.0",
    "python-dotenv>=1.0.0",
    "pydantic-settings>=2.10.0",
    "psycopg2-binary>=2.9.0",
    "pgvector>=0.2.0",
    "openai>=1.0.0",
]
```
