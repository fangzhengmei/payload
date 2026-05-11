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

## 9. 提交失败时的一致性收敛机制

当客户端校验通过但提交阶段失败时，Payload CMS 有完整的闭环机制将服务端错误同步回客户端表单状态，确保两侧一致性收敛。

### 9.1 完整闭环流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   提交失败一致性收敛闭环                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 1：服务端错误判定与收集                                            │
│  ─────────────────────────────                                          │
│  位置：beforeChange/promise.ts:202-264                                   │
│  - 遍历所有字段执行 validate 函数                                        │
│  - 收集 ValidationFieldError: { path, message, label? }                  │
│  - 支持嵌套字段路径 (如 "blocks.0.title")                                │
│  - 支持 blocks filterOptions 错误展开                                    │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 2：错误包装为 HTTP 响应                                            │
│  ─────────────────────────────                                          │
│  位置：errors/ValidationError.ts:21-78                                   │
│  - ValidationError extends APIError                                     │
│  - HTTP 状态码：400 BAD_REQUEST                                          │
│  - 响应体：{ collection?, global?, errors: ValidationFieldError[] }      │
│  - label 函数在错误构造时执行 (使用 req.i18n)                            │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 3：客户端接收与解析错误                                            │
│  ─────────────────────────────                                          │
│  位置：ui/src/forms/Form/index.tsx:456-517                               │
│  - res.status >= 400 判定为失败                                          │
│  - 解析 JSON 响应                                                        │
│  - 分离 fieldErrors (有 path) 和 nonFieldErrors                          │
│  - 错误分类逻辑：                                                        │
│    - err.data?.errors 中每个元素                                         │
│    - 有 path → fieldErrors                                              │
│    - 无 path → nonFieldErrors                                           │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 4：表单状态更新                                                    │
│  ─────────────────────────────                                          │
│  位置：ui/src/forms/Form/fieldReducer.ts:72-132                          │
│  - Action: 'ADD_SERVER_ERRORS'                                          │
│  - 为每个错误路径更新字段状态：                                           │
│    - valid: false                                                        │
│    - errorMessage: message                                               │
│  - 递归更新父级 errorPaths（用于聚合显示）                                │
│    - 如 "blocks.0.title" 错误 → "blocks.0" 和 "blocks" 的 errorPaths     │
│      都包含该路径                                                        │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 5：UI 显示错误                                                     │
│  ─────────────────────────────                                          │
│  位置：ui/src/forms/useField/index.tsx:62                                │
│  - showError = valid === false && submitted                              │
│  - 表单进入 submitted 状态后显示错误                                     │
│  - 组件通过 useField 获取 errorMessage 和 errorPaths                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 关键代码路径

#### 9.2.1 服务端错误收集

**位置：** `packages/payload/src/fields/hooks/beforeChange/promise.ts:202-264`

```typescript
// 服务端验证失败收集
if (typeof validationResult === 'string') {
  let filterOptionsError = false
  
  // blocks 字段特殊处理：展开每个无效 block 的错误
  if (field.type === 'blocks' && field.filterOptions) {
    const validationResult = await validateBlocksFilterOptions({...})
    if (validationResult?.invalidBlockSlugs?.length) {
      // 为每个无效 block 单独生成错误
      for (const block of siblingData[field.name] as JsonObject[]) {
        if (validationResult.invalidBlockSlugs.includes(block.blockType as string)) {
          errors.push({
            label: blockLabelPath,
            message: req.t('validation:invalidBlock', { block: block.blockType }),
            path: `${path}.${rowIndex}.id`,  // 嵌套路径
          })
        }
      }
    }
  }
  
  // 普通字段错误
  if (!filterOptionsError) {
    errors.push({
      label: fieldLabel,
      message: validationResult,  // 验证函数返回的错误消息
      path,                        // 字段完整路径
    })
  }
}
```

#### 9.2.2 客户端错误解析与状态更新

**位置：** `packages/ui/src/forms/Form/index.tsx:478-517`

