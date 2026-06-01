# FastAPI 项目实践与 Demo

> 完整项目示例 + 参数详解 + 常见业务模式

---

## 目录

1. [API 参数传递详解](#1-api-参数传递详解)
2. [完整项目：博客 API](#2-完整项目博客-api)
3. [常见业务模式](#3-常见业务模式)

---

## 1. API 参数传递详解

### 1.1 参数类型总览

| 来源 | 声明方式 | 示例 URL | 说明 |
|---|---|---|---|
| 路径参数 | `{param}` | `/users/42` | URL 路径的一部分 |
| 查询参数 | `param=` | `/users/?page=1` | URL 问号后 `?key=value` |
| 请求体 | Pydantic 模型 | POST `/users/` | JSON 请求体 |
| 表单参数 | `Form()` | POST `/login` | `multipart/form-data` 或 `urlencoded` |
| 文件参数 | `UploadFile` | POST `/upload` | 文件上传 |
| Header | `Header()` | 任意 | HTTP 请求头 |
| Cookie | `Cookie()` | 任意 | Cookie |
| 依赖注入 | `Depends()` | 任意 | 可复用的参数组合 |

### 1.2 路径参数详解

路径参数是 URL 路径中的动态部分，用花括号 `{}` 声明：

```python
# 基础用法
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}


# 多个路径参数
@app.get("/users/{user_id}/posts/{post_id}")
async def get_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}


# 路径参数类型转换（自动 422 校验失败）
@app.get("/items/{item_id}")
async def get_item(item_id: int):    # 传入 "abc" → 422
    ...


@app.get("/items/{item_id}")
async def get_item(item_id: str):    # 任何值都接受
    ...


# 路径参数校验
from fastapi import Path


@app.get("/items/{item_id}")
async def get_item(
    item_id: int = Path(gt=0, description="物品 ID"),
):
    ...


# 枚举路径参数
from enum import Enum


class UserRole(str, Enum):
    admin = "admin"
    user = "user"
    guest = "guest"


@app.get("/roles/{role}")
async def get_role(role: UserRole):   # 只接受 admin/user/guest
    return {"role": role}
```

**请求示例：**

```bash
curl http://localhost:8000/users/42
# → {"user_id": 42}

curl http://localhost:8000/users/abc
# → 422: {"detail": [{"type": "int_parsing", "msg": "Input should be a valid integer"}]}

curl http://localhost:8000/roles/admin
# → {"role": "admin"}

curl http://localhost:8000/roles/superadmin
# → 422: 只接受 admin/user/guest
```

### 1.3 查询参数详解

查询参数是 URL 中间号 `?` 后的 `key=value&key=value`：

```python
# 基本查询参数
@app.get("/items/")
async def list_items(
    skip: int = 0,           # 可选，默认 0
    limit: int = 10,         # 可选，默认 10
    q: str | None = None,    # 可选，默认 None
):
    return {"skip": skip, "limit": limit, "q": q}


# 必填查询参数
@app.get("/search/")
async def search(
    keyword: str,             # 必填，没有默认值
    page: int = 1,
):
    ...


# 查询参数类型校验
from fastapi import Query


@app.get("/items/")
async def list_items(
    q: str | None = Query(default=None, max_length=50, description="搜索关键词"),
    page: int = Query(default=1, ge=1),
    size: int = Query(default=20, ge=1, le=100),
    sort: str = Query(default="id", pattern=r"^(id|title|created_at)$"),
):
    return {"q": q, "page": page, "size": size, "sort": sort}


# 布尔值查询参数
@app.get("/items/")
async def list_items(
    is_done: bool | None = None,    # ?is_done=true / ?is_done=1 / ?is_done=yes
):
    ...


# 列表查询参数
@app.get("/items/")
async def list_items(
    tags: list[str] = Query(default=[]),  # ?tags=a&tags=b&tags=c
    ids: list[int] = Query(default=[]),   # ?ids=1&ids=2&ids=3
):
    ...


# 弃用参数
@app.get("/items/")
async def list_items(
    old_param: str | None = Query(default=None, deprecated=True),
):
    ...
```

**请求示例：**

```bash
# 基础
curl "http://localhost:8000/items/?skip=0&limit=10"

# 可选参数可省略
curl "http://localhost:8000/items/"

# 必填参数不能省略
curl "http://localhost:8000/search/"       # 422，缺少 keyword

# 列表参数
curl "http://localhost:8000/items/?tags=a&tags=b"

# 布尔值多种写法
curl "http://localhost:8000/items/?is_done=true"
curl "http://localhost:8000/items/?is_done=1"
curl "http://localhost:8000/items/?is_done=yes"
```

### 1.4 请求体参数详解

请求体参数通过 Pydantic 模型声明，从 JSON 请求体自动解析：

```python
from pydantic import BaseModel, Field


class CreateItem(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=1000)
    price: float = Field(gt=0, le=999999)
    tags: list[str] = []


@app.post("/items/")
async def create_item(item: CreateItem):   # 自动从 JSON 解析
    return item


# 请求体 + 路径参数 + 查询参数混用
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,                # 路径参数
    item: CreateItem,            # 请求体（JSON）
    force: bool | None = None,   # 查询参数
):
    return {"item_id": item_id, "force": force, **item.model_dump()}


# 多个请求体参数
from pydantic import BaseModel


class Item(BaseModel):
    name: str


class User(BaseModel):
    username: str


@app.put("/combine/")
async def combine(
    item: Item,      # {"name": "foo"}
    user: User,      # {"username": "bar"}
):
    return {"item": item, "user": user}


# 单个值作为请求体
from fastapi import Body


@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    name: str = Body(embed=True),  # {"name": "new name"}
):
    ...


# 请求体字段别名
class Item(BaseModel):
    item_name: str = Field(alias="name")  # JSON 中用 "name"，代码中用 item_name

    model_config = {"populate_by_name": True}
```

**请求示例：**

```bash
# 标准请求体
curl -X POST http://localhost:8000/items/ \
  -H "Content-Type: application/json" \
  -d '{"title": "My Item", "price": 9.99}'

# 请求体 + 路径参数 + 查询参数
curl -X PUT http://localhost:8000/items/42?force=true \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated", "price": 19.99}'
```

### 1.5 表单参数详解

```python
from fastapi import Form, File, UploadFile


# 登录表单
@app.post("/login/")
async def login(
    username: str = Form(),
    password: str = Form(),
):
    return {"username": username}


# 表单 + 文件混合
@app.post("/register/")
async def register(
    username: str = Form(min_length=3),
    avatar: UploadFile = File(),
):
    return {"username": username, "filename": avatar.filename}


# 表单可选参数
@app.post("/feedback/")
async def feedback(
    name: str = Form(),
    email: str | None = Form(default=None),
    message: str = Form(max_length=500),
):
    ...
```

**请求示例：**

```bash
# 表单登录
curl -X POST http://localhost:8000/login/ \
  -d "username=admin&password=123456"

# 表单 + 文件
curl -X POST http://localhost:8000/register/ \
  -F "username=john" \
  -F "avatar=@photo.jpg"
```

### 1.6 Header 和 Cookie 参数

```python
from fastapi import Header, Cookie, Response


@app.get("/")
async def read_root(
    user_agent: str | None = Header(default=None),
    accept_language: str | None = Header(default=None),
    x_token: str | None = Header(default=None, alias="X-Token"),
):
    return {
        "user_agent": user_agent,
        "accept_language": accept_language,
        "x_token": x_token,
    }


@app.get("/cookies/")
async def read_cookies(
    session_id: str | None = Cookie(default=None),
    response: Response = None,
):
    # 读取 Cookie
    print(session_id)

    # 设置 Cookie
    response.set_cookie(key="session_id", value="abc123", max_age=3600)
    return {"message": "Cookie set"}


# 重复 Header 值
@app.get("/headers/")
async def read_headers(
    values: list[str] | None = Header(default=None),  # X-Value: a, X-Value: b
):
    return {"values": values}
```

**请求示例：**

```bash
# 自定义 Header
curl -H "X-Token: my-secret-token" http://localhost:8000/

# Cookie
curl --cookie "session_id=abc123" http://localhost:8000/cookies/
```

### 1.7 参数优先级与混用

FastAPI 按以下规则自动识别参数来源：

| 参数位置 | 识别方式 |
|---|---|
| 路径 `{param}` | 声明在路径模板中 |
| 查询参数 | 非路径、非 Pydantic、非特殊类型 |
| 请求体 | Pydantic BaseModel |
| 表单 | 显式 `Form()` |
| 文件 | `UploadFile` / `File()` |
| Header | 显式 `Header()` |
| Cookie | 显式 `Cookie()` |
| 依赖 | 显式 `Depends()` |

```python
# 完整混用示例
@app.put("/users/{user_id}/items/{item_id}")
async def complex_api(
    # 路径参数
    user_id: int,
    item_id: int,

    # 查询参数
    force: bool = False,
    q: str | None = None,

    # 请求体
    data: UpdateItem,

    # Header
    x_request_id: str | None = Header(default=None),

    # Cookie
    session: str | None = Cookie(default=None),

    # 依赖注入
    db: AsyncSession = Depends(get_db),
):
    ...
```

**请求示例：**

```bash
curl -X PUT "http://localhost:8000/users/42/items/7?force=true&q=test" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: req-123" \
  --cookie "session=abc123" \
  -d '{"title": "Updated Item", "price": 29.99}'
```

---

## 2. 完整项目：博客 API

本节构建一个完整的博客后端，包含用户认证、文章 CRUD、评论系统。

### 2.1 项目结构

```
blog/
├── pyproject.toml
├── .env
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── dependencies.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── router.py
│   │   ├── auth.py
│   │   ├── posts.py
│   │   └── comments.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── post.py
│   │   └── comment.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── post.py
│   │   └── comment.py
│   └── core/
│       ├── __init__.py
│       ├── security.py
│       └── exceptions.py
```

### 2.2 配置与数据库

`app/config.py`：

```python
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    database_url: str = "mysql+aiomysql://root:root@localhost:3306/blog_db"
    jwt_secret_key: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    cors_origins: str = "http://localhost:3000"

    @property
    def cors_origin_list(self) -> list[str]:
        return [o.strip() for o in self.cors_origins.split(",")]

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`app/database.py`：

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase
from collections.abc import AsyncGenerator

from app.config import get_settings

settings = get_settings()

engine = create_async_engine(settings.database_url, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### 2.3 模型定义

`app/models/user.py`：

```python
from sqlalchemy import String, Boolean, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime, timezone

from app.database import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    bio: Mapped[str | None] = mapped_column(String(500), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(default=_utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=_utcnow, onupdate=_utcnow)

    posts: Mapped[list["Post"]] = relationship(back_populates="author", cascade="all, delete-orphan")
    comments: Mapped[list["Comment"]] = relationship(back_populates="author", cascade="all, delete-orphan")
```

`app/models/post.py`：

```python
from sqlalchemy import String, Text, Boolean, ForeignKey, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime, timezone

from app.database import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    published: Mapped[bool] = mapped_column(Boolean, default=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    created_at: Mapped[datetime] = mapped_column(default=_utcnow)
    updated_at: Mapped[datetime] = mapped_column(default=_utcnow, onupdate=_utcnow)

    author: Mapped["User"] = relationship(back_populates="posts")
    comments: Mapped[list["Comment"]] = relationship(back_populates="post", cascade="all, delete-orphan")
```

`app/models/comment.py`：

```python
from sqlalchemy import String, Text, ForeignKey, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime, timezone

from app.database import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class Comment(Base):
    __tablename__ = "comments"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    post_id: Mapped[int] = mapped_column(ForeignKey("posts.id", ondelete="CASCADE"))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    created_at: Mapped[datetime] = mapped_column(default=_utcnow)

    post: Mapped["Post"] = relationship(back_populates="comments")
    author: Mapped["User"] = relationship(back_populates="comments")
```

### 2.4 核心工具

`app/core/security.py`：

```python
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone

from app.config import get_settings

settings = get_settings()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.access_token_expire_minutes)
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)


def decode_token(token: str) -> dict | None:
    try:
        return jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])
    except JWTError:
        return None
```

`app/core/exceptions.py`：

```python
from fastapi import HTTPException, status


class NotFound(HTTPException):
    def __init__(self, detail: str = "Resource not found"):
        super().__init__(status_code=status.HTTP_404_NOT_FOUND, detail=detail)


class Unauthorized(HTTPException):
    def __init__(self, detail: str = "Not authenticated"):
        super().__init__(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=detail,
            headers={"WWW-Authenticate": "Bearer"},
        )


class Forbidden(HTTPException):
    def __init__(self, detail: str = "Permission denied"):
        super().__init__(status_code=status.HTTP_403_FORBIDDEN, detail=detail)


class Conflict(HTTPException):
    def __init__(self, detail: str = "Resource already exists"):
        super().__init__(status_code=status.HTTP_409_CONFLICT, detail=detail)
```

### 2.5 认证接口

`app/api/auth.py` —— 注册、登录、获取当前用户：

```python
from fastapi import APIRouter, Depends
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, Field

from app.database import get_db
from app.models.user import User
from app.core.security import hash_password, verify_password, create_access_token, decode_token
from app.core.exceptions import Unauthorized, Conflict
from app.dependencies import get_current_user

router = APIRouter(prefix="/auth", tags=["认证"])


# ─── Schemas ───────────────────────────────────────────

class RegisterRequest(BaseModel):
    username: str = Field(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    email: str = Field(max_length=100)
    password: str = Field(min_length=8, max_length=100)


class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    bio: str | None = None

    model_config = {"from_attributes": True}


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


# ─── Routes ────────────────────────────────────────────

@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: RegisterRequest, db: AsyncSession = Depends(get_db)):
    """注册新用户"""
    # 检查用户名是否已存在
    result = await db.execute(select(User).where(User.username == data.username))
    if result.scalar_one_or_none():
        raise Conflict("Username already taken")

    # 检查邮箱是否已存在
    result = await db.execute(select(User).where(User.email == data.email))
    if result.scalar_one_or_none():
        raise Conflict("Email already registered")

    # 创建用户
    user = User(
        username=data.username,
        email=data.email,
        hashed_password=hash_password(data.password),
    )
    db.add(user)
    await db.flush()
    await db.refresh(user)
    return user


@router.post("/login", response_model=TokenResponse)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    """登录获取令牌"""
    # 查找用户（支持用户名或邮箱登录）
    result = await db.execute(
        select(User).where(
            (User.username == form_data.username) | (User.email == form_data.username)
        )
    )
    user = result.scalar_one_or_none()

    if not user or not verify_password(form_data.password, user.hashed_password):
        raise Unauthorized("Invalid username or password")

    if not user.is_active:
        raise Unauthorized("Account is disabled")

    token = create_access_token({"sub": str(user.id)})
    return TokenResponse(access_token=token)


@router.get("/me", response_model=UserResponse)
async def get_me(current_user: User = Depends(get_current_user)):
    """获取当前登录用户信息"""
    return current_user
```

**接口调用示例：**

```bash
# 注册
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "email": "alice@example.com", "password": "password123"}'

