# PayloadCMS 字段配置多层面驱动机制分析报告

## 概述

PayloadCMS 采用单一字段配置驱动三个不同层面的架构设计：管理后台 React 组件、REST/GraphQL API Schema 和数据库 Schema。这种设计实现了"一次配置，多处使用"的 DRY 原则，极大地简化了开发流程。

**核心设计原则：** 所有层面都从 `SanitizedConfig` 分叉，配置清理是唯一的单点入口。

---

## 1. 配置生命周期与分叉边界

### 1.1 完整配置流程图

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                    用户字段配置 (Field Config)               │
                    │  { name: 'title', type: 'text', required: true, defaultValue: '...' } │
                    └─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                    ┌─────────────────────────────────────────────────────────────┐
                    │              配置构建 (buildConfig) - 启动时                  │
                    │  packages/payload/src/config/build.ts                         │
                    │  - 插件处理 → sanitizeConfig → SanitizedConfig              │
                    └─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                    ┌─────────────────────────────────────────────────────────────┐
                    │              配置清理 (sanitizeConfig) - 启动时              │
                    │  packages/payload/src/fields/config/sanitize.ts              │
                    │  - 类型验证、默认值设置、嵌套字段递归处理、钩子初始化          │
                    │  - 输出: SanitizedConfig (单一数据源)                         │
                    └─────────────────────────────────────────────────────────────┘
                                              │
            ┌─────────────────────────────────┼─────────────────────────────────┐
            │                                 │                                 │
            ▼                                 ▼                                 ▼
┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│  启动时同步处理        │    │  按需处理             │    │  开发时/构建时         │
│                       │    │                       │    │                       │
│ • GraphQL Schema     │    │ • 管理后台 ClientConfig │    │ • 类型生成 (payload-  │
│ • 数据库 Schema      │    │ • REST API 运行时      │    │   types.ts)           │
└───────────────────────┘    └───────────────────────┘    └───────────────────────┘
            │                                 │                                 │
            ▼                                 ▼                                 ▼
┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│  GraphQL:             │    │  管理后台:            │    │  类型生成:            │
│  fieldToSchemaMap     │    │  createClientConfig() │    │  configToJSONSchema() │
│  构建 GraphQL Schema  │    │  createClientFields() │    │  json-schema-to-ts    │
│  (服务端内存中)        │    │  移除服务端专有属性   │    │  (仅用于生成 TS 类型) │
└───────────────────────┘    └───────────────────────┘    └───────────────────────┘
            │                                 │
            ▼                                 ▼
┌───────────────────────┐    ┌───────────────────────┐
│  数据库:              │    │  REST API 运行时:      │
│  buildTable()         │    │  直接使用 SanitizedConfig │
│  traverseFields()     │    │  字段钩子处理 (before-  │
│  创建 Drizzle ORM 表  │    │  Validate, beforeChange) │
└───────────────────────┘    └───────────────────────┘
```

### 1.2 分叉边界关键节点

| 层面 | 触发时机 | 入口函数 | 输入 | 输出 |
|------|---------|---------|------|------|
| **GraphQL Schema** | 启动时 | `packages/graphql/src/schema/buildSchema.ts` | `SanitizedConfig` | `GraphQLSchema` |
| **数据库 Schema** | 启动时 | `packages/drizzle/src/schema/build.ts` | `SanitizedConfig` | `RawTable` (Drizzle 表定义) |
| **管理后台 ClientConfig** | 请求时 | `packages/payload/src/config/client.ts` | `SanitizedConfig` | `ClientConfig` (发送到浏览器) |
| **类型生成** | 开发时 | `packages/payload/src/bin/generateTypes.ts` | `SanitizedConfig` | `payload-types.ts` |
| **REST API 运行时** | 请求时 | `packages/payload/src/collections/operations/create.ts` 等 | `SanitizedConfig` | 数据处理结果 |

**关键洞察：**
- `SanitizedConfig` 是所有层面的**单一数据源**
- 配置清理（sanitize）只在**启动时执行一次**
- 各层从 `SanitizedConfig` 独立分叉，互不干扰

---

## 2. 字段配置核心结构

### 2.1 类型定义体系

字段配置定义在 `packages/payload/src/fields/config/types.ts` 中，采用双层类型设计：

**服务端完整类型**（如 `TextField`）：
```typescript
export type TextField = {
  admin?: {
    autoComplete?: string
    components?: {
      afterInput?: CustomComponent[]
      beforeInput?: CustomComponent[]
      Error?: CustomComponent<...>
      Label?: CustomComponent<...>
    } & FieldAdmin['components']
    placeholder?: Record<string, string> | string
    rtl?: boolean
  } & FieldAdmin
  maxLength?: number
  minLength?: number
  type: 'text'
} & (
  | { hasMany: true; maxRows?: number; minRows?: number; validate?: TextFieldManyValidation }
  | { hasMany?: false; validate?: TextFieldSingleValidation }
) & Omit<FieldBase, 'validate'>
```

**客户端精简类型**（如 `TextFieldClient`）：
```typescript
export type TextFieldClient = {
  admin?: AdminClient & Pick<TextField['admin'], 'autoComplete' | 'placeholder' | 'rtl'>
} & FieldBaseClient &
  Pick<TextField, 'hasMany' | 'maxLength' | 'maxRows' | 'minLength' | 'minRows' | 'type'>
