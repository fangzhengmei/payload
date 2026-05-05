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
│     │ → 调用 getFormState(returnLockStatus=true)             │    │
│     │ → 若当前没有活动锁，服务端创建锁记录                    │    │
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

### 3.2 集合文档 vs 全局配置：两条写入链路的锁行为差异

#### 3.2.1 关键发现：两条链路的默认行为完全不同

**深入代码追踪后，发现了一个重要的差异：**

| 链路类型 | API 入口文件 | overrideLock 处理方式 | 默认行为 |
|---------|-----------|----------------------|---------|
| **集合文档 (Collection)** | `collections/endpoints/updateByID.ts:33` | `overrideLock ?? false` | **默认拦住（执行锁检查）** |
| **全局配置 (Global)** | `globals/endpoints/update.ts` | **没有传递 overrideLock 参数** | **默认放行（绕过锁检查）** |

这是一个潜在的不一致性设计！

---

#### 3.2.2 集合文档写入链路详解

**REST API 入口** (`collections/endpoints/updateByID.ts`)：

```typescript
const { overrideLock, ... } = parseParams(req.query)

const doc = await updateByIDOperation({
  // ...
  overrideLock: overrideLock ?? false,  // ⚠️ 关键：默认是 false！
  // ...
})
```

**完整链路追踪**：

```
集合文档 REST API 更新请求链路：

1. 前端 PATCH 请求：/api/posts/123
   └─ URL Query 中没有 overrideLock 参数

2. REST Handler (updateByID.ts)
   └─ overrideLock = parseParams(req.query).overrideLock  → undefined
   └─ 传递给 updateByIDOperation：overrideLock ?? false  → false

3. updateByIDOperation → updateDocument
   └─ 传递 overrideLock: false

4. checkDocumentLockStatus (overrideLock = false)
   └─ 执行严格锁检查！
   └─ 如果被其他用户锁定且未过期 → 抛出 Locked 错误
```

**结论：集合文档的 REST API 默认执行锁检查！**

---

#### 3.2.3 全局配置写入链路详解

**REST API 入口** (`globals/endpoints/update.ts`)：

```typescript
// ⚠️ 注意：根本没有读取或传递 overrideLock 参数！
const result = await updateOperation({
  slug: globalConfig.slug,
  autosave,
  data: req.data!,
  // ... 其他参数
  // 没有 overrideLock！
})
```

**完整链路追踪**：

```
全局配置 REST API 更新请求链路：

1. 前端 POST 请求：/api/globals/menu
   └─ URL Query 中没有 overrideLock 参数

2. REST Handler (globals/endpoints/update.ts)
   └─ 没有读取 overrideLock
   └─ 调用 updateOperation 时没有传递 overrideLock

3. updateOperation (globals/operations/update.ts)
   └─ 解构参数：overrideLock,  // undefined
   └─ 调用 checkDocumentLockStatus({ overrideLock, ... })

4. checkDocumentLockStatus (overrideLock = undefined)
   └─ 函数默认值：overrideLock = true  // checkDocumentLockStatus.ts:24
   └─ 跳过锁检查！直接放行
```

**结论：全局配置的 REST API 默认绕过锁检查！**

---

#### 3.2.4 Local API 的行为

**Local API 中没有设置默认值**，直接传递：

```typescript
// collections/operations/local/update.ts
const args = {
  // ...
  overrideLock,  // 直接传递用户传入的值，没有 ?? false
  // ...
}
```

所以 Local API 的行为取决于 `checkDocumentLockStatus` 的函数默认值：

```typescript
// checkDocumentLockStatus.ts:24
overrideLock = true,  // 函数参数默认值
```

**Local API 行为总结**：

| 调用方式 | overrideLock 值 | 行为 |
|---------|----------------|------|
| 不显式传递 | `undefined` → 使用函数默认 `true` | 绕过锁检查 |
| 显式传递 `overrideLock: false` | `false` | 执行锁检查 |
| 显式传递 `overrideLock: true` | `true` | 绕过锁检查 |

---

#### 3.2.5 各操作的锁检查行为汇总

**服务端调用 `checkDocumentLockStatus` 的操作：**

| 操作 | 集合/全局 | REST API 默认行为 | Local API 默认行为 |
|------|-----------|------------------|-------------------|
| **updateByID** | 集合 | **拦住** (执行锁检查) | 放行 (绕过检查) |
| **update** | 全局 | **放行** (绕过检查) | 放行 (绕过检查) |
| **deleteByID** | 集合 | 需查看 endpoint | 需查看 endpoint |
| **delete** | 集合 | 需查看 endpoint | 需查看 endpoint |