```typescript
// 解析服务端返回的错误
if (Array.isArray(json.errors)) {
  const [fieldErrors, nonFieldErrors] = json.errors.reduce(
    ([fieldErrs, nonFieldErrs], err) => {
      // 分类：有 path 是字段错误，否则是非字段错误
      if (err?.data?.errors) {
        err.data.errors.forEach((dataError) => {
          if (dataError?.path) {
            newFieldErrs.push(dataError)  // → fieldErrors
          } else {
            newNonFieldErrs.push(dataError)  // → nonFieldErrors
          }
        })
      }
      return [[...fieldErrs, ...newFieldErrs], [...nonFieldErrs, ...newNonFieldErrs]]
    },
    [[], []]
  )
  
  // 更新表单状态
  dispatchFields({
    type: 'ADD_SERVER_ERRORS',
    errors: fieldErrors,  // 传入 reducer
  })
  
  // 非字段错误显示为 toast
  nonFieldErrors.forEach((err) => {
    errorToast(<FieldErrorsToast errorMessage={err.message} />)
  })
}
```

#### 9.2.3 表单状态更新逻辑

**位置：** `packages/ui/src/forms/Form/fieldReducer.ts:72-132`

```typescript
case 'ADD_SERVER_ERRORS': {
  let newState = { ...state }
  
  // 1. 为每个错误路径设置字段状态
  action.errors.forEach(({ message, path: fieldPath }) => {
    newState[fieldPath] = {
      ...(newState[fieldPath] || {
        initialValue: null,
        value: null,
      }),
      errorMessage: message,  // 错误消息
      valid: false,            // 标记为无效
    }
    
    // 收集父级路径（用于 errorPaths）
    const segments = fieldPath.split('.')
    if (segments.length > 1) {
      errorPaths.push({
        fieldErrorPath: fieldPath,
        parentPath: segments.slice(0, segments.length - 1).join('.'),
      })
    }
  })
  
  // 2. 递归更新父级字段的 errorPaths
  // 例如 "blocks.0.title" 错误要反映在 "blocks.0" 和 "blocks" 上
  newState = Object.entries(newState).reduce((acc, [path, fieldState]) => {
    const fieldErrorPaths = errorPaths.reduce((errorACC, { fieldErrorPath, parentPath }) => {
      if (parentPath.startsWith(path)) {
        errorACC.push(fieldErrorPath)
      }
      return errorACC
    }, [])
    
    if (fieldErrorPaths.length > 0) {
      acc[path] = {
        ...fieldState,
        errorPaths: [...(fieldState.errorPaths || []), ...fieldErrorPaths],
      }
    }
    return acc
  }, {})
  
  return newState
}
```

### 9.3 状态标志更新

除了字段状态，表单还会更新以下全局状态：

```typescript
// Form/index.tsx:457-469
setProcessing(false)   // 结束处理状态
setSubmitted(true)     // 标记为已提交（触发 UI 显示错误）

// 草稿提交失败特殊处理
if (overridesFromArgs['_status'] === 'draft') {
  setModified(true)     // 保持修改状态，允许重试
  if (!validateDrafts) {
    setSubmitted(false) // 草稿不显示验证错误
  }
}

setIsValid(false)       // 全局标记为无效
contextRef.current = { ...contextRef.current }  // 触发订阅组件重渲染
```

### 9.4 典型失配场景与分析

#### 场景 1：数据库唯一性检查失配

**问题描述：**
- 客户端验证通过（格式检查、长度限制等）
- 服务端提交时发现数据库中已存在相同值（唯一性约束）
- 这种检查无法在客户端执行，因为需要数据库查询

**字段配置示例：**

