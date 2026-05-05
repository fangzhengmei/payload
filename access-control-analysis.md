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

## 三、鉴权执行层

### 3.1 executeAccess 核心函数

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

  if (req.user) {
    return true
  }

  if (!disableErrors) {
    throw new Forbidden(req.t)
  }
  return false
}
```

### 3.2 执行逻辑分析

| 场景 | 行为 |
|------|------|
| 定义了 access 函数且返回 `true` | 返回 `true`，允许访问 |
| 定义了 access 函数且返回 `Where` | 返回 `Where` 对象，用于查询过滤 |
| 定义了 access 函数且返回 `false` | 抛出 `Forbidden` 错误（除非 `disableErrors=true`） |
| 未定义 access 函数但有用户 | 返回 `true`（默认登录用户可访问） |
| 未定义 access 函数且无用户 | 抛出 `Forbidden` 错误 |

### 3.3 overrideAccess 机制

在所有操作中都支持 `overrideAccess` 参数，用于绕过访问控制（主要用于内部操作或钩子）：

```typescript
// 例如在 findOperation 中
if (!overrideAccess) {
  accessResult = await executeAccess({ disableErrors, req }, collectionConfig.access.read)
  // ...
}
```

## 四、查询层权限裁剪

### 4.1 权限与查询的结合

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

### 4.2 hasWhereAccessResult 辅助函数

```typescript
// packages/payload/src/auth/types.ts:324-326

export function hasWhereAccessResult(result: boolean | Where): result is Where {
  return result && typeof result === 'object'
}
```

### 4.3 查询操作中的权限裁剪流程

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

// ...

result = await payload.db.find<DataFromCollectionSlug<TSlug>>({
  collection: collectionConfig.slug,
  // ...
  where: fullWhere,  // 使用合并后的查询
})
```

### 4.4 更新/删除操作的权限裁剪

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
// Retrieve document
// /////////////////////////////////////

const where = { id: { equals: id } }

let fullWhere = combineQueries(where, accessResults)

// ...

const docWithLocales = await getLatestCollectionVersion<...>({
  id,
  config: collectionConfig,
  payload,
  query: findOneArgs,
  req,
})

// 根据是否有 where 策略区分错误类型
if (!docWithLocales && !hasWherePolicy) {
  throw new NotFound(req.t)  // 文档不存在
}
if (!docWithLocales && hasWherePolicy) {
  throw new Forbidden(req.t)   // 文档存在但无权访问
}
```

## 五、完整调用链路

### 5.1 查询操作 (find) 完整链路

```
1. 用户请求 GET /api/collection
        ↓
2. 路由层调用 findOperation
        ↓
3. buildBeforeOperation (可选)
        ↓
4. 检查 overrideAccess
        ↓
5. ┌─ overrideAccess === false ──────────────────────────────────┐
   │  executeAccess({ disableErrors, req }, collectionConfig.access.read)
   │         ↓
   │  调用用户定义的 access.read({ req, ... })
   │         ↓
   │  返回 AccessResult (boolean | Where)
   │         ↓
   │  accessResult === false → 返回空结果
   └──────────────────────────────────────────────────────────────┘
        ↓
6. combineQueries(where, accessResult)
   - 用户查询 + 权限查询 使用 AND 合并
        ↓
7. 调用数据库层 payload.db.find({ where: fullWhere, ... })
        ↓
8. 执行 afterRead 钩子
        ↓
9. 返回结果
```

### 5.2 更新操作 (updateByID) 完整链路

```
1. 用户请求 PATCH /api/collection/:id
        ↓
2. 路由层调用 updateByIDOperation
        ↓
3. buildBeforeOperation
        ↓
4. 检查 overrideAccess
        ↓
5. ┌─ overrideAccess === false ──────────────────────────────────┐
   │  executeAccess({ id, data, req }, collectionConfig.access.update)
   │         ↓
   │  调用用户定义的 access.update({ req, id, data })
   │         ↓
   │  返回 AccessResult
   │         ↓
   │  结果为 false → 抛出 Forbidden
   └──────────────────────────────────────────────────────────────┘
        ↓
6. combineQueries({ id: { equals: id } }, accessResults)
        ↓
7. 使用合并后的查询获取文档 (getLatestCollectionVersion)
        ↓
8. 根据结果判断错误类型：
   - 无 where 策略且文档不存在 → NotFound
   - 有 where 策略且文档不存在 → Forbidden（可能有权限问题）
        ↓
9. 执行 updateDocument
        ↓
10. 执行 afterRead, afterDelete 等钩子
        ↓
11. 返回结果
```

## 六、权限获取系统

### 6.1 getAccessResults

用于获取用户对所有集合/全局的权限，主要用于 Admin UI 显示：

```typescript
// packages/payload/src/auth/getAccessResults.ts:10-81

