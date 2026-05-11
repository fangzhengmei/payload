# Payload CMS Live Preview 实时预览通信协议与数据同步机制分析报告

## 1. 概述

Payload CMS 的 Live Preview（实时预览）功能允许管理员在编辑内容时实时查看前端展示效果。该功能通过在后台管理界面和前端预览页面之间建立双向通信，实现数据变更的即时同步。

## 2. 通信协议

### 2.1 核心协议：window.postMessage

Live Preview 采用浏览器原生的 `window.postMessage` API 作为跨域通信协议。这是一种安全、高效的跨上下文通信机制，支持：

- **iframe 模式**：后台管理界面嵌入前端页面的 iframe
- **Popup 模式**：前端页面在独立弹出窗口中打开

### 2.2 通信架构

```
┌─────────────────────────────────────┐
│      后台管理界面 (Admin Panel)      │
│  ┌───────────────────────────────┐  │
│  │  LivePreviewWindow 组件        │  │
│  │  - 监听表单状态变化             │  │
│  │  - 发送 postMessage           │  │
│  └───────────────┬───────────────┘  │
│                  │                  │
│  ┌───────────────▼───────────────┐  │
│  │  iframe / Popup Window        │  │
│  │  (前端预览页面)                │  │
│  │  - 监听 message 事件          │  │
│  │  - 调用 handleMessage         │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

## 3. 消息类型与格式

### 3.1 消息类型定义

系统定义了两种主要消息类型：

#### 3.1.1 `payload-live-preview` 消息

用于传输实时预览数据，消息格式如下：

```typescript
type LivePreviewMessageEvent<T> = MessageEvent<{
  collectionSlug?: string        // 集合 slug（用于文档）
  data: T                        // 表单数据
  externallyUpdatedRelationship?: DocumentEvent  // 外部更新的关系数据
  globalSlug?: string            // 全局设置 slug
  locale?: string                // 当前语言
  type: 'payload-live-preview'   // 消息类型标识
}>
```

**消息示例**：
```javascript
{
  type: 'payload-live-preview',
  collectionSlug: 'pages',
  data: {
    id: 123,
    title: '我的页面',
    content: '页面内容...'
  },
  locale: 'en'
}
```

#### 3.1.2 `payload-document-event` 消息

用于 SSR（服务端渲染）场景的文档事件通知，触发服务器端刷新。

```javascript
{
  type: 'payload-document-event'
}
```

### 3.2 Ready 消息

前端页面加载完成后发送的就绪信号：

```javascript
{
  type: 'payload-live-preview',
  ready: true
}
```

## 4. 数据同步机制

### 4.1 完整同步流程

#### 阶段 1：初始化连接

1. **后台管理端** (`LivePreviewProvider`)：
   - 监听 `message` 事件等待 `ready` 信号
   - 加载 iframe 或打开 popup 窗口

2. **前端预览端** (`useLivePreview` hook)：
   - 调用 `subscribe()` 注册消息监听器
   - 调用 `ready()` 发送就绪信号

**前端订阅代码** (`packages/live-preview/src/subscribe.ts:12-37`)：
```typescript
export const subscribe = <T extends Record<string, any>>(args: {
  apiRoute?: string
  callback: (data: T) => void
  depth?: number
  initialData: T
  requestHandler?: CollectionPopulationRequestHandler
  serverURL: string
}): ((event: MessageEvent) => Promise<void> | void) => {
  // 重置缓存，确保新订阅不继承旧数据
  resetCache()

  const onMessage = async (event: MessageEvent) => {
    const mergedData = await handleMessage<T>({
      apiRoute,
      depth,
      event,
      initialData,
      requestHandler,
      serverURL,
    })
    callback(mergedData)
  }

  if (typeof window !== 'undefined') {
    window.addEventListener('message', onMessage)
  }

  return onMessage
}
```

#### 阶段 2：数据变更检测与发送

后台管理端通过 `LivePreviewWindow` 组件监听表单状态变化：

**数据发送代码** (`packages/ui/src/elements/LivePreview/Window/index.tsx:48-94`)：
```typescript
useEffect(() => {
  if (!isLivePreviewing || !appIsReady) {
    return
  }

  if (formState) {
    // 将表单状态转换为实际值
    const values = reduceFieldsToValues(formState, true)

    if (!values.id) {
      values.id = id
    }

    const message = {
      type: 'payload-live-preview',
      collectionSlug,
      data: values,
      externallyUpdatedRelationship: mostRecentUpdate,
      globalSlug,
      locale: locale.code,
    }

    // 发送到 popup 窗口
    if (previewWindowType === 'popup' && popupRef.current) {
      popupRef.current.postMessage(message, url)
    }

    // 发送到 iframe
    if (previewWindowType === 'iframe' && iframeRef.current) {
      iframeRef.current.contentWindow?.postMessage(message, url)
    }
  }
}, [formState, url, collectionSlug, globalSlug, id, /* ... */])
```

#### 阶段 3：前端接收与数据合并

前端通过 `handleMessage` 处理接收到的消息：

**消息处理代码** (`packages/live-preview/src/handleMessage.ts:23-64`)：
```typescript
export const handleMessage = async <T extends Record<string, any>>(args: {
  apiRoute?: string
  depth?: number
  event: LivePreviewMessageEvent<T>
  initialData: T
  requestHandler?: CollectionPopulationRequestHandler
  serverURL: string
}): Promise<T> => {
  const { apiRoute, depth, event, initialData, requestHandler, serverURL } = args

  // 验证消息是否来自合法源
  if (isLivePreviewEvent(event, serverURL)) {
    const { collectionSlug, data, globalSlug, locale } = event.data

    // 必须有明确的目标（集合或全局设置）
    if (!collectionSlug && !globalSlug) {
      return initialData
    }

    // 合并数据（可能涉及 API 调用解析关系字段）
    const mergedData = await mergeData<T>({
      apiRoute,
      collectionSlug,
      depth,
      globalSlug,
      incomingData: data,
      initialData: _payloadLivePreview?.previousData || initialData,
      locale,
      requestHandler,
      serverURL,
    })

    // 缓存合并后的数据，使变更能够累积
    _payloadLivePreview.previousData = mergedData

    return mergedData
  }

  // 非预览事件，返回缓存数据
  if (!_payloadLivePreview.previousData) {
    _payloadLivePreview.previousData = initialData
  }

  return _payloadLivePreview.previousData as T
}
```

### 4.2 数据合并机制

`mergeData` 函数负责处理关系字段的深度解析：

**数据合并代码** (`packages/live-preview/src/mergeData.ts:22-62`)：
```typescript
const defaultRequestHandler: CollectionPopulationRequestHandler = ({
  apiPath,
  data,
  endpoint,
  serverURL,
}) => {
  const url = `${serverURL}${apiPath}/${endpoint}`

  return fetch(url, {
    body: JSON.stringify(data),
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      'X-Payload-HTTP-Method-Override': 'GET',
    },
    method: 'POST',
  })
}