```typescript
// 自定义用户名字段
const usernameField: TextField = {
  name: 'username',
  type: 'text',
  required: true,
  unique: true,  // 数据库层面唯一约束
  
  validate: async (value, options) => {
    const { event, req, collectionSlug, id } = options
    
    // 1. 基础格式验证 - 客户端和服务端都执行
    if (!value || value.length < 3) {
      return '用户名至少需要 3 个字符'
    }
    
    if (!/^[a-zA-Z0-9_]+$/.test(value)) {
      return '用户名只能包含字母、数字和下划线'
    }
    
    // 2. 唯一性检查 - 仅在服务端提交时执行
    //    客户端无法访问数据库，这个检查会被跳过
    if (event === 'submit' && req?.payload?.db) {
      const existingDocs = await req.payload.find({
        collection: collectionSlug!,
        where: {
          username: { equals: value },
          ...(id ? { id: { not_equals: id } } : {}), // 排除自身
        },
        limit: 1,
      })
      
      if (existingDocs.docs.length > 0) {
        return '该用户名已被使用，请选择另一个'
      }
    }
    
    return true
  }
}
```

**失配流程：**
1. 用户输入 `admin123`
2. 客户端 `useField` 验证：
   - `event = 'onChange'`
   - 跳过数据库查询分支
   - 格式检查通过 → `valid: true`
3. 用户点击保存
4. 服务端 `beforeChange` 验证：
   - `event = 'submit'`
   - 执行数据库查询
   - 发现已存在 → 返回错误消息
5. 服务端返回 400 响应：
   ```json
   {
     "errors": [{
       "data": {
         "errors": [{
           "message": "该用户名已被使用，请选择另一个",
           "path": "username"
         }]
       }
     }]
   }
   ```
6. 客户端 `ADD_SERVER_ERRORS` 动作：
   - `username` 字段 `valid: false`
   - `errorMessage: '该用户名已被使用，请选择另一个'`
   - `submitted: true` 触发 UI 显示

**关键设计：**
- 验证函数通过 `event === 'submit'` 隔离数据库查询
- 错误通过 `path` 精确映射到对应字段
- 错误消息在服务端构造（支持 i18n），客户端直接显示

#### 场景 2：跨字段业务规则验证失配

**问题描述：**
- 单个字段验证通过
- 但字段之间的组合关系不符合业务规则
- 这种验证依赖完整的表单数据和上下文

**字段配置示例：**

```typescript
// 优惠码系统：开始日期必须早于结束日期
const startDateField: DateField = {
  name: 'startDate',
  type: 'date',
  required: true,
  validate: (value, { event, data, siblingData }) => {
    if (!value) return '请选择开始日期'
    
    // 跨字段验证仅在服务端提交时执行
    // 客户端 onChange 时 siblingData 可能不完整
    if (event === 'submit') {
      const endDate = data?.endDate || siblingData?.endDate
      if (endDate && new Date(value) > new Date(endDate)) {
        return '开始日期必须早于结束日期'
      }
    }
    
    return true
  }
}

const endDateField: DateField = {
  name: 'endDate',
  type: 'date',
  required: true,
  validate: (value, { event, data, siblingData }) => {
    if (!value) return '请选择结束日期'
    
    if (event === 'submit') {
      const startDate = data?.startDate || siblingData?.startDate
      if (startDate && new Date(value) < new Date(startDate)) {
        return '结束日期必须晚于开始日期'
      }
    }
    
    return true
  }
}

// 或使用 collection 级别的钩子
const collectionConfig = {
  slug: 'promotions',
  fields: [startDateField, endDateField],
  hooks: {
    beforeChange: [
      async ({ data }) => {
        if (data.startDate && data.endDate) {
          if (new Date(data.startDate) > new Date(data.endDate)) {
            throw new ValidationError([{
              message: '开始日期必须早于结束日期',
              path: 'startDate',
            }])
          }
        }
        return data
      }
    ]
  }
}
```

**失配流程：**
1. 用户先选择开始日期 `2025-01-15` → 客户端验证通过
2. 用户选择结束日期 `2025-01-10`（早于开始日期）→ 单独验证通过
3. 客户端实时验证 `event = 'onChange'`，跨字段检查被跳过
4. 用户点击保存
5. 服务端 `beforeChange` 验证：
   - `event = 'submit'`
   - 有完整的 `data` 和 `siblingData`
   - 发现日期顺序错误