export async function getAccessResults({
  req,
}: GetAccessResultsArgs): Promise<SanitizedPermissions> {
  const results = {
    collections: {},
    globals: {},
  } as Permissions
  const { payload, user } = req

  // 检查 admin 访问权限
  const userCollectionConfig = 
    user && user.collection ? payload?.collections?.[user.collection]?.config : null

  if (userCollectionConfig && payload.config.admin.user === user?.collection) {
    results.canAccessAdmin = userCollectionConfig.access.admin
      ? await userCollectionConfig.access.admin({ req })
      : isLoggedIn
  }

  // 遍历所有集合计算权限
  await Promise.all(
    payload.config.collections.map(async (collection) => {
      const collectionOperations: AllOperations[] = ['create', 'read', 'update', 'delete']
      // ... 根据配置添加 unlock, readVersions 等

      const collectionPermissions = await getEntityPermissions({
        blockReferencesPermissions,
        entity: collection,
        entityType: 'collection',
        fetchData: false,  // 不获取数据，只计算权限
        operations: collectionOperations,
        req,
      })
      results.collections![collection.slug] = collectionPermissions
    }),
  )

  // 遍历所有全局...
  
  return sanitizePermissions(results)
}
```

### 6.2 getEntityPermissions

为单个实体（集合/全局）计算详细权限：

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

  // Phase 2: 处理 where 查询（如果 fetchData=true）
  for (const { operation, result: accessResult } of resolvedAccessResults) {
    if (typeof accessResult === 'object') {
      // accessResult 是 Where 对象
      processWhereQuery({
        // ...
        accessResult,
        fetchData,
        // ...
      })
    } else {
      // accessResult 是 boolean
      entityPermissions[operation] = { permission: !!accessResult }
    }
  }

  // 计算字段级权限...
  populateFieldPermissions({ ... })

  return entityPermissions
}
```

## 七、关键代码位置汇总

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| 集合配置 Access 类型 | `collections/config/types.ts` | 586-594 |
| Access 核心类型定义 | `config/types.ts` | 334-357 |
| 执行 access 函数 | `auth/executeAccess.ts` | 1-42 |
| 合并查询与权限 | `database/combineQueries.ts` | 1-22 |
| 查询操作权限处理 | `collections/operations/find.ts` | 114-149 |
| 更新操作权限处理 | `collections/operations/updateByID.ts` | 115-177 |
| 删除操作权限处理 | `collections/operations/deleteByID.ts` | 80-108 |
| 获取所有权限 | `auth/getAccessResults.ts` | 10-81 |
| 获取实体权限详情 | `utilities/getEntityPermissions/getEntityPermissions.ts` | 86-240 |
| 判断 Where 权限结果 | `auth/types.ts` | 324-326 |

## 八、流程图总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PayloadCMS 访问控制完整链路                            │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │  1. 配置定义  │
  │ CollectionConfig.access.read/update/delete/create
  │ 类型: Access = (args) => AccessResult (boolean | Where)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  2. 操作入口  │
  │ findOperation / updateByIDOperation / deleteByIDOperation
  └──────┬───────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    3. overrideAccess 检查                     │
  │  ┌─────────────────┐          ┌──────────────────────────┐  │
  │  │ overrideAccess  │          │ overrideAccess = false   │  │
  │  │     = true      │          │   (正常流程)              │  │
  │  │   (跳过权限)    │          │                          │  │
  │  └────────┬────────┘          └────────────┬─────────────┘  │
  │           │                                │                  │
  │           ▼                                ▼                  │
  │    直接执行查询              executeAccess(accessFn, args)    │
  │                                    │                          │
  │                                    ▼                          │
  │                         返回 AccessResult:                    │
  │                         • true  → 完全允许                    │
  │                         • false → 抛出 Forbidden             │
  │                         • Where → 需附加查询条件              │
  └─────────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    4. 查询层权限裁剪                          │
  │                                                              │
  │   combineQueries(userWhere, accessResult)                   │
  │                                                              │
  │   结果: {                                                    │
  │     and: [                                                   │
  │       userWhere,      // 用户原始查询                        │
  │       accessWhere     // 权限过滤条件 (如果是 Where)         │
  │     ]                                                        │
  │   }                                                          │
  └─────────────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────┐
  │  5. 数据库查询 │
  │ payload.db.find/update/delete
  │ 使用合并后的 where 条件
  └──────┬───────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    6. 结果处理                               │
  │  • find 操作: accessResult=false → 返回空结果               │
  │  • update/delete 操作:                                       │
  │    - 无 where 策略且无文档 → NotFound                        │
  │    - 有 where 策略且无文档 → Forbidden (可能是权限问题)      │
  └─────────────────────────────────────────────────────────────┘
```

## 九、设计亮点

1. **灵活的权限模型**：支持布尔权限和基于查询的权限，后者允许细粒度的行级权限控制

2. **查询合并策略**：使用 `AND` 操作符合并用户查询和权限查询，确保权限过滤始终生效

3. **错误区分**：在单文档操作中区分 NotFound 和 Forbidden 错误，提升安全性（不暴露文档是否存在）

4. **overrideAccess 机制**：允许内部操作绕过权限检查，便于钩子和内部服务使用

5. **并行执行**：在 `getEntityPermissions` 中并行执行所有 access 函数，提升性能

6. **类型安全**：完整的 TypeScript 类型定义，包括 `AccessArgs`、`AccessResult` 等

## 十、使用示例

```typescript
// 集合配置中的访问控制示例
const PostsCollection: CollectionConfig = {
  slug: 'posts',
  access: {
    // 所有人可读取已发布文章
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
      return { author: { equals: req.user?.id } }
    },
    // 只能删除自己的文章或管理员
    delete: ({ req }) => {
      if (req.user?.roles?.includes('admin')) return true
      return { author: { equals: req.user?.id } }
    }
  },
  fields: [
    // ... 字段定义
  ]
}
```
