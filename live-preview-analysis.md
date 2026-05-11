# Payload CMS Live Preview 实时预览通信协议与数据同步机制分析报告

## 1. 概述

Payload CMS 的 Live Preview（实时预览）功能允许管理员在编辑内容时实时查看前端展示效果。该功能通过后台管理界面与前端预览页面之间的 `window.postMessage` 通信实现数据变更同步。

## 2. 通信架构

### 2.1 核心协议

Live Preview 采用浏览器原生的 `window.postMessage` API 作为跨上下文通信协议。支持两种预览模式：

- **iframe 模式**：后台管理界面嵌入前端页面的 iframe
- **Popup 模式**：前端页面在独立弹出窗口中打开

### 2.2 双向消息通道

系统定义了 **两条独立的消息通道**，共享相同的消息传输层但用途完全不同：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         window.postMessage 传输层                              │
│  注：postMessage 与 CORS 是两套独立机制，边界不同。                              │
│  - CORS：控制 XMLHttpRequest/fetch 的跨域请求权限                              │
│  - postMessage：控制窗口间消息投递权限，通过 targetOrigin 指定接收方 origin     │
├─────────────────────────────────────┬────────────────────────────────────────┤
│      通道 A: Ready 握手通道          │  通道 B: 数据更新通道       │
│  (方向：前端 → 后台)                 │  (方向：后台 → 前端)         │
├─────────────────────────────────────┼───────────────────────────┤
│  消息类型：payload-live-preview     │  B.1: payload-live-preview │
│  载荷：{ type, ready: true }        │       (表单实时数据)        │
│                                     │  B.2: payload-document-event│
│                                     │       (SSR 文档保存事件)     │
└─────────────────────────────────────┴───────────────────────────┘
```

## 3. 通道 A：Ready 握手通道

### 3.1 作用

前端预览页面向后台管理界面发送"已就绪"信号，告知后台可以开始发送数据更新。

### 3.2 消息格式

**前端发送** (`packages/live-preview/src/ready.ts:1-18`)：
```typescript
// 发送目标：window.opener (popup) 或 window.parent (iframe)
windowToPostTo.postMessage(
  {
    type: 'payload-live-preview',
    ready: true,
  },
  serverURL,  // targetOrigin
)
```

**后台接收与校验** (`packages/ui/src/providers/LivePreview/index.tsx:212-233`)：
```typescript
const handleMessage = (event: MessageEvent) => {
  if (
    url?.startsWith(event.origin) &&        // origin 宽松校验
    event.data &&
    typeof event.data === 'object' &&
    event.data.type === 'payload-live-preview'
  ) {
    if (event.data.ready) {
      setAppIsReady(true)
    }
  }
}
```

### 3.3 Origin 校验口径（Ready 通道）

**校验位置**：后台管理端（接收方）

**校验逻辑**：`url?.startsWith(event.origin)`

**校验特点**：
- 宽松校验：只要 `event.origin` 是 `url` 的前缀即可通过
- 原因：`url` 可能包含路径（如 `http://localhost:3000/preview`），而 `event.origin` 仅包含协议 + 主机 + 端口（如 `http://localhost:3000`）

### 3.4 握手触发时机

**前端端**：
- `useLivePreview` hook：`useEffect` 首次执行时发送（`packages/live-preview-react/src/useLivePreview.ts:83-89`）
- `RefreshRouteOnSave` 组件：`useEffect` 首次执行时发送（`packages/live-preview-react/src/RefreshRouteOnSave.tsx:36-44`）

**后台端**：
- 收到 `ready` 消息后设置 `appIsReady = true`
- `appIsReady` 是后续数据发送的前置条件

## 4. 通道 B：数据更新通道

后台管理界面向前端预览页面发送数据更新，包含 **两条子通道**。

### 4.1 子通道 B.1：表单实时数据（Client-side）

**适用场景**：纯客户端前端应用（如 Next.js Pages Router、React Router 等）

#### 消息格式

**消息类型**：`payload-live-preview`

**消息结构** (`packages/live-preview/src/types.ts:19-26`)：
```typescript
type LivePreviewMessageEvent<T> = MessageEvent<{
  collectionSlug?: string        // 集合 slug（文档编辑时）
  data: T                        // 表单完整数据（reduceFieldsToValues 结果）
  externallyUpdatedRelationship?: DocumentEvent  // 嵌套文档更新事件
  globalSlug?: string            // 全局设置 slug
  locale?: string                // 当前语言代码
  type: 'payload-live-preview'   // 固定类型标识
}>
```

