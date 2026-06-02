# Vue 3 + TypeScript + Nuxt 4 完整学习文档

> **版本锁定**
>
> ```ini
> NODE_VERSION=22.x
> PNPM_VERSION=10.x
> NUXT_VERSION=4.0.0
> VUE_VERSION=3.5.x
> TYPESCRIPT_VERSION=5.8.x
> ```

---

## 1. 环境与工具链

### 1.1 为什么需要这套工具链

构建现代前端应用需要多个工具的配合：包管理器管理依赖、框架提供开发范式、语言提供类型安全。Nuxt 4 以 Vue 3 为核心，Nitro 为服务端引擎，组合成全栈开发框架。

### 1.2 环境要求

```bash
# 检查 Node.js 版本（推荐 22.x）
node --version   # ≥ 22.0.0

# 安装 pnpm（推荐）
corepack enable
corepack prepare pnpm@latest --activate
pnpm --version   # 10.x

# 安装 Nuxt CLI（可选，npx 方式更推荐）
npx nuxi@latest --version
```

### 1.3 创建项目

```bash
pnpm dlx nuxi@latest init my-nuxt-app
cd my-nuxt-app
pnpm install
```

默认生成的 `nuxt.config.ts`：

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },
})
```

### 1.4 目录结构（Nuxt 4）

Nuxt 4 的核心变化是 `app/` 目录作为源码根目录：

```
my-nuxt-app/
├── app/
│   ├── app.vue              # 根组件
│   ├── app.config.ts        # 运行时配置
│   ├── error.vue            # 错误页面
│   ├── components/          # 自动导入的组件
│   ├── composables/         # 自动导入的组合式函数
│   ├── layouts/             # 布局组件
│   ├── middleware/          # 路由中间件
│   ├── pages/               # 文件系统路由
│   ├── plugins/             # Vue 插件
│   └── utils/               # 自动导入的工具函数
├── server/                  # 服务端代码（Nitro）
│   ├── api/                 # API 路由
│   ├── middleware/          # 服务端中间件
│   ├── plugins/             # Nitro 插件
│   └── utils/               # 服务端工具（自动导入）
├── public/                  # 静态资源
├── nuxt.config.ts           # Nuxt 配置
├── package.json
└── tsconfig.json
```

**关键说明：**
- `app/` 目录是 Nuxt 4 新增的默认源码根目录
- `server/` 目录提供全栈 API 能力
- `public/` 下的文件直接暴露在根路径

> **💡 Tip:** 如果你是从 Nuxt 3 迁移，nuxt.config 中添加 `srcDir: '.'` 可恢复旧结构。

### 1.5 开发命令

| 命令 | 用途 |
|------|------|
| `pnpm dev` | 启动开发服务器（HMR） |
| `pnpm build` | 构建生产版本 |
| `pnpm generate` | 预渲染静态站点 |
| `pnpm preview` | 预览生产构建 |

```bash
# 启动开发
pnpm dev

# 构建并预览
pnpm build
pnpm preview
```

---

## 2. Nuxt 4 核心概念

### 2.1 Nuxt 是什么

Nuxt 是一个基于 Vue 3 的全栈框架，核心目标是在不牺牲开发体验的前提下提供**服务端渲染（SSR）**。

与纯 Vue 3 SPA 的区别：

| 能力 | Vue 3 SPA | Nuxt 4 |
|------|-----------|--------|
| SSR | 需手动配置 | 内置 |
| 路由 | 需 vue-router | 文件系统路由 |
| 数据获取 | 自行处理 | useFetch 自动去重 |
| 服务端 | 无 | Nitro 引擎 |
| SEO | 差（纯客户端） | 优秀（HTML 直出） |

### 2.2 渲染模式

Nuxt 支持三种渲染模式，可以在 `nuxt.config.ts` 中配置：

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  // 默认：SSR（服务器端渲染）
  ssr: true,

  // 可选：CSR（纯客户端渲染）
  // ssr: false,
})
```

**SSR 的工作流程：**

```
请求 → Nitro 服务端 → 渲染 Vue 组件 → 产出 HTML → 发送给浏览器
                                                      ↓
                                              浏览器加载 JS → 水合（Hydration）→ 交互
```

**混合渲染（Hybrid）**：不同路由不同模式：

```ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },              // 静态生成
    '/blog/**': { isr: 3600 },             // ISR：1小时缓存
    '/admin/**': { ssr: false },           // SPA
    '/api/**': { cors: true },             // API
  },
})
```

> **💡 Tip:** ISR（增量静态再生）让动态页面也能获得静态速度——首次请求后缓存，后台异步重新生成。

### 2.3 app.vue — 应用的根

```vue
<!-- app/app.vue -->
<script setup lang="ts">
useSeoMeta({
  title: '我的 Nuxt 应用',
  description: '基于 Nuxt 4 的全栈应用',
})
</script>

<template>
  <div>
    <AppHeader />
    <NuxtPage />
    <AppFooter />
  </div>
</template>
```

`<NuxtPage />` 是路由出口，相当于 Vue Router 的 `<RouterView />`。

### 2.4 应用生命周期

```
Nuxt 启动
  │
  ├─ ① Nuxt 插件 (plugins/): 初始化
  ├─ ② 中间件 (middleware/): 路由守卫
  ├─ ③ 页面组件 setup(): 数据获取
  ├─ ④ 服务端渲染: HTML 产出
  ├─ ⑤ 客户端水合 (Hydration)
  └─ ⑥ 客户端交互
```

**关键说明：**
- Vue 的 `onMounted` 只在客户端执行
- `useFetch` 在服务端 + 客户端各执行一次（但数据会序列化传递，避免重复请求）
- `import.meta.server` / `import.meta.client` 可判断执行环境

---

## 3. Vue 3 基础强化

### 3.1 Composition API 与 script setup

Nuxt 4 默认使用 Composition API + `<script setup lang="ts">`。

```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

watch(count, (newVal) => {
  console.log('count changed:', newVal)
})

onMounted(() => {
  console.log('mounted on client')
})
</script>

<template>
  <button @click="count++">
    {{ count }} × 2 = {{ doubled }}
  </button>
</template>
```

### 3.2 响应式核心

```vue
<script setup lang="ts">
import { ref, reactive, shallowRef, computed, watch } from 'vue'

// ref: 基本响应式
const name = ref('Nuxt')
const user = ref<User | null>(null)

// reactive: 对象响应式
const state = reactive({ count: 0, list: [] as string[] })

// shallowRef: 浅层响应（性能优化）
const largeData = shallowRef({ items: new Array(10000) })

// computed: 派生状态
const greeting = computed(() => `Hello, ${name.value}!`)
</script>
```