6. 错误可能关联到：
   - `startDate`（提示用户检查）
   - `endDate`（提示用户检查）
   - 或两个字段都关联
7. 客户端显示错误，用户需要调整其中一个日期

**为什么客户端不执行跨字段验证？**

查看 `useField` 的验证调用：
```typescript
// useField/index.tsx
const data = getData()  // 当前表单数据快照

// 客户端 onChange 时：
// 1. 用户刚修改完字段，data 可能还是旧值
// 2. 节流 150ms 后执行，时机不确定
// 3. blockData 明确传递为 undefined（性能考虑）

// 而服务端 submit 时：
const data = deepMergeWithSourceArrays(doc, data)  // 合并已有文档和提交数据
const siblingData = deepMergeWithSourceArrays(siblingDoc, siblingData)
const blockData = blockData!  // 完整的 block 数据
```

**收敛机制特点：**

1. **路径精确映射**：服务端错误通过 `path` 精确对应到字段
2. **嵌套路径支持**：支持 `blocks.0.fieldName` 等嵌套结构
3. **父级聚合**：`errorPaths` 让父字段知道子字段有错误
4. **草稿保护**：草稿提交失败保持 `modified` 状态，允许重试
5. **状态同步**：`submitted` 标志控制 UI 是否显示错误

### 9.5 一致性收敛的关键保证

| 保证维度 | 实现机制 | 关键文件 |
|---------|---------|---------|
| **错误精确映射** | `path` 字段作为桥梁 | ValidationError.ts, fieldReducer.ts |
| **嵌套结构支持** | 点分隔路径，父级 errorPaths 聚合 | fieldReducer.ts:87-129 |
| **状态同步** | `ADD_SERVER_ERRORS` action 原子更新 | fieldReducer.ts:72-132 |
| **UI 触发条件** | `submitted && !valid` 双条件 | useField/index.tsx:62 |
| **i18n 一致性** | 错误消息在服务端构造时翻译 | ValidationError.ts:39-44, 52-70 |
| **草稿友好** | 失败后保持 modified 状态 | Form/index.tsx:463-468 |

## 10. 提交失败后的恢复链路

当用户修正输入并重试提交时，系统需要清除历史错误并重新对齐状态。以下是完整的恢复流程。

### 10.1 恢复链路总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   提交失败恢复链路                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 1：用户修正输入 → 本地验证触发                                      │
│  ───────────────────────────────────────                                │
│  - 用户修改错误字段的值                                                   │
│  - useField 节流验证 (150ms) 重新运行                                    │
│  - 根据新值重新计算 errorMessage 和 valid                                 │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 2：本地状态更新 → 清除历史错误                                      │
│  ───────────────────────────────────────                                │
│  - UPDATE action 更新字段状态                                            │
│  - 新验证结果覆盖旧的服务端错误                                           │
│  - errorPaths 不会自动清除（需等服务端状态合并）                          │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 3：onChange 防抖 → 服务端状态重建                                  │
│  ───────────────────────────────────────                                │
│  - 防抖 250ms 后触发 onChange 回调                                       │
│  - 发送完整表单状态到服务端重新构建                                       │
│  - 服务端返回新的 FormState（无错误状态）                                │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 4：MERGE_SERVER_STATE → 状态对齐                                   │
│  ───────────────────────────────────────                                │
│  - mergeServerFormState 合并服务端状态                                   │
│  - 清除 errorPaths（服务端状态无 errorPaths）                            │
│  - 恢复 valid = true, passesCondition = true                            │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 5：再次提交 → 提交前全量验证                                        │
│  ───────────────────────────────────────                                │
│  - validateForm 遍历所有字段                                             │
│  - event = 'submit' 的客户端验证                                         │
│  - REPLACE_STATE 用最新验证结果替换                                      │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 6：提交成功 → 服务端权威状态覆盖                                    │
│  ───────────────────────────────────────                                │
│  - MERGE_SERVER_STATE (acceptValues = true)                             │
│  - 服务端是权威，覆盖本地值                                               │
│  - setSubmitted(false) 隐藏错误 UI                                       │
│  - 客户端与服务端状态完全对齐                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 用户修正输入时的错误清除