# 登录
curl -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=alice&password=password123"

# 获取当前用户（需要 Authorization header）
curl http://localhost:8000/api/v1/auth/me \
  -H "Authorization: Bearer <token>"
```

### 2.6 文章接口

`app/api/posts.py` —— 文章的增删改查、分页、搜索、排序：

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy import select, func, desc, asc
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, Field
from datetime import datetime

from app.database import get_db
from app.models.post import Post
from app.models.user import User
from app.core.exceptions import NotFound, Forbidden
from app.dependencies import get_current_user, get_optional_user

router = APIRouter(prefix="/posts", tags=["文章"])


# ─── Schemas ───────────────────────────────────────────

class PostCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    content: str = Field(min_length=1)
    published: bool = False


class PostUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    content: str | None = None
    published: bool | None = None


class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    published: bool
    author_id: int
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}


class PostListResponse(BaseModel):
    items: list[PostResponse]
    total: int
    page: int
    size: int


# ─── Routes ────────────────────────────────────────────

@router.get("/", response_model=PostListResponse)
async def list_posts(
    page: int = Query(default=1, ge=1, description="页码"),
    size: int = Query(default=20, ge=1, le=100, description="每页数量"),
    search: str | None = Query(default=None, description="搜索标题"),
    sort: str = Query(default="-created_at", pattern=r"^(-?)(created_at|updated_at|title)$", description="排序：字段名或 -字段名"),
    published: bool | None = Query(default=None, description="筛选发布状态"),
    db: AsyncSession = Depends(get_db),
):
    """文章列表（分页 + 搜索 + 排序 + 筛选）"""

    # 构建查询
    query = select(Post)

    # 筛选
    if published is not None:
        query = query.where(Post.published == published)
    if search:
        query = query.where(Post.title.ilike(f"%{search}%"))

    # 排序（"-" 前缀表示降序）
    sort_field_str = sort
    sort_desc = False
    if sort.startswith("-"):
        sort_desc = True
        sort_field_str = sort[1:]

    sort_column = getattr(Post, sort_field_str)
    query = query.order_by(desc(sort_column) if sort_desc else asc(sort_column))

    # 统计总数
    count_query = select(func.count()).select_from(query.subquery())
    total_result = await db.execute(count_query)
    total = total_result.scalar()

    # 分页
    query = query.offset((page - 1) * size).limit(size)
    result = await db.execute(query)
    posts = result.scalars().all()

    return PostListResponse(items=posts, total=total, page=page, size=size)


@router.post("/", response_model=PostResponse, status_code=201)
async def create_post(
    data: PostCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """创建文章（需登录）"""
    post = Post(**data.model_dump(), author_id=current_user.id)
    db.add(post)
    await db.flush()
    await db.refresh(post)
    return post


@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: int,
    db: AsyncSession = Depends(get_db),
):
    """获取文章详情"""
    post = await db.get(Post, post_id)
    if not post:
        raise NotFound("Post not found")
    return post


@router.put("/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    data: PostUpdate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """更新文章（仅作者可操作）"""
    post = await db.get(Post, post_id)
    if not post:
        raise NotFound("Post not found")
    if post.author_id != current_user.id:
        raise Forbidden("You can only edit your own posts")

    for key, value in data.model_dump(exclude_unset=True).items():
        setattr(post, key, value)
    await db.flush()
    await db.refresh(post)
    return post


@router.delete("/{post_id}", status_code=204)
async def delete_post(
    post_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """删除文章（仅作者可操作）"""
    post = await db.get(Post, post_id)
    if not post:
        raise NotFound("Post not found")
    if post.author_id != current_user.id:
        raise Forbidden("You can only delete your own posts")
    await db.delete(post)
```

