# Nuxt 4 博客项目实践与 Demo

> **本文档需要配合 `01-Vue3+Nuxt4完整学习文档.md` 使用**，各章节中的概念在此项目中均有体现。

---

## 1. 项目概述

### 1.1 项目简介

一个完整的博客系统，包含以下功能：

- 用户注册与登录（JWT 认证）
- 文章 CRUD（分页、搜索、排序、过滤）
- 评论系统（嵌套关系、级联删除）
- 资源所有权（仅作者可编辑）
- 输入/输出模型分离
- 生产级特性（缓存、限流、导出、审计等）

### 1.2 技术栈

| 层 | 技术 |
|---|------|
| 框架 | Nuxt 4 + Nitro |
| 语言 | TypeScript 5.8+ |
| 样式 | UnoCSS |
| 状态 | Pinia + useState |
| 构建 | Node 22 + pnpm |

### 1.3 快速启动

```bash
# 创建项目
pnpm dlx nuxi@latest init nuxt4-blog
cd nuxt4-blog

# 安装依赖
pnpm add @pinia/nuxt @vueuse/core
pnpm add -D @unocss/nuxt vitest @vue/test-utils jsdom

# 运行
pnpm dev
```

---

## 2. 项目结构

```
nuxt4-blog/
├── app/
│   ├── components/
│   │   ├── base/
│   │   │   ├── AppButton.vue
│   │   │   └── AppInput.vue
│   │   ├── ui/
│   │   │   ├── PostCard.vue
│   │   │   └── CommentList.vue
│   │   └── TheHeader.vue
│   ├── composables/
│   │   ├── useAuth.ts
│   │   ├── usePagination.ts
│   │   └── useDebounce.ts
│   ├── layouts/
│   │   ├── default.vue
│   │   └── admin.vue
│   ├── middleware/
│   │   ├── auth.global.ts
│   │   └── admin.ts
│   ├── pages/
│   │   ├── index.vue
│   │   ├── login.vue
│   │   ├── register.vue
│   │   ├── posts/
│   │   │   ├── index.vue
│   │   │   ├── [slug].vue
│   │   │   └── create.vue
│   │   ├── admin/
│   │   │   └── index.vue
│   │   └── [...slug].vue
│   ├── plugins/
│   │   └── api.ts
│   ├── stores/
│   │   └── user.ts
│   ├── utils/
│   │   ├── format.ts
│   │   └── validation.ts
│   ├── app.vue
│   ├── app.config.ts
│   └── error.vue
├── server/
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login.post.ts
│   │   │   ├── register.post.ts
│   │   │   └── me.get.ts
│   │   ├── posts/
│   │   │   ├── index.get.ts
│   │   │   ├── [id].get.ts
│   │   │   ├── [id].put.ts
│   │   │   └── [id].delete.ts
│   │   ├── comments/
│   │   │   ├── index.post.ts
│   │   │   └── posts/[postId].get.ts
│   │   ├── export/
│   │   │   └── posts.get.ts
│   │   └── health.get.ts
│   ├── middleware/
│   │   └── auth.ts
│   ├── plugins/
│   │   └── init.ts
│   └── utils/
│       ├── auth.ts
│       ├── db.ts
│       ├── rate-limit.ts
│       └── validation.ts
├── shared/
│   └── types.ts
├── nuxt.config.ts
└── package.json
```

---

## 3. 共享类型定义

```ts
// shared/types.ts
export interface User {
  id: number
  name: string
  email: string
  avatar?: string
  role: 'user' | 'admin'
  createdAt: string
}

export interface Post {
  id: number
  title: string
  slug: string
  content: string
  excerpt: string
  published: boolean
  tags: string[]
  authorId: number
  author?: User
  createdAt: string
  updatedAt: string
}

export interface Comment {
  id: number
  content: string
  postId: number
  authorId: number
  author?: User
  createdAt: string
}

export interface PaginatedResponse<T> {
  data: T[]
  total: number
  page: number
  limit: number
  totalPages: number
}

export interface ApiError {
  statusCode: number
  message: string
  errors?: Record<string, string[]>
}
```

---

## 4. 配置

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },

  modules: [
    '@pinia/nuxt',
    '@unocss/nuxt',
  ],

  runtimeConfig: {
    jwtSecret: 'change-this-in-production',
    public: {
      apiBase: '/api',
      siteName: 'Nuxt 4 Blog',
    },
  },

  routeRules: {
    '/': { prerender: true },
    '/posts/**': { isr: 3600 },
    '/admin/**': { ssr: false },
    '/api/**': { cors: true },
  },

  nitro: {
    routeRules: {
      '/api/auth/login': {
        headers: { 'Cache-Control': 'no-store' },
      },
    },
  },
})
```

```ts
// app/app.config.ts
export default defineAppConfig({
  title: 'Nuxt 4 Blog',
  theme: {
    primary: '#00dc82',
    dark: false,
  },
})
```

---

## 5. 服务端实现

### 5.1 模拟数据库

```ts
// server/utils/db.ts
import type { User, Post, Comment } from '~/shared/types'

// 模拟内存数据库
const users: User[] = [
  {
    id: 1,
    name: '管理员',
    email: 'admin@example.com',
    role: 'admin',
    createdAt: '2025-01-01T00:00:00.000Z',
  },
  {
    id: 2,
    name: '张三',
    email: 'zhangsan@example.com',
    role: 'user',
    createdAt: '2025-01-15T00:00:00.000Z',
  },
]

