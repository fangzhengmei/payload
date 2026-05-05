# Payload CMS REST 与 GraphQL 统一分析报告

## 1. 架构概览

### 1.1 三层 API 架构

Payload CMS 采用 **三层 API 架构**，所有接口共享同一核心业务逻辑层：

```
┌─────────────────────────────────────────────────────────────────┐
│                        入口层 (Entry Layer)                        │
├─────────────────────┬─────────────────────┬─────────────────────┤
│    REST API         │    GraphQL API      │    Local API        │
│  (HTTP 端点)         │  (GraphQL 端点)     │  (代码级调用)        │
└──────────┬──────────┴──────────┬──────────┴──────────┬──────────┘
           │                     │                     │
           ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   核心操作层 (Operations Layer)                    │
│  ┌─────────────┬─────────────┬─────────────┬─────────────────┐  │
│  │ findOperation│createOperation│updateByIDOp │ deleteByIDOp   │  │
│  └─────────────┴─────────────┴─────────────┴─────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  共享: 访问控制 (executeAccess) + 钩子 (Hooks)               │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 关键文件位置

| 层级 | 文件路径 | 说明 |
|------|----------|------|
| REST 路由 | `packages/next/src/routes/rest/index.ts` | REST API 入口处理器 |
| GraphQL 解析器 | `packages/graphql/src/resolvers/collections/*.ts` | GraphQL 各操作解析器 |
| Local API | `packages/payload/src/collections/operations/local/*.ts` | 本地 API 封装 |
| 核心操作 | `packages/payload/src/collections/operations/*.ts` | 共享的核心操作实现 |
| 访问控制 | `packages/payload/src/auth/executeAccess.ts` | 访问控制执行函数 |
| 端点处理 | `packages/payload/src/utilities/handleEndpoints.ts` | REST 端点匹配与分发 |

---

## 2. 统一配置机制

### 2.1 单一配置源

Payload CMS 的核心设计理念是 **"一次配置，多端生效"**。所有 API 层共享同一套 Collection 配置：

```typescript
// CollectionConfig 示例
const Posts: CollectionConfig = {
  slug: 'posts',
  
  // 访问控制 - 被所有 API 层共用
  access: {
    read: ({ req }) => { /* ... */ },
    create: ({ req }) => { /* ... */ },
    update: ({ req }) => { /* ... */ },
    delete: ({ req }) => { /* ... */ },
  },
  
  // 钩子 - 被所有 API 层共用
  hooks: {
    beforeOperation: [/* ... */],
    beforeValidate: [/* ... */],
    beforeChange: [/* ... */],
    afterChange: [/* ... */],
    afterRead: [/* ... */],
    afterOperation: [/* ... */],
  },
  
  // 字段配置
  fields: [
    {
      name: 'title',
      type: 'text',
      // 字段级钩子和访问控制同样被共用
      hooks: {
        beforeChange: [/* ... */],
      },
      access: {
        read: ({ req }) => { /* ... */ },
      },
    },
  ],
}
```

### 2.2 配置如何被多层 API 使用

同一配置 `access.read` 会被以下所有场景调用：

| API 类型 | 调用方式 | 触发场景 |
|----------|----------|----------|
| REST | `GET /api/posts` | HTTP GET 请求 |
| GraphQL | `query { Posts { ... } }` | GraphQL 查询 |
| Local | `payload.find({ collection: 'posts' })` | 服务端代码调用 |

---

## 3. 调用链分析

### 3.1 REST API 调用链

```
HTTP Request
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ packages/next/src/routes/rest/index.ts                    │
│  - REST_GET / REST_POST / REST_PATCH / REST_DELETE       │
└───────────────────────────┬───────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│ packages/payload/src/utilities/handleEndpoints.ts         │
│  1. 创建 PayloadRequest                                    │
│  2. 根据路径匹配端点配置                                    │
│  3. 设置 req.payloadAPI = 'REST'                          │
│  4. 调用端点 handler                                        │
└───────────────────────────┬───────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│ 端点 Handler (内部端点或自定义端点)                         │
│  内部端点调用 Local API 或直接调用核心操作                  │
└───────────────────────────┬───────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│ packages/payload/src/collections/operations/*.ts          │
│  核心操作函数:                                              │
│  - findOperation                                           │
│  - createOperation                                         │
│  - updateByIDOperation                                     │
│  - deleteByIDOperation                                     │
└───────────────────────────────────────────────────────────┘
```

### 3.2 GraphQL API 调用链

```
GraphQL Query/Mutation
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ packages/graphql/src/resolvers/collections/*.ts           │
│  解析器函数:                                                │
│  - findResolver (packages/graphql/src/resolvers/collections/find.ts)
│  - createResolver (packages/graphql/src/resolvers/collections/create.ts)
│  - updateResolver (packages/graphql/src/resolvers/collections/update.ts)
│
│  关键步骤:                                                  │
│  1. 从 GraphQL args 提取参数                               │
│  2. 调用 isolateObjectProperty 隔离请求上下文               │
│  3. 直接调用核心操作函数                                    │
└───────────────────────────┬───────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│ packages/payload/src/collections/operations/*.ts          │
│  核心操作函数 (与 REST 完全相同)                            │
│  - req.payloadAPI = 'GraphQL' (在 createPayloadRequest 中设置)
└───────────────────────────────────────────────────────────┘
```

**GraphQL 解析器示例** (`find.ts:30-71`):

```typescript
export function findResolver(collection: Collection): Resolver {
  return async function resolver(_, args, context, info) {
    const req = (context.req = isolateObjectProperty(context.req, [
      'locale',
      'fallbackLocale',
      'transactionID',
    ]))
    const select = (context.select = args.select ? buildSelectForCollectionMany(info) : undefined)

    // ... 参数处理 ...

    const options = {
      collection,
      depth: 0,
      draft: args.draft,
      limit: args.limit,
      page: args.page,
      pagination: args.pagination,
      req,
      select,
      sort: sort && typeof sort === 'string' ? sort.split(',') : undefined,
      trash: args.trash,
      where: args.where,
    }

    // 直接调用核心操作
    const result = await findOperation(options)
    return result
  }
}
```

### 3.3 Local API 调用链

```
payload.find({ collection: 'posts' })
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ packages/payload/src/collections/operations/local/find.ts  │
│  1. 验证 collection 是否存在                                │
│  2. 创建 Local Request (createLocalReq)                    │
│  3. 设置 req.payloadAPI = 'local'                          │
│  4. 调用核心操作函数                                        │
└───────────────────────────┬───────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│ packages/payload/src/collections/operations/*.ts          │
│  核心操作函数 (与 REST/GraphQL 完全相同)                    │
└───────────────────────────────────────────────────────────┘
```

---

## 4. 访问控制共享机制

### 4.1 核心函数 `executeAccess`

所有 API 层通过同一个函数 `executeAccess` 执行访问控制检查：

**位置**: `packages/payload/src/auth/executeAccess.ts:14-41`

```typescript
export const executeAccess = async (
  { id, data, disableErrors, isReadingStaticFile = false, req }: OperationArgs,
  access: Access,
): Promise<AccessResult> => {
  if (access) {
    const resolvedConstraint = await access({
      id,
      data,
      isReadingStaticFile,
      req,
    })

    if (!resolvedConstraint) {
      if (!disableErrors) {
        throw new Forbidden(req.t)
      }
    }

    return resolvedConstraint
  }

  // 默认规则: 已登录用户允许所有操作
  if (req.user) {
    return true
  }

  if (!disableErrors) {
    throw new Forbidden(req.t)
  }
  return false
}
```

### 4.2 在核心操作中的调用方式

**`findOperation` 中的访问控制** (`find.ts:117-137`):

```typescript
// /////////////////////////////////////
// Access
// /////////////////////////////////////

let accessResult: AccessResult

if (!overrideAccess) {
  accessResult = await executeAccess(
    { disableErrors, req }, 
    collectionConfig.access.read  // 使用配置中的 access.read
  )

  // 如果访问被拒绝且禁用错误，返回空结果
  if (accessResult === false) {
    return {
      docs: [],
      hasNextPage: false,
      hasPrevPage: false,
      limit: limit!,
      nextPage: null,
      page: 1,
      pagingCounter: 1,
      prevPage: null,
      totalDocs: 0,
      totalPages: 1,
    }
  }
}

// 访问控制返回的约束会与用户查询合并
let fullWhere = combineQueries(where!, accessResult!)
```

**`createOperation` 中的访问控制** (`create.ts:154-156`):

```typescript
if (!overrideAccess) {
  await executeAccess({ data, req }, collectionConfig.access.create)
}
```

### 4.3 访问控制函数参数

无论通过哪个 API 层调用，访问控制函数接收的参数都是一致的：

```typescript
// 访问控制函数签名
type AccessFunction = ({
  req: PayloadRequest,      // 请求对象，包含 user, locale 等
  id?: string | number,      // 文档 ID (用于 update/delete)
  data?: any,                // 数据 (用于 create/update)
}: AccessArgs) => 
  | boolean                    // true/false: 允许/拒绝
  | Promise<boolean>
  | WhereQuery                 // 返回查询约束 (仅 read 操作)
  | Promise<WhereQuery>
```

### 4.4 三层 API 访问控制对比

| 维度 | REST API | GraphQL API | Local API |
|------|----------|-------------|-----------|
| 执行函数 | `executeAccess` | `executeAccess` | `executeAccess` |
| 配置来源 | `collectionConfig.access.*` | `collectionConfig.access.*` | `collectionConfig.access.*` |
| `req.user` 来源 | HTTP 认证头/JWT | HTTP 认证头/JWT | 手动传入或上下文继承 |
| `req.payloadAPI` | `'REST'` | `'GraphQL'` | `'local'` |
| 默认 `overrideAccess` | `false` | `false` | `true` (可修改) |

> **注意**: Local API 默认 `overrideAccess: true`，这意味着默认跳过访问控制。这是因为 Local API 通常用于服务端内部逻辑，调用者需要显式设置 `overrideAccess: false` 来启用访问控制。

---

## 5. 钩子逻辑共享机制

### 5.1 钩子执行流程

所有核心操作函数都遵循相同的钩子执行顺序。以 `createOperation` 为例：

**完整钩子执行流程** (`create.ts`):

```
┌──────────────────────────────────────────────────────────────────────┐
│                         createOperation 执行流程                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. beforeOperation (集合级)                                          │
│     └─ buildBeforeOperation() 调用 collectionConfig.hooks.beforeOperation │
│                                                                       │
│  2. 访问控制检查 (executeAccess)                                      │
│                                                                       │
│  3. beforeValidate (字段级)                                           │
│     └─ 遍历所有字段，调用每个字段的 hooks.beforeValidate              │
│                                                                       │
│  4. beforeValidate (集合级)                                           │
│     └─ collectionConfig.hooks.beforeValidate                         │
│                                                                       │
│  5. beforeChange (集合级)                                             │
│     └─ collectionConfig.hooks.beforeChange                           │
│                                                                       │
│  6. beforeChange (字段级)                                             │
│     └─ 遍历所有字段，调用每个字段的 hooks.beforeChange                │
│                                                                       │
│  7. 数据库操作 (db.create)                                            │
│                                                                       │
│  8. afterRead (字段级)                                                │
│     └─ 遍历所有字段，调用每个字段的 hooks.afterRead                   │
│                                                                       │
│  9. afterRead (集合级)                                                │
│     └─ collectionConfig.hooks.afterRead                              │
│                                                                       │
│  10. afterChange (字段级)                                             │
│      └─ 遍历所有字段，调用每个字段的 hooks.afterChange                │
│                                                                       │
│  11. afterChange (集合级)                                             │
│      └─ collectionConfig.hooks.afterChange                           │
│                                                                       │
│  12. afterOperation (集合级)                                          │
│      └─ buildAfterOperation() 调用 collectionConfig.hooks.afterOperation │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 核心钩子调用代码示例

**`beforeOperation` 钩子** (`buildBeforeOperation.ts:80-108`):

```typescript
export async function buildBeforeOperation<TOperationGeneric extends CollectionSlug>(
  operationArgs: Omit<BeforeOperationArg<TOperationGeneric>, 'context' | 'req'>,
): Promise<unknown> {
  const { args, collection, operation, overrideAccess } = operationArgs

  let newArgs = args

  if (args.collection.config.hooks?.beforeOperation?.length) {
    // 操作类型映射（向后兼容）
    const hookOperation = operationToHookOperation[operation]

    for (const hook of args.collection.config.hooks.beforeOperation) {
      const hookResult = await hook({
        args: newArgs,
        collection,
        context: args.req!.context,
        operation: hookOperation,
        overrideAccess,
        req: args.req!,
      } as BeforeOperationArg<TOperationGeneric>)

      if (hookResult !== undefined) {
        newArgs = hookResult
      }
    }
  }

  return newArgs
}
```

**`beforeRead` 和 `afterRead` 钩子** (`find.ts:292-365`):

```typescript
// beforeRead - Collection
if (collectionConfig?.hooks?.beforeRead?.length) {
  result.docs = await Promise.all(
    result.docs.map(async (doc) => {
      let docRef = doc

      for (const hook of collectionConfig.hooks.beforeRead) {
        docRef =
          (await hook({
            collection: collectionConfig,
            context: req.context,
            doc: docRef,
            overrideAccess: overrideAccess!,
            query: fullWhere,
            req,
          })) || docRef
      }

      return docRef
    }),
  )
}

// afterRead - Fields
result.docs = await Promise.all(
  result.docs.map(async (doc) =>
    afterRead<DataFromCollectionSlug<TSlug>>({
      collection: collectionConfig,
      context: req.context,
      currentDepth,
      depth: depth!,
      doc,
      draft: draftsEnabled!,
      fallbackLocale: fallbackLocale!,
      findMany: true,
      global: null,
      locale: locale!,
      overrideAccess: overrideAccess!,
      populate,
      req,
      select,
      showHiddenFields: showHiddenFields!,
    }),
  ),
)
```

### 5.3 各操作钩子对比

| 钩子类型 | find | findByID | create | updateByID | deleteByID |
|----------|------|----------|--------|------------|------------|
| beforeOperation | ✅ | ✅ | ✅ | ✅ | ✅ |
| beforeValidate | ❌ | ❌ | ✅ | ✅ | ❌ |
| beforeChange | ❌ | ❌ | ✅ | ✅ | ❌ |
| beforeRead | ✅ | ✅ | ✅ | ✅ | ✅ |
| afterRead | ✅ | ✅ | ✅ | ✅ | ✅ |
| afterChange | ❌ | ❌ | ✅ | ✅ | ✅ |
| afterOperation | ✅ | ✅ | ✅ | ✅ | ✅ |

### 5.4 字段级钩子

字段级钩子在 `fields/hooks/` 目录下统一处理，被所有操作共用：

- `packages/payload/src/fields/hooks/beforeValidate/index.ts`
- `packages/payload/src/fields/hooks/beforeChange/index.ts`
- `packages/payload/src/fields/hooks/afterRead/index.ts`
- `packages/payload/src/fields/hooks/afterChange/index.ts`

字段级钩子会递归遍历所有字段（包括嵌套在 group、array、blocks 中的字段），确保每个字段的钩子都被正确执行。

---

## 6. 语义差异对齐机制

### 6.1 请求来源标识

Payload 通过 `req.payloadAPI` 字段标识请求来源，用于处理协议间的语义差异：

**位置**: `packages/payload/src/types/index.ts:49`

```typescript
payloadAPI: 'GraphQL' | 'local' | 'REST'
```

**设置时机**:

| API 类型 | 设置位置 | 代码 |
|----------|----------|------|
| REST/GraphQL | `createPayloadRequest.ts:106` | `payloadAPI: isGraphQL ? 'GraphQL' : 'REST'` |
| Local | `createLocalReq.ts:140` | `req.payloadAPI = req?.payloadAPI \|\| 'local'` |

### 6.2 基于来源的差异处理

**示例: GraphQL 禁用 Joins** (`find.ts:184, 208`):

```typescript
// 在 findOperation 中
result = await payload.db.queryDrafts<DataFromCollectionSlug<TSlug>>({
  collection: collectionConfig.slug,
  // GraphQL 有自己的关系解析机制，禁用 joins
  joins: req.payloadAPI === 'GraphQL' ? false : sanitizedJoins,
  // ...
})

// 同样在普通查询中
result = await payload.db.find<DataFromCollectionSlug<TSlug>>({
  collection: collectionConfig.slug,
  joins: req.payloadAPI === 'GraphQL' ? false : sanitizedJoins,
  // ...
})
```

**原因分析**:
- REST API 使用 Payload 内置的 `joins` 机制处理关系字段的嵌套查询
- GraphQL 使用其原生的字段解析器（resolver）机制处理关系
- 为避免重复解析和潜在冲突，GraphQL 请求禁用 joins

### 6.3 参数格式差异与对齐

#### REST 参数格式

REST 参数通过 URL 查询字符串或请求体传递：

```
GET /api/posts?where[title][equals]=Hello&limit=10&page=1&sort=-createdAt
```

在 `handleEndpoints` 中通过 `qs` 解析：

```typescript
// createPayloadRequest.ts:70-76
const query = queryToParse
  ? qs.parse(queryToParse, {
      arrayLimit: 1000,
      depth: 10,
      ignoreQueryPrefix: true,
    })
  : {}
```

#### GraphQL 参数格式

GraphQL 参数通过查询字段参数传递：

```graphql
query {
  Posts(
    where: { title: { equals: "Hello" } }
    limit: 10
    page: 1
    sort: "-createdAt"
  ) {
    docs {
      id
      title
    }
  }
}
```

在解析器中转换为操作选项：

```typescript
// find.ts:55-67
const options = {
  collection,
  depth: 0,
  draft: args.draft,
  limit: args.limit,
  page: args.page,
  pagination: args.pagination,
  req,
  select,
  sort: sort && typeof sort === 'string' ? sort.split(',') : undefined,
  trash: args.trash,
  where: args.where,
}
```

### 6.4 响应格式差异

| 维度 | REST API | GraphQL API |
|------|----------|-------------|
| 分页结构 | 固定: `{ docs: [...], totalDocs, page, ... }` | 灵活: 由查询字段决定 |
| 错误格式 | `{ message: "...", errors: [...] }` | GraphQL 标准错误格式 |
| 字段选择 | 通过 `?select=title,content` 参数 | 通过查询字段精确控制 |
| 关系深度 | 通过 `?depth=2` 参数 | 通过查询嵌套控制 |

### 6.5 错误处理统一

所有 API 层使用统一的错误类，位于 `packages/payload/src/errors/`:

```typescript
// 核心错误类
class APIError extends Error {
  status: number
  data?: unknown
  isPublic: boolean
  // ...
}

// 常用错误子类
class Forbidden extends APIError { /* status: 403 */ }
class NotFound extends APIError { /* status: 404 */ }
class Unauthorized extends APIError { /* status: 401 */ }
class ValidationError extends APIError { /* status: 400 */ }
```

**错误处理流程**:
1. 核心操作抛出 `APIError` 或其子类
2. REST: `routeError` 函数统一处理并返回 JSON 响应
3. GraphQL: 错误被 GraphQL 执行器捕获并按规范格式化

---

## 7. 关键设计模式总结

### 7.1 架构模式

| 模式 | 应用场景 | 实现方式 |
|------|----------|----------|
| **统一操作层** | 所有 API 共享核心逻辑 | `findOperation`, `createOperation` 等核心函数 |
| **模板方法** | 固定的钩子执行顺序 | 操作函数中按预定义顺序调用各类钩子 |
| **策略模式** | 访问控制可配置 | `collectionConfig.access.*` 可传入自定义函数 |
| **中间件模式** | 钩子链式调用 | 钩子数组按顺序执行，前一个的输出是后一个的输入 |
| **上下文传递** | 请求上下文在各层间传递 | `PayloadRequest` 对象贯穿整个调用链 |

### 7.2 数据流图

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              请求入口                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                          │
│  │  REST API   │  │ GraphQL API │  │ Local API   │                          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                          │
│         │                │                │                                   │
│         ▼                ▼                ▼                                   │
│  ┌─────────────────────────────────────────────────┐                          │
│  │         参数解析与转换层                          │                          │
│  │  REST: qs.parse(queryString)                    │                          │
│  │  GraphQL: args → options                        │                          │
│  │  Local: 直接传入 options                         │                          │
│  └─────────────────────┬───────────────────────────┘                          │
│                        │                                                       │
│                        ▼                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    核心操作层 (Operations Layer)                          │ │
│  │                                                                             │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 1. beforeOperation (可选: 修改操作参数)                               │ │ │
│  │  └─────────────────────────────┬───────────────────────────────────────┘ │ │
│  │                                │                                           │ │
│  │                                ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 2. executeAccess (访问控制检查)                                        │ │ │
│  │  │    - 调用 collectionConfig.access.{operation}                          │ │ │
│  │  │    - 返回 true/false 或查询约束                                        │ │ │
│  │  └─────────────────────────────┬───────────────────────────────────────┘ │ │
│  │                                │                                           │ │
│  │                                ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 3. 数据验证与转换钩子                                                  │ │ │
│  │  │    - beforeValidate (字段级 + 集合级)                                 │ │ │
│  │  │    - beforeChange (集合级 + 字段级)                                   │ │ │
│  │  └─────────────────────────────┬───────────────────────────────────────┘ │ │
│  │                                │                                           │ │
│  │                                ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 4. 数据库操作                                                          │ │ │
│  │  │    - payload.db.find() / create() / update() / delete()              │ │ │
│  │  └─────────────────────────────┬───────────────────────────────────────┘ │ │
│  │                                │                                           │ │
│  │                                ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 5. 数据后处理钩子                                                      │ │ │
│  │  │    - afterRead (字段级 + 集合级)                                      │ │ │
│  │  │    - afterChange (字段级 + 集合级)                                    │ │ │
│  │  └─────────────────────────────┬───────────────────────────────────────┘ │ │
│  │                                │                                           │ │
│  │                                ▼                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 6. afterOperation (可选: 修改最终结果)                                │ │ │
│  │  └─────────────────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                   │                                              │
│                                   ▼                                              │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                              响应格式化层                                    │ │
│  │  REST: 返回固定结构 JSON                                                    │ │
│  │  GraphQL: 根据查询字段返回                                                  │ │
│  │  Local: 直接返回原始数据                                                    │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 最佳实践与注意事项

### 8.1 钩子编写最佳实践

1. **保持幂等性**: 钩子可能被多次调用（如重试场景），确保逻辑可重复执行
2. **使用 `context` 传递数据**: 利用 `req.context` 在钩子间共享数据，避免副作用
3. **区分 `payloadAPI` 来源**: 如有需要，可通过 `req.payloadAPI` 为不同协议做特殊处理
4. **异步钩子正确返回**: 确保异步钩子正确返回 Promise，以便 Payload 等待执行

```typescript
// 推荐写法
const beforeChangeHook: CollectionBeforeChangeHook = async ({ data, req }) => {
  // 使用 context 传递数据
  if (!req.context.processed) {
    data.slug = slugify(data.title)
    req.context.processed = true
  }
  
  // 正确返回修改后的数据
  return data
}
```

### 8.2 访问控制最佳实践

1. **返回查询约束而非仅布尔值**: 对于 `read` 操作，返回 Where 查询可实现行级权限控制
2. **利用 `req.user`**: 基于当前用户实现细粒度权限控制
3. **Local API 显式设置 `overrideAccess: false`**: 当需要在服务端执行权限检查时

```typescript
// 推荐：返回查询约束实现行级权限
const readAccess: CollectionConfig['access']['read'] = ({ req }) => {
  if (req.user?.role === 'admin') {
    return true // 管理员可查看所有
  }
  // 普通用户只能查看自己创建的或已发布的
  return {
    or: [
      { author: { equals: req.user?.id } },
      { status: { equals: 'published' } },
    ],
  }
}