```

### 2.2 基础字段接口 `FieldBase`

所有字段类型都继承自 `FieldBase`，包含通用配置：

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 字段名称，必须唯一 |
| `label` | `false \| LabelFunction \| StaticLabel` | 字段标签 |
| `required` | `boolean` | 是否必填 |
| `unique` | `boolean` | 是否唯一 |
| `index` | `boolean` | 是否创建数据库索引 |
| `localized` | `boolean` | 是否支持多语言 |
| `hidden` | `boolean` | 是否隐藏 |
| `defaultValue` | `DefaultValue` | 默认值 **（服务端专有）** |
| `admin` | `FieldAdmin` | 管理后台配置 |
| `access` | `{ create?, read?, update? }` | 访问控制 **（服务端专有）** |
| `hooks` | `{ beforeChange?, afterChange?, beforeValidate?, ... }` | 生命周期钩子 **（服务端专有）** |
| `validate` | `Validate` | 验证函数 **（服务端专有）** |
| `virtual` | `boolean \| string` | 虚拟字段标记 |

### 2.3 字段清理 (Sanitization)

字段配置在使用前会经过 `sanitize.ts` 中的 `sanitizeField` 函数处理：

**主要处理步骤：**
1. **类型检查**：确保 `field.type` 存在
2. **保留字段名检查**：防止使用系统保留字段名（如 `__v`, `salt`, `hash`, `file`）
3. **自动标签生成**：根据 `name` 自动生成 `label`
4. **默认值设置**：如 Checkbox 字段在 `required=true` 时默认值设为 `false`
5. **验证函数绑定**：根据字段类型绑定默认验证函数
6. **钩子和访问控制初始化**：确保 `hooks` 和 `access` 对象存在
7. **嵌套字段递归处理**：Array、Group、Blocks、Tabs 等字段的子字段递归清理
8. **时区字段插入**：Date 字段启用 `timezone` 时自动插入时区选择字段

---

## 3. 管理后台 React 组件驱动

### 3.1 服务端专有属性移除清单

**关键发现：** `defaultValue` 属于 `serverOnlyFieldProperties`，在 `createClientField` 时会被移除！

查看 `packages/payload/src/fields/config/client.ts:53-65`：

```typescript
const serverOnlyFieldProperties: Partial<ServerOnlyFieldProperties>[] = [
  'hooks',
  'access',
  'validate',
  'defaultValue',  // ← 被移除！
  'filterOptions',
  'editor',
  'custom',
  'typescriptSchema',
  'dbName',
  'enumName',
  'graphQL',
]
```

**admin 中被移除的属性：**
```typescript
const serverOnlyFieldAdminProperties: Partial<ServerOnlyFieldAdminProperties>[] = [
  'condition',   // ← 条件显示逻辑在服务端处理
  'components',  // ← 组件在服务端渲染后传递
]
```

### 3.2 管理后台数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           服务端                                               │
│  ┌──────────────┐    ┌─────────────────────┐    ┌─────────────────────────┐ │
│  │ Sanitized-   │───▶│ createClientConfig()│───▶│ ClientConfig            │ │
│  │ Config       │    │ createClientFields()│    │ (无 defaultValue)       │ │
│  └──────────────┘    └─────────────────────┘    └─────────────────────────┘ │
│         │                                                         │            │
│         ▼                                                         ▼            │
│  ┌──────────────────────────────────────────┐      ┌──────────────────────┐ │
│  │ buildFormState() - 表单状态构建           │      │ 发送到客户端          │ │
│  │ packages/payload/src/admin/forms/Form.ts │      │                      │ │
│  │                                          │      │                      │ │
│  │ • 调用 getFallbackValue()                │      │                      │ │
│  │ • 计算 defaultValue → initialValue       │      │                      │ │
│  │ • 生成 FieldState 包含 initialValue      │      │                      │ │
│  └──────────────────────────────────────────┘      └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           客户端 (浏览器)                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ FormState 结构:                                                          │ │
│  │ {                                                                        │ │
│  │   'title': {                                                            │ │
│  │     initialValue: '默认值',  // ← 从服务端接收，不是从配置读取           │ │
│  │     value: '用户输入',                                                 │ │
│  │     valid: true,                                                       │ │
│  │     // 注意：没有 defaultValue 字段！                                   │ │
│  │   }                                                                     │ │
│  │ }                                                                        │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 默认值承载边界分析

| 位置 | defaultValue 状态 | 说明 |
|------|------------------|------|
| **用户配置** | ✅ 存在 | 用户定义的 `defaultValue: '...'` |
| **SanitizedConfig** | ✅ 存在 | 清理后仍保留 |
| **ClientConfig** | ❌ 被移除 | `serverOnlyFieldProperties` 列表中 |
| **客户端 FormState** | ❌ 不存在 | 客户端从未直接访问 `defaultValue` |
| **FormState.initialValue** | ✅ 存在 | **服务端计算后传递** |

**关键代码验证：**

`packages/payload/src/fields/hooks/beforeValidate/getFallbackValue.ts:20-27`：
```typescript
} else if ('defaultValue' in field && typeof field.defaultValue !== 'undefined') {
  fallbackValue = await getDefaultValue({
    defaultValue: field.defaultValue,
    locale: req.locale || '',
    req,
    user: req.user,
  })
}
```

`packages/payload/src/fields/getDefaultValue.ts:24-26`：
```typescript
if (defaultValue && typeof defaultValue === 'function') {
  return await defaultValue({ locale, req, user })  // 函数类型在服务端执行
}
```

**结论：**
- `defaultValue` 完全是**服务端概念**
- 管理后台的默认值通过 `initialValue` 传递，这是在**服务端计算**的
- 客户端组件不直接访问 `defaultValue`，只使用 `initialValue`
- 函数类型的 `defaultValue` 只能在服务端执行（需要 `req`、`user` 等上下文）

### 3.4 管理后台组件映射

管理后台根据字段的 `type` 属性渲染对应的 React 组件：

| 字段类型 | 管理后台组件路径 | 组件功能 |
|----------|-----------------|----------|
| `text` | `admin/fields/Text.ts` | 文本输入框 |
| `number` | `admin/fields/Number.ts` | 数字输入框 |
| `email` | `admin/fields/Email.ts` | 邮箱输入框 |
| `textarea` | `admin/fields/Textarea.ts` | 多行文本框 |
| `checkbox` | `admin/fields/Checkbox.ts` | 复选框 |
| `date` | `admin/fields/Date.ts` | 日期选择器 |
| `select` | `admin/fields/Select.ts` | 下拉选择器 |
| `radio` | `admin/fields/Radio.ts` | 单选按钮组 |
| `relationship` | `admin/fields/Relationship.ts` | 关系选择器 |
| `upload` | `admin/fields/Upload.ts` | 文件上传 |
| `richText` | `admin/fields/RichText.ts` | 富文本编辑器 |
| `code` | `admin/fields/Code.ts` | 代码编辑器 |
| `json` | `admin/fields/JSON.ts` | JSON 编辑器 |
| `array` | `admin/fields/Array.ts` | 数组字段 |
| `blocks` | `admin/fields/Blocks.ts` | 块字段 |
| `group` | `admin/fields/Group.ts` | 分组字段 |
| `tabs` | `admin/fields/Tabs.ts` | 标签页字段 |
| `row` | `admin/fields/Row.ts` | 行布局字段 |
| `collapsible` | `admin/fields/Collapsible.ts` | 可折叠字段 |
| `ui` | `admin/fields/UI.ts` | 纯 UI 组件 |
| `join` | `admin/fields/Join.ts` | 关联查询字段 |
| `point` | `admin/fields/Point.ts` | 地理坐标字段 |
| `hidden` | `admin/fields/Hidden.ts` | 隐藏字段 |

---

## 4. API Schema 驱动机制

### 4.1 REST API 分层：类型生成 vs 运行时处理

**重要澄清：** REST API 有两个完全独立的层面，之前的报告将它们混为一谈了。

#### 层面 A：类型生成（开发时/构建时）

**用途：** 生成 TypeScript 类型定义 `payload-types.ts`

**触发时机：** 运行 `payload generate:types` 命令

**数据流：**
```
SanitizedConfig
    │
    ▼