const posts: Post[] = [
  {
    id: 1,
    title: 'Nuxt 4 入门指南',
    slug: 'nuxt-4-guide',
    content: 'Nuxt 4 是一个全栈 Vue 框架...',
    excerpt: '了解 Nuxt 4 的核心概念和用法',
    published: true,
    tags: ['nuxt', 'vue'],
    authorId: 1,
    createdAt: '2025-02-01T00:00:00.000Z',
    updatedAt: '2025-02-01T00:00:00.000Z',
  },
  {
    id: 2,
    title: 'TypeScript 高级类型',
    slug: 'typescript-advanced-types',
    content: '探索 TypeScript 的高级类型系统...',
    excerpt: '深入理解 TypeScript 类型编程',
    published: true,
    tags: ['typescript'],
    authorId: 1,
    createdAt: '2025-02-10T00:00:00.000Z',
    updatedAt: '2025-02-10T00:00:00.000Z',
  },
  {
    id: 3,
    title: '未发布的草稿',
    slug: 'draft-post',
    content: '这篇还在编辑中...',
    excerpt: '',
    published: false,
    tags: [],
    authorId: 2,
    createdAt: '2025-03-01T00:00:00.000Z',
    updatedAt: '2025-03-01T00:00:00.000Z',
  },
]

const comments: Comment[] = [
  {
    id: 1,
    content: '好文章，学到了！',
    postId: 1,
    authorId: 2,
    createdAt: '2025-02-02T00:00:00.000Z',
  },
  {
    id: 2,
    content: '补充一点...',
    postId: 1,
    authorId: 1,
    createdAt: '2025-02-03T00:00:00.000Z',
  },
]

const nextId = { user: 3, post: 4, comment: 3 }

// 工具函数
function paginate<T>(items: T[], page: number, limit: number) {
  const start = (page - 1) * limit
  return {
    data: items.slice(start, start + limit),
    total: items.length,
    page,
    limit,
    totalPages: Math.ceil(items.length / limit),
  }
}

function slugify(text: string): string {
  return text.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '')
}

export { users, posts, comments, nextId, paginate, slugify }
```

### 5.2 认证工具

```ts
// server/utils/auth.ts
import type { JwtPayload } from 'jsonwebtoken'

const jwt = require('jsonwebtoken') as typeof import('jsonwebtoken')

const config = useRuntimeConfig()

export function generateToken(payload: { userId: number; email: string; role: string }): string {
  return jwt.sign(payload, config.jwtSecret, { expiresIn: '7d' })
}

export function verifyToken(token: string): { userId: number; email: string; role: string } | null {
  try {
    return jwt.verify(token, config.jwtSecret)
  } catch {
    return null
  }
}

export function getAuthUser(event: any): { userId: number; email: string; role: string } | null {
  const token = getCookie(event, 'token') || getHeader(event, 'authorization')?.replace('Bearer ', '')
  if (!token) return null
  return verifyToken(token)
}