export const mergeData = async <T extends Record<string, any>>(args: {
  // ... 参数
}): Promise<T> => {
  const requestHandler = args.requestHandler || defaultRequestHandler

  // 调用 API 解析关系字段
  const result = await requestHandler({
    apiPath: apiRoute || '/api',
    data: {
      data: incomingData,
      depth,
      flattenLocales: false,
      locale,
    },
    endpoint: encodeURI(
      `${globalSlug ? 'globals/' : ''}${collectionSlug ?? globalSlug}${collectionSlug ? `/${initialData.id}` : ''}`,
    ),
    serverURL,
  }).then((res) => res.json())

  return result
}
```

### 4.3 SSR 模式下的同步

对于服务端渲染的页面，通过发送 `payload-document-event` 消息触发服务器端刷新：

**SSR 事件发送代码** (`packages/ui/src/elements/LivePreview/Window/index.tsx:101-119`)：
```typescript
useEffect(() => {
  if (!isLivePreviewing || !appIsReady) {
    return
  }

  const message = {
    type: 'payload-document-event',
  }

  // 发送到 popup
  if (previewWindowType === 'popup' && popupRef.current) {
    popupRef.current.postMessage(message, url)
  }

  // 发送到 iframe
  if (previewWindowType === 'iframe' && iframeRef.current) {
    iframeRef.current.contentWindow?.postMessage(message, url)
  }
}, [mostRecentUpdate, /* ... */])
```

## 5. React Hook 封装

前端应用通过 `useLivePreview` hook 简化使用：

**Hook 实现** (`packages/live-preview-react/src/useLivePreview.ts:41-100`)：
```typescript
export const useLivePreview = <T extends Record<string, any>>(props: {
  apiRoute?: string
  depth?: number
  initialData: T
  requestHandler?: CollectionPopulationRequestHandler
  serverURL: string
}): {
  data: T
  isLoading: boolean
} => {
  const { apiRoute, depth, initialData, requestHandler, serverURL } = props
  const [data, setData] = useState<T>(initialData)
  const [isLoading, setIsLoading] = useState<boolean>(true)
  const hasSentReadyMessage = useRef<boolean>(false)

  const onChange = useCallback((mergedData: T) => {
    setData(mergedData)
    setIsLoading(false)
  }, [])

  useEffect(() => {
    // 订阅消息
    const subscription = subscribe({
      apiRoute,
      callback: onChange,
      depth,
      initialData,
      requestHandler,
      serverURL,
    })

    // 发送就绪信号
    if (!hasSentReadyMessage.current) {
      hasSentReadyMessage.current = true
      ready({ serverURL })
    }

    // 清理
    return () => {
      unsubscribe(subscription)
    }
  }, [serverURL, onChange, depth, initialData, apiRoute, requestHandler])

  return { data, isLoading }
}
```

## 6. 安全机制

### 6.1 消息源验证

所有消息都必须通过 `isLivePreviewEvent` 验证：

**验证代码** (`packages/live-preview/src/isLivePreviewEvent.ts:1-5`)：
```typescript
export const isLivePreviewEvent = (event: MessageEvent, serverURL: string): boolean =>
  event.origin === serverURL &&
  event.data &&
  typeof event.data === 'object' &&
  event.data.type === 'payload-live-preview'