configToJSONSchema()  ← packages/payload/src/utilities/configToJSONSchema.ts
    │
    ▼
JSON Schema
    │
    ▼
json-schema-to-typescript  ← 第三方库
    │
    ▼
payload-types.ts
```

**关键代码：** `packages/payload/src/bin/generateTypes.ts:32`
```typescript
const jsonSchema = configToJSONSchema(config, config.db.defaultIDType, i18n)

let compiled = await compile(jsonSchema, 'Config', {
  // ...
})
```

**JSON Schema 的作用：**
- 仅用于生成 TypeScript 类型
- 不用于运行时数据验证
- 开发时/构建时执行，不影响运行时

#### 层面 B：运行时处理（请求时）

**用途：** 处理实际的 API 请求

**触发时机：** HTTP 请求到达时

**数据流：**
```
HTTP Request (POST /api/posts)
    │
    ▼
REST Handler (createHandler)  ← packages/payload/src/collections/endpoints/create.ts
    │
    ▼
createOperation()  ← packages/payload/src/collections/operations/create.ts
    │
    ├─▶ beforeValidate 钩子  ← 使用 SanitizedConfig 中的字段定义
    │       │
    │       ├─▶ getFallbackValue()  ← 处理 defaultValue
    │       └─▶ 字段级 validate 函数
    │
    ├─▶ beforeChange 钩子
    │
    ├─▶ 数据库操作 (payload.db.create)
    │
    └─▶ afterChange / afterRead 钩子
```

**关键代码：** `packages/payload/src/collections/operations/create.ts:181-207`
```typescript
data = await beforeValidate({
  collection: collectionConfig,
  context: req.context,
  data,
  doc: duplicatedFromDoc,
  global: null,
  operation: 'create',
  overrideAccess: overrideAccess!,
  req,
})