**Key Points:**
- `ref` 用于基本类型和需要重新赋值的场景
- `reactive` 用于对象内部属性变更
- `shallowRef` 对大数组/对象做性能优化

> **💡 Tip:** 在 `<script setup>` 中，ref 在模板中自动解包，不需要 `.value`。

### 3.3 Props 与 Emits

```vue
<script setup lang="ts">
// 定义 Props（类型方式）
const props = defineProps<{
  title: string
  count?: number
  items: string[]
}>()

// 定义 Emits
const emit = defineEmits<{
  update: [value: string]
  delete: [id: number]
}>()

// v-model（双向绑定）
const model = defineModel<string>()

function handleClick() {
  emit('update', 'new value')
}
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <input v-model="model" />
    <button @click="handleClick">Update</button>
  </div>
</template>
```

### 3.4 TypeScript 集成

Nuxt 4 提供完整的类型支持：

```ts
// 自动生成的类型
// .nuxt/nuxt.d.ts 包含自动导入的类型声明

// 定义接口
interface Post {
  id: number
  title: string
  content: string
  createdAt: Date
  author: {
    name: string
    avatar?: string
  }
}

// 在组件中使用
const posts = ref<Post[]>([])
```

| 工具 | 用途 |
|------|------|
| `npx nuxi typecheck` | 运行 vue-tsc 类型检查 |
| `definePageMeta` | 页面元数据（自动类型推导） |
| `useFetch<Post[]>` | 泛型数据获取 |

---

## 4. 自动导入机制

### 4.1 为什么要自动导入

手动 import 每个组件和工具函数会带来大量样板代码。Nuxt 的自动导入机制让你专注于业务逻辑。

### 4.2 组件自动导入

`app/components/` 下的 Vue 文件会自动注册：

```
app/components/
├── Button.vue           → <Button />
├── Card.vue             → <Card />
├── base/
│   └── Input.vue        → <BaseInput />
├── ui/
│   ├── Modal.vue        → <UiModal />
│   └── Table.vue        → <UiTable />
```

使用方式：

```vue
<template>
  <div>
    <Button>点击</Button>
    <BaseInput v-model="search" />
    <UiModal :visible="showModal" @close="showModal = false" />
  </div>
</template>
```

**懒加载**：前缀 `Lazy` 实现动态导入：

```vue
<template>
  <LazyHeavyChart v-if="showChart" />
  <!-- 只有当 showChart 为 true 时才会加载 HeavyChart 组件 -->
</template>
```

**客户端/服务端专属**：

```
components/
├── Comments.client.vue     → 仅在客户端渲染
└── Dashboard.server.vue    → 仅在服务端渲染
```

### 4.3 组合式函数自动导入

`app/composables/` 下的导出函数自动可用：

```ts
// app/composables/useCounter.ts
export function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  const decrement = () => count.value--

  return { count, increment, decrement }
}
```

在页面中直接使用：

```vue
<script setup lang="ts">
const { count, increment } = useCounter(10)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">+</button>
  </div>
</template>
```

> **💡 Tip:** 只有顶层导出的函数才会自动导入。嵌套目录需要通过 `composables/index.ts` 重新导出。

### 4.4 工具函数自动导入

`app/utils/` 下导出的纯函数自动可用：

```ts
// app/utils/format.ts
export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('zh-CN').format(date)
}

export function slugify(text: string): string {
  return text.toLowerCase().replace(/\s+/g, '-')
}
```

```vue
<script setup lang="ts">
const date = formatDate(new Date())
const slug = slugify('Hello World') // → 'hello-world'
</script>
```

### 4.5 自动导入规则总结

| 目录 | 导入内容 | 使用方式 |
|------|---------|---------|
| `components/` | Vue 组件 | `<ComponentName />` |
| `composables/` | 组合式函数 | `useXxx()` |
| `utils/` | 工具函数 | `functionName()` |
| `server/utils/` | 服务端工具 | `useXxx()`（服务端） |

---

## 5. 文件系统路由

### 5.1 为什么用文件系统路由

传统 Vue Router 需要手动维护路由配置表。文件系统路由通过目录结构自动生成路由——无需配置，文件即路由。

### 5.2 基本路由

```
app/pages/
├── index.vue          → /
├── about.vue          → /about
├── blog/
│   ├── index.vue      → /blog
│   └── [slug].vue     → /blog/:slug
└── users/
    └── [id]/
        └── profile.vue → /users/:id/profile
```

```vue
<!-- app/pages/blog/[slug].vue -->
<script setup lang="ts">
const route = useRoute()
// /blog/hello-world → route.params.slug = 'hello-world'
const slug = route.params.slug as string
</script>

<template>
  <article>
    <h1>Blog Post: {{ slug }}</h1>
  </article>
</template>
```

### 5.3 导航

**声明式导航**：

```vue
<template>
  <nav>
    <NuxtLink to="/">首页</NuxtLink>
    <NuxtLink to="/about">关于</NuxtLink>
    <NuxtLink :to="{ name: 'blog-slug', params: { slug: 'hello' } }">
      文章
    </NuxtLink>
  </nav>
</template>
```

**编程式导航**：

```vue
<script setup lang="ts">
const router = useRouter()

function goToPost(slug: string) {
  // 方式 1：navigateTo（推荐）
  navigateTo(`/blog/${slug}`)

  // 方式 2：router.push
  router.push({ name: 'blog-slug', params: { slug } })
}
</script>
```

> **💡 Tip:** `<NuxtLink>` 在视口内自动预取目标页面，提升用户体验。

### 5.4 布局系统

布局在 `app/layouts/` 中定义：

```vue
<!-- app/layouts/default.vue -->
<template>
  <div>
    <AppHeader />
    <slot />
    <AppFooter />
  </div>
</template>
```

```vue
<!-- app/layouts/admin.vue -->
<template>
  <div class="admin-layout">
    <aside>
      <AdminSidebar />
    </aside>
    <main>
      <slot />
    </main>
  </div>
</template>
```

页面选择布局：

```vue
<script setup lang="ts">
definePageMeta({
  layout: 'admin',        // 使用 admin 布局
  // layout: false        // 不使用布局
})
</script>
```

