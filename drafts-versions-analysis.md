# PayloadCMS 文档锁（多人编辑）实现分析报告

## 目录

1. [概述](#1-概述)
2. [文档锁的生效条件](#2-文档锁的生效条件)
3. [锁的触发时机](#3-锁的触发时机)
4. [绕过锁校验的写入操作](#4-绕过锁校验的写入操作)
5. [锁记录的创建和续期机制](#5-锁记录的创建和续期机制)
6. [锁的过期和删除](#6-锁的过期和删除)
7. [关键代码参考](#7-关键代码参考)

---

## 1. 概述

PayloadCMS 的文档锁（Document Lock）系统用于防止多人同时编辑同一文档时的数据冲突。当一个用户正在编辑文档时，其他用户无法修改该文档，直到锁过期或被释放。

**核心特性**：
- 基于时间的乐观锁机制
- 支持集合（Collection）和全局（Global）文档
- 支持"抢占锁"（Take Over）功能
- 可配置锁的持续时间

---

## 2. 文档锁的生效条件

### 2.1 系统级条件

文档锁功能的启用需要满足以下所有条件：

```
┌─────────────────────────────────────────────────────────────┐
│              文档锁系统启用条件                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  条件 1: 存在至少一个可锁定的集合或全局                      │
│    └─ 集合: lockDocuments !== false (默认 true)            │
│    └─ 全局: lockDocuments !== false (默认 true)            │
│                                                              │
│  条件 2: 存在至少一个认证集合 (auth collection)             │
│    └─ 用于追踪谁锁定了文档                                  │
│                                                              │
│  条件 3: payload-locked-documents 集合存在                 │
│    └─ 由系统自动创建，存储所有锁记录                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**关键代码**（`packages/payload/src/locked-documents/config.ts:8-30`）:

```typescript
export const getLockedDocumentsCollection = (config: Config): CollectionConfig | null => {
  const lockableCollections = config
    .collections!.filter((collectionConfig) => collectionConfig.lockDocuments !== false)
    .map((collectionConfig) => collectionConfig.slug)

  const lockableGlobals = config.globals
    ? config.globals.filter((globalConfig) => globalConfig.lockDocuments !== false)
    : []

  const authCollections = config
    .collections!.filter((collectionConfig) => collectionConfig.auth)
    .map((collectionConfig) => collectionConfig.slug)

  // 如果没有可锁定的集合或全局，不创建锁集合
  if (lockableCollections.length === 0 && lockableGlobals.length === 0) {
    return null
  }

  // 如果没有认证集合，无法追踪锁定者，不创建锁集合
  if (authCollections.length === 0) {
    return null
  }

  // ... 创建锁集合配置
}
```

### 2.2 集合/全局级配置

**配置类型**（`packages/payload/src/collections/config/types.ts:716-720`）:

```typescript
lockDocuments?:
  | {
      duration: number  // 锁的持续时间（秒）
    }
  | false  // 禁用文档锁
```

**配置示例**:

```typescript
// 方式 1: 使用默认配置（锁持续 300 秒 = 5 分钟）
const collection = {
  slug: 'posts',
  // lockDocuments 默认为 true
}

// 方式 2: 自定义锁持续时间
const collection = {
  slug: 'posts',
  lockDocuments: {
    duration: 600,  // 10 分钟
  },
}

// 方式 3: 禁用文档锁
const collection = {
  slug: 'posts',
  lockDocuments: false,
}
```

### 2.3 操作级条件

即使系统和集合级别都启用了文档锁，锁校验**仅在以下条件满足时才会执行**：

| 条件 | 说明 |
|------|------|
| `overrideLock === false` | 显式传入 `overrideLock: false` |
| 集合/全局 `lockDocuments !== false` | 集合级锁启用 |

**关键点**: `overrideLock` 的**默认值为 `true`**，这意味着大多数写入操作默认会**绕过**锁校验！

---

## 3. 锁的触发时机

### 3.1 锁校验的触发场景

锁校验在**写入操作**（update/delete）中执行，通过 `checkDocumentLockStatus` 函数。

**触发锁校验的操作**：

| 操作 | 位置 | 代码引用 |
|------|------|----------|
| 更新文档（按 ID） | `collections/operations/utilities/update.ts:133` | `checkDocumentLockStatus({ id, collectionSlug, overrideLock, req })` |
| 更新全局文档 | `globals/operations/update.ts:185` | `checkDocumentLockStatus({ globalSlug, overrideLock, req })` |
| 删除文档 | `collections/operations/delete.ts:155` | `checkDocumentLockStatus({ id, collectionSlug, overrideLock, req })` |

**锁校验核心逻辑**（`packages/payload/src/utilities/checkDocumentLockStatus.ts:62-97`）:

```typescript
// Only perform lock checks if overrideLock is false and locking is enabled
if (!overrideLock) {
  const lockedDocumentResult = await payload.db.find({
    collection: lockedDocumentsCollectionSlug,
    limit: 1,
    pagination: false,
    sort: '-updatedAt',
    where: lockedDocumentQuery,
  })

  const lockedDoc = lockedDocumentResult?.docs[0]
  if (lockedDoc) {
    const lastEditedAt = new Date(lockedDoc?.updatedAt).getTime()
    const now = new Date().getTime()

    const lockDuration =
      typeof lockDocumentsProp === 'object' ? lockDocumentsProp.duration : lockDurationDefault

    const lockDurationInMilliseconds = lockDuration * 1000
    const currentUserId = req.user?.id

    // document is locked by another user and the lock hasn't expired
    if (
      lockedDoc.user?.value !== currentUserId &&
      now - lastEditedAt <= lockDurationInMilliseconds
    ) {
      throw new Locked(finalLockErrorMessage)
    }
  }
}
```

### 3.2 锁校验通过的条件

锁校验通过（不抛出 `Locked` 错误）的条件：

```
┌─────────────────────────────────────────────────────────────┐
│                    锁校验通过条件                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  情况 1: overrideLock === true (默认)                       │
│    └─ 直接跳过锁校验                                        │
│                                                              │
│  情况 2: 没有活跃的锁记录                                    │
│    └─ lockedDocumentResult.docs.length === 0               │
│                                                              │
│  情况 3: 锁已过期                                            │
│    └─ now - lastEditedAt > lockDurationInMilliseconds     │
│                                                              │
│  情况 4: 当前用户是锁的持有者                                │
│    └─ lockedDoc.user?.value === currentUserId              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 绕过锁校验的写入操作

### 4.1 默认行为分析

**关键发现**: `overrideLock` 的默认值为 `true`，这意味着**大多数写入操作默认会绕过锁校验**。

#### 场景 1: Local API（默认绕过）

**代码位置**: `packages/payload/src/collections/operations/local/update.ts:85-90`

```typescript
/**
 * By default, document locks are ignored (`true`). 
 * Set to `false` to enforce locks and prevent operations when a document is locked by another user.
 * @default true
 */
overrideLock?: boolean
```

**Local API 调用示例**:

```typescript
// 情况 1: 默认行为 - 绕过锁校验
await payload.update({
  collection: 'posts',
  id: '123',
  data: { title: 'Updated' },
  // overrideLock 未指定，默认 true
})

// 情况 2: 显式绕过锁校验
await payload.update({
  collection: 'posts',
  id: '123',
  data: { title: 'Updated' },
  overrideLock: true,  // 显式绕过
})

// 情况 3: 执行锁校验
await payload.update({
  collection: 'posts',
  id: '123',
  data: { title: 'Updated' },
  overrideLock: false,  // 执行锁校验
})
```

#### 场景 2: REST API - Collection UpdateByID（执行锁校验）

**代码位置**: `packages/payload/src/collections/endpoints/updateByID.ts:33`

```typescript
const doc = await updateByIDOperation({
  // ...
  overrideLock: overrideLock ?? false,  // 默认 false，执行锁校验
  // ...
})
```

**REST API 调用示例**:

```http
# 情况 1: 默认行为 - 执行锁校验
PUT /api/posts/123
Content-Type: application/json

{"title": "Updated"}

# 情况 2: 绕过锁校验（需要显式传递）
PUT /api/posts/123?overrideLock=true
Content-Type: application/json

{"title": "Updated"}
```

#### 场景 3: REST API - Global Update（默认绕过）

**代码位置**: `packages/payload/src/globals/endpoints/update.ts:22-35`

```typescript
const result = await updateOperation({
  slug: globalConfig.slug,
  autosave,
  data: req.data!,
  // ...
  // 注意: 没有传递 overrideLock 参数！
})
```

**关键点**: Global 的 REST API 更新端点**没有显式设置 `overrideLock`**，导致使用默认值 `true`，绕过锁校验。

### 4.2 绕过锁校验的场景总结

| 场景 | overrideLock 值 | 是否执行锁校验 |
|------|-----------------|----------------|
| Local API（默认） | `true`（默认） | ❌ 绕过 |
| Local API（显式 `false`） | `false` | ✅ 执行 |
| REST API - Collection UpdateByID（默认） | `false`（`?? false`） | ✅ 执行 |
| REST API - Collection UpdateByID（`?overrideLock=true`） | `true` | ❌ 绕过 |
| REST API - Global Update（默认） | `true`（默认） | ❌ 绕过 |
| Admin UI 保存操作 | `false` | ✅ 执行 |

### 4.3 设计意图分析

这种设计有其合理性：

1. **管理后台（Admin UI）**: 需要严格的锁保护，防止多人同时编辑冲突
2. **Local API**: 常用于服务器端脚本、定时任务等，需要绕过锁以确保操作成功
3. **REST API Collection**: 可能被前端应用使用，需要锁保护
4. **REST API Global**: 设计上可能存在不一致或 bug

---

## 5. 锁记录的创建和续期机制

### 5.1 锁记录的数据结构

**集合名称**: `payload-locked-documents`

**字段定义**（`packages/payload/src/locked-documents/config.ts:32-59`）:

```typescript
const fields: CollectionConfig['fields'] = []

// 集合文档关联字段（仅当有可锁定集合时）
if (lockableCollections.length > 0) {
  fields.push({
    name: 'document',
    type: 'relationship',
    index: true,
    maxDepth: 0,
    relationTo: lockableCollections,  // 动态关联所有可锁定集合
  })
}

// 全局文档关联字段（始终存在）
fields.push({
  name: 'globalSlug',
  type: 'text',
  index: true,
})

// 锁定者字段（始终存在）
fields.push({
  name: 'user',
  type: 'relationship',
  maxDepth: 1,
  relationTo: authCollections,  // 动态关联所有认证集合
  required: true,
})
```

**锁记录示例**（集合文档）:

```json
{
  "id": "lock-record-id",
  "document": {
    "relationTo": "posts",
    "value": "post-id-123"
  },
  "globalSlug": null,
  "user": {
    "relationTo": "users",
    "value": "user-id-456"
  },
  "createdAt": "2026-05-05T10:00:00.000Z",
  "updatedAt": "2026-05-05T10:03:00.000Z"
}
```

**锁记录示例**（全局文档）:

```json
{
  "id": "lock-record-id",
  "document": null,
  "globalSlug": "menu",
  "user": {
    "relationTo": "users",
    "value": "user-id-456"
  },
  "createdAt": "2026-05-05T10:00:00.000Z",
  "updatedAt": "2026-05-05T10:03:00.000Z"
}
```

### 5.2 锁记录的创建时机

锁记录在**管理后台获取表单状态时**创建，而非在保存时创建。

**触发位置**: `packages/ui/src/utilities/buildFormState.ts:243-251`

```typescript
let lockedStateResult: LockedState | undefined

if (returnLockStatus) {
  lockedStateResult = await handleFormStateLocking({
    id,
    collectionSlug,
    globalSlug,
    req,
    updateLastEdited,  // 是否更新锁的时间（续期）
  })
}
```

**创建流程**（`packages/ui/src/utilities/handleFormStateLocking.ts:19-159`）:

```typescript
export const handleFormStateLocking = async ({
  id,
  collectionSlug,
  globalSlug,
  req,
  updateLastEdited,
}: Args): Promise<Result> => {
  // 1. 检查锁集合是否存在
  if (!req.payload.collections?.['payload-locked-documents']) {
    return result
  }

  if (id || globalSlug) {
    // 构建查询条件
    let lockedDocumentQuery = {
      and: [
        // 集合文档: { 'document.relationTo': ..., 'document.value': ... }
        // 或
        // 全局文档: { globalSlug: ... }
      ],
    }

    // 添加锁过期时间过滤（只查询活跃的锁）
    lockedDocumentQuery.and.push({
      updatedAt: {
        greater_than: new Date(now - lockDurationInMilliseconds).toISOString(),
      },
    })

    // 2. 查询是否已有活跃的锁
    const lockedDocument = await req.payload.find({
      collection: 'payload-locked-documents',
      // ...
    })

    if (lockedDocument.docs && lockedDocument.docs.length > 0) {
      // 情况 A: 已有活跃的锁
      result = {
        isLocked: true,
        lastEditedAt: lockedDocument.docs[0]?.updatedAt,
        user: lockedDocument.docs[0]?.user?.value,
      }

      // 如果当前用户是锁的持有者，且需要续期
      if (updateLastEdited && req.user && lockOwnerID === req.user.id) {
        await req.payload.db.updateOne({
          id: lockedDocument.docs[0].id,
          collection: 'payload-locked-documents',
          data: {},  // 空数据，只更新 updatedAt
          returning: false,
        })
      }
    } else {
      // 情况 B: 没有活跃的锁，创建新锁

      // 首先删除任何过期的锁
      await req.payload.db.deleteMany({
        collection: 'payload-locked-documents',
        where: deleteExpiredLocksQuery,
      })

      // 创建新锁
      await req.payload.db.create({
        collection: 'payload-locked-documents',
        data: {
          document: collectionSlug
            ? {
                relationTo: collectionSlug,
                value: id,
              }
            : undefined,
          globalSlug: globalSlug ? globalSlug : undefined,
          user: {
            relationTo: req.user.collection,
            value: req.user.id,
          },
        },
        returning: false,
      })

      result = {
        isLocked: true,
        lastEditedAt: new Date().toISOString(),
        user: req.user,
      }
    }
  }

  return result
}
```

### 5.3 锁的创建流程图

```
┌─────────────────────────────────────────────────────────────┐
│                用户打开文档编辑页面                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│            buildFormState 请求（returnLockStatus: true）    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              handleFormStateLocking 被调用                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│           查询是否存在活跃的锁记录                           │
│         (updatedAt > now - lockDuration)                    │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │                                     │
            ▼                                     ▼
┌───────────────────────┐           ┌───────────────────────┐
│   存在活跃的锁记录    │           │   无活跃的锁记录      │
└───────────────────────┘           └───────────────────────┘
            │                                     │
            ▼                                     ▼
┌───────────────────────┐           ┌───────────────────────┐
│  返回锁信息:          │           │  1. 删除过期锁记录   │
│  - isLocked: true     │           │  2. 创建新锁记录     │
│  - user: 锁定者       │           │  3. 返回锁信息       │
│  - lastEditedAt: 时间 │           └───────────────────────┘
└───────────────────────┘
```

### 5.4 锁的续期机制

锁通过**更新 `updatedAt` 字段**来续期。

**续期触发条件**:
1. `updateLastEdited: true`（由前端传入）
2. 当前用户是锁的持有者
3. 调用 `db.updateOne`（空数据，只更新 `updatedAt`）

**续期触发时机**:

管理后台在以下场景会调用 `buildFormState` 并传入 `updateLastEdited: true`:

1. **定期轮询**（Polling）: 前端定时请求以保持锁活跃
2. **用户交互**: 当用户与表单交互时触发

**续期核心代码**（`packages/ui/src/utilities/handleFormStateLocking.ts:84-96`）:

```typescript
const lockOwnerID =
  typeof lockedDocument.docs[0]?.user?.value === 'object'
    ? lockedDocument.docs[0]?.user?.value?.id
    : lockedDocument.docs[0]?.user?.value

// 只在当前用户是锁持有者时更新
if (updateLastEdited && req.user && lockOwnerID === req.user.id) {
  await req.payload.db.updateOne({
    id: lockedDocument.docs[0].id,
    collection: 'payload-locked-documents',
    data: {},           // 空数据
    returning: false,
  })
}
```

**关键点**: 调用 `db.updateOne` 时传入空对象 `{}`，这只会更新 `updatedAt` 字段（由数据库自动更新）。

---

## 6. 锁的过期和删除

### 6.1 锁的过期判断

锁通过 `updatedAt` 字段判断是否过期：

```typescript
// packages/payload/src/utilities/checkDocumentLockStatus.ts:79-95

const lastEditedAt = new Date(lockedDoc?.updatedAt).getTime()
const now = new Date().getTime()

const lockDuration =
  typeof lockDocumentsProp === 'object' ? lockDocumentsProp.duration : lockDurationDefault

const lockDurationInMilliseconds = lockDuration * 1000

// 锁未过期: now - lastEditedAt <= lockDurationInMilliseconds
if (
  lockedDoc.user?.value !== currentUserId &&
  now - lastEditedAt <= lockDurationInMilliseconds
) {
  throw new Locked(finalLockErrorMessage)
}
```

**过期计算**:

```
锁是否过期 = (当前时间 - 最后更新时间) > 锁持续时间
```

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `lastEditedAt` | 锁记录的 `updatedAt` 字段 | - |
| `lockDuration` | 锁持续时间（秒） | 300（5分钟） |

### 6.2 锁的删除时机

锁记录在以下场景被删除：

#### 场景 1: 写入操作执行时（成功写入后）

**位置**: `packages/payload/src/utilities/checkDocumentLockStatus.ts:99-105`

```typescript
// Perform the delete operation regardless of overrideLock status
await payload.db.deleteMany({
  collection: lockedDocumentsCollectionSlug,
  // Not passing req fails on postgres
  req: payload.db.name === 'mongoose' ? undefined : req,
  where: lockedDocumentQuery,
})
```

**关键点**:
- 无论 `overrideLock` 是 `true` 还是 `false`，这段代码**始终执行**
- 这意味着任何成功的写入操作都会删除该文档的锁记录

#### 场景 2: 创建新锁时（清理过期锁）

**位置**: `packages/ui/src/utilities/handleFormStateLocking.ts:98-147`

```typescript
} else {
  // If NO ACTIVE lock document exists, first delete any expired locks and then create a fresh lock
  let deleteExpiredLocksQuery

  // 构建过期锁查询条件
  // updatedAt < now - lockDuration

  await req.payload.db.deleteMany({
    collection: 'payload-locked-documents',
    where: deleteExpiredLocksQuery,
  })

  // 然后创建新锁
  await req.payload.db.create({...})
}
```

#### 场景 3: 用户主动解锁（管理后台）

当用户离开编辑页面时，管理后台可以主动删除锁记录。

**位置**: `packages/ui/src/providers/DocumentInfo/index.tsx:177-210`

```typescript
const unlockDocument = useCallback(
  async (docID: number | string, slug: string) => {
    if (!hasLockedDocumentsCollection) {
      return
    }

    // 查询锁记录
    const request = await requests.get(`${baseAPIPath}/payload-locked-documents`, {
      // ...
    })

    const { docs } = await request.json()

    // 删除锁记录
    if (docs && docs.length > 0) {
      await requests.delete(
        `${baseAPIPath}/payload-locked-documents/${docs[0].id}`,
        { credentials: 'include' },
      )
    }
  },
  // ...
)
```

### 6.3 锁的生命周期总结

```
┌─────────────────────────────────────────────────────────────┐
│                    锁的完整生命周期                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 创建:                                                    │
│     └─ 用户打开文档编辑页面                                  │
│     └─ handleFormStateLocking 检测无活跃锁                  │
│     └─ 创建新的锁记录                                       │
│                                                              │
│  2. 续期:                                                    │
│     └─ 前端定期轮询（updateLastEdited: true）               │
│     └─ 如果是锁持有者，更新 updatedAt                       │
│     └─ 锁过期时间 = updatedAt + lockDuration                │
│                                                              │
│  3. 过期:                                                    │
│     └─ 用户关闭页面且未保存                                  │
│     └─ now - updatedAt > lockDuration                       │
│     └─ 锁记录仍在数据库中，但不再被视为活跃                  │
│                                                              │
│  4. 删除:                                                    │
│     情况 A: 主动保存                                         │
│       └─ update/delete 操作执行                             │
│       └─ checkDocumentLockStatus 删除锁记录                 │
│                                                              │
│     情况 B: 下次有人打开文档                                 │
│       └─ handleFormStateLocking 清理过期锁                  │
│                                                              │
│     情况 C: 用户主动解锁                                     │
│       └─ 管理后台调用 unlockDocument                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 关键代码参考

### 7.1 核心文件列表

| 文件路径 | 职责 |
|---------|------|
| `packages/payload/src/locked-documents/config.ts` | 锁集合配置和创建逻辑 |
| `packages/payload/src/utilities/checkDocumentLockStatus.ts` | 锁校验逻辑（写入时） |
| `packages/ui/src/utilities/handleFormStateLocking.ts` | 锁创建和续期逻辑（读取时） |
| `packages/payload/src/collections/operations/utilities/update.ts` | 集合更新操作中的锁校验 |
| `packages/payload/src/globals/operations/update.ts` | 全局更新操作中的锁校验 |
| `packages/payload/src/collections/operations/delete.ts` | 删除操作中的锁校验 |
| `packages/payload/src/collections/endpoints/updateByID.ts` | REST API 更新端点 |
| `packages/payload/src/globals/endpoints/update.ts` | REST API 全局更新端点 |
| `packages/ui/src/utilities/buildFormState.ts` | 表单状态构建（触发锁创建） |

### 7.2 锁集合字段详解

**集合 Slug**: `payload-locked-documents`

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `document` | `relationship` | 关联的集合文档（多态关联） |
| `document.relationTo` | `string` | 集合 slug |
| `document.value` | `ID` | 文档 ID |
| `globalSlug` | `text` | 关联的全局文档 slug |
| `user` | `relationship` | 锁定者（多态关联） |
| `user.relationTo` | `string` | 用户集合 slug |
| `user.value` | `ID` | 用户 ID |
| `createdAt` | `datetime` | 锁创建时间 |
| `updatedAt` | `datetime` | 最后更新时间（用于判断过期） |

### 7.3 配置类型定义

**集合级别**（`packages/payload/src/collections/config/types.ts:716-720`）:

```typescript
lockDocuments?:
  | {
      duration: number  // 锁持续时间（秒）
    }
  | false
```

**操作级别**（`packages/payload/src/collections/operations/local/update.ts:87-90`）:

```typescript
/**
 * By default, document locks are ignored (`true`). 
 * Set to `false` to enforce locks.
 * @default true
 */
overrideLock?: boolean
```

### 7.4 错误类型

当文档被其他用户锁定时，抛出 `Locked` 错误：

**位置**: `packages/payload/src/errors/Locked.ts`

```typescript
class Locked extends APIError {
  constructor(message?: string) {
    super(
      message || 'This document is locked.',
      httpStatus.LOCKED,  // HTTP 423
    )
  }
}
```

---

## 8. 最佳实践和注意事项

### 8.1 使用 Local API 时的注意事项

由于 Local API 默认 `overrideLock: true`，如果需要强制执行锁校验，必须显式传入：

```typescript
// 推荐：在需要锁保护的场景显式设置
await payload.update({
  collection: 'posts',
  id: '123',
  data: { title: 'Updated' },
  overrideLock: false,  // 执行锁校验
})
```

### 8.2 锁持续时间的配置建议

| 场景 | 推荐 duration | 说明 |
|------|--------------|------|
| 短文档（如博客文章） | 300-600 秒 | 5-10 分钟 |
| 长文档（如产品描述） | 600-1800 秒 | 10-30 分钟 |
| 复杂表单（多步骤） | 1800-3600 秒 | 30-60 分钟 |

### 8.3 常见问题

**Q: 为什么我在 Local API 中更新文档时，即使被锁定也能成功？**

A: 因为 Local API 默认 `overrideLock: true`，绕过了锁校验。需要显式传入 `overrideLock: false`。

**Q: 用户打开文档后关闭浏览器，锁会持续多久？**

A: 锁会持续 `lockDuration` 秒（默认 5 分钟），之后自动过期。过期的锁会在下次有人打开文档时被清理。

**Q: 如何实现"抢占锁"功能？**

A: 管理后台的"Take Over"按钮通过更新锁记录的 `user` 字段实现：

```typescript
// 模拟抢占锁
await payload.update({
  collection: 'payload-locked-documents',
  id: lockedDocId,
  data: {
    user: { relationTo: 'users', value: currentUserId },
  },
})
```

---

*报告生成日期: 2026-05-05*
*基于 PayloadCMS 代码库分析*
