# FastAPI 完整学习文档

> 目标：学完本教程能独立开发完整的后端项目

---

## 目录

1. [环境与工具链](#1-环境与工具链)
2. [FastAPI 基础](#2-fastapi-基础)
3. [Pydantic v2 数据验证](#3-pydantic-v2-数据验证)
4. [请求与响应](#4-请求与响应)
5. [依赖注入](#5-依赖注入)
6. [异常处理](#6-异常处理)
7. [中间件](#7-中间件)
8. [SQLAlchemy 2.0 数据库](#8-sqlalchemy-20-数据库)
9. [Alembic 数据库迁移](#9-alembic-数据库迁移)
10. [认证与授权（JWT）](#10-认证与授权jwt)
11. [文件上传](#11-文件上传)
12. [后台任务](#12-后台任务)
13. [CORS 与跨域](#13-cors-与跨域)
14. [日志配置](#14-日志配置)
15. [WebSocket](#15-websocket)
16. [测试](#16-测试)
17. [部署](#17-部署)
18. [最佳实践与项目结构](#18-最佳实践与项目结构)
19. [常见错误排查](#19-常见错误排查)
20. [参考链接](#20-参考链接)

---

## 1. 环境与工具链

### 1.1 Python 版本

使用 Python >= 3.12，支持最新的类型注解语法。

### 1.2 uv 包管理

```bash
# 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 初始化项目
uv init --app
uv add fastapi uvicorn sqlalchemy alembic pydantic-settings
uv add --dev ruff pytest httpx

# 运行
uv run uvicorn app.main:app --reload

# 安装依赖
uv sync

# 添加/移除依赖
uv add <package>
uv remove <package>
```

### 1.3 依赖速查表

| 依赖 | 用途 |
|---|---|
| `fastapi[standard]` | Web 框架 |
| `uvicorn[standard]` | ASGI 服务器 |
| `pydantic` | 数据验证（FastAPI 内置） |
| `pydantic-settings` | 配置管理 |
| `sqlalchemy[asyncio]` | 异步 ORM |
| `aiomysql` / `asyncpg` / `aiosqlite` | 数据库异步驱动 |
| `alembic` | 数据库迁移 |
| `python-jose[cryptography]` | JWT 令牌 |
| `passlib[bcrypt]` | 密码哈希 |
| `python-multipart` | 文件上传 |
| `httpx` | 异步 HTTP 客户端 |
| `ruff` | 代码检查/格式化 |
| `pytest` / `pytest-asyncio` | 测试 |

---

## 2. FastAPI 基础

### 2.1 核心概念：ASGI 与异步

在学习代码之前，先理解 FastAPI 的根基——ASGI。

**什么是 ASGI？**

ASGI（Asynchronous Server Gateway Interface）是 Python 的异步 Web 服务器接口标准。它解决了传统 WSGI（如 Flask/Django 用的）不能处理 WebSocket、长连接、并发请求的问题。

```
传统 WSGI:  请求 → 处理 → 返回（同步，一次一个）
ASGI:       请求 → 处理 → 返回（异步，可同时处理多个）

FastAPI  →  基于 ASGI  →  支持异步 →  高并发 + WebSocket
Flask    →  基于 WSGI  →  同步     →  简单但并发弱
```

**为什么用异步？**

```python
# 同步方式：2 个请求依次处理，共 4 秒
# 请求1：开始 → 查数据库(2秒) → 返回
# 请求2：                        开始 → 查数据库(2秒) → 返回

# 异步方式：2 个请求同时处理，共 2 秒
# 请求1：开始 → 查数据库(2秒，等待时不阻塞) → 返回
# 请求2：开始 → 查数据库(2秒，等待时不阻塞) → 返回
```

异步编程的核心思想：**等待 I/O（数据库查询、HTTP 请求、文件读写）时，CPU 不空闲等着，而是去处理其他请求**。这就是 FastAPI 能支撑高并发的根本原因。

**请求在 FastAPI 中的完整流程：**

```
客户端 → Uvicorn(ASGI服务器) → 中间件1 → 中间件2 → 路由匹配
→ 依赖注入解析 → 参数校验(Pydantic) → 路由函数执行
→ 响应模型转换 → 中间件2 → 中间件1 → 响应给客户端
```

### 2.2 应用实例

```python
from fastapi import FastAPI

app = FastAPI(title="My API", version="1.0.0")


@app.get("/")
async def root():
    return {"message": "Hello World"}
```

只有 `app` 实例是必须的，`title`/`version` 等参数用于生成 OpenAPI 文档。

`async def` 定义异步路由。如果不需要在函数内 `await` 异步操作，也可以用 `def`：

```python
@app.get("/sync")
def sync_route():
    # 纯计算、无 I/O 等待的操作，用 def 即可
    return {"message": "sync"}

@app.get("/async")
async def async_route():
    # 涉及数据库查询、HTTP 请求等 I/O 操作，用 async def
    result = await db_query()
    return {"message": "async"}
```

FastAPI 会自动在独立线程中运行 `def` 路由，不会阻塞主循环。

### 2.3 lifespan 生命周期

控制应用启动和关闭时的行为：

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator

from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # 启动时执行
    print("Starting up...")
    yield
    # 关闭时执行
    print("Shutting down...")


app = FastAPI(lifespan=lifespan)
```

常见用途：创建数据库连接池、加载模型、初始化缓存。

### 2.3 路由装饰器

```python
@app.get("/")
@app.post("/")
@app.put("/")
@app.delete("/")
@app.patch("/")
@app.options("/")
@app.head("/")
```

### 2.4 路径参数

```python
@app.get("/items/{item_id}")
async def get_item(item_id: int):  # 自动类型转换和验证
    return {"item_id": item_id}
```

路径参数按声明顺序匹配，类型注解让 FastAPI 自动做类型转换和验证。如果传入非数字，返回 422。

### 2.5 查询参数

```python
@app.get("/items/")
async def list_items(
    skip: int = 0,
    limit: int = 10,
    q: str | None = None,  # 可选参数
):
    return {"skip": skip, "limit": limit, "q": q}
```

未给默认值的参数是必填的，有默认值的参数是可选的。

### 2.6 路径参数和查询参数混用

```python
@app.get("/users/{user_id}/items/")
async def get_user_items(
    user_id: int,
    skip: int = 0,
    limit: int = 10,
):
    pass
```

---

## 3. Pydantic v2 数据验证

### 3.1 基础模型

```python
from pydantic import BaseModel


class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = False  # 有默认值，可选
    description: str | None = None  # 可为空
```

### 3.2 字段验证

```python
from pydantic import BaseModel, Field


class Item(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0, le=10000)  # greater than, less or equal
    quantity: int = Field(ge=0, default=0)  # greater or equal
    email: str = Field(pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$")
```

### 3.3 自定义验证器

```python
from pydantic import BaseModel, field_validator


class Item(BaseModel):
    name: str
    password: str

    @field_validator("name")
    @classmethod
    def name_must_be_meaningful(cls, v: str) -> str:
        if len(v) < 2:
            raise ValueError("name too short")
        return v.strip()

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("password must be at least 8 characters")
        return v
```

### 3.4 模型配置

```python
from pydantic import BaseModel
from datetime import datetime


class UserResponse(BaseModel):
    id: int
    username: str
    created_at: datetime

    model_config = {"from_attributes": True}
    # 允许从 ORM 模型创建：UserResponse.model_validate(db_user)
```

### 3.5 序列化与反序列化

```python
data = {"name": "foo", "price": 9.99}
item = Item(**data)       # 字典 → Pydantic
item.model_dump()          # Pydantic → 字典
item.model_dump_json()     # Pydantic → JSON 字符串
item.model_dump(exclude_unset=True)  # 只返回显式设置的字段
```

### 3.6 嵌套模型

```python
class Image(BaseModel):
    url: str
    name: str


class Item(BaseModel):
    name: str
    image: Image | None = None
    tags: list[str] = []
```

### 3.7 Pydantic v2 进阶

```python
from pydantic import BaseModel, field_validator, model_validator, field_serializer, computed_field
from datetime import datetime


# ── 模型级验证器 ──────────────────────────────────

class Registration(BaseModel):
    username: str
    password: str
    confirm_password: str

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.confirm_password:
            raise ValueError("passwords do not match")
        return self


# ── 计算字段（不存数据，由其他字段运算得出） ──────

class Rectangle(BaseModel):
    width: float
    height: float

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height

    @computed_field
    @property
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)


# ── 自定义序列化器 ────────────────────────────────

class User(BaseModel):
    name: str
    joined_at: datetime

    @field_serializer("joined_at")
    def serialize_datetime(self, value: datetime) -> str:
        return value.strftime("%Y-%m-%d %H:%M:%S")


# ── 联合模型（同一接口返回多类型） ────────────────

from typing import Union
from pydantic import BaseModel


class Cat(BaseModel):
    pet_type: str = "cat"
    meow_volume: int


class Dog(BaseModel):
    pet_type: str = "dog"
    bark_pitch: float


Pet = Union[Cat, Dog]


@app.get("/pets/{pet_id}", response_model=Pet)
async def get_pet(pet_id: int):
    # FastAPI 根据响应数据自动判断返回 Cat 还是 Dog
    ...


# ── 严格模式（禁止额外字段） ──────────────────────

class StrictItem(BaseModel):
    name: str

    model_config = {"extra": "forbid"}
    # 请求体中多传字段 → 422 错误
```

---

## 4. 请求与响应

### 4.1 请求体

```python
from pydantic import BaseModel


class ItemCreate(BaseModel):
    name: str
    price: float


@app.post("/items/")
async def create_item(item: ItemCreate):  # FastAPI 自动从 JSON 解析
    return item
```

### 4.2 请求体 + 路径参数 + 查询参数

```python
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,                    # 路径参数
    item: ItemCreate,                # 请求体（JSON）
    q: str | None = None,            # 查询参数
):
    return {"item_id": item_id, **item.model_dump(), "q": q}
```

### 4.3 表单数据

```python
from fastapi import Form


@app.post("/login/")
async def login(
    username: str = Form(),
    password: str = Form(),
):
    return {"username": username}
```

需要安装 `python-multipart`。

### 4.4 响应模型

```python
from pydantic import BaseModel
from datetime import datetime


class ItemResponse(BaseModel):
    id: int
    name: str
    price: float
    created_at: datetime

    model_config = {"from_attributes": True}


@app.post("/items/", response_model=ItemResponse, status_code=201)
async def create_item(item: ItemCreate):
    # 返回的 dict/ORM 对象会被自动转换为 ItemResponse
    return db_item


@app.get("/items/", response_model=list[ItemResponse])
async def list_items():
    return db_items
```

`response_model` 的作用：
- 限制输出字段（过滤敏感信息）
- 自动序列化（datetime → ISO 字符串）
- 生成正确的 OpenAPI 文档

### 4.5 OpenAPI 文档自定义

```python
from fastapi import APIRouter
from pydantic import BaseModel


class ItemResponse(BaseModel):
    id: int
    name: str


router = APIRouter(prefix="/items", tags=["items"])


@router.get(
    "/",
    summary="列出所有物品",
    description="支持分页和关键词搜索的物品列表接口",
    response_description="物品列表",
    deprecated=False,
)
async def list_items():
    """函数文档字符串也会被 FastAPI 提取到 OpenAPI 文档"""
    pass


@router.get(
    "/{item_id}",
    response_model=ItemResponse,
    responses={
        404: {"description": "物品不存在"},
        403: {"description": "无权限访问"},
    },
)
async def get_item(item_id: int):
    pass
```

参数说明：
- `summary` — 接口短名称
- `description` — 接口详细说明
- `response_description` — 响应说明
- `deprecated` — 标记为已弃用
- `responses` — 自定义错误响应文档
- 函数 docstring 也会被自动提取

### 4.6 响应类型

```python
from fastapi.responses import (
    JSONResponse,
    HTMLResponse,
    PlainTextResponse,
    RedirectResponse,
    FileResponse,
    StreamingResponse,
)
import json


@app.get("/json")
async def json_response():
    return JSONResponse(content={"message": "ok"})


@app.get("/html")
async def html_response():
    return HTMLResponse(content="<h1>Hello</h1>")


@app.get("/redirect")
async def redirect():
    return RedirectResponse(url="/login")


@app.get("/file")
async def file_response():
    return FileResponse(path="static/photo.jpg", media_type="image/jpeg")


@app.get("/stream")
async def stream_response():
    async def generate():
        for i in range(10):
            yield f"data: {i}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

默认情况下 FastAPI 会自动将 dict/list/Pydantic 转为 JSONResponse。其他响应类型用于特定场景。

### 4.8 状态码

```python
from fastapi import status


@app.post("/items/", status_code=status.HTTP_201_CREATED)
@app.delete("/items/{id}", status_code=status.HTTP_204_NO_CONTENT)
```

### 4.9 Header 和 Cookie

```python
from fastapi import Header, Cookie


@app.get("/")
async def read_root(
    user_agent: str | None = Header(default=None),
    session_id: str | None = Cookie(default=None),
):
    return {"user_agent": user_agent}
```

---

## 5. 依赖注入

### 5.1 为什么需要依赖注入？

依赖注入的目的是**解耦**和**复用**。看一个反面例子：

```python
# ❌ 不好的做法：每个路由自己创建数据库连接
@app.get("/items/")
async def list_items():
    db = create_connection()   # 重复代码
    items = db.query(...)
    db.close()
    return items

@app.get("/users/")
async def list_users():
    db = create_connection()   # 重复代码
    users = db.query(...)
    db.close()
    return users

# ✅ 依赖注入：由 FastAPI 统一管理
@app.get("/items/")
async def list_items(db: Session = Depends(get_db)):  # 注入
    return db.query(...)

@app.get("/users/")
async def list_users(db: Session = Depends(get_db)):   # 复用
    return db.query(...)
```

依赖注入的好处：
- **复用**：相同的逻辑（数据库连接、认证、分页）只写一次
- **测试**：注入 mock 对象，无需修改路由代码
- **可替换**：修改数据库驱动时只需改依赖函数，不用改所有路由
- **生命周期管理**：FastAPI 自动处理资源的创建和释放（如关闭数据库连接）

### 5.2 基础用法

### 5.3 类作为依赖

```python
from fastapi import Depends
from pydantic import BaseModel


class Pagination:
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit


@app.get("/items/")
async def list_items(pagination: Pagination = Depends()):
    return {"skip": pagination.skip, "limit": pagination.limit}
```

当依赖是类时，`Depends()` 会自动实例化。

### 5.4 可调用对象作为依赖

```python
from fastapi import Depends


class DatabaseSession:
    def __call__(self):
        return get_db_session()


db_session = DatabaseSession()


@app.get("/items/")
async def list_items(db=Depends(db_session)):
    pass
```

### 5.5 依赖链

```python
async def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


async def get_current_user(db=Depends(get_db)):
    user = db.query(User).first()
    return user


@app.get("/users/me")
async def read_current_user(current_user=Depends(get_current_user)):
    return current_user
```

依赖可以嵌套依赖，FastAPI 会自动解析整个依赖树。

### 5.6 全局依赖

```python
app = FastAPI(dependencies=[Depends(verify_token)])
```

所有路由都会先执行 `verify_token`。

### 5.7 路由级依赖（APIRouter）

```python
from fastapi import APIRouter, Depends


# 该路由器下所有路由都会执行 get_current_user
router = APIRouter(
    prefix="/api/v1/admin",
    tags=["admin"],
    dependencies=[Depends(get_current_user)],  # 路由级依赖
)


# 子路由可以额外叠加依赖
@router.get("/dashboard", dependencies=[Depends(require_admin)])
async def dashboard():
    return {"message": "Admin dashboard"}


# 不同路由器的依赖互不影响
public_router = APIRouter(prefix="/api/v1/public")  # 无认证
admin_router = APIRouter(prefix="/api/v1/admin", dependencies=[Depends(verify_admin)])  # 需认证
```

执行顺序：`全局依赖 → 路由级依赖 → 路由函数依赖 → 路由函数`。

---

## 6. 异常处理

### 6.1 HTTPException

```python
from fastapi import HTTPException


@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id < 1:
        raise HTTPException(
            status_code=400,
            detail="Invalid item ID",
        )
    item = find_item(item_id)
    if not item:
        raise HTTPException(
            status_code=404,
            detail="Item not found",
        )
    return item
```

### 6.2 自定义异常

```python
from fastapi import HTTPException


class CredentialsException(HTTPException):
    def __init__(self):
        super().__init__(
            status_code=401,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )


# 使用
raise CredentialsException()
```

### 6.3 全局异常处理器

```python
from fastapi import Request
from fastapi.responses import JSONResponse


@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={"message": str(exc)},
    )


@app.exception_handler(Exception)
async def general_error_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"message": "Internal server error"},
    )
```

---

## 7. 中间件

### 7.1 基础中间件

```python
from fastapi import Request
import time


@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.perf_counter()
    response = await call_next(request)
    process_time = time.perf_counter() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

中间件接收请求 → 执行路由处理 → 接收响应 → 返回响应。

### 7.2 CORS 中间件

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 7.3 中间件执行顺序

中间件按添加顺序包裹路由，执行时是从外到内再到外：

```
Middleware1 (request) → Middleware2 (request) → Route → Middleware2 (response) → Middleware1 (response)
```

### 7.4 常用内置中间件

```python
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
import uuid


# ── Gzip 压缩 ────────────────────────────────────────
# 压缩响应体，减少传输大小
app.add_middleware(GZipMiddleware, minimum_size=1000)  # 大于 1KB 的响应才压缩


# ── TrustedHost 安全限制 ─────────────────────────────
# 只允许指定 Host 的请求访问，防止 Host 头攻击
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "*.example.com"],
)


# ── 请求 ID 追踪 ─────────────────────────────────────
# 每个请求分配唯一 ID，方便串联日志和排查问题
@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

---

## 8. SQLAlchemy 2.0 数据库

### 8.1 配置引擎和会话

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "mysql+aiomysql://root:root@localhost:3306/app_db"

engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    pool_size=10,           # 连接池大小
    max_overflow=20,        # 超出 pool_size 后允许的最大连接数
    pool_pre_ping=True,     # 每次取连接前检查是否有效（避免使用已断开的连接）
    pool_recycle=3600,      # 连接最大存活时间（秒），MySQL 默认 8 小时断开空闲连接
)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db():
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### 8.2 定义模型

```python
from sqlalchemy import String, Integer, Boolean, Text, Float, ForeignKey, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime, timezone


def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    email: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(default=_utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=_utcnow, onupdate=_utcnow)

    items: Mapped[list["Item"]] = relationship(back_populates="owner")


class Item(Base):
    __tablename__ = "items"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(100), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    price: Mapped[float] = mapped_column(Float, default=0)
    is_done: Mapped[bool] = mapped_column(Boolean, default=False)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(default=_utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=_utcnow, onupdate=_utcnow)

    owner: Mapped["User"] = relationship(back_populates="items")
```

### 8.3 字段类型速查

| Python 类型 | mapped_column 参数 | 数据库类型 |
|---|---|---|
| `int` | 默认 | INTEGER |
| `str` | 默认 | VARCHAR |
| `str` | `String(100)` | VARCHAR(100) |
| `str` | `Text` | TEXT |
| `float` | 默认 | FLOAT |
| `bool` | `Boolean` | BOOLEAN/TINYINT |
| `datetime` | `DateTime` 或默认 | DATETIME |
| `bytes` | `LargeBinary` | BLOB |
| `int` | `ForeignKey("t.id")` | 外键 |

### 8.4 数据库索引

合理使用索引大幅提升查询性能：

```python
from sqlalchemy import Index, UniqueConstraint


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)  # 单列索引 + 唯一
    email: Mapped[str] = mapped_column(String(100), unique=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=False)

    __table_args__ = (
        Index("idx_user_active_created", "is_active", "created_at"),     # 联合索引
        UniqueConstraint("username", "email", name="uq_username_email"), # 联合唯一
    )


# 常见索引策略：
#
# 经常查询的字段       → index=True
# 经常排序的字段       → index=True
# 外键字段             → 自动创建索引或手动加 index=True
# 唯一约束字段         → unique=True（自动创建索引）
# 组合查询条件         → 联合索引 Index("name", "col1", "col2")
# 文本搜索             → 使用数据库全文索引，而非 LIKE
#
# 避免：
# 每个字段都加索引（写入变慢、占用空间）
# 在低选择度字段上建索引（如 boolean 字段单独建索引意义不大）

# Alembic 迁移文件中也需添加：
# def upgrade():
#     op.create_index("idx_user_active_created", "users", ["is_active", "created_at"])
```

### 8.5 增删改查

```python
from sqlalchemy import select, delete, func


# 查询列表
result = await db.execute(select(Item).where(Item.is_done == False).order_by(Item.id))
items = result.scalars().all()

# 分页
result = await db.execute(
    select(Item).offset(skip).limit(limit).order_by(Item.id)
)
items = result.scalars().all()

# 单条查询
item = await db.get(Item, item_id)

# 条件查询
result = await db.execute(
    select(Item).where(Item.title.ilike(f"%{keyword}%"))
)
items = result.scalars().all()

# 统计
result = await db.execute(select(func.count()).select_from(Item))
total = result.scalar()

# 创建
item = Item(title="foo", owner_id=1)
db.add(item)
await db.flush()
await db.refresh(item)  # 获取自增 id

# 更新
item = await db.get(Item, item_id)
item.title = "new title"
# 自动在 get_db 的 commit 时保存

# 删除
item = await db.get(Item, item_id)
await db.delete(item)

# 关联查询
result = await db.execute(
    select(Item).where(Item.owner.has(User.username == "admin"))
)
```

### 8.6 分页查询模式

```python
from pydantic import BaseModel
from sqlalchemy import select, func


class PaginationParams:
    def __init__(self, skip: int = 0, limit: int = 20):
        self.skip = skip
        self.limit = min(limit, 100)  # 防止恶意大查询


class Page(BaseModel):
    items: list
    total: int
    skip: int
    limit: int


async def paginate(
    db: AsyncSession,
    query,
    pagination: PaginationParams,
) -> Page:
    # 统计总数
    count_query = select(func.count()).select_from(query.subquery())
    total_result = await db.execute(count_query)
    total = total_result.scalar()

    # 分页查询
    result = await db.execute(
        query.offset(pagination.skip).limit(pagination.limit)
    )
    items = result.scalars().all()

    return Page(items=items, total=total, skip=pagination.skip, limit=pagination.limit)


# 使用
@app.get("/items/", response_model=Page)
async def list_items(
    pagination: PaginationParams = Depends(),
    db: AsyncSession = Depends(get_db),
):
    query = select(Item).order_by(Item.id)
    return await paginate(db, query, pagination)
```

### 8.7 同步方式（SQLite 不需要外部数据库）

```python
# 适用于开发阶段
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("sqlite:///./app.db")
SessionLocal = sessionmaker(engine)


def get_db():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```

### 8.8 事务控制

```python
# get_db 已经自动 commit/rollback
# 但可以手动控制：

async def create_order(user_id: int, item_ids: list[int], db: AsyncSession):
    order = Order(user_id=user_id)
    db.add(order)
    await db.flush()

    for item_id in item_ids:
        item = await db.get(Item, item_id)
        if not item:
            await db.rollback()
            raise HTTPException(404, f"Item {item_id} not found")
        order.items.append(item)

    # get_db 会自动 commit
```

### 8.9 N+1 查询与关系加载策略

N+1 问题是新手最容易踩的性能陷阱：

```python
# ── 错误：N+1 查询 ─────────────────────────────────

# 假设有 100 篇文章，每篇需要查作者
posts = (await db.execute(select(Post))).scalars().all()
for post in posts:
    print(post.author.name)     # 每次循环都发一条 SQL！共 1+100 条


# ── 正确：预先加载关系 ─────────────────────────────

from sqlalchemy.orm import selectinload, joinedload


# 方式一：selectinload（发额外 SELECT 一次性加载所有关联）
query = select(Post).options(selectinload(Post.author))
posts = (await db.execute(query)).scalars().all()
for post in posts:
    print(post.author.name)     # 仅 2 条 SQL


# 方式二：joinedload（JOIN 在同一 SQL 中加载） 
query = select(Post).options(joinedload(Post.author))
posts = (await db.execute(query)).scalars().all()


# ── 加载多级关系 ───────────────────────────────────

from sqlalchemy.orm import selectinload

# 文章 + 作者 + 作者的所有文章（嵌套）
query = select(Post).options(
    selectinload(Post.author).selectinload(User.posts)
)


# ── 只加载部分字段 ─────────────────────────────────

from sqlalchemy.orm import load_only

query = select(User).options(load_only(User.id, User.username))


# ── 关系加载策略速查 ───────────────────────────────

# selectinload()   → 发额外 SELECT ... WHERE id IN (...)，适用于 to-many
# joinedload()     → LEFT JOIN 在同一 SQL 中加载，适用于 to-one
# subqueryload()   → 子查询方式，不常用
# lazy=True        → 默认，按需加载（导致 N+1）
# lazy="selectin"  → 自动使用 selectinload
# lazy="joined"    → 自动使用 joinedload
# lazy="subquery"  → 自动使用 subqueryload
#
# 建议：在 query 时用 options() 显式指定，而不是修改模型默认 lazy
```

### 8.10 批量插入与更新

```python
from sqlalchemy import insert, update


# ── 批量插入（比逐条 insert 快 10-100 倍） ──────────

# 方式一：逐条 add（适合少量数据，< 100 条）
for i in range(10):
    db.add(Item(title=f"item {i}", owner_id=1))

# 方式二：bulk_insert_mappings（适合大量数据，100-10000 条）
await db.execute(
    insert(Item),
    [
        {"title": f"item {i}", "owner_id": 1}
        for i in range(1000)
    ]
)
# 注意：bulk insert 不会触发 ORM 事件，也不会自动设置 created_at

# 方式三：批量更新
await db.execute(
    update(Item)
    .where(Item.owner_id == 1)
    .values(is_done=True)
)


# ── upsert（存在则更新，不存在则插入，MySQL 语法）──

from sqlalchemy.dialects.mysql import insert as mysql_insert

stmt = mysql_insert(Item).values(
    id=1, title="upserted", owner_id=1
)
stmt = stmt.on_duplicate_key_update(title=stmt.inserted.title)
await db.execute(stmt)
```

### 8.11 并发控制

```python
from sqlalchemy import select


# ── 问题场景 ────────────────────────────────────────
# 用户 A 和 B 同时读取库存 = 1，各自下单，最终库存变成 -1

# ── 方案一：悲观锁（适合写多读少） ──────────────────
# 查询时锁定该行，其他事务必须等待
result = await db.execute(
    select(Item).where(Item.id == item_id).with_for_update()
)
item = result.scalar_one_or_none()
# 在当前事务提交前，其他事务无法修改这条记录
if item.stock < quantity:
    raise HTTPException(400, "Insufficient stock")
item.stock -= quantity
# 事务提交后解锁


# ── 方案二：乐观锁（适合读多写少） ──────────────────
# 在模型中加版本号字段，更新时检查版本号

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    stock: Mapped[int]
    version: Mapped[int] = mapped_column(default=1)  # 版本号


# 更新时验证版本号
result = await db.execute(
    update(Product)
    .where(
        Product.id == product_id,
        Product.version == current_version,  # 版本号匹配才更新
    )
    .values(stock=new_stock, version=current_version + 1)
)
if result.rowcount == 0:
    # 版本号不匹配，说明被其他事务修改了
    raise HTTPException(409, "Conflict: product was modified by another user")


# ── 选择策略 ─────────────────────────────────────────
#
# 悲观锁 with_for_update()：
#   - 适合写冲突频繁的场景
#   - 会降低并发性能（事务排队）
#
# 乐观锁 version 字段：
#   - 适合写冲突少的场景
#   - 不阻塞读操作，性能好
#   - 冲突时需重试（客户端重新读取并重试）
```

---

## 9. Alembic 数据库迁移

### 9.1 初始化

```bash
# 异步项目
alembic init -t async alembic

# 同步项目
alembic init alembic
```

### 9.2 配置

修改 `alembic.ini`：

```ini
# 注释掉硬编码的 URL
# sqlalchemy.url = driver://user:pass@localhost/dbname
```

修改 `alembic/env.py`：

```python
from app.config import get_settings
from app.database import Base
from app.models import User, Item  # 导入所有模型

config = context.config
settings = get_settings()
config.set_main_option("sqlalchemy.url", settings.database_url)
target_metadata = Base.metadata
```

### 9.3 常用命令

```bash
# 自动生成迁移文件
alembic revision --autogenerate -m "描述变更"

# 查看状态
alembic history
alembic current

# 执行迁移
alembic upgrade head      # 升级到最新
alembic upgrade +2        # 升级两步
alembic downgrade -1      # 回退一步
alembic downgrade <hash>  # 回退到指定版本

# 手动创建空迁移
alembic revision -m "description"
```

在空迁移文件中编写 SQL：

```python
def upgrade():
    op.create_table(
        "my_table",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("name", sa.String(50), nullable=False),
    )


def downgrade():
    op.drop_table("my_table")
```

---

## 10. 认证与授权（JWT）

### 10.1 密码哈希

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

### 10.2 JWT 令牌

```python
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_token(token: str) -> dict | None:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None
```

### 10.3 登录接口

```python
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel

router = APIRouter(prefix="/auth", tags=["auth"])


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


@router.post("/login", response_model=TokenResponse)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    # 查询用户
    result = await db.execute(
        select(User).where(User.username == form_data.username)
    )
    user = result.scalar_one_or_none()

    # 验证密码
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # 生成令牌
    token = create_access_token({"sub": str(user.id)})
    return TokenResponse(access_token=token)
```

### 10.4 认证依赖

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
):
    token = credentials.credentials
    payload = decode_token(token)
    if payload is None or payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="Invalid token")

    user_id = int(payload.get("sub"))
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

### 10.5 保护路由

```python
@app.get("/users/me")
async def read_current_user(current_user: User = Depends(get_current_user)):
    return current_user


@app.get("/admin/dashboard")
async def admin_dashboard(current_user: User = Depends(get_current_user)):
    if not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Admin only")
    return {"message": "Welcome admin"}
```

### 10.6 刷新令牌

```python
REFRESH_TOKEN_EXPIRE_DAYS = 7


def create_refresh_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


@router.post("/refresh")
async def refresh_token(
    token_data: RefreshTokenRequest,
):
    payload = decode_token(token_data.refresh_token)
    if payload is None or payload.get("type") != "refresh":
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    user_id = int(payload.get("sub"))
    new_access = create_access_token({"sub": str(user_id)})
    return TokenResponse(access_token=new_access)
```

---

## 11. 文件上传

```python
from fastapi import APIRouter, UploadFile, File
import os

router = APIRouter(prefix="/upload", tags=["upload"])

UPLOAD_DIR = "uploads"
os.makedirs(UPLOAD_DIR, exist_ok=True)


@router.post("/")
async def upload_file(file: UploadFile = File()):
    # 读取文件内容
    contents = await file.read()

    # 保存到磁盘
    file_path = os.path.join(UPLOAD_DIR, file.filename)
    with open(file_path, "wb") as f:
        f.write(contents)

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }


@router.post("/multiple/")
async def upload_multiple(files: list[UploadFile] = File()):
    return [{"filename": f.filename} for f in files]
```

---

## 12. 后台任务

```python
from fastapi import APIRouter, BackgroundTasks

router = APIRouter(prefix="/tasks", tags=["tasks"])


def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(f"{message}\n")


async def send_email(to: str, subject: str):
    # 模拟发送邮件
    await asyncio.sleep(5)
    print(f"Email sent to {to}: {subject}")


@router.post("/send-notification")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks,
):
    background_tasks.add_task(send_email, email, "Welcome!")
    return {"message": "Email will be sent in background"}
```

后台任务在返回响应后执行，适用于**非关键**操作（日志、通知、缓存清理）。关键操作应使用消息队列（Celery、RabbitMQ）。

---

## 13. CORS 与跨域

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://myapp.com",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Process-Time"],  # 暴露自定义响应头
)
```

开发阶段建议：

```python
# 开发环境放行所有来源
allow_origins=["*"]

# 生产环境严格限制
allow_origins=["https://your-frontend.com"]
```

---

## 14. 日志配置

### 14.1 基础日志

```python
import logging
import sys

# 配置根日志
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    stream=sys.stdout,  # 用 stdout 而非 stderr，避免日志顺序错乱
)

logger = logging.getLogger("app")  # 应用级 logger


# 在代码中使用
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    logger.info(f"Fetching item {item_id}")
    try:
        item = await db.get(Item, item_id)
        if not item:
            logger.warning(f"Item {item_id} not found")
            raise HTTPException(404)
        logger.debug(f"Item found: {item.title}")  # debug 级别默认不输出
        return item
    except Exception:
        logger.exception(f"Error fetching item {item_id}")  # 自动记录异常堆栈
        raise
```

### 14.2 结构化日志（生产推荐）

```bash
uv add structlog
```

```python
import structlog
import logging

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),  # JSON 格式，直接输送到日志系统
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger("app")


@app.get("/items/{item_id}")
async def get_item(item_id: int, request: Request):
    logger.info("fetching_item", item_id=item_id, path=str(request.url))
    return item
```

输出示例（JSON 格式，可直接输入到 ELK/Datadog）：

```json
{"event": "fetching_item", "item_id": 42, "path": "/items/42",
 "level": "info", "timestamp": "2024-01-01T12:00:00Z", "logger": "app"}
```

### 14.3 日志分级

| 级别 | 方法 | 使用场景 |
|---|---|---|
| `DEBUG` | `logger.debug()` | 开发调试，生产一般关闭 |
| `INFO` | `logger.info()` | 正常业务流程记录 |
| `WARNING` | `logger.warning()` | 可疑但非错误（如重试） |
| `ERROR` | `logger.error()` | 操作失败但不影响整体 |
| `CRITICAL` | `logger.critical()` | 系统级故障 |

### 14.4 请求日志中间件

```python
import time
import logging

logger = logging.getLogger("app.access")


@app.middleware("http")
async def access_log_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    logger.info(
        "%s %s → %s (%.0fms)",
        request.method,
        request.url.path,
        response.status_code,
        duration * 1000,
    )
    return response
```

---

## 15. WebSocket

### 15.1 基础 WebSocket

```python
from fastapi import WebSocket, WebSocketDisconnect


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

### 15.2 WebSocket 连接管理器

```python
from fastapi import WebSocket
import json


class ConnectionManager:
    def __init__(self):
        self.active: dict[int, list[WebSocket]] = {}  # room_id → [ws, ws, ...]

    async def connect(self, websocket: WebSocket, room_id: int):
        await websocket.accept()
        if room_id not in self.active:
            self.active[room_id] = []
        self.active[room_id].append(websocket)

    def disconnect(self, websocket: WebSocket, room_id: int):
        self.active[room_id].remove(websocket)
        if not self.active[room_id]:
            del self.active[room_id]

    async def broadcast(self, room_id: int, message: dict):
        for ws in self.active.get(room_id, []):
            try:
                await ws.send_json(message)
            except Exception:
                pass

    @property
    def stats(self) -> dict:
        return {
            "rooms": len(self.active),
            "connections": sum(len(v) for v in self.active.values()),
        }


manager = ConnectionManager()


@router.websocket("/ws/chat/{room_id}")
async def chat_websocket(websocket: WebSocket, room_id: int):
    await manager.connect(websocket, room_id)
    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)
            await manager.broadcast(room_id, {
                "user": message.get("user", "anonymous"),
                "text": message.get("text", ""),
                "room_id": room_id,
            })
    except WebSocketDisconnect:
        manager.disconnect(websocket, room_id)
        await manager.broadcast(room_id, {
            "type": "system",
            "text": f"User left room {room_id}",
        })


@router.get("/ws/stats")
async def ws_stats():
    return manager.stats
```

WebSocket 客户端示例：

```javascript
// 浏览器端
const ws = new WebSocket("ws://localhost:8000/api/v1/ws/chat/1");

ws.onopen = () => ws.send(JSON.stringify({user: "alice", text: "hello"}));
ws.onmessage = (e) => console.log(JSON.parse(e.data));
```

### 15.3 WebSocket 鉴权

```python
from fastapi import WebSocket, status
from app.core.security import decode_token


@router.websocket("/ws/auth")
async def auth_websocket(websocket: WebSocket):
    token = websocket.query_params.get("token")
    if not token or not decode_token(token):
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    await websocket.accept()
    # ... 正常处理
```

### 15.4 SSE（Server-Sent Events）

```python
from fastapi.responses import StreamingResponse
import asyncio


@app.get("/events")
async def event_stream():
    async def event_generator():
        while True:
            yield f"data: {{\"time\": \"{datetime.now(timezone.utc).isoformat()}\"}}\n\n"
            await asyncio.sleep(1)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        },
    )
```

---

## 16. 测试

### 16.1 配置

```bash
uv add --dev pytest httpx pytest-asyncio
```

`pyproject.toml`：

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### 16.2 编写测试

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app


@pytest.fixture
def client():
    transport = ASGITransport(app=app)
    with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac


@pytest.mark.asyncio
async def test_health_check(client: AsyncClient):
    response = await client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


@pytest.mark.asyncio
async def test_create_item(client: AsyncClient):
    response = await client.post(
        "/api/v1/items/",
        json={"title": "test item"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "test item"
    assert "id" in data
```

### 16.3 运行测试

```bash
uv run pytest -v
uv run pytest -v -k "item"     # 按名称筛选
uv run pytest -v --cov=app     # 覆盖率
```

### 16.4 测试数据库

```python
import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app
from app.database import engine, Base, async_session


@pytest.fixture(autouse=True)
async def setup_database():
    """每个测试函数前重建表结构，保证测试隔离"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac


@pytest.fixture
async def db():
    async with async_session() as session:
        yield session


@pytest.fixture
async def auth_headers(client: AsyncClient, db: AsyncSession):
    """注册用户并返回认证头"""
    await client.post("/api/v1/auth/register", json={
        "username": "testuser",
        "email": "test@example.com",
        "password": "testpass123",
    })
    resp = await client.post("/api/v1/auth/login", data={
        "username": "testuser",
        "password": "testpass123",
    })
    token = resp.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


# 使用
@pytest.mark.asyncio
async def test_create_item(client: AsyncClient, auth_headers: dict):
    response = await client.post(
        "/api/v1/items/",
        json={"title": "test"},
        headers=auth_headers,
    )
    assert response.status_code == 201
```

### 16.5 Mock 外部服务

```python
from unittest.mock import patch, AsyncMock


@pytest.mark.asyncio
async def test_send_email():
    """Mock 邮件服务，不真实发送"""
    with patch("app.services.email.EmailService.send", new_callable=AsyncMock) as mock_send:
        mock_send.return_value = True

        response = await client.post("/auth/forgot-password", json={"email": "test@test.com"})
        assert response.status_code == 200
        mock_send.assert_called_once()
```

---

## 17. 部署

### 17.1 多阶段 Dockerfile

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock* ./
RUN uv sync --no-dev --frozen

FROM python:3.12-slim AS runner

WORKDIR /app

COPY --from=builder /app/.venv /app/.venv
COPY . .

ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 17.2 Docker Compose

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=mysql+aiomysql://root:root@db:3306/app_db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=app_db
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
```

### 17.3 环境变量管理

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    cors_origins: str = "http://localhost:3000"
    debug: bool = False

    @property
    def cors_origin_list(self) -> list[str]:
        return [o.strip() for o in self.cors_origins.split(",")]

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

### 多环境配置（dev/staging/prod）

```python
# ── .env.dev（开发环境） ─────────────────────────────
# DATABASE_URL=mysql+aiomysql://root:root@localhost:3306/app_db
# DEBUG=true

# ── .env.prod（生产环境） ────────────────────────────
# DATABASE_URL=mysql+aiomysql://user:pass@prod-host:3306/app_db
# DEBUG=false
# JWT_SECRET_KEY=real-secret-key

# ── config.py 中按环境加载不同的文件 ────────────────

import os


class Settings(BaseSettings):
    environment: str = "development"
    debug: bool = False
    database_url: str
    jwt_secret_key: str
    ...

    model_config = {
        "env_file": f".env.{os.getenv('APP_ENV', 'dev')}",  # 根据 APP_ENV 加载不同文件
        "env_file_encoding": "utf-8",
        "extra": "ignore",  # 忽略多余的环境变量
    }


# 运行方式：
# APP_ENV=prod uv run uvicorn app.main:app

# 或者使用 Docker 环境变量覆盖（推荐）：
# docker run -e DATABASE_URL=... -e JWT_SECRET_KEY=... myapp
```

### 17.4 Sentry 错误监控

```bash
uv add sentry-sdk
```

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

from app.config import get_settings

settings = get_settings()

sentry_sdk.init(
    dsn=settings.sentry_dsn,   # 从 Sentry 项目设置中获取
    integrations=[
        FastApiIntegration(),
        SqlalchemyIntegration(),
    ],
    traces_sample_rate=0.1,     # 性能追踪采样率 10%
    environment=settings.environment,  # "development" / "staging" / "production"
)


# 手动捕获异常
try:
    risky_operation()
except Exception as e:
    sentry_sdk.capture_exception(e)


# 设置用户上下文（方便定位问题用户）
from sentry_sdk import set_user
set_user({"id": current_user.id, "username": current_user.username})
```

### 17.5 GitHub Actions CI/CD

`.github/workflows/ci.yml`：

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DATABASE_URL: sqlite+aiosqlite:///./test.db
  JWT_SECRET_KEY: test-secret-key

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: latest

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync

      - name: Lint
        run: uv run ruff check .

      - name: Type check
        run: uv run mypy app/

      - name: Test
        run: uv run pytest -v --cov=app --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:latest .

      - name: Push to registry
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        run: |
          echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
          docker tag myapp:latest $DOCKER_USERNAME/myapp:latest
          docker push $DOCKER_USERNAME/myapp:latest
```

---

## 18. 最佳实践与项目结构

### 18.1 推荐项目结构

```
backend/
├── pyproject.toml
├── uv.lock
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
└── app/
    ├── __init__.py
    ├── main.py              # 应用入口
    ├── config.py            # 配置
    ├── database.py          # 数据库引擎
    ├── dependencies.py      # 全局依赖
    ├── api/
    │   ├── __init__.py
    │   ├── router.py        # 汇总所有路由
    │   └── v1/
    │       ├── __init__.py
    │       ├── auth.py      # 认证接口
    │       ├── users.py     # 用户接口
    │       └── items.py     # 资源接口
    ├── core/
    │   ├── __init__.py
    │   ├── security.py      # JWT/密码工具
    │   └── exceptions.py    # 自定义异常
    ├── crud/
    │   ├── __init__.py
    │   ├── user.py          # 用户数据库操作
    │   └── item.py          # 资源数据库操作
    ├── models/
    │   ├── __init__.py
    │   ├── user.py
    │   └── item.py
    └── schemas/
        ├── __init__.py
        ├── user.py
        └── item.py
```

### 18.2 分层职责

| 层 | 职责 | 依赖 |
|---|---|---|
| `api/` | 路由定义、参数解析、状态码 | `schemas/`, `crud/`, `dependencies` |
| `schemas/` | Pydantic 输入/输出模型 | 无 |
| `crud/` | 数据库操作（增删改查） | `models/`, `database` |
| `models/` | SQLAlchemy ORM 模型 | `database` |
| `core/` | 工具函数（JWT、密码、异常） | 无 |

### 18.3 编码规范

```python
# 1. 异步优先
async def get_user(db: AsyncSession, user_id: int):
    ...

# 2. 类型注解完整
def create_item(data: ItemCreate, db: AsyncSession) -> Item:
    ...

# 3. 显式状态码
@app.post("/items/", status_code=201)
@app.delete("/items/{id}", status_code=204)

# 4. 路由按资源组织
prefix="/api/v1/items", tags=["items"]

# 5. 使用依赖注入管理会话
def get_current_user(db=Depends(get_db)):
    ...

# 6. 环境变量分离
settings = get_settings()  # 从 .env 读取

# 7. 敏感信息不过滤
# UserResponse 不包含 hashed_password 字段

# 8. 错误信息有含义
raise HTTPException(404, detail="User not found")
```

### 18.4 API 版本管理

```python
# 方式一：URL 路径版本（推荐）
# /api/v1/posts → v1
# /api/v2/posts → v2

# app/api/router.py
from fastapi import APIRouter

router_v1 = APIRouter(prefix="/api/v1")
router_v1.include_router(posts_router)

# 新版本变化大时，创建 v2 路由器
router_v2 = APIRouter(prefix="/api/v2")
router_v2.include_router(posts_v2_router)


# 方式二：Header 版本
from fastapi import Header


@app.get("/posts")
async def list_posts(
    api_version: str = Header(default="v1", alias="X-API-Version"),
):
    if api_version == "v2":
        # v2 逻辑
        ...
    else:
        # v1 逻辑
        ...


# 方式三：兼容策略（新增字段用 optional，不破坏旧接口）
class PostV1(BaseModel):
    id: int
    title: str


class PostV2(PostV1):
    content: str | None = None  # 新增字段设为可选
```

### 18.5 常见陷阱

| 陷阱 | 正确做法 |
|---|---|
| `datetime.utcnow()` | `datetime.now(timezone.utc)` |
| `default=datetime.now()`（导入时执行） | `default=function_ref`（不带括号） |
| 同步数据库驱动阻塞事件循环 | 使用异步驱动（aiomysql/asyncpg） |
| 密码明文存储 | 使用 passlib bcrypt 哈希 |
| JWT 密钥硬编码 | 从环境变量读取 |
| `response_model` 包含密码字段 | 输出模型不包含敏感字段 |
| CORS 生产环境设为 `*` | 严格限制来源 |
| 在路由中直接操作数据库 | 通过 crud 层封装 |

### 18.6 学习路径总结

```
第一阶段：基础
├── 路由、参数、请求体
├── Pydantic 验证
└── uvicorn 启动

第二阶段：数据库
├── SQLAlchemy 2.0 ORM
├── Alembic 迁移
└── CRUD 操作

第三阶段：进阶
├── 依赖注入
├── 认证（JWT）
├── 错误处理
└── 中间件

第四阶段：完善
├── 文件上传
├── 后台任务
├── 测试
└── 部署（Docker）

第五阶段：扩展
├── WebSocket
├── 消息队列（Celery / RabbitMQ）
├── 缓存（Redis）
├── 监控与日志
├── CI/CD
└── GraphQL（Strawberry）
```

---

## 19. 常见错误排查

### 19.1 启动时报错

| 错误信息 | 原因 | 解决方法 |
|---|---|---|
| `ModuleNotFoundError: No module named 'app'` | 运行目录不对 | 确保在 `backend/` 目录下执行 `uvicorn app.main:app` |
| `Error loading ASGI app. Could not import module "app.main"` | `main.py` 位置错误 | `main.py` 应该在 `app/main.py`，不是项目根目录 |
| `sqlalchemy.exc.OperationalError: Can't connect to MySQL server` | MySQL 未启动或连接信息错误 | 检查 `.env` 中 `DATABASE_URL` 和 MySQL 服务状态 |
| `ModuleNotFoundError: No module named 'aiomysql'` | 缺少依赖 | 执行 `uv add aiomysql` |
| `pydantic_core.ValidationError: 1 validation error for Settings` | `.env` 文件缺少必要字段 | 检查 `.env` 是否包含所有必填变量 |
| `ImportError: cannot import name 'AsyncGenerator' from 'collections.abc'` | Python 版本过低 | 需要使用 Python >= 3.10 |

### 19.2 运行时错误

```python
# ── AttributeError: 'AsyncSession' object has no attribute 'query'
# SQLAlchemy 2.0 异步会话使用 execute()，不是 query()
# ❌ 错误
items = db.query(Item).all()
# ✅ 正确
result = await db.execute(select(Item))
items = result.scalars().all()


# ── RuntimeError: You cannot use AsyncToSync in the same process
# 在异步函数中调用了同步数据库操作
# ✅ 要么全部异步，要么全部同步，不要混用


# ── pydantic.ValidationError: Field required (type=missing)
# 请求体缺少必填字段
# ✅ 检查 JSON 请求体是否包含了所有必填字段


# ── sqlalchemy.exc.IntegrityError: (1062, "Duplicate entry 'admin' for key 'users.username'")
# 唯一约束冲突（用户名/邮箱重复）
# ✅ 操作前先检查唯一字段是否存在


# ── sqlalchemy.exc.OperationalError: (2006, "MySQL server has gone away")
# 数据库连接超时断开
# ✅ 设置 pool_recycle=3600（小于 MySQL 的 wait_timeout）
```

### 19.3 Alembic 迁移问题

| 问题 | 原因 | 解决 |
|---|---|---|
| `Target database is not up to date` | 迁移版本落后 | 执行 `alembic upgrade head` |
| `No changes detected` | 模型未 import | 在 `alembic/env.py` 中 import 所有模型 |
| `FAILED: No such revision` | 迁移文件被删除 | 删除 `alembic/versions/` 中手动删除的文件，或用 `alembic stamp head` |
| `Can't locate revision identified by` | 迁移历史不一致 | `alembic stamp head` 标记当前状态 |

### 19.4 Docker 问题

```bash
# ── port is already allocated
# 端口被占用
# 方案一：停掉占用端口的服务
lsof -i :8000
kill -9 <PID>

# 方案二：在 docker-compose.yml 中更换端口
ports:
  - "8001:8000"


# ── container name already in use
# 容器未清理
docker compose down


# ── ERROR: failed to solve: failed to read dockerfile
# Dockerfile 不存在
ls Dockerfile*


# ── Module not found in Docker container
# .dockerignore 排除了源码 或 Dockerfile 未 COPY 源码
# 检查 Dockerfile 中是否有 COPY . . 或 COPY app/ app/
```

---

## 20. 参考链接

### 官方文档

| 技术 | 地址 |
|---|---|
| FastAPI | https://fastapi.tiangolo.com/ |
| Pydantic v2 | https://docs.pydantic.dev/latest/ |
| SQLAlchemy 2.0 | https://docs.sqlalchemy.org/en/20/ |
| Alembic | https://alembic.sqlalchemy.org/en/latest/ |
| Uvicorn | https://www.uvicorn.org/ |
| python-jose | https://python-jose.readthedocs.io/ |
| passlib | https://passlib.readthedocs.io/ |
| httpx | https://www.python-httpx.org/ |
| pytest | https://docs.pytest.org/ |
| Ruff | https://docs.astral.sh/ruff/ |
| uv | https://docs.astral.sh/uv/ |
| Docker | https://docs.docker.com/ |
| Docker Compose | https://docs.docker.com/compose/ |

### 学习资源

| 资源 | 地址 |
|---|---|
| FastAPI 官方教程 | https://fastapi.tiangolo.com/tutorial/ |
| FastAPI 进阶指南 | https://fastapi.tiangolo.com/advanced/ |
| FastAPI 最佳实践（GitHub） | https://github.com/zhanymkanov/fastapi-best-practices |
| SQLAlchemy 2.0 教程 | https://docs.sqlalchemy.org/en/20/tutorial/ |
| Alembic 教程 | https://alembic.sqlalchemy.org/en/latest/tutorial.html |
| Real Python FastAPI | https://realpython.com/fastapi-python-web-apis/ |
| Full Stack FastAPI Template | https://github.com/fastapi/full-stack-fastapi-template |

### 第三方扩展库

| 库 | 用途 | 地址 |
|---|---|---|
| Strawberry | GraphQL | https://strawberry.rocks/ |
| Celery | 任务队列 | https://docs.celeryq.dev/ |
| Redis-py | Redis 客户端 | https://github.com/redis/redis-py |
| SlowApi | 限流 | https://github.com/laurents/slowapi |
| loguru | 日志 | https://github.com/Delgan/loguru |
| SQLAdmin | 数据库管理后台 | https://github.com/aminalaee/sqladmin |
| Piccolo | ORM（可选） | https://piccolo-orm.com/ |
```