#### 发送触发条件

**发送位置**：`packages/ui/src/elements/LivePreview/Window/index.tsx:48-94`

**前置条件**：
```typescript
if (!isLivePreviewing || !appIsReady) {
  return  // 必须同时满足：预览模式已开启 + 前端已就绪
}
```

**依赖变化触发**（`useEffect` deps）：
- `formState`：表单状态变化（用户输入时触发）
- `mostRecentUpdate`：嵌套文档更新
- `locale`：语言切换
- `url`、`collectionSlug`、`globalSlug` 等配置变化

#### 数据提取流程

```typescript
const values = reduceFieldsToValues(formState, true)  // 表单状态 → 实际值

if (!values.id) {
  values.id = id  // 确保包含文档 ID
}
```

### 4.2 子通道 B.2：文档保存事件（Server-side / SSR）

**适用场景**：服务端渲染框架（如 Next.js App Router + React Server Components）

#### 消息格式

**消息类型**：`payload-document-event`

**消息结构**：
```typescript
{
  type: 'payload-document-event'  // 无数据载荷
}
```

#### 发送触发条件

**发送位置**：`packages/ui/src/elements/LivePreview/Window/index.tsx:101-119`

**前置条件**：与 B.1 相同，需 `isLivePreviewing && appIsReady`

**依赖变化触发**（`useEffect` deps）：
- `mostRecentUpdate`：文档事件（如保存、自动保存、发布等）

## 5. 数据同步机制（Client-side）

### 5.1 完整流程

```
后台管理端 (Admin)                          前端预览端 (Frontend)
        |                                         |
        |  1. 加载 iframe/popup                  |
        |--------------------------------------->|
        |                                         |
        |         [通道 A: Ready 握手]            |
        |                                         |
        |<---------------------------------------|  2. ready({ serverURL })
        |  收到 { type: 'payload-live-preview',   |
        |         ready: true }                   |
        |  → setAppIsReady(true)                  |
        |                                         |
        |         [通道 B.1: 表单数据更新]          |
        |                                         |
        |  3. formState 变化                      |
        |  4. reduceFieldsToValues()              |
        |  5. postMessage({ type: 'payload-      |
        |         live-preview', data: values })  |
        |--------------------------------------->|
        |                                         |  6. isLivePreviewEvent() 校验
        |                                         |  7. handleMessage()
        |                                         |  8. mergeData() [API 调用]
        |                                         |  9. callback(mergedData)
        |                                         |  10. 更新 React state
        |                                         |  11. 重新渲染 UI
        |                                         |
        |  (循环：步骤 3-11 持续重复)               |
```

### 5.2 前端接收与处理

**订阅入口** (`packages/live-preview/src/subscribe.ts:5-37`)：
```typescript
export const subscribe = <T>(args) => {
  resetCache()  // 每次订阅前重置缓存

  const onMessage = async (event) => {
    const mergedData = await handleMessage<T>({
      apiRoute, depth, event, initialData, requestHandler, serverURL
    })
    callback(mergedData)
  }

  window.addEventListener('message', onMessage)
  return onMessage
}
```

**消息处理核心** (`packages/live-preview/src/handleMessage.ts:23-64`)：
```typescript
export const handleMessage = async <T>(args): Promise<T> => {
  // 1. Origin 严格校验
  if (isLivePreviewEvent(event, serverURL)) {
    const { collectionSlug, data, globalSlug, locale } = event.data

    // 2. 必须有明确目标
    if (!collectionSlug && !globalSlug) {
      return initialData
    }

    // 3. 数据合并（可能触发 API 调用）
    const mergedData = await mergeData<T>({
      incomingData: data,
      initialData: _payloadLivePreview?.previousData || initialData,
      // ... 其他参数
    })

    // 4. 缓存合并结果
    _payloadLivePreview.previousData = mergedData
    return mergedData
  }

  // 非预览事件：返回缓存或初始数据
  if (!_payloadLivePreview.previousData) {
    _payloadLivePreview.previousData = initialData
  }
  return _payloadLivePreview.previousData as T
}
```

### 5.3 Origin 校验口径（数据通道）

**校验函数** (`packages/live-preview/src/isLivePreviewEvent.ts:1-5`)：
```typescript
export const isLivePreviewEvent = (event: MessageEvent, serverURL: string): boolean =>
  event.origin === serverURL &&        // 严格相等校验
  event.data &&
  typeof event.data === 'object' &&
  event.data.type === 'payload-live-preview'
```