```vue
<!-- app/error.vue 错误页面 -->
<script setup lang="ts">
const props = defineProps<{
  error: { statusCode: number; statusMessage?: string }
}>()
</script>

<template>
  <div>
    <h1>{{ error.statusCode }}</h1>
    <p>{{ error.statusMessage || '页面未找到' }}</p>
    <NuxtLink to="/">返回首页</NuxtLink>
  </div>
</template>
```

### 5.5 路由验证

```vue
<script setup lang="ts">
definePageMeta({
  validate: (route) => {
    // 验证路由参数
    return /^\d+$/.test(route.params.id as string)
  },
})
</script>
```

返回 `true` 继续导航，返回 `false` 显示 404。

### 5.6 路由中间件

```ts
// app/middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const token = useCookie('token')

  if (!token.value && to.path !== '/login') {
    return navigateTo('/login')
  }
})
```

```ts
// app/middleware/logging.global.ts（全局中间件）
export default defineNuxtRouteMiddleware((to, from) => {
  console.log(`[Navigation] ${from.path} → ${to.path}`)
})
```

在页面中使用：

```vue
<script setup lang="ts">
definePageMeta({
  middleware: ['auth'],  // 可以是数组
})
</script>
```

| 文件后缀 | 行为 |
|---------|------|
| `.ts` | 指定页面手动引用 |
| `.global.ts` | 所有路由自动执行 |

---

## 6. 组件系统

### 6.1 组件基础

Nuxt 4 中组件自动导入，无需手动注册。

```vue
<!-- app/components/MyButton.vue -->
<script setup lang="ts">
defineProps<{
  variant?: 'primary' | 'secondary'
}>()
</script>

<template>
  <button :class="[`btn-${variant || 'primary'}`]">
    <slot />
  </button>
</template>
```

```vue
<!-- 在页面中使用 -->
<template>
  <MyButton variant="primary">保存</MyButton>
  <MyButton variant="secondary">取消</MyButton>
</template>
```

### 6.2 内置组件

```vue
<template>
  <!-- ClientOnly: 仅在客户端渲染 -->
  <ClientOnly>
    <ClientChart />
    <template #fallback>
      <div>加载中...</div>
    </template>
  </ClientOnly>

  <!-- Teleport: 传送门 -->
  <Teleport to="body">
    <Modal v-if="showModal" />
  </Teleport>
</template>
```

### 6.3 事件与 v-model

```vue
<!-- app/components/SearchInput.vue -->
<script setup lang="ts">
const model = defineModel<string>({ required: true })

const emit = defineEmits<{
  search: [query: string]
}>()

function onSearch() {
  emit('search', model.value)
}
</script>

<template>
  <div class="search-input">
    <input v-model="model" @keyup.enter="onSearch" />
    <button @click="onSearch">搜索</button>
  </div>
</template>
```

```vue
<!-- 使用 -->
<template>
  <SearchInput v-model="query" @search="handleSearch" />
</template>
```

### 6.4 组件命名约定与性能

```
components/
├── TheHeader.vue         → <TheHeader />（唯一实例）
├── base/
│   ├── Button.vue        → <BaseButton />
│   └── Card.vue          → <BaseCard />
└── ui/
    └── Table.vue         → <UiTable />
```

| 模式 | 用法 | 说明 |
|------|------|------|
| `TheXxx` | 全局唯一 | 如导航栏、页脚 |
| `BaseXxx` | 基础组件 | 如按钮、输入框 |
| `UiXxx` | UI 组件 | 如表格、弹窗 |
| `LazyXxx` | 懒加载 | 按需加载大型组件 |

> **💡 Tip:** 对于不在页面视口内的组件，使用 `Lazy` 前缀减少首屏 JS 大小。

---

## 7. 组合式函数 (Composables)

### 7.1 为什么需要组合式函数

Vue 3 的 Options API 在组件逻辑复杂时会面临"混入不清晰、命名冲突、类型推导差"等问题。Composables 将状态逻辑提取到独立函数中，实现关注点分离。

### 7.2 基础组合式函数

```ts
// app/composables/useLocalStorage.ts
export function useLocalStorage<T>(key: string, defaultValue: T) {
  const data = ref<T>(defaultValue)

  if (import.meta.client) {
    const stored = localStorage.getItem(key)
    if (stored) {
      data.value = JSON.parse(stored)
    }
  }

  watch(data, (newVal) => {
    if (import.meta.client) {
      localStorage.setItem(key, JSON.stringify(newVal))
    }
  }, { deep: true })

  return data
}
```

```vue
<script setup lang="ts">
// 自动导入，直接使用
const theme = useLocalStorage('theme', 'light')
</script>
```

### 7.3 处理异步状态

```ts
// app/composables/useAsync.ts
export function useAsync<T>(fetcher: () => Promise<T>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      data.value = await fetcher()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  return { data, error, loading, execute }
}
```

### 7.4 服务端安全的组合式函数

SSR 环境下需要避免使用浏览器 API：

```ts
// app/composables/useMediaQuery.ts
export function useMediaQuery(query: string) {
  const matches = ref(false)

  if (import.meta.client) {
    const mql = window.matchMedia(query)
    matches.value = mql.matches

    const handler = (e: MediaQueryListEvent) => {
      matches.value = e.matches
    }
    mql.addEventListener('change', handler)
    onUnmounted(() => mql.removeEventListener('change', handler))
  }

  return matches
}
```

### 7.5 VueUse 集成

VueUse 是一个经过生产验证的组合式函数库：

```bash
pnpm add @vueuse/core
```

```vue
<script setup lang="ts">
import { useLocalStorage, useDark, useToggle, useDebounceFn } from '@vueuse/core'

const theme = useLocalStorage('theme', 'light')
const isDark = useDark()
const toggleDark = useToggle(isDark)

const save = useDebounceFn((value: string) => {
  console.log('Debounced save:', value)
}, 500)
</script>
```

| 场景 | 推荐函数 |
|------|---------|
| 本地存储 | `useLocalStorage` / `useStorage` |
| 暗色模式 | `useDark` / `useColorMode` |
| 防抖/节流 | `useDebounceFn` / `useThrottleFn` |
| 浏览器 API | `useEventListener` / `useMediaQuery` |
| 网络状态 | `useOnline` / `useFetch` |

---

## 8. 数据获取

### 8.1 为什么需要专门的 API

在 SSR 中如果直接使用 `fetch()` 或 `axios`，数据会在服务端和客户端各请求一次，造成浪费。Nuxt 的 `useFetch` / `useAsyncData` 会自动去重和序列化。

### 8.2 useFetch

