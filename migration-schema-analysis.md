# Payload CMS 数据库迁移与结构同步差异分析报告

## 一、概述

Payload CMS 支持五种数据库适配器：MongoDB、Postgres、SQLite、Vercel Postgres 和 D1 SQLite。本文档详细分析了这些适配器之间在数据库迁移和结构同步方面如何通过统一接口和抽象层来处理差异。

## 二、整体架构

### 2.1 统一接口定义

**文件**: `packages/payload/src/database/types.ts:17-170`

`BaseDatabaseAdapter` 接口定义了所有数据库适配器必须实现的统一契约：

```typescript
export interface BaseDatabaseAdapter {
  // 迁移相关方法（所有适配器必须实现或使用默认实现）
  migrate: (args?: { migrations?: Migration[] }) => Promise<void>
  migrateDown: () => Promise<void>
  migrateFresh: (args: { forceAcceptWarning?: boolean }) => Promise<void>
  migrateRefresh: () => Promise<void>
  migrateReset: () => Promise<void>
  migrateStatus: () => Promise<void>
  createMigration: CreateMigration
  
  // 通用元数据
  defaultIDType: 'number' | 'text'
  name: string
  packageName: string
  migrationDir: string
  
  // 生命周期方法
  init?: Init
  connect?: Connect
  destroy?: Destroy
  
  // 事务支持
  beginTransaction: BeginTransaction
  commitTransaction: CommitTransaction
  rollbackTransaction: RollbackTransaction
  
  // ... 其他 CRUD 方法
}
```

### 2.2 适配器工厂与默认实现

**文件**: `packages/payload/src/database/createDatabaseAdapter.ts:24-62`

`createDatabaseAdapter` 函数提供了默认实现，适配器可以选择性覆盖：

```typescript
export function createDatabaseAdapter<T extends BaseDatabaseAdapter>(
  args: MarkOptional<T, 
    | 'createMigration'      // 可选，有默认实现
    | 'migrate'              // 可选，有默认实现
    | 'migrateDown'          // 可选，有默认实现
    | 'migrateFresh'         // 可选，有默认实现（空操作）
    | 'migrateRefresh'       // 可选，有默认实现
    | 'migrateReset'         // 可选，有默认实现
    | 'migrateStatus'        // 可选，有默认实现
    | 'migrationDir'         // 可选，默认 'migrations'
    | 'allowIDOnCreate'
    | 'bulkOperationsSingleTransaction'
    | 'updateJobs'
  >
): T {
  return {
    // 默认 'null' transaction 函数（无操作）
    beginTransaction,
    commitTransaction,
    rollbackTransaction,
    
    // 默认迁移实现
    createMigration,        // packages/payload/src/database/migrations/createMigration.ts
    migrate,                // packages/payload/src/database/migrations/migrate.ts
    migrateDown,
    migrateRefresh,
    migrateReset,
    migrateStatus,
    migrateFresh: () => Promise.resolve(null),
    
    // 默认值
    migrationDir: args.migrationDir || 'migrations',
    bulkOperationsSingleTransaction: args.bulkOperationsSingleTransaction ?? false,
    updateJobs: defaultUpdateJobs,
    
    ...args,  // 适配器可以覆盖默认实现
  } as T
}
```

### 2.3 适配器体系分类

```
┌─────────────────────────────────────────────────────────────┐
│                  BaseDatabaseAdapter 接口                    │
│  (packages/payload/src/database/types.ts)                    │
│  migrate | migrateDown | migrateFresh | migrateRefresh       │
│  migrateReset | migrateStatus | createMigration              │
│  init | connect | beginTransaction | commitTransaction        │
└──────────────────────────┬──────────────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
┌─────────────────────┐         ┌─────────────────────────────┐
│  MongoDB Adapter    │         │  Drizzle Adapter (SQL 系)   │
│  @payloadcms/db-    │         │  @payloadcms/drizzle         │
│  mongodb            │         │  共享层：抽象 Schema + 共享逻辑 │
└─────────────────────┘         └────────────┬────────────────┘
                                             │
                                ┌────────────┼────────────┐
                                ▼            ▼            ▼
                      ┌─────────────┐ ┌────────────┐ ┌────────────┐
                      │   Postgres  │ │   SQLite   │ │  (派生)     │
                      │   适配器     │ │   适配器    │ │            │
                      └──────┬──────┘ └──────┬─────┘ └─────┬──────┘
                             │               │              │
                             ▼               ▼              ▼
                    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                    │ Vercel Postgres│ │   D1 SQLite  │ │(更多适配器) │
                    │   (Postgres  │ │  (SQLite 派生)│ │              │
                    │    派生)     │ │              │ │              │
                    └──────────────┘ └──────────────┘ └──────────────┘
```

## 三、两类适配器体系详解

### 3.1 文档型数据库：MongoDB 适配器

**包**: `@payloadcms/db-mongodb`
**底层**: Mongoose
**特点**: Schema-less，无结构迁移概念

**核心文件**:
- `packages/db-mongodb/src/index.ts` - 适配器入口
- `packages/db-mongodb/src/init.ts` - 初始化流程
- `packages/db-mongodb/src/models/buildSchema.ts` - Schema 构建
- `packages/db-mongodb/src/createMigration.ts` - 迁移文件生成

### 3.2 关系型数据库：Drizzle 系适配器

**底层**: Drizzle ORM
**统一层**: `@payloadcms/drizzle`

