# Payload CMS Drafts & Versions 机制深度分析

## 1. 自动保存 (Autosave) 机制

### 1.1 核心设计原理

Payload CMS 的自动保存功能建立在 **Versions** 和 **Drafts** 两个核心功能之上。其设计目标是：
- 确保编辑过程中的数据不会丢失
- 避免频繁创建版本导致数据库膨胀
- 不影响已发布的文档状态

### 1.2 配置选项

```typescript
// Autosave 类型定义 (packages/payload/src/versions/types.ts)
export type Autosave = {
  interval?: number        // 可覆盖自动保存间隔
  showSaveDraftButton?: boolean  // 是否显示手动保存按钮，默认 false
}
```

**默认值说明：**
- 类型注释和文档里仍写 `800ms`
- 当前运行时代码实际从 `versionDefaults.autosaveInterval` 取默认值，仓库现值是 **2000ms**
- 如果在集合或 Global 配置里显式传 `autosave.interval`，则以配置值覆盖

**配置示例：**
```typescript
versions: {
  drafts: {
    autosave: {
      interval: 1500,      // 1.5秒间隔
      showSaveDraftButton: true
    }
  }
}
```

### 1.3 前端周期触发机制

自动保存的周期写入通过 **Debounce (防抖)** 机制实现：

1. **触发条件**：用户编辑文档时的字段变化
2. **防抖处理**：每次输入后等待 `interval` 毫秒；当前代码默认值是 2000ms
3. **取消机制**：如果在等待期间有新的输入，重置计时器
4. **真正发请求前再比对一次表单值**：只在去抖后的表单值相对上一次快照发生变化时，才发送 `draft=true&autosave=true` 的保存请求

### 1.4 服务端存储优化策略

**关键实现**：`updateLatestVersion` 函数 (`packages/payload/src/versions/updateLatestVersion.ts`)

```
自动保存存储流程：
1. 查询最新版本
2. 检查是否为 autosave 版本 (autosave: true)
3. 如果是 → 更新该版本（而非创建新版本）
4. 如果不是 → 创建新的 autosave 版本
```

**核心代码逻辑** (`saveVersion.ts:73-84`)：
```typescript
if (unpublish || autosave) {
  result = await updateLatestVersion({
    id,
    collection,
    global,
    now,
    payload,
    req,
    // 关键：只更新标记为 autosave 的版本
    shouldUpdate: autosave ? (v) => 'autosave' in v && v.autosave === true : undefined,
    versionData,
  })
}
```

### 1.5 数据库结构

**版本表结构** (`_{collectionSlug}_versions`)：

```json
{
  "_id": "版本唯一ID",
  "parent": "父文档ID",
  "autosave": true,           // 标记为自动保存版本
  "version": {
    // 完整的文档数据
    "_status": "draft",
    // ... 其他字段
  },
  "createdAt": "创建时间",
  "updatedAt": "更新时间"
}
```

### 1.6 自动保存 vs 手动保存

| 特性 | Autosave | 手动 Save Draft |
|------|----------|-----------------|
| 触发方式 | 防抖计时器 | 用户点击按钮 |
| 版本标记 | `autosave: true` | `autosave: false` |
| 版本创建 | 始终更新同一个 | 每次创建新版本 |
| 对发布版本影响 | 无 | 无 |
| 验证 | 默认不验证，开启 drafts.validate 后会验证 | 同样受 drafts.validate 控制 |

---

## 2. 版本树与发布状态管理

### 2.1 版本树架构

Payload CMS 采用 **线性版本链** 而非复杂的分支树结构：

```
主文档表 (主集合)       版本表 (_versions)
┌─────────────┐        ┌───────────────────┐
│  发布版本    │        │  V1 (published)   │ ← 首次发布
│             │        ├───────────────────┤
│  _status    │        │  V2 (draft)       │ ← 草稿1
│  'published'│        ├───────────────────┤
│             │        │  V3 (draft)       │ ← 草稿2 (最新)
│  updatedAt  │        ├───────────────────┤
│  2024-01-01 │        │  V4 (autosave)    │ ← 自动保存
└─────────────┘        └───────────────────┘
```