// 字段级 beforeValidate 钩子处理默认值
```

**REST 运行时验证方式：**
- 不使用 JSON Schema 验证
- 使用字段配置中的 `validate` 函数
- 使用 `beforeValidate`、`beforeChange` 等钩子
- 直接操作 `SanitizedConfig` 中的字段定义

#### REST API 两层对比表

| 维度 | 类型生成层 | 运行时处理层 |
|------|-----------|-------------|
| **触发时机** | `payload generate:types` | HTTP 请求 |
| **入口函数** | `configToJSONSchema()` | `createOperation()` 等 |
| **使用的配置** | `SanitizedConfig` | `SanitizedConfig` |
| **输出** | `payload-types.ts` | 处理后的数据 |
| **验证方式** | 无（仅类型生成） | `validate` 函数 + 钩子 |
| **defaultValue** | 不使用 | `getFallbackValue()` 处理 |
| **是否影响运行时** | 否 | 是 |

### 4.2 GraphQL Schema 生成

GraphQL Schema 在**启动时**从 `SanitizedConfig` 构建，与 REST 类型生成完全独立。

**核心映射表结构：** `packages/graphql/src/schema/fieldToSchemaMap.ts`
```typescript
type FieldToSchemaMap = {
  array: (args: { field: ArrayField } & SharedArgs) => ObjectTypeConfig
  blocks: (args: { field: BlocksField } & SharedArgs) => ObjectTypeConfig
  checkbox: (args: { field: CheckboxField } & SharedArgs) => ObjectTypeConfig
  code: (args: { field: CodeField } & SharedArgs) => ObjectTypeConfig
  // ... 所有字段类型
  text: (args: { field: TextField } & SharedArgs) => ObjectTypeConfig
}
```

**字段类型到 GraphQL 类型映射：**

| Payload 字段类型 | GraphQL 类型 | 特殊处理 |
|-----------------|-------------|----------|
| `text` | `GraphQLString` | `hasMany=true` 时为 `[String!]` |
| `number` | `GraphQLFloat` / `GraphQLInt` | ID 字段用 `Int`，其他用 `Float` |
| `email` | `EmailAddressResolver` (自定义标量) | 邮箱格式验证 |
| `textarea` | `GraphQLString` | 纯文本 |
| `checkbox` | `GraphQLBoolean` | 布尔值 |
| `date` | `DateTimeResolver` (自定义标量) | ISO 日期时间格式 |
| `select` | `GraphQLEnumType` | 动态生成枚举类型 |
| `radio` | `GraphQLEnumType` | 动态生成枚举类型 |
| `relationship` | 关联对象类型 | 支持 `hasMany` 和多态关系 |
| `upload` | 关联对象类型 | 类似 relationship，指向 media collection |
| `richText` | `GraphQLJSON` | JSON 格式存储 |
| `code` | `GraphQLString` | 纯文本代码 |
| `json` | `GraphQLJSON` | 自定义 JSON Schema 支持 |
| `array` | `[ArrayItemType!]` | 递归构建子对象类型 |
| `blocks` | `[BlockUnionType!]` | 生成 Union 类型 |
| `group` | `GroupObjectType` | 嵌套对象类型 |
| `tabs` | 展开或嵌套对象 | named tab 生成独立类型 |
| `join` | 自定义对象类型 | 包含 `docs`, `hasNextPage`, `totalDocs` |
| `point` | `[Float!]` | 经纬度坐标数组 |

### 4.3 JSON Schema 生成（仅用于类型生成）

**重要澄清：** JSON Schema 仅用于生成 TypeScript 类型，不用于运行时验证。

`packages/payload/src/utilities/configToJSONSchema.ts:1258-1259` 注释明确说明：
```typescript
/**
 * This is used for generating the TypeScript types (payload-types.ts) with the payload generate:types command.
 */