| 适配器 | 包 | 驱动 | 派生关系 |
|--------|-----|------|---------|
| Postgres | `@payloadcms/db-postgres` | `pg` (node-postgres) | 直接实现 |
| SQLite | `@payloadcms/db-sqlite` | `@libsql/client` | 直接实现 |
| Vercel Postgres | `@payloadcms/db-vercel-postgres` | `@vercel/postgres` | Postgres 派生 |
| D1 SQLite | `@payloadcms/db-d1-sqlite` | `drizzle-orm/d1` | SQLite 派生 |

## 四、迁移机制：差异与统一

### 4.1 迁移执行流程对比

#### 4.1.1 MongoDB 迁移流程

**文件**: `packages/db-mongodb/src/createMigration.ts:22-62`

MongoDB 迁移特点：
- 生成简单的 TypeScript 模板（无 SQL）
- 使用 `session` 进行 MongoDB 事务
- 用户手写数据操作逻辑

```typescript
export const createMigration: CreateMigration = async function createMigration({
  file, migrationName, payload, skipEmpty,
}) {
  // 加载预定义迁移
  const predefinedMigration = await getPredefinedMigration({ dirname, file, migrationName, payload })
  
  // 生成模板
  const migrationFileContent = migrationTemplate(predefinedMigration)
  // 写入文件...
}

const migrationTemplate = ({ downSQL, imports, upSQL }: MigrationTemplateArgs): string => `
import { MigrateDownArgs, MigrateUpArgs } from '@payloadcms/db-mongodb'
${imports ?? ''}

export async function up({ payload, req, session }: MigrateUpArgs): Promise<void> {
  ${upSQL ?? `  // 直接操作 Mongoose 模型`}
}

export async function down({ payload, req, session }: MigrateDownArgs): Promise<void> {
  ${downSQL ?? `  // 回滚逻辑`}
}
`
```

#### 4.1.2 Drizzle 系共享迁移逻辑

**文件**: `packages/drizzle/src/migrate.ts:18-118`

所有 SQL 系适配器共享此迁移执行逻辑：

```typescript
export const migrate: DrizzleAdapter['migrate'] = async function migrate(
  this: DrizzleAdapter,
  args,
): Promise<void> {
  const { payload } = this
  const migrationFiles = args?.migrations || (await readMigrationFiles({ payload }))
  
  // Postgres 特有：创建扩展
  if ('createExtensions' in this && typeof this.createExtensions === 'function') {
    await this.createExtensions()
  }
  
  // 检查 dev migration (batch = -1)
  const hasMigrationTable = await migrationTableExists(this)
  if (hasMigrationTable) {
    const { docs: migrationsInDB } = await payload.find({ collection: 'payload-migrations' })
    if (migrationsInDB.find((m) => m.batch === -1)) {
      // 警告用户：dev 模式后运行迁移可能导致数据丢失
    }
  }
  
  // 执行每个 migration
  for (const migration of migrationFiles) {
    const alreadyRan = migrationsInDB.find((existing) => existing.name === migration.name)
    if (alreadyRan) continue
    
    await runMigrationFile(payload, migration, newBatch)
  }
}
```

#### 4.1.3 迁移参数差异

| 适配器 | up/down 参数 | 参数来源 |
|--------|-------------|---------|
| MongoDB | `{ payload, req, session }` | MongoDB session |
| Postgres | `{ db, payload, req }` | `NodePgDatabase` 事务 |
| SQLite | `{ db, payload, req }` | `LibSQLDatabase` 事务 |
| Vercel Postgres | `{ db, payload, req }` | `NodePgDatabase` (VercelPool) |
| D1 SQLite | `{ db, payload, req }` | `DrizzleD1Database` |

### 4.2 迁移文件生成：策略模式

**文件**: `packages/drizzle/src/utilities/buildCreateMigration.ts:13-162`

`buildCreateMigration` 使用**策略模式**，通过参数定制不同数据库的行为：

```typescript
export const buildCreateMigration = ({
  executeMethod,      // 策略1: SQL 执行方法名
  filename,
  sanitizeStatements, // 策略2: SQL 语句清洗函数
}: {
  executeMethod: string
  filename: string
  sanitizeStatements: (args: { sqlExecute: string; statements: string[] }) => string
}): CreateMigration => {
  return async function createMigration(
    this: DrizzleAdapter,
    { file, forceAcceptWarning, migrationName, payload, skipEmpty },
  ) {
    // 使用 Drizzle Kit 生成快照
    const drizzleJsonAfter = await generateDrizzleJson(this.schema)
    
    // 对比前后快照生成 SQL
    const sqlStatementsUp = await generateMigration(drizzleJsonBefore, drizzleJsonAfter)
    const sqlStatementsDown = await generateMigration(drizzleJsonAfter, drizzleJsonBefore)
    
    // 使用策略函数清洗语句
    if (sqlStatementsUp?.length) {
      const sqlExecute = `await db.${executeMethod}(` + 'sql`'
      upSQL = sanitizeStatements({ sqlExecute, statements: sqlStatementsUp })
    }
    // ...
  }
}
```

#### 4.2.1 Postgres 的策略配置

**文件**: `packages/db-postgres/src/index.ts:100-108`

```typescript
const executeMethod = 'execute'

const sanitizeStatements = ({ sqlExecute, statements }): string => {
  // Postgres: 多行合并，无需转义
  return `${sqlExecute}\n ${statements.join('\n')}\`)`
}
```

#### 4.2.2 SQLite 的策略配置

**文件**: `packages/db-sqlite/src/index.ts:112-123`

```typescript
const executeMethod = 'run'  // 区别于 Postgres