### 2.2 状态字段 `_status`

**字段定义** (`packages/payload/src/versions/baseFields.ts`)：
```typescript
{
  name: '_status',
  type: 'select',
  defaultValue: 'draft',      // 默认草稿状态
  options: [
    { value: 'draft', label: 'Draft' },
    { value: 'published', label: 'Published' }
  ],
  localized: Boolean(localized) // 支持本地化状态
}
```

### 2.3 Admin UI 三种状态显示

| 状态 | 条件 | 含义 |
|------|------|------|
| **Draft** | 从未发布，只有草稿版本 | 新文档，未发布 |
| **Published** | 已发布，无更新草稿 | 当前无未发布更改 |
| **Changed** | 已发布，但有更新草稿 | 有未发布的更改 |

### 2.4 发布流程详解

**核心逻辑** (`packages/payload/src/collections/operations/utilities/update.ts:112-127`)

```typescript
const isSavingDraft =
  Boolean(draftArg && hasDraftsEnabled(collectionConfig)) &&
  data._status !== 'published' &&
  !publishAllLocales
```

**发布判定条件：**
- `draft: false` 或 `_status: 'published'` → 触发发布
- `draft: true` 且 `_status !== 'published'` → 仅保存草稿

### 2.5 写入位置决策

| 操作 | draft 参数 | _status | 主集合 | 版本表 |
|------|-----------|---------|--------|--------|
| 创建 | 任意 | 省略 | ✅ 更新为 draft | ✅ 创建版本 |
| 创建 | 任意 | published | ✅ 更新为 published | ✅ 创建版本 |
| 更新 | true | 省略/draft | ❌ 不更新 | ✅ 仅版本表 |
| 更新 | true | published | ✅ 更新 (优先级高) | ✅ 创建版本 |
| 更新 | false/省略 | 省略 | ✅ 更新为 draft | ✅ 创建版本 |
| 更新 | false/省略 | published | ✅ 更新为 published | ✅ 创建版本 |

### 2.6 版本数量限制

**配置**：
- 集合：`versions.maxPerDoc` (默认 100)
- Global：`versions.max`

**清理逻辑** (`enforceMaxVersions.ts`)：
1. 每次创建新版本后检查
2. 用 `updatedAt` 倒序找到第 `max + 1` 条记录作为截断点
3. 删除该截断点及更旧的版本记录；逻辑本身不会单独排除 autosave
4. 设为 0 表示不限制

### 2.7 本地化状态 (Localize Status)

当 `localizeStatus: true` 时：
- `_status` 字段支持多语言独立控制
- 可单独发布/取消发布某个语言版本
- 需要 `publishAllLocales` 参数控制全语言发布

---

## 3. 多人编辑冲突处理机制

### 3.1 文档锁定 (Document Locking) 完整生命周期

#### 3.1.1 锁定功能何时生效

**启用条件**：

锁定功能的启用遵循以下优先级：

```
1. 必须有至少一个 auth 集合
   └─ 没有认证用户 → 无法追踪锁定者 → 不创建 locked-documents 集合

2. 集合/Global 级别配置
   ├─ lockDocuments: false → 明确禁用
   ├─ lockDocuments: true 或 { duration: X } → 明确启用
   └─ 未定义 (undefined) → 默认启用
   
3. 最终判定：lockDocuments !== false 才启用
```

**自动禁用锁定的系统集合** (`locked-documents/config.ts:10-14`)：

```typescript
// 这些集合自动设置 lockDocuments: false，防止递归锁定
collections.filter((collectionConfig) => collectionConfig.lockDocuments !== false)

// 系统内部禁用锁定的集合：
// - queues
// - query-presets  
// - preferences
// - locked-documents 本身 (防止递归)
// - kv-adapter (DatabaseKVAdapter)
// - migrations
```

**锁定集合创建条件** (`config.ts:26-30`)：
```typescript
// 如果没有 auth 集合，无法追踪是谁锁定了文档
// 所以不创建 locked-documents 集合
if (authCollections.length === 0) {
  return null
}
```

#### 3.1.2 锁定集合结构