**Admin UI 的实际保护机制：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Admin UI 锁保护的真实机制                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  集合文档：双层保护                                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 第一层：前端保护（强）                                    │  │
│  │   - 用户B打开编辑页时，getFormState 返回 lockedState      │  │
│  │   - 显示 "Document Locked" 模态框                        │  │
│  │   - 选择 "View Read-Only" → Form disabled               │  │
│  │   - 用户根本无法提交请求！                                 │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ 第二层：服务端保护（强）                                  │  │
│  │   - 集合 REST API 默认 overrideLock: false               │  │
│  │   - 即使前端被绕过（如直接调用 API），服务端也会拦住       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  全局配置：只有前端保护                                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 第一层：前端保护（强）                                    │  │
│  │   - 同集合文档，Form disabled                           │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ 第二层：服务端保护（无！）                                │  │
│  │   - 全局 REST API 没有传递 overrideLock                   │  │
│  │   - 默认绕过锁检查！                                      │  │
│  │   - ⚠️ 如果直接调用 API，可以绕过前端保护直接写入！            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 3.2.6 关键代码对照

**集合 REST API（拦住）**：
```typescript
// collections/endpoints/updateByID.ts:33
overrideLock: overrideLock ?? false,  // ⚠️ 默认 false，执行锁检查
```

**全局 REST API（放行）**：
```typescript
// globals/endpoints/update.ts
// 没有 overrideLock 参数！直接调用 updateOperation 时没有传递
```

**checkDocumentLockStatus 默认值**：
```typescript
// checkDocumentLockStatus.ts:24
overrideLock = true,  // ⚠️ 函数参数默认 true，绕过锁检查
```

---

#### 3.2.7 锁检查后清理哪些锁记录？

这是一个非常关键但容易被忽视的细节。让我们深入分析 `checkDocumentLockStatus` 的执行流程。

**完整的函数结构** (`checkDocumentLockStatus.ts`)：

```typescript
// 第 18-26 行：函数参数
export const checkDocumentLockStatus = async ({
  id,
  collectionSlug,
  globalSlug,
  lockDurationDefault = 300,
  lockErrorMessage,
  overrideLock = true,  // ⚠️ 默认绕过锁检查
  req,
}: CheckDocumentLockStatusArgs): Promise<void> => {
```

**执行顺序分析**：

```
┌─────────────────────────────────────────────────────────────────────┐
│            checkDocumentLockStatus 完整执行流程                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1：前置检查                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 30-33 行：检查 locked-documents 集合是否存在                  │  │
│  │ 36-40 行：获取 lockDocuments 配置，判断是否启用锁定            │  │
│  │ 42-55 行：构建 lockedDocumentQuery                            │  │
│  │ 57-59 行：如果 !isLockingEnabled → 直接返回（不删除任何锁）   │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                        │
│  阶段 2：锁检查（仅当 overrideLock = false 时执行）                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 62-97 行：if (!overrideLock) { ... }                         │  │
│  │                                                              │  │
│  │ 69-75 行：查询锁记录                                          │  │
│  │ 80-95 行：检查锁定条件                                        │  │
│  │                                                              │  │
│  │ 关键判断（90-95 行）：                                        │  │
│  │ if (                                                         │  │
│  │   lockedDoc.user?.value !== currentUserId &&  // 不是当前用户 │  │
│  │   now - lastEditedAt <= lockDurationInMilliseconds  // 锁未过期│  │
│  │ ) {                                                          │  │
│  │   throw new Locked(finalLockErrorMessage)  // ⚠️ 抛出错误！   │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ ⚠️ 如果抛出 Locked 错误：                                      │  │
│  │    → 函数在此处终止                                           │  │
│  │    → 不会执行后面的删除操作                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                        │
│  阶段 3：删除锁记录（无论 overrideLock 是什么，只要没抛出错误就执行） │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 99-105 行：                                                  │  │
│  │ await payload.db.deleteMany({                                │  │
│  │   collection: lockedDocumentsCollectionSlug,                │  │
│  │   req: payload.db.name === 'mongoose' ? undefined : req,   │  │
│  │   where: lockedDocumentQuery,  // ⚠️ 关键：只删除当前文档的锁 │  │
│  │ })                                                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**lockedDocumentQuery 的定义**（第 42-55 行）：

```typescript
let lockedDocumentQuery = {}