```

### 6.2 CORS 配置

测试配置中明确设置了 CORS 和 CSRF 白名单：

**配置示例** (`test/live-preview/config.ts:57-58`)：
```typescript
cors: [`http://localhost:${process.env.PORT || 3000}`, 'http://localhost:3001'],
csrf: [`http://localhost:${process.env.PORT || 3000}`, 'http://localhost:3001'],
```

## 7. 数据流时序图

```
后台管理端 (Admin)                    前端预览端 (Frontend)
        |                                   |
        |  1. 加载 iframe/popup            |
        |--------------------------------->|
        |                                   |
        |                                   |  2. subscribe()
        |                                   |  3. ready()
        |<---------------------------------|
        |  收到 ready 信号                  |
        |                                   |
        |  4. 表单值变化                    |
        |  5. reduceFieldsToValues()       |
        |  6. postMessage(data)            |
        |--------------------------------->|
        |                                   |  7. handleMessage()
        |                                   |  8. 验证 origin
        |                                   |  9. mergeData() [可能调用 API]
        |                                   | 10. 更新 React state
        |                                   | 11. 重新渲染 UI
        |                                   |
        |  (循环：步骤 4-11 持续重复)        |
```

## 8. 关键文件清单

| 文件路径 | 功能说明 |
|---------|---------|
| `packages/live-preview/src/types.ts` | 消息类型定义 |
| `packages/live-preview/src/subscribe.ts` | 消息订阅 |
| `packages/live-preview/src/unsubscribe.ts` | 取消订阅 |
| `packages/live-preview/src/ready.ts` | 就绪信号发送 |
| `packages/live-preview/src/handleMessage.ts` | 消息处理核心 |
| `packages/live-preview/src/mergeData.ts` | 数据合并与关系解析 |
| `packages/live-preview/src/isLivePreviewEvent.ts` | 消息验证 |
| `packages/live-preview-react/src/useLivePreview.ts` | React Hook 封装 |
| `packages/ui/src/elements/LivePreview/Window/index.tsx` | 后台数据发送组件 |
| `packages/ui/src/providers/LivePreview/index.tsx` | Live Preview 上下文提供者 |

## 9. 总结

### 9.1 核心技术栈

- **通信协议**：`window.postMessage`
- **消息格式**：JSON 对象，带类型标识
- **前端框架**：React（提供 Hook 封装）
- **关系解析**：REST API POST 请求（带 `X-Payload-HTTP-Method-Override: GET`）

### 9.2 设计亮点

1. **双向通信**：前后端通过 postMessage 双向交互
2. **数据累积**：通过 `previousData` 缓存实现增量变更累积
3. **模式灵活**：支持 iframe 和 popup 两种预览模式
4. **SSR 兼容**：通过 `payload-document-event` 触发服务端刷新
5. **安全验证**：严格的 origin 验证确保消息安全

### 9.3 适用场景

- 内容编辑实时预览
- 多语言内容预览
- 响应式布局预览（通过 breakpoints 配置）
- 草稿状态内容预览
