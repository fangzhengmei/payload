# PayloadCMS 访问控制完整链路分析

## 一、概述

PayloadCMS 的访问控制系统采用了一套灵活且强大的机制，允许开发者在集合配置中定义访问控制函数，这些函数能够在运行时动态计算权限，甚至返回查询条件来限制数据库查询。

## 二、访问规则定义层

### 2.1 配置结构

访问控制规则在集合配置的 `access` 字段中定义，位于 `CollectionConfig` 类型中：

```typescript
// packages/payload/src/collections/config/types.ts:586-594

export type CollectionConfig<TSlug extends CollectionSlug = any> = {
  // ...
  access?: {
    admin?: ({ req }: { req: PayloadRequest }) => boolean | Promise<boolean>
    create?: Access
    delete?: Access
    read?: Access
    readVersions?: Access
    unlock?: Access
    update?: Access
  }
  // ...
}
```

### 2.2 Access 类型定义

`Access` 类型定义在 `config/types.ts` 中：

```typescript
// packages/payload/src/config/types.ts:334-357

/**
 * This result is calculated on the server
 * and then sent to the client allowing the dashboard to show accessible data and actions.
 *
 * If the result is `true`, the user has access.
 * If the result is an object, it is interpreted as a MongoDB query.
 *
 * @example `{ createdBy: { equals: id } }`
 *
 * @example `{ tenant: { in: tenantIds } }`
 */
export type AccessResult = boolean | Where

export type AccessArgs<TData = any> = {
  /**
   * The relevant resource that is being accessed.
   *
   * `data` is null when a list is requested
   */
  data?: TData
  /** ID of the resource being accessed */
  id?: DefaultDocumentIDType
  /** If true, the request is for a static file */
  isReadingStaticFile?: boolean
  /** The original request that requires an access check */
  req: PayloadRequest
}

/**
 * Access function runs on the server
 * and is sent to the client allowing the dashboard to show accessible data and actions.
 */
export type Access<TData = any> = (args: AccessArgs<TData>) => AccessResult | Promise<AccessResult>
```

### 2.3 关键特性

1. **返回值类型灵活**：`AccessResult` 可以是 `boolean` 或 `Where` 查询对象
   - `true`：完全允许访问
   - `false`：完全禁止访问
   - `Where` 对象：允许访问但附加查询条件过滤

2. **上下文丰富**：`AccessArgs` 提供了完整的上下文：
   - `req`：包含用户信息、payload 实例等
   - `id`：操作的文档 ID（单文档操作时）
   - `data`：文档数据（更新/创建操作时）

## 三、用户身份注入上下文

### 3.1 请求创建与身份验证流程

每个 HTTP 请求进入系统时，都会通过 `createPayloadRequest` 函数创建 `PayloadRequest` 对象，并在这个过程中完成用户身份的注入。

```typescript
// packages/payload/src/utilities/createPayloadRequest.ts:26-138

export const createPayloadRequest = async ({
  canSetHeaders,
  config: configPromise,
  params,
  payloadInstanceCacheKey,
  request,
}: Args): Promise<PayloadRequest> => {
  const cookies = parseCookies(request.headers)
  const payload = await getPayload({
    config: configPromise,
    cron: true,
    key: payloadInstanceCacheKey,
  })

  // ... 初始化 i18n、locale、query 等 ...

  const customRequest: CustomPayloadRequestProperties = {
    context: {},
    fallbackLocale: fallbackLocale!,
    // ... 其他属性
    user: null,  // 初始化为 null
  }

  const req: PayloadRequest = Object.assign(request, customRequest)

  req.payloadDataLoader = getDataLoader(req)

  // 执行认证策略，获取用户身份
  const { responseHeaders, user } = await executeAuthStrategies({
    canSetHeaders,
    headers: req.headers,
    isGraphQL,
    payload,
  })

  req.user = user  // 将用户注入请求上下文

  if (responseHeaders) {
    req.responseHeaders = responseHeaders
  }

  return req
}
```

### 3.2 认证策略执行

`executeAuthStrategies` 函数遍历所有注册的认证策略，直到找到一个能成功验证用户的策略：

```typescript
// packages/payload/src/auth/executeAuthStrategies.ts:5-38

export const executeAuthStrategies = async (
  args: AuthStrategyFunctionArgs,
): Promise<AuthStrategyResult> => {
  let result: AuthStrategyResult = { user: null }

  if (!args.payload.authStrategies?.length) {
    return result
  }

  for (const strategy of args.payload.authStrategies) {
    // 添加配置的 AuthStrategy name 到策略函数参数
    args.strategyName = strategy.name
    args.isGraphQL = Boolean(args.isGraphQL)
    args.canSetHeaders = Boolean(args.canSetHeaders)

    try {
      const authResult = await strategy.authenticate(args)
      if (authResult.responseHeaders) {
        authResult.responseHeaders = mergeHeaders(
          result.responseHeaders || new Headers(),
          authResult.responseHeaders || new Headers(),
        )
      }
      result = authResult
    } catch (err) {
      logError({ err, payload: args.payload })
    }

    if (result.user) {
      return result  // 找到用户，立即返回
    }
  }
  return result
}
```

### 3.3 JWT 认证策略

默认的 JWT 策略负责从请求中提取并验证 JWT Token：