```vue
<script setup lang="ts">
interface Post {
  id: number
  title: string
  content: string
}

const { data: posts, status, error, refresh } = await useFetch<Post[]>('/api/posts', {
  // 查询参数
  query: { page: 1, limit: 10 },
  // 只提取需要的字段
  pick: ['id', 'title'],
  // 转换响应
  transform: (posts) => posts.map(p => ({ ...p, slug: slugify(p.title) })),
})
</script>

<template>
  <div v-if="status === 'pending'">加载中...</div>
  <div v-else-if="error">错误: {{ error.message }}</div>
  <div v-else>
    <article v-for="post in posts" :key="post.id">
      <h2>{{ post.title }}</h2>
    </article>
  </div>
</template>
```

### 8.3 响应式参数

```vue
<script setup lang="ts">
const page = ref(1)
const category = ref('tech')

// 参数变化自动重新获取
const { data } = await useFetch('/api/posts', {
  query: { page, category },
})

// URL 响应式
const { data: post } = await useFetch(() => `/api/posts/${id.value}`)
</script>
```

### 8.4 useAsyncData

用于封装自定义异步逻辑：

```vue
<script setup lang="ts">
const { data, error } = await useAsyncData('homepage', async () => {
  const [featured, recent] = await Promise.all([
    $fetch('/api/posts/featured'),
    $fetch('/api/posts/recent'),
  ])
  return { featured, recent }
})
</script>
```

### 8.5 $fetch — 客户端事件

在交互事件中使用（不要用于初始数据获取）：

```vue
<script setup lang="ts">
async function submitForm() {
  try {
    const result = await $fetch('/api/contact', {
      method: 'POST',
      body: { name: '张三', email: 'test@example.com' },
    })
    showSuccess.value = true
  } catch (e) {
    showError.value = true
  }
}
</script>
```

### 8.6 缓存与共享

```vue
<script setup lang="ts">
// 页面 A
const { data } = await useFetch('/api/user', { key: 'current-user' })

// 页面 B — 使用缓存数据
const { data: user } = useNuxtData('current-user')

// 全局刷新
await refreshNuxtData('current-user')

// 清除缓存
clearNuxtData('current-user')
</script>
```

### 8.7 拦截器

```vue
<script setup lang="ts">
const { data } = await useFetch('/api/secure', {
  onRequest({ options }) {
    options.headers.set('Authorization', `Bearer ${token.value}`)
  },
  onResponseError({ response }) {
    if (response.status === 401) {
      navigateTo('/login')
    }
  },
})
</script>
```

### 8.8 延迟加载

不阻塞页面导航：

```vue
<script setup lang="ts">
const { data, status } = await useLazyFetch('/api/heavy-data')

// 等价于：
// const { data, status } = await useFetch('/api/heavy-data', { lazy: true })
</script>

<template>
  <div v-if="status === 'pending'">加载骨架屏...</div>
  <div v-else>{{ data }}</div>
</template>
```

| Composable | 特点 |
|------------|------|
| `useFetch` | 首选，集成了 SSR 数据去重 |
| `useAsyncData` | 自定义异步逻辑的封装 |
| `useLazyFetch` | 异步获取，不阻塞导航 |
| `$fetch` | 交互事件中的请求 |
| `useNuxtData` | 访问已缓存的数据 |

> **💡 Tip:** 服务端到客户端的数据传递通过 `payload` 实现，这是 Nuxt 自动处理的。不要直接使用 `fetch` 在 setup 中获取初始数据，否则会双重请求。

---

## 9. 状态管理

### 9.1 为什么需要状态管理

跨组件/跨页面共享状态时，props 层层传递变得不可维护。Nuxt 提供了两层方案：轻量的 `useState` 和全功能的 Pinia。

### 9.2 useState — 轻量级状态

`useState` 是 Nuxt 内置的 SSR 安全状态管理：

```ts
// app/composables/useCounterState.ts
export function useCounterState() {
  return useState('counter', () => 0)
}
```

```vue
<script setup lang="ts">
const counter = useCounterState()

// 读取
console.log(counter.value)

// 修改
counter.value++
</script>
```

**与 ref 的关键区别：**
- `useState` 在服务端和客户端之间**共享状态**
- `useState` 状态会序列化到客户端（SSR hydration）
- `useState` 需要唯一的 key

```vue
<script setup lang="ts">
// ❌ 错误：ref 在 SSR 中无法跨请求隔离
const count = ref(0)

// ✅ 正确：useState 在请求间隔离
const count = useState('count', () => 0)
</script>
```

### 9.3 Pinia — 生产级状态管理

```bash
pnpm add @pinia/nuxt
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@pinia/nuxt'],
})
```

定义 Store：

```ts
// app/stores/user.ts
export const useUserStore = defineStore('user', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  const isLoggedIn = computed(() => !!token.value)

  async function login(email: string, password: string) {
    const result = await $fetch('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    })
    token.value = result.token
    user.value = result.user
  }

  function logout() {
    token.value = null
    user.value = null
    navigateTo('/login')
  }

  return { user, token, isLoggedIn, login, logout }
})
```

在组件中使用：

```vue
<script setup lang="ts">
const store = useUserStore()
const { user, isLoggedIn } = storeToRefs(store)
const { login, logout } = store

async function handleLogin() {
  await login('test@example.com', 'password123')
}
</script>
```

> **💡 Tip:** 使用 `storeToRefs` 解构 state/getters 以保持响应性。actions 可以直接解构。

### 9.4 Pinia 的 SSR 注意事项

```ts
// app/stores/user.ts
export const useUserStore = defineStore('user', () => {
  const user = ref<User | null>(null)

  // ✅ 正确：在 action 中调用 $fetch
  async function fetchUser() {
    user.value = await $fetch('/api/user')
  }

  // ❌ 错误：不要在模块顶层调用
  // fetchUser()

  return { user, fetchUser }
})
```

```vue
<script setup lang="ts">
// ✅ 正确：在 setup 中初始化
const store = useUserStore()
if (import.meta.server) {
  await store.fetchUser()
}
</script>
```

### 9.5 状态管理选型

| 方案 | 适用场景 | 特性 |
|------|---------|------|
| `useState` | 简单跨组件状态 | 内置、轻量、SSR 安全 |
| Pinia | 复杂应用状态 | 完整生态、DevTools、插件 |
| `useCookie` | 持久化到 Cookie | 服务端可读写 |
| `useStorage` (VueUse) | 持久化到 localStorage | 客户端专用 |

---

## 10. 服务端路由 (Server Routes)

