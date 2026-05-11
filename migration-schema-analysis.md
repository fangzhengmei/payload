# Payload CMS 数据库迁移与结构同步差异分析报告

## 一、概述

Payload CMS 支持多种数据库适配器，包括 MongoDB、Postgres、SQLite、Vercel Postgres 和 D1 SQLite。本文档详细分析了这些适配器之间在数据库迁移和结构同步方面如何统一处理差异。

## 二、整体架构

### 2.1 统一接口定义

**文件**: `packages/payload/src/database/types.ts`

`BaseDatabaseAdapter` 接口定义了所有数据库适配器必须实现的统一契约：

```typescript
export interface BaseDatabaseAdapter {
  // 迁移相关方法
  migrate: (args?: { migrations?: Migration[] }) => Promise<void>
  migrateDown: () => Promise<void>
  migrateFresh: (args: { forceAcceptWarning?: boolean }) => Promise<void>
  migrateRefresh: () => Promise<void>
  migrateReset: () => Promise<void>
  migrateStatus: () => Promise<void>
  createMigration: CreateMigration
  
  // 其他通用方法
  defaultIDType: 'number' | 'text'
  name: string
  packageName: string
  migrationDir: string
  // ... 其他 CRUD 方法
}
```

### 2.2 适配器工厂

**文件**: `packages/payload/src/database/createDatabaseAdapter.ts`

`createDatabaseAdapter` 函数提供了默认实现，适配器可以选择性覆盖：

```typescript
export function createDatabaseAdapter<T extends BaseDatabaseAdapter>(
  args: MarkOptional<T, 
    | 'createMigration'
    | 'migrate'
    | 'migrateDown'
    | 'migrateFresh'
    | 'migrateRefresh'
    | 'migrateReset'
    | 'migrateStatus'
    | 'migrationDir'
    | ...
  >
): T
```

## 三、两类适配器体系

### 3.1 文档型数据库 (MongoDB)

- **包**: `@payloadcms/db-mongodb`
- **底层**: Mongoose
- **特点**: 无 schema 迁移概念（schema-less）

### 3.2 关系型数据库 (SQL 系)

- **Postgres**: `@payloadcms/db-postgres`
- **SQLite**: `@payloadcms/db-sqlite`
- **Vercel Postgres**: `@payloadcms/db-vercel-postgres`
- **D1 SQLite**: `@payloadcms/db-d1-sqlite`
- **底层**: Drizzle ORM
- **统一层**: `@payloadcms/drizzle`

## 四、迁移机制差异处理

### 4.1 迁移执行流程对比

#### MongoDB 迁移流程

**文件**: `packages/db-mongodb/src/createMigration.ts`

- 生成简单的 TypeScript 模板
- 使用 `session` 进行事务操作
- 迁移参数: `{ payload, req, session }`

```typescript
export async function up({ payload, req, session }: MigrateUpArgs): Promise<void> {
  // 直接操作 Mongoose 模型
}
```

#### Drizzle 系迁移流程

**文件**: `packages/drizzle/src/migrate.ts`

- 使用 Drizzle Kit 生成 SQL 语句
- 支持开发模式 schema push
- 迁移参数: `{ db, payload, req }`

```typescript
export const migrate: DrizzleAdapter['migrate'] = async function migrate(
  this: DrizzleAdapter,
  args
) {
  const migrationFiles = args?.migrations || (await readMigrationFiles({ payload }))
  
  // 检查是否存在 dev migration (batch = -1)
  // 执行每个 migration 的 up 函数
}
```

### 4.2 迁移文件生成差异

#### MongoDB 模板

```typescript
// packages/db-mongodb/src/createMigration.ts
const migrationTemplate = ({ downSQL, imports, upSQL }): string => `
import { MigrateDownArgs, MigrateUpArgs } from '@payloadcms/db-mongodb'
${imports ?? ''}

export async function up({ payload, req, session }: MigrateUpArgs): Promise<void> {
  ${upSQL ?? `  // Migration code`}
}

export async function down({ payload, req, session }: MigrateDownArgs): Promise<void> {
  ${downSQL ?? `  // Migration code`}
}
`
```

#### Postgres 模板

**文件**: `packages/drizzle/src/utilities/buildCreateMigration.ts`