if (collectionSlug) {
  // 集合文档：精确匹配
  lockedDocumentQuery = {
    and: [
      { 'document.relationTo': { equals: collectionSlug } },
      { 'document.value': { equals: id } },
    ],
  }
} else if (globalSlug) {
  // 全局配置：精确匹配
  lockedDocumentQuery = { globalSlug: { equals: globalSlug } }
}
```

---

#### 3.2.8 清理行为汇总

| 场景 | 阶段 1 锁定启用 | 阶段 2 锁检查 | 阶段 3 删除锁记录 | 删除哪些记录 |
|------|----------------|--------------|------------------|-------------|
| **正常写入（当前用户是锁定者）** | 是 | 通过 | ✅ 执行 | 当前文档的锁 |
| **正常写入（无锁记录）** | 是 | 通过 | ✅ 执行 | 当前文档的锁（无记录，空删除） |
| **overrideLock=true（绕过检查）** | 是 | 跳过 | ✅ 执行 | 当前文档的锁 |
| **被其他用户锁定且锁未过期** | 是 | ❌ 抛出 Locked 错误 | ❌ 不执行 | 无（函数提前终止） |
| **被其他用户锁定但锁已过期** | 是 | 通过（过期视为无锁） | ✅ 执行 | 当前文档的锁（此时应该是空的？） |
| **集合 lockDocuments: false** | 否 | 跳过 | ❌ 不执行 | 无（函数提前返回） |

---

#### 3.2.9 关键发现总结

**发现 1：成功写入时会自动解锁**

每次成功的写入操作（更新、删除）都会调用 `checkDocumentLockStatus`，而该函数在通过检查后会**删除当前文档的锁记录**。

这意味着：
- 用户 A 编辑文档 → 创建锁
- 用户 A 保存文档 → 锁被删除
- 用户 B 此时可以正常编辑

**发现 2：锁检查失败时不会删除锁**

如果用户 B 尝试写入被用户 A 锁定的文档：
1. `overrideLock = false`（集合 REST API 默认）
2. 检查发现：`lockedDoc.user?.value !== currentUserId` 且锁未过期
3. **抛出 `Locked` 错误**
4. 函数在此处终止，**不会执行删除操作**
5. 用户 A 的锁保持不变

**发现 3：只删除当前操作文档的锁**

`lockedDocumentQuery` 是精确匹配：
- 集合文档：`document.relationTo = collectionSlug` AND `document.value = id`
- 全局配置：`globalSlug = globalSlug`

**不会删除其他文档的锁记录**。

**发现 4：overrideLock=true 仍然会删除锁**

即使设置 `overrideLock: true` 绕过了锁检查，只要锁定功能启用且没有抛出错误，仍然会执行删除当前文档锁记录的操作。

这意味着：
- 如果你通过 API 调用 `overrideLock: true` 强行写入
- 写入成功后，原来的锁会被删除
- 相当于你强行接管了编辑权并解锁

---

#### 3.2.10 实际场景示例

**场景 1：正常编辑流程**

```
时间线：
T1: 用户A打开 posts/123 编辑页
    → 首次字段变更触发 getFormState(returnLockStatus=true)
    → 若当前没有活动锁，则创建锁记录：{ document: { relationTo: 'posts', value: 123 }, user: A }

T2: 用户A持续编辑（每10秒续期一次）
    → 锁记录的 updatedAt 不断更新

T3: 用户A点击保存
    → REST PATCH /api/posts/123
    → 集合 REST API → overrideLock ?? false → false
    → checkDocumentLockStatus(overrideLock=false)
      ├─ 查询锁记录 → 找到，锁定者是用户A
      ├─ 检查：lockedDoc.user?.value === currentUserId → 通过
      └─ 删除当前文档的锁记录 ✅
    → 保存成功
    → 锁已被清除

T4: 用户B打开 posts/123 编辑页
    → 没有锁记录
    → 可以正常编辑
```

**场景 2：多人冲突（集合文档，服务端拦住）**

```
时间线：
T1: 用户A打开 posts/123 编辑页
    → 创建锁记录（user: A）

T2: 用户B尝试打开 posts/123 编辑页
    → getFormState 返回 lockedState={ user: A }
    → 前端显示 "Document Locked" 模态框
    → 用户B选择 "View Read-Only"
    → Form disabled，无法提交

T3: 用户B绕过前端，直接调用 REST API
    → PATCH /api/posts/123
    → 集合 REST API → overrideLock = false
    → checkDocumentLockStatus(overrideLock=false)
      ├─ 查询锁记录 → 找到，锁定者是用户A
      ├─ 检查：A.id !== B.id 且锁未过期
      └─ 抛出 Locked 错误 ❌
      └─ ⚠️ 函数在此终止，不会删除锁记录
    → API 返回 423 Locked
    → 用户A的锁保持不变
```

**场景 3：多人冲突（全局配置，服务端放行）**

```
时间线：
T1: 用户A打开全局配置 "menu" 编辑页
    → 创建锁记录（user: A）

T2: 用户B绕过前端，直接调用 REST API
    → POST /api/globals/menu
    → 全局 REST API → 没有传递 overrideLock
    → updateOperation 中 overrideLock = undefined
    → checkDocumentLockStatus(overrideLock=undefined)
      ├─ 函数默认值：overrideLock = true
      ├─ 跳过锁检查（第 62-97 行不执行）
      └─ 删除当前文档的锁记录 ✅
    → 保存成功！⚠️
    → 用户A的锁被删除了
```

**场景 4：使用 overrideLock=true 强行接管**

```
时间线：
T1: 用户A打开 posts/123 编辑页
    → 创建锁记录（user: A）

T2: 用户B调用 API 并设置 overrideLock=true
    → PATCH /api/posts/123?overrideLock=true
    → 集合 REST API → overrideLock = true
    → checkDocumentLockStatus(overrideLock=true)
      ├─ 跳过锁检查
      └─ 删除当前文档的锁记录 ✅
    → 保存成功
    → 用户A的锁被删除
    → 用户A后续编辑时会检测到锁变化，显示 "Take Over" 模态框
```

---

#### 3.2.11 Take Over (抢占编辑权) 机制

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
