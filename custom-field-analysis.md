# 自定义字段客户端编辑与服务端校验边界分析报告

## 1. 概述

本文档分析 Payload CMS 中自定义字段在**客户端编辑**和**服务端清洗校验**两侧的边界划分，以及两侧如何保持一致性。

## 2. 架构概览

Payload CMS 采用三层验证架构：

```
┌─────────────────────────────────────────────────────────────┐
│                     用户输入流程                            │
├─────────────────────────────────────────────────────────────┤
│  1. 客户端实时验证 (useField hook)                          │
│     - 节流验证 (150ms)                                       │
│     - event: 'onChange'                                     │
│     - 提供即时反馈，提升用户体验                              │
├─────────────────────────────────────────────────────────────┤
│  2. 服务端表单状态构建 (fieldSchemasToFormState)            │
│     - 在 RSC 服务器端执行                                    │
│     - 使用服务端完整的 validate 函数                         │
│     - event: 'onChange'                                     │
│     - 构建完整的 FormState，包含验证结果                      │
├─────────────────────────────────────────────────────────────┤
│  3. 提交时服务端验证 (beforeChange)                          │
│     - 最终数据清洗和类型转换                                  │
│     - 完整的业务规则验证                                      │
│     - event: 'submit'                                       │
│     - 数据持久化前的最后一道防线                              │
└─────────────────────────────────────────────────────────────┘
```

## 3. 客户端编辑边界

### 3.1 客户端字段配置 (ClientField)

客户端字段通过 `createClientFields` 函数从服务端字段转换而来，转换时会**剥离服务端专有属性**。

**服务端专有属性（被剥离）：**
- `hooks` - 字段钩子（beforeValidate, beforeChange 等）
- `access` - 访问控制
- `validate` - 验证函数
- `defaultValue` - 默认值
- `filterOptions` - 过滤选项函数
- `editor` - 富文本编辑器配置
- `custom` - 服务端自定义数据
- `typescriptSchema` - TypeScript 生成配置
- `dbName` / `enumName` - 数据库相关
- `graphQL` - GraphQL 配置
- `admin.components` - 服务端组件引用
- `admin.condition` - 条件函数

**客户端保留属性：**
- 基本属性：`name`, `type`, `required`, `localized`, `unique`, `index`
- 类型特定属性：`min`, `max`, `hasMany`, `minRows`, `maxRows`, `minLength`, `maxLength`, `options` 等
- UI 控制：`admin.hidden`, `admin.readOnly`, `admin.disabled`, `admin.width`, `admin.style`
- 标签：`label`（函数会被执行，结果被序列化）
- `admin.custom` - 客户端和服务端共享的自定义数据

### 3.2 客户端验证机制

**位置：** `packages/ui/src/forms/useField/index.tsx`

**特点：**
1. **节流验证**：使用 `useThrottledEffect`，150ms 延迟
2. **模拟请求对象**：构建简化的 `PayloadRequest`，包含 `config`, `t`, `user`
3. **event: 'onChange'**：标识为客户端变更验证
4. **有限上下文**：`blockData` 传递为 `undefined`（注释说明不传递是因为性能考虑）

**验证函数调用：**
```typescript
// 客户端调用
await validate(valueToValidate, {
  id,
  blockData: undefined, // 客户端不传递
  collectionSlug,
  data: documentForm?.getData ? documentForm.getData() : data,
  event: 'onChange',  // 重要：客户端标识
  operation,
  path: pathSegments,
  preferences: {} as any,
  req: {
    payload: { config },
    t,
    user,
  } as unknown as PayloadRequest,
  siblingData: getSiblingData(path),
})
```

### 3.3 客户端验证边界

**客户端负责：**
- 即时 UI 反馈（错误消息显示）
- 简单的格式验证（如果验证函数支持）
- 字段值的本地状态管理
- 表单提交前的初步检查

**客户端不负责：**
- 数据库查询验证
- 复杂的业务逻辑验证
- 数据类型转换和清洗
- 访问控制检查
- 钩子执行

## 4. 服务端清洗校验边界

### 4.1 服务端验证的三个阶段

#### 阶段 1：字段配置清理 (sanitizeField)

**位置：** `packages/payload/src/fields/config/sanitize.ts`

**职责：**
- 字段名验证（保留字、重复名、格式）
- 默认验证函数注入
- 默认值设置
- 子字段递归清理
- 时区字段自动插入
- Rich Text 编辑器配置

#### 阶段 2：提交前数据清洗 (beforeValidate)

**位置：** `packages/payload/src/fields/hooks/beforeValidate/promise.ts`

**职责：**
- **类型转换**（核心清洗逻辑）：
  - `number`: 字符串转数字，空字符串转 `null`
  - `checkbox`: 字符串 "true"/"false"/"" 转布尔值
  - `array`/`blocks`: "0" 或 `0` 转空数组 `[]`
  - `point`: 坐标字符串转数字数组
  - `relationship`/`upload`: 关系值归一化（提取 ID、类型转换）
  - `richText`: JSON 字符串解析
  - `id`: 类型转换（数字 ID 或字符串 ID）