const sanitizeStatements = ({ sqlExecute, statements }) => {
  // SQLite: 独立语句，需要转义反引号
  return statements
    .map((statement) => `${sqlExecute}${statement?.replaceAll('`', '\\`')}\`)`)
    .join('\n')
}
```

#### 4.2.3 D1 SQLite 的策略配置

**文件**: `packages/db-d1-sqlite/src/index.ts:88-100`

与 SQLite 相同的策略（因为都是 SQLite 方言）：

```typescript
const executeMethod = 'run'

const sanitizeStatements = ({ sqlExecute, statements }) => {
  return statements
    .map((statement) => `${sqlExecute}${statement?.replaceAll('`', '\\`')}\`)`)
    .join('\n')
}
```

#### 4.2.4 Vercel Postgres 的策略配置

**文件**: `packages/db-vercel-postgres/src/index.ts:96-103`

与 Postgres 相同的策略：

```typescript
const executeMethod = 'execute'
const sanitizeStatements = ({ sqlExecute, statements }) => 
  `${sqlExecute}\n ${statements.join('\n')}\`)`
```

### 4.3 迁移语句执行差异

#### 4.3.1 D1 SQLite 的特殊执行封装

**文件**: `packages/db-d1-sqlite/src/execute.ts:40-67`

D1 SQLite 需要特殊的结果映射（D1 API → LibSQL 兼容格式）：

```typescript
export const execute: Execute<any> = function execute({ db, drizzle, raw, sql: statement }) {
  const executeFrom: any = (db ?? drizzle)!
  
  const mapToLibSql = (query: SQLiteRaw<D1Result<unknown>>): any => {
    const execute = query.execute
    query.execute = async () => {
      const result: D1Result = await execute()
      // 映射 D1 结果到 LibSQL 格式
      const resultLibSQL = {
        columns: undefined,
        columnTypes: undefined,
        lastInsertRowid: BigInt(result.meta.last_row_id),
        rows: result.results as any[],
        rowsAffected: result.meta.rows_written,
      }
      return Object.assign(result, resultLibSQL)
    }
    return query
  }
  
  if (raw) {
    return mapToLibSql(executeFrom.run(sql.raw(raw)))
  }
  // ...
}
```

### 4.4 预定义迁移系统

**文件**: `packages/payload/src/database/migrations/getPredefinedMigration.ts:18-88`

统一的预定义迁移加载机制：

```typescript
export const getPredefinedMigration = async ({
  dirname, file, migrationName, payload,
}): Promise<MigrationTemplateArgs> => {
  const importPath = file ?? migrationName
  
  // 路径 1: @payloadcms/db-* 适配器包 - 直接从 predefinedMigrations 文件夹加载
  if (importPath?.startsWith('@payloadcms/db-')) {
    const migrationName = importPath.split('/').slice(2).join('/')
    let cleanPath = path.join(dirname, `./predefinedMigrations/${migrationName}`)
    // 支持 .mjs, .js, .ts
    const { downSQL, dynamic, imports, upSQL } = await dynamicImport<MigrationTemplateArgs>(cleanPath)
    return { downSQL, dynamic, imports, upSQL }
  }
  
  // 路径 2: 其他包或绝对路径 - 使用动态导入
  else if (importPath) {
    const { downSQL, dynamic, imports, upSQL } = await dynamicImport<MigrationTemplateArgs>(importPath)
    return { downSQL, dynamic, imports, upSQL }
  }
  
  return {}
}
```

**各适配器预定义迁移**:

| 适配器 | 预定义迁移 | 用途 |
|--------|-----------|------|
| MongoDB | `relationships-v2-v3.ts` | v2 → v3 关系字段迁移 |
| MongoDB | `versions-v1-v2.ts` | 版本系统迁移 |
| Postgres | `relationships-v2-v3.ts` | 同上 |
| Postgres | `blocks-as-json.ts` | Block 字段 JSON 化 |
| SQLite | `blocks-as-json.ts` | 同上 |
| Vercel Postgres | `relationships-v2-v3.ts`, `blocks-as-json.ts` | 继承自 Postgres |
| D1 SQLite | `blocks-as-json.ts` | 继承自 SQLite |

## 五、结构同步（Schema Synchronization）

### 5.1 Drizzle 系：三层抽象架构

```
Payload Config (集合/字段定义)
            │
            ▼