export function configToJSONSchema(...)
```

**字段类型到 JSON Schema 映射：**

| Payload 字段类型 | JSON Schema 类型 | 特殊处理 |
|-----------------|-----------------|----------|
| `text` | `string` 或 `array` | `hasMany=true` 时为 `{ type: 'array', items: { type: 'string' } }` |
| `number` | `number` 或 `array` | `hasMany=true` 时为数组 |
| `email` | `string` | 纯字符串 |
| `textarea` | `string` | 纯字符串 |
| `checkbox` | `boolean` | 布尔值 |
| `date` | `string` | ISO 日期时间字符串 |
| `select` | `string` + `enum` 或 `array` | `hasMany=true` 时为字符串数组 |
| `radio` | `string` + `enum` | 单选枚举 |
| `relationship` | 复杂类型 | 支持 ID 字符串或完整对象引用 |
| `upload` | 复杂类型 | 类似 relationship |
| `richText` | `array` 或自定义 | 默认为对象数组 |
| `code` | `string` | 纯字符串 |
| `json` | 多种类型 | `['object', 'array', 'string', 'number', 'boolean', 'null']` |
| `array` | `array` | `items` 为子对象 schema |
| `blocks` | `array` | `items` 使用 `oneOf` 引用各 block 类型 |
| `group` | `object` | `properties` 包含子字段 |
| `tabs` | 展开或 `object` | named tab 生成独立类型 |
| `join` | `object` | 包含 `docs`, `hasNextPage`, `totalDocs` |
| `point` | `array` | `[number, number]`，minItems=2, maxItems=2 |

---

## 5. 数据库 Schema 驱动机制

### 5.1 数据库表构建流程

数据库 Schema 构建从 `packages/drizzle/src/schema/build.ts` 中的 `buildTable` 函数开始，在**启动时**执行：

```typescript
export const buildTable = ({
  adapter,
  fields,
  tableName,
  // ...
}: Args): Result => {
  // 1. 设置 ID 列类型
  const idColType: IDType = setColumnID({ adapter, columns, fields })

  // 2. 遍历字段构建列
  const {
    hasLocalizedField,
    hasManyNumberField,
    hasManyTextField,
    hasLocalizedRelationshipField,
  } = traverseFields({ ... })

  // 3. 添加时间戳列
  if (timestamps) {
    columns.createdAt = { name: 'created_at', type: 'timestamp', ... }
    columns.updatedAt = { name: 'updated_at', type: 'timestamp', ... }
  }

  // 4. 构建主表
  const table: RawTable = {
    name: tableName,
    columns,
    foreignKeys: baseForeignKeys,
    indexes,
  }
  adapter.rawTables[tableName] = table

  // 5. 处理本地化表
  if (hasLocalizedField || localizedRelations.size) {
    // 创建 locales 表
    const localeTableName = `${tableName}${adapter.localesSuffix}`
  }

  // 6. 处理 hasMany 文本/数字的关联表
  if (hasManyTextField) {
    // 创建 {tableName}_texts 表
  }
  if (hasManyNumberField) {
    // 创建 {tableName}_numbers 表
  }

  // 7. 处理关系表
  if (relationships.size) {
    // 创建 {tableName}_rels 表
  }

  return { ... }
}
```

### 5.2 字段遍历与列构建

`traverseFields` 函数负责将字段配置转换为数据库列定义：

```typescript
export const traverseFields = ({
  adapter,
  columns,
  fields,
  parentIsLocalized,
  // ...
}: Args): Result => {
  fields.forEach((field) => {
    // 跳过虚拟字段和 id 字段
    if (fieldIsVirtual(field) || field.name === 'id') {
      return
    }

    // 计算列名（转换为 snake_case）
    const columnName = `${columnPrefix || ''}${field.name[0] === '_' ? '_' : ''}${toSnakeCase(field.name)}`

    // 判断是否本地化字段
    const isFieldLocalized = fieldShouldBeLocalized({ field, parentIsLocalized })

    // 本地化字段放到 locales 表
    let targetTable = columns
    if (isFieldLocalized && /* 条件判断 */) {
      hasLocalizedField = true
      targetTable = localesColumns
    }

    // 添加索引
    if (field.unique || field.index || ['relationship', 'upload'].includes(field.type)) {
      targetIndexes[indexName] = {
        name: indexName,
        on: isFieldLocalized ? [fieldName, '_locale'] : fieldName,
        unique,
      }
    }

    // 根据字段类型创建列
    switch (field.type) {
      case 'text':
        if (field.hasMany) {
          // hasMany 文本创建单独的关联表
        } else {
          targetTable[fieldName] = withDefault(
            { name: columnName, type: 'varchar' },
            field,
          )
        }
        break
      // ... 其他字段类型处理
    }

    // 处理 NOT NULL 约束
    if (
      !disableNotNull &&
      targetTable[fieldName] &&
      'required' in field &&
      field.required &&
      !condition
    ) {
      targetTable[fieldName].notNull = true
    }
  })

  return { ... }
}
```

### 5.3 默认值在数据库层的处理

**与管理后台不同的边界：**

| 层面 | 静态 defaultValue | 函数类型 defaultValue |
|------|------------------|----------------------|
| **管理后台** | 服务端计算为 `initialValue` | 服务端执行，结果为 `initialValue` |
| **数据库 Schema** | 设为列的 `DEFAULT` 约束 | ❌ **不设置数据库默认值** |
| **REST/GraphQL 运行时** | `getFallbackValue` 处理 | `getFallbackValue` 中执行 |

**关键代码：** `packages/drizzle/src/schema/withDefault.ts`
```typescript
export const withDefault = (
  column: RawColumn,
  field: Field,
): RawColumn => {
  if (!('defaultValue' in field) || field.defaultValue === undefined) {
    return column
  }

  const defaultValue = field.defaultValue

  // 处理不同类型的默认值
  if (typeof defaultValue === 'string') {
    column.default = `'${defaultValue}'`
  } else if (typeof defaultValue === 'number') {
    column.default = defaultValue.toString()
  } else if (typeof defaultValue === 'boolean') {
    column.default = defaultValue.toString()
  } else if (defaultValue === null) {
    column.default = 'null'
  }
  // 函数类型的 defaultValue 在运行时计算，不设置数据库默认值 ← 重要！

  return column
}
```

**边界说明：**
- 静态 `defaultValue`（字符串、数字、布尔、null）会设置为数据库 `DEFAULT` 约束
- 函数类型 `defaultValue` **不会**设置数据库默认值，只在运行时通过 `getFallbackValue` 执行

### 5.4 字段类型到数据库列映射

| Payload 字段类型 | 数据库列类型 | 特殊处理 |
|-----------------|-------------|----------|
| `text` | `varchar` | `hasMany=true` 创建 `{table}_texts` 关联表 |
| `number` | `numeric` | `hasMany=true` 创建 `{table}_numbers` 关联表 |
| `email` | `varchar` | 纯文本 |
| `textarea` | `varchar` | 纯文本 |
| `checkbox` | `boolean` | 布尔值 |
| `date` | `timestamp` | `precision: 3`, `withTimezone: true` |
| `select` | `enum` 或关联表 | `hasMany=true` 创建单独关联表 |
| `radio` | `enum` | 数据库枚举类型 |
| `relationship` (简单) | 外键列 | `{fieldName}_id` + 外键约束 |
| `relationship` (hasMany/多态) | 关系表 | 创建 `{table}_rels` 关联表 |
| `upload` | 同 relationship | 指向 media collection |
| `richText` | `jsonb` (PostgreSQL) | JSON 格式存储 |
| `code` | `varchar` | 纯文本 |
| `json` | `jsonb` | JSON 格式存储 |
| `array` | 独立关联表 | 创建 `{parentTable}_{arrayField}` 表 |
| `blocks` | 独立关联表 或 `jsonb` | `blocksAsJSON=true` 时存为 JSONB |
| `group` (named) | 列前缀展开 | `{groupName}_{fieldName}` |
| `group` (unnamed) | 直接展开 | 子字段直接作为列 |
| `tabs` (named) | 同 named group | 列前缀展开 |
| `tabs` (unnamed) | 直接展开 | 子字段直接作为列 |
| `join` | 无列 | 虚拟关联，不存储数据 |
| `point` | `geometry` (PostgreSQL) | 地理空间类型 |
| `hidden` | 同对应类型 | 仅 UI 隐藏，数据库正常存储 |

### 5.5 关联表结构

**Texts 表（hasMany 文本）：**
| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | `serial` | 主键 |
| `order` | `integer` | 排序 |
| `parent_id` | `IDType` | 外键，关联主表 |
| `path` | `varchar` | 字段路径 |
| `text` | `varchar` | 文本值 |
| `locale` | `enum` | 可选，本地化标识 |

**Numbers 表（hasMany 数字）：**
| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | `serial` | 主键 |
| `order` | `integer` | 排序 |
| `parent_id` | `IDType` | 外键，关联主表 |
| `path` | `varchar` | 字段路径 |
| `number` | `numeric` | 数字值 |
| `locale` | `enum` | 可选，本地化标识 |

**Rels 表（关系）：**
| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | `serial` | 主键 |
| `order` | `integer` | 排序 |
| `parent_id` | `IDType` | 外键，关联主表 |
| `path` | `varchar` | 字段路径 |
| `{relationTo}ID` | `IDType` | 各关联 collection 的外键 |
| `locale` | `enum` | 可选，本地化标识 |

**Locales 表（本地化）：**
| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | `serial` | 主键 |
| `_locale` | `enum` | 语言标识 |
| `_parent_id` | `IDType` | 外键，关联主表 |
| `{localizedField}` | 各类型 | 本地化字段值 |

---

## 6. 完整数据流与边界汇总

### 6.1 配置生命周期全景图

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                    用户字段配置                                │
                    │  {                                                        │
                    │    name: 'status',                                       │
                    │    type: 'select',                                       │
                    │    options: ['draft', 'published'],                     │
                    │    required: true,                                       │
                    │    defaultValue: 'draft',  ← 静态或函数                  │
                    │    validate: (value) => { ... },  ← 服务端专有           │
                    │    hooks: { beforeChange: ... },  ← 服务端专有           │
                    │    admin: { condition: ... }  ← 服务端处理               │
                    │  }                                                         │
                    └─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                    ┌─────────────────────────────────────────────────────────────┐
                    │              配置构建 (buildConfig) - 启动时                  │
                    │  - 插件处理                                                  │
                    │  - sanitizeConfig 清理配置                                   │
                    │  - 输出: SanitizedConfig (单一数据源)                        │
                    └─────────────────────────────────────────────────────────────┘
                                              │
            ┌─────────────────────────────────┼─────────────────────────────────┐
            │                                 │                                 │
            ▼                                 ▼                                 ▼
┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│  启动时同步构建        │    │  请求时按需转换       │    │  开发时按需执行         │
│                       │    │                       │    │                       │
│ • GraphQL Schema     │    │ • 管理后台 ClientConfig│    │ • 类型生成            │
│ • 数据库 Table Schema│    │ • REST API 运行时      │    │                       │
└───────────────────────┘    └───────────────────────┘    └───────────────────────┘
```