**自动创建的锁定集合** (`packages/payload/src/locked-documents/config.ts`)：

```typescript
{
  slug: 'payload-locked-documents',
  lockDocuments: false,  // 自身不锁定，防止递归
  fields: [
    { 
      name: 'document', 
      type: 'relationship', 
      relationTo: 所有可锁定集合,
      admin: { readOnly: true }
    },
    { 
      name: 'globalSlug', 
      type: 'text',
      admin: { readOnly: true, condition: ({ document }) => !document }
    },
    { 
      name: 'user', 
      type: 'relationship', 
      relationTo: 认证集合, 
      required: true,
      admin: { readOnly: true }
    }
  ]
}
```

**锁定记录数据结构**：
```json
{
  "_id": "锁记录ID",
  "document": {
    "relationTo": "posts",
    "value": "文档ID"
  },
  "globalSlug": null,  // 仅用于 Global
  "user": {
    "relationTo": "users",
    "value": "用户ID"
  },
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-01T00:05:00.000Z"  // 关键：用于判断锁是否过期
}
```

#### 3.1.3 锁记录的创建与续期机制

**锁在哪里创建？**

锁记录的创建和续期**不在写入操作时**，而是在**Admin UI 编辑页面的表单状态请求时**。

**核心代码位置**：
- 前端：`packages/ui/src/views/Edit/index.tsx:468-566` (onChange 回调)
- 服务端：表单状态处理时的 `handleFormStateLocking`

**前端触发时机** (`Edit/index.tsx:484-491`)：

```typescript
const currentTime = Date.now()
const timeSinceLastUpdate = currentTime - editSessionStartTime

// 每 10 秒才会触发一次锁续期
const updateLastEdited = isLockingEnabled && timeSinceLastUpdate >= 10000 // 10 seconds

if (updateLastEdited) {
  setEditSessionStartTime(currentTime)
}
```

**服务端调用参数**：
```typescript
const result = await getFormState({
  // ... 其他参数
  returnLockStatus: isLockingEnabled,   // 是否返回锁状态
  updateLastEdited,                      // 是否更新锁时间（每10秒一次）
})
```

**锁的完整生命周期**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        锁的生命周期                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 锁的创建 (Create)                                                │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 用户A打开文档编辑页 → 首次字段变更触发 onChange         │    │
│     │ → 调用 getFormState(updateLastEdited=true)             │    │
│     │ → 服务端创建/更新 payload-locked-documents 记录        │    │
│     │ → 返回 lockedState 给前端                                │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              ↓                                        │
│  2. 锁的续期 (Renew)                                                  │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 用户A持续编辑，每 10 秒                                  │    │
│     │ → 触发 updateLastEdited=true                            │    │
│     │ → 更新锁记录的 updatedAt 字段                            │    │
│     │ → 锁过期时间 = updatedAt + duration (默认5分钟)         │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              ↓                                        │
│  3. 锁的释放 (Release)                                                │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 方式A：保存/发布文档                                      │    │
│     │   onSave 成功后 → setDocumentIsLocked(false)            │    │
│     │                                                          │    │
│     │ 方式B：离开编辑页面                                       │    │
│     │   handleLeaveConfirm → unlockDocument API 调用          │    │
│     │                                                          │    │
│     │ 方式C：锁过期                                             │    │
│     │   当前时间 > updatedAt + lockDuration                   │    │
│     │   → 其他用户可获取锁                                     │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**锁过期判断逻辑** (`Edit/index.tsx:191-192`)：
```typescript
const lockExpiryTime = lastUpdateTime + lockDurationInMilliseconds
const isLockExpired = Date.now() > lockExpiryTime
```

#### 3.1.4 写入前锁校验流程

**服务端校验函数** (`packages/payload/src/utilities/checkDocumentLockStatus.ts`)

**核心参数**：
```typescript
export const checkDocumentLockStatus = async ({
  id,
  collectionSlug,
  globalSlug,
  lockDurationDefault = 300,  // 默认 5 分钟
  lockErrorMessage,
  overrideLock = true,         // ⚠️ 关键：默认绕过锁检查！
  req,
}: CheckDocumentLockStatusArgs): Promise<void>
```