**触发源：** `useField` 中的节流验证

**位置：** `packages/ui/src/forms/useField/index.tsx:135-224`

当用户修改字段值时，`value` 作为依赖项变化，触发节流验证：

```typescript
// useField/index.tsx:135-224
useThrottledEffect(
  () => {
    const validateField = async () => {
      let valueToValidate = value

      // ... 获取要验证的值

      // 1. 初始值：使用上一次的错误状态
      let errorMessage: string | undefined = prevErrorMessage.current
      let valid: boolean | string = prevValid.current

      const data = getData()
      const isValid =
        typeof validate === 'function'
          ? await validate(valueToValidate, {
              // ... 验证选项
              event: 'onChange',  // 注意：是 onChange，不是 submit
            })
          : typeof prevErrorMessage.current === 'string'
            ? prevErrorMessage.current
            : prevValid.current

      // 2. 根据验证结果更新状态
      if (typeof isValid === 'string') {
        valid = false
        errorMessage = isValid
      } else if (typeof isValid === 'boolean') {
        valid = isValid
        errorMessage = undefined  // 验证通过，清除错误消息
      }

      // 3. 仅在状态变化时 dispatch
      if (valid !== prevValid.current || errorMessage !== prevErrorMessage.current) {
        prevValid.current = valid
        prevErrorMessage.current = errorMessage

        const update: UPDATE = {
          type: 'UPDATE',
          errorMessage,  // 新的 errorMessage（可能是 undefined）
          path,
          valid,         // 新的 valid
          value,
        }

        dispatchField(update)
      }
    }

    void validateField()
  },
  150,
  [value, ...]  // value 变化时触发
)
```

**关键机制：**
1. 验证函数使用 `event: 'onChange'`，但会重新评估所有验证逻辑
2. 如果验证通过（`isValid === true`），`errorMessage` 被显式设为 `undefined`
3. `UPDATE` action 用新状态覆盖旧状态（包括之前的服务端错误）

### 10.3 UPDATE action 的状态更新

**位置：** `packages/ui/src/forms/Form/fieldReducer.ts:398-442`

```typescript
case 'UPDATE': {
  const newField = Object.entries(action).reduce(
    (field, [key, value]) => {
      if (
        [
          'disableFormData',
          'errorMessage',  // ← 允许更新
          'initialValue',
          'rows',
          'valid',         // ← 允许更新
          'validate',
          'value',
        ].includes(key)
      ) {
        return {
          ...field,
          [key]: value,  // 新值覆盖旧值
          ...(key === 'value' ? { isModified: true } : {}),
        }
      }
      return field
    },
    state?.[action.path] || ({} as FormField),
  )

  const newState = {
    ...state,
    [action.path]: newField,  // 完全替换该字段的状态
  }

  return newState
}
```

**状态覆盖示例：**

假设之前服务端返回的错误状态：
```typescript
// 服务端错误后的状态
{
  'username': {
    value: 'admin123',
    valid: false,
    errorMessage: '该用户名已被使用',
    errorPaths: [],
    // ...
  }
}
```

用户修改为 `admin456` 后，useField 验证通过：
```typescript
// UPDATE action 带来的新状态
{
  'username': {
    value: 'admin456',
    valid: true,            // ← 覆盖为 true
    errorMessage: undefined, // ← 覆盖为 undefined（清除）
    errorPaths: [],         // ← 保持不变（不会自动清除）
    isModified: true,       // ← 新标记
    // ...
  }
}
```

**注意：** `errorPaths` 不会被 `UPDATE` action 清除，因为它不在允许更新的 key 列表中。需要等待 `MERGE_SERVER_STATE` 来清除。

### 10.4 onChange 防抖与服务端状态重建