### 6.2 各层配置属性使用对比

| 配置属性 | 管理后台 ClientConfig | GraphQL Schema | REST 运行时 | 数据库 Schema | 类型生成 (JSON Schema) |
|---------|----------------------|---------------|------------|--------------|------------------------|
| `name` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `type` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `label` | ✅ (函数执行后) | ❌ | ❌ | ❌ | ✅ (用于 JSDoc) |
| `required` | ✅ | ✅ (NonNull) | ✅ (验证) | ✅ (NOT NULL) | ✅ |
| `unique` | ✅ (提示) | ❌ | ❌ | ✅ (UNIQUE) | ❌ |
| `index` | ❌ | ❌ | ❌ | ✅ (索引) | ❌ |
| `localized` | ✅ | ✅ | ✅ | ✅ (分离表) | ✅ |
| `hidden` | ✅ | ❌ | ❌ | ✅ (正常存储) | ❌ |
| `defaultValue` | ❌ **(移除)** | ❌ | ✅ **(getFallbackValue)** | ✅ **(静态值设 DEFAULT)** | ❌ |
| `admin.*` (非 components/condition) | ✅ | ❌ | ❌ | ❌ | ✅ (description) |
| `admin.components` | ❌ **(服务端渲染)** | ❌ | ❌ | ❌ | ❌ |
| `admin.condition` | ❌ **(服务端处理)** | ❌ | ❌ | ❌ | ❌ |
| `access.*` | ❌ **(移除)** | ❌ | ✅ | ❌ | ❌ |
| `hooks.*` | ❌ **(移除)** | ❌ | ✅ | ❌ | ❌ |
| `validate` | ❌ **(移除)** | ❌ | ✅ | ❌ | ❌ |
| `virtual` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `hasMany` | ✅ | ✅ | ✅ | ✅ (关联表) | ✅ |
| `options` | ✅ | ✅ (枚举) | ✅ | ✅ (枚举/关联表) | ✅ (enum) |
| `relationTo` | ✅ | ✅ (关联) | ✅ | ✅ (外键/关系表) | ✅ |
| `dbName` | ❌ **(移除)** | ❌ | ❌ | ✅ | ❌ |
| `enumName` | ❌ **(移除)** | ❌ | ❌ | ✅ | ❌ |
| `graphQL.*` | ❌ **(移除)** | ✅ | ❌ | ❌ | ❌ |
| `typescriptSchema` | ❌ **(移除)** | ❌ | ❌ | ❌ | ✅ |