### 10.1 为什么需要服务端路由

Nuxt 的 `server/` 目录让你无需单独搭建后端框架即可创建 API 端点。Nitro 引擎自动处理路由、参数解析、响应处理。

### 10.2 基础 API

```ts
// server/api/hello.ts
export default defineEventHandler(() => {
  return { message: 'Hello from Nitro!' }
})
```

访问 `/api/hello` 返回 `{ "message": "Hello from Nitro!" }`。

### 10.3 HTTP 方法

```ts
// server/api/posts.get.ts
export default defineEventHandler(() => {
  return getPosts()
})

// server/api/posts.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  return createPost(body)
})

// server/api/posts/[id].put.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)
  return updatePost(id, body)
})

// server/api/posts/[id].delete.ts
export default defineEventHandler((event) => {
  const id = getRouterParam(event, 'id')
  return deletePost(id)
})
```

### 10.4 请求处理

```ts
// server/api/search.ts
// GET /api/search?q=nuxt&page=1
export default defineEventHandler((event) => {
  const query = getQuery(event)
  // { q: 'nuxt', page: '1' }
  return searchResults(query.q as string, Number(query.page))
})
```

```ts
// server/api/submit.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const headers = getHeaders(event)
  const cookie = getCookie(event, 'session')

  setHeader(event, 'X-Custom', 'value')
  setCookie(event, 'token', 'abc123', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    maxAge: 60 * 60 * 24,
  })

  return { success: true, data: body }
})
```

### 10.5 服务端中间件

在请求到达 API 处理函数之前执行：

```ts
// server/middleware/auth.ts
export default defineEventHandler((event) => {
  const token = getCookie(event, 'token')

  if (token) {
    event.context.user = verifyToken(token)
  }
})
```

```ts
// server/api/profile.ts
export default defineEventHandler((event) => {
  const user = event.context.user

  if (!user) {
    throw createError({
      statusCode: 401,
      statusMessage: '未登录',
    })
  }

  return user
})
```

### 10.6 错误处理

```ts
// server/api/posts/[id].ts
export default defineEventHandler((event) => {
  const id = getRouterParam(event, 'id')
  const post = findPost(id)

  if (!post) {
    throw createError({
      statusCode: 404,
      statusMessage: '文章不存在',
    })
  }

  return post
})
```

### 10.7 服务端工具

`server/utils/` 下的函数自动导入：

```ts
// server/utils/db.ts
interface User {
  id: number
  name: string
  email: string
}

const users: User[] = [
  { id: 1, name: '张三', email: 'zhangsan@example.com' },
  { id: 2, name: '李四', email: 'lisi@example.com' },
]

export function getUsers(): User[] {
  return users
}

export function getUserById(id: number): User | undefined {
  return users.find(u => u.id === id)
}
```

```ts
// server/api/users.ts
export default defineEventHandler(() => {
  return getUsers() // 自动导入
})
```

### 10.8 服务端插件（启动初始化）

```ts
// server/plugins/init.ts
export default defineNitroPlugin((nitroApp) => {
  console.log('Nitro server starting...')

  nitroApp.hooks.hook('request', (event) => {
    event.context.startTime = Date.now()
  })

  nitroApp.hooks.hook('afterResponse', (event) => {
    const duration = Date.now() - (event.context.startTime || Date.now())
    console.log(`${event.path} - ${duration}ms`)
  })
})
```

### 10.9 API 设计总结

| 工具 | 用途 |
|------|------|
| `defineEventHandler` | 定义 API 处理函数 |
| `readBody` | 读取请求体 |
| `getQuery` | 获取查询参数 |
| `getRouterParam` | 获取路由参数 |
| `getCookie` / `setCookie` | Cookie 操作 |
| `getHeader` / `setHeader` | Header 操作 |
| `createError` | 创建 HTTP 错误 |
| `defineNitroPlugin` | 服务端启动插件 |

---

## 11. 错误处理与生命周期

### 11.1 为什么需要分层错误处理

用户应该看到友好的错误提示，而不是白屏或原始异常信息。Nuxt 提供从服务端到客户端的完整错误处理链。

### 11.2 错误页面

```vue
<!-- app/error.vue -->
<script setup lang="ts">
const props = defineProps<{
  error: { statusCode: number; statusMessage?: string; url?: string }
}>()

const handleError = () => clearError({ redirect: '/' })
</script>

<template>
  <div class="error-page">
    <h1>{{ error.statusCode }}</h1>
    <p>{{ error.statusMessage || '发生未知错误' }}</p>
    <button @click="handleError">返回首页</button>
  </div>
</template>
```

### 11.3 组件级错误边界

```vue
<script setup lang="ts">
const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false // 阻止错误继续传播
})
</script>

<template>
  <div>
    <ErrorState v-if="error" :error="error" @retry="error = null" />
    <slot v-else />
  </div>
</template>
```

### 11.4 生命周期钩子

```vue
<script setup lang="ts">
// 页面生命周期
onBeforeMount(() => {})    // 客户端：挂载前
onMounted(() => {})        // 客户端：已挂载
onBeforeUpdate(() => {})   // 更新前
onUpdated(() => {})        // 更新后
onBeforeUnmount(() => {})  // 卸载前
onUnmounted(() => {})      // 已卸载

// 路由生命周期
onBeforeRouteLeave((to, from) => {
  // 离开当前路由前
})

onBeforeRouteUpdate((to, from) => {
  // 路由参数变化但组件复用时
})
</script>
```

**环境判断：**

```vue
<script setup lang="ts">
if (import.meta.server) {
  // 仅在服务端执行
  console.log('Server side')
}

if (import.meta.client) {
  // 仅在客户端执行
  console.log('Client side')
}
</script>
```

---

## 12. 中间件与插件

### 12.1 为什么需要中间件和插件

- **中间件**：在路由导航之前执行逻辑（权限控制、重定向、数据预加载）
- **插件**：在 Vue 应用初始化时注入全局功能（第三方库、全局组件）

### 12.2 路由中间件（复习）

```ts
// app/middleware/admin.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { user } = useUserStore()

  if (to.path.startsWith('/admin') && user.value?.role !== 'admin') {
    return navigateTo('/403')
  }
})
```

执行顺序：全局中间件 → 页面中间件

### 12.3 Nuxt 插件

```ts
// app/plugins/axios.ts
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
        navigateTo('/login')
      }
    },
  })

  return {
    provide: {
      api,
    },
  }
})
```

在组件中使用：