```typescript
// packages/payload/src/auth/strategies/jwt.ts:77-133

export const JWTAuthentication: AuthStrategyFunction = async ({
  headers,
  isGraphQL = false,
  payload,
  strategyName = 'local-jwt',
}) => {
  try {
    const token = extractJWT({ headers, payload })

    if (!token) {
      if (headers.get('DisableAutologin') !== 'true') {
        return await autoLogin({ isGraphQL, payload, strategyName })
      }
      return { user: null }
    }

    // 验证 JWT Token
    const secretKey = new TextEncoder().encode(payload.secret)
    const { payload: decodedPayload } = await jwtVerify<JWTToken>(token, secretKey)
    const collection = payload.collections[decodedPayload.collection]

    // 根据 Token 中的信息获取用户文档
    const user = (await payload.findByID({
      id: decodedPayload.id,
      collection: decodedPayload.collection,
      depth: isGraphQL ? 0 : collection!.config.auth.depth,
    })) as AuthStrategyResult['user']

    // 验证用户状态（如邮箱验证、会话有效性等）
    if (user && (!collection!.config.auth.verify || user._verified)) {
      if (collection!.config.auth.useSessions) {
        const existingSession = (user.sessions || []).find(({ id }) => id === decodedPayload.sid)

        if (!existingSession || !decodedPayload.sid) {
          return { user: null }
        }

        user._sid = decodedPayload.sid
      }

      user.collection = collection!.config.slug
      user._strategy = strategyName
      return { user }
    } else {
      // ... autoLogin  fallback
    }
  } catch (ignore) {
    // ... autoLogin fallback
  }
}
```

### 3.4 完整身份注入流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户身份注入上下文流程                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  HTTP 请求进入
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              createPayloadRequest()                          │
  │  1. 解析 cookies、query parameters                           │
  │  2. 初始化 payload 实例                                       │
  │  3. 初始化 i18n、locale 等                                  │
  │  4. 创建 DataLoader (用于批量关联查询)                       │
  │  5. 调用 executeAuthStrategies() 进行身份验证               │
  │  6. 将返回的 user 赋值给 req.user                            │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │            executeAuthStrategies()                           │
  │  遍历所有认证策略 (JWT策略、自定义策略等):                    │
  │  for (const strategy of payload.authStrategies) {           │
  │    result = strategy.authenticate(args)                      │
  │    if (result.user) return result  // 找到用户立即返回       │
  │  }                                                           │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              JWTAuthentication()  (默认策略)                │
  │                                                              │
  │  1. extractJWT({ headers, payload })                        │
  │     - 按顺序从 Cookie、Authorization header、query 参数提取  │
  │                                                              │
  │  2. 无 Token: 检查 autoLogin 配置                           │
  │                                                              │
  │  3. 有 Token: jwtVerify() 解码验证                          │
  │     - 从解码结果获取 collection 和 id                       │
  │                                                              │
  │  4. payload.findByID() 获取用户文档                         │
  │     - 注意: 这里使用的是 Local API，会再次经过访问控制       │
  │                                                              │
  │  5. 验证用户状态:                                            │
  │     - 邮箱验证 (如果 auth.verify 启用)                      │
  │     - 会话有效性 (如果 auth.useSessions 启用)               │
  │                                                              │
  │  6. 返回 { user: user }                                     │
  │     - user.collection = 集合 slug                          │
  │     - user._strategy = 策略名称                            │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  req.user = user  ← 用户身份注入完成，可在 access 函数中使用
```

## 四、鉴权执行层

### 4.1 executeAccess 核心函数

访问控制函数的执行由 `executeAccess` 函数处理：

```typescript
// packages/payload/src/auth/executeAccess.ts:1-42

import type { Access, AccessResult } from '../config/types.js'
import type { PayloadRequest } from '../types/index.js'

import { Forbidden } from '../errors/index.js'

type OperationArgs = {
  data?: any
  disableErrors?: boolean
  id?: number | string
  isReadingStaticFile?: boolean
  req: PayloadRequest
}

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

  // 没有定义 access 函数时的默认行为
  if (req.user) {
    return true  // 登录用户默认有权限
  }

  if (!disableErrors) {
    throw new Forbidden(req.t)  // 未登录用户默认无权限
  }
  return false
}
```

### 4.2 执行逻辑分析

| 场景 | 行为 |
|------|------|
| 定义了 access 函数且返回 `true` | 返回 `true`，允许访问 |
| 定义了 access 函数且返回 `Where` | 返回 `Where` 对象，用于查询过滤 |
| 定义了 access 函数且返回 `false` | 抛出 `Forbidden` 错误（除非 `disableErrors=true`） |
| 未定义 access 函数但有用户 | 返回 `true`（默认登录用户可访问） |
| 未定义 access 函数且无用户 | 抛出 `Forbidden` 错误 |

### 4.3 overrideAccess 机制

在所有操作中都支持 `overrideAccess` 参数，用于绕过访问控制（主要用于内部操作或钩子）：

```typescript
// 例如在 findOperation 中
if (!overrideAccess) {
  accessResult = await executeAccess({ disableErrors, req }, collectionConfig.access.read)
  // ...
}
```

**关键使用场景**：
- `dataloader` 批量加载关联数据时（继承父级请求的 `overrideAccess`）
- `autoLogin` 功能获取用户时
- 内部钩子需要绕过权限检查时

## 五、查询层权限裁剪

### 5.1 权限与查询的结合

当访问控制函数返回 `Where` 查询对象时，需要将其与用户查询合并。这由 `combineQueries` 函数处理：

```typescript
// packages/payload/src/database/combineQueries.ts:1-22

import type { Where } from '../types/index.js'

import { hasWhereAccessResult } from '../auth/index.js'

/**
 * Combines two queries into a single query, using an AND operator
 */
export const combineQueries = (where: Where, access: boolean | Where): Where => {
  if (!where && !access) {
    return {}
  }

  const and: Where[] = where ? [where] : []

  if (hasWhereAccessResult(access)) {
    and.push(access)
  }

  return {
    and,
  }
}
```

### 5.2 hasWhereAccessResult 辅助函数

```typescript
// packages/payload/src/auth/types.ts:324-326