**位置：** `packages/ui/src/forms/Form/index.tsx:832-867`

用户输入 250ms 后，触发 `onChange` 回调链：

```typescript
const executeOnChange = useEffectEvent((submitted: boolean) => {
  queueTask(async () => {
    if (Array.isArray(onChange)) {
      let serverState: FormState

      for (const onChangeFn of onChange) {
        // Edit view 的 onChange 会调用 getFormState
        serverState = await onChangeFn({
          formState: deepCopyObjectSimpleWithoutReactComponents(formState, {
            excludeFiles: true,
          }),
          submitted,
        })
      }

      // 用服务端返回的新状态合并
      dispatchFields({
        type: 'MERGE_SERVER_STATE',
        prevStateRef: prevFormState,
        serverState,
      })
    }
  })
})

// 防抖 250ms
useDebouncedEffect(
  () => {
    if ((isFirstRenderRef.current || !dequal(formState, prevFormState.current)) && modified) {
      executeOnChange(submitted)
    }
    prevFormState.current = formState
    isFirstRenderRef.current = false
  },
  [modified, submitted, formState],
  250,  // 防抖时间
)
```

### 10.5 MERGE_SERVER_STATE 的状态合并

**位置：** `packages/ui/src/forms/Form/mergeServerFormState.ts:82-238`

服务端返回的新 FormState 会通过 `mergeServerFormState` 合并到客户端：

```typescript
export const mergeServerFormState = ({
  acceptValues,
  currentState = {},
  incomingState,
}: Args): FormState => {
  const newState = { ...currentState }

  for (const [path, incomingField] of Object.entries(incomingState || {})) {
    // ... 值合并逻辑（见下文）

    newState[path] = {
      ...currentState[path],
      ...sanitizedIncomingField,
    }

    // 关键：清除 errorPaths
    // 如果当前状态有 errorPaths，但服务端状态没有，说明错误已解决
    if (
      currentState[path] &&
      'errorPaths' in currentState[path] &&
      !('errorPaths' in incomingField)
    ) {
      newState[path].errorPaths = []  // ← 清除历史错误路径
    }

    // 如果服务端不标记为 false，就认为是 true
    if (incomingField.valid !== false) {
      newState[path].valid = true  // ← 确保 valid 为 true
    }

    if (incomingField.passesCondition !== false) {
      newState[path].passesCondition = true  // ← 确保条件通过
    }
  }

  return dequal(newState, currentState) ? currentState : newState
}
```

**mergeServerFormState 的清除效果：**

```typescript
// 合并前的客户端状态（有历史错误标记）
{
  'username': {
    value: 'admin456',
    valid: true,           // 已被 useField 更新
    errorMessage: undefined, // 已被清除
    errorPaths: [],        // 空数组
    isModified: true,
  }
}

// 服务端返回的状态（干净状态）
{
  'username': {
    value: 'admin456',
    valid: true,
    passesCondition: true,
    // 没有 errorMessage, errorPaths
  }
}

// 合并后的状态
{
  'username': {
    value: 'admin456',
    valid: true,
    passesCondition: true,
    errorPaths: [],        // 被清除
    isModified: true,      // 保留本地修改标记
  }
}
```

### 10.6 再次提交前的全量验证

用户点击保存后，提交流程会先执行全量验证：

**位置：** `packages/ui/src/forms/Form/index.tsx:177-246`

```typescript
const validateForm = useCallback(async () => {
  const validatedFieldState = {}
  let isValid = true

  const data = contextRef.current.getData()

  // 遍历所有字段
  const validationPromises = Object.entries(contextRef.current.fields).map(
    async ([path, field]) => {
      const validatedField = field

      if (field.passesCondition !== false) {
        let validationResult: boolean | string = validatedField.valid

        if ('validate' in field && typeof field.validate === 'function') {
          // 重新验证
          validationResult = await field.validate(valueToValidate, {
            // ...
            event: 'submit',  // 注意：这里是 'submit'
          })

          if (typeof validationResult === 'string') {
            validatedField.errorMessage = validationResult
            validatedField.valid = false
          } else {
            validatedField.valid = true
            validatedField.errorMessage = undefined  // 确保清除
          }
        }

        if (validatedField.valid === false) {
          isValid = false
        }
      }

      validatedFieldState[path] = validatedField
    },
  )

  await Promise.all(validationPromises)

  // 用 REPLACE_STATE 完全替换表单状态
  if (!dequal(contextRef.current.fields, validatedFieldState)) {
    dispatchFields({ type: 'REPLACE_STATE', state: validatedFieldState })
  }

  setIsValid(isValid)
  return isValid
}, [...])
```