```vue
<script setup lang="ts">
const { $api } = useNuxtApp()
const data = await $api('/users')
</script>
```

### 12.4 插件执行顺序

文件命名前缀控制顺序：

```
plugins/
├── 01.pinia.ts           # 先执行
├── 02.api.ts             # 其次
└── 03.router.ts          # 最后
```

| 插件后缀 | 执行环境 |
|---------|---------|
| `.client.ts` | 仅在客户端 |
| `.server.ts` | 仅在服务端 |
| `.ts` | 两端都执行 |

---

## 13. 样式方案

### 13.1 为什么需要样式方案

Nuxt 4 支持多种样式方案，从原生 CSS 到原子化 CSS，从 CSS 预处理器到 CSS-in-JS。

### 13.2 原生 CSS

```vue
<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1rem;
}
.title {
  font-size: 2rem;
  color: #333;
}
</style>
```

### 13.3 UnoCSS — 原子化 CSS

```bash
pnpm add @unocss/nuxt
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@unocss/nuxt'],
  unocss: {
    presets: [
      presetUno(),
      presetIcons(),
      presetAttributify(),
    ],
  },
})
```

```vue
<template>
  <div class="flex items-center gap-4 p-6 bg-gray-100 rounded-lg">
    <div class="i-carbon-sun text-2xl text-yellow-500" />
    <span class="text-lg font-bold text-gray-800">Hello UnoCSS</span>
  </div>
</template>
```

### 13.4 CSS 预处理器

```bash
pnpm add -D sass
```

```vue
<style scoped lang="scss">
$primary: #00dc82;

.card {
  background: $primary;
  &:hover {
    opacity: 0.8;
  }
}
</style>
```

### 13.5 全局样式

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  css: ['~/assets/css/main.css'],
})
```

```css
/* app/assets/css/main.css */
body {
  font-family: 'Inter', sans-serif;
  line-height: 1.6;
}
```

### 13.6 样式方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Scoped CSS | 组件内样式 | 无副作用 | 跨组件复用困难 |
| UnoCSS | 快速开发 | 零构建、按需生成 | 类名记忆成本 |
| SCSS | 复杂样式 | 变量、嵌套、Mixin | 需要预处理 |
| CSS Modules | 模块化样式 | 类型安全 | 配置较复杂 |

---

## 14. 认证与安全

### 14.1 为什么需要认证系统

认证是 Web 应用的基本需求。Nuxt 4 的全栈特性让你可以在同一项目中处理前后端认证。

### 14.2 JWT 认证流程

```
客户端                服务端
  │                    │
  ├── POST /login ──→  │
  │                    ├── 验证凭据
  │                    ├── 生成 JWT
  │  ←── { token } ───┤
  │                    │
  ├── GET /profile ──→ │
  │  Authorization     ├── 验证 JWT
  │  Bearer <token>    ├── 返回用户数据
  │  ←── { user } ────┤
```

### 14.3 服务端实现

```ts
// server/utils/auth.ts
import jwt from 'jsonwebtoken'

const SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-production'

export interface JwtPayload {
  userId: number
  email: string
  role: 'user' | 'admin'
}

export function generateToken(payload: JwtPayload): string {
  return jwt.sign(payload, SECRET, { expiresIn: '7d' })
}

export function verifyToken(token: string): JwtPayload | null {
  try {
    return jwt.verify(token, SECRET) as JwtPayload
  } catch {
    return null
  }
}
```

```bash
pnpm add jsonwebtoken
pnpm add -D @types/jsonwebtoken
```

```ts
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const { email, password } = await readBody(event)

  // 模拟验证（实际应从数据库查询）
  if (email !== 'admin@example.com' || password !== 'password') {
    throw createError({
      statusCode: 401,
      statusMessage: '邮箱或密码错误',
    })
  }

  const token = generateToken({
    userId: 1,
    email,
    role: 'admin',
  })

  setCookie(event, 'token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7, // 7 天
  })

  return { token }
})
```

### 14.4 认证中间件

```ts
// server/middleware/auth.ts
export default defineEventHandler((event) => {
  const token = getCookie(event, 'token') || getHeader(event, 'authorization')?.replace('Bearer ', '')

  if (token) {
    const payload = verifyToken(token)
    if (payload) {
      event.context.auth = payload
    }
  }
})
```

### 14.5 客户端认证状态

```ts
// app/composables/useAuth.ts
export function useAuth() {
  const user = useState<User | null>('auth-user', () => null)
  const token = useCookie('token')

  const isLoggedIn = computed(() => !!token.value)

  async function login(email: string, password: string) {
    const result = await $fetch<{ token: string }>('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    })
    token.value = result.token
    await fetchUser()
  }

  async function fetchUser() {
    if (token.value) {
      try {
        user.value = await $fetch('/api/auth/me')
      } catch {
        token.value = null
        user.value = null
      }
    }
  }

  function logout() {
    token.value = null
    user.value = null
    navigateTo('/login')
  }

  return { user, token, isLoggedIn, login, logout, fetchUser }
}
```

### 14.6 路由守卫

```ts
// app/middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const token = useCookie('token')

  if (!token.value && to.path !== '/login') {
    return navigateTo('/login')
  }

  if (token.value && to.path === '/login') {
    return navigateTo('/dashboard')
  }
})
```

### 14.7 CORS 配置

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/api/**': { cors: true },
  },
  nitro: {
    routeRules: {
      '/api/**': {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        },
      },
    },
  },
})
```

### 14.8 安全最佳实践

| 实践 | 说明 |
|------|------|
| 密码哈希 | 使用 `bcrypt` 或 `argon2`，不要明文存储 |
| HTTP Only Cookie | 防止 XSS 读取 token |
| CSRF Token | 对身份认证后的 POST/PUT/DELETE 请求验证 |
| 输入验证 | 使用 Zod 或 Valibot 验证请求体 |
| Rate Limiting | 防止暴力破解，尤其是登录接口 |
| CSP Header | 内容安全策略防止 XSS 攻击 |

---

## 15. SEO 与性能优化

### 15.1 为什么 SSR 应用也需要 SEO 优化

即使有 SSR，合理的 SEO 配置能显著提升搜索引擎的收录质量和排名。

### 15.2 useHead / useSeoMeta

```vue
<script setup lang="ts">
useSeoMeta({
  title: '我的博客',
  description: '一个基于 Nuxt 4 的现代博客',
  ogTitle: '我的博客',
  ogDescription: '一个基于 Nuxt 4 的现代博客',
  ogImage: '/images/og.png',
  twitterCard: 'summary_large_image',
})

useHead({
  htmlAttrs: {
    lang: 'zh-CN',
  },
  link: [
    { rel: 'icon', href: '/favicon.ico' },
  ],
})
</script>
```

