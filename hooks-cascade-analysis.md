# PayloadCMS 钩子级联触发与递归保护机制分析报告

## 一、钩子注册的三层架构

PayloadCMS 的钩子系统支持在三个层级注册，形成了灵活的扩展点：

### 1. 全局级别 (Global Config)
- **注册位置**: `payload.config.ts` 的 `hooks` 属性
- **支持的钩子类型**: 仅 `afterError`
- **适用范围**: 整个 Payload 实例的全局错误处理

### 2. 集合级别 (Collection Config)
- **注册位置**: 每个 Collection 配置的 `hooks` 属性
- **支持的钩子类型**:
  - `beforeOperation` / `afterOperation` - 操作前后
  - `beforeValidate` / `beforeChange` / `afterChange` - 写入流程
  - `beforeRead` / `afterRead` - 读取流程
  - `beforeDelete` / `afterDelete` - 删除流程
  - `beforeLogin` / `afterLogin` / `afterLogout` - 认证流程
  - `me` / `refresh` - 特定操作

### 3. 字段级别 (Field Config)
- **注册位置**: 每个 Field 配置的 `hooks` 属性
- **支持的钩子类型**:
  - `beforeValidate`
  - `beforeChange`
  - `afterChange`
  - `afterRead`

---

## 二、写入前钩子 (beforeChange) 级联机制

### 2.1 执行顺序

以 `create` 操作为例 (`packages/payload/src/collections/operations/create.ts`):

```
1. beforeOperation (Collection级别)
        ↓
2. beforeValidate (Field级别 - traverseFields遍历)
        ↓
3. beforeValidate (Collection级别)
        ↓
4. beforeChange (Collection级别)
        ↓
5. beforeChange (Field级别 - traverseFields遍历)
        ↓
6. 数据库写入
```

### 2.2 字段级别的 beforeChange 执行

核心实现位于 `packages/payload/src/fields/hooks/beforeChange/promise.ts`:

```typescript
// 第135-162行 - 字段级 beforeChange 钩子执行
if ('hooks' in field && field.hooks?.beforeChange) {
  for (const hook of field.hooks.beforeChange) {
    const hookedValue = await hook({
      blockData,
      collection,
      context,
      data,
      field,
      global,
      indexPath: indexPathSegments,
      operation,
      originalDoc: doc,
      path: pathSegments,
      previousSiblingDoc: siblingDoc,
      previousValue: siblingDoc[field.name],
      req,
      schemaPath: schemaPathSegments,
      siblingData,
      siblingDocWithLocales,
      siblingFields: siblingFields!,
      value: siblingData[field.name],
    })

    if (hookedValue !== undefined) {
      siblingData[field.name] = hookedValue
    }
  }
}
```

### 2.3 跨集合关联字段的 beforeChange 级联

**重要发现**: 关系字段 (`relationship` / `join` / `upload`) 的 `beforeChange` 钩子**不会自动级联触发**关联集合的钩子。

**机制分析**:
- `beforeChange` 钩子仅在当前操作的文档字段树中递归遍历 (`traverseFields`)
- 关系字段存储的只是关联文档的 ID，不会自动加载关联文档执行其钩子
- **唯一的级联触发方式**: 在字段/集合级别的钩子中**显式调用** `payload.create` / `payload.update` / `payload.delete` 等 API

**代码证据** (`packages/payload/src/fields/hooks/beforeChange/promise.ts`):
- 第 293-447 行的 `switch (field.type)` 中，`relationship` / `upload` / `join` 类型没有特殊的级联处理逻辑
- 只有 `array`, `blocks`, `group`, `tabs` 等复合字段会递归调用 `traverseFields` 处理内部字段

---

## 三、读取后钩子 (afterRead) 级联机制

### 3.1 执行顺序

以 `findByID` 操作为例 (`packages/payload/src/collections/operations/findByID.ts`):

```
1. beforeOperation (Collection级别)
        ↓
2. 数据库查询
        ↓
3. beforeRead (Collection级别)
        ↓
4. afterRead (Field级别 - 含关联填充级联)
        ↓
5. afterRead (Collection级别)
        ↓
6. afterOperation (Collection级别)
```

### 3.2 字段级别的 afterRead 与关联填充