```typescript
export const buildCreateMigration = ({
  executeMethod,      // 'execute' for Postgres
  filename,
  sanitizeStatements, // 特定于数据库的语句处理
}): CreateMigration => {
  // 使用 Drizzle Kit 生成快照
  const drizzleJsonAfter = await generateDrizzleJson(this.schema)
  
  // 对比前后快照生成 SQL
  const sqlStatementsUp = await generateMigration(drizzleJsonBefore, drizzleJsonAfter)
  const sqlStatementsDown = await generateMigration(drizzleJsonAfter, drizzleJsonBefore)
}
```

#### SQLite 模板

**文件**: `packages/db-sqlite/src/index.ts:112-123`

```typescript
const executeMethod = 'run'  // 区别于 Postgres 的 'execute'
const sanitizeStatements = ({ sqlExecute, statements }) => {
  return statements
    .map((statement) => `${sqlExecute}${statement?.replaceAll('`', '\\`')}\`)`)
    .join('\n')
}
```

### 4.3 预定义迁移系统

**文件**: `packages/payload/src/database/migrations/getPredefinedMigration.ts`

统一的预定义迁移加载机制：

```typescript
export const getPredefinedMigration = async ({
  dirname,
  file,
  migrationName,
  payload,
}): Promise<MigrationTemplateArgs> => {
  // 路径 1: @payloadcms/db-* 包
  if (importPath?.startsWith('@payloadcms/db-')) {
    const migrationName = importPath.split('/').slice(2).join('/')
    // 从 predefinedMigrations 文件夹加载
  }
  
  // 路径 2: 其他包或绝对路径
  else if (importPath) {
    // 动态导入
  }
}
```

**各适配器预定义迁移示例**:

| 适配器 | 预定义迁移 | 用途 |
|--------|-----------|------|
| MongoDB | `relationships-v2-v3.ts` | v2 到 v3 关系字段迁移 |
| MongoDB | `versions-v1-v2.ts` | 版本系统迁移 |
| Postgres | `relationships-v2-v3.ts` | 同上 |
| Postgres | `blocks-as-json.ts` | Block 字段 JSON 化 |
| SQLite | `blocks-as-json.ts` | 同上 |

## 五、结构同步（Schema Synchronization）

### 5.1 抽象 Schema 层

**文件**: `packages/drizzle/src/types.ts`

定义了数据库无关的抽象 Schema 类型：

```typescript
// 抽象 SQL 表
export type RawTable = {
  name: string
  columns: Record<string, RawColumn>
  foreignKeys?: Record<string, RawForeignKey>
  indexes?: Record<string, RawIndex>
}

// 抽象列类型 - 包含数据库特定注释
export type TimestampRawColumn = {
  defaultNow?: boolean
  mode: 'date' | 'string'
  precision: Precision
  type: 'timestamp'
  withTimezone?: boolean  // Postgres 特有
} & BaseRawColumn

export type EnumRawColumn = (
  | { enumName: string; options: string[]; type: 'enum' }  // Postgres 原生枚举
  | { locale: true; type: 'enum' }
) & BaseRawColumn
```

### 5.2 抽象 Schema 构建

**文件**: `packages/drizzle/src/schema/build.ts`

统一的 Schema 构建逻辑：

```typescript
export const buildTable = ({
  adapter,
  fields,
  setColumnID,  // 数据库特定的 ID 类型设置
  tableName,
  timestamps,
  versions,
}: Args): Result => {
  // 1. 构建基础列
  // 2. 处理多语言字段 → 生成 _locales 表
  // 3. 处理多值字段 → 生成 _texts / _numbers 表
  // 4. 处理关系字段 → 生成 _rels 表
  // 5. 写入 adapter.rawTables
  adapter.rawTables[tableName] = table
}
```

### 5.3 数据库特定转换

#### Postgres 转换

**文件**: `packages/drizzle/src/postgres/schema/buildDrizzleTable.ts`

```typescript
const rawColumnBuilderMap: Partial<Record<RawColumn['type'], any>> = {
  boolean,
  geometry: geometryColumn,
  integer,
  jsonb,
  numeric,
  serial,
  text,
  uuid,
  varchar,
  vector,      // pgvector 支持
  halfvec,     // pgvector 支持
  sparsevec,   // pgvector 支持
  bit,         // pgvector 支持
}