export function requireAuth(event: any): { userId: number; email: string; role: string } {
  const user = getAuthUser(event)
  if (!user) {
    throw createError({ statusCode: 401, statusMessage: '请先登录' })
  }
  return user
}
```

### 5.3 认证 API

```ts
// server/api/auth/register.post.ts
export default defineEventHandler(async (event) => {
  const { name, email, password } = await readBody(event)

  if (!name || !email || !password) {
    throw createError({ statusCode: 400, statusMessage: '缺少必填字段' })
  }

  if (users.find(u => u.email === email)) {
    throw createError({ statusCode: 409, statusMessage: '邮箱已被注册' })
  }

  const user: User = {
    id: nextId.user++,
    name,
    email,
    role: 'user',
    createdAt: new Date().toISOString(),
  }
  users.push(user)

  const token = generateToken({ userId: user.id, email: user.email, role: user.role })

  setCookie(event, 'token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7,
  })

  return { user, token }
})
```

```ts
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const { email, password } = await readBody(event)

  if (!email || !password) {
    throw createError({ statusCode: 400, statusMessage: '请输入邮箱和密码' })
  }

  const user = users.find(u => u.email === email)
  if (!user) {
    throw createError({ statusCode: 401, statusMessage: '邮箱或密码错误' })
  }

  const token = generateToken({ userId: user.id, email: user.email, role: user.role })

  setCookie(event, 'token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7,
  })

  return { user, token }
})
```

```ts
// server/api/auth/me.get.ts
export default defineEventHandler((event) => {
  const auth = requireAuth(event)
  const user = users.find(u => u.id === auth.userId)

  if (!user) {
    throw createError({ statusCode: 404, statusMessage: '用户不存在' })
  }

  return user
})
```

### 5.4 文章 API

```ts
// server/api/posts/index.get.ts
export default defineEventHandler((event) => {
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const limit = Number(query.limit) || 10
  const search = query.search as string
  const tag = query.tag as string
  const sortBy = (query.sortBy as string) || 'createdAt'
  const sortOrder = (query.sortOrder as string) || 'desc'

  let filtered = posts.filter(p => p.published)

  if (search) {
    const q = search.toLowerCase()
    filtered = filtered.filter(p =>
      p.title.toLowerCase().includes(q) || p.content.toLowerCase().includes(q),
    )
  }

  if (tag) {
    filtered = filtered.filter(p => p.tags.includes(tag))
  }

  // 排序
  filtered.sort((a: any, b: any) => {
    const mod = sortOrder === 'desc' ? -1 : 1
    return a[sortBy] > b[sortBy] ? mod : -mod
  })

  return paginate(filtered, page, limit)
})
```

```ts
// server/api/posts/index.post.ts
export default defineEventHandler(async (event) => {
  const auth = requireAuth(event)
  const body = await readBody(event)

  if (!body.title || !body.content) {
    throw createError({ statusCode: 400, statusMessage: '标题和内容不能为空' })
  }

  const post: Post = {
    id: nextId.post++,
    title: body.title,
    slug: slugify(body.title),
    content: body.content,
    excerpt: body.excerpt || body.content.slice(0, 200),
    published: body.published ?? true,
    tags: body.tags || [],
    authorId: auth.userId,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  }
  posts.unshift(post)

  return post
})
```

```ts
// server/api/posts/[id].get.ts
export default defineEventHandler((event) => {
  const id = Number(getRouterParam(event, 'id'))
  const post = posts.find(p => p.id === id && p.published)

  if (!post) {
    throw createError({ statusCode: 404, statusMessage: '文章不存在' })
  }

  const author = users.find(u => u.id === post.authorId)
  return { ...post, author }
})
```

```ts
// server/api/posts/[id].put.ts
export default defineEventHandler(async (event) => {
  const auth = requireAuth(event)
  const id = Number(getRouterParam(event, 'id'))
  const post = posts.find(p => p.id === id)

  if (!post) {
    throw createError({ statusCode: 404, statusMessage: '文章不存在' })
  }

  if (post.authorId !== auth.userId && auth.role !== 'admin') {
    throw createError({ statusCode: 403, statusMessage: '无权修改此文章' })
  }

  const body = await readBody(event)

  Object.assign(post, {
    title: body.title ?? post.title,
    slug: body.title ? slugify(body.title) : post.slug,
    content: body.content ?? post.content,
    excerpt: body.excerpt ?? body.content?.slice(0, 200) ?? post.excerpt,
    published: body.published ?? post.published,
    tags: body.tags ?? post.tags,
    updatedAt: new Date().toISOString(),
  })

  return post
})
```

```ts
// server/api/posts/[id].delete.ts
export default defineEventHandler((event) => {
  const auth = requireAuth(event)
  const id = Number(getRouterParam(event, 'id'))
  const index = posts.findIndex(p => p.id === id)

  if (index === -1) {
    throw createError({ statusCode: 404, statusMessage: '文章不存在' })
  }

  if (posts[index].authorId !== auth.userId && auth.role !== 'admin') {
    throw createError({ statusCode: 403, statusMessage: '无权删除此文章' })
  }

  // 级联删除评论
  const deletedPost = posts.splice(index, 1)[0]
  const commentIndices = comments
    .map((c, i) => (c.postId === id ? i : -1))
    .filter(i => i !== -1)
    .reverse()
  for (const i of commentIndices) {
    comments.splice(i, 1)
  }

  return { success: true, deleted: deletedPost }
})
```

### 5.5 评论 API

```ts
// server/api/comments/posts/[postId].get.ts
export default defineEventHandler((event) => {
  const postId = Number(getRouterParam(event, 'postId'))
  const postComments = comments
    .filter(c => c.postId === postId)
    .map(c => ({
      ...c,
      author: users.find(u => u.id === c.authorId),
    }))

  return postComments
})
```

```ts
// server/api/comments/index.post.ts
export default defineEventHandler(async (event) => {
  const auth = requireAuth(event)
  const body = await readBody(event)

  if (!body.content || !body.postId) {
    throw createError({ statusCode: 400, statusMessage: '内容和文章ID不能为空' })
  }

  const post = posts.find(p => p.id === body.postId && p.published)
  if (!post) {
    throw createError({ statusCode: 404, statusMessage: '文章不存在' })
  }

  const comment: Comment = {
    id: nextId.comment++,
    content: body.content,
    postId: body.postId,
    authorId: auth.userId,
    createdAt: new Date().toISOString(),
  }
  comments.push(comment)

  return { ...comment, author: users.find(u => u.id === auth.userId) }
})
```

### 5.6 健康检查与导出

```ts
// server/api/health.get.ts
export default defineEventHandler(() => {
  return {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    stats: {
      users: users.length,
      posts: posts.length,
      comments: comments.length,
    },
  }
})

// server/api/export/posts.get.ts
export default defineEventHandler(() => {
  const csv = [
    'ID,标题,作者,发布时间',
    ...posts.filter(p => p.published).map(p => {
      const author = users.find(u => u.id === p.authorId)
      return `${p.id},"${p.title}",${author?.name || '未知'},${p.createdAt}`
    }),
  ].join('\n')

  setHeader(event, 'Content-Type', 'text/csv')
  setHeader(event, 'Content-Disposition', 'attachment; filename="posts.csv"')

  return csv
})
```

### 5.7 限流中间件

```ts
// server/utils/rate-limit.ts
interface RateLimitEntry {
  count: number
  resetAt: number
}

const store = new Map<string, RateLimitEntry>()

export function checkRateLimit(key: string, maxRequests: number, windowMs: number): boolean {
  const now = Date.now()
  const entry = store.get(key)

  if (!entry || now > entry.resetAt) {
    store.set(key, { count: 1, resetAt: now + windowMs })
    return true
  }

  if (entry.count >= maxRequests) {
    return false
  }

  entry.count++
  return true
}

// server/middleware/rate-limit.ts
export default defineEventHandler((event) => {
  const ip = getHeader(event, 'x-forwarded-for') || event.node.req.socket.remoteAddress || 'unknown'
  const key = `rate-limit:${ip}`

  if (!checkRateLimit(key, 100, 60000)) {
    throw createError({
      statusCode: 429,
      statusMessage: '请求过于频繁，请稍后再试',
    })
  }
})
```

---

## 6. 前端实现

### 6.1 根组件和布局

```vue
<!-- app/app.vue -->
<script setup lang="ts">
useSeoMeta({
  title: 'Nuxt 4 Blog',
  description: '一个基于 Nuxt 4 的全栈博客系统',
})

const appConfig = useAppConfig()
</script>