**校验函数** (`packages/live-preview/src/isDocumentEvent.ts:1-5`)：
```typescript
export const isDocumentEvent = (event: MessageEvent, serverURL: string): boolean =>
  event.origin === serverURL &&        // 严格相等校验
  event.data &&
  typeof event.data === 'object' &&
  event.data.type === 'payload-document-event'
```

**校验特点**：
- 严格校验：`event.origin === serverURL` 必须完全相等
- 适用于数据通道的安全要求

### 5.4 previousData 缓存机制

**缓存位置**：模块级闭包变量 (`packages/live-preview/src/handleMessage.ts:6-15`)

```typescript
const _payloadLivePreview: {
  previousData: any
} = {
  /**
   * Each time the data is merged, cache the result as a `previousData` variable
   * This will ensure changes compound overtop of each other
   */
  previousData: undefined,
}
```

**职责**：
1. **跨消息累积状态**：每次 `mergeData` 后缓存结果，确保后续消息的 `initialData` 基于上次合并结果
2. **非预览事件回退**：收到非 `payload-live-preview` 类型消息时，返回缓存数据避免闪烁
3. **导航隔离**：`subscribe()` 调用前会执行 `resetCache()`，确保路由切换后不继承旧页面数据

### 5.5 mergeData 职责

**核心逻辑** (`packages/live-preview/src/mergeData.ts:22-62`)：

```typescript
export const mergeData = async <T>(args): Promise<T> => {
  const requestHandler = args.requestHandler || defaultRequestHandler

  // 通过 API 解析关系字段
  const result = await requestHandler({
    apiPath: apiRoute || '/api',
    data: {
      data: incomingData,     // 表单实时数据
      depth,                  // 关系解析深度
      flattenLocales: false,
      locale,
    },
    endpoint: encodeURI(
      // 集合文档：{collectionSlug}/{id}
      // 全局设置：globals/{globalSlug}
      `${globalSlug ? 'globals/' : ''}${collectionSlug ?? globalSlug}${collectionSlug ? `/${initialData.id}` : ''}`,
    ),
    serverURL,
  }).then((res) => res.json())

  return result
}
```

**职责**：
1. **关系字段深度解析**：表单实时数据只包含关系字段的 `id`，需通过 API 调用获取完整关系数据
2. **请求封装**：使用 POST 请求 + `X-Payload-HTTP-Method-Override: GET` 头，支持携带大体积表单数据
3. **自定义扩展**：支持传入 `requestHandler` 拦截请求（如路由到中间件）

**默认请求处理** (`packages/live-preview/src/mergeData.ts:3-20`)：
```typescript
const defaultRequestHandler = ({ apiPath, data, endpoint, serverURL }) => {
  const url = `${serverURL}${apiPath}/${endpoint}`

  return fetch(url, {
    body: JSON.stringify(data),
    credentials: 'include',  // 携带 cookie（认证）
    headers: {
      'Content-Type': 'application/json',
      'X-Payload-HTTP-Method-Override': 'GET',
    },
    method: 'POST',
  })
}
```

## 6. 数据同步机制（Server-side / SSR）

### 6.1 设计理念

服务端渲染页面无法直接接收实时表单数据并更新 UI，因此采用 **"事件通知 + 服务端重渲染"** 模式：

- 后台发送 `payload-document-event` 事件（无数据载荷）
- 前端收到事件后调用 `router.refresh()` 触发服务端重新获取数据
- 适用于：文档保存、自动保存、发布等操作

### 6.2 完整触发链路

