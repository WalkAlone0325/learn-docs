# LangGraph 项目实践与 Demo：多 Agent 研究助手

> **版本:** 1.0.0
> **运行时:** Python 3.14.5 · LangChain 1.3+ · LangGraph 1.2+ · PostgreSQL 16

---

## 1 项目概述

### 1.1 项目架构

本项目实现了一个多 Agent 研究助手系统，采用 Supervisor + 子 Agent 架构。用户提交研究课题，Supervisor Agent 协调多个专业 Agent 完成研究、分析和报告生成。

```
用户请求 → Supervisor Agent
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
Researcher  RAG Agent   Data Analyst
    │           │           │
    └───────────┼───────────┘
                ▼
         Writer Agent
                │
                ▼
         Reviewer Agent ← 人机交互断点
                │
                ▼
           最终报告
```

### 1.2 技术栈

| 层级 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 语言 | Python | 3.14.5 | 应用开发语言 |
| Agent 框架 | LangChain | 1.3+ | Agent 抽象 |
| 运行时 | LangGraph | 1.2+ | 状态图编排 |
| Web 框架 | FastAPI | 0.115+ | API 服务 |
| ASGI | Uvicorn | 0.30+ | HTTP 服务 |
| 向量存储 | langchain-postgres | 0.0.17+ | pgvector |
| 嵌入模型 | OpenAI | text-embedding-3-small | 向量化 |
| 数据库 | PostgreSQL | 16 | 主数据库 + 向量 |
| 检查点 | SQLite | - | 会话持久化 |
| 配置 | pydantic-settings | 2.10+ | 环境变量 |
| Auth | python-jose + passlib | - | JWT + 密码 |

### 1.3 API 端点总览

| 方法 | 路径 | 认证 | 说明 | 涉及概念 |
|------|------|------|------|---------|
| POST | `/api/v1/auth/register` | 无 | 用户注册 | JWT, 密码哈希 |
| POST | `/api/v1/auth/login` | 无 | 用户登录 | JWT |
| POST | `/api/v1/research` | JWT | 提交研究任务 | create_agent, Supervisor |
| GET | `/api/v1/research/{id}` | JWT | 获取研究状态 | State management |
| GET | `/api/v1/research/{id}/report` | JWT | 获取研究报告 | Structured output |
| POST | `/api/v1/research/{id}/approve` | JWT | 批准/驳回报告 | HumanInTheLoop |
| POST | `/api/v1/knowledge/documents` | JWT | 上传知识文档 | pgvector ingestion |
| GET | `/api/v1/knowledge/search` | JWT | 搜索知识库 | RAG, similarity search |
| DELETE | `/api/v1/knowledge/documents/{id}` | JWT | 删除文档 | 文档管理 |
| GET | `/api/v1/research` | JWT | 研究历史列表 | 分页查询 |
| DELETE | `/api/v1/research/{id}` | JWT | 删除研究 | 资源所有权 |
| GET | `/health` | 无 | 健康检查 | 健康检查 |

---

## 2 项目结构

```
agent-research/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI 入口
│   ├── config.py            # 配置管理
│   ├── database.py          # 数据库连接
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py          # 用户模型
│   │   └── research.py      # 研究任务模型
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── auth.py          # 认证 schema
│   │   ├── research.py      # 研究 schema
│   │   └── knowledge.py     # 知识库 schema
│   ├── api/
│   │   ├── __init__.py
│   │   ├── auth.py          # 认证路由
│   │   ├── research.py      # 研究路由
│   │   └── knowledge.py     # 知识库路由
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── team.py          # 多 Agent 团队
│   │   ├── researcher.py    # 研究员 Agent
│   │   ├── rag_agent.py     # RAG Agent
│   │   ├── writer.py        # 写手 Agent
│   │   └── reviewer.py      # 审查 Agent
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── auth.py          # JWT 认证中间件
│   │   └── logging.py       # 审计日志中间件
│   └── tools/
│       ├── __init__.py
│       ├── web_search.py    # 网络搜索工具
│       └── knowledge.py     # 知识库工具
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
    app_name: str = "Research Agent API"
    debug: bool = False

    # Auth
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 60

    # Database
    database_url: str = "sqlite+aiosqlite:///./research.db"

    # pgvector
    pgvector_host: str = "localhost"
    pgvector_port: int = 5432
    pgvector_user: str = "langchain"
    pgvector_password: str = "langchain"
    pgvector_database: str = "langchain"

    @property
    def pgvector_connection_string(self) -> str:
        return (
            f"postgresql+asyncpg://{self.pgvector_user}:{self.pgvector_password}"
            f"@{self.pgvector_host}:{self.pgvector_port}/{self.pgvector_database}"
        )

    # LLM
    openai_api_key: str = ""
    anthropic_api_key: str = ""
    default_model: str = "openai:gpt-4o-mini"

    # LangSmith
    langchain_tracing: bool = False
    langchain_api_key: str = ""

    model_config = {"env_file": ".env"}

settings = Settings()
```