### 6.3 defaultValue 承载边界详解

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                    defaultValue 边界                          │
                    └─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  SanitizedConfig (服务端内存)                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │  {                                                                           ││
│  │    name: 'status',                                                          ││
│  │    defaultValue: 'draft',  ← 静态值                                        ││
│  │    // 或                                                                     ││
│  │    defaultValue: ({ req, user }) => req.locale === 'zh' ? '草稿' : 'draft'││
│  │  }                                                                           ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────┘
           │                                    │                                    │
           ▼                                    ▼                                    ▼
┌───────────────────────┐        ┌───────────────────────┐        ┌───────────────────────┐
│  管理后台              │        │  REST/GraphQL 运行时   │        │  数据库 Schema         │
│                       │        │                       │        │                       │
│ createClientField()   │        │ getFallbackValue()    │        │ withDefault()         │
│ 移除 defaultValue     │        │                       │        │                       │
│                       │        │ • 静态值: 直接使用     │        │ • 静态值: 列 DEFAULT  │
│ 结果: ClientField     │        │ • 函数: 执行获取结果   │        │ • 函数: ❌ 不设置      │
│ 无 defaultValue       │        │                       │        │                       │
│                       │        │ beforeValidate 钩子    │        │                       │
│                       │        │ 中应用到 data          │        │                       │
└───────────────────────┘        └───────────────────────┘        └───────────────────────┘
           │                                    │
           ▼                                    ▼
┌───────────────────────┐        ┌───────────────────────┐
│  buildFormState()     │        │  数据库默认值行为       │
│                       │        │                       │
│ 服务端计算 initialValue│        │ • INSERT 时未提供值   │
│                       │        │   使用数据库 DEFAULT   │
│ 结果: FieldState      │        │                       │
│ {                     │        │ • 但函数类型           │
│   initialValue: '...' │        │   defaultValue 不会    │
│ }                     │        │   设置数据库默认值      │
└───────────────────────┘        │   必须依赖运行时处理    │
                                 └───────────────────────┘