<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>
```

```vue
<!-- app/layouts/default.vue -->
<script setup lang="ts">
const { isLoggedIn, user, logout } = useAuth()
</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <TheHeader :user="user" :is-logged-in="isLoggedIn" @logout="logout" />
    <main class="max-w-4xl mx-auto px-4 py-8">
      <slot />
    </main>
    <footer class="text-center py-6 text-gray-400 text-sm">
      &copy; 2025 Nuxt 4 Blog. All rights reserved.
    </footer>
  </div>
</template>
```

```vue
<!-- app/layouts/admin.vue -->
<script setup lang="ts">
const { isLoggedIn, user } = useAuth()

if (!isLoggedIn.value || user.value?.role !== 'admin') {
  navigateTo('/login')
}
</script>

<template>
  <div class="min-h-screen bg-gray-900 text-white">
    <aside class="fixed left-0 top-0 h-full w-64 bg-gray-800 p-6">
      <h2 class="text-xl font-bold mb-6">管理后台</h2>
      <nav class="space-y-2">
        <NuxtLink to="/admin" class="block px-4 py-2 rounded hover:bg-gray-700">
          概览
        </NuxtLink>
        <NuxtLink to="/admin/posts" class="block px-4 py-2 rounded hover:bg-gray-700">
          文章管理
        </NuxtLink>
      </nav>
    </aside>
    <main class="ml-64 p-8">
      <slot />
    </main>
  </div>
</template>
```

### 6.2 认证组合式函数

```ts
// app/composables/useAuth.ts
export function useAuth() {
  const user = useState<User | null>('auth-user', () => null)
  const token = useCookie('token')

  const isLoggedIn = computed(() => !!token.value)

  async function fetchUser() {
    if (!token.value) return
    try {
      user.value = await $fetch('/api/auth/me')
    } catch {
      token.value = null
      user.value = null
    }
  }

  async function login(email: string, password: string) {
    const result = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    })
    token.value = result.token
    user.value = result.user
    navigateTo('/')
  }

  async function register(name: string, email: string, password: string) {
    const result = await $fetch<{ user: User; token: string }>('/api/auth/register', {
      method: 'POST',
      body: { name, email, password },
    })
    token.value = result.token
    user.value = result.user
    navigateTo('/')
  }

  function logout() {
    token.value = null
    user.value = null
    navigateTo('/')
  }

  // 初始化时获取用户
  if (import.meta.client && token.value && !user.value) {
    fetchUser()
  }

  return { user, token, isLoggedIn, login, register, logout, fetchUser }
}
```

### 6.3 分页组合式函数

```ts
// app/composables/usePagination.ts
export function usePagination<T>(url: string, options?: {
  defaultLimit?: number
  watch?: Ref<any>[]
}) {
  const page = ref(1)
  const limit = ref(options?.defaultLimit || 10)
  const data = ref<T[]>([])
  const total = ref(0)
  const totalPages = ref(0)

  const { status, refresh } = useFetch(url, {
    query: { page, limit },
    watch: [page, ...(options?.watch || [])],
    onResponse({ response }) {
      const result = response._data
      data.value = result.data
      total.value = result.total
      totalPages.value = result.totalPages
    },
  })

  const hasMore = computed(() => page.value < totalPages.value)

  function nextPage() {
    if (hasMore.value) page.value++
  }

  function prevPage() {
    if (page.value > 1) page.value--
  }

  function goToPage(n: number) {
    page.value = Math.max(1, Math.min(n, totalPages.value))
  }

  return {
    data,
    total,
    page,
    limit,
    totalPages,
    hasMore,
    status,
    nextPage,
    prevPage,
    goToPage,
    refresh,
  }
}
```

### 6.4 防抖组合式函数

```ts
// app/composables/useDebounce.ts
export function useDebounce<T>(value: Ref<T>, delay = 300) {
  const debouncedValue = ref(value.value) as Ref<T>
  let timer: ReturnType<typeof setTimeout>

  watch(value, (newVal) => {
    clearTimeout(timer)
    timer = setTimeout(() => {
      debouncedValue.value = newVal
    }, delay)
  })

  return debouncedValue
}
```

### 6.5 API 插件

```ts
// app/plugins/api.ts
export default defineNuxtPlugin(() => {
  const api = $fetch.create({
    baseURL: '/api',
    headers: {
      'Content-Type': 'application/json',
    },
    onRequest({ options }) {
      const token = useCookie('token')
      if (token.value) {
        options.headers.set('Authorization', `Bearer ${token.value}`)
      }
    },
    onResponseError({ response }) {
      if (response.status === 401) {
        const token = useCookie('token')
        token.value = null
        navigateTo('/login')
      }
    },
  })

  return {
    provide: { api },
  }
})
```

### 6.6 基础组件

```vue
<!-- app/components/base/AppButton.vue -->
<script setup lang="ts">
withDefaults(defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
  loading?: boolean
  disabled?: boolean
}>(), {
  variant: 'primary',
})

defineEmits<{
  click: []
}>()
</script>

<template>
  <button
    :class="[
      'px-4 py-2 rounded-lg font-medium transition disabled:opacity-50',
      variant === 'primary' && 'bg-green-500 text-white hover:bg-green-600',
      variant === 'secondary' && 'bg-gray-200 text-gray-700 hover:bg-gray-300',
      variant === 'danger' && 'bg-red-500 text-white hover:bg-red-600',
    ]"
    :disabled="disabled || loading"
    @click="$emit('click')"
  >
    <span v-if="loading" class="inline-block animate-spin mr-2">&#9696;</span>
    <slot />
  </button>