### 3.2 数据库模型

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(settings.database_url, echo=settings.debug)
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
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

```python
# app/models/research.py
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Enum as SAEnum
from sqlalchemy.sql import func
import enum
from app.database import Base

class ResearchStatus(str, enum.Enum):
    PENDING = "pending"
    RESEARCHING = "researching"
    REVIEWING = "reviewing"
    APPROVED = "approved"
    REJECTED = "rejected"
    FAILED = "failed"

class Research(Base):
    __tablename__ = "research"

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False, index=True)
    title = Column(String(255), nullable=False)
    topic = Column(Text, nullable=False)
    status = Column(SAEnum(ResearchStatus), default=ResearchStatus.PENDING)
    report = Column(Text, default="")
    error_message = Column(Text, default="")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### 3.3 Auth 认证

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr

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

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
```

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
from app.database import get_db
from app.models.user import User
from app.schemas.auth import RegisterRequest, LoginRequest, TokenResponse, UserResponse
from app.middleware.auth import hash_password, verify_password, create_access_token, get_current_user

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])

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

@router.get("/me", response_model=UserResponse)
async def get_me(user: User = Depends(get_current_user)):
    return UserResponse(id=user.id, username=user.username, email=user.email)
```

### 3.4 工具定义

```python
# app/tools/web_search.py
from langchain_core.tools import tool
import httpx

@tool
def web_search(query: str) -> str:
    """搜索网络获取最新信息"""
    # 生产环境应接入真实搜索 API（如 SerpAPI, Bing Search）
    return f"关于'{query}'的搜索结果：\n1. 标题：{query} 最新发展...\n2. 标题：{query} 研究进展...\n3. 标题：{query} 应用案例..."

@tool
def fetch_webpage(url: str) -> str:
    """获取网页内容"""
    try:
        response = httpx.get(url, timeout=10)
        response.raise_for_status()
        return response.text[:5000]
    except Exception as e:
        return f"获取失败：{e}"
```

```python
# app/tools/knowledge.py
from langchain_core.tools import tool
from langchain_postgres import PGVectorStore
from langchain_openai import OpenAIEmbeddings
from app.config import settings

_embedding = OpenAIEmbeddings(model="text-embedding-3-small")

async def get_vector_store() -> PGVectorStore:
    from langchain_postgres import PGEngine
    engine = PGEngine.from_connection_string(settings.pgvector_connection_string)
    return await PGVectorStore.create(
        engine=engine,
        table_name="knowledge_docs",
        embedding_service=_embedding,
    )

@tool
async def search_knowledge(query: str, top_k: int = 3) -> str:
    """从内部知识库搜索相关信息"""
    store = await get_vector_store()
    docs = await store.asimilarity_search(query, k=top_k)
    if not docs:
        return "知识库未找到相关信息"
    return "\n\n".join(
        f"[{d.metadata.get('source', 'unknown')}] {d.page_content}"
        for d in docs
    )
```

### 3.5 Agent 定义

```python
# app/agents/researcher.py
from langchain.agents import create_agent
from app.tools.web_search import web_search, fetch_webpage
from app.config import settings

researcher_agent = create_agent(
    model=settings.default_model,
    tools=[web_search, fetch_webpage],
    system_prompt="""你是一个专业研究员。
    你的任务是：
    1. 使用搜索工具查找与课题相关的信息
    2. 深入阅读重要网页
    3. 整理研究笔记，包括关键发现和数据来源
    输出格式：结构化的研究笔记，包含主题概述、关键发现、信息来源。""",
)
```

```python
# app/agents/rag_agent.py
from langchain.agents import create_agent
from app.tools.knowledge import search_knowledge
from app.tools.web_search import web_search
from app.config import settings