┌─────────────────────────────────────┐
│  第一层：抽象 Schema 构建             │
│  buildRawSchema()                    │
│  packages/drizzle/src/schema/build.ts │
│  输出: RawTable / RawColumn          │
│  (数据库无关的抽象表示)               │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  第二层：数据库特定转换               │
│  Postgres: buildDrizzleTable()      │
│  SQLite: buildDrizzleTable()        │
│  输出: Drizzle Table / Column        │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  第三层：执行层                      │
│  开发模式: pushDevSchema()          │
│  生产模式: migrate()                │
└─────────────────────────────────────┘
```

### 5.2 第一层：抽象 Schema 构建（数据库无关）

**文件**: `packages/drizzle/src/schema/build.ts:71-798`

统一的 Schema 构建逻辑，输出 `RawTable` 抽象结构：

```typescript
export const buildTable = ({
  adapter, fields, setColumnID, tableName, timestamps, versions,
}: Args): Result => {
  const columns: Record<string, RawColumn> = baseColumns
  const indexes: Record<string, RawIndex> = baseIndexes
  
  // 1. 构建基础列
  const idColType: IDType = setColumnID({ adapter, columns, fields })
  
  // 2. 遍历字段，填充 columns/indexes/relationships
  const { hasLocalizedField, hasLocalizedManyNumberField, ... } = traverseFields({
    adapter, columns, indexes, fields, /* ... */
  })
  
  // 3. 处理多语言字段 → 生成 _locales 表
  if (hasLocalizedField || localizedRelations.size) {
    const localeTableName = `${tableName}${adapter.localesSuffix}`
    adapter.rawTables[localeTableName] = localesTable
  }
  
  // 4. 处理多值字段 → 生成 _texts / _numbers 表
  if (hasManyTextField) {
    const textsTableName = `${rootTableName}_texts`
    adapter.rawTables[textsTableName] = textsTable
  }
  
  // 5. 处理关系字段 → 生成 _rels 表
  if (relationships.size) {
    const relationshipsTableName = `${tableName}${adapter.relationshipsSuffix}`
    adapter.rawTables[relationshipsTableName] = relationshipsTable
  }
  
  // 6. 写入 adapter.rawTables
  adapter.rawTables[tableName] = table
}
```

**抽象类型定义** (`packages/drizzle/src/types.ts`):

```typescript
// 抽象 SQL 表
export type RawTable = {
  name: string
  columns: Record<string, RawColumn>
  foreignKeys?: Record<string, RawForeignKey>
  indexes?: Record<string, RawIndex>
}

// 抽象列（包含数据库特定注释）
export type RawColumn =
  | ({ type: 'boolean' | 'geometry' | 'jsonb' | 'numeric' | 'serial' | 'text' | 'varchar' } & BaseRawColumn)
  | BinaryVecRawColumn   // pgvector 特有
  | EnumRawColumn        // Postgres: 原生枚举; SQLite: TEXT + check
  | HalfVecRawColumn     // pgvector 特有
  | IntegerRawColumn
  | SparseVecRawColumn   // pgvector 特有
  | TimestampRawColumn   // Postgres: 原生 timestamp; SQLite: TEXT
  | UUIDRawColumn        // Postgres: 原生 uuid; SQLite: TEXT
  | VectorRawColumn      // pgvector 特有