```
┌─────────────────────────────────────────────────────────────────────┐
│  后台管理端 (Admin Panel)                                            │
│                                                                     │
│  1. 用户操作：保存 / 自动保存 / 发布文档                               │
│     ↓                                                               │
│  2. useDocumentEvents 捕获事件                                        │
│     (packages/ui/src/providers/DocumentEvents/index.tsx)            │
│     ↓                                                               │
│  3. mostRecentUpdate 状态更新                                        │
│     ↓                                                               │
│  4. useEffect 依赖变化触发                                            │
│     (packages/ui/src/elements/LivePreview/Window/index.tsx:101-119) │
│     ↓                                                               │
│  5. postMessage({ type: 'payload-document-event' })                 │
│     ↓                                                               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ window.postMessage
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  前端预览端 (SSR Frontend)                                           │
│                                                                     │
│  6. RefreshRouteOnSave 监听 message 事件                             │
│     (packages/live-preview-react/src/RefreshRouteOnSave.tsx)        │
│     ↓                                                               │
│  7. isDocumentEvent(event, serverURL) 校验                          │
│     (origin === serverURL 严格相等)                                  │
│     ↓                                                               │
│  8. 调用 refresh() 回调                                              │
│     ↓                                                               │
│  9. Next.js: router.refresh()                                       │
│     (用户在 page.tsx 中传入)                                         │
│     ↓                                                               │
│  10. 服务端重新执行数据查询                                           │
│      (通过 Local API 获取最新文档)                                    │
│     ↓                                                               │
│  11. 页面重新渲染                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 RefreshRouteOnSave 实现

**组件代码** (`packages/live-preview-react/src/RefreshRouteOnSave.tsx:1-55`)：

```typescript
export const RefreshRouteOnSave: React.FC<{
  refresh: () => void    // 用户传入的刷新函数
  serverURL: string
}> = (props) => {
  const { refresh, serverURL } = props
  const hasSentReadyMessage = useRef(false)

  const onMessage = useCallback(
    (event: MessageEvent) => {
      if (isDocumentEvent(event, serverURL)) {  // 严格 origin 校验
        if (typeof refresh === 'function') {
          refresh()
        }
      }
    },
    [refresh, serverURL],
  )

  useEffect(() => {
    window.addEventListener('message', onMessage)

    if (!hasSentReadyMessage.current) {
      hasSentReadyMessage.current = true
      ready({ serverURL })  // 发送 ready 信号
      refresh()             // 首次加载时刷新获取最新数据
    }

    return () => window.removeEventListener('message', onMessage)
  }, [serverURL, onMessage, refresh])

  return null
}
```

### 6.4 使用示例（Next.js App Router）

**用户封装** (`docs/live-preview/server.mdx:66-84`)：
```tsx
// RefreshRouteOnSave.tsx
'use client'
import { RefreshRouteOnSave as PayloadLivePreview } from '@payloadcms/live-preview-react'
import { useRouter } from 'next/navigation.js'

export const RefreshRouteOnSave: React.FC = () => {
  const router = useRouter()

  return (
    <PayloadLivePreview
      refresh={() => router.refresh()}
      serverURL={process.env.NEXT_PUBLIC_PAYLOAD_URL}
    />
  )
}
```

**页面使用**：
```tsx
// page.tsx
import { RefreshRouteOnSave } from './RefreshRouteOnSave'

export default async function Page({ params }) {
  const page = await payload.findByID({
    collection: 'pages',
    id: params.id,
    draft: true,  // 查询草稿数据
  })

  return (
    <>
      <RefreshRouteOnSave />
      <h1>{page.title}</h1>
    </>
  )
}
```

## 7. Origin 校验口径汇总

| 校验场景 | 校验位置 | 校验函数/逻辑 | 校验口径 |
|---------|---------|--------------|---------|
| Ready 握手接收 | 后台管理端 | `url?.startsWith(event.origin)` | **宽松**：origin 是 url 前缀 |
| 表单数据接收 | 前端预览端 | `event.origin === serverURL` | **严格**：完全相等 |
| 文档事件接收 | 前端预览端 | `event.origin === serverURL` | **严格**：完全相等 |

### 7.1 不一致性说明

**Ready 握手通道使用宽松校验的原因**：
- 后台管理端的 `url` 变量是预览页面完整 URL（可能包含路径，如 `http://localhost:3000/preview/posts/123`）
- `event.origin` 是浏览器标准化的来源（仅协议 + 主机 + 端口，如 `http://localhost:3000`）
- 使用 `startsWith` 可确保无论 `url` 是否包含路径，校验都能通过

**数据通道使用严格校验的原因**：
- 前端预览端传入的 `serverURL` 配置通常是纯 origin（如 `http://localhost:3000`）
- 数据通道涉及敏感内容（表单数据、文档更新），需要更严格的安全校验

## 8. 关键文件清单