动态 SEO：

```vue
<script setup lang="ts">
const route = useRoute()
const { data: post } = await useFetch(`/api/posts/${route.params.slug}`)

useSeoMeta({
  title: () => post.value?.title || '文章',
  description: () => post.value?.excerpt || '',
})
</script>
```

### 15.3 路由规则与缓存策略

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 静态页面 — 构建时预渲染
    '/': { prerender: true },
    '/about': { prerender: true },

    // ISR — 增量静态再生
    '/blog/**': { isr: 3600 },     // 每小时重新生成
    '/products/**': { swr: true }, // 按需重新生成

    // SPA — 客户端渲染
    '/admin/**': { ssr: false },

    // 自定义缓存
    '/api/**': {
      cors: true,
      headers: {
        'Cache-Control': 's-maxage=60',
      },
    },
  },
})
```

### 15.4 性能优化策略

| 策略 | 方法 | 效果 |
|------|------|------|
| 代码分割 | `Lazy` 组件 + `defineAsyncComponent` | 减少首屏 JS |
| 图片优化 | `<NuxtImg>` + `<NuxtPicture>` | 自动 WebP、响应式 |
| 字体优化 | `@unocss/preset-web-fonts` | 减少字体加载阻塞 |
| 预取 | `<NuxtLink>` 自动预取 | 页面秒开 |
| ISR | `routeRules` 配置 | 动态页面静态速度 |
| 客户端缓存 | `useFetch` 的 key 机制 | 避免重复请求 |

```vue
<template>
  <!-- NuxtImg 自动优化 -->
  <NuxtImg
    src="/images/hero.jpg"
    alt="Hero"
    width="1200"
    height="600"
    loading="lazy"
    format="webp"
  />
</template>
```

---

## 16. 测试

### 16.1 为什么需要测试

自动化测试确保代码修改不破坏已有功能。Nuxt 4 项目推荐 Vitest（单元测试）+ Playwright（E2E 测试）。

### 16.2 Vitest + Vue Test Utils

```bash
pnpm add -D vitest @vue/test-utils jsdom
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/test-utils-module'],
})
```

或使用 `@nuxt/test-utils`：

```bash
pnpm add -D @nuxt/test-utils vitest
```

```ts
// vitest.config.ts
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environment: 'nuxt',
  },
})
```

### 16.3 组合式函数测试

```ts
// app/composables/useCounter.ts
export function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  const decrement = () => count.value--
  return { count, increment, decrement }
}
```

```ts
// tests/composables/useCounter.test.ts
import { describe, it, expect } from 'vitest'

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('should initialize with custom value', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('should increment', () => {
    const { count, increment } = useCounter(0)
    increment()
    expect(count.value).toBe(1)
  })

  it('should decrement', () => {
    const { count, decrement } = useCounter(5)
    decrement()
    expect(count.value).toBe(4)
  })
})
```

### 16.4 组件测试

```vue
<!-- app/components/Greeting.vue -->
<script setup lang="ts">
defineProps<{
  name: string
}>()
</script>

<template>
  <div class="greeting">Hello, {{ name }}!</div>
</template>
```

```ts
// tests/components/Greeting.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import Greeting from '~/components/Greeting.vue'

describe('Greeting', () => {
  it('renders name', () => {
    const wrapper = mount(Greeting, {
      props: { name: 'Nuxt' },
    })
    expect(wrapper.text()).toContain('Hello, Nuxt!')
  })
})
```

### 16.5 E2E 测试（Playwright）

```bash
pnpm add -D @playwright/test
pnpm exec playwright install
```

```ts
// e2e/home.spec.ts
import { test, expect } from '@playwright/test'

test('home page should render', async ({ page }) => {
  await page.goto('/')
  await expect(page.locator('h1')).toContainText('欢迎')
})

test('navigation works', async ({ page }) => {
  await page.goto('/')
  await page.click('text=关于')
  await expect(page).toHaveURL('/about')
})
```

### 16.6 测试策略

| 测试类型 | 工具 | 覆盖范围 | 运行速度 |
|---------|------|---------|---------|
| 单元测试 | Vitest | 组合式函数、工具函数 | 快 |
| 组件测试 | Vue Test Utils | 组件渲染、交互 | 中 |
| E2E 测试 | Playwright | 用户完整流程 | 慢 |

---

## 17. 构建与部署

### 17.1 为什么需要容器化部署

容器化确保开发、测试、生产环境一致。Nuxt 4 的 Nitro 引擎支持多种部署方式。

### 17.2 Docker 多阶段构建

```dockerfile
# Dockerfile
FROM node:22-alpine AS builder

WORKDIR /app
RUN corepack enable
COPY pnpm-lock.yaml package.json ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:22-alpine AS runner

WORKDIR /app
RUN corepack enable
COPY --from=builder /app/.output /app/.output

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NUXT_PUBLIC_API_BASE=https://api.example.com
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

### 17.3 健康检查端点

```ts
// server/api/health.ts
export default defineEventHandler(() => {
  return {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  }
})
```

### 17.4 环境配置

```ini
# .env（开发环境）
NUXT_PUBLIC_API_BASE=/api
NUXT_SESSION_SECRET=dev-secret
```

```ini
# .env.production（生产环境）
NUXT_PUBLIC_API_BASE=https://api.example.com
NUXT_SESSION_SECRET=prod-secret-must-be-strong
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    sessionSecret: '',
    public: {
      apiBase: '/api',
    },
  },
})
```

### 17.5 CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: corepack enable
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: corepack enable
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      # 根据平台部署（示例：Cloudflare Pages）
      - run: npx wrangler pages deploy .output/public