```

### 6.4 关键代码位置汇总

| 功能模块 | 文件路径 | 关键函数/类型 |
|---------|---------|--------------|
| **配置构建** | `packages/payload/src/config/build.ts` | `buildConfig()` |
| **配置清理** | `packages/payload/src/fields/config/sanitize.ts` | `sanitizeField()`, `sanitizeFields()` |
| **客户端配置转换** | `packages/payload/src/fields/config/client.ts` | `createClientField()`, `serverOnlyFieldProperties` |
| **表单状态构建** | `packages/payload/src/admin/forms/Form.ts` | `BuildFormStateArgs`, `FieldState` |
| **默认值获取** | `packages/payload/src/fields/getDefaultValue.ts` | `getDefaultValue()` |
| **默认值回退** | `packages/payload/src/fields/hooks/beforeValidate/getFallbackValue.ts` | `getFallbackValue()` |
| **GraphQL Schema** | `packages/graphql/src/schema/fieldToSchemaMap.ts` | `fieldToSchemaMap` |
| **JSON Schema (类型生成)** | `packages/payload/src/utilities/configToJSONSchema.ts` | `configToJSONSchema()`, `fieldsToJSONSchema()` |
| **类型生成** | `packages/payload/src/bin/generateTypes.ts` | `generateTypes()` |
| **REST 创建操作** | `packages/payload/src/collections/operations/create.ts` | `createOperation()` |
| **数据库表构建** | `packages/drizzle/src/schema/build.ts` | `buildTable()` |
| **字段到列映射** | `packages/drizzle/src/schema/traverseFields.ts` | `traverseFields()` |
| **数据库默认值处理** | `packages/drizzle/src/schema/withDefault.ts` | `withDefault()` |

---

## 7. 设计亮点与架构优势

### 7.1 单一数据源 (SSOT)

PayloadCMS 的字段配置采用单一数据源设计，所有层面的行为都由同一份配置驱动：

**优势：**
- 减少重复代码，避免配置不一致
- 修改一处配置，所有层面自动同步
- 降低维护成本，减少错误几率

### 7.2 清晰的分层边界

**服务端 vs 客户端分离：**
- 服务端保留完整配置（hooks、access、validate、defaultValue 等）
- 客户端精简配置（移除服务端执行逻辑）
- 通过 `serverOnlyFieldProperties` 明确界定移除的属性

**边界设计亮点：**
1. **defaultValue**：完全在服务端处理，客户端通过 `initialValue` 接收计算结果
2. **函数类型属性**（label、condition 等）：在服务端执行，结果序列化后发送到客户端
3. **组件**：在服务端渲染（RSC），客户端接收渲染结果

### 7.3 类型安全保障

**TypeScript 类型系统：**
- 每种字段类型有独立的 TypeScript 接口
- 服务端和客户端类型明确区分
- 编译时检查配置正确性

### 7.4 扩展性设计

**自定义能力：**
1. **自定义组件**：`admin.components.Field` 等
2. **自定义验证**：`validate` 函数
3. **自定义 Schema**：`typescriptSchema` 修饰器
4. **自定义数据库名**：`dbName`, `enumName`
5. **自定义 GraphQL 配置**：`graphQL.complexity`

---

## 8. 常见误区澄清

### 8.1 误区 1：REST API 使用 JSON Schema 验证

**真相：**
- JSON Schema 仅用于生成 TypeScript 类型（`payload-types.ts`）
- REST 运行时使用 `validate` 函数和钩子进行验证
- 两者是完全独立的层面

### 8.2 误区 2：管理后台直接使用 defaultValue

**真相：**
- `defaultValue` 属于 `serverOnlyFieldProperties`，在 `createClientField` 时被移除
- 管理后台通过 `initialValue` 获取默认值
- `initialValue` 是在服务端通过 `getFallbackValue` 计算的

### 8.3 误区 3：函数类型的 defaultValue 会设置数据库默认值

**真相：**
- `withDefault()` 只处理静态类型的 `defaultValue`
- 函数类型的 `defaultValue` 不会设置数据库 `DEFAULT` 约束
- 函数类型的 `defaultValue` 必须依赖运行时的 `getFallbackValue` 处理

### 8.4 误区 4：各层配置在不同时机清理

**真相：**
- 配置清理（sanitize）只在**启动时执行一次**
- 所有层面都从 `SanitizedConfig` 分叉
- `SanitizedConfig` 是唯一的单一数据源

---

## 9. 总结

PayloadCMS 的字段配置多层面驱动机制是其架构设计的核心亮点之一。通过精心设计的类型系统、清晰的分层转换逻辑和统一的配置模型，实现了：

### 核心边界原则

1. **单一数据源**：`SanitizedConfig` 是所有层面的起点
2. **启动时清理**：配置清理只执行一次，各层从清理结果分叉
3. **服务端专有属性明确移除**：`serverOnlyFieldProperties` 清单界定边界
4. **函数类型属性服务端执行**：客户端接收执行结果，不是函数本身
5. **defaultValue 完全在服务端处理**：
   - 管理后台：服务端计算为 `initialValue`
   - 运行时：`getFallbackValue` 应用到数据
   - 数据库：静态值设 `DEFAULT`，函数类型不设置

### 各层职责分离

| 层面 | 配置来源 | 主要职责 |
|------|---------|---------|
| **GraphQL Schema** | `SanitizedConfig` | 启动时构建类型定义 |
| **数据库 Schema** | `SanitizedConfig` | 启动时构建表/列定义 |
| **管理后台 ClientConfig** | `SanitizedConfig` → `createClientConfig()` | 请求时转换，移除服务端属性 |
| **REST 运行时** | `SanitizedConfig` 直接使用 | 请求时通过钩子处理数据 |
| **类型生成** | `SanitizedConfig` → `configToJSONSchema()` | 开发时生成 TypeScript 类型 |

这种设计模式不仅简化了开发流程，也为系统的可维护性和可扩展性奠定了坚实基础。开发者只需关注业务领域模型的定义，系统会自动处理各层面的适配工作。