// 枚举处理 - 使用原生 Postgres 枚举
case 'enum':
  if ('locale' in column) {
    columns[key] = adapter.enums.enum__locales(column.name)
  } else {
    adapter.enums[column.enumName] = adapter.pgSchema.enum(
      column.enumName,
      column.options as [string, ...string[]],
    )
    columns[key] = adapter.enums[column.enumName](column.name)
  }
  break

// 时间戳处理
case 'timestamp':
  let builder = timestamp(column.name, {
    mode: column.mode,
    precision: column.precision,
    withTimezone: column.withTimezone,  // 支持时区
  })
```

#### SQLite 转换

**文件**: `packages/drizzle/src/sqlite/schema/buildDrizzleTable.ts`

```typescript
const rawColumnBuilderMap: Partial<Record<RawColumn['type'], any>> = {
  integer,
  numeric,
  text,
}

// 布尔值处理 - 使用 INTEGER
case 'boolean':
  columns[key] = integer(column.name, { mode: 'boolean' })
  break

// 枚举处理 - 使用 TEXT 带 check 约束
case 'enum':
  if ('locale' in column) {
    columns[key] = text(column.name, { enum: locales as [string, ...string[]] })
  } else {
    columns[key] = text(column.name, { enum: column.options as [string, ...string[]] })
  }
  break

// JSON/Geometry 处理 - 使用 TEXT JSON 模式
case 'geometry':
case 'jsonb':
  columns[key] = text(column.name, { mode: 'json' })
  break