核心实现位于 `packages/payload/src/fields/hooks/afterRead/promise.ts`:

**关键点**: `relationship` / `upload` / `join` 字段会在 `afterRead` 阶段触发**关联文档的填充**，这是**自动级联**的关键。

```typescript
// 第409-426行 - 关系字段触发填充
if (field.type === 'relationship' || field.type === 'upload' || field.type === 'join') {
  populationPromises.push(
    relationshipPopulationPromise({
      currentDepth,
      depth,
      draft,
      fallbackLocale,
      field,
      locale,
      overrideAccess,
      parentIsLocalized: parentIsLocalized!,
      populate,
      req,
      showHiddenFields,
      siblingDoc,
    }),
  )
}
```

### 3.3 关联填充的级联流程

`relationshipPopulationPromise` (`packages/payload/src/fields/hooks/afterRead/relationshipPopulationPromise.ts`) 的执行流程:

```
┌─────────────────────────────────────────────────────────────┐
│  relationshipPopulationPromise                               │
├─────────────────────────────────────────────────────────────┤
│  1. 检查填充条件: shouldPopulate = depth && currentDepth <= depth │
│  2. 使用 DataLoader 加载关联文档                              │
│  3. DataLoader 内部会调用 payload.find()                     │
│  4. payload.find() 会触发完整的钩子流程:                      │
│     - beforeRead (Collection)                                │
│     - afterRead (Field) → 可能触发更多级联                    │
│     - afterRead (Collection)                                 │
└─────────────────────────────────────────────────────────────┘
```

**填充深度检查** (`relationshipPopulationPromise.ts:65`):
```typescript
const shouldPopulate = depth && currentDepth <= depth
```

**级联调用 DataLoader** (`relationshipPopulationPromise.ts:77-94`):
```typescript
if (shouldPopulate) {
  relationshipValue = await req.payloadDataLoader.load(
    createDataloaderCacheKey({
      collectionSlug: relatedCollection.config.slug,
      currentDepth: currentDepth + 1,  // 深度+1
      depth,
      docID: id as string,
      // ...其他参数
    }),
  )
}
```

---

## 四、递归保护机制

PayloadCMS 实现了**多层递归保护**来防止循环引用导致的无限递归:

### 4.1 深度限制 (Depth Control)

**配置层面**:
- `defaultDepth`: 默认填充深度 (通常为 2)
- `maxDepth`: 最大允许深度 (防止恶意请求)
- 字段级别的 `maxDepth`: 单个关系字段的深度限制

**实现位置**: `packages/payload/src/fields/hooks/afterRead/index.ts:67-75`

```typescript
let depth =
  incomingDepth || incomingDepth === 0
    ? parseInt(String(incomingDepth), 10)
    : req.payload.config.defaultDepth
if (depth > req.payload.config.maxDepth) {
  depth = req.payload.config.maxDepth  // 强制限制不超过 maxDepth
}

const currentDepth = incomingCurrentDepth || 1
```

**字段级别的深度覆盖** (`relationshipPopulationPromise.ts:169`):
```typescript
const populateDepth = fieldHasMaxDepth(field) && field.maxDepth! < depth 
  ? field.maxDepth 
  : depth
```

### 4.2 DataLoader 缓存机制

DataLoader 不仅用于解决 N+1 问题，同时也是**递归保护**的关键:

**缓存键生成** (`packages/payload/src/collections/dataloader.ts:240-267`):

```typescript
export const createDataloaderCacheKey = ({
  collectionSlug,
  currentDepth,
  depth,
  docID,
  draft,
  fallbackLocale,
  locale,
  overrideAccess,
  populate,
  select,
  showHiddenFields,
  transactionID,
}: CreateCacheKeyArgs): string =>
  JSON.stringify([
    transactionID,
    collectionSlug,
    docID,
    depth,
    currentDepth,  // 关键: 深度不同视为不同请求
    locale,
    fallbackLocale,
    overrideAccess,
    showHiddenFields,
    draft,
    select,
    populate,
  ])
```

**缓存机制的保护效果**:
- 相同 `(collectionSlug, docID, currentDepth)` 的请求会被缓存
- **注意**: 循环引用如 `A→B→A` 在不同深度下会继续执行，直到 `currentDepth > depth`

### 4.3 无限循环检测