**请求示例：**

```bash
# 创建文章（需登录）
curl -X POST http://localhost:8000/api/v1/posts/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"title": "My First Post", "content": "Hello World!", "published": true}'

# 文章列表（分页 + 搜索 + 排序）
curl "http://localhost:8000/api/v1/posts/?page=1&size=10&search=Hello&sort=-created_at&published=true"

# 获取单篇文章
curl http://localhost:8000/api/v1/posts/1

# 更新文章（仅作者）
curl -X PUT http://localhost:8000/api/v1/posts/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"title": "Updated Title"}'

# 删除文章（仅作者）
curl -X DELETE http://localhost:8000/api/v1/posts/1 \
  -H "Authorization: Bearer <token>"
```

### 2.7 评论接口

`app/api/comments.py` —— 评论的增删改查：

```python
from fastapi import APIRouter, Depends
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, Field
from datetime import datetime

from app.database import get_db
from app.models.comment import Comment
from app.models.user import User
from app.core.exceptions import NotFound, Forbidden
from app.dependencies import get_current_user

router = APIRouter(prefix="/posts/{post_id}/comments", tags=["评论"])


# ─── Schemas ───────────────────────────────────────────

class CommentCreate(BaseModel):
    content: str = Field(min_length=1, max_length=2000)


class CommentResponse(BaseModel):
    id: int
    content: str
    post_id: int
    author_id: int
    created_at: datetime

    model_config = {"from_attributes": True}


class CommentListResponse(BaseModel):
    items: list[CommentResponse]
    total: int


# ─── Routes ────────────────────────────────────────────

@router.get("/", response_model=CommentListResponse)
async def list_comments(
    post_id: int,
    db: AsyncSession = Depends(get_db),
):
    """获取文章的所有评论"""
    query = (
        select(Comment)
        .where(Comment.post_id == post_id)
        .order_by(Comment.created_at)
    )

    count_query = select(func.count()).select_from(query.subquery())
    total_result = await db.execute(count_query)
    total = total_result.scalar()

    result = await db.execute(query)
    comments = result.scalars().all()

    return CommentListResponse(items=comments, total=total)


@router.post("/", response_model=CommentResponse, status_code=201)
async def create_comment(
    post_id: int,
    data: CommentCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """发表评论（需登录）"""
    comment = Comment(content=data.content, post_id=post_id, author_id=current_user.id)
    db.add(comment)
    await db.flush()
    await db.refresh(comment)
    return comment


@router.delete("/{comment_id}", status_code=204)
async def delete_comment(
    post_id: int,
    comment_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """删除评论（仅评论作者或文章作者可操作）"""
    comment = await db.get(Comment, comment_id)
    if not comment:
        raise NotFound("Comment not found")

    if comment.author_id != current_user.id:
        # 检查是否为文章作者
        from app.models.post import Post
        post = await db.get(Post, post_id)
        if not post or post.author_id != current_user.id:
            raise Forbidden("You cannot delete this comment")

    await db.delete(comment)
```

