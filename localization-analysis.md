# PayloadCMS 多语言字段实现分析报告

## 目录

1. [概述](#1-概述)
2. [管理后台多语言字段展示逻辑](#2-管理后台多语言字段展示逻辑)
3. [API 查询参数中的 locale 处理](#3-api-查询参数中的-locale-处理)
4. [不同数据库适配器的多语言存储机制](#4-不同数据库适配器的多语言存储机制)
5. [Fallback 语言链工作原理](#5-fallback-语言链工作原理)
6. [数据结构示例](#6-数据结构示例)
7. [关键代码参考](#7-关键代码参考)

---

## 1. 概述

PayloadCMS 的多语言（Localization）系统是一个多层协作的架构，涉及：

- **配置层**：定义支持的语言、默认语言、fallback 策略
- **管理后台层**：语言切换 UI、多语言字段编辑
- **API 层**：查询参数解析、语言上下文传递
- **数据层**：不同数据库适配器的存储策略
- **逻辑层**：fallback 链解析、数据合并

### 核心配置类型

```typescript
// packages/payload/src/config/types.ts:513-582

type BaseLocalizationConfig = {
  defaultLocale: string           // 默认语言代码
  fallback?: boolean               // 是否启用全局 fallback
  defaultLocalePublishOption?: 'active' | 'all'  // 发布按钮行为
  filterAvailableLocales?: (args: {
    locales: Locale[]
    req: PayloadRequest
  }) => Locale[] | Promise<Locale[]>  // 动态过滤可用语言
}

type Locale = {
  code: string                     // 语言代码，如 'en', 'es', 'pt'
  label: Record<string, string> | string  // 显示标签
  rtl?: boolean                    // 是否为 RTL（从右到左）语言
  fallbackLocale?: string | string[]  // 该语言的 fallback 语言
}
```

### 配置示例

```typescript
// test/localization/config.ts:478-535

localization: {
  defaultLocale: 'en',
  fallback: true,
  locales: [
    {
      code: 'en',
      label: { en: 'English', es: 'Inglés', de: 'Englisch' },
      rtl: false,
    },
    {
      code: 'es',
      label: { en: 'Spanish', es: 'Español', de: 'Spanisch' },
      rtl: false,
    },
    {
      code: 'pt',
      fallbackLocale: 'es',  // 葡萄牙语 fallback 到西班牙语
      label: { en: 'Portuguese', es: 'Portugués', de: 'Portugiesisch' },
    },
    {
      code: 'ar',
      label: { en: 'Arabic', es: 'Árabe', de: 'Arabisch' },
      rtl: true,  // 阿拉伯语是 RTL 语言
    },
  ],
}
```

---

## 2. 管理后台多语言字段展示逻辑

### 2.1 LocaleProvider 组件

管理后台的语言上下文由 `LocaleProvider` 组件统一管理：

**位置**: `packages/ui/src/providers/Locale/index.tsx`

**核心职责**:
1. 从多个来源解析当前语言
2. 监听 URL 参数变化
3. 提供 `useLocale` hook 供组件使用

### 2.2 语言解析优先级

```
URL 查询参数 (?locale=) > 用户偏好设置 > 默认语言 > 第一个配置的语言
```

**关键代码逻辑**:

```typescript
// packages/ui/src/providers/Locale/index.tsx:54-66

const [locale, setLocale] = React.useState<Locale>(() => {
  if (!localization || (localization && !localization.locales.length)) {
    return {} as Locale
  }

  return (
    findLocaleFromCode(localization, localeFromParams) ||  // 1. URL 参数
    findLocaleFromCode(localization, initialLocaleFromPrefs) ||  // 2. 用户偏好
    findLocaleFromCode(localization, defaultLocale) ||  // 3. 默认语言
    findLocaleFromCode(localization, localization.locales[0].code)  // 4. 第一个语言
  )
})
```

### 2.3 语言切换机制

当用户在管理后台切换语言时：

1. **URL 参数更新**: `?locale=es` 被添加到 URL
2. **状态更新**: `LocaleProvider` 检测到 `localeFromParams` 变化
3. **重新渲染**: 所有使用 `useLocale()` 的组件重新获取数据
4. **数据重新加载**: API 请求携带新的 `locale` 参数

### 2.4 字段本地化标记

字段通过 `localized: true` 配置启用多语言：

```typescript
{
  name: 'title',
  type: 'text',
  localized: true,  // 启用多语言
}
```

**支持本地化的字段类型**:
- 基础类型: `text`, `textarea`, `number`, `checkbox`, `date`, `select`, `email`, `password`
- 复合类型: `group`, `array`, `blocks`, `tabs` (命名 tab)
- 关系类型: `relationship`

### 2.5 本地化字段的显示值获取

**位置**: `packages/ui/src/utilities/getDisplayedFieldValue.ts`

该工具函数处理管理后台列表视图和详情视图中本地化字段的显示值。

---

## 3. API 查询参数中的 locale 处理

### 3.1 请求创建时的参数解析

**位置**: `packages/payload/src/utilities/createPayloadRequest.ts`

当 HTTP 请求到达时，`createPayloadRequest` 从多个来源解析语言参数：

```typescript
// 从 URL searchParams 获取
let locale = searchParams.get('locale')

// 从 query 字符串解析 fallbackLocale
const fallbackFromRequest = (query.fallbackLocale ||
  searchParams.get('fallback-locale') ||
  searchParams.get('fallbackLocale')) as TypedFallbackLocale
```

### 3.2 支持的参数格式

| 参数 | 格式 | 说明 |
|------|------|------|
| `locale` | `string` | 指定查询/操作的语言 |
| `locale` | `'all'` \| `'*'` | 获取所有语言版本 |
| `fallbackLocale` \| `fallback-locale` | `string` \| `string[]` | 指定 fallback 语言 |
| `fallbackLocale` | `'false'` \| `'none'` \| `'null'` | 禁用 fallback |

### 3.3 参数 sanitize 流程

**位置**: `packages/payload/src/utilities/addLocalesToRequest.ts`

```typescript
export const sanitizeLocales = ({
  fallbackLocale,
  locale,
  localization,
}: SanitizeLocalesArgs): SanitizeLocalesReturn => {
  if (localization) {
    fallbackLocale = sanitizeFallbackLocale({
      fallbackLocale,
      locale,
      localization,
    })!
  }

  if (['*', 'all'].includes(locale)) {
    locale = 'all'  // 标准化 'all' 值
  } else if (localization && !localization.localeCodes.includes(locale) && localization.fallback) {
    locale = localization.defaultLocale  // 无效语言回退到默认
  }

  return { fallbackLocale, locale }
}
```

### 3.4 REST API 示例

```http
# 查询英语版本
GET /api/posts?locale=en

# 查询西班牙语版本，禁用 fallback
GET /api/posts?locale=es&fallbackLocale=false

# 查询所有语言版本
GET /api/posts?locale=all

# 使用自定义 fallback 链
GET /api/posts?locale=pt&fallbackLocale=es&fallbackLocale=en
```

### 3.5 Local API 示例

```typescript
// packages/payload/src/utilities/createLocalReq.ts:88-96

type CreateLocalReqOptions = {
  locale?: string
  fallbackLocale?: false | TypedLocale
  // ... 其他选项
}

// 使用示例
await payload.find({
  collection: 'posts',
  locale: 'pt',
  fallbackLocale: ['es', 'en'],  // 自定义 fallback 链
})
```

---

## 4. 不同数据库适配器的多语言存储机制

### 4.1 Drizzle 适配器（PostgreSQL / SQLite）

**核心设计原则**: 本地化字段存储在独立的 `_locales` 表中。

#### 4.1.1 表结构设计

| 表名 | 用途 |
|------|------|
| `{collection}` | 主表，存储非本地化字段 |
| `{collection}_locales` | 本地化字段表，每个语言一行 |
| `{collection}_{array_field}` | Array 字段子表 |
| `{collection}_{array_field}_locales` | 本地化 Array 字段子表 |

#### 4.1.2 locales 表结构

```sql
-- PostgreSQL/SQLite 通用结构
CREATE TABLE posts_locales (
  id SERIAL PRIMARY KEY,
  _parent_id INTEGER NOT NULL,  -- 关联主表 ID
  _locale VARCHAR(10) NOT NULL,  -- 语言代码，如 'en', 'es'
  
  -- 本地化字段
  title VARCHAR(255),
  localized_description TEXT,
  -- ... 其他本地化字段
  
  FOREIGN KEY (_parent_id) REFERENCES posts(id)
)
```

#### 4.1.3 本地化字段检测

**位置**: `packages/drizzle/src/utilities/hasLocalesTable.ts`

```typescript
export const hasLocalesTable = ({
  fields,
  parentIsLocalized,
}: {
  fields: Field[]
  parentIsLocalized?: boolean
}): boolean => {
  return fields.some((field) => {
    // arrays 总是有单独的表
    if (field.type === 'array') {
      return false
    }
    // 检查字段是否影响数据且应该本地化
    if (fieldAffectsData(field) && fieldShouldBeLocalized({ field, parentIsLocalized })) {
      return true
    }
    // 递归检查子字段
    if (fieldHasSubFields(field)) {
      return hasLocalesTable({
        fields: field.fields,
        parentIsLocalized: parentIsLocalized || ('localized' in field && field.localized),
      })
    }
    // 检查 tabs
    if (field.type === 'tabs') {
      return field.tabs.some((tab) =>
        hasLocalesTable({
          fields: tab.fields,
          parentIsLocalized: parentIsLocalized || tab.localized,
        }),
      )
    }
    return false
  })
}
```

#### 4.1.4 表名后缀配置

**位置**: `packages/drizzle/src/types.ts:420`

```typescript
type DrizzleAdapter = {
  localesSuffix?: string  // 默认为 '_locales'
  // ...
}
```

#### 4.1.5 数据写入流程

**位置**: `packages/drizzle/src/upsertRow/index.ts`

写入时，本地化数据被分离到 `locales` 表：

```typescript
// 1. 主表写入
const insertedRow = await adapter.insert({
  db,
  tableName,
  values: rowToInsert.row,
})

// 2. locales 表写入
if (localesToInsert.length > 0) {
  const localeTableName = `${tableName}${adapter.localesSuffix}`
  const localeTable = adapter.tables[`${tableName}${adapter.localesSuffix}`]

  // 更新操作：先删除旧的 locale 数据
  if (operation === 'update') {
    await adapter.deleteWhere({
      db,
      tableName: localeTableName,
      where: eq(localeTable._parentID, insertedRow.id),
    })
  }

  // 插入新的 locale 数据
  await adapter.insert({
    db,
    tableName: localeTableName,
    values: localesToInsert.map((localeRow) => ({
      ...localeRow,
      _parentID: insertedRow.id,
    })),
  })
}
```

#### 4.1.6 数据读取流程

**位置**: `packages/drizzle/src/transform/read/traverseFields.ts`

读取时，从 locales 表 JOIN 数据：

```typescript
// 本地化字段查询时关联 locales 表
const arrayTableNameWithLocales = `${arrayTableName}${adapter.localesSuffix}`

if (adapter.tables[arrayTableNameWithLocales]) {
  withArray.with._locales = {
    columns:
      typeof arraySelect === 'object'
        ? { _locale: true }
        : { id: false, _parentID: false },
    with: {},
  }
}
```

### 4.2 本地化字段的两种存储模式

#### 模式 A：独立 locales 表（推荐）

适用于 PostgreSQL、SQLite（Drizzle 适配器）

```
主表 posts:
┌────┬───────────────┐
│ id │ description   │  ← 非本地化字段
├────┼───────────────┤
│ 1  │ "Non-local"   │
└────┴───────────────┘

locales 表 posts_locales:
┌────┬───────────┬────────┬─────────────────────┐
│ id │ _parent_id│ _locale│ title               │  ← 本地化字段
├────┼───────────┼────────┼─────────────────────┤
│ 1  │ 1         │ en     │ "English Title"     │
│ 2  │ 1         │ es     │ "Spanish Title"     │
└────┴───────────┴────────┴─────────────────────┘
```

#### 模式 B：JSON 内嵌（部分字段类型）

对于某些复杂结构（如 blocksAsJSON 模式），本地化数据以 JSON 格式存储：

```
posts 表中的 blocks 字段（JSON 列）:
{
  "en": [{"blockType": "text", "text": "English content"}],
  "es": [{"blockType": "text", "text": "Spanish content"}]
}
```

### 4.3 复合字段的本地化处理

#### 4.3.1 本地化 Array 字段

当 `array` 字段标记为 `localized: true` 时：

```typescript
{
  name: 'items',
  type: 'array',
  localized: true,
  fields: [
    { name: 'name', type: 'text' },
  ],
}
```

**数据结构**（查询 `locale=all` 时）:

```typescript
{
  items: {
    en: [{ name: 'English Item 1' }, { name: 'English Item 2' }],
    es: [{ name: 'Spanish Item 1' }],
  }
}
```

**关键代码**（读取时）:

```typescript
// packages/drizzle/src/find/traverseFields.ts:207-231

if (typeof arraySelect === 'object') {
  if (adapter.tables[arrayTableName]._locale) {
    withArray.columns._locale = true  // 包含 _locale 字段
  }
  // ...
}

const arrayTableNameWithLocales = `${arrayTableName}${adapter.localesSuffix}`

if (adapter.tables[arrayTableNameWithLocales]) {
  withArray.with._locales = {
    columns: typeof arraySelect === 'object'
      ? { _locale: true }
      : { id: false, _parentID: false },
    with: {},
  }
}
```

#### 4.3.2 本地化 Group 字段

```typescript
{
  name: 'meta',
  type: 'group',
  localized: true,
  fields: [
    { name: 'title', type: 'text' },
    { name: 'description', type: 'text' },
  ],
}
```

**数据结构**:

```typescript
{
  meta: {
    en: { title: 'EN Title', description: 'EN Desc' },
    es: { title: 'ES Title', description: 'ES Desc' },
  }
}
```

#### 4.3.3 本地化 Blocks 字段

```typescript
{
  name: 'layout',
  type: 'blocks',
  localized: true,
  blocks: [
    {
      slug: 'text',
      fields: [{ name: 'content', type: 'text' }],
    },
  ],
}
```

**数据结构**:

```typescript
{
  layout: {
    en: [
      { blockType: 'text', id: '1', content: 'English content' },
    ],
    es: [
      { blockType: 'text', id: '2', content: 'Spanish content' },
    ],
  }
}
```

---

## 5. Fallback 语言链工作原理

### 5.1 核心概念

Fallback 机制允许在当前语言没有值时，使用其他语言的值作为替补。

**配置层级**:
1. **全局配置**: `localization.fallback: boolean`
2. **语言特定配置**: `locale.fallbackLocale: string | string[]`
3. **请求级别配置**: `fallbackLocale` 参数

### 5.2 Fallback 解析流程

**位置**: `packages/payload/src/utilities/sanitizeFallbackLocale.ts`

```
┌─────────────────────────────────────────────────────────────┐
│                    sanitizeFallbackLocale                    │
├─────────────────────────────────────────────────────────────┤
│  1. fallbackLocale 为 undefined/null?                        │
│     ├─ 是 → 检查 localization.fallback                       │
│     │       ├─ true → 检查当前语言的 fallbackLocale          │
│     │       │       ├─ 存在 → 使用该语言                    │
│     │       │       └─ 不存在 → 使用 defaultLocale          │
│     │       └─ false → 返回 false (禁用 fallback)            │
│     └─ 否 → 继续检查其他情况                                  │
│                                                              │
│  2. fallbackLocale 是数组?                                   │
│     └─ 是 → 过滤掉无效的语言代码                              │
│                                                              │
│  3. fallbackLocale 是字符串?                                 │
│     ├─ 'false'/'none'/'null' → 返回 false                   │
│     └─ 其他 → 验证是否为有效语言代码                          │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 完整代码实现

```typescript
// packages/payload/src/utilities/sanitizeFallbackLocale.ts:18-51

export const sanitizeFallbackLocale = ({
  fallbackLocale,
  locale,
  localization,
}: Args): TypedFallbackLocale => {
  // 情况 1: fallbackLocale 未指定
  if (fallbackLocale === undefined || fallbackLocale === null) {
    if (localization && localization.fallback) {
      // 检查语言特定的 fallback
      const localeSpecificFallback = localization.locales.length
        ? localization.locales.find((localeConfig) => localeConfig.code === locale)?.fallbackLocale
        : undefined

      if (localeSpecificFallback) {
        return localeSpecificFallback  // 使用语言特定的 fallback
      }

      return localization.defaultLocale  // 使用默认语言
    }

    return false  // 全局 fallback 禁用
  } 
  // 情况 2: fallbackLocale 是数组（自定义 fallback 链）
  else if (Array.isArray(fallbackLocale)) {
    return fallbackLocale.filter((localeCode) => localization.localeCodes.includes(localeCode))
  } 
  // 情况 3: fallbackLocale 是字符串
  else if (fallbackLocale) {
    // 特殊值：禁用 fallback
    if (['false', 'none', 'null'].includes(fallbackLocale)) {
      return false
    }

    // 验证是否为有效语言
    if (localization.localeCodes.includes(fallbackLocale)) {
      return fallbackLocale
    }
  }

  return false
}
```

### 5.4 Fallback 链示例

#### 示例 1：简单配置

```typescript
localization: {
  defaultLocale: 'en',
  fallback: true,
  locales: [
    { code: 'en', label: 'English' },
    { code: 'es', label: 'Spanish' },
    { code: 'pt', label: 'Portuguese', fallbackLocale: 'es' },
  ],
}
```

**Fallback 行为**:

| 查询语言 | Fallback 链 | 说明 |
|---------|------------|------|
| `en` | 无 | 默认语言，不 fallback |
| `es` | `en` | 无语言特定 fallback，使用 defaultLocale |
| `pt` | `es` → `en` | 先 fallback 到 es，再 fallback 到 en |

#### 示例 2：请求级别覆盖

```typescript
// 覆盖全局配置，指定自定义 fallback 链
await payload.find({
  collection: 'posts',
  locale: 'pt',
  fallbackLocale: ['fr', 'en'],  // 先 fr，再 en
})
```

#### 示例 3：禁用 fallback

```typescript
// 方式 1: URL 参数
GET /api/posts?locale=pt&fallbackLocale=false

// 方式 2: Local API
await payload.find({
  collection: 'posts',
  locale: 'pt',
  fallbackLocale: false,
})
```

### 5.5 字段级别 Fallback 值获取

**位置**: `packages/payload/src/fields/hooks/beforeValidate/getFallbackValue.ts`

在保存数据时，如果当前语言没有值，会尝试获取 fallback 值：

```typescript
export async function getFallbackValue({
  field,
  req,
  siblingDoc,
}: {
  field: FieldAffectingData
  req: PayloadRequest
  siblingDoc: JsonObject
}): Promise<JsonValue> {
  let fallbackValue: JsonValue = undefined
  
  if ('name' in field && field.name) {
    // 1. 从 siblingDoc 获取现有值（可能来自其他语言）
    if (typeof siblingDoc[field.name] !== 'undefined') {
      fallbackValue = cloneDataFromOriginalDoc(siblingDoc[field.name])
    } 
    // 2. 使用字段的 defaultValue
    else if ('defaultValue' in field && typeof field.defaultValue !== 'undefined') {
      fallbackValue = await getDefaultValue({
        defaultValue: field.defaultValue,
        locale: req.locale || '',
        req,
        user: req.user,
      })
    }
  }

  return fallbackValue
}
```

---

## 6. 数据结构示例

### 6.1 查询 `locale=en` 时的返回结构

```typescript
// GET /api/posts/1?locale=en

{
  id: '1',
  title: 'English Title',  // 直接返回字符串，不是对象
  description: 'Non-localized description',
  localizedDescription: 'English localized description',
  items: [
    // 本地化 array 只返回当前语言的 items
    { name: 'English Item 1' },
    { name: 'English Item 2' },
  ],
  meta: {
    // 本地化 group 只返回当前语言的数据
    title: 'EN Meta Title',
    description: 'EN Meta Description',
  },
}
```

### 6.2 查询 `locale=all` 时的返回结构

```typescript
// GET /api/posts/1?locale=all

{
  id: '1',
  title: {
    en: 'English Title',
    es: 'Spanish Title',
    pt: null,  // 葡萄牙语无值
  },
  description: 'Non-localized description',  // 非本地化字段不变
  localizedDescription: {
    en: 'English description',
    es: 'Spanish description',
  },
  items: {
    // 本地化 array 是对象，key 是语言代码
    en: [{ name: 'EN Item 1' }, { name: 'EN Item 2' }],
    es: [{ name: 'ES Item 1' }],
  },
  meta: {
    // 本地化 group 是对象
    en: { title: 'EN Title', description: 'EN Desc' },
    es: { title: 'ES Title', description: 'ES Desc' },
  },
}
```

### 6.3 数据库层面的原始数据

**主表 `posts`**:

| id | description |
|----|-------------|
| 1  | "Non-local" |

**Locales 表 `posts_locales`**:

| id | _parent_id | _locale | title | localized_description |
|----|------------|---------|-------|----------------------|
| 1  | 1          | en      | "English Title" | "English desc" |
| 2  | 1          | es      | "Spanish Title" | "Spanish desc" |
| 3  | 1          | pt      | NULL | NULL |

---

## 7. 关键代码参考

### 7.1 核心文件列表

| 文件路径 | 职责 |
|---------|------|
| `packages/payload/src/config/types.ts:513-582` | 本地化配置类型定义 |
| `packages/payload/src/utilities/sanitizeFallbackLocale.ts` | Fallback 语言解析 |
| `packages/payload/src/utilities/addLocalesToRequest.ts` | 请求语言参数处理 |
| `packages/payload/src/utilities/createPayloadRequest.ts` | HTTP 请求转 PayloadRequest |
| `packages/payload/src/utilities/mergeLocalizedData.ts` | 多语言数据合并 |
| `packages/payload/src/utilities/filterDataToSelectedLocales.ts` | 按语言过滤数据 |
| `packages/ui/src/providers/Locale/index.tsx` | 管理后台语言上下文 |
| `packages/drizzle/src/utilities/hasLocalesTable.ts` | 检测是否需要 locales 表 |
| `packages/drizzle/src/upsertRow/index.ts` | 数据写入（含 locales 表） |
| `packages/drizzle/src/transform/read/index.ts` | 数据读取转换 |
| `packages/drizzle/src/transform/write/index.ts` | 数据写入转换 |

### 7.2 数据合并逻辑

**位置**: `packages/payload/src/utilities/mergeLocalizedData.ts`

用于更新数据时，只更新指定语言，保留其他语言的数据：

```typescript
// 示例：更新英语版本，保留西班牙语版本
const result = mergeLocalizedData({
  configBlockReferences: [],
  dataWithLocales: {
    title: { en: 'Updated English' },  // 只提供英语新值
  },
  docWithLocales: {
    title: { en: 'Old English', es: 'Old Spanish' },  // 现有数据
  },
  fields: [{ name: 'title', type: 'text', localized: true }],
  selectedLocales: ['en'],  // 只更新英语
})

// 结果:
// {
//   title: {
//     en: 'Updated English',  // 已更新
//     es: 'Old Spanish',      // 保留不变
//   }
// }
```

### 7.3 数据过滤逻辑

**位置**: `packages/payload/src/utilities/filterDataToSelectedLocales.ts`

用于从 `locale=all` 返回的完整数据中，过滤出指定语言的数据：

```typescript
const result = filterDataToSelectedLocales({
  configBlockReferences: [],
  docWithLocales: {
    title: { en: 'English', es: 'Spanish', de: 'German' },
  },
  fields: [{ name: 'title', type: 'text', localized: true }],
  selectedLocales: ['en'],  // 只保留英语
})

// 结果:
// { title: { en: 'English' } }
```

---

## 8. 总结

### 8.1 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│                      管理后台层 (Admin UI)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │LocaleProvider│  │ useLocale   │  │ 语言切换 UI 组件     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        API 层 (API Layer)                      │
│  ┌─────────────────┐  ┌──────────────────────────────────┐   │
│  │createPayloadReq │  │ sanitizeLocales / sanitizeFall-  │   │
│  │                 │  │ backLocale                        │   │
│  └─────────────────┘  └──────────────────────────────────┘   │
│  参数: locale, fallbackLocale, locale=all                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据层 (Data Layer)                       │
│  ┌─────────────────┐  ┌──────────────────────────────────┐   │
│  │ 主表 (Main)     │  │ Locales 表 ({table}_locales)     │   │
│  │ - 非本地化字段   │  │ - _parent_id (外键)               │   │
│  │                 │  │ - _locale (语言代码)               │   │
│  │                 │  │ - 本地化字段                        │   │
│  └─────────────────┘  └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 关键决策点

1. **存储策略**: 本地化字段独立存储在 `_locales` 表，通过 `_parent_id` 关联主表
2. **查询模式**: 支持单语言查询（返回扁平结构）和全语言查询（返回 `{en, es, ...}` 对象）
3. **Fallback 链**: 支持全局配置、语言特定配置、请求级别覆盖三层
4. **数据一致性**: 更新时使用 `mergeLocalizedData` 保证只更新指定语言，不影响其他语言

### 8.3 使用建议

1. **字段设计**: 仔细评估哪些字段需要本地化，避免过度本地化
2. **Fallback 配置**: 合理设计 fallback 链，特别是对于区域语言（如 pt → es → en）
3. **性能考虑**: `locale=all` 查询会返回更多数据，仅在必要时使用
4. **数据库索引**: 考虑为 `_locale` 字段和常用查询组合创建索引

---

*报告生成日期: 2026-05-05*
*基于 PayloadCMS 代码库分析*