**Promise 迭代保护** (`packages/payload/src/fields/hooks/afterRead/index.ts:112-126`):

```typescript
let iterations = 0
while (fieldPromises.length > 0 || populationPromises.length > 0) {
  const currentFieldPromises = fieldPromises.splice(0, fieldPromises.length)
  const currentPopulationPromises = populationPromises.splice(0, populationPromises.length)

  await Promise.all(currentFieldPromises)
  await Promise.all(currentPopulationPromises)

  iterations++
  if (iterations >= 100) {
    throw new Error(
      'Infinite afterRead promise loop detected. A hook is likely adding field promises in an infinitely recursive way.',
    )
  }
}
```

**保护原理**:
- 钩子可能在执行时动态添加新的 `fieldPromises`
- 这会导致 `while` 循环持续执行
- 当迭代次数 >= 100 时，主动抛出错误终止

---

## 五、Global 级别的钩子机制

Global 的钩子机制与 Collection 类似，同样支持三层结构:

**读取流程** (`packages/payload/src/globals/operations/findOne.ts`):

```
1. beforeOperation (Global级别)
        ↓
2. 数据库查询
        ↓
3. beforeRead (Global级别)
        ↓
4. afterRead (Field级别 - 与Collection使用相同的traverseFields)
        ↓
5. afterRead (Global级别)
```

**关键**: Global 的字段级 `afterRead` 与 Collection 使用**完全相同**的 `traverseFields` 函数，因此**递归保护机制完全一致**。

---

## 六、关键代码位置汇总

| 功能 | 文件路径 |
|------|---------|
| Collection create 操作钩子流程 | `packages/payload/src/collections/operations/create.ts` |
| Collection findByID 操作钩子流程 | `packages/payload/src/collections/operations/findByID.ts` |
| Global findOne 操作钩子流程 | `packages/payload/src/globals/operations/findOne.ts` |
| beforeChange 字段钩子执行 | `packages/payload/src/fields/hooks/beforeChange/promise.ts` |
| afterRead 字段钩子执行 | `packages/payload/src/fields/hooks/afterRead/promise.ts` |
| 关系字段填充逻辑 | `packages/payload/src/fields/hooks/afterRead/relationshipPopulationPromise.ts` |
| DataLoader 缓存与批量加载 | `packages/payload/src/collections/dataloader.ts` |
| afterRead Promise 循环保护 | `packages/payload/src/fields/hooks/afterRead/index.ts:112-126` |

---

## 七、总结

### 7.1 beforeChange 级联特性

| 特性 | 说明 |
|------|------|
| 自动级联 | **不支持** - 仅遍历当前文档的字段树 |
| 关系字段 | 仅处理 ID 值，不加载关联文档 |
| 触发方式 | 必须在钩子中**显式调用** Payload API 才能触发其他集合的钩子 |
| 递归保护 | 无内置保护 - 需开发者自行避免循环调用 |

### 7.2 afterRead 级联特性

| 特性 | 说明 |
|------|------|
| 自动级联 | **支持** - 关系字段自动触发关联文档的完整读取流程 |
| 深度限制 | 通过 `depth` / `currentDepth` / `maxDepth` 三级控制 |
| 缓存机制 | DataLoader 缓存相同 `(collection, docID, currentDepth)` 请求 |
| 循环保护 | Promise 迭代次数限制 (100次) |

### 7.3 最佳实践建议

1. **beforeChange 中修改关联数据**:
   - 总是检查是否已在处理同一文档 (通过 `context` 传递状态)
   - 使用 `context` 防止无限循环:
   ```typescript
   beforeChange: [{
     handler: ({ context, data, req }) => {
       if (context.processingRelated) return data
       // 处理关联数据时标记 context
       await payload.update({
         collection: 'other',
         id: data.relationId,
         data: { ... },
         context: { processingRelated: true }
       })
       return data
     }
   }]
   ```

2. **afterRead 深度控制**:
   - 根据实际需求设置合理的 `depth` (避免使用过大的值)
   - 对可能产生循环引用的字段，考虑设置字段级 `maxDepth`
   - 使用 `populate` 参数精确控制需要填充的路径

3. **测试循环引用场景**:
   - 单元测试中应验证循环引用场景的行为
   - 确保 `maxDepth` 配置能正确终止递归