</template>
```

```vue
<!-- app/components/base/AppInput.vue -->
<script setup lang="ts">
defineProps<{
  label?: string
  type?: string
  placeholder?: string
  error?: string
}>()

const model = defineModel<string>()
</script>

<template>
  <div class="space-y-1">
    <label v-if="label" class="block text-sm font-medium text-gray-700">
      {{ label }}
    </label>
    <input
      v-model="model"
      :type="type || 'text'"
      :placeholder="placeholder"
      :class="[
        'w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2',
        error ? 'border-red-300 focus:ring-red-500' : 'border-gray-300 focus:ring-green-500',
      ]"
    />
    <p v-if="error" class="text-sm text-red-500">{{ error }}</p>
  </div>
</template>
```

### 6.7 UI 组件

```vue
<!-- app/components/ui/PostCard.vue -->
<script setup lang="ts">
defineProps<{
  post: Post
}>()
</script>

<template>
  <article class="bg-white rounded-xl shadow-sm p-6 hover:shadow-md transition">
    <NuxtLink :to="`/posts/${post.slug}`" class="block">
      <h2 class="text-xl font-bold mb-2 hover:text-green-600 transition">
        {{ post.title }}
      </h2>
    </NuxtLink>
    <p class="text-gray-600 mb-4">{{ post.excerpt }}</p>
    <div class="flex items-center gap-4 text-sm text-gray-400">
      <span>{{ formatDate(post.createdAt) }}</span>
      <span class="flex gap-1">
        <span
          v-for="tag in post.tags"
          :key="tag"
          class="px-2 py-0.5 bg-green-50 text-green-600 rounded-full text-xs"
        >
          {{ tag }}
        </span>
      </span>
    </div>
  </article>
</template>
```

```vue
<!-- app/components/ui/CommentList.vue -->
<script setup lang="ts">
defineProps<{
  comments: (Comment & { author?: User })[]
}>()

const emit = defineEmits<{
  refresh: []
}>()
</script>

<template>
  <div class="space-y-4">
    <h3 class="text-lg font-bold">评论 ({{ comments.length }})</h3>
    <div v-if="comments.length === 0" class="text-gray-400 text-center py-8">
    暂无评论，来写第一条吧
    </div>
    <div
      v-for="comment in comments"
      :key="comment.id"
      class="bg-gray-50 rounded-lg p-4"
    >
      <div class="flex items-center gap-2 mb-2">
        <span class="font-medium">{{ comment.author?.name || '匿名' }}</span>
        <span class="text-sm text-gray-400">{{ formatDate(comment.createdAt) }}</span>
      </div>
      <p class="text-gray-700">{{ comment.content }}</p>
    </div>
  </div>
</template>
```

### 6.8 路由守卫

```ts
// app/middleware/auth.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const token = useCookie('token')

  // 需要登录的页面
  const protectedRoutes = ['/posts/create', '/admin']
  if (protectedRoutes.some(p => to.path.startsWith(p)) && !token.value) {
    return navigateTo('/login')
  }

  // 已登录用户无需访问登录页
  if (token.value && to.path === '/login') {
    return navigateTo('/')
  }
})
```

### 6.9 工具函数

```ts
// app/utils/format.ts
export function formatDate(date: string): string {
  return new Date(date).toLocaleDateString('zh-CN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  })
}

export function formatRelativeTime(date: string): string {
  const now = Date.now()
  const diff = now - new Date(date).getTime()
  const minutes = Math.floor(diff / 60000)
  const hours = Math.floor(minutes / 60)
  const days = Math.floor(hours / 24)

  if (minutes < 1) return '刚刚'
  if (minutes < 60) return `${minutes} 分钟前`
  if (hours < 24) return `${hours} 小时前`
  if (days < 30) return `${days} 天前`
  return formatDate(date)
}
```

```ts
// app/utils/validation.ts
export function validateEmail(email: string): string | null {
  if (!email) return '请输入邮箱'
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) return '邮箱格式不正确'
  return null
}

export function validatePassword(password: string): string | null {
  if (!password) return '请输入密码'
  if (password.length < 6) return '密码至少 6 位'
  return null
}

export function validateTitle(title: string): string | null {
  if (!title) return '请输入标题'
  if (title.length > 200) return '标题不超过 200 字'
  return null
}
```

---

## 7. 页面实现

### 7.1 首页

```vue
<!-- app/pages/index.vue -->
<script setup lang="ts">
const { data, page, totalPages, hasMore, nextPage, prevPage, status } = usePagination<Post>('/api/posts', {
  defaultLimit: 6,
})
</script>

<template>
  <div>
    <section class="text-center mb-12">
      <h1 class="text-4xl font-bold mb-4">欢迎来到 Nuxt 4 Blog</h1>
      <p class="text-gray-600 text-lg">基于 Nuxt 4 构建的全栈博客系统</p>
    </section>

    <div v-if="status === 'pending'" class="text-center py-12">
      <div class="animate-spin inline-block w-8 h-8 border-4 border-green-500 border-t-transparent rounded-full" />
    </div>

    <div v-else class="grid gap-6 md:grid-cols-2">
      <PostCard v-for="post in data" :key="post.id" :post="post" />
    </div>

    <div class="flex justify-center items-center gap-4 mt-8">
      <AppButton :disabled="page === 1" @click="prevPage">
        上一页
      </AppButton>
      <span class="text-gray-500">第 {{ page }} / {{ totalPages }} 页</span>
      <AppButton :disabled="!hasMore" @click="nextPage">
        下一页
      </AppButton>
    </div>
  </div>
</template>
```

### 7.2 文章详情

```vue
<!-- app/pages/posts/[slug].vue -->
<script setup lang="ts">
const route = useRoute('posts-slug')
const { data: post, status, error } = await useFetch<Post>(`/api/posts/${route.params.slug}`)