- **默认值计算**：`getFallbackValue`
- **beforeValidate 钩子执行**
- **访问控制检查**：无权限则删除字段
- **子字段递归处理**

#### 阶段 3：持久化前验证 (beforeChange)

**位置：** `packages/payload/src/fields/hooks/beforeChange/promise.ts`

**职责：**
- **条件检查**：`admin.condition` 函数执行
- **beforeChange 钩子执行**
- **最终验证**：`validate` 函数执行，`event: 'submit'`
- **数据转换为存储格式**：
  - `point`: 数组转 GeoJSON `{ type: 'Point', coordinates: [...] }`
- **本地化数据合并**
- **错误收集**：`ValidationFieldError[]`

### 4.2 服务端验证调用

```typescript
// 服务端 beforeChange 中的验证调用
const validationResult = await validateFn(valueToValidate, {
  ...field,
  id,
  blockData: blockData!,      // 完整的 blockData
  collectionSlug: collection?.slug,
  data: deepMergeWithSourceArrays(doc, data),  // 合并后的数据
  event: 'submit',            // 重要：服务端标识
  jsonError,
  operation,
  overrideAccess,
  path: pathSegments,
  preferences: { fields: {} },
  previousValue: siblingDoc[field.name],
  req,                         // 完整的 PayloadRequest
  siblingData: deepMergeWithSourceArrays(siblingDoc, siblingData),
})
```

### 4.3 服务端验证边界

**服务端负责：**
- 数据类型安全转换
- 完整的业务规则验证
- 数据库一致性检查
- 访问控制强制执行
- 钩子执行链
- 数据持久化格式转换

## 5. 一致性保障机制

### 5.1 单一验证函数源

**关键设计：** 验证函数 `validate` 在字段配置中只定义**一次**，但在多个场景下执行。

```typescript
// 字段配置中的 validate
const field: TextField = {
  name: 'username',
  type: 'text',
  validate: async (value, options) => {
    // 同一函数在客户端和服务端都可能被调用
    // 通过 options.event 区分执行环境
  }
}
```

**执行场景：**

| 场景 | 执行位置 | event | 说明 |
|------|---------|-------|------|
| 客户端实时验证 | 浏览器 | `'onChange'` | 节流，模拟 req |
| 表单状态构建 | RSC 服务端 | `'onChange'` | 完整 req，构建 FormState |
| 提交时最终验证 | API 服务端 | `'submit'` | 完整上下文，最终检查 |

### 5.2 验证函数的环境感知

验证函数可以通过 `options.event` 和 `options.req` 感知执行环境：

```typescript
validate: (value, { event, req, collectionSlug, data }) => {
  // 仅在服务端提交时执行数据库查询
  if (event === 'submit' && req?.payload?.db) {
    // 执行数据库唯一性检查等
  }
  
  // 客户端和服务端都执行的基本验证
  if (!value?.trim()) {
    return '此字段不能为空'
  }
  
  return true
}
```

### 5.3 表单状态的服务端构建

**位置：** `packages/ui/src/forms/fieldSchemasToFormState/addFieldStatePromise.ts`

**机制：**
- 在 RSC（React Server Components）环境中执行
- 使用完整的服务端字段配置（包含 validate 函数）
- 构建 `FormState`，包含 `valid`, `errorMessage`, `errorPaths`
- 此状态通过序列化传递给客户端

```typescript
// addFieldStatePromise 中的验证
if (typeof validate === 'function' && !skipValidation && passesCondition) {
  validationResult = await validate(data?.[field.name], {
    ...field,
    id,
    blockData,          // 完整 blockData
    collectionSlug,
    data: fullData,
    event: 'onChange',  // 注意：这里也是 onChange
    jsonError,
    operation,
    preferences,
    previousValue: previousFormState?.[path]?.initialValue,
    req,                // 完整的 PayloadRequest
    siblingData: data,
  })
}
```

### 5.4 字段配置的序列化传递

**位置：** `packages/payload/src/fields/config/client.ts`

**机制：**
- `createClientFields` 将服务端 `Field` 转换为 `ClientField`
- 函数类型属性（`label`, `description`）被**执行**，结果序列化
- 函数引用被剥离（无法序列化）

**关键代码：**
```typescript
// label 函数在转换时执行
if (typeof incomingField.label === 'function') {
  clientField.label = incomingField.label({ i18n, t: i18n.t })
} else {
  clientField.label = incomingField.label
}
```

### 5.5 双重类型检查

Payload 通过两种机制确保数据类型正确：

1. **beforeValidate 类型转换**：主动将字符串转为目标类型
2. **JSON Schema 生成**：`configToJSONSchema` 生成 TypeScript 类型定义