```

### 5.3 第二层：数据库特定转换

#### 5.3.1 Postgres 转换

**文件**: `packages/drizzle/src/postgres/schema/buildDrizzleTable.ts`

```typescript
const rawColumnBuilderMap: Partial<Record<RawColumn['type'], any>> = {
  boolean,
  geometry: geometryColumn,  // PostGIS
  integer,
  jsonb,
  numeric,
  serial,
  text,
  uuid,
  varchar,
  vector,      // pgvector
  halfvec,     // pgvector
  sparsevec,   // pgvector
  bit,         // pgvector
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

// 时间戳处理 - 原生类型
case 'timestamp':
  let builder = timestamp(column.name, {
    mode: column.mode,
    precision: column.precision,
    withTimezone: column.withTimezone,  // 支持时区
  })
  if (column.defaultNow) {
    builder = builder.defaultNow()  // 数据库级别默认值
  }
  break

// UUID 处理 - 原生类型 + 数据库级别默认值
case 'uuid':
  let builder = uuid(column.name)
  if (column.defaultRandom) {
    builder = builder.defaultRandom()  // gen_random_uuid()
  }
  if (column.defaultV7) {
    builder = builder.$defaultFn(() => uuidv7())  // 应用层
  }
  break
```

#### 5.3.2 SQLite 转换

**文件**: `packages/drizzle/src/sqlite/schema/buildDrizzleTable.ts`

```typescript
const rawColumnBuilderMap: Partial<Record<RawColumn['type'], any>> = {
  integer,
  numeric,
  text,
  // 无 vector/halfvec/sparsevec/bit (SQLite 不支持)
}

// 布尔值处理 - 使用 INTEGER + mode
case 'boolean':
  columns[key] = integer(column.name, { mode: 'boolean' })
  break

// 枚举处理 - 使用 TEXT + check 约束
case 'enum':
  if ('locale' in column) {
    columns[key] = text(column.name, { enum: locales as [string, ...string[]] })
  } else {
    columns[key] = text(column.name, { enum: column.options as [string, ...string[]] })
  }
  break

// JSON/Geometry 处理 - 使用 TEXT + json mode
case 'geometry':
case 'jsonb':
  columns[key] = text(column.name, { mode: 'json' })
  break

// 时间戳处理 - TEXT + strftime
case 'timestamp':
  let builder = text(column.name)
  if (column.defaultNow) {
    builder = builder.default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`)
  }
  break

// UUID 处理 - TEXT + 应用层默认值
case 'uuid':
  let builder = text(column.name, { length: 36 })
  if (column.defaultRandom) {
    builder = builder.$defaultFn(() => uuidv4())  // 应用层生成
  }
  if (column.defaultV7) {
    builder = builder.$defaultFn(() => uuidv7())  // 应用层生成
  }
  break
```

### 5.4 第三层：初始化流程

#### 5.4.1 Postgres 初始化

**文件**: `packages/drizzle/src/postgres/init.ts:11-45`

```typescript
export const init: Init = async function init(this: BasePostgresAdapter) {
  // 1. 清空旧数据
  this.rawRelations = {}
  this.rawTables = {}
  
  // 2. 构建抽象 Schema
  buildRawSchema({ adapter: this, setColumnID })
  
  // 3. 执行 beforeSchemaInit 钩子
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  // 4. Postgres 特有: 创建 locale 枚举
  if (this.payload.config.localization) {
    this.enums.enum__locales = this.pgSchema.enum(
      '_locales',
      this.payload.config.localization.locales.map(({ code }) => code)
    )
  }
  
  // 5. 转换抽象表为 Drizzle 表
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter: this, rawTable: this.rawTables[tableName] })
  }
  
  // 6. 构建关系
  buildDrizzleRelations({ adapter: this })
  
  // 7. 执行 afterSchemaInit 钩子
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  // 8. 组装完整 schema (包含枚举)
  this.schema = {
    pgSchema: this.pgSchema,
    ...this.tables,
    ...this.relations,
    ...this.enums,  // Postgres 特有
  }
}
```

#### 5.4.2 SQLite 初始化

**文件**: `packages/drizzle/src/sqlite/init.ts:12-45`

```typescript
export const init: Init = async function init(this: BaseSQLiteAdapter) {
  let locales: string[] | undefined
  
  // 1. 准备 locales（用于 TEXT enum 约束）
  if (this.payload.config.localization) {
    locales = this.payload.config.localization.locales.map(({ code }) => code)
  }
  
  // 2. 构建抽象 Schema
  buildRawSchema({ adapter, setColumnID })
  
  // 3. 执行 beforeSchemaInit 钩子
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  // 4. 转换抽象表（传入 locales 用于 enum）
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter, locales, rawTable: this.rawTables[tableName] })
  }
  
  // 5. 构建关系
  buildDrizzleRelations({ adapter })
  
  // 6. 执行 afterSchemaInit 钩子
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  // 7. 组装 schema（无枚举）
  this.schema = {
    ...this.tables,
    ...this.relations,
  }
}
```

### 5.5 MongoDB 结构同步链路（Schema-less 但有 Mongoose Schema）

**关键文件**:
- `packages/db-mongodb/src/init.ts:21-118` - 初始化流程
- `packages/db-mongodb/src/models/buildCollectionSchema.ts:9-49` - 集合 Schema 构建
- `packages/db-mongodb/src/models/buildSchema.ts:130-943` - 字段 Schema 生成器

#### 5.5.1 MongoDB 初始化流程

```typescript
export const init: Init = async function init(this: MongooseAdapter) {
  // 1. 创建 scoped connection（未打开）
  this.connection ??= mongoose.createConnection()
  
  if (this.afterCreateConnection) {
    await this.afterCreateConnection(this)
  }
  
  // 2. 遍历集合配置，构建 Mongoose Schema
  this.payload.config.collections.forEach((collection: SanitizedCollectionConfig) => {
    const schemaOptions = this.collectionsSchemaOptions?.[collection.slug]
    
    // 3. 构建集合 Schema
    const schema = buildCollectionSchema(collection, this.payload, schemaOptions)
    
    // 4. 如果启用版本控制，构建版本集合 Schema
    if (collection.versions) {
      const versionCollectionFields = buildVersionCollectionFields(this.payload.config, collection)
      const versionSchema = buildSchema({
        buildSchemaOptions: {
          disableUnique: true,
          draftsEnabled: true,
          indexSortableFields: this.payload.config.indexSortableFields,
        },
        configFields: versionCollectionFields,
        payload: this.payload,
      })
      // 注册版本模型
      this.versions[collection.slug] = this.connection.model(versionModelName, versionSchema, versionCollectionName)
    }
    
    // 5. 注册主集合模型
    this.collections[collection.slug] = this.connection.model(modelName, schema, collectionName)
  })
  
  // 6. 构建全局模型
  this.globals = buildGlobalModel(this)
  
  // 7. 处理全局的版本控制
  this.payload.config.globals.forEach((global) => {
    if (global.versions) {
      // 构建版本 Schema...
    }
  })
}
```

#### 5.5.2 集合 Schema 构建

**文件**: `packages/db-mongodb/src/models/buildCollectionSchema.ts:9-49`

```typescript
export const buildCollectionSchema = (
  collection: SanitizedCollectionConfig,
  payload: Payload,
  schemaOptions = {},
): Schema => {
  // 1. 构建基础 Schema
  const schema = buildSchema({
    buildSchemaOptions: {
      draftsEnabled: Boolean(
        typeof collection?.versions === 'object' && collection.versions.drafts,
      ),
      indexSortableFields: payload.config.indexSortableFields,
      options: {
        minimize: false,
        timestamps: collection.timestamps !== false,
        ...schemaOptions,
      },
    },
    compoundIndexes: collection.sanitizedIndexes,
    configFields: collection.fields,
    flattenedFields: collection.flattenedFields,
    payload,
  })
  
  // 2. 上传文件的复合索引
  if (Array.isArray(collection.upload.filenameCompoundIndex)) {
    schema.index(indexDefinition, { unique: true })
  }
  
  // 3. 注册插件（分页、查询构建）
  schema
    .plugin(paginate, { useEstimatedCount: true })
    .plugin(getBuildQueryPlugin({ collectionSlug: collection.slug }))
  
  return schema
}
```

#### 5.5.3 字段 Schema 生成器（核心）

**文件**: `packages/db-mongodb/src/models/buildSchema.ts:130-206`

```typescript
export const buildSchema = (args: {
  buildSchemaOptions: BuildSchemaOptions
  compoundIndexes?: SanitizedCompoundIndex[]
  configFields: Field[]
  flattenedFields?: FlattenedField[]
  parentIsLocalized?: boolean
  payload: Payload
}): Schema => {
  // 1. 处理自定义 ID
  let fields = {}
  if (!allowIDField) {
    const idField = fieldsToSearch.find((field) => fieldAffectsData(field) && field.name === 'id')
    if (idField) {
      fields = {
        _id: idField.type === 'number'
          ? payload.db.useBigIntForNumberIDs
            ? mongoose.Schema.Types.BigInt
            : Number
          : String,
      }
    }
  }
  
  const schema = new mongoose.Schema(fields, options as any)
  
  // 2. 遍历字段，使用对应生成器添加到 Schema
  schemaFields.forEach((field) => {
    if (!fieldIsPresentationalOnly(field)) {
      const addFieldSchema = getSchemaGenerator(field.type)
      if (addFieldSchema) {
        addFieldSchema(field, schema, payload, buildSchemaOptions, parentIsLocalized ?? false)
      }
    }
  })
  
  // 3. 处理复合索引
  if (args.compoundIndexes) {
    for (const index of args.compoundIndexes) {
      const indexDefinition: Record<string, 1> = {}
      for (const field of index.fields) {
        if (field.pathHasLocalized && payload.config.localization) {
          for (const locale of payload.config.localization.locales) {
            indexDefinition[field.localizedPath.replace('<locale>', locale.code)] = 1
          }
        } else {
          indexDefinition[field.path] = 1
        }
      }
      schema.index(indexDefinition, { unique: args.buildSchemaOptions.disableUnique ? false : index.unique })
    }
  }
  
  return schema
}
```

#### 5.5.4 字段类型映射表

```typescript
const fieldToSchemaMap = {
  array,        // ArrayField → mongoose Schema type: []
  blocks,       // BlocksField → discriminator schema
  checkbox,     // CheckboxField → Boolean
  code,         // CodeField → String
  collapsible,  // CollapsibleField → 递归处理子字段
  date,         // DateField → Date
  email,        // EmailField → String
  group,        // GroupField → 嵌套 Schema 或递归
  json,         // JSONField → Mixed
  number,       // NumberField → Number (或 [Number] if hasMany)
  point,        // PointField → GeoJSON (2dsphere 索引)
  radio,        // RadioField → String + enum
  relationship, // RelationshipField → ObjectId 或 Mixed
  richText,     // RichTextField → Mixed
  row,          // RowField → 递归处理子字段
  select,       // SelectField → String + enum
  tabs,         // TabsField → 递归处理
  text,         // TextField → String (或 [String] if hasMany)
  textarea,     // TextareaField → String
  upload,       // UploadField → ObjectId 或 Mixed
}
```

#### 5.5.5 多语言字段处理

**文件**: `packages/db-mongodb/src/models/buildSchema.ts:103-128`

```typescript
const localizeSchema = (
  entity: NonPresentationalField | Tab,
  schema: SchemaTypeOptions<any>,
  localization: false | SanitizedLocalizationConfig,
  parentIsLocalized: boolean,
) => {
  if (
    fieldShouldBeLocalized({ field: entity, parentIsLocalized }) &&
    localization &&
    Array.isArray(localization.locales)
  ) {
    // 多语言字段: 转换为 { en: schema, zh: schema, ... }
    return {
      type: localization.localeCodes.reduce(
        (localeSchema, locale) => ({
          ...localeSchema,
          [locale]: schema,
        }),
        { _id: false },
      ),
      localized: true,
    }
  }
  return schema
}
```

**对比 SQL 系**:
- **MongoDB**: 同一文档内嵌多语言字段 `{ title: { en: "Hello", zh: "你好" } }`
- **SQL 系**: 独立 `_locales` 表，外键关联主表

## 六、开发模式 Schema Push

### 6.1 统一 Push 机制

**文件**: `packages/drizzle/src/utilities/pushDevSchema.ts:21-114`

```typescript
export const pushDevSchema = async (adapter: DrizzleAdapter) => {
  // 1. 检查 schema 是否变化（优化热重载）
  const equal = dequal(previousSchema, {
    localeCodes,
    rawTables: adapter.rawTables,
  })
  if (equal) return  // 无变化，跳过
  
  // 2. 使用 Drizzle Kit 的 pushSchema
  const { pushSchema } = adapter.requireDrizzleKit()
  
  // 3. 调用 pushSchema（数据库特定参数）
  const { extensions = {}, tablesFilter } = adapter as BasePostgresAdapter
  const { apply, hasDataLoss, warnings } = await pushSchema(
    adapter.schema,
    adapter.drizzle,
    adapter.schemaName ? [adapter.schemaName] : undefined,  // Postgres 特有
    tablesFilter,
    extensions.postgis ? ['postgis'] : undefined,  // Postgres 扩展
  )
  
  // 4. 处理警告和数据丢失提示
  if (warnings.length) {
    // 交互式确认
  }
  
  // 5. 应用变更
  await apply()
  
  // 6. 记录 dev migration (batch = -1)
  // 用于区分 dev 模式推送和生产迁移
  await drizzle.insert(adapter.tables.payload_migrations).values({
    name: 'dev',
    batch: -1,
  })
}
```

## 七、适配器差异总览

### 7.1 功能差异矩阵

| 特性 | MongoDB | Postgres | Vercel Postgres | SQLite | D1 SQLite |
|------|---------|----------|-----------------|--------|-----------|
| **底层 ORM** | Mongoose | Drizzle | Drizzle | Drizzle | Drizzle |
| **Schema 概念** | Mongoose Schema | SQL DDL | SQL DDL | SQL DDL | SQL DDL |
| **迁移生成** | 空白模板 | Drizzle Kit 快照 | Drizzle Kit 快照 | Drizzle Kit 快照 | Drizzle Kit 快照 |
| **迁移执行参数** | `{ payload, req, session }` | `{ db, payload, req }` | `{ db, payload, req }` | `{ db, payload, req }` | `{ db, payload, req }` |
| **SQL 执行方法** | 无 | `db.execute()` | `db.execute()` | `db.run()` | `db.run()` |
| **ID 类型** | ObjectId (text) | serial/uuid | serial/uuid | integer/uuid | integer/uuid |
| **枚举实现** | String + enum | 原生 ENUM | 原生 ENUM | TEXT + check | TEXT + check |
| **布尔类型** | 原生 boolean | 原生 boolean | 原生 boolean | INTEGER + mode | INTEGER + mode |
| **JSON 类型** | 原生 Mixed | jsonb | jsonb | TEXT + json mode | TEXT + json mode |
| **时间戳** | 原生 Date | timestamp (TZ可选) | timestamp (TZ可选) | TEXT + strftime | TEXT + strftime |
| **UUID 默认值** | 无 | `gen_random_uuid()` | `gen_random_uuid()` | 应用层 uuidv4/7 | 应用层 uuidv4/7 |
| **向量支持** | 无 | pgvector | pgvector | 无 | 无 |
| **Schema 扩展** | 无 | postgis 等 | postgis 等 | 无 | 无 |
| **事务支持** | session | Drizzle transaction | Drizzle transaction | Drizzle transaction | Drizzle transaction |

### 7.2 Postgres 派生适配器：Vercel Postgres 差异

**文件**: `packages/db-vercel-postgres/src/connect.ts:12-115`

```typescript
export const connect: Connect = async function connect(this: VercelPostgresAdapter, options) {
  const connectionString = this.poolOptions?.connectionString ?? process.env.POSTGRES_URL
  
  // 差异1: 本地开发自动降级到 node-postgres
  if (
    !this.forceUseVercelPostgres &&
    connectionString &&
    ['127.0.0.1', 'localhost'].includes(new URL(connectionString).hostname)
  ) {
    client = new pg.Pool(this.poolOptions ?? { connectionString })
  } else {
    // 生产环境使用 Vercel Postgres
    client = this.poolOptions ? new VercelPool(this.poolOptions) : sql
  }
  
  // 差异2: 只读副本支持 (Vercel Postgres 特有)
  if (this.readReplicaOptions) {
    this.primaryDrizzle = this.drizzle as any
    const readReplicas = this.readReplicaOptions.map((connectionString) => {
      const pool = new VercelPool(options)
      return drizzle({ client: pool as unknown as pg.Pool, logger, schema: this.schema })
    })
    const myReplicas = withReplicas(this.drizzle, readReplicas as any)
    this.drizzle = myReplicas
  }
}
```

### 7.3 SQLite 派生适配器：D1 SQLite 差异

**文件**: `packages/db-d1-sqlite/src/index.ts:102-205`

```typescript
const adapter = createDatabaseAdapter<SQLiteD1Adapter>({
  // 差异1: D1 binding
  binding: args.binding,  // Cloudflare Workers env.DB
  
  // 差异2: 有限制的绑定参数
  limitedBoundParameters: true,  // D1 限制
  
  // 差异3: 自定义 execute（结果映射）
  execute,  // packages/db-d1-sqlite/src/execute.ts
  
  // 差异4: 只读副本策略
  readReplicas: args.readReplicas,  // 'first-primary'
  
  // 差异5: upsert 用 updateOne 实现
  upsert: updateOne,
})
```

**连接差异** (`packages/db-d1-sqlite/src/connect.ts:9-73`):

```typescript
export const connect: Connect = async function connect(this: SQLiteD1Adapter, options) {
  let binding = this.binding
  
  // 只读副本支持
  if (readReplicas && readReplicas === 'first-primary') {
    binding = this.binding.withSession('first-primary')
  }
  
  // 使用 D1 drizzle 驱动
  this.drizzle = drizzle(binding, { logger, schema: this.schema })
  this.client = this.drizzle.$client as any
}
```

## 八、统一接口如何收敛差异

### 8.1 迁移方法：默认实现 + 选择性覆盖

```
BaseDatabaseAdapter 接口
├─ migrate()
│   ├─ 默认实现: packages/payload/src/database/migrations/migrate.ts
│   │   (读取文件，顺序执行)
│   │
│   ├─ MongoDB: 使用默认实现
│   │   (因为迁移文件是用户手写的)
│   │
│   └─ Drizzle 系: 覆盖实现
│       packages/drizzle/src/migrate.ts
│       (检查 dev migration, 创建扩展等)
│
├─ createMigration()
│   ├─ 默认实现: packages/payload/src/database/migrations/createMigration.ts
│   │   (MongoDB 使用: 生成空白模板)
│   │
│   └─ Drizzle 系: 通过 buildCreateMigration 工厂创建
│       (传入 executeMethod 和 sanitizeStatements 策略)
│
└─ migrateFresh/migrateRefresh/migrateReset/migrateStatus
    ├─ 默认实现 (部分为空操作)
    └─ Drizzle 系: 覆盖实现
```

### 8.2 结构同步：抽象层 + 数据库特定转换

```
Payload Config (用户定义)
         │
         ▼
┌─────────────────────────────────────────────────┐
│  统一入口: init() 方法                            │
│  BaseDatabaseAdapter.init?: Init                 │
└───────────────────┬─────────────────────────────┘
                    │
    ┌───────────────┴───────────────┐
    │                               │
    ▼                               ▼
┌─────────────────┐       ┌───────────────────────────┐
│  MongoDB init() │       │  Drizzle 系 init()        │
│  构建 Mongoose   │       │  构建抽象 RawTable        │
│  Schema + Model │       │  → 数据库特定转换        │
└─────────────────┘       └────────────┬──────────────┘
                                       │
                            ┌──────────┴──────────┐
                            ▼                     ▼
                    ┌─────────────┐       ┌─────────────┐
                    │ Postgres    │       │ SQLite      │
                    │ buildDrizzle│       │ buildDrizzle│
                    │ Table()     │       │ Table()     │
                    │ 原生类型     │       │ 兼容性映射   │
                    └─────────────┘       └─────────────┘
```

### 8.3 关键设计模式

| 设计模式 | 应用场景 | 代码位置 |
|---------|---------|---------|
| **策略模式** | 迁移语句生成 (`executeMethod`, `sanitizeStatements`) | `packages/drizzle/src/utilities/buildCreateMigration.ts` |
| **模板方法模式** | Schema 构建骨架 (`buildRawSchema`) | `packages/drizzle/src/schema/build.ts` |
| **抽象工厂** | 适配器创建 + 默认实现 | `packages/payload/src/database/createDatabaseAdapter.ts` |
| **适配器模式** | Drizzle 适配不同 SQL 方言 | `packages/drizzle/src/postgres/`, `packages/drizzle/src/sqlite/` |
| **桥接模式** | 抽象 Schema (`RawTable`) 与实现分离 | `packages/drizzle/src/types.ts` |

## 九、完整代码引用索引

| 功能模块 | MongoDB | Postgres | SQLite | Vercel Postgres | D1 SQLite | 共享层 |
|---------|---------|----------|--------|-----------------|-----------|--------|
| **适配器入口** | `db-mongodb/src/index.ts:252-357` | `db-postgres/src/index.ts:69-243` | `db-sqlite/src/index.ts:67-245` | `db-vercel-postgres/src/index.ts:69-235` | `db-d1-sqlite/src/index.ts:65-222` | - |
| **初始化流程** | `db-mongodb/src/init.ts:21-118` | `drizzle/src/postgres/init.ts:11-45` | `drizzle/src/sqlite/init.ts:12-45` | 继承 Postgres | 继承 SQLite | - |
| **连接逻辑** | `db-mongodb/src/connect.ts` | `db-postgres/src/connect.ts:47-133` | `db-sqlite/src/connect.ts:10-76` | `db-vercel-postgres/src/connect.ts:12-115` | `db-d1-sqlite/src/connect.ts:9-73` | - |
| **Schema 构建** | `db-mongodb/src/models/buildSchema.ts:130-943` | `drizzle/src/postgres/schema/buildDrizzleTable.ts` | `drizzle/src/sqlite/schema/buildDrizzleTable.ts` | 继承 Postgres | 继承 SQLite | `drizzle/src/schema/build.ts:71-798` |
| **集合 Schema** | `db-mongodb/src/models/buildCollectionSchema.ts:9-49` | - | - | - | - | - |
| **迁移执行** | 使用默认 | `drizzle/src/migrate.ts:18-118` | 共享 | 共享 | 共享 | `drizzle/src/migrate.ts` |
| **迁移生成** | `db-mongodb/src/createMigration.ts:22-62` | `db-postgres/src/index.ts:120-124` | `db-sqlite/src/index.ts:186-190` | `db-vercel-postgres/src/index.ts:172-176` | `db-d1-sqlite/src/index.ts:165-169` | `drizzle/src/utilities/buildCreateMigration.ts:13-162` |
| **执行封装** | - | - | - | - | `db-d1-sqlite/src/execute.ts:40-67` | - |
| **Dev Push** | - | 共享 | 共享 | 共享 | 共享 | `drizzle/src/utilities/pushDevSchema.ts:21-114` |
| **预定义迁移** | `db-mongodb/src/predefinedMigrations/` | `db-postgres/src/predefinedMigrations/` | `db-sqlite/src/predefinedMigrations/` | 继承 Postgres | 继承 SQLite | `payload/src/database/migrations/getPredefinedMigration.ts:18-88` |
| **统一接口** | - | - | - | - | - | `payload/src/database/types.ts:17-170` |
| **适配器工厂** | - | - | - | - | - | `payload/src/database/createDatabaseAdapter.ts:24-62` |

## 十、总结

Payload CMS 通过三层架构实现了多数据库支持的差异收敛：

1. **接口层** (`BaseDatabaseAdapter`)：定义统一契约，提供默认实现
2. **抽象层** (`@payloadcms/drizzle` + `RawTable/RawColumn`)：SQL 系共享逻辑，数据库无关的 Schema 表示
3. **实现层** (各适配器)：通过策略模式和特定转换处理数据库差异

关键收敛点：
- 迁移方法：默认实现 + 选择性覆盖
- Schema 同步：抽象 `RawTable` → 数据库特定转换
- 预定义迁移：统一加载机制，各适配器提供特定实现
- 派生适配器 (Vercel Postgres, D1 SQLite)：通过组合和继承复用逻辑