const comments = ref<(Comment & { author?: User })[]>([])
const newComment = ref('')

async function loadComments() {
  if (post.value) {
    comments.value = await $fetch(`/api/comments/posts/${post.value.id}`)
  }
}

async function submitComment() {
  if (!newComment.value.trim()) return
  await $fetch('/api/comments', {
    method: 'POST',
    body: { content: newComment.value, postId: post.value!.id },
  })
  newComment.value = ''
  await loadComments()
}

watch(post, () => {
  if (post.value) loadComments()
}, { immediate: true })
</script>

<template>
  <div v-if="status === 'pending'" class="text-center py-12">
    <div class="animate-spin inline-block w-8 h-8 border-4 border-green-500 border-t-transparent rounded-full" />
  </div>

  <div v-else-if="error" class="text-center py-12">
    <h2 class="text-2xl font-bold text-red-500">文章不存在</h2>
    <NuxtLink to="/" class="text-green-500 hover:underline mt-4 inline-block">
      返回首页
    </NuxtLink>
  </div>

  <article v-else class="bg-white rounded-xl shadow-sm p-8">
    <h1 class="text-3xl font-bold mb-4">{{ post!.title }}</h1>
    <div class="flex items-center gap-4 text-sm text-gray-400 mb-8">
      <span>{{ formatDate(post!.createdAt) }}</span>
      <span>{{ formatRelativeTime(post!.createdAt) }}</span>
    </div>
    <div class="prose max-w-none mb-8">
      {{ post!.content }}
    </div>

    <hr class="my-8" />

    <div class="space-y-4">
      <CommentList :comments="comments" @refresh="loadComments" />
      <div v-if="useAuth().isLoggedIn.value" class="flex gap-2">
        <AppInput v-model="newComment" placeholder="写下你的评论..." />
        <AppButton @click="submitComment">发表</AppButton>
      </div>
      <p v-else class="text-gray-400 text-sm">
        <NuxtLink to="/login" class="text-green-500 hover:underline">登录</NuxtLink>后可以发表评论
      </p>
    </div>
  </article>
</template>
```

### 7.3 创建文章

```vue
<!-- app/pages/posts/create.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: ['auth'],
})

const title = ref('')
const content = ref('')
const tags = ref('')
const published = ref(true)
const submitting = ref(false)
const errors = ref<Record<string, string>>({})

async function submit() {
  errors.value = {}

  const titleErr = validateTitle(title.value)
  if (titleErr) errors.value.title = titleErr
  if (!content.value) errors.value.content = '请输入内容'
  if (Object.keys(errors.value).length) return

  submitting.value = true
  try {
    const post = await $fetch('/api/posts', {
      method: 'POST',
      body: {
        title: title.value,
        content: content.value,
        tags: tags.value.split(',').map(t => t.trim()).filter(Boolean),
        published: published.value,
      },
    })
    navigateTo(`/posts/${post.id}`)
  } catch (e: any) {
    errors.value.form = e.data?.message || '创建失败'
  } finally {
    submitting.value = false
  }
}
</script>

<template>
  <div class="max-w-2xl mx-auto">
    <h1 class="text-2xl font-bold mb-6">写文章</h1>
    <p v-if="errors.form" class="text-red-500 mb-4">{{ errors.form }}</p>
    <form class="space-y-4" @submit.prevent="submit">
      <AppInput
        v-model="title"
        label="标题"
        placeholder="文章标题"
        :error="errors.title"
      />
      <div class="space-y-1">
        <label class="block text-sm font-medium text-gray-700">内容</label>
        <textarea
          v-model="content"
          rows="12"
          class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-green-500"
          placeholder="写下你的文章..."
        />
        <p v-if="errors.content" class="text-sm text-red-500">{{ errors.content }}</p>
      </div>
      <AppInput v-model="tags" label="标签（逗号分隔）" placeholder="nuxt, vue, typescript" />
      <label class="flex items-center gap-2">
        <input v-model="published" type="checkbox" class="rounded" />
        <span class="text-sm">立即发布</span>
      </label>
      <AppButton :loading="submitting" type="submit">发布文章</AppButton>
    </form>
  </div>
</template>
```

### 7.4 登录注册

```vue
<!-- app/pages/login.vue -->
<script setup lang="ts">
const email = ref('')
const password = ref('')
const error = ref('')
const { login } = useAuth()

async function handleLogin() {
  error.value = ''
  try {
    await login(email.value, password.value)
  } catch (e: any) {
    error.value = e.data?.message || '登录失败'
  }
}
</script>

<template>
  <div class="max-w-md mx-auto mt-12">
    <h1 class="text-2xl font-bold mb-6 text-center">登录</h1>
    <p v-if="error" class="text-red-500 text-center mb-4">{{ error }}</p>
    <form class="space-y-4" @submit.prevent="handleLogin">
      <AppInput v-model="email" type="email" label="邮箱" placeholder="your@email.com" />
      <AppInput v-model="password" type="password" label="密码" placeholder="******" />
      <AppButton class="w-full" type="submit">登录</AppButton>
    </form>
    <p class="text-center mt-4 text-sm text-gray-500">
      还没有账号？<NuxtLink to="/register" class="text-green-500 hover:underline">注册</NuxtLink>
    </p>
  </div>
</template>
```

```vue
<!-- app/pages/register.vue -->
<script setup lang="ts">
const name = ref('')
const email = ref('')
const password = ref('')
const confirmPassword = ref('')
const error = ref('')
const { register } = useAuth()