**校验流程详解**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    checkDocumentLockStatus 执行流程                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  第1步：检查锁定功能是否启用                                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ const isLockingEnabled = lockDocumentsProp !== false        │  │
│  │ if (!isLockingEnabled) return                                 │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                        │
│  第2步：检查 overrideLock 参数                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ if (overrideLock === true) {                                 │  │
│  │   // ⚠️ 直接跳过锁检查！                                      │  │
│  │   // 但仍会执行第4步：删除过期锁                              │  │
│  │   跳到第4步                                                   │  │
│  │ }                                                             │  │
│  │ else {                                                        │  │
│  │   // overrideLock === false → 执行严格的锁检查               │  │
│  │   继续第3步                                                   │  │
│  │ }                                                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                        │
│  第3步：严格锁检查 (仅当 overrideLock=false 时)                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 3.1 查询锁定记录                                              │  │
│  │     where: {                                                 │  │
│  │       'document.relationTo': collectionSlug,                │  │
│  │       'document.value': id,                                  │  │
│  │       updatedAt: { greater_than: now - lockDuration }      │  │
│  │     }                                                         │  │
│  │                                                              │  │
│  │ 3.2 检查锁定者                                                │  │
│  │     ├─ 无锁定记录 → 允许写入 ✓                               │  │
│  │     ├─ 锁定者是当前用户 → 允许写入 ✓                         │  │
│  │     └─ 锁定者是其他用户 → 抛出 Locked 错误 ✗                │  │
│  │                                                              │  │
│  │ 抛出的错误信息：                                              │  │
│  │ "Document with ID ${id} is currently locked by another     │  │
│  │  user and cannot be updated."                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                        │
│  第4步：删除过期锁 (无论 overrideLock 是什么，都会执行)              │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 删除所有过期的锁定记录：                                      │  │
│  │ updatedAt < now - lockDuration                               │  │
│  │                                                              │  │
│  │ 这一步很重要：防止锁记录无限积累，保持数据库清洁              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.1.5 锁参数配置详解

```typescript
// 集合/Global 配置中的 lockDocuments 选项

// 方式1：默认启用 (不写任何配置)
// lockDocuments 未定义 → 等同于 lockDocuments: true
// 默认锁持续时间：300 秒 (5 分钟)

// 方式2：明确启用，自定义时长
lockDocuments: {
  duration: 600,  // 10 分钟，单位：秒
}

// 方式3：明确禁用
lockDocuments: false
```

**锁持续时间计算** (`checkDocumentLockStatus.ts:82-86`)：
```typescript
const lockDocumentsProp = collectionConfig?.lockDocuments

const lockDuration =
  typeof lockDocumentsProp === 'object' 
    ? lockDocumentsProp.duration 
    : lockDurationDefault  // 300 秒

const lockDurationInMilliseconds = lockDuration * 1000
```

---

### 3.2 哪些操作会绕过锁校验

#### 3.2.1 关键发现：overrideLock 默认值

**Local API 的默认行为** (`collections/operations/local/update.ts:86-90`)：

```typescript
/**
 * By default, document locks are ignored (`true`). 
 * Set to `false` to enforce locks and prevent operations 
 * when a document is locked by another user.
 * 
 * @default true  // ⚠️ 默认绕过锁检查！
 */
overrideLock?: boolean
```

**这是一个极其重要的设计决策**：
- **默认情况下，Local API 调用会绕过所有锁检查**
- 只有显式设置 `overrideLock: false` 才会强制执行锁校验

#### 3.2.2 各层 API 的 overrideLock 行为

| API 层 | overrideLock 默认值 | 行为说明 |
|--------|---------------------|---------|
| **Local API** | `true` | 默认绕过锁检查；需显式设为 `false` 才检查 |
| **REST API** | 从 query 参数解析 | `/api/posts/123?overrideLock=false` |
| **GraphQL API** | 从变量解析 | mutation { updatePost(overrideLock: false, ...) } |
| **Admin UI 保存/更新** | 未显式传递 → undefined | 需要追踪实际调用链 |

#### 3.2.3 会执行锁检查的操作