**请求示例：**

```bash
# 发表评论
curl -X POST http://localhost:8000/api/v1/posts/1/comments/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"content": "Great post!"}'

# 查看评论
curl http://localhost:8000/api/v1/posts/1/comments/

# 删除评论
curl -X DELETE http://localhost:8000/api/v1/posts/1/comments/5 \
  -H "Authorization: Bearer <token>"
```

### 2.8 路由汇总

`app/api/router.py`：

```python
from fastapi import APIRouter

from app.api.auth import router as auth_router
from app.api.posts import router as posts_router
from app.api.comments import router as comments_router

api_router = APIRouter(prefix="/api/v1")
api_router.include_router(auth_router)
api_router.include_router(posts_router)
api_router.include_router(comments_router)
```

### 2.9 依赖模块

`app/dependencies.py`：

```python
from fastapi import Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.models.user import User
from app.core.security import decode_token
from app.core.exceptions import Unauthorized

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    """从 JWT 令牌获取当前用户"""
    payload = decode_token(credentials.credentials)
    if payload is None or payload.get("type") != "access":
        raise Unauthorized("Invalid token")

    user_id = int(payload.get("sub"))
    user = await db.get(User, user_id)
    if not user or not user.is_active:
        raise Unauthorized("User not found or disabled")
    return user


async def get_optional_user(
    credentials: HTTPAuthorizationCredentials | None = Depends(HTTPBearer(auto_error=False)),
    db: AsyncSession = Depends(get_db),
) -> User | None:
    """可选的用户认证（未登录返回 None）"""
    if credentials is None:
        return None
    payload = decode_token(credentials.credentials)
    if payload is None:
        return None
    user = await db.get(User, int(payload.get("sub")))
    return user
```