// 时间戳处理 - 使用 TEXT + strftime
case 'timestamp':
  let builder = text(column.name)
  if (column.defaultNow) {
    builder = builder.default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`)
  }
  break

// UUID 处理 - 使用 TEXT + 应用层默认值
case 'uuid':
  let builder = text(column.name, { length: 36 })
  if (column.defaultRandom) {
    builder = builder.$defaultFn(() => uuidv4())  // 应用层生成
  }
  break
```

### 5.4 初始化流程对比

**Postgres 初始化**: `packages/drizzle/src/postgres/init.ts`

```typescript
export const init: Init = async function init(this: BasePostgresAdapter) {
  buildRawSchema({ adapter: this, setColumnID })
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  // Postgres 特有: 创建 locale 枚举
  if (this.payload.config.localization) {
    this.enums.enum__locales = this.pgSchema.enum(
      '_locales',
      this.payload.config.localization.locales.map(({ code }) => code)
    )
  }
  
  // 转换抽象表为 Drizzle 表
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter: this, rawTable: this.rawTables[tableName] })
  }
  
  buildDrizzleRelations({ adapter: this })
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  // 组装完整 schema
  this.schema = {
    pgSchema: this.pgSchema,
    ...this.tables,
    ...this.relations,
    ...this.enums,  // 包含枚举
  }
}
```

**SQLite 初始化**: `packages/drizzle/src/sqlite/init.ts`

```typescript
export const init: Init = async function init(this: BaseSQLiteAdapter) {
  let locales: string[] | undefined
  if (this.payload.config.localization) {
    locales = this.payload.config.localization.locales.map(({ code }) => code)
  }
  
  buildRawSchema({ adapter, setColumnID })
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  // 传入 locales 用于 enum 列
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter, locales, rawTable: this.rawTables[tableName] })
  }
  
  buildDrizzleRelations({ adapter })
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  // SQLite 无枚举
  this.schema = {
    ...this.tables,
    ...this.relations,
  }
}
```

## 六、开发模式 Schema Push

### 6.1 统一 Push 机制

**文件**: `packages/drizzle/src/utilities/pushDevSchema.ts`

```typescript
export const pushDevSchema = async (adapter: DrizzleAdapter) => {
  // 检查 schema 是否变化
  const equal = dequal(previousSchema, {
    localeCodes,
    rawTables: adapter.rawTables,
  })
  if (equal) return
  
  // 使用 Drizzle Kit 的 pushSchema
  const { pushSchema } = adapter.requireDrizzleKit()
  
  const { apply, hasDataLoss, warnings } = await pushSchema(
    adapter.schema,
    adapter.drizzle,
    adapter.schemaName ? [adapter.schemaName] : undefined,
    tablesFilter,
    extensions.postgis ? ['postgis'] : undefined,  // Postgres 扩展
  )
  
  // 处理警告和数据丢失提示
  await apply()
  
  // 记录 dev migration (batch = -1)
  await drizzle.insert(adapter.tables.payload_migrations).values({
    name: 'dev',
    batch: -1,
  })
}
```

## 七、差异总结表

| 特性 | MongoDB | Postgres | SQLite |
|------|---------|----------|--------|
| **底层 ORM** | Mongoose | Drizzle | Drizzle |
| **Schema 概念** | 无 | 有 | 有 |
| **迁移生成** | 空白模板 | Drizzle Kit 快照对比 | Drizzle Kit 快照对比 |
| **迁移执行参数** | `{ payload, req, session }` | `{ db, payload, req }` | `{ db, payload, req }` |
| **SQL 执行方法** | 无 | `db.execute()` | `db.run()` |
| **ID 类型** | ObjectId (text) | serial/uuid | integer/uuid |
| **枚举实现** | 无 | 原生 ENUM 类型 | TEXT + check 约束 |
| **布尔类型** | 原生 boolean | 原生 boolean | INTEGER + mode |
| **JSON 类型** | 原生 | jsonb | TEXT + json mode |
| **时间戳** | 原生 Date | timestamp (可选时区) | TEXT + strftime |
| **UUID 默认值** | 无 | `gen_random_uuid()` | 应用层 `uuidv4/v7()` |
| **向量支持** | 无 | pgvector (vector/halfvec/sparsevec/bit) | 无 |
| **Schema 扩展** | 无 | postgis 等 | 无 |
| **事务支持** | session | Drizzle transaction | Drizzle transaction |

## 八、统一处理策略

### 8.1 三层抽象

1. **接口层** (`BaseDatabaseAdapter`)
   - 定义统一的方法签名
   - 所有适配器必须实现

2. **Drizzle 共享层** (`@payloadcms/drizzle`)
   - 抽象表定义 (`RawTable`, `RawColumn`)
   - 共享迁移执行逻辑
   - 共享 Schema 构建框架

3. **数据库特定层** (`db-postgres`, `db-sqlite`, `db-mongodb`)
   - `buildDrizzleTable` - 抽象列到具体列
   - `sanitizeStatements` - SQL 语句处理
   - `setColumnID` - ID 类型选择

### 8.2 关键设计模式

- **策略模式**: 不同数据库通过 `buildCreateMigration` 的参数 (`executeMethod`, `sanitizeStatements`) 定制行为
- **模板方法模式**: `buildRawSchema` 定义骨架，数据库特定步骤在 `buildDrizzleTable` 中实现
- **抽象工厂**: `createDatabaseAdapter` 提供默认实现，允许选择性覆盖
- **适配器模式**: Drizzle 适配器适配不同 SQL 方言

## 九、代码引用索引

| 功能模块 | 核心文件 |
|---------|---------|
| 统一接口定义 | `packages/payload/src/database/types.ts:17-170` |
| 适配器工厂 | `packages/payload/src/database/createDatabaseAdapter.ts:24-62` |
| 抽象 Schema 构建 | `packages/drizzle/src/schema/build.ts:71-798` |
| Postgres 表转换 | `packages/drizzle/src/postgres/schema/buildDrizzleTable.ts` |
| SQLite 表转换 | `packages/drizzle/src/sqlite/schema/buildDrizzleTable.ts` |
| Drizzle 迁移执行 | `packages/drizzle/src/migrate.ts:18-118` |
| 开发模式 Push | `packages/drizzle/src/utilities/pushDevSchema.ts:21-114` |
| 迁移文件生成 | `packages/drizzle/src/utilities/buildCreateMigration.ts:13-162` |
| MongoDB 适配器 | `packages/db-mongodb/src/index.ts:252-357` |
| Postgres 适配器 | `packages/db-postgres/src/index.ts:69-243` |
| SQLite 适配器 | `packages/db-sqlite/src/index.ts:67-245` |
| 预定义迁移加载 | `packages/payload/src/database/migrations/getPredefinedMigration.ts:18-88` |