**服务端哪些操作调用了 `checkDocumentLockStatus`？**

通过搜索源码，以下操作会调用锁检查：

| 操作 | 文件位置 | 锁检查目的 |
|------|---------|-----------|
| **update (Collection)** | `collections/operations/utilities/update.ts:133` | 更新文档前检查 |
| **update (Global)** | `globals/operations/update.ts:185` | 更新 Global 前检查 |
| **deleteByID** | `collections/operations/deleteByID.ts:135` | 删除文档前检查 |
| **delete** | `collections/operations/delete.ts:155` | 批量删除前检查 (每条记录) |

**调用示例** (`update.ts:133-139`)：
```typescript
await checkDocumentLockStatus({
  id,
  collectionSlug: collectionConfig.slug,
  lockErrorMessage: `Document with ID ${id} is currently locked by another user and cannot be updated.`,
  overrideLock,  // 传入的参数
  req,
})
```

#### 3.2.4 实际场景分析

**场景1：Admin UI 中用户 A 编辑，用户 B 尝试保存**

```
Admin UI 保存请求流程：
1. 前端 Form 提交 PATCH 请求
2. REST API endpoint 接收
3. parseParams 解析 query 参数
   - overrideLock 从 URL query 读取
   - 如果 URL 中没有 ?overrideLock=xxx，则为 undefined
4. 调用 local update 操作
   - Local API 中 overrideLock 也是 undefined
5. 调用 checkDocumentLockStatus
   - overrideLock 默认 true (checkDocumentLockStatus.ts:24)
   - 所以会绕过锁检查？？？
```

**等等，这和预期不符。让我重新追踪 Admin UI 的实际调用...**

从前端 `Edit/index.tsx` 的 `onSave` 回调来看：
- 它调用的是 Form 的 `action` (REST API)
- 但 Form 的提交可能没有显式传递 `overrideLock`

**但从测试用例 `e2e.spec.ts` 来看**，锁确实在 Admin UI 中生效了。让我查看测试中的描述：

```typescript
// e2e.spec.ts 中的测试描述了：
// - 用户A编辑文档时会创建锁
// - 用户B尝试编辑时会看到 "Document Locked" 模态框
// - 用户B只能选择：Go Back / View Read-Only / Take Over
```

**实际机制**：

Admin UI 的锁保护是**双层的**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Admin UI 锁保护机制                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一层：前端保护 (强保护)                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 1. 用户打开编辑页时，通过 getFormState 获取 lockedState   │  │
│  │ 2. 如果被其他用户锁定：                                    │  │
│  │    - 显示 "Document Locked" 模态框                        │  │
│  │    - 用户选择 "View Read-Only" 时：                       │  │
│  │      → setIsReadOnlyForIncomingUser(true)                 │  │
│  │      → Form 被禁用 (disabled=true)                        │  │
│  │      → 所有输入字段 disabled                                │  │
│  │      → 保存按钮 disabled                                   │  │
│  │ 3. 用户根本无法提交请求！                                   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  第二层：服务端保护 (弱保护，需显式配置)                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 需要显式传递 overrideLock: false 才会检查                 │  │
│  │ Admin UI 的 REST 调用可能没有传递这个参数                  │  │
│  │ 所以主要依靠前端第一层保护                                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码** (`Edit/index.tsx:623-628`)：
```typescript
<Form
  // ...
  disabled={
    isReadOnlyForIncomingUser ||  // 被其他用户锁定时设为 true
    isInitializing || 
    !hasSavePermission || 
    isTrashed
  }
  // ...
>
```

#### 3.2.5 Take Over (抢占编辑权) 机制

当用户 B 看到 "Document Locked" 模态框时，有三个选项：

1. **Go Back**：返回列表页，不做任何操作
2. **View Read-Only**：以只读模式查看文档
3. **Take Over**：抢占编辑权

**Take Over 的实现** (`handleTakeOver.tsx`)：