async function handleRegister() {
  error.value = ''
  if (password.value !== confirmPassword.value) {
    error.value = '两次密码不一致'
    return
  }
  try {
    await register(name.value, email.value, password.value)
  } catch (e: any) {
    error.value = e.data?.message || '注册失败'
  }
}
</script>

<template>
  <div class="max-w-md mx-auto mt-12">
    <h1 class="text-2xl font-bold mb-6 text-center">注册</h1>
    <p v-if="error" class="text-red-500 text-center mb-4">{{ error }}</p>
    <form class="space-y-4" @submit.prevent="handleRegister">
      <AppInput v-model="name" label="用户名" placeholder="你的名字" />
      <AppInput v-model="email" type="email" label="邮箱" placeholder="your@email.com" />
      <AppInput v-model="password" type="password" label="密码" placeholder="******" />
      <AppInput v-model="confirmPassword" type="password" label="确认密码" placeholder="******" />
      <AppButton class="w-full" type="submit">注册</AppButton>
    </form>
    <p class="text-center mt-4 text-sm text-gray-500">
      已有账号？<NuxtLink to="/login" class="text-green-500 hover:underline">登录</NuxtLink>
    </p>
  </div>
</template>
```

### 7.5 404 页面

```vue
<!-- app/pages/[...slug].vue -->
<script setup lang="ts">
const route = useRoute()
</script>

<template>
  <div class="text-center py-24">
    <h1 class="text-6xl font-bold text-gray-200 mb-4">404</h1>
    <p class="text-xl text-gray-500 mb-8">
      页面 "{{ route.params.slug }}" 不存在
    </p>
    <NuxtLink to="/" class="text-green-500 hover:underline">
      返回首页
    </NuxtLink>
  </div>
</template>
```

---

## 8. Pinia 状态管理

```ts
// app/stores/user.ts
export const useUserStore = defineStore('user', () => {
  const user = ref<User | null>(null)
  const token = useCookie('token')

  const isLoggedIn = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.role === 'admin')

  async function login(email: string, password: string) {
    const result = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    })
    token.value = result.token
    user.value = result.user
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      user.value = await $fetch('/api/auth/me')
    } catch {
      user.value = null
      token.value = null
    }
  }

  function logout() {
    user.value = null
    token.value = null
    navigateTo('/login')
  }

  // SSR 初始化
  if (import.meta.server && token.value) {
    fetchUser()
  }

  return { user, token, isLoggedIn, isAdmin, login, fetchUser, logout }
})
```

---

## 9. API 端点汇总

| 方法 | 路径 | 认证 | 说明 | 文档章节 |
|------|------|------|------|---------|
| POST | `/api/auth/login` | 否 | 用户登录 | 14 认证 |
| POST | `/api/auth/register` | 否 | 用户注册 | 14 认证 |
| GET | `/api/auth/me` | 是 | 获取当前用户 | 14 认证 |
| GET | `/api/posts` | 否 | 文章列表（分页/搜索/排序） | 8 数据获取 |
| POST | `/api/posts` | 是 | 创建文章 | 10 服务端路由 |
| GET | `/api/posts/:id` | 否 | 文章详情 | 10 服务端路由 |
| PUT | `/api/posts/:id` | 是 | 更新文章（仅作者/管理员） | 10 服务端路由 |
| DELETE | `/api/posts/:id` | 是 | 删除文章（级联删除评论） | 10 服务端路由 |
| GET | `/api/comments/posts/:postId` | 否 | 文章评论列表 | 10 服务端路由 |
| POST | `/api/comments` | 是 | 发表评论 | 10 服务端路由 |
| GET | `/api/health` | 否 | 健康检查 | 17 部署 |
| GET | `/api/export/posts` | 否 | CSV 导出 | 18 最佳实践 |

### curl 示例

```bash
# 注册
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"测试用户","email":"test@example.com","password":"123456"}'

# 登录
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}'

# 获取文章列表
curl http://localhost:3000/api/posts?page=1&limit=10

# 搜索文章
curl "http://localhost:3000/api/posts?search=nuxt&tag=vue&sortBy=createdAt&sortOrder=desc"

# 获取单篇文章
curl http://localhost:3000/api/posts/1

# 创建文章（需要认证）
curl -X POST http://localhost:3000/api/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"title":"新文章","content":"正文内容","tags":["nuxt","vue"]}'

# 更新文章
curl -X PUT http://localhost:3000/api/posts/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"title":"更新后的标题","published":true}'

# 删除文章
curl -X DELETE http://localhost:3000/api/posts/1 \
  -H "Authorization: Bearer <TOKEN>"

# 发表评论
curl -X POST http://localhost:3000/api/comments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"content":"好文章！","postId":1}'

# 获取评论
curl http://localhost:3000/api/comments/posts/1

# 健康检查
curl http://localhost:3000/api/health

# CSV 导出
curl http://localhost:3000/api/export/posts -o posts.csv

# 批量删除文章
curl -X DELETE http://localhost:3000/api/posts/batch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"ids":[1,2,3]}'

# 批量创建文章
curl -X POST http://localhost:3000/api/posts/batch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"posts":[{"title":"文章A","content":"内容A"},{"title":"文章B","content":"内容B"}]}'

# 获取当前用户信息
curl http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer <TOKEN>"
```

---

## 10. 业务模式库

### 10.1 数据导出（CSV 流式）

已在上文 `server/api/export/posts.get.ts` 实现，使用 `setHeader` 设置 `Content-Disposition` 触发浏览器下载。

### 10.2 审计日志

```ts
// server/utils/audit.ts
interface AuditEntry {
  action: string
  userId: number
  resource: string
  resourceId: number | string
  timestamp: string
  details?: Record<string, any>
}

const auditLog: AuditEntry[] = []