### 2.10 应用入口

`app/main.py`：

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.router import api_router
from app.config import get_settings

settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    yield


app = FastAPI(
    title="Blog API",
    description="完整的博客后端系统",
    version="1.0.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origin_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router)


@app.get("/health")
async def health():
    return {"status": "ok"}
```

### 2.11 博客 API 接口一览

| 方法 | 路径 | 认证 | 说明 |
|---|---|---|---|
| POST | `/api/v1/auth/register` | 否 | 注册 |
| POST | `/api/v1/auth/login` | 否 | 登录获取 token |
| GET | `/api/v1/auth/me` | 是 | 获取当前用户 |
| GET | `/api/v1/posts/` | 否 | 文章列表（分页/搜索/排序/筛选） |
| POST | `/api/v1/posts/` | 是 | 创建文章 |
| GET | `/api/v1/posts/{id}` | 否 | 文章详情 |
| PUT | `/api/v1/posts/{id}` | 是 | 更新文章 |
| DELETE | `/api/v1/posts/{id}` | 是 | 删除文章 |
| GET | `/api/v1/posts/{id}/comments/` | 否 | 评论列表 |
| POST | `/api/v1/posts/{id}/comments/` | 是 | 发表评论 |
| DELETE | `/api/v1/posts/{id}/comments/{cid}` | 是 | 删除评论 |
| GET | `/health` | 否 | 健康检查 |

**核心知识点对应：**

| 接口 | 涉及知识点 |
|---|---|
| 注册 | 请求体验证、密码哈希、唯一约束 |
| 登录 | 表单参数、JWT 生成、密码验证 |
| 文章列表 | 分页、搜索、排序、筛选、可选参数、查询参数校验 |
| 文章 CRUD | 路径参数、资源归属认证、HTTP 状态码 |
| 评论 | 嵌套路由、关联查询、级联删除 |

---

## 3. 常见业务模式

### 3.1 软删除

不真正删除数据，而是标记为已删除：

```python
from sqlalchemy import Boolean, DateTime
from datetime import datetime, timezone


class SoftDeleteMixin:
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)


