# PayloadCMS 多语言字段实现深度分析报告

## 目录

1. [概述](#1-概述)
2. [请求参数如何落到数据库查询：完整链路](#2-请求参数如何落到数据库查询完整链路)
3. [管理后台本地化字段显示链路](#3-管理后台本地化字段显示链路)
4. [MongoDB vs SQL 适配器：存储与查询的本质差异](#4-mongodb-vs-sql-适配器存储与查询的本质差异)
5. [Fallback 机制：三层规则与优先级](#5-fallback-机制三层规则与优先级)
6. [完整数据结构示例](#6-完整数据结构示例)
7. [关键代码位置索引](#7-关键代码位置索引)

---

## 1. 概述

PayloadCMS 的多语言（Localization）系统采用**分层协作架构**，不同层级有明确的职责分工。理解这个系统的关键是要区分：

| 层级 | 核心职责 | 关键参数 |
|------|----------|----------|
| **API 层** | 参数解析、上下文传递 | `locale`, `fallbackLocale`, `flattenLocales`, `selectedLocales` |
| **查询层** | 根据 `locale` 定位数据 | 只使用 `locale` |
| **后处理层** | 合并 fallback 值 | 只使用 `fallbackLocale` |
| **存储层** | 适配器决定存储方式 | 无 |

**核心设计原则**：
- `locale` 决定**查询什么数据**（从哪里读、写到哪里）
- `fallbackLocale` 决定**如何处理缺失值**（查询后合并）
- 这两个参数在代码中是**完全分离**的！

---

## 2. 请求参数如何落到数据库查询：完整链路

### 2.1 参数解析阶段

**入口**：`createPayloadRequest`

当 HTTP 请求到达时，参数从以下来源解析：

```typescript
// packages/payload/src/utilities/createPayloadRequest.ts:66-93

// 1. 从 URL searchParams 获取
let locale = searchParams.get('locale')

// 2. 从 query string 解析（支持数组格式）
const query = qs.parse(queryToParse, {
  arrayLimit: 1000,
  depth: 10,
})

const fallbackFromRequest = (
  query.fallbackLocale ||
  searchParams.get('fallback-locale') ||
  searchParams.get('fallbackLocale')
)

// 3. 调用 sanitizeLocales 处理
if (localization) {
  const locales = sanitizeLocales({
    fallbackLocale: fallbackLocale!,
    locale: locale!,
    localization,
  })
  fallbackLocale = locales.fallbackLocale!
  locale = locales.locale!
}

// 4. 结果存储到 req
const customRequest: CustomPayloadRequestProperties = {
  locale,
  fallbackLocale: fallbackLocale!,
  // ...
}
```

### 2.2 支持的参数格式

| 参数 | 格式 | 说明 |
|------|------|------|
| `locale` | `string` | 指定查询语言，如 `'en'`, `'es'` |
| `locale` | `'all'` 或 `'*'` | 获取所有语言版本 |
| `fallbackLocale` | `string` | 指定单个 fallback 语言 |
| `fallbackLocale` | `string[]` | 指定 fallback 链，按顺序尝试 |
| `fallbackLocale` | `'false'` / `'none'` / `'null'` | 禁用 fallback |
| `flattenLocales` | `boolean` | 当 `locale=all` 时是否扁平化 |
| `selectedLocales` | `string[]` | 当 `locale=all` 时筛选特定语言 |

### 2.3 参数 sanitize 流程

**入口**：`sanitizeLocales` → `sanitizeFallbackLocale`

```typescript
// packages/payload/src/utilities/addLocalesToRequest.ts:63-88

export const sanitizeLocales = ({
  fallbackLocale,
  locale,
  localization,
}: SanitizeLocalesArgs): SanitizeLocalesReturn => {
  // 1. 处理 fallbackLocale
  if (localization) {
    fallbackLocale = sanitizeFallbackLocale({
      fallbackLocale,
      locale,
      localization,
    })!
  }

  // 2. 处理 locale
  if (['*', 'all'].includes(locale)) {
    locale = 'all'  // 标准化 'all' 值
  } else if (localization && !localization.localeCodes.includes(locale) && localization.fallback) {
    locale = localization.defaultLocale  // 无效语言回退到默认
  }

  return { fallbackLocale, locale }
}
```

### 2.4 数据库查询阶段

**关键发现**：**`locale` 在查询时使用，`fallbackLocale` 不在查询阶段使用！**

```typescript
// packages/payload/src/collections/operations/find.ts:205-217

// 调用数据库查询时，只传递 locale，没有 fallbackLocale！
result = await payload.db.find<DataFromCollectionSlug<TSlug>>({
  collection: collectionConfig.slug,
  draftsEnabled,
  joins: req.payloadAPI === 'GraphQL' ? false : sanitizedJoins,
  limit: sanitizedLimit,
  locale: locale!,        // <-- 只有 locale
  page: sanitizedPage,
  pagination,
  req,
  select,
  sort,
  where: fullWhere,
  // 注意：没有 fallbackLocale 参数！
})
```

### 2.5 afterRead 阶段：fallback 生效

**`fallbackLocale` 只在 `afterRead` 阶段使用**，用于合并 fallback 值：

```typescript
// packages/payload/src/collections/operations/find.ts:318-338

// afterRead 阶段才使用 fallbackLocale
result.docs = await Promise.all(
  result.docs.map(async (doc) =>
    afterRead<DataFromCollectionSlug<TSlug>>({
      collection: collectionConfig,
      context: req.context,
      currentDepth,
      depth: depth!,
      doc,
      draft: draftsEnabled!,
      fallbackLocale: fallbackLocale!,  // <-- 这里才使用
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

### 2.6 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HTTP 请求到达                                    │
│                    URL: /api/posts?locale=pt&fallbackLocale=es,en           │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        createPayloadRequest                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 1. 从 searchParams 解析:                                                │  │
│  │    locale = 'pt'                                                        │  │
│  │    fallbackLocale = ['es', 'en']                                       │  │
│  │                                                                         │  │
│  │ 2. 调用 sanitizeLocales → sanitizeFallbackLocale                      │  │
│  │    - 验证 locale 是否有效                                               │  │
│  │    - 验证 fallbackLocale 是否有效                                       │  │
│  │                                                                         │  │
│  │ 3. 存储到 req:                                                          │  │
│  │    req.locale = 'pt'                                                   │  │
│  │    req.fallbackLocale = ['es', 'en']                                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           findOperation                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 1. 从 req 提取参数:                                                     │  │
│  │    locale = 'pt'                                                        │  │
│  │    fallbackLocale = ['es', 'en']                                       │  │
│  │                                                                         │  │
│  │ 2. 调用 payload.db.find({ locale: 'pt' })  ← 注意：只传 locale!       │  │
│  │    └─> 数据库适配器根据 locale 查询数据                                  │  │
│  │        - MongoDB: 查询路径转换为 'field.pt'                           │  │
│  │        - SQL: JOIN _locales 表 WHERE _locale = 'pt'                  │  │
│  │                                                                         │  │
│  │ 3. afterRead 阶段:                                                      │  │
│  │    调用 afterRead({ fallbackLocale: ['es', 'en'], ... })              │  │
│  │    └─> 合并 fallback 值                                                 │  │
│  │        如果 'pt' 无值，尝试 'es'，再尝试 'en'                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 管理后台本地化字段显示链路

### 3.1 LocaleProvider：语言上下文管理

**位置**：`packages/ui/src/providers/Locale/index.tsx`

管理后台的语言状态由 `LocaleProvider` 统一管理，这是一个 React Context Provider。

#### 语言解析优先级

```typescript
// packages/ui/src/providers/Locale/index.tsx:54-66

const [locale, setLocale] = React.useState<Locale>(() => {
  if (!localization || (localization && !localization.locales.length)) {
    return {} as Locale
  }

  return (
    findLocaleFromCode(localization, localeFromParams) ||  // 1. URL 参数 ?locale=es
    findLocaleFromCode(localization, initialLocaleFromPrefs) ||  // 2. 用户偏好设置
    findLocaleFromCode(localization, defaultLocale) ||  // 3. 配置中的默认语言
    findLocaleFromCode(localization, localization.locales[0].code)  // 4. 第一个配置的语言
  )
})
```

#### 完整的状态管理逻辑

```typescript
// packages/ui/src/providers/Locale/index.tsx:79-105

// 监听 URL 参数变化
const localeFromParams = searchParams?.get('locale')
const initialLocaleFromPrefs = preferences?.locale

// 当 URL 参数变化时更新状态
React.useEffect(() => {
  if (localization?.locales?.length) {
    if (localeFromParams) {
      const matchingLocale = findLocaleFromCode(localization, localeFromParams)
      if (matchingLocale) {
        if (!locale || matchingLocale.code !== locale.code) {
          setLocale(matchingLocale)  // 更新状态
        }
      }
    } else if (initialLocaleFromPrefs) {
      const matchingLocale = findLocaleFromCode(localization, initialLocaleFromPrefs)
      if (matchingLocale) {
        setLocale(matchingLocale)
      }
    } else if (defaultLocale) {
      const matchingLocale = findLocaleFromCode(localization, defaultLocale)
      if (matchingLocale) {
        setLocale(matchingLocale)
      }
    }
  }
}, [localeFromParams, localization, initialLocaleFromPrefs, defaultLocale])
```

### 3.2 管理后台 → API 请求链路

当管理后台切换语言时，触发以下流程：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 用户点击语言切换按钮 (如 "Spanish")                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. URL 更新: /admin/collections/posts?id=1&locale=es                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. LocaleProvider 检测到 searchParams 变化                                  │
│     调用 setLocale({ code: 'es', label: 'Spanish', ... })                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 组件重新渲染，调用 useLocale() 获取新语言                                  │
│     const { locale } = useLocale()  // { code: 'es', ... }                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 数据获取 Hook (如 usePayloadAPI) 发起 API 请求                           │
│     GET /api/posts/1?locale=es                                               │
│     └─> locale 参数传递到后端                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. 后端处理请求 (createPayloadRequest)                                       │
│     解析 locale = 'es'，调用数据库查询                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 管理后台字段显示

管理后台使用 `getDisplayedFieldValue` 来获取字段的显示值：

**位置**：`packages/ui/src/utilities/getDisplayedFieldValue.ts`

但更重要的是，管理后台通过 API 获取数据时，已经根据 `locale` 参数获取了特定语言的数据。因此：

- 当 `locale=en` 时，API 返回的 `title` 就是英语版本（或 fallback 后的值）
- 当 `locale=all` 时，API 返回的 `title` 是对象 `{ en: "...", es: "...", ... }`

### 3.4 管理后台 API 请求示例

管理后台的 API 请求通过 `usePayloadAPI` 发起：

```typescript
// packages/ui/src/hooks/usePayloadAPI.ts

// 当 locale 变化时，API 请求会携带新的 locale 参数
const { data } = usePayloadAPI(
  `${serverURL}/api/posts/${id}?locale=${locale.code}`,
  { initialData }
)
```

---

## 4. MongoDB vs SQL 适配器：存储与查询的本质差异

这是之前报告中最不准确的部分。**两种适配器的存储方式完全不同**，理解这一点至关重要。

### 4.1 核心差异对比

| 维度 | MongoDB 适配器 | SQL (Drizzle) 适配器 |
|------|----------------|----------------------|
| **存储模型** | 嵌入式对象 | 规范化表结构 |
| **本地化字段位置** | 同文档，字段值为对象 | 独立 `_locales` 表 |
| **查询方式** | 路径转换 (`field.en`) | JOIN + WHERE `_locale` |
| **语言版本数量** | 一个文档包含所有语言 | 每个语言一条记录 |
| **写入方式** | 单文档更新 | 主表 + locales 表分离写入 |

### 4.2 MongoDB 适配器：嵌入式对象模型

#### 存储结构

**所有语言版本存储在同一个文档中**，本地化字段的值是一个对象，key 是语言代码：

```javascript
// MongoDB 文档结构 (posts 集合)
{
  _id: ObjectId("..."),
  id: "1",
  
  // 非本地化字段：直接存储值
  description: "This is a non-localized field",
  
  // 本地化字段：存储为对象，key 是语言代码
  title: {
    en: "English Title",
    es: "Spanish Title",
    pt: null  // 葡萄牙语版本不存在或为 null
  },
  
  localizedDescription: {
    en: "English description",
    es: "Spanish description"
  },
  
  // 本地化 array：也是对象
  items: {
    en: [
      { name: "English Item 1" },
      { name: "English Item 2" }
    ],
    es: [
      { name: "Spanish Item 1" }
    ]
  },
  
  // 本地化 group：对象
  meta: {
    en: { seoTitle: "EN SEO", seoDesc: "EN Description" },
    es: { seoTitle: "ES SEO", seoDesc: "ES Description" }
  }
}
```

#### 查询路径转换

**关键函数**：`getLocalizedPaths`

当查询本地化字段时，MongoDB 适配器会自动将路径转换为包含语言代码的路径：

```typescript
// packages/payload/src/database/getLocalizedPaths.ts:145-156

// 检查下一个 segment 是否是语言代码
const nextSegmentIsLocale =
  localizationConfig &&
  localizationConfig.localeCodes.includes(nextSegment) &&
  currentFieldIsLocalized

if (nextSegmentIsLocale) {
  // 如果查询显式指定了语言（如 title.en），跳过自动添加
  i += 1
  currentPath = `${currentPath}.${nextSegment}`
} else if (localizationConfig && currentFieldIsLocalized) {
  // 否则自动添加当前 locale 作为后缀
  currentPath = `${currentPath}.${locale}`
}
```

#### 查询示例

```
场景：locale = 'en'，查询 { title: { equals: 'Test' } }

MongoDB 实际查询：
{ "title.en": "Test" }  // 路径自动转换为 title.en


场景：locale = 'all'，查询 { title: { exists: true } }

MongoDB 实际查询：
{ "title": { $exists: true } }  // 不添加语言后缀，查询整个对象
```

#### 排序处理

**位置**：`packages/db-mongodb/src/queries/getLocalizedSortProperty.ts`

排序时也需要本地化路径：

```typescript
// 按 title 排序，locale = 'en'
// 实际排序字段：title.en
```

### 4.3 SQL (Drizzle) 适配器：规范化表模型

#### 表结构设计

**主表和 locales 表分离**，通过 `_parent_id` 关联：

```sql
-- 主表：posts
-- 只存储非本地化字段
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  description TEXT  -- 非本地化字段
);

-- locales 表：posts_locales
-- 存储本地化字段，每个语言一条记录
CREATE TABLE posts_locales (
  id SERIAL PRIMARY KEY,
  _parent_id INTEGER NOT NULL,    -- 关联主表 id
  _locale VARCHAR(10) NOT NULL,   -- 语言代码: 'en', 'es', 'pt'
  title VARCHAR(255),              -- 本地化字段
  localized_description TEXT,      -- 本地化字段
  FOREIGN KEY (_parent_id) REFERENCES posts(id)
);

-- 主表数据
INSERT INTO posts (id, description) VALUES
  (1, 'Non-localized description');

-- locales 表数据
INSERT INTO posts_locales (_parent_id, _locale, title, localized_description) VALUES
  (1, 'en', 'English Title', 'English description'),
  (1, 'es', 'Spanish Title', 'Spanish description');
```

#### 本地化 Array 字段的表结构

如果 `items` 是本地化 array 字段：

```sql
-- 主表 posts 不变

-- array 子表：posts_items
-- 注意：本地化 array 的子表也有 _locale 字段！
CREATE TABLE posts_items (
  id SERIAL PRIMARY KEY,
  _parent_id INTEGER NOT NULL,
  _locale VARCHAR(10),           -- 语言代码（本地化 array 特有）
  _order INTEGER,                -- 排序
  name VARCHAR(255)
);

-- 数据
INSERT INTO posts_items (_parent_id, _locale, _order, name) VALUES
  (1, 'en', 1, 'English Item 1'),
  (1, 'en', 2, 'English Item 2'),
  (1, 'es', 1, 'Spanish Item 1');
```

#### 查询流程

**Drizzle 适配器的查询不依赖路径转换**，而是通过 JOIN 来获取本地化数据：

```typescript
// 读取时：从主表 + locales 表 JOIN
// 写入时：分离主表数据和 locales 数据

// 位置：packages/drizzle/src/upsertRow/index.ts

// 1. 主表写入
const insertedRow = await adapter.insert({
  db,
  tableName,
  values: rowToInsert.row,
})

// 2. locales 表写入（如果有本地化字段）
if (localesToInsert.length > 0) {
  const localeTableName = `${tableName}${adapter.localesSuffix}`  // 'posts_locales'
  
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

#### 读取转换

**位置**：`packages/drizzle/src/transform/read/traverseFields.ts`

Drizzle 适配器从数据库读取后，需要将规范化的数据转换为 Payload 期望的格式：

```typescript
// 对于本地化 array 字段，Drizzle 返回的每条记录有 _locale 字段
// 需要将它们重组为 { en: [...], es: [...] } 格式

if (isLocalized) {
  result[field.name] = fieldData.reduce((arrayResult, row) => {
    if (typeof row._locale === 'string') {
      if (!arrayResult[row._locale]) {
        arrayResult[row._locale] = []
      }
      const locale = row._locale
      const data = {}
      delete row._locale
      
      // 递归处理子字段
      const rowResult = traverseFields<T>({
        adapter,
        blocks,
        config,
        currentTableName: arrayTableName,
        dataRef: data,
        deletions,
        fieldPrefix: '',
        fields: field.flattenedFields,
        numbers,
        parentIsLocalized: parentIsLocalized || field.localized,
        path: `${sanitizedPath}${field.name}.${row._order - 1}`,
        relationships,
        table: row,
        tablePath: '',
        texts,
        topLevelTableName,
        withinArrayOrBlockLocale: locale,
      })
      
      arrayResult[locale].push(rowResult)
    }
    return arrayResult
  }, {})
}
```

### 4.4 两种适配器的查询对比

#### 相同查询：`locale='en'`，查询 `title = 'Test'`

**MongoDB**:
```javascript
// 单集合查询，路径转换
db.posts.find({ "title.en": "Test" })
```

**SQL**:
```sql
-- JOIN 两个表
SELECT p.*, pl.title, pl.localized_description
FROM posts p
JOIN posts_locales pl ON p.id = pl._parent_id
WHERE pl._locale = 'en' AND pl.title = 'Test'
```

#### 相同查询：`locale='all'`

**MongoDB**:
```javascript
// 直接查询，返回完整对象
db.posts.find({})
// 返回: { title: { en: "...", es: "..." }, ... }
```

**SQL**:
```sql
-- 需要收集所有 _locale 的记录并重组
SELECT p.*, pl._locale, pl.title, pl.localized_description
FROM posts p
LEFT JOIN posts_locales pl ON p.id = pl._parent_id
-- 然后在代码中重组为 { en: {...}, es: {...} } 格式
```

### 4.5 两种适配器的数据写入对比

#### 写入数据：`{ title: { en: "New EN", es: "New ES" } }`

**MongoDB**:
```javascript
// 单文档更新
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $set: { "title.en": "New EN", "title.es": "New ES" } }
)
```

**SQL**:
```sql
-- 1. 主表更新（如果有非本地化字段）
UPDATE posts SET description = '...' WHERE id = 1;

-- 2. 删除旧的 locales 记录
DELETE FROM posts_locales WHERE _parent_id = 1;

-- 3. 插入新的 locales 记录
INSERT INTO posts_locales (_parent_id, _locale, title) VALUES
  (1, 'en', 'New EN'),
  (1, 'es', 'New ES');
```

### 4.6 表名配置

**位置**：`packages/drizzle/src/types.ts:420`

```typescript
type DrizzleAdapter = {
  localesSuffix?: string  // 默认为 '_locales'
  // 可以配置为其他后缀，如 '_i18n'
}
```

---

## 5. Fallback 机制：三层规则与优先级

这是另一个需要澄清的关键点。**Fallback 有三个层级，优先级明确**。

### 5.1 核心函数：sanitizeFallbackLocale

**位置**：`packages/payload/src/utilities/sanitizeFallbackLocale.ts`

这是理解 fallback 机制的关键：

```typescript
export const sanitizeFallbackLocale = ({
  fallbackLocale,  // 来自请求的参数
  locale,          // 当前查询语言
  localization,    // 配置
}: Args): TypedFallbackLocale => {
  
  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 第一层：请求级别 fallbackLocale (优先级最高)                      │
  // │ 如果请求明确提供了 fallbackLocale 参数，直接使用                   │
  // └─────────────────────────────────────────────────────────────────┘
  
  // 情况 1: 请求未提供 fallbackLocale (undefined 或 null)
  if (fallbackLocale === undefined || fallbackLocale === null) {
    // 进入配置层判断
    if (localization && localization.fallback) {
      // 检查是否有语言专属的 fallback
      const localeSpecificFallback = localization.locales.length
        ? localization.locales.find((localeConfig) => localeConfig.code === locale)?.fallbackLocale
        : undefined

      if (localeSpecificFallback) {
        return localeSpecificFallback  // 使用语言专属 fallback
      }

      return localization.defaultLocale  // 使用默认语言
    }

    return false  // 全局 fallback 禁用
  }
  
  // 情况 2: 请求提供的 fallbackLocale 是数组（自定义 fallback 链）
  else if (Array.isArray(fallbackLocale)) {
    // 过滤掉无效的语言代码
    return fallbackLocale.filter((localeCode) => localization.localeCodes.includes(localeCode))
  }
  
  // 情况 3: 请求提供的 fallbackLocale 是字符串
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

### 5.2 三层 Fallback 规则：互斥而非串联

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Fallback 优先级（从高到低）                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  第一层：请求自定义 fallbackLocale (最高优先级)                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 触发条件: fallbackLocale !== undefined && fallbackLocale !== null     │  │
│  │ 来源: URL 参数 (?fallbackLocale=es) 或 Local API 选项                 │  │
│  │ 格式支持:                                                               │  │
│  │   - 字符串: ?fallbackLocale=es  (单个 fallback)                        │  │
│  │   - 数组: ?fallbackLocale=es&fallbackLocale=en (按顺序尝试)            │  │
│  │   - 特殊值: ?fallbackLocale=false (禁用)                                │  │
│  │ 行为: 完全使用请求参数，忽略配置中的任何 fallback 规则                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                              │
│  第二层：Locale 专属 fallbackLocale                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 触发条件:                                                                │  │
│  │   1. fallbackLocale === undefined || fallbackLocale === null          │  │
│  │   2. localization.fallback === true                                    │  │
│  │   3. 当前语言有 fallbackLocale 配置                                     │  │
│  │ 来源: 配置中 locales[i].fallbackLocale                                  │  │
│  │ 格式支持: 字符串或数组                                                    │  │
│  │ 行为: 返回当前语言配置的 fallbackLocale                                 │  │
│  │                                                                         │  │
│  │ 注意: 一旦满足此条件，就不会再触发默认 fallback！                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                              │
│  第三层：默认 fallback (defaultLocale)                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 触发条件:                                                                │  │
│  │   1. fallbackLocale === undefined || fallbackLocale === null          │  │
│  │   2. localization.fallback === true                                    │  │
│  │   3. 当前语言没有配置 fallbackLocale                                    │  │
│  │ 来源: localization.defaultLocale                                       │  │
│  │ 格式支持: 字符串（单个 defaultLocale）                                   │  │
│  │ 行为: fallback 到默认语言                                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**核心原则**：
> **Locale 专属 fallback 和 默认 fallback 是互斥的，不会自动串联。**
> 
> - 如果当前语言有 `fallbackLocale` 配置，就用这个配置（第二层）
> - 只有当当前语言**没有** `fallbackLocale` 配置时，才用 `defaultLocale`（第三层）
> - 数组形式的 `fallbackLocale` 本身就定义了完整的 fallback 链

### 5.3 配置示例与行为

```typescript
// 配置示例
localization: {
  defaultLocale: 'en',
  fallback: true,  // 全局启用 fallback
  locales: [
    { code: 'en', label: 'English' },
    { code: 'es', label: 'Spanish' },
    { code: 'pt', label: 'Portuguese', fallbackLocale: 'es' },
    { code: 'fr', label: 'French', fallbackLocale: ['es', 'en'] },
    { code: 'de', label: 'German' },  // 没有配置 fallbackLocale
  ],
}
```

#### 场景 1：查询 `locale='pt'`，**不提供** `fallbackLocale` 参数

```
fallback 链: pt → es → en

解析过程:
1. 请求未提供 fallbackLocale → 检查配置
2. localization.fallback === true → 继续
3. 查找 'pt' 的配置 → 找到 fallbackLocale: 'es'
4. 返回 'es'（作为 fallback 语言）

实际行为:
- 如果 'pt' 版本的字段无值，使用 'es' 版本
- 如果 'es' 版本也无值，使用 'en' 版本（defaultLocale）
```

#### 场景 2：查询 `locale='pt'`，**提供** `fallbackLocale='fr'`

```
fallback 链: pt → fr

解析过程:
1. 请求提供了 fallbackLocale = 'fr' → 直接使用
2. 忽略配置中的任何 fallbackLocale 规则

实际行为:
- 如果 'pt' 版本无值，使用 'fr' 版本
- 注意：配置中 'pt' 的 fallbackLocale: 'es' 被完全忽略！
```

#### 场景 3：查询 `locale='de'`，**不提供** `fallbackLocale` 参数

```
fallback 链: de → en

解析过程:
1. 请求未提供 fallbackLocale → 检查配置
2. localization.fallback === true → 继续
3. 查找 'de' 的配置 → 没有 fallbackLocale
4. 返回 defaultLocale = 'en'

实际行为:
- 如果 'de' 版本无值，使用 'en' 版本
```

#### 场景 4：查询 `locale='pt'`，提供 `fallbackLocale=false`

```
fallback 链: 禁用！

解析过程:
1. 请求提供了 fallbackLocale = 'false' → 返回 false
2. fallback 完全禁用

实际行为:
- 如果 'pt' 版本无值，返回 null/undefined
- 不使用任何 fallback 值
```

#### 场景 5：查询 `locale='fr'`，**不提供** `fallbackLocale` 参数

```
fallback 链: fr → es → en

解析过程:
1. 请求未提供 fallbackLocale → 检查配置
2. localization.fallback === true → 继续
3. 查找 'fr' 的配置 → 找到 fallbackLocale: ['es', 'en']（数组！）
4. 返回 ['es', 'en']

实际行为:
- 如果 'fr' 版本无值，尝试 'es' 版本
- 如果 'es' 版本也无值，尝试 'en' 版本
- 数组形式的 fallbackLocale 定义了完整的 fallback 链
```

### 5.4 两种 Fallback 的关键区别

| 维度 | Locale 专属 fallback | 请求自定义 fallback |
|------|----------------------|---------------------|
| **定义位置** | 配置中的 `locales[i].fallbackLocale` | URL 参数或 Local API 选项 |
| **优先级** | 中等（第二层） | 最高（第一层） |
| **触发条件** | `localization.fallback=true` 且请求未提供参数 | 请求明确提供参数 |
| **格式支持** | 字符串或数组 | 字符串、数组、特殊值 `'false'` |
| **行为** | 只影响配置的语言 | 完全覆盖任何配置 |

**核心原则**：
> **请求级别的 `fallbackLocale` 参数是"覆盖模式"，不是"补充模式"。**
> 
> 一旦提供，配置中的任何 `fallbackLocale` 规则都被忽略。

### 5.5 Fallback 实际应用位置

**重要**：`fallbackLocale` 不在数据库查询阶段使用，只在 `afterRead` 阶段使用。

```typescript
// afterRead 阶段才会合并 fallback 值
// 位置：packages/payload/src/fields/hooks/afterRead/index.ts

// 伪代码逻辑
for (const field of fields) {
  if (field.localized) {
    // 获取当前语言的值
    const currentValue = doc[field.name]?.[locale]
    
    if (currentValue !== null && currentValue !== undefined) {
      // 当前语言有值，使用当前值
      result[field.name] = currentValue
    } else if (fallbackLocale) {
      // 当前语言无值，尝试 fallback
      if (Array.isArray(fallbackLocale)) {
        // 数组形式：按顺序尝试
        for (const fallbackLang of fallbackLocale) {
          const fallbackValue = doc[field.name]?.[fallbackLang]
          if (fallbackValue !== null && fallbackValue !== undefined) {
            result[field.name] = fallbackValue
            break
          }
        }
      } else {
        // 字符串形式：直接使用
        const fallbackValue = doc[field.name]?.[fallbackLocale]
        if (fallbackValue !== null && fallbackValue !== undefined) {
          result[field.name] = fallbackValue
        }
      }
    }
    // 如果没有 fallback 或 fallback 也无值，返回 null/undefined
  }
}
```

### 5.6 数据结构：locale=all vs 单语言查询

#### 查询 `locale=all`

返回的是**完整的语言对象**，不应用任何 fallback：

```javascript
{
  id: "1",
  title: {
    en: "English Title",
    es: "Spanish Title",
    pt: null,  // 葡萄牙语版本明确为 null 或不存在
    fr: "French Title"
  },
  description: "Non-localized field"
}
```

#### 查询 `locale='pt'`，`fallbackLocale=['es', 'en']`

返回的是**扁平结构**，已应用 fallback：

```javascript
{
  id: "1",
  title: "Spanish Title",  // pt 无值，fallback 到 es
  description: "Non-localized field"
}
```

---

## 6. 完整数据结构示例

### 6.1 MongoDB 存储结构

```javascript
// posts 集合中的单个文档
{
  _id: ObjectId("650000000000000000000001"),
  id: "1",
  
  // 非本地化字段
  description: "This is a non-localized field that appears in all languages.",
  viewCount: 100,
  
  // 本地化文本字段
  title: {
    en: "Welcome to Our Website",
    es: "Bienvenido a Nuestro Sitio Web",
    pt: null,
    de: "Willkommen auf unserer Website"
  },
  
  // 本地化复选框
  localizedCheckbox: {
    en: true,
    es: false,
    de: true
  },
  
  // 本地化 group
  meta: {
    en: {
      seoTitle: "English SEO Title",
      seoDescription: "English SEO Description"
    },
    es: {
      seoTitle: "Spanish SEO Title",
      seoDescription: "Spanish SEO Description"
    }
  },
  
  // 本地化 array
  items: {
    en: [
      { id: "1", name: "English Product A", price: 100 },
      { id: "2", name: "English Product B", price: 200 }
    ],
    es: [
      { id: "3", name: "Spanish Producto A", price: 100 }
    ]
  },
  
  // 时间戳
  createdAt: ISODate("2024-01-01T00:00:00Z"),
  updatedAt: ISODate("2024-01-15T12:00:00Z")
}
```

### 6.2 SQL 存储结构（PostgreSQL 示例）

```sql
-- 主表
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  description TEXT,
  view_count INTEGER,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- locales 表
CREATE TABLE posts_locales (
  id SERIAL PRIMARY KEY,
  _parent_id INTEGER NOT NULL REFERENCES posts(id),
  _locale VARCHAR(10) NOT NULL,
  title VARCHAR(255),
  localized_checkbox BOOLEAN,
  UNIQUE(_parent_id, _locale)
);

-- group 的 locales 表（如果 group 是本地化的）
-- 或者 group 的本地化字段也在 posts_locales 中

-- array 子表
CREATE TABLE posts_items (
  id SERIAL PRIMARY KEY,
  _parent_id INTEGER NOT NULL REFERENCES posts(id),
  _locale VARCHAR(10),  -- 本地化 array 特有
  _order INTEGER NOT NULL,
  name VARCHAR(255),
  price INTEGER
);

-- 数据
INSERT INTO posts (id, description, view_count, created_at, updated_at) VALUES
  (1, 'This is a non-localized field...', 100, '2024-01-01', '2024-01-15');

INSERT INTO posts_locales (_parent_id, _locale, title, localized_checkbox) VALUES
  (1, 'en', 'Welcome to Our Website', true),
  (1, 'es', 'Bienvenido a Nuestro Sitio Web', false),
  (1, 'de', 'Willkommen auf unserer Website', true);

INSERT INTO posts_items (_parent_id, _locale, _order, name, price) VALUES
  (1, 'en', 1, 'English Product A', 100),
  (1, 'en', 2, 'English Product B', 200),
  (1, 'es', 1, 'Spanish Producto A', 100);
```

### 6.3 API 返回示例

#### 查询 `GET /api/posts/1?locale=pt&fallbackLocale=es,en`

```json
{
  "id": "1",
  "title": "Bienvenido a Nuestro Sitio Web",
  "description": "This is a non-localized field...",
  "localizedCheckbox": false,
  "meta": {
    "seoTitle": "Spanish SEO Title",
    "seoDescription": "Spanish SEO Description"
  },
  "items": [
    { "id": "3", "name": "Spanish Producto A", "price": 100 }
  ],
  "viewCount": 100,
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-15T12:00:00.000Z"
}
```

**注意**：
- `title` 是西班牙语版本（因为 pt 无值，fallback 到 es）
- `items` 也是西班牙语版本（只有一条）
- 所有字段都是扁平结构，不是对象

#### 查询 `GET /api/posts/1?locale=all`

```json
{
  "id": "1",
  "title": {
    "en": "Welcome to Our Website",
    "es": "Bienvenido a Nuestro Sitio Web",
    "de": "Willkommen auf unserer Website"
  },
  "description": "This is a non-localized field...",
  "localizedCheckbox": {
    "en": true,
    "es": false,
    "de": true
  },
  "meta": {
    "en": {
      "seoTitle": "English SEO Title",
      "seoDescription": "English SEO Description"
    },
    "es": {
      "seoTitle": "Spanish SEO Title",
      "seoDescription": "Spanish SEO Description"
    }
  },
  "items": {
    "en": [
      { "id": "1", "name": "English Product A", "price": 100 },
      { "id": "2", "name": "English Product B", "price": 200 }
    ],
    "es": [
      { "id": "3", "name": "Spanish Producto A", "price": 100 }
    ]
  },
  "viewCount": 100,
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-15T12:00:00.000Z"
}
```

**注意**：
- 所有本地化字段都是对象，key 是语言代码
- pt 版本不显示（因为是 null 或不存在）
- **没有应用任何 fallback**！`locale=all` 返回原始数据

---

## 7. 关键代码位置索引

### 7.1 核心参数处理

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/payload/src/utilities/createPayloadRequest.ts` | HTTP 请求参数解析 | 66-93 |
| `packages/payload/src/utilities/addLocalesToRequest.ts` | locale/fallbackLocale sanitize | 63-88 |
| `packages/payload/src/utilities/sanitizeFallbackLocale.ts` | Fallback 规则解析 | 18-51 |
| `packages/payload/src/utilities/parseParams/index.ts` | URL 参数类型转换 | 18-65 |

### 7.2 查询操作

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/payload/src/collections/operations/find.ts` | 集合查询操作 | 105, 205-217, 318-338 |
| `packages/payload/src/collections/operations/findByID.ts` | 按 ID 查询 | 90, 143-150 |
| `packages/payload/src/database/getLocalizedPaths.ts` | MongoDB 路径转换 | 145-156 |

### 7.3 数据库适配器

#### MongoDB 适配器

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/db-mongodb/src/find.ts` | MongoDB 查询实现 | 59-65 |
| `packages/db-mongodb/src/queries/buildQuery.ts` | MongoDB 查询构建 | 22-30 |
| `packages/db-mongodb/src/queries/getLocalizedSortProperty.ts` | 本地化排序 | - |
| `packages/db-mongodb/src/utilities/transform.ts` | 数据读写转换 | 131-149, 244-249 |

#### Drizzle (SQL) 适配器

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/drizzle/src/upsertRow/index.ts` | 数据写入（含 locales 表） | - |
| `packages/drizzle/src/transform/read/index.ts` | 读取转换入口 | 23-80 |
| `packages/drizzle/src/transform/read/traverseFields.ts` | 读取字段遍历 | 127-268 |
| `packages/drizzle/src/transform/write/index.ts` | 写入转换入口 | 18-75 |
| `packages/drizzle/src/utilities/hasLocalesTable.ts` | 检测是否需要 locales 表 | - |

### 7.4 管理后台

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/ui/src/providers/Locale/index.tsx` | 语言上下文管理 | 54-105 |
| `packages/ui/src/providers/Translation/index.tsx` | 翻译功能 | - |
| `packages/ui/src/hooks/usePayloadAPI.ts` | API 请求 Hook | - |
| `packages/ui/src/utilities/getDisplayedFieldValue.ts` | 字段显示值获取 | - |

### 7.5 数据合并与过滤

| 文件路径 | 职责 | 关键行 |
|---------|------|--------|
| `packages/payload/src/utilities/mergeLocalizedData.ts` | 更新时合并多语言数据 | - |
| `packages/payload/src/utilities/filterDataToSelectedLocales.ts` | 按语言过滤数据 | 42-274 |
| `packages/payload/src/fields/hooks/afterRead/index.ts` | afterRead 阶段处理 | - |

---

## 8. 常见误区澄清

### 误区 1：`fallbackLocale` 在查询时使用

**实际情况**：`fallbackLocale` 只在 `afterRead` 阶段使用。

- `locale` 决定**从哪里查询**（路径转换或表过滤）
- `fallbackLocale` 决定**查询后如何合并缺失值**

这意味着：
- 查询 `locale='pt'` 时，数据库只查询葡萄牙语版本
- 如果葡萄牙语版本不存在，返回空或 null
- `fallbackLocale` 只在 `afterRead` 阶段尝试用其他语言的值填充

### 误区 2：MongoDB 和 SQL 适配器的存储方式相似

**实际情况**：两种适配器的存储模型完全不同。

| MongoDB | SQL (Drizzle) |
|---------|----------------|
| 嵌入式对象 `{ en: "...", es: "..." }` | 独立 `_locales` 表 |
| 单文档包含所有语言 | 每个语言一条记录 |
| 查询时路径转换 | 查询时 JOIN |
| 写入时单文档更新 | 写入时主表 + locales 表分离 |

### 误区 3：请求级 `fallbackLocale` 补充配置的 fallback

**实际情况**：请求级 `fallbackLocale` **完全覆盖**配置的 fallback。

```typescript
// 配置: pt 的 fallbackLocale 是 'es'
// 请求: ?locale=pt&fallbackLocale=fr

// 实际 fallback 链: pt → fr
// 配置中的 'es' 被完全忽略！
```

### 误区 4：`locale=all` 时也应用 fallback

**实际情况**：`locale=all` 返回原始数据，**不应用任何 fallback**。

```javascript
// locale=all 的返回
{
  title: {
    en: "English",
    es: "Spanish"
    // pt 版本缺失，不会被填充
  }
}
```

---

*报告生成日期: 2026-05-05*
*基于 PayloadCMS 代码库深度分析*
*修正了之前报告中的不准确之处*