```typescript
export const handleTakeOver = async ({
  id,
  collectionSlug,
  globalSlug,
  updateDocumentEditor,  // 关键函数：更新锁的拥有者
  user,                   // 当前用户
  // ...
}: HandleTakeOverParams): Promise<void> => {
  
  // 调用 updateDocumentEditor 将锁的拥有者改为当前用户
  await updateDocumentEditor(id, collectionSlug ?? globalSlug, user)
  
  // 更新前端状态
  documentLockStateRef.current = {
    hasShownLockedModal: true,
    isLocked: true,
    user,  // 现在是当前用户
  }
  setCurrentEditor(user)
  setIsReadOnlyForIncomingUser(false)  // 解除只读
}
```

**被抢占用户的体验**：
- 用户 A 继续编辑时，前端会检测到 `lockedState.user` 变化
- 显示 "Document Take Over" 模态框
- 用户 A 只能选择：Go Back / View Read-Only

**关键检测代码** (`Edit/index.tsx:212-247`)：
```typescript
const handleDocumentLocking = useCallback(
  (lockedState: LockedState) => {
    const previousOwnerID = ...
    
    if (lockedState && lockedState.user) {
      const lockedUserID = ...
      
      // 检测到编辑权被抢占
      if (previousOwnerID === user.id && lockedUserID !== user.id) {
        setShowTakeOverModal(true)
        documentLockState.current.hasShownLockedModal = true
      }
      // ...
    }
  },
  [...]
)
```

---

### 3.3 并发写入冲突处理

#### 3.3.1 乐观锁策略 (Autosave 场景)

在 `updateLatestVersion.ts` 中实现了针对 autosave 的并发冲突检测：

```typescript
// packages/payload/src/versions/updateLatestVersion.ts

try {
  // 尝试更新最新的 autosave 版本
  return await payload.db.updateVersion({
    collection,
    data: versionData,
    id: latestVersion.id,
    locale,
    req,
  })
} catch (err) {
  versionUpdateFailed = true
  payload.logger.warn({
    err,
    msg: `Failed to update latest version — checking if a concurrent write already succeeded.`
  })
}

// 冲突解决策略
if (versionUpdateFailed) {
  // 重新查询最新版本
  const freshVersions = await payload.db.findVersions({
    collection,
    limit: 1,
    locale,
    pagination: false,
    req,
    sort: '-updatedAt',
    where: versionWhere,
  })
  
  const [freshVersion] = freshVersions.docs
  
  // 如果最新版本的 updatedAt 比我们读取时新，
  // 说明并发请求已经成功更新了版本
  if (freshVersion && new Date(freshVersion.updatedAt) > new Date(latestVersion.updatedAt)) {
    return freshVersion  // 返回并发请求的结果，视为成功
  }
}
```

**设计意图**：
- Autosave 是高频操作，不应该让用户看到错误
- 如果并发请求已成功，直接返回那个结果即可
- 数据不会丢失，只是"后写入"的请求返回"先写入"的结果

#### 3.3.2 冲突处理策略汇总

| 场景 | 层级 | 处理方式 | 结果 |
|------|------|---------|------|
| A获取锁 → A写入 | 前端+服务端 | 正常流程 | A成功 |
| A获取锁 → B尝试编辑 | 前端 | 显示锁定模态框 | B无法编辑 |
| B选择 Take Over | 前端+服务端 | 更新锁拥有者 | B获得编辑权 |
| A继续编辑 | 前端 | 检测到锁变化 | 显示被抢占模态框 |
| 并发 autosave | 服务端 | 乐观锁检测 | 一方成功，另一方返回成功结果 |
| A锁过期 → B获取锁 | 前端+服务端 | 清除过期锁 | B获得编辑权 |

---

### 3.4 事务支持

**核心保证**：

1. **单文档操作**：每个文档更新使用独立事务
2. **批量操作**：可配置 `bulkOperationsSingleTransaction` 控制事务模式
3. **失败回滚**：操作失败时通过 `killTransaction` 回滚

```typescript
// update.ts:79-80
const shouldCommit = !args.disableTransaction && (await initTransaction(args.req))

// 成功提交
if (shouldCommit) {
  await commitTransaction(req)
}

// 失败回滚
catch (error) {
  await killTransaction(args.req)
  throw error
}
```

---