```

### 17.6 部署平台

| 平台 | 预设 | 特点 |
|------|------|------|
| Node.js | 默认 | 自托管服务器 |
| Cloudflare Pages | `nitro.preset: 'cloudflare-pages'` | 边缘网络、免费套餐 |
| Vercel | 自动检测 | Serverless 函数 |
| Netlify | 自动检测 | 边缘函数 |
| Docker | 任意 | 通用容器化 |

---

## 18. 最佳实践与架构

### 18.1 项目结构规范

```
my-app/
├── app/
│   ├── components/
│   │   ├── base/          # 基础 UI 组件
│   │   ├── ui/            # 业务 UI 组件
│   │   └── TheHeader.vue  # 全局组件
│   ├── composables/       # 组合式函数
│   ├── layouts/           # 布局
│   ├── middleware/        # 路由中间件
│   ├── pages/             # 页面路由
│   ├── plugins/           # 插件
│   ├── stores/            # Pinia 状态
│   ├── utils/             # 工具函数
│   ├── app.vue
│   ├── app.config.ts
│   └── error.vue
├── server/
│   ├── api/               # API 路由
│   ├── middleware/         # 服务端中间件
│   ├── plugins/           # Nitro 插件
│   └── utils/             # 服务端工具
├── shared/                # 前后端共享的类型
│   └── types.ts
├── tests/                 # 测试
│   ├── components/
│   ├── composables/
│   └── e2e/
└── nuxt.config.ts
```

### 18.2 组合式函数规范

```ts
// ✅ 好的组合式函数
export function usePagination<T>(
  fetcher: (page: number) => Promise<T[]>,
  initialPage = 1,
) {
  const page = ref(initialPage)
  const data = ref<T[]>([])
  const loading = ref(false)
  const hasMore = ref(true)

  async function loadMore() {
    if (loading.value || !hasMore.value) return
    loading.value = true
    try {
      const items = await fetcher(page.value)
      data.value.push(...items)
      hasMore.value = items.length > 0
      page.value++
    } finally {
      loading.value = false
    }
  }

  function reset() {
    page.value = initialPage
    data.value = []
    hasMore.value = true
  }

  return { data, loading, hasMore, loadMore, reset }
}
```

### 18.3 常见模式

#### 分页

```ts
// server/api/posts.get.ts
export default defineEventHandler((event) => {
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const limit = Number(query.limit) || 10
  const skip = (page - 1) * limit

  return getPosts({ skip, limit, page })
})
```

```vue
<script setup lang="ts">
const page = ref(1)
const { data, status } = await useFetch('/api/posts', {
  query: { page, limit: 10 },
  watch: [page],
})
</script>
```

#### 搜索与排序

```vue
<script setup lang="ts">
const search = ref('')
const sortBy = ref('createdAt')
const sortOrder = ref<'asc' | 'desc'>('desc')

const { data } = await useFetch('/api/posts', {
  query: { search, sortBy, sortOrder },
  watch: [search, sortBy, sortOrder],
})
</script>
```

### 18.4 性能检查清单

- [ ] 大型组件使用 `Lazy` 前缀懒加载
- [ ] `shallowRef` 替代 `ref` 处理大对象
- [ ] 使用 `computed` 而不是方法
- [ ] 合理配置 `routeRules` 渲染模式
- [ ] 图片使用 `<NuxtImg>` 自动优化
- [ ] 数据获取使用 `pick` 减少传输量

---

## 19. 常见问题排查

### 19.1 开发阶段问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 页面空白 | 服务端渲染错误 | 检查浏览器控制台和终端日志 |
| Hydration 不匹配 | 服务端和客户端 HTML 不一致 | 确保 `useState` 初始化一致，避免客户端直接修改 DOM |
| 模块安装失败 | Node.js 版本过低 | `node --version` 检查版本 |
| 热更新不生效 | 文件命名不符合规范 | 检查文件名拼写和大小写 |

### 19.2 数据获取问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 数据重复请求 | setup 中同时使用了 `useFetch` 和 `$fetch` | 初始化数据用 `useFetch`，交互用 `$fetch` |
| 数据不更新 | `watch` 数组未包含响应式源 | 检查 `watch` 参数是否正确 |
| 跨域请求失败 | API 服务器未配置 CORS | 在 Nuxt 中使用 `server/api` 代理请求 |

### 19.3 部署问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `.env` 未加载 | 配置名为 `env` 而非 `.env` | 确保文件名为 `.env` |
| 构建内存不足 | 项目过大 | 增加 Node 内存: `NODE_OPTIONS="--max-old-space-size=4096"` |
| 404 页面 | `routeRules` 配置错误 | 检查 `prerender` 路由规则 |

### 19.4 TypeScript 问题

```ts
// 解决自动导入类型提示
// 确保 .nuxt/nuxt.d.ts 已生成（重启 dev server）

// 获取路由参数的类型安全
const route = useRoute('blog-slug')
// 参数自动推导为 { slug: string }
```

### 19.5 性能问题

```ts
// 检查是否存在不必要的响应式
// ❌ 不好：每次访问都创建新对象
function useExpensive() {
  return { data: computed(...) }
}

// ✅ 好：缓存计算属性
const data = computed(...)
function useExpensive() {
  return { data }
}
```

---

## 20. 参考资源

### 官方文档

- [Nuxt 4 文档](https://nuxt.com/docs)
- [Vue 3 文档](https://vuejs.org/)
- [Nitro 引擎](https://nitro.build/)
- [TypeScript 文档](https://www.typescriptlang.org/)

### 模块与工具

- [Nuxt 模块市场](https://nuxt.com/modules)
- [Pinia 文档](https://pinia.vuejs.org/)
- [VueUse 函数库](https://vueuse.org/)
- [UnoCSS 文档](https://unocss.dev/)
- [Vitest 文档](https://vitest.dev/)
- [Playwright 文档](https://playwright.dev/)

### 推荐学习路径

1. 先通读本文档 **第 3-8 章**
2. 跟着 **文档 02** 创建博客项目
3. 深入理解 **第 10 章** 服务端路由
4. 学习 **第 14-15 章** 认证与 SEO
5. 最后掌握 **第 16-17 章** 测试与部署

### 速查命令

```bash
# 开发
pnpm dev              # 启动开发服务器
pnpm dev --port 4000  # 指定端口

# 构建
pnpm build            # 生产构建
pnpm generate         # 静态生成
pnpm preview          # 预览构建

# 检查
pnpm typecheck        # TypeScript 类型检查
pnpm lint             # ESLint 检查
pnpm test             # 运行测试

# 创建
pnpm nuxi init        # 初始化项目
pnpm nuxi info        # 查看项目信息
```

### CLI 速查

| 命令 | 用途 |
|------|------|
| `nuxi init` | 创建新项目 |
| `nuxi dev` | 启动开发服务器 |
| `nuxi build` | 生产构建 |
| `nuxi generate` | 静态站点生成 |
| `nuxi preview` | 预览构建结果 |
| `nuxi typecheck` | 类型检查 |
| `nuxi info` | 显示项目信息 |
| `nuxi upgrade` | 升级 Nuxt 版本 |