// Local API 中启用访问控制
const posts = await payload.find({
  collection: 'posts',
  overrideAccess: false, // 显式启用访问控制
  user: currentUser,      // 传入当前用户
})
```

### 8.3 常见陷阱

| 陷阱 | 说明 | 解决方案 |
|------|------|----------|
| Local API 默认跳过权限 | `overrideAccess` 默认 `true` | 显式设置 `overrideAccess: false` |
| GraphQL 与 REST 关系解析差异 | GraphQL 禁用 joins | 确保关系查询逻辑与协议无关 |
| 钩子中修改 `req` 对象 | 可能影响后续操作 | 使用 `req.context` 存储临时数据 |
| 忘记返回钩子结果 | 钩子修改不生效 | 确保返回修改后的 `data` 或 `doc` |
| 非异步钩子中的异步操作 | 操作未完成就继续执行 | 使用 `async/await` 或正确返回 Promise |

### 8.4 协议选择建议

| 场景 | 推荐协议 | 理由 |
|------|----------|------|
| 简单 CRUD、移动端应用 | REST | 简单直观，缓存友好 |
| 复杂关系查询、前端应用 | GraphQL | 灵活的字段选择，减少请求次数 |
| 服务端内部逻辑 | Local API | 无 HTTP 开销，可共享事务 |
| 需要精确控制字段返回 | GraphQL | 查询即是契约 |
| 需要 HTTP 缓存 | REST | 标准 HTTP 缓存机制 |

---

## 9. 代码引用索引

### 9.1 核心文件

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 访问控制执行 | `packages/payload/src/auth/executeAccess.ts` | 14-41 |
| 创建请求对象 | `packages/payload/src/utilities/createPayloadRequest.ts` | 26-137 |
| REST 端点处理 | `packages/payload/src/utilities/handleEndpoints.ts` | 63-283 |
| 操作前钩子构建 | `packages/payload/src/collections/operations/utilities/buildBeforeOperation.ts` | 80-108 |
| Find 核心操作 | `packages/payload/src/collections/operations/find.ts` | 59-387 |
| Create 核心操作 | `packages/payload/src/collections/operations/create.ts` | 63-457 |
| GraphQL Find 解析器 | `packages/graphql/src/resolvers/collections/find.ts` | 30-71 |
| GraphQL Create 解析器 | `packages/graphql/src/resolvers/collections/create.ts` | 25-42 |
| Local Find 封装 | `packages/payload/src/collections/operations/local/find.ts` | 190-253 |

### 9.2 关键类型定义

| 类型 | 文件路径 |
|------|----------|
| PayloadRequest | `packages/payload/src/types/index.ts` |
| CollectionConfig | `packages/payload/src/config/types.ts` |
| Access | `packages/payload/src/config/types.ts` |
| Hook 类型 | `packages/payload/src/config/types.ts` |

---

## 10. 总结

Payload CMS 通过精心设计的三层架构实现了 **"一次配置，多端生效"** 的目标：

### 核心优势

1. **配置单一源**: 同一套 `CollectionConfig` 被 REST、GraphQL、Local API 共用
2. **访问控制统一**: `executeAccess` 函数确保所有协议使用相同的权限逻辑
3. **钩子执行一致**: 核心操作中固定的钩子执行顺序保证行为一致性
4. **语义差异可感知**: `req.payloadAPI` 允许钩子根据协议来源做差异化处理
5. **灵活选择协议**: 开发者可根据场景选择最合适的 API 协议，无需担心逻辑分裂

### 架构价值

这种设计的核心价值在于：

- **减少重复代码**: 业务逻辑只需编写一次
- **降低维护成本**: 修改一处，所有协议层同步更新
- **行为可预测**: 无论通过哪种协议访问，系统行为保持一致
- **渐进式采用**: 可从 REST 开始，逐步引入 GraphQL，无需重写业务逻辑

这是 Payload CMS 区别于其他 CMS 的重要架构优势，也是其被称为 "应用框架" 而非单纯 "内容管理系统" 的原因之一。