**关键差异：**
- `validateForm` 使用 `event: 'submit'`（虽然是在客户端执行）
- 用 `REPLACE_STATE` 完全替换，而不是部分更新
- 验证通过时显式设置 `errorMessage = undefined`

### 10.7 提交成功后的状态对齐

**位置：** `packages/ui/src/forms/Form/index.tsx:431-455`

```typescript
if (res.status < 400) {
  if (typeof onSuccess === 'function') {
    const newFormState = await onSuccess(json, {
      context,
      formState: serializableFormState,
    })

    if (newFormState) {
      dispatchFields({
        type: 'MERGE_SERVER_STATE',
        acceptValues: true,  // 关键：acceptValues = true
        prevStateRef: prevFormState,
        serverState: newFormState,
      })
    }
  }

  setSubmitted(false)  // 关键：隐藏错误 UI
  setProcessing(false)
  // ...
}
```

**acceptValues = true 的含义：**

在 `mergeServerFormState` 中：
```typescript
let shouldAcceptValue =
  incomingField.addedByServer ||
  acceptValues === true ||  // ← 这里为 true
  // ...
```

**效果：**
- 服务端返回的所有值都会被接受（覆盖本地值）
- 服务端是权威状态
- 客户端状态被重置为服务端的最新状态

**完整的状态重置：**

```typescript
// 提交前的状态（可能有本地修改标记）
{
  'username': {
    value: 'admin456',
    valid: true,
    isModified: true,      // 本地修改标记
    errorPaths: [],
  }
}

// 服务端返回的状态（权威状态）
{
  'username': {
    value: 'admin456',
    initialValue: 'admin456',  // 最新的初始值
    valid: true,
    passesCondition: true,
  }
}

// 合并后的状态（acceptValues = true）
{
  'username': {
    value: 'admin456',
    initialValue: 'admin456',  // 从服务端同步
    valid: true,
    passesCondition: true,
    errorPaths: [],
    // isModified 可能被清除（依赖 mergeServerFormState 逻辑）
  }
}

// 同时设置：
// submitted: false  → 错误 UI 隐藏
// modified: false   → 不再显示未保存更改
```

### 10.8 恢复链路完整示例

让我们用之前的**用户名唯一性场景**演示完整的恢复链路：

#### 初始失败状态
```
服务端：username='admin123' 已存在
客户端：
  - valid: false
  - errorMessage: '该用户名已被使用'
  - submitted: true  (显示错误)
  - modified: true   (可重试)
```

#### 阶段 1-2：用户修正输入
```
用户操作：将 username 改为 'admin456'

触发：useField 节流验证 (150ms 后)
  - validate('admin456', { event: 'onChange' })
  - 格式检查通过，数据库检查被跳过 (event !== 'submit')
  - 结果：isValid = true

UPDATE action：
  - username.valid = true
  - username.errorMessage = undefined
  - username.isModified = true

当前状态：
  - valid: true ✓
  - errorMessage: undefined ✓
  - submitted: true (仍显示 UI，但错误消息为空)
  - errorPaths: [] (未被 UPDATE 清除，但已空)
```

#### 阶段 3-4：onChange 服务端重建
```
触发：防抖 250ms 后
  - onChange 回调 → getFormState (RSC)
  - 服务端重新构建表单状态

服务端返回：
  - username.valid = true
  - 无 errorMessage
  - 无 errorPaths

MERGE_SERVER_STATE：
  - username.errorPaths = [] (显式清除)
  - username.valid = true (确认)
  - username.passesCondition = true (确认)

当前状态：
  - 完全干净的验证状态
  - 仅保留 isModified = true
```