| 文件路径 | 功能说明 |
|---------|---------|
| `packages/live-preview/src/ready.ts` | Ready 握手信号发送（前端 → 后台） |
| `packages/live-preview/src/subscribe.ts` | 消息订阅与缓存重置 |
| `packages/live-preview/src/handleMessage.ts` | 表单数据消息处理与 previousData 缓存 |
| `packages/live-preview/src/mergeData.ts` | 关系字段解析 API 调用封装 |
| `packages/live-preview/src/isLivePreviewEvent.ts` | 表单数据消息校验（严格 origin） |
| `packages/live-preview/src/isDocumentEvent.ts` | 文档事件消息校验（严格 origin） |
| `packages/live-preview/src/types.ts` | 消息类型定义 |
| `packages/live-preview-react/src/useLivePreview.ts` | Client-side React Hook（表单实时数据） |
| `packages/live-preview-react/src/RefreshRouteOnSave.tsx` | Server-side React 组件（文档事件 + refresh） |
| `packages/live-preview-vue/src/index.ts` | Vue 3 Composable（仅 Client-side） |
| `packages/ui/src/elements/LivePreview/Window/index.tsx` | 后台数据发送（两条子通道） |
| `packages/ui/src/providers/LivePreview/index.tsx` | Ready 信号接收与 appIsReady 状态管理 |
| `packages/ui/src/providers/DocumentEvents/index.tsx` | 文档事件上下文（mostRecentUpdate） |

## 9. 总结

### 9.1 核心技术栈

- **传输协议**：`window.postMessage`
- **握手通道**：`payload-live-preview` + `ready: true`（前端 → 后台）
- **数据通道 B.1**：`payload-live-preview` + 表单数据（后台 → 前端，Client-side）
- **数据通道 B.2**：`payload-document-event`（后台 → 前端，Server-side）
- **关系解析**：REST API POST + `X-Payload-HTTP-Method-Override: GET`

### 9.2 两种同步模式对比

| 特性 | Client-side（表单实时数据） | Server-side（文档保存事件） |
|-----|---------------------------|---------------------------|
| **触发时机** | 表单输入（实时） | 文档保存 / 自动保存 / 发布 |
| **消息载荷** | 完整表单数据 | 无（仅事件信号） |
| **前端处理** | 接收数据 → 合并 → 更新 state | 接收事件 → refresh() → 服务端重查 |
| **延迟** | 低（即时） | 高（需等待保存完成 + 服务端重渲染） |
| **React 封装** | `useLivePreview` hook | `RefreshRouteOnSave` 组件 |
| **Vue 封装** | `useLivePreview` composable | 无（需自行实现） |

### 9.3 数据流时序图（完整版）

```
┌─────────────────┐                                 ┌─────────────────┐
│   后台管理端     │                                 │   前端预览端     │
│   (Admin)       │                                 │  (Frontend)     │
└────────┬────────┘                                 └────────┬────────┘
         │                                                     │
         │  [初始化阶段]                                        │
         │                                                     │
         │  1. 加载 iframe / 打开 popup                        │
         │────────────────────────────────────────────────────>│
         │                                                     │
         │                                                     │  2. subscribe()
         │                                                     │  3. resetCache()
         │                                                     │  4. ready({ serverURL })
         │  { type: 'payload-live-preview', ready: true }      │
         │<────────────────────────────────────────────────────│
         │                                                     │
         │  5. url?.startsWith(event.origin) 校验               │
         │  6. setAppIsReady(true)                             │
         │                                                     │
         │  [Client-side 同步]                                 │
         │                                                     │
         │  7. 用户输入 → formState 变化                        │
         │  8. reduceFieldsToValues(formState)                 │
         │  9. postMessage({                                   │
         │       type: 'payload-live-preview',                 │
         │       data: values,                                 │
         │       collectionSlug, ...                           │
         │     })                                              │
         │────────────────────────────────────────────────────>│
         │                                                     │  10. event.origin === serverURL 校验
         │                                                     │  11. mergeData() → API 调用
         │                                                     │  12. _payloadLivePreview.previousData = result
         │                                                     │  13. callback(mergedData)
         │                                                     │  14. setState → 重新渲染
         │                                                     │
         │  [Server-side 同步]                                 │
         │                                                     │
         │  15. 文档保存 → mostRecentUpdate 变化                │
         │  16. postMessage({                                  │
         │        type: 'payload-document-event'               │
         │      })                                             │
         │────────────────────────────────────────────────────>│
         │                                                     │  17. event.origin === serverURL 校验
         │                                                     │  18. refresh()
         │                                                     │  19. Next.js router.refresh()
         │                                                     │  20. 服务端重新查询数据
         │                                                     │  21. 页面重新渲染
         │                                                     │
         │  (两种同步模式独立运行，互不干扰)                      │
```