**位置：** `packages/payload/src/utilities/configToJSONSchema.ts`

```typescript
// 例如 number 字段的 Schema
case 'number': {
  if (field.hasMany === true) {
    fieldSchema = {
      type: withNullableJSONSchemaType('array', isRequired),
      items: { type: 'number' },
    }
  } else {
    fieldSchema = {
      type: withNullableJSONSchemaType('number', isRequired),
    }
  }
}
```

## 6. 边界划分总结表

| 职责 | 客户端编辑 | 服务端清洗校验 |
|------|-----------|---------------|
| **数据类型转换** | ❌ 不负责 | ✅ beforeValidate 阶段 |
| **默认值计算** | ❌ 不负责 | ✅ getFallbackValue |
| **即时 UI 反馈** | ✅ useField 节流验证 | ❌ 不负责 |
| **表单状态验证** | ✅ 接收服务端构建的状态 | ✅ fieldSchemasToFormState |
| **钩子执行** | ❌ 不负责 | ✅ beforeValidate / beforeChange |
| **访问控制** | ❌ 不负责 | ✅ 服务端各阶段检查 |
| **最终数据验证** | ❌ 不负责 | ✅ beforeChange 阶段 |
| **数据库查询验证** | ❌ 不负责 | ✅ event === 'submit' 时 |
| **错误消息生成** | ✅ 显示服务端返回的消息 | ✅ 验证函数返回 |
| **字段条件判断** | ✅ 客户端执行简化版本 | ✅ 服务端完整执行 |

## 7. 自定义字段开发指南

### 7.1 验证函数编写最佳实践

```typescript
const customField: Field = {
  name: 'customField',
  type: 'text',
  validate: async (value, options) => {
    const { event, req, data, siblingData } = options
    
    // 1. 基础验证 - 客户端和服务端都执行
    if (!value) {
      return options.required ? '此字段必填' : true
    }
    
    // 2. 格式验证 - 可在客户端执行
    if (!/^[a-zA-Z0-9_]+$/.test(value)) {
      return '只能包含字母、数字和下划线'
    }
    
    // 3. 仅服务端执行的验证
    if (event === 'submit') {
      // 数据库查询、API 调用等
      const exists = await req.payload.find({
        collection: 'my-collection',
        where: { customField: { equals: value } },
        limit: 1,
      })
      
      if (exists.docs.length > 0) {
        return '该值已被使用'
      }
    }
    
    return true
  }
}
```

### 7.2 客户端与服务端共享数据

使用 `admin.custom` 在客户端和服务端共享配置：

```typescript
const field: TextField = {
  name: 'myField',
  type: 'text',
  // 仅服务端可用
  custom: {
    serverOnlyConfig: 'secret'
  },
  admin: {
    // 客户端和服务端都可用
    custom: {
      sharedConfig: 'visible-both-sides'
    }
  }
}
```

### 7.3 数据清洗注意事项

如果自定义字段需要特殊的类型转换，应在 `beforeValidate` 钩子中处理：

```typescript
const customField: Field = {
  name: 'customDate',
  type: 'text',
  hooks: {
    beforeValidate: [
      ({ value }) => {
        // 将客户端传来的各种日期格式统一转换
        if (typeof value === 'string') {
          return new Date(value).toISOString()
        }
        if (value instanceof Date) {
          return value.toISOString()
        }
        return value
      }
    ]
  }
}
```

## 8. 关键代码位置

| 功能 | 文件路径 |
|------|---------|
| 客户端字段转换 | `packages/payload/src/fields/config/client.ts` |
| 服务端字段清理 | `packages/payload/src/fields/config/sanitize.ts` |
| 客户端实时验证 | `packages/ui/src/forms/useField/index.tsx` |
| 表单状态构建 | `packages/ui/src/forms/fieldSchemasToFormState/addFieldStatePromise.ts` |
| 数据清洗 (beforeValidate) | `packages/payload/src/fields/hooks/beforeValidate/promise.ts` |
| 最终验证 (beforeChange) | `packages/payload/src/fields/hooks/beforeChange/promise.ts` |
| 类型验证函数库 | `packages/payload/src/fields/validations.ts` |
| JSON Schema 生成 | `packages/payload/src/utilities/configToJSONSchema.ts` |

## 9. 结论

Payload CMS 的自定义字段验证采用**"单点定义，多点执行"**的策略：

1. **单一真相源**：`validate` 函数只定义一次
2. **渐进式验证**：客户端 → RSC 服务端 → API 服务端
3. **环境感知**：通过 `event` 参数区分执行场景
4. **职责分离**：
   - 客户端负责 UX（即时反馈）
   - 服务端负责安全（数据清洗、最终验证）
5. **类型安全**：通过自动类型转换和 JSON Schema 双重保障

这种设计既保证了良好的用户体验（即时反馈），又确保了数据的安全性（服务端最终验证），同时通过共享验证函数避免了逻辑重复。