## 4. API 接口汇总

### 4.1 版本操作 API

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/{collection}/versions` | 查询版本列表（分页） |
| GET | `/api/{collection}/versions/:id` | 获取单个版本详情 |
| POST | `/api/{collection}/versions/:id` | 恢复到指定版本 |

### 4.2 核心参数

**写操作参数**：
- `draft: boolean` - 控制是否仅保存到版本表
- `autosave: boolean` - 标记为自动保存版本
- `_status: 'draft' | 'published'` - 控制文档状态
- `overrideLock: boolean` - 覆盖文档锁

**读操作参数**：
- `draft: boolean` - `true` 时从版本表返回最新草稿

### 4.3 Local API 示例

```typescript
// 创建草稿
await payload.create({
  collection: 'posts',
  data: { title: 'Draft Post' },
  draft: true,
})

// 保存草稿（不更新发布版本）
await payload.update({
  collection: 'posts',
  id: 'doc-id',
  data: { title: 'Updated Draft' },
  draft: true,
})

// 发布文档
await payload.update({
  collection: 'posts',
  id: 'doc-id',
  data: { title: 'Published', _status: 'published' },
  draft: false,
})

// 自动保存
await payload.update({
  collection: 'posts',
  id: 'doc-id',
  data: { title: 'Autosaved' },
  draft: true,
  autosave: true,
})

// 读取最新草稿
await payload.findByID({
  collection: 'posts',
  id: 'doc-id',
  draft: true,
})
```

---

## 5. 架构总结

### 5.1 核心模块关系

```
┌─────────────────────────────────────────────────────────────┐
│                        Admin UI                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Debounce    │  │ Status      │  │ Version History     │ │
│  │ Autosave    │  │ Indicator   │  │ Diff Viewer         │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
└─────────┼─────────────────┼────────────────────┼─────────────┘
          │                 │                    │
          ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Layer                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ draft       │  │ autosave    │  │ _status             │ │
│  │ overrideLock│  │ versions    │  │ publishAllLocales   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
└─────────┼─────────────────┼────────────────────┼─────────────┘
          │                 │                    │
          ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   Business Logic Layer                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ updateDocument  │  │ saveVersion     │  │ checkDocLock │ │
│  │ (draft判定)     │  │ (版本存储)      │  │ (冲突检测)   │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘ │
└───────────┼─────────────────────┼───────────────────┼─────────┘
            │                     │                   │
            ▼                     ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    Database Layer                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Main Collection │  │ _versions       │  │ locked-docs  │ │
│  │ (发布版本)      │  │ (版本历史)      │  │ (锁定记录)   │ │
│  │ _status         │  │ autosave        │  │ user         │ │
│  │                 │  │ parent          │  │ document     │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 关键设计决策

| 决策 | 设计选择 | 优势 |
|------|---------|------|
| 版本存储 | 独立 `_versions` 表 | 不修改主表结构，查询隔离 |
| Autosave | 单版本更新 | 避免版本爆炸 |
| 冲突处理 | 文档锁 + 乐观锁 | 平衡性能与一致性 |
| 状态管理 | `_status` 字段 + draft 参数 | 灵活的发布控制 |
| 本地化 | 独立 `_status` 值 | 多语言独立发布 |

---

## 6. 文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/payload/src/versions/saveVersion.ts` | 版本保存核心逻辑 |
| `packages/payload/src/versions/updateLatestVersion.ts` | 自动保存/取消发布的版本更新 |
| `packages/payload/src/versions/types.ts` | 版本配置类型定义 |
| `packages/payload/src/versions/baseFields.ts` | `_status` 等基础字段定义 |
| `packages/payload/src/collections/operations/utilities/update.ts` | 文档更新流程，draft 判定 |
| `packages/payload/src/utilities/checkDocumentLockStatus.ts` | 文档锁定检查 |
| `packages/payload/src/locked-documents/config.ts` | 锁定文档集合配置 |
| `docs/versions/overview.mdx` | 版本功能概述文档 |
| `docs/versions/drafts.mdx` | Drafts 功能详细文档 |
| `docs/versions/autosave.mdx` | Autosave 功能文档 |