class Post(SoftDeleteMixin, Base):
    __tablename__ = "posts"
    # ... 其他字段


# 查询时自动过滤已删除
@app.get("/posts/")
async def list_posts(db: AsyncSession = Depends(get_db)):
    query = select(Post).where(Post.is_deleted == False)
    ...


# "删除"操作改为标记
@app.delete("/posts/{post_id}", status_code=204)
async def delete_post(post_id: int, db: AsyncSession = Depends(get_db)):
    post = await db.get(Post, post_id)
    post.is_deleted = True
    post.deleted_at = datetime.now(timezone.utc)
```

### 3.2 批量操作

```python
from pydantic import BaseModel


class BatchDelete(BaseModel):
    ids: list[int] = Field(min_length=1, max_length=100)


@app.post("/posts/batch-delete", status_code=204)
async def batch_delete_posts(
    data: BatchDelete,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    result = await db.execute(
        delete(Post).where(
            Post.id.in_(data.ids),
            Post.author_id == current_user.id,
        )
    )
    if result.rowcount == 0:
        raise NotFound("No posts found to delete")


@app.post("/posts/batch-publish", status_code=204)
async def batch_publish(
    data: BatchDelete,  # 复用，只是 ids
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    await db.execute(
        update(Post)
        .where(Post.id.in_(data.ids), Post.author_id == current_user.id)
        .values(published=True)
    )
```

### 3.3 数据导出

```python
import csv
import io
from fastapi.responses import StreamingResponse


@app.get("/posts/export/csv")
async def export_posts_csv(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    result = await db.execute(
        select(Post).where(Post.author_id == current_user.id)
    )
    posts = result.scalars().all()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "Title", "Published", "Created At"])
    for post in posts:
        writer.writerow([post.id, post.title, post.published, post.created_at])

    output.seek(0)
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=posts.csv"},
    )
