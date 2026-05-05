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
2. 超过限制时删除最旧的非 autosave 版本
3. 设为 0 表示不限制

### 2.7 本地化状态 (Localize Status)

当 `localizeStatus: true` 时：
- `_status` 字段支持多语言独立控制
- 可单独发布/取消发布某个语言版本
- 需要 `publishAllLocales` 参数控制全语言发布

---

## 3. 多人编辑冲突处理机制

### 3.1 文档锁定 (Document Locking)

#### 3.1.1 核心配置

**锁定集合** (`packages/payload/src/locked-documents/config.ts`)：

```typescript
// 自动创建的锁定文档集合
{
  slug: 'payload-locked-documents',
  fields: [
    { name: 'document', type: 'relationship', relationTo: 可锁定集合 },
    { name: 'globalSlug', type: 'text' },  // 用于 Global 锁定
    { name: 'user', type: 'relationship', relationTo: 认证集合, required: true }
  ]
}
```

#### 3.1.2 锁定检查流程

**服务端实际分成两段：**
- **建锁 / 续锁**：编辑页读表单状态时通过 `handleFormStateLocking` 创建或续期锁记录
- **写入前校验**：真正更新文档时通过 `checkDocumentLockStatus` 拦截其他用户的写入

**写入前校验函数** (`packages/payload/src/utilities/checkDocumentLockStatus.ts`)

```
写入前锁检查流程：
1. 检查锁定功能是否启用 (lockDocuments !== false)
2. 查询 'payload-locked-documents' 集合
3. 检查是否存在锁定记录：
   a. 锁定者是当前用户 → 允许写入
   b. 锁定者是其他用户且锁未过期 → 抛出 Locked 错误
   c. 锁已过期 → 允许写入，并清除旧锁
4. 当前函数只负责校验和清理，不在这里新建锁记录
```

#### 3.1.3 锁参数配置

```typescript
// 锁定持续时间（秒）
lockDurationDefault = 300  // 默认 5 分钟

// 自定义配置
lockDocuments: {
  duration: 600  // 10 分钟
}
```

### 3.2 并发写入冲突处理

#### 3.2.1 乐观锁策略

在 `updateLatestVersion.ts` 中实现了并发冲突检测：

```typescript
try {
  // 尝试更新最新版本
  return await payload.db.updateVersion({...})
} catch (err) {
  versionUpdateFailed = true
  payload.logger.warn({
    err,
    msg: `Failed to update latest version — checking if a concurrent write already succeeded.`
  })
}

// 冲突解决：检查是否已有并发写入成功
if (versionUpdateFailed) {
  // 重新查询最新版本
  const [freshVersion] = freshDocs
  
  // 如果 updatedAt 比我们读取时新，说明并发请求已成功
  if (freshVersion && new Date(freshVersion.updatedAt) > new Date(latestVersion.updatedAt)) {
    return freshVersion  // 返回并发请求的结果
  }
}
```

#### 3.2.2 冲突处理策略

| 场景 | 处理方式 | 结果 |
|------|---------|------|
| A获取锁 → A写入 | 正常流程 | A成功 |
| A获取锁 → B尝试获取 | 检查锁，B被拒绝 | B收到 Locked 错误 |
| A锁过期 → B获取锁 | 清除过期锁，B获取 | B成功 |
| 并发 autosave 更新 | 乐观锁检测 | 一方成功，另一方返回成功结果 |

### 3.3 覆盖锁选项 (Override Lock)

**参数**：`overrideLock: boolean`

```typescript
// updateDocument.ts:133-139
await checkDocumentLockStatus({
  id,
  collectionSlug: collectionConfig.slug,
  lockErrorMessage: `Document with ID ${id} is currently locked by another user and cannot be updated.`,
  overrideLock,
  req,
})
```

**使用场景**：
- `overrideLock: true`：跳过锁检查（Admin UI 内部操作、API 手动调用）
- `overrideLock: false`：严格检查锁（用户编辑操作）

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