#### 阶段 5：再次提交
```
用户操作：点击保存

触发：validateForm (提交前全量验证)
  - 遍历所有字段
  - validate('admin456', { event: 'submit' })
    - 注意：客户端无数据库访问，可能通过
    - 或使用新值后服务端逻辑

如果客户端验证通过：
  - REPLACE_STATE 替换为全量验证结果
  - 发送请求到服务端
```

#### 阶段 6：提交成功
```
服务端：
  - beforeChange 验证
    - validate('admin456', { event: 'submit' })
    - 数据库检查：admin456 不存在 ✓
  - 持久化成功

客户端：
  - res.status = 200
  - MERGE_SERVER_STATE (acceptValues = true)
    - 服务端权威状态覆盖
  - setSubmitted(false)  → 错误 UI 隐藏
  - setModified(false)   → 保存按钮变灰

最终状态：
  - 客户端与服务端完全对齐
  - 无错误状态
  - 可进行下一次编辑
```

### 10.9 恢复链路的关键保证

| 保证维度 | 触发时机 | 机制 | 关键文件 |
|---------|---------|------|---------|
| **errorMessage 清除** | 用户输入后 150ms | useField 节流验证 → UPDATE | useField/index.tsx:135-224 |
| **valid 状态恢复** | 用户输入后 150ms | useField 节流验证 → UPDATE | useField/index.tsx:172-202 |
| **errorPaths 清除** | onChange 防抖后 | MERGE_SERVER_STATE | mergeServerFormState.ts:148-154 |
| **全量重新验证** | 点击保存时 | validateForm → REPLACE_STATE | Form/index.tsx:177-246 |
| **UI 状态重置** | 提交成功后 | setSubmitted(false) | Form/index.tsx:448 |
| **值权威对齐** | 提交成功后 | MERGE_SERVER_STATE (acceptValues=true) | mergeServerFormState.ts:100-108 |

### 10.10 潜在问题与注意事项

1. **errorPaths 清除延迟**
   - UPDATE action 不会清除 errorPaths
   - 需等待 onChange 250ms 防抖后的 MERGE_SERVER_STATE
   - 影响：errorPaths 短暂残留（但通常不影响 UI）

2. **客户端与服务端 validate 的 event 差异**
   - validateForm 使用 `event: 'submit'`，但在客户端执行
   - 如果验证函数依赖 `req.payload.db` 等服务端特性，仍会跳过
   - 影响：提交前可能仍无法发现服务端才会检测的问题

3. **isModified 状态**
   - 服务端成功后，modified 被设为 false
   - 但如果 onSuccess 回调返回的状态包含 isModified，行为可能不同

## 11. 结论

Payload CMS 的自定义字段验证采用**"单点定义，多点执行"**的策略：

1. **单一真相源**：`validate` 函数只定义一次
2. **渐进式验证**：客户端 → RSC 服务端 → API 服务端
3. **环境感知**：通过 `event` 参数区分执行场景
4. **职责分离**：
   - 客户端负责 UX（即时反馈）
   - 服务端负责安全（数据清洗、最终验证）
5. **类型安全**：通过自动类型转换和 JSON Schema 双重保障

**失败收敛核心机制：**
- 服务端错误通过 `ValidationFieldError` 结构化收集
- `path` 字段作为服务端-客户端的精确映射桥梁
- `ADD_SERVER_ERRORS` 动作原子更新表单状态
- `submitted` 标志控制错误 UI 的显示时机

这种设计既保证了良好的用户体验（即时反馈），又确保了数据的安全性（服务端最终验证），同时通过共享验证函数避免了逻辑重复。在客户端和服务端验证结果不一致时，系统有完整的闭环机制将服务端的权威结果同步回客户端，最终达到一致性收敛。