```

### 3.4 审计日志（记录操作人 + 时间）

```python
from sqlalchemy import String, DateTime, Text, Integer
from datetime import datetime, timezone


class AuditLog(Base):
    __tablename__ = "audit_logs"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int | None] = mapped_column(Integer, nullable=True)
    action: Mapped[str] = mapped_column(String(50))       # create / update / delete
    resource: Mapped[str] = mapped_column(String(50))     # post / comment
    resource_id: Mapped[int | None] = mapped_column(Integer, nullable=True)
    detail: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc).replace(tzinfo=None))


def log_action(
    db: AsyncSession,
    user_id: int | None,
    action: str,
    resource: str,
    resource_id: int | None = None,
    detail: str | None = None,
):
    log = AuditLog(
        user_id=user_id,
        action=action,
        resource=resource,
        resource_id=resource_id,
        detail=detail,
    )
    db.add(log)


# 使用
@router.post("/posts/", status_code=201)
async def create_post(
    data: PostCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    post = Post(**data.model_dump(), author_id=current_user.id)
    db.add(post)
    await db.flush()

    log_action(db, current_user.id, "create", "post", post.id)

    await db.refresh(post)
    return post
```

### 3.5 限流（使用 SlowApi）

```bash
uv add slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


@app.get("/posts/")
@limiter.limit("100/minute")          # 每分钟最多 100 次
async def list_posts(request: Request, db: AsyncSession = Depends(get_db)):
    ...


@app.post("/auth/login")
@limiter.limit("5/minute")            # 登录接口更严格
async def login(request: Request, ...):
    ...
```

### 3.6 缓存查询结果

```python
from functools import lru_cache
from sqlalchemy import select


# 缓存不常变动的数据
@lru_cache(maxsize=128)
def get_category_list_sync():
    """同步函数，结果缓存 60 秒"""
    ...


# 使用 Redis 缓存（需要 redis-py）
# pip install redis
import json
import aioredis


async def get_cached_or_query(db: AsyncSession, cache_key: str, query, ttl: int = 300):
    redis = await aioredis.from_url("redis://localhost:6379")
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    result = await db.execute(query)
    data = result.scalars().all()

    # 序列化并缓存
    serialized = [{"id": item.id, "title": item.title} for item in data]
    await redis.setex(cache_key, ttl, json.dumps(serialized))
    return data
```

### 3.7 带事务的批量操作

```python
from sqlalchemy.ext.asyncio import AsyncSession


async def transfer_post_ownership(
    db: AsyncSession,
    old_owner_id: int,
    new_owner_id: int,
):
    """将旧作者的所有文章转移给新作者（原子操作）"""
    try:
        result = await db.execute(
            select(Post).where(Post.author_id == old_owner_id)
        )
        posts = result.scalars().all()

        for post in posts:
            post.author_id = new_owner_id

        # 无需手动 commit，get_db 会自动提交
        # 如果中途出错，get_db 会自动回滚
    except Exception:
        await db.rollback()
        raise
```

### 3.8 游标分页（Keyset Pagination）

offset 分页在数据量大时性能急剧下降（OFFSET 越深越慢）。游标分页用 WHERE 条件跳过已翻过的页：

```python
from pydantic import BaseModel, Field
from sqlalchemy import select


class CursorParams:
    def __init__(
        self,
        cursor: int | None = None,   # 上一页最后一条的 id
        limit: int = 20,
    ):
        self.cursor = cursor
        self.limit = min(limit, 100)


class CursorPage(BaseModel):
    items: list
    next_cursor: int | None = None   # 下一页的 cursor 值
    has_more: bool = False


@app.get("/posts/", response_model=CursorPage)
async def list_posts_cursor(
    pagination: CursorParams = Depends(),
    db: AsyncSession = Depends(get_db),
):
    query = select(Post).order_by(Post.id).limit(pagination.limit + 1)

    if pagination.cursor:
        query = query.where(Post.id > pagination.cursor)

    result = await db.execute(query)
    posts = result.scalars().all()

    has_more = len(posts) > pagination.limit
    if has_more:
        posts = posts[:pagination.limit]

    next_cursor = posts[-1].id if posts else None

    return CursorPage(items=posts, next_cursor=next_cursor, has_more=has_more)
```

**请求示例：**

```bash
curl "http://localhost:8000/api/v1/posts/?cursor=50&limit=20"
# → {"items": [...], "next_cursor": 70, "has_more": true}
```

游标分页比 offset 分页的优势：
- 数据量越大优势越明显（O(1) vs O(n)）
- 翻页过程中插入/删除不会导致重复或遗漏
- 缺点：不能直接跳到指定页码

### 3.9 发送邮件

```bash
uv add aiosmtplib
```

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from app.config import get_settings


class EmailService:
    def __init__(self):
        settings = get_settings()
        self.host = settings.smtp_host
        self.port = settings.smtp_port
        self.user = settings.smtp_user
        self.password = settings.smtp_password

    async def send(
        self,
        to: str,
        subject: str,
        body: str,
        html: str | None = None,
    ):
        msg = MIMEMultipart("alternative")
        msg["Subject"] = subject
        msg["From"] = self.user
        msg["To"] = to

        msg.attach(MIMEText(body, "plain", "utf-8"))
        if html:
            msg.attach(MIMEText(html, "html", "utf-8"))

        # 同步方式（推荐放到 BackgroundTasks 中执行）
        with smtplib.SMTP(self.host, self.port) as server:
            server.starttls()
            server.login(self.user, self.password)
            server.sendmail(self.user, [to], msg.as_string())


email_service = EmailService()


# 使用（结合后台任务）
@app.post("/auth/forgot-password")
async def forgot_password(
    email: str,
    background_tasks: BackgroundTasks,
):
    token = create_reset_token(email)
    reset_url = f"http://localhost:3000/reset-password?token={token}"

    background_tasks.add_task(
        email_service.send,
        to=email,
        subject="Reset your password",
        body=f"Click here to reset: {reset_url}",
        html=f"<a href='{reset_url}'>Reset password</a>",
    )
    return {"message": "Email sent if account exists"}
```

配置项：

```python
class Settings(BaseSettings):
    smtp_host: str = "smtp.gmail.com"
    smtp_port: int = 587
    smtp_user: str = ""
    smtp_password: str = ""
```

### 3.10 预提交钩子（Pre-commit）

```bash
# 安装 pre-commit
brew install pre-commit   # macOS
pip install pre-commit    # pip

# 项目根目录创建 .pre-commit-config.yaml
```

`.pre-commit-config.yaml`：

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: ["sqlalchemy", "pydantic"]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace    # 移除行尾空格
      - id: end-of-file-fixer      # 文件末尾换行
      - id: check-yaml             # YAML 语法检查
      - id: check-json             # JSON 语法检查
      - id: check-merge-conflict   # 检查合并冲突
      - id: detect-private-key     # 检查私钥泄露
```

```bash
# 安装钩子
pre-commit install

# 手动运行
pre-commit run --all-files

# 跳过钩子（紧急提交）
git commit --no-verify
```

---

## 附录：API 设计规范速查

### URL 设计

```
GET     /api/v1/posts              # 列表
POST    /api/v1/posts              # 创建
GET     /api/v1/posts/{id}         # 详情
PUT     /api/v1/posts/{id}         # 全量更新
PATCH   /api/v1/posts/{id}         # 部分更新
DELETE  /api/v1/posts/{id}         # 删除
GET     /api/v1/posts/{id}/comments  # 子资源列表
POST    /api/v1/posts/{id}/comments  # 创建子资源
```

### 查询参数规范

```
?page=1&size=20                   # 分页
&search=keyword                   # 搜索
&sort=-created_at                 # 排序（- 表示降序）
&published=true                   # 筛选
&fields=id,title                  # 字段选择
&include=author,comments          # 包含关联
```

### 状态码规范

| 状态码 | 含义 | 使用场景 |
|---|---|---|
| 200 | OK | 查询成功 |
| 201 | Created | 创建成功 |
| 204 | No Content | 删除成功 |
| 400 | Bad Request | 参数校验失败 |
| 401 | Unauthorized | 未登录或 token 无效 |
| 403 | Forbidden | 无权限操作 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突（如用户名已存在） |
| 422 | Unprocessable Entity | 请求体验证失败 |
| 429 | Too Many Requests | 限流 |
| 500 | Internal Server Error | 服务器内部错误 |

### 错误响应格式

```json
{
    "detail": "Human readable error message"
}
```

Pydantic 校验失败时：

```json
{
    "detail": [
        {
            "type": "string_too_short",
            "loc": ["body", "title"],
            "msg": "String should have at least 1 character",
            "input": ""
        }
    ]
}
```