export function hasWhereAccessResult(result: boolean | Where): result is Where {
  return result && typeof result === 'object'
}
```

### 5.3 查询操作中的权限裁剪流程

以 `findOperation` 为例：

```typescript
// packages/payload/src/collections/operations/find.ts:114-149

// /////////////////////////////////////
// Access
// /////////////////////////////////////

let accessResult: AccessResult

if (!overrideAccess) {
  accessResult = await executeAccess({ disableErrors, req }, collectionConfig.access.read)

  // If errors are disabled, and access returns false, return empty results
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

// /////////////////////////////////////
// Find
// /////////////////////////////////////

// ...

let fullWhere = combineQueries(where!, accessResult!)
sanitizeWhereQuery({ fields: collectionConfig.flattenedFields, payload, where: fullWhere })

// 处理 Join 查询的权限
const sanitizedJoins = await sanitizeJoinQuery({
  collectionConfig,
  joins,
  overrideAccess: overrideAccess!,
  req,
})

// ...

result = await payload.db.find<DataFromCollectionSlug<TSlug>>({
  collection: collectionConfig.slug,
  // ...
  joins: req.payloadAPI === 'GraphQL' ? false : sanitizedJoins,
  where: fullWhere,  // 使用合并后的查询
})
```

### 5.4 更新/删除操作的权限裁剪

以 `updateByIDOperation` 为例，展示单文档操作的权限处理：

```typescript
// packages/payload/src/collections/operations/updateByID.ts:115-177

// /////////////////////////////////////
// Access
// /////////////////////////////////////

const accessResults = !overrideAccess
  ? await executeAccess({ id, data, req }, collectionConfig.access.update)
  : true
const hasWherePolicy = hasWhereAccessResult(accessResults)

// /////////////////////////////////////
// Retrieve document (使用权限查询来获取要更新的文档)
// /////////////////////////////////////

const where = { id: { equals: id } }

let fullWhere = combineQueries(where, accessResults)

// 特殊处理: 如果是删除到回收站操作，还需要检查 delete 权限
const isTrashAttempt =
  collectionConfig.trash &&
  typeof data === 'object' &&
  data !== null &&
  'deletedAt' in data &&
  data.deletedAt != null

if (isTrashAttempt && !overrideAccess) {
  const deleteAccessResult = await executeAccess({ data, req }, collectionConfig.access.delete)
  fullWhere = combineQueries(fullWhere, deleteAccessResult)
}

// Exclude trashed documents when trash: false
fullWhere = appendNonTrashedFilter({
  enableTrash: collectionConfig.trash,
  trash,
  where: fullWhere,
})

const findOneArgs: FindOneArgs = {
  collection: collectionConfig.slug,
  locale: locale!,
  req,
  where: fullWhere,  // 使用合并后的查询
}

// 关键: 使用权限查询来查找文档
const docWithLocales = await getLatestCollectionVersion<...>({
  id,
  config: collectionConfig,
  payload,
  query: findOneArgs,
  req,
})

// 根据是否有 where 策略区分错误类型
// 这是一个安全设计: 不向未授权用户暴露文档是否存在
if (!docWithLocales && !hasWherePolicy) {
  throw new NotFound(req.t)  // 文档确实不存在
}
if (!docWithLocales && hasWherePolicy) {
  throw new Forbidden(req.t)   // 可能文档存在但用户无权访问
}
if (!docWithLocales) {
  throw new NotFound(req.t)
}

// ... 后续执行 updateDocument 更新文档
```

**重要修正**：更新操作的完整流程是：
1. **先查后改**：使用合并了权限条件的查询来查找文档
2. **找不到文档时**：根据是否有 `where` 权限策略区分 `NotFound` 和 `Forbidden`
3. **找到文档后**：执行 `updateDocument` 进行实际更新
4. **更新后**：通过 `afterRead` 处理字段级权限和关联数据填充

## 六、关联读取的权限叠加

当读取包含 `Relationship`、`Join` 或 `Upload` 字段的文档时，Payload 会通过 `depth` 参数自动填充关联文档。这个过程中，每个关联集合的访问控制都会被独立检查。

### 6.1 Join 字段的权限处理

`sanitizeJoinQuery` 函数负责处理 Join 字段的权限验证和查询合并：

```typescript
// packages/payload/src/database/sanitizeJoinQuery.ts:17-87

const sanitizeJoinFieldQuery = async ({
  collectionSlug,
  errors,
  join,
  joinsQuery,
  overrideAccess,
  promises,
  req,
}: {
  collectionSlug: string
  errors: { path: string }[]
  join: SanitizedJoin
  joinsQuery: JoinQuery
  overrideAccess: boolean
  promises: Promise<void>[]
  req: PayloadRequest
}) => {
  const { joinPath } = join

  if ((joinsQuery as any)[joinPath] === false) {
    return
  }

  const joinCollectionConfig = req.payload.collections[collectionSlug]!.config

  // 关键: 对关联集合执行访问控制检查
  const accessResult = !overrideAccess
    ? await executeAccess({ disableErrors: true, req }, joinCollectionConfig.access.read)
    : true

  // 如果关联集合的 access 函数返回 false，完全禁止该 join
  if (accessResult === false) {
    ;(joinsQuery as any)[joinPath] = false
    return
  }

  if (!(joinsQuery as any)[joinPath]) {
    ;(joinsQuery as any)[joinPath] = {}
  }

  const joinQuery = (joinsQuery as any)[joinPath]

  if (!joinQuery.where) {
    joinQuery.where = {}
  }

  // 合并字段定义的默认 where 条件
  if (join.field.where) {
    joinQuery.where = combineQueries(joinQuery.where, join.field.where)
  }

  promises.push(
    validateQueryPaths({
      collectionConfig: joinCollectionConfig,
      errors,
      overrideAccess,
      polymorphicJoin: Array.isArray(join.field.collection),
      req,
      where: joinQuery.where,
    }),
  )

  // 关键: 如果 access 返回 Where 对象，合并到查询中
  if (typeof accessResult === 'object') {
    sanitizeWhereQuery({
      fields: joinCollectionConfig.flattenedFields,
      payload: req.payload,
      where: accessResult,
    })
    joinQuery.where = combineQueries(joinQuery.where, accessResult)
  }
}
```

### 6.2 Relationship 字段的权限处理（DataLoader 路径）

对于普通的 `Relationship` 字段，关联数据通过 `DataLoader` 批量加载。这个过程在 `afterRead` 钩子的 `relationshipPopulationPromise` 中触发：

```typescript
// packages/payload/src/fields/hooks/afterRead/relationshipPopulationPromise.ts:26-95

const populate = async ({
  currentDepth,
  data,
  dataReference,
  depth,
  draft,
  fallbackLocale,
  field,
  index,
  key,
  locale,
  overrideAccess,  // 继承父级的 overrideAccess
  populateArg,
  req,
  showHiddenFields,
}: PopulateArgs) => {
  // ...

  const relatedCollection =
    req.payload.collections[relation as keyof typeof req.payload.collections]

  if (relatedCollection) {
    // ...

    const shouldPopulate = depth && currentDepth <= depth

    if (shouldPopulate) {
      // 通过 DataLoader 批量加载
      relationshipValue = await req.payloadDataLoader.load(
        createDataloaderCacheKey({
          collectionSlug: relatedCollection.config.slug,
          currentDepth: currentDepth + 1,
          depth,
          docID: id as string,
          draft,
          fallbackLocale: fallbackLocale!,
          locale: locale!,
          overrideAccess,  // 传递 overrideAccess
          populate: populateArg,
          select:
            populateArg?.[relatedCollection.config.slug] ??
            relatedCollection.config.defaultPopulate,
          showHiddenFields,
          transactionID: req.transactionID!,
        }),
      )
    }

    // 关键点: ids are visible regardless of access controls
    // 如果无法 populate（权限问题或 depth 限制），只显示原始 ID
    if (relatedCollection.config.trash && relationshipValue) {
      if ((relationshipValue as Record<string, unknown>).deletedAt) {
        relationshipValue = null
      }
    } else if (!relationshipValue) {
      // 权限被拒绝时，返回原始 ID 而不是 null
      relationshipValue = id
    }
    // ...
  }
}
```

### 6.3 DataLoader 的权限处理

DataLoader 使用 `batchAndLoadDocs` 函数批量处理关联查询，它会调用 `payload.find` 并传递 `overrideAccess` 参数：

```typescript
// packages/payload/src/collections/dataloader.ts:22-164

const batchAndLoadDocs =
  (req: PayloadRequest): BatchLoadFn<string, TypeWithID> =>
  async (keys: readonly string[]): Promise<TypeWithID[]> => {
    // ...

    for (const [batchKey, ids] of Object.entries(batchByFindArgs)) {
      const [
        transactionID,
        collection,
        depth,
        currentDepth,
        locale,
        fallbackLocale,
        overrideAccess,  // 从缓存键解析
        showHiddenFields,
        draft,
        select,
        populate,
      ] = JSON.parse(batchKey)

      req.transactionID = transactionID

      // 关键: 使用 payload.find 进行批量查询，自动应用访问控制
      const result = await payload.find({
        collection,
        currentDepth,
        depth,
        disableErrors: true,  // 不抛出错误，返回空结果
        draft,
        fallbackLocale,
        locale,
        overrideAccess: Boolean(overrideAccess),
        pagination: false,
        populate,
        req,
        select: selectWithDeletedAt,
        showHiddenFields: Boolean(showHiddenFields),
        ...(enableTrash ? { trash: true } : {}),
        where: {
          id: {
            in: ids,
          },
        },
      })

      // ...
    }

    return docs as TypeWithID[]
  }
```

### 6.4 关联读取权限叠加流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      关联读取权限叠加流程                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  场景: 查询 Posts 集合，Post 有 author 字段关联 Users 集合
        Post.access.read = { status: { equals: 'published' } }
        User.access.read = { role: { in: ['admin', 'editor'] } }

       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  第1层: 主集合查询 (Posts)                                    │
  │                                                              │
  │  findOperation({                                              │
  │    collection: 'posts',                                       │
  │    where: { category: { equals: 'news' } },                  │
  │    depth: 2,                                                  │
  │  })                                                           │
  │                                                              │
  │  执行:                                                        │
  │  1. executeAccess(Post.access.read)                          │
  │     → 返回 { status: { equals: 'published' } }               │
  │                                                              │
  │  2. combineQueries(                                          │
  │       { category: { equals: 'news' } },                      │
  │       { status: { equals: 'published' } }                    │
  │     )                                                         │
  │     → { and: [                                                │
  │           { category: { equals: 'news' } },                  │
  │           { status: { equals: 'published' } }                │
  │         ] }                                                   │
  │                                                              │
  │  3. db.find(where: fullWhere)                               │
  │     → 返回符合条件的 Post 文档                                │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  第2层: afterRead 钩子处理                                  │
  │                                                              │
  │  afterRead({                                                 │
  │    doc: post,                                                │
  │    overrideAccess: false,  ← 注意这个值会传递给关联查询    │
  │    depth: 2,                                                 │
  │  })                                                          │
  │                                                              │
  │  traverseFields 遍历字段:                                    │
  │  - 遇到 author (Relationship 字段)                          │
  │  - 调用 relationshipPopulationPromise                       │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  第3层: DataLoader 批量加载关联数据                          │
  │                                                              │
  │  relationshipPopulationPromise 中:                          │
  │                                                              │
  │  payloadDataLoader.load(                                     │
  │    createDataloaderCacheKey({                                │
  │      collectionSlug: 'users',                                │
  │      currentDepth: 2,                                        │
  │      overrideAccess: false,  ← 继承父级                    │
  │      // ...                                                  │
  │    })                                                         │
  │  )                                                            │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  第4层: 关联集合查询 (Users) - 再次经过完整权限检查         │
  │                                                              │
  │  batchAndLoadDocs 中调用:                                    │
  │                                                              │
  │  payload.find({                                              │
  │    collection: 'users',                                      │
  │    overrideAccess: false,                                    │
  │    disableErrors: true,  ← 不抛出错误，只返回空结果        │
  │    where: { id: { in: [authorId1, authorId2, ...] } },     │
  │  })                                                           │
  │                                                              │
  │  这会触发 Users 集合的完整访问控制:                          │
  │                                                              │
  │  1. findOperation 开始                                       │
  │  2. executeAccess(User.access.read)                         │
  │     → 返回 { role: { in: ['admin', 'editor'] } }            │
  │                                                              │
  │  3. combineQueries(                                          │
  │       { id: { in: [authorId1, authorId2, ...] } },          │
  │       { role: { in: ['admin', 'editor'] } }                 │
  │     )                                                         │
  │     → { and: [                                                │
  │           { id: { in: [...] } },                              │
  │           { role: { in: ['admin', 'editor'] } }              │
  │         ] }                                                   │
  │                                                              │
  │  4. db.find(where: fullWhere)                               │
  │     → 只返回 role 是 admin 或 editor 的用户                  │
  └─────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  第5层: 结果处理                                             │
  │                                                              │
  │  在 relationshipPopulationPromise 中:                        │
  │                                                              │
  │  if (!relationshipValue) {                                   │
  │    // 关键点: ids are visible regardless of access controls  │
  │    // 如果权限被拒绝，返回原始 ID 而不是 null                │
  │    relationshipValue = id                                    │
  │  }                                                            │
  │                                                              │
  │  这意味着:                                                    │
  │  - 如果 author 是 admin → post.author = { id: 1, name: 'A', role: 'admin' }
  │  - 如果 author 是普通用户 → post.author = 2 (只显示 ID)    │
  └─────────────────────────────────────────────────────────────┘
```

### 6.5 关联读取权限关键点总结

| 关键点 | 说明 |
|--------|------|
| **权限独立检查** | 每个关联集合都独立执行自己的 `access.read` 函数 |
| **overrideAccess 传递** | 父级查询的 `overrideAccess` 值会传递给所有关联查询 |
| **DataLoader 复用** | 同一请求中相同的关联查询会被缓存和复用 |
| **权限失败的处理** | 关联权限被拒绝时，只显示原始 ID 而不是 null（不暴露文档不存在） |
| **depth 限制** | 关联深度受 `depth` 参数限制，但权限检查不受 depth 影响 |

## 七、权限结果的不同落地方式

PayloadCMS 中有两种截然不同的权限结果处理方式：

### 7.1 权限面板/UI 显示模式 (`getAccessResults`)

这种模式用于 Admin UI 中的权限显示，例如决定哪些按钮可见、哪些集合显示在侧边栏等。

```typescript
// packages/payload/src/auth/getAccessResults.ts:10-81

export async function getAccessResults({
  req,
}: GetAccessResultsArgs): Promise<SanitizedPermissions> {
  const results = {
    collections: {},
    globals: {},
  } as Permissions

  // ...

  // 关键: fetchData = false
  const collectionPermissions = await getEntityPermissions({
    blockReferencesPermissions,
    entity: collection,
    entityType: 'collection',
    fetchData: false,  // ← 不获取数据，只计算权限
    operations: collectionOperations,
    req,
  })

  // ...
}
```

### 7.2 `getEntityPermissions` 的双重模式

`getEntityPermissions` 函数根据 `fetchData` 参数的不同有不同的行为：

```typescript
// packages/payload/src/utilities/getEntityPermissions/getEntityPermissions.ts:86-240

export async function getEntityPermissions<...>(...): Promise<...> {
  // Phase 1: 并行执行所有 access 函数
  const accessResults: {
    operation: keyof typeof entity.access
    result: Promise<boolean | Where>
  }[] = []

  for (const _operation of operations) {
    const operation = _operation as keyof typeof entity.access
    const accessFunction = entity.access[operation]

    if (typeof accessFunction === 'function') {
      accessResults.push({
        operation,
        result: Promise.resolve(accessFunction({ id, data, req })) as Promise<boolean | Where>,
      })
    } else {
      // 没有定义 access 函数，默认登录用户有权限
      entityPermissions[operation] = {
        permission: isLoggedIn,
      }
    }
  }

  // Phase 2: 处理 where 查询（关键差异点）
  for (const { operation, result: accessResult } of resolvedAccessResults) {
    if (typeof accessResult === 'object') {
      // accessResult 是 Where 对象
      processWhereQuery({
        accessResult,
        entityPermissions,
        fetchData,  // ← 这个参数决定行为
        operation,
        req,
        // ...
      })
    } else {
      // accessResult 是 boolean
      entityPermissions[operation] = { permission: !!accessResult }
    }
  }
  // ...
}
```

### 7.3 `processWhereQuery` 的关键差异

```typescript
// packages/payload/src/utilities/getEntityPermissions/getEntityPermissions.ts:242-304

const processWhereQuery = ({
  accessResult,
  entityPermissions,
  fetchData,  // 关键参数
  // ...
}: {
  // ...
  fetchData: boolean
  // ...
}): void => {
  if (fetchData) {
    // fetchData = true 模式（用于编辑单文档时的权限检查）
    // 执行实际的数据库查询来验证用户是否有权限访问该文档

    // 检查缓存
    let cached = whereQueryCache.find((entry) => isDeepStrictEqual(entry.where, accessResult))

    if (!cached) {
      // 执行数据库查询: "是否存在满足 where 条件的文档?"
      cached = {
        result: entityDocExists({
          id,
          slug,
          entityType,
          locale,
          operation,
          req,
          where: accessResult,  // 用权限条件去查文档
        }),
        where: accessResult,
      }
      whereQueryCache.push(cached)
    }

    // 根据查询结果设置 permission
    wherePromises.push(
      cached.result.then((hasPermission) => {
        entityPermissions[operation] = {
          permission: hasPermission,  // 实际查询结果
          where: accessResult,        // 同时保留 where 条件
        } as Permission
      }),
    )
  } else {
    // fetchData = false 模式（用于权限面板/列表视图 UI）
    // 不执行数据库查询，直接假设用户"可能有权限"

    // TODO: 4.0: Investigate defaulting to `false` here
    // 代码注释承认这是一个安全隐患:
    // "如果 where 查询被返回但因为没有文档数据而被忽略，我们应该默认为 false"

    entityPermissions[operation] = {
      permission: true,    // ← 总是返回 true!
      where: accessResult,  // 只保留 where 条件供后续使用
    } as Permission
  }
}
```

### 7.4 两种模式的对比

| 特性 | 权限面板模式 (`fetchData=false`) | 实际查询模式 |
|------|-----------------------------------|--------------|
| **触发场景** | Admin 初始化、权限面板、侧边栏显示 | find/findByID/update/delete 等实际数据操作 |
| **access 函数** | 执行，获取返回值 | 执行，获取返回值 |
| **返回 boolean** | `permission = boolean` | `permission = boolean`，直接决定允许/拒绝 |
| **返回 Where** | `permission = true` + 保留 `where` | `where` 被 `combineQueries` 合并到数据库查询 |
| **是否查库** | 否 | 是（通过 `combineQueries` 合并后查询） |
| **安全性** | UI 层面的显示控制 | 数据库层面的强制过滤 |
| **存在的问题** | Where 权限在 UI 层面被"乐观"地认为有权限 | 无（数据库强制过滤） |

### 7.5 设计意图与潜在问题

**设计意图**：
1. **权限面板模式**：用于快速判断 UI 元素是否显示。如果 `access` 返回 `Where`，很难在列表层面判断用户"到底有没有权限"，所以乐观地显示按钮，实际权限由后续操作强制执行。

2. **实际查询模式**：通过 `combineQueries` 将 `Where` 条件合并到数据库查询，从底层确保用户只能看到符合条件的数据。

**潜在问题（代码注释中承认）**：
```typescript
// packages/payload/src/utilities/getEntityPermissions/getEntityPermissions.ts:298-302

// TODO: 4.0: Investigate defaulting to `false` here, if where query is returned but ignored as we don't
// have the document data available. This seems more secure.
// Alternatively, we could set permission to a third state, like 'unknown'.
// Even after calling sanitizePermissions, the permissions will still be true if the where query is returned but ignored as we don't have the document data available.
```

这意味着：
- 如果 `access.read = { author: { equals: req.user.id } }`
- 在权限面板中，用户会看到"有 read 权限"
- 但实际上用户只能看到自己创建的文档
- 这可能导致 UI 显示与实际能力不一致（例如用户看到"新建"按钮但无法保存，或者看到列表按钮但列表为空）

### 7.6 两种模式的流程图对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    权限面板模式 vs 实际查询模式对比                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────┐    ┌───────────────────────────────┐
│      权限面板模式              │    │        实际查询模式            │
│   (fetchData = false)         │    │   (find/findByID 等)          │
├───────────────────────────────┤    ├───────────────────────────────┤
│                               │    │                               │
│  触发: /api/access 端点       │    │  触发: GET /api/posts         │
│        Admin UI 初始化         │    │        实际数据查询           │
│                               │    │                               │
│  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │
│  │ 1. 执行 access 函数     │  │    │  │ 1. 执行 access 函数     │  │
│  │                         │  │    │  │                         │  │
│  │ access.read({ req })    │  │    │  │ access.read({ req })    │  │
│  │ → 返回 { author: ... }  │  │    │  │ → 返回 { author: ... }  │  │
│  └───────────┬─────────────┘  │    │  └───────────┬─────────────┘  │
│              │                  │    │              │                  │
│              ▼                  │    │              ▼                  │
│  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │
│  │ 2. 处理 Where 结果      │  │    │  │ 2. 处理 Where 结果      │  │
│  │                         │  │    │  │                         │  │
│  │ processWhereQuery       │  │    │  │ combineQueries          │  │
│  │ fetchData: false        │  │    │  │ (userWhere, accessWhere)│  │
│  │                         │  │    │  │                         │  │
│  │ 结果:                    │  │    │  │ 结果:                    │  │
│  │ {                        │  │    │  │ {                        │  │
│  │   permission: true,  ←──┼──┼────┼──┼─── 不合并到查询         │  │
│  │   where: { author: ... } │  │    │  │   and: [                 │  │
│  │ }                        │  │    │  │     userWhere,          │  │
│  │                          │  │    │  │     { author: ... }  ←──┼──┤
│  └─────────────────────────┘  │    │  │   ]                      │  │
│              │                  │    │  │ }                        │  │
│              ▼                  │    │  └───────────┬─────────────┘  │
│  ┌─────────────────────────┐  │    │              │                  │
│  │ 3. 发送到前端           │  │    │              ▼                  │
│  │                         │  │    │  ┌─────────────────────────┐  │
│  │ permissions: {          │  │    │  │ 3. 数据库查询           │  │
│  │   read: {               │  │    │  │                         │  │
│  │     permission: true    │  │    │  │ db.find({               │  │
│  │   }                     │  │    │  │   where: fullWhere  ←──┼──┤
│  │ }                       │  │    │  │ })                       │  │
│  └─────────────────────────┘  │    │  └───────────┬─────────────┘  │
│              │                  │    │              │                  │
│              ▼                  │    │              ▼                  │
│  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │
│  │ 4. 前端行为             │  │    │  │ 4. 返回结果             │  │
│  │                         │  │    │  │                         │  │
│  │ - 显示"查看"按钮         │  │    │  │ - 只返回符合条件的文档  │  │
│  │ - 显示集合在侧边栏       │  │    │  │ - 权限在数据库层强制执行│  │
│  │ - 但实际可能看不到数据   │  │    │  └─────────────────────────┘  │
│  └─────────────────────────┘  │    │                               │
│                               │    │                               │
└───────────────────────────────┘    └───────────────────────────────┘
```

## 八、完整调用链路

### 8.1 查询操作 (find) 完整链路

```
1. HTTP 请求进入
        ↓
2. createPayloadRequest()
   - 解析 JWT Token
   - 执行认证策略
   - 注入 req.user
        ↓
3. 路由层调用 findOperation
        ↓
4. buildBeforeOperation (可选)
        ↓
5. 检查 overrideAccess
        ↓
6. ┌─ overrideAccess === false ──────────────────────────────────┐
   │  executeAccess({ disableErrors, req }, collectionConfig.access.read)
   │         ↓
   │  调用用户定义的 access.read({ req, ... })
   │         ↓
   │  返回 AccessResult (boolean | Where)
   │         ↓
   │  accessResult === false → 返回空结果
   └──────────────────────────────────────────────────────────────┘
        ↓
7. combineQueries(where, accessResult)
   - 用户查询 + 权限查询 使用 AND 合并
        ↓
8. sanitizeJoinQuery() - 处理 Join 字段的权限叠加
        ↓
9. 调用数据库层 payload.db.find({ where: fullWhere, joins: sanitizedJoins, ... })
        ↓
10. 执行 afterRead 钩子
    - traverseFields 遍历所有字段
    - relationshipPopulationPromise 处理关联字段
    - DataLoader 批量加载时再次经过完整权限检查
        ↓
11. 返回结果
```

### 8.2 更新操作 (updateByID) 完整链路（修正版）

```
1. HTTP 请求 PATCH /api/collection/:id
        ↓
2. createPayloadRequest() - 身份注入
        ↓
3. 路由层调用 updateByIDOperation
        ↓
4. buildBeforeOperation
        ↓
5. 检查 overrideAccess
        ↓
6. ┌─ overrideAccess === false ──────────────────────────────────┐
   │  executeAccess({ id, data, req }, collectionConfig.access.update)
   │         ↓
   │  调用用户定义的 access.update({ req, id, data })
   │         ↓
   │  返回 AccessResult
   │         ↓
   │  结果为 false → 抛出 Forbidden
   └──────────────────────────────────────────────────────────────┘
        ↓
7. combineQueries({ id: { equals: id } }, accessResults)
        ↓
8. 特殊处理: 如果是删除到回收站操作
   - 额外检查 access.delete
   - combineQueries(fullWhere, deleteAccessResult)
        ↓
9. 使用合并后的查询获取文档
   - getLatestCollectionVersion({ query: { where: fullWhere } })
   - 关键: 这一步确保只能获取有权限更新的文档
        ↓
10. 根据结果判断错误类型:
    - 无 where 策略且文档不存在 → NotFound (文档确实不存在)
    - 有 where 策略且文档不存在 → Forbidden (可能存在但无权)
        ↓
11. 执行 updateDocument - 实际更新文档
        ↓
12. 执行 afterRead 钩子
    - 字段级权限检查
    - 关联数据填充 (同样经过权限检查)
        ↓
13. 执行 afterChange 钩子 (如果配置了)
        ↓
14. buildAfterOperation
        ↓
15. 返回更新后的文档
```

### 8.3 关联读取深度权限检查链路

```
场景: depth = 2，Post → Author → Company

1. 初始查询 Post
   - Post.access.read 检查
   - combineQueries 合并后查询
        ↓
2. afterRead 处理 Post 的 author 字段
   - overrideAccess = false 传递给 DataLoader
        ↓
3. DataLoader 批量查询 Author
   - payload.find({ collection: 'authors', overrideAccess: false, ... })
   - 这会触发 Author 集合的完整权限检查:
     * Author.access.read 执行
     * combineQueries 合并
     * 数据库查询
        ↓
4. afterRead 处理 Author 的 company 字段
   - 同样的流程
        ↓
5. DataLoader 批量查询 Company
   - Company.access.read 检查
   - ...以此类推，直到达到 depth 限制
```

## 九、关键代码位置汇总

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| 请求创建与身份注入 | `utilities/createPayloadRequest.ts` | 26-138 |
| 认证策略执行 | `auth/executeAuthStrategies.ts` | 5-38 |
| JWT 认证策略 | `auth/strategies/jwt.ts` | 77-133 |
| 集合配置 Access 类型 | `collections/config/types.ts` | 586-594 |
| Access 核心类型定义 | `config/types.ts` | 334-357 |
| 执行 access 函数 | `auth/executeAccess.ts` | 1-42 |
| 合并查询与权限 | `database/combineQueries.ts` | 1-22 |
| 查询操作权限处理 | `collections/operations/find.ts` | 114-149 |
| 更新操作权限处理 | `collections/operations/updateByID.ts` | 115-177 |
| 删除操作权限处理 | `collections/operations/deleteByID.ts` | 80-108 |
| Join 字段权限处理 | `database/sanitizeJoinQuery.ts` | 17-87 |
| 关联字段填充 | `fields/hooks/afterRead/relationshipPopulationPromise.ts` | 26-95 |
| DataLoader 批量查询 | `collections/dataloader.ts` | 22-164 |
| 获取所有权限 | `auth/getAccessResults.ts` | 10-81 |
| 获取实体权限详情 | `utilities/getEntityPermissions/getEntityPermissions.ts` | 86-240, 242-304 |
| 判断 Where 权限结果 | `auth/types.ts` | 324-326 |

## 十、设计亮点与注意事项

### 10.1 设计亮点

1. **灵活的权限模型**：支持布尔权限和基于查询的权限，后者允许细粒度的行级权限控制

2. **查询合并策略**：使用 `AND` 操作符合并用户查询和权限查询，确保权限过滤始终生效，用户无法绕过

3. **错误区分**：在单文档操作中区分 `NotFound` 和 `Forbidden` 错误，提升安全性（不向未授权用户暴露文档是否存在）

4. **overrideAccess 机制**：允许内部操作绕过权限检查，便于钩子和内部服务使用

5. **关联权限独立检查**：每个关联集合都独立执行自己的访问控制，确保深度查询时的权限安全

6. **DataLoader 复用**：同一请求中相同的关联查询会被缓存，提升性能的同时保持权限检查

7. **类型安全**：完整的 TypeScript 类型定义，包括 `AccessArgs`、`AccessResult` 等

### 10.2 注意事项与潜在问题

1. **权限面板的乐观假设**：
   - `fetchData=false` 模式下，返回 `Where` 的 `access` 函数会被乐观地认为 `permission: true`
   - 这可能导致 UI 显示与实际能力不一致
   - 代码注释承认这是一个需要在 v4.0 中调查的安全问题

2. **关联查询的 ID 可见性**：
   - 关联权限被拒绝时，返回原始 ID 而不是 `null`
   - 这是设计行为：`ids are visible regardless of access controls`
   - 但需要注意这可能泄露文档 ID 的存在

3. **性能考虑**：
   - 深度关联查询时，每个层级都会触发独立的权限检查和数据库查询
   - 使用 DataLoader 缓解 N+1 问题，但每个集合的 `access` 函数仍会执行

4. **Trash 功能的特殊处理**：
   - 更新操作中设置 `deletedAt` 会额外检查 `access.delete`
   - 查询时需要显式传递 `trash: true` 才能看到已删除的文档

### 10.3 安全最佳实践

1. **始终使用 Where 权限进行行级控制**：
   ```typescript
   // 推荐：返回 Where 条件，在数据库层强制执行
   read: ({ req }) => {
     if (req.user?.roles?.includes('admin')) return true
     return { author: { equals: req.user?.id } }
   }
   ```

2. **不要依赖 UI 级别的权限**：
   - 权限面板的结果只用于 UI 显示控制
   - 始终确保实际数据操作有相应的 `access` 函数保护

3. **关联集合也要配置权限**：
   - 不要假设主集合有权限就意味着关联集合也有权限
   - 每个集合的 `access.read` 都会独立检查

4. **注意 `overrideAccess` 的使用**：
   - 只有在绝对必要时才使用 `overrideAccess: true`
   - 自定义端点和钩子中要特别注意

## 十一、使用示例

```typescript
// 集合配置中的访问控制示例
const PostsCollection: CollectionConfig = {
  slug: 'posts',
  access: {
    // 所有人可读取已发布文章，登录用户可读取自己的草稿
    read: ({ req }) => {
      if (req.user?.roles?.includes('admin')) {
        return true  // 管理员可看全部
      }
      // 普通用户只能看到自己的草稿或已发布的文章
      return {
        or: [
          { _status: { equals: 'published' } },
          { 
            and: [
              { _status: { equals: 'draft' } },
              { author: { equals: req.user?.id } }
            ]
          }
        ]
      }
    },
    // 只有登录用户可创建
    create: ({ req }) => !!req.user,
    // 只能更新自己的文章或管理员
    update: ({ req, id }) => {
      if (req.user?.roles?.includes('admin')) return true
      // 返回 Where 条件：只能更新 author = 当前用户的文档
      return { author: { equals: req.user?.id } }
    },
    // 只能删除自己的文章或管理员
    delete: ({ req }) => {
      if (req.user?.roles?.includes('admin')) return true
      return { author: { equals: req.user?.id } }
    }
  },
  fields: [
    {
      name: 'title',
      type: 'text',
      required: true,
    },
    {
      name: 'author',
      type: 'relationship',
      relationTo: 'users',
      required: true,
      // 注意：关联查询时，users 集合的 access.read 也会被检查
    },
    {
      name: '_status',
      type: 'select',
      options: [
        { label: 'Draft', value: 'draft' },
        { label: 'Published', value: 'published' },
      ],
      defaultValue: 'draft',
    },
  ]
}
```