export function logAudit(entry: Omit<AuditEntry, 'timestamp'>) {
  auditLog.push({
    ...entry,
    timestamp: new Date().toISOString(),
  })
  // 生产环境应写入数据库或发送到日志服务
}
```

### 10.3 批量操作

```ts
// server/api/posts/batch.delete.ts
export default defineEventHandler(async (event) => {
  const auth = requireAuth(event)
  const { ids } = await readBody(event)

  if (!Array.isArray(ids) || ids.length === 0) {
    throw createError({ statusCode: 400, statusMessage: '请提供要删除的文章 ID 列表' })
  }

  const deleted: number[] = []
  const forbidden: number[] = []

  for (const id of ids) {
    const index = posts.findIndex(p => p.id === id)
    if (index === -1) continue

    if (posts[index].authorId !== auth.userId && auth.role !== 'admin') {
      forbidden.push(id)
      continue
    }

    posts.splice(index, 1)
    deleted.push(id)
  }

  return { deleted, forbidden, success: deleted.length }
})
```

### 10.4 速率限制（API 限流）

已在上文 `server/utils/rate-limit.ts` 实现，基于 IP 的内存限流器。

### 10.5 缓存策略（Redis 就绪）

```ts
// server/utils/cache.ts
const cache = new Map<string, { data: any; expiresAt: number }>()

export function getCached<T>(key: string): T | null {
  const entry = cache.get(key)
  if (!entry || Date.now() > entry.expiresAt) {
    cache.delete(key)
    return null
  }
  return entry.data as T
}

export function setCache(key: string, data: any, ttlSeconds: number) {
  cache.set(key, {
    data,
    expiresAt: Date.now() + ttlSeconds * 1000,
  })
}

export function clearCache(pattern?: string) {
  if (!pattern) {
    cache.clear()
    return
  }
  for (const key of cache.keys()) {
    if (key.includes(pattern)) {
      cache.delete(key)
    }
  }
}
```

### 10.6 软删除

```ts
// 扩展 Post 类型：添加 deleted 字段
// shared/types.ts 中 Post 结构扩展 deleted?: boolean

// server/api/posts/[id]/soft-delete.ts
export default defineEventHandler((event) => {
  const auth = requireAuth(event)
  const id = Number(getRouterParam(event, 'id'))
  const post = posts.find(p => p.id === id)

  if (!post) {
    throw createError({ statusCode: 404, statusMessage: '文章不存在' })
  }

  if (post.authorId !== auth.userId && auth.role !== 'admin') {
    throw createError({ statusCode: 403, statusMessage: '无权操作' })
  }

  // 软删除不实际移除
  ;(post as any).deleted = true
  ;(post as any).deletedAt = new Date().toISOString()

  return { success: true }
})
```

### 10.7 事务性批量操作

批量创建文章并返回结果：

```ts
// server/api/posts/batch.post.ts
export default defineEventHandler(async (event) => {
  const auth = requireAuth(event)
  const { posts: newPosts } = await readBody(event)

  if (!Array.isArray(newPosts)) {
    throw createError({ statusCode: 400, statusMessage: '请提供文章列表' })
  }

  const created: Post[] = []
  const errors: { index: number; message: string }[] = []

  for (const [index, data] of newPosts.entries()) {
    if (!data.title || !data.content) {
      errors.push({ index, message: '标题和内容不能为空' })
      continue
    }

    const post: Post = {
      id: nextId.post++,
      title: data.title,
      slug: slugify(data.title),
      content: data.content,
      excerpt: data.excerpt || data.content.slice(0, 200),
      published: data.published ?? true,
      tags: data.tags || [],
      authorId: auth.userId,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    }
    posts.unshift(post)
    created.push(post)
  }

  return { created, errors, success: created.length }
})
```

### 10.8 Pre-commit 钩子

```bash
pnpm add -D simple-git-hooks lint-staged
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,vue}": ["eslint --fix"],
    "*.vue": ["vue-tsc --noEmit"]
  },
  "simple-git-hooks": {
    "pre-commit": "pnpm lint-staged && pnpm typecheck"
  }
}
```

---

## 11. 工具函数速查

### 11.1 日期格式化

```ts
import { format, formatDistanceToNow } from 'date-fns'
// 或使用 Intl API（不需要额外依赖）

export function formatDateTime(date: string): string {
  return new Date(date).toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  })
}
```

### 11.2 文本处理

```ts
export function truncate(text: string, maxLength: number): string {
  if (text.length <= maxLength) return text
  return text.slice(0, maxLength) + '...'
}

export function stripHtml(html: string): string {
  return html.replace(/<[^>]*>/g, '')
}

export function pluralize(count: number, singular: string, plural?: string): string {
  return count === 1 ? singular : (plural || `${singular}s`)
}
```

### 11.3 类型守卫

```ts
export function isApiError(error: unknown): error is ApiError {
  return typeof error === 'object' && error !== null && 'statusCode' in error
}

export function isPost(obj: unknown): obj is Post {
  return typeof obj === 'object' && obj !== null && 'title' in obj && 'content' in obj
}
```

---

## 12. 参考资源

- [Nuxt 4 文档](https://nuxt.com/docs)
- [Vue 3 组合式 API](https://vuejs.org/guide/reusability/composables)
- [Pinia 文档](https://pinia.vuejs.org/)
- [UnoCSS 预设](https://unocss.dev/presets/)
- [Vitest 配置](https://vitest.dev/config/)
- [Playwright 入门](https://playwright.dev/docs/intro)
- [VueUse 函数库](https://vueuse.org/functions)
- [TypeScript 工具类型](https://www.typescriptlang.org/docs/handbook/utility-types.html)