rag_agent = create_agent(
    model=settings.default_model,
    tools=[search_knowledge, web_search],
    system_prompt="""你是一个知识检索专家。
    你的任务：
    1. 优先从内部知识库搜索信息
    2. 内部知识不足时，使用网络搜索补充
    3. 整合信息并提供来源引用
    输出格式：信息摘要，包含来源标记 [内部/网络]。""",
)
```

```python
# app/agents/writer.py
from langchain.agents import create_agent
from app.config import settings

writer_agent = create_agent(
    model=settings.default_model,
    tools=[],
    system_prompt="""你是一个专业报告撰写人。
    你的任务：
    1. 基于研究笔记和知识库信息撰写报告
    2. 报告结构：摘要、背景、研究发现、分析讨论、结论建议
    3. 使用专业但易懂的语言
    4. 标注信息来源引用
    输出格式：完整的 Markdown 格式研究报告。""",
)
```

```python
# app/agents/reviewer.py
from langchain.agents import create_agent
from app.config import settings

reviewer_agent = create_agent(
    model=settings.default_model,
    tools=[],
    system_prompt="""你是一个质量审查员。
    审查标准：
    1. 内容准确性 - 事实是否可靠
    2. 结构完整性 - 是否包含所有必要部分
    3. 逻辑连贯性 - 论证是否合理
    4. 引用规范性 - 来源是否标注
    5. 语言质量 - 是否清晰专业
    输出：如果通过，回复 APPROVED 并附简要评价。
    如果不通过，列出具体改进建议。""",
)
```

### 3.6 多 Agent 团队

```python
# app/agents/team.py
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.sqlite import SqliteSaver
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage, SystemMessage
from app.agents.researcher import researcher_agent
from app.agents.rag_agent import rag_agent
from app.agents.writer import writer_agent
from app.agents.reviewer import reviewer_agent
from app.config import settings
import json

class ResearchState(TypedDict):
    messages: Annotated[list, add_messages]
    topic: str
    research_notes: str
    knowledge_context: str
    report: str
    review: str
    iteration: int

checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

async def research_node(state: ResearchState) -> dict:
    result = await researcher_agent.ainvoke({
        "messages": [
            SystemMessage(f"研究课题：{state['topic']}"),
            HumanMessage(f"请对'{state['topic']}'进行深入研究"),
        ]
    })
    return {"research_notes": result["messages"][-1].content}

async def knowledge_node(state: ResearchState) -> dict:
    result = await rag_agent.ainvoke({
        "messages": [
            SystemMessage(f"检索与以下课题相关的内部知识：{state['topic']}"),
            HumanMessage(f"查找 '{state['topic']}' 的相关资料"),
        ]
    })
    return {"knowledge_context": result["messages"][-1].content}

async def write_node(state: ResearchState) -> dict:
    context = f"研究笔记：\n{state['research_notes']}\n\n知识库信息：\n{state['knowledge_context']}"
    result = await writer_agent.ainvoke({
        "messages": [
            SystemMessage(f"研究课题：{state['topic']}\n\n{context}"),
            HumanMessage("请撰写完整的研究报告"),
        ]
    })
    return {"report": result["messages"][-1].content}

async def review_node(state: ResearchState) -> dict:
    result = await reviewer_agent.ainvoke({
        "messages": [
            SystemMessage(f"研究报告：\n{state['report']}"),
            HumanMessage("请审查这份报告的质量"),
        ]
    })
    return {"review": result["messages"][-1].content, "iteration": state["iteration"] + 1}

def decide_after_review(state: ResearchState) -> str:
    if "APPROVED" in state["review"] or state["iteration"] >= 3:
        return "end"
    return "rewrite"

async def rewrite_node(state: ResearchState) -> dict:
    result = await writer_agent.ainvoke({
        "messages": [
            SystemMessage(f"原始研究报告：\n{state['report']}"),
            SystemMessage(f"审查意见：\n{state['review']}"),
            HumanMessage("请根据审查意见修改报告"),
        ]
    })
    return {"report": result["messages"][-1].content}

def build_research_team():
    builder = StateGraph(ResearchState)

    builder.add_node("research", research_node)
    builder.add_node("knowledge", knowledge_node)
    builder.add_node("write", write_node)
    builder.add_node("review", review_node)
    builder.add_node("rewrite", rewrite_node)

    builder.add_edge(START, "research")
    builder.add_edge("research", "knowledge")
    builder.add_edge("knowledge", "write")
    builder.add_edge("write", "review")
    builder.add_conditional_edges("review", decide_after_review, {
        "rewrite": "rewrite",
        "end": END,
    })
    builder.add_edge("rewrite", "review")

    return builder.compile(
        checkpointer=checkpointer,
        interrupt_before=["review"],
   )

research_team = build_research_team()
```

### 3.7 API 路由

```python
# app/schemas/research.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class ResearchCreate(BaseModel):
    title: str
    topic: str

class ResearchResponse(BaseModel):
    id: int
    title: str
    topic: str
    status: str
    report: str
    created_at: datetime
    updated_at: Optional[datetime] = None

class ResearchListResponse(BaseModel):
    items: list[ResearchResponse]
    total: int

class ApproveRequest(BaseModel):
    approved: bool
    feedback: str = ""
```

```python
# app/api/research.py
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.models.research import Research, ResearchStatus
from app.models.user import User
from app.schemas.research import ResearchCreate, ResearchResponse, ResearchListResponse, ApproveRequest
from app.middleware.auth import get_current_user
from app.agents.team import research_team

router = APIRouter(prefix="/api/v1/research", tags=["research"])

@router.post("", response_model=ResearchResponse)
async def create_research(
    req: ResearchCreate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    research = Research(
        user_id=user.id,
        title=req.title,
        topic=req.topic,
        status=ResearchStatus.RESEARCHING,
    )
    db.add(research)
    await db.commit()
    await db.refresh(research)

    thread_id = f"research-{research.id}"

    try:
        result = await research_team.ainvoke(
            {
                "topic": req.topic,
                "research_notes": "",
                "knowledge_context": "",
                "report": "",
                "review": "",
                "iteration": 0,
            },
            config={"configurable": {"thread_id": thread_id}},
        )

        if "interrupts" in result and result["interrupts"]:
            research.status = ResearchStatus.REVIEWING
            research.report = result.get("report", "")
        else:
            research.status = ResearchStatus.APPROVED
            research.report = result.get("report", "")

    except Exception as e:
        research.status = ResearchStatus.FAILED
        research.error_message = str(e)

    await db.commit()
    await db.refresh(research)
    return _research_to_response(research)

@router.get("/{research_id}", response_model=ResearchResponse)
async def get_research(
    research_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(Research.id == research_id, Research.user_id == user.id)
    )
    research = result.scalar_one_or_none()
    if not research:
        raise HTTPException(status_code=404, detail="研究任务不存在")
    return _research_to_response(research)

@router.get("/{research_id}/report", response_model=ResearchResponse)
async def get_report(
    research_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(Research.id == research_id, Research.user_id == user.id)
    )
    research = result.scalar_one_or_none()
    if not research:
        raise HTTPException(status_code=404, detail="研究任务不存在")
    if not research.report:
        raise HTTPException(status_code=404, detail="报告尚未生成")
    return _research_to_response(research)

@router.post("/{research_id}/approve", response_model=ResearchResponse)
async def approve_research(
    research_id: int,
    req: ApproveRequest,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(Research.id == research_id, Research.user_id == user.id)
    )
    research = result.scalar_one_or_none()
    if not research:
        raise HTTPException(status_code=404, detail="研究任务不存在")

    thread_id = f"research-{research.id}"

    if req.approved:
        research.status = ResearchStatus.APPROVED
        research.report = research.report + f"\n\n> 审查通过：{req.feedback}"
    else:
        # 驳回后重新修改
        research.status = ResearchStatus.RESEARCHING
        await research_team.ainvoke(
            None,
            config={
                "configurable": {"thread_id": thread_id},
                "context": {"feedback": req.feedback},
            },
        )

    await db.commit()
    await db.refresh(research)
    return _research_to_response(research)

@router.get("", response_model=ResearchListResponse)
async def list_research(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size
    total = await db.scalar(
        select(func.count(Research.id)).where(Research.user_id == user.id)
    )
    result = await db.execute(
        select(Research)
        .where(Research.user_id == user.id)
        .order_by(Research.created_at.desc())
        .offset(offset)
        .limit(page_size)
    )
    items = [_research_to_response(r) for r in result.scalars().all()]
    return ResearchListResponse(items=items, total=total or 0)

@router.delete("/{research_id}")
async def delete_research(
    research_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(Research.id == research_id, Research.user_id == user.id)
    )
    research = result.scalar_one_or_none()
    if not research:
        raise HTTPException(status_code=404, detail="研究任务不存在")
    await db.delete(research)
    await db.commit()
    return {"message": "已删除"}

def _research_to_response(r: Research) -> ResearchResponse:
    return ResearchResponse(
        id=r.id,
        title=r.title,
        topic=r.topic,
        status=r.status.value if hasattr(r.status, "value") else r.status,
        report=r.report or "",
        created_at=r.created_at,
        updated_at=r.updated_at,
    )
```

```python
# app/schemas/knowledge.py
from pydantic import BaseModel
from typing import Optional

class DocumentUpload(BaseModel):
    content: str
    source: str
    metadata: dict = {}

class DocumentResponse(BaseModel):
    id: str
    source: str
    content_preview: str

class SearchRequest(BaseModel):
    query: str
    top_k: int = 3

class SearchResult(BaseModel):
    content: str
    source: str
    score: float
```

```python
# app/api/knowledge.py
from fastapi import APIRouter, Depends, HTTPException
from langchain_postgres import PGEngine, PGVectorStore
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document
from app.config import settings
from app.models.user import User
from app.middleware.auth import get_current_user
from app.schemas.knowledge import DocumentUpload, SearchRequest, SearchResult

router = APIRouter(prefix="/api/v1/knowledge", tags=["knowledge"])

_embedding = OpenAIEmbeddings(model="text-embedding-3-small")
TABLE_NAME = "knowledge_docs"

async def get_store() -> PGVectorStore:
    engine = PGEngine.from_connection_string(settings.pgvector_connection_string)
    await engine.ainit_vectorstore_table(
        table_name=TABLE_NAME,
        vector_size=1536,
    )
    return await PGVectorStore.create(
        engine=engine,
        table_name=TABLE_NAME,
        embedding_service=_embedding,
    )

@router.post("/documents")
async def upload_document(
    req: DocumentUpload,
    user: User = Depends(get_current_user),
):
    store = await get_store()
    doc = Document(
        page_content=req.content,
        metadata={"source": req.source, "user_id": str(user.id), **req.metadata},
    )
    await store.aadd_documents([doc])
    return {"message": "文档已上传", "source": req.source}

@router.post("/search")
async def search_knowledge(
    req: SearchRequest,
    user: User = Depends(get_current_user),
):
    store = await get_store()
    docs = await store.asimilarity_search_with_score(
        req.query,
        k=req.top_k,
        filter={"user_id": str(user.id)},
    )
    return [
        SearchResult(content=d.page_content, source=d.metadata.get("source", ""), score=float(s))
        for d, s in docs
    ]

@router.delete("/documents/{document_id}")
async def delete_document(
    document_id: str,
    user: User = Depends(get_current_user),
):
    store = await get_store()
    await store.adelete([document_id])
    return {"message": "已删除"}
```

### 3.8 FastAPI 入口

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.database import init_db
from app.api import auth, research, knowledge

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
app.include_router(research.router)
app.include_router(knowledge.router)

@app.get("/health")
async def health():
    return {
        "status": "ok",
        "version": "1.0.0",
        "python": "3.14.5",
    }
```

---

## 4 API 端点参考

### 4.1 认证

```bash
# 注册
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "email": "alice@example.com", "password": "secret123"}'

# 登录
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "secret123"}'

# 获取当前用户
curl http://localhost:8000/api/v1/auth/me \
  -H "Authorization: Bearer <token>"
```

### 4.2 研究任务

```bash
# 提交研究任务
curl -X POST http://localhost:8000/api/v1/research \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"title": "AI Agent 最新趋势", "topic": "2026年AI Agent领域的最新研究进展和应用趋势"}'

# 获取任务状态
curl http://localhost:8000/api/v1/research/1 \
  -H "Authorization: Bearer <token>"

# 获取研究报告
curl http://localhost:8000/api/v1/research/1/report \
  -H "Authorization: Bearer <token>"

# 批准/驳回报告
curl -X POST http://localhost:8000/api/v1/research/1/approve \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"approved": true, "feedback": "内容完整，通过"}'

# 列出研究历史
curl "http://localhost:8000/api/v1/research?page=1&page_size=10" \
  -H "Authorization: Bearer <token>"

# 删除研究
curl -X DELETE http://localhost:8000/api/v1/research/1 \
  -H "Authorization: Bearer <token>"
```

### 4.3 知识库

```bash
# 上传文档
curl -X POST http://localhost:8000/api/v1/knowledge/documents \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"content": "LangChain v1 于2025年发布，引入Middleware系统", "source": "官方文档"}'

# 搜索知识库
curl -X POST http://localhost:8000/api/v1/knowledge/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"query": "LangChain v1 特性", "top_k": 3}'

# 删除文档
curl -X DELETE http://localhost:8000/api/v1/knowledge/documents/<id> \
  -H "Authorization: Bearer <token>"
```

### 4.4 健康检查

```bash
curl http://localhost:8000/health
```

---

## 5 业务模式库

### 5.1 软删除

```python
from sqlalchemy import Column, Boolean, DateTime

class SoftDeleteMixin:
    is_deleted = Column(Boolean, default=False)
    deleted_at = Column(DateTime, nullable=True)

    def soft_delete(self):
        self.is_deleted = True
        self.deleted_at = func.now()

# 查询时过滤
stmt = select(Model).where(Model.is_deleted == False)
```

### 5.2 批量操作

```python
@router.post("/research/batch-delete")
async def batch_delete(
    ids: list[int],
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(
            Research.id.in_(ids),
            Research.user_id == user.id,
        )
    )
    for research in result.scalars().all():
        await db.delete(research)
    await db.commit()
    return {"deleted": len(ids)}
```

### 5.3 CSV 数据导出

```python
from fastapi.responses import StreamingResponse
import csv
import io

@router.get("/research/export")
async def export_research(
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Research).where(Research.user_id == user.id)
    )
    researches = result.scalars().all()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "标题", "主题", "状态", "创建时间"])
    for r in researches:
        writer.writerow([r.id, r.title, r.topic, r.status, r.created_at])

    output.seek(0)
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=research.csv"},
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

# Middleware: 记录 API 调用
from fastapi import Request
import json

@app.middleware("http")
async def audit_middleware(request: Request, call_next):
    response = await call_next(request)
    if request.url.path.startswith("/api/"):
        audit = AuditLog(
            user_id=getattr(request.state, "user_id", 0),
            action=request.method,
            resource=request.url.path,
            details=json.dumps(dict(request.query_params)),
        )
        # 异步写入
    return response
```

### 5.5 速率限制

```python
from fastapi import HTTPException
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_requests: int = 60, window: int = 60):
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

rate_limiter = RateLimiter()

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    if request.url.path.startswith("/api/"):
        key = request.client.host
        await rate_limiter.check(key)
    return await call_next(request)
```

### 5.6 缓存策略

```python
from functools import lru_cache
from app.config import settings

class ResearchCache:
    def __init__(self):
        self._cache: dict[str, tuple[float, str]] = {}
        self.ttl = 300  # 5 分钟

    def get(self, key: str) -> str | None:
        if key in self._cache:
            timestamp, value = self._cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            del self._cache[key]
        return None

    def set(self, key: str, value: str):
        self._cache[key] = (time.time(), value)

research_cache = ResearchCache()
```

### 5.7 游标分页

```python
@router.get("/research/cursor")
async def cursor_pagination(
    cursor: int | None = Query(None),
    limit: int = Query(20, le=100),
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    query = select(Research).where(Research.user_id == user.id)
    if cursor:
        query = query.where(Research.id < cursor)
    query = query.order_by(Research.id.desc()).limit(limit + 1)

    result = await db.execute(query)
    items = result.scalars().all()

    has_more = len(items) > limit
    if has_more:
        items = items[:limit]

    next_cursor = items[-1].id if has_more else None

    return {
        "items": [_research_to_response(r) for r in items],
        "next_cursor": next_cursor,
        "has_more": has_more,
    }
```

### 5.8 异步通知

```python
import asyncio
from typing import Any

class NotificationManager:
    def __init__(self):
        self._subscribers: dict[str, list[asyncio.Queue]] = defaultdict(list)

    def subscribe(self, topic: str) -> asyncio.Queue:
        queue: asyncio.Queue = asyncio.Queue()
        self._subscribers[topic].append(queue)
        return queue

    async def publish(self, topic: str, message: Any):
        for queue in self._subscribers.get(topic, []):
            await queue.put(message)

    def unsubscribe(self, topic: str, queue: asyncio.Queue):
        self._subscribers[topic].remove(queue)

notifier = NotificationManager()

# SSE 端点
from fastapi.responses import StreamingResponse

@router.get("/research/{research_id}/stream")
async def stream_research(research_id: int, user: User = Depends(get_current_user)):
    queue = notifier.subscribe(f"research-{research_id}")

    async def event_stream():
        try:
            while True:
                message = await queue.get()
                yield f"data: {json.dumps(message)}\n\n"
        except asyncio.CancelledError:
            notifier.unsubscribe(f"research-{research_id}", queue)

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

---

## 6 构建与运行

### 6.1 环境配置

```bash
# 创建 .env
cat > .env << EOF
JWT_SECRET=your-secret-key-change-in-production
OPENAI_API_KEY=sk-...
DATABASE_URL=sqlite+aiosqlite:///./research.db
PGVECTOR_HOST=localhost
PGVECTOR_PORT=5432
PGVECTOR_USER=langchain
PGVECTOR_PASSWORD=langchain
PGVECTOR_DATABASE=langchain
EOF
```

### 6.2 安装与启动

```bash
# 安装依赖
uv add "langchain>=1.3.0" "langchain-core>=0.3.0" "langgraph>=1.2.0"
uv add "langchain-openai" "langchain-anthropic"
uv add "langchain-postgres>=0.0.17"
uv add "fastapi" "uvicorn" "sqlalchemy[asyncio]" "aiosqlite"
uv add "python-jose[cryptography]" "passlib[bcrypt]" "bcrypt"
uv add "httpx" "python-dotenv" "pydantic-settings"

# 启动 PostgreSQL（如未运行）
docker run --name pgvector-db \
  -e POSTGRES_USER=langchain \
  -e POSTGRES_PASSWORD=langchain \
  -e POSTGRES_DB=langchain \
  -p 5432:5432 \
  -d pgvector/pgvector:pg16

# 启动应用
uv run uvicorn app.main:app --reload --port 8000
```

### 6.3 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.14-slim

WORKDIR /app

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
  app:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./checkpoints.db:/app/checkpoints.db

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: langchain
      POSTGRES_PASSWORD: langchain
      POSTGRES_DB: langchain
    ports:
      - "5432:5432"
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

### 6.4 测试运行

```bash
# 健康检查
curl http://localhost:8000/health
# {"status":"ok","version":"1.0.0","python":"3.14.5"}

# 注册用户
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@test.com","password":"test123"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 提交研究任务
curl -X POST http://localhost:8000/api/v1/research \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"AI趋势","topic":"2026年AI Agent趋势"}'

# 查看状态
curl http://localhost:8000/api/v1/research/1 \
  -H "Authorization: Bearer $TOKEN"
```

---

## 7 版本锁定

```ini
PYTHON_VERSION=3.14.5
LANGCHAIN_VERSION=1.3+
LANGGRAPH_VERSION=1.2+
LANGCHAIN_CORE_VERSION=0.3+
LANGCHAIN_POSTGRES_VERSION=0.0.17+
LANGCHAIN_OPENAI_VERSION=0.3+
FASTAPI_VERSION=0.115+
UVICORN_VERSION=0.30+
SQLALCHEMY_VERSION=2.0+
POSTGRESQL_VERSION=16
PGVECTOR_VERSION=0.8+
```

```toml
# pyproject.toml
[project]
name = "agent-research"
version = "1.0.0"
requires-python = ">=3.10"

dependencies = [
    "langchain>=1.3.0",
    "langchain-core>=0.3.0",
    "langgraph>=1.2.0",
    "langchain-openai>=0.3.0",
    "langchain-postgres>=0.0.17",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "sqlalchemy[asyncio]>=2.0",
    "aiosqlite>=0.20.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.0",
    "python-dotenv>=1.0.0",
    "pydantic-settings>=2.10.0",
    "httpx>=0.28.0",
]
```
