# Payload CMS 数据库迁移与结构同步差异分析报告

## 一、概述

Payload CMS 支持五种数据库适配器：MongoDB、Postgres、SQLite、Vercel Postgres 和 D1 SQLite。本文档详细分析了这些适配器之间在数据库迁移和结构同步方面如何通过统一接口和抽象层来处理差异。

**核心主线：连接层差异不改变结构同步结果。** 派生适配器（Vercel Postgres、D1 SQLite）通过组合模式复用父适配器的完整结构同步逻辑，仅在连接层做最小覆盖。

---

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

---

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

---

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
  const predefinedMigration = await getPredefinedMigration({ dirname, file, migrationName, payload })
  
  const migrationFileContent = migrationTemplate(predefinedMigration)
  await fs.promises.writeFile(file, migrationFileContent)
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
  
  if ('createExtensions' in this && typeof this.createExtensions === 'function') {
    await this.createExtensions()
  }
  
  const hasMigrationTable = await migrationTableExists(this)
  if (hasMigrationTable) {
    const { docs: migrationsInDB } = await payload.find({ collection: 'payload-migrations' })
    if (migrationsInDB.find((m) => m.batch === -1)) {
      payload.logger.warn(
        'WARNING: `payload dev` was run after migrations were created...',
      )
    }
  }
  
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

`buildCreateMigration` 使用策略模式，通过参数定制不同数据库的行为：

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
    const drizzleJsonAfter = await generateDrizzleJson(this.schema)
    
    const sqlStatementsUp = await generateMigration(drizzleJsonBefore, drizzleJsonAfter)
    const sqlStatementsDown = await generateMigration(drizzleJsonAfter, drizzleJsonBefore)
    
    if (sqlStatementsUp?.length) {
      const sqlExecute = `await db.${executeMethod}(` + 'sql`'
      upSQL = sanitizeStatements({ sqlExecute, statements: sqlStatementsUp })
    }
  }
}
```

#### 4.2.1 Postgres 的策略配置

**文件**: `packages/db-postgres/src/index.ts:100-108`

```typescript
const executeMethod = 'execute'

const sanitizeStatements = ({ sqlExecute, statements }): string => {
  return `${sqlExecute}\n ${statements.join('\n')}\`)`
}
```

#### 4.2.2 SQLite 的策略配置

**文件**: `packages/db-sqlite/src/index.ts:112-123`

```typescript
const executeMethod = 'run'

const sanitizeStatements = ({ sqlExecute, statements }) => {
  return statements
    .map((statement) => `${sqlExecute}${statement?.replaceAll('`', '\\`')}\`)`)
    .join('\n')
}
```

#### 4.2.3 D1 SQLite 的策略配置

**文件**: `packages/db-d1-sqlite/src/index.ts:88-100`

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

```typescript
const executeMethod = 'execute'
const sanitizeStatements = ({ sqlExecute, statements }) => 
  `${sqlExecute}\n ${statements.join('\n')}\`)`
```

**关键结论：Vercel Postgres 与 Postgres 完全相同，D1 SQLite 与 SQLite 完全相同。

### 4.3 迁移语句执行差异

#### 4.3.1 D1 SQLite 的特殊执行封装

**文件**: `packages/db-d1-sqlite/src/execute.ts:40-67`

```typescript
export const execute: Execute<any> = function execute({ db, drizzle, raw, sql: statement }) {
  const executeFrom: any = (db ?? drizzle)!

  const mapToLibSql = (query: SQLiteRaw<D1Result<unknown>>): any => {
    const execute = query.execute
    query.execute = async () => {
      const result: D1Result = await execute()

      // D1 特有：需要映射到 LibSQL 兼容格式
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

**重要：这是运行时执行层的差异，不影响结构同步结果。

### 4.4 预定义迁移系统

**文件**: `packages/payload/src/database/migrations/getPredefinedMigration.ts:18-88`

```typescript
export const getPredefinedMigration = async ({
  dirname, file, migrationName, payload,
}): Promise<MigrationTemplateArgs> => {
  const importPath = file ?? migrationName
  
  if (importPath?.startsWith('@payloadcms/db-')) {
    const migrationName = importPath.split('/').slice(2).join('/')
    let cleanPath = path.join(dirname, `./predefinedMigrations/${migrationName}`)
    const { downSQL, dynamic, imports, upSQL } = await dynamicImport<MigrationTemplateArgs>(cleanPath)
    return { downSQL, dynamic, imports, upSQL }
  }
  
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

---

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

```typescript
export const buildTable = ({
  adapter, fields, setColumnID, tableName, timestamps, versions,
}: Args): Result => {
  const columns: Record<string, RawColumn> = baseColumns
  const indexes: Record<string, RawIndex> = baseIndexes
  
  const idColType: IDType = setColumnID({ adapter, columns, fields })
  
  const { hasLocalizedField, hasLocalizedManyNumberField, ... } = traverseFields({
    adapter, columns, indexes, fields,
  })
  
  if (hasLocalizedField || localizedRelations.size) {
    const localeTableName = `${tableName}${adapter.localesSuffix}`
    adapter.rawTables[localeTableName] = localesTable
  }
  
  if (hasManyTextField) {
    const textsTableName = `${rootTableName}_texts`
    adapter.rawTables[textsTableName] = textsTable
  }
  
  if (relationships.size) {
    const relationshipsTableName = `${tableName}${adapter.relationshipsSuffix}`
    adapter.rawTables[relationshipsTableName] = relationshipsTable
  }
  
  adapter.rawTables[tableName] = table
}
```

**抽象类型定义** (`packages/drizzle/src/types.ts`):

```typescript
export type RawTable = {
  name: string
  columns: Record<string, RawColumn>
  foreignKeys?: Record<string, RawForeignKey>
  indexes?: Record<string, RawIndex>
}

export type RawColumn =
  | ({ type: 'boolean' | 'geometry' | 'jsonb' | 'numeric' | 'serial' | 'text' | 'varchar' } & BaseRawColumn)
  | BinaryVecRawColumn
  | EnumRawColumn
  | HalfVecRawColumn
  | IntegerRawColumn
  | SparseVecRawColumn
  | TimestampRawColumn
  | UUIDRawColumn
  | VectorRawColumn
```

### 5.3 第二层：数据库特定转换

#### 5.3.1 Postgres 转换

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
  vector,
  halfvec,
  sparsevec,
  bit,
}

// 枚举：原生 ENUM
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

// 时间戳：原生 timestamp
case 'timestamp':
  let builder = timestamp(column.name, {
    mode: column.mode,
    precision: column.precision,
    withTimezone: column.withTimezone,
  })
  if (column.defaultNow) {
    builder = builder.defaultNow()
  }
  break

// UUID：原生 uuid
case 'uuid':
  let builder = uuid(column.name)
  if (column.defaultRandom) {
    builder = builder.defaultRandom()
  }
  if (column.defaultV7) {
    builder = builder.$defaultFn(() => uuidv7())
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
}

// 布尔值：INTEGER + mode
case 'boolean':
  columns[key] = integer(column.name, { mode: 'boolean' })
  break

// 枚举：TEXT + check 约束
case 'enum':
  if ('locale' in column) {
    columns[key] = text(column.name, { enum: locales as [string, ...string[]] })
  } else {
    columns[key] = text(column.name, { enum: column.options as [string, ...string[]] })
  }
  break

// JSON/Geometry：TEXT + json mode
case 'geometry':
case 'jsonb':
  columns[key] = text(column.name, { mode: 'json' })
  break

// 时间戳：TEXT + strftime
case 'timestamp':
  let builder = text(column.name)
  if (column.defaultNow) {
    builder = builder.default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`)
  }
  break

// UUID：TEXT + 应用层默认值
case 'uuid':
  let builder = text(column.name, { length: 36 })
  if (column.defaultRandom) {
    builder = builder.$defaultFn(() => uuidv4())
  }
  if (column.defaultV7) {
    builder = builder.$defaultFn(() => uuidv7())
  }
  break
```

### 5.4 第三层：初始化流程

#### 5.4.1 Postgres 初始化

**文件**: `packages/drizzle/src/postgres/init.ts:11-45`

```typescript
export const init: Init = async function init(this: BasePostgresAdapter) {
  this.rawRelations = {}
  this.rawTables = {}
  
  buildRawSchema({ adapter: this, setColumnID })
  
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  if (this.payload.config.localization) {
    this.enums.enum__locales = this.pgSchema.enum(
      '_locales',
      this.payload.config.localization.locales.map(({ code }) => code) as [string, ...string[]],
    )
  }
  
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter: this, rawTable: this.rawTables[tableName] })
  }
  
  buildDrizzleRelations({ adapter: this })
  
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  this.schema = {
    pgSchema: this.pgSchema,
    ...this.tables,
    ...this.relations,
    ...this.enums,
  }
}
```

#### 5.4.2 SQLite 初始化

**文件**: `packages/drizzle/src/sqlite/init.ts:12-45`

```typescript
export const init: Init = async function init(this: BaseSQLiteAdapter) {
  let locales: string[] | undefined
  
  this.rawRelations = {}
  this.rawTables = {}
  
  if (this.payload.config.localization) {
    locales = this.payload.config.localization.locales.map(({ code }) => code)
  }
  
  const adapter = this as unknown as DrizzleAdapter
  
  buildRawSchema({ adapter, setColumnID })
  
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })
  
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter, locales, rawTable: this.rawTables[tableName] })
  }
  
  buildDrizzleRelations({ adapter })
  
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })
  
  this.schema = {
    ...this.tables,
    ...this.relations,
  }
}
```

### 5.5 MongoDB 结构同步链路（Schema-less 但有 Mongoose Schema）

**关键文件**:
- `packages/db-mongodb/src/init.ts:21-118`
- `packages/db-mongodb/src/models/buildCollectionSchema.ts:9-49`
- `packages/db-mongodb/src/models/buildSchema.ts:130-943`

#### 5.5.1 MongoDB 初始化流程

```typescript
export const init: Init = async function init(this: MongooseAdapter) {
  this.connection ??= mongoose.createConnection()
  
  if (this.afterCreateConnection) {
    await this.afterCreateConnection(this)
  }
  
  this.payload.config.collections.forEach((collection: SanitizedCollectionConfig) => {
    const schemaOptions = this.collectionsSchemaOptions?.[collection.slug]
    
    const schema = buildCollectionSchema(collection, this.payload, schemaOptions)
    
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
      this.versions[collection.slug] = this.connection.model(
        versionModelName,
        versionSchema,
        versionCollectionName,
      )
    }
    
    this.collections[collection.slug] = this.connection.model(modelName, schema, collectionName)
  })
  
  this.globals = buildGlobalModel(this)
  
  this.payload.config.globals.forEach((global) => {
    if (global.versions) {
      const versionCollectionFields = buildGlobalVersionFields(this.payload.config, global)
      const versionSchema = buildSchema({
        configFields: versionCollectionFields,
        payload: this.payload,
      })
      this.globalVersions[global.slug] = this.connection.model(
        `${global.slug}Versions`,
        versionSchema,
      )
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
  
  if (Array.isArray(collection.upload.filenameCompoundIndex)) {
    schema.index(indexDefinition, { unique: true })
  }
  
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
  
  schemaFields.forEach((field) => {
    if (!fieldIsPresentationalOnly(field)) {
      const addFieldSchema = getSchemaGenerator(field.type)
      if (addFieldSchema) {
        addFieldSchema(field, schema, payload, buildSchemaOptions, parentIsLocalized ?? false)
      }
    }
  })
  
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

---

## 六、开发模式 Schema Push

### 6.1 统一 Push 机制

**文件**: `packages/drizzle/src/utilities/pushDevSchema.ts:21-114`

```typescript
export const pushDevSchema = async (adapter: DrizzleAdapter) => {
  const equal = dequal(previousSchema, {
    localeCodes,
    rawTables: adapter.rawTables,
  })
  if (equal) return
  
  const { pushSchema } = adapter.requireDrizzleKit()
  
  const { extensions = {}, tablesFilter } = adapter as BasePostgresAdapter
  const { apply, hasDataLoss, warnings } = await pushSchema(
    adapter.schema,
    adapter.drizzle,
    adapter.schemaName ? [adapter.schemaName] : undefined,
    tablesFilter,
    extensions.postgis ? ['postgis'] : undefined,
  )
  
  if (warnings.length) {
    const warningStrings = warnings.map((warning) => `${warning.code}: ${warning.message}`)
    const shouldContinue = await adapter.payload.confirmContinue(
      `Warnings detected during schema push...`,
    )
    if (!shouldContinue) {
      return
    }
  }
  
  if (hasDataLoss) {
    const shouldContinue = await adapter.payload.confirmContinue(
      `Data loss is possible...`,
    )
    if (!shouldContinue) {
      return
    }
  }
  
  await apply()
  
  await drizzle.insert(adapter.tables.payload_migrations).values({
    name: 'dev',
    batch: -1,
  })
}
```

---

## 七、派生适配器的继承边界分析

### 7.1 继承模式：组合 vs 继承

**关键洞察：Vercel Postgres 和 D1 SQLite 并不通过传统的类继承方式（如 `class VercelPostgresAdapter extends PostgresAdapter`）来复用代码，而是通过**组合模式**：

- 直接导入 `@payloadcms/drizzle/postgres` 或 `@payloadcms/drizzle/sqlite` 中的函数
- 在 `createDatabaseAdapter` 配置中选择性覆盖特定属性

```
┌──────────────────────────────────────────────────────────────┐
│                      组合复用模式                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   @payloadcms/drizzle/postgres                               │
│   ├─ init (结构同步入口)         ← 直接导入复用                │
│   ├─ buildDrizzleTable (列转换)  ← 直接导入复用                │
│   ├─ execute (执行器)           ← 直接导入复用                │
│   └─ ...                                                       │
│                              ↓                                 │
│   @payloadcms/db-vercel-postgres                              │
│   ├─ 导入 postgres 的 init/buildDrizzleTable 等               │
│   ├─ 自定义 connect (连接层)    ← 覆盖                         │
│   ├─ forceUseVercelPostgres      ← 新增属性                    │
│   └─ readReplicasAfterWriteInterval ← 新增属性                 │
│                                                              │
│   (没有任何结构同步相关的自定义代码)                            │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 SQLite 家族：D1 SQLite 与 SQLite 的复用链路

#### 7.2.1 导入关系对比

**SQLite 适配器** (`packages/db-sqlite/src/index.ts:1-56`):

```typescript
import {
  beginTransaction,
  buildCreateMigration,
  buildSchemaGenerator,
  // ... CRUD 方法
} from '@payloadcms/drizzle'

import {
  columnToCodeConverter,
  countDistinct,
  defaultDrizzleSnapshot,
  dropDatabase,
  execute,               // SQLite 版本的 execute
  init,                  // ← 结构同步入口
  insert,
  requireDrizzleKit,
} from '@payloadcms/drizzle/sqlite'
```

**D1 SQLite 适配器** (`packages/db-d1-sqlite/src/index.ts:1-62`):

```typescript
// 从 @payloadcms/drizzle 导入 ← 完全相同
import {
  beginTransaction,
  buildCreateMigration,
  buildSchemaGenerator,
} from '@payloadcms/drizzle'

// 从 @payloadcms/drizzle/sqlite 导入 ← 完全相同
import {
  columnToCodeConverter,
  countDistinct,
  defaultDrizzleSnapshot,
  dropDatabase,
  init,                  // ← 复用同一个 init！
  insert,
  requireDrizzleKit,
  // 注意：没有导入 execute！
} from '@payloadcms/drizzle/sqlite'

// 自定义实现
import { connect } from './connect.js'   // ← 自定义 connect
import { execute } from './execute.js'   // ← 自定义 execute
```

**关键差异：execute 和 connect 被覆盖，init 和 buildDrizzleTable 完全复用。

#### 7.2.2 结构同步复用链路（完全相同）

**SQLite init** (`packages/drizzle/src/sqlite/init.ts:12-45`):

```typescript
// 这个 init 函数同时被 SQLite 和 D1 SQLite 使用
export const init: Init = async function init(this: BaseSQLiteAdapter) {
  let locales: string[] | undefined

  this.rawRelations = {}
  this.rawTables = {}

  if (this.payload.config.localization) {
    locales = this.payload.config.localization.locales.map(({ code }) => code)
  }

  const adapter = this as unknown as DrizzleAdapter

  // 步骤 1: 构建抽象 RawTable（数据库无关）
  buildRawSchema({ adapter, setColumnID })

  // 步骤 2: beforeSchemaInit 钩子
  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })

  // 步骤 3: SQLite 特定转换
  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter, locales, rawTable: this.rawTables[tableName] })
  }

  // 步骤 4: 构建关系
  buildDrizzleRelations({ adapter })

  // 步骤 5: afterSchemaInit 钩子
  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })

  // 步骤 6: 组装 schema
  this.schema = {
    ...this.tables,
    ...this.relations,
  }
}
```

**buildDrizzleTable** (`packages/drizzle/src/sqlite/schema/buildDrizzleTable.ts:23-150`):

```typescript
// 这个函数同时被 SQLite 和 D1 SQLite 使用
export const buildDrizzleTable: BuildDrizzleTable = ({ adapter, locales, rawTable }) => {
  const columns: Record<string, any> = {}

  for (const [key, column] of Object.entries(rawTable.columns)) {
    switch (column.type) {
      case 'boolean':
        columns[key] = integer(column.name, { mode: 'boolean' })
        break
      case 'enum':
        if ('locale' in column) {
          columns[key] = text(column.name, { enum: locales as [string, ...string[]] })
        } else {
          columns[key] = text(column.name, { enum: column.options as [string, ...string[]] })
        }
        break
      // ... 其他列类型处理
    }
  }
  // ... 外键、索引等
}
```

#### 7.2.3 继承边界总结

| 模块 | SQLite | D1 SQLite | 差异原因 |
|------|--------|-----------|---------|
| **init** | 导入自 @payloadcms/drizzle/sqlite | 完全相同 | 结构同步逻辑与驱动无关 |
| **buildRawSchema** | 导入自 @payloadcms/drizzle/schema | 完全相同 | 完全抽象的 Schema 构建 |
| **buildDrizzleTable** | SQLite 方言转换 | 完全相同 | SQLite 方言统一 |
| **connect** | 本地实现（libsql + WAL 配置） | 自定义 | 不同的驱动（libsql vs D1） |
| **execute** | 导入自 @payloadcms/drizzle/sqlite | 自定义 | D1 需要结果格式映射 |
| **buildCreateMigration** | SQLite 策略（run + 转义） | 完全相同 | SQLite 方言统一 |

**结论：D1 SQLite 在结构同步层面与 SQLite 完全一致，差异只在连接和执行层。

### 7.3 Postgres 家族：Vercel Postgres 与 Postgres 的复用链路

#### 7.3.1 导入关系对比

**Postgres 适配器** (`packages/db-postgres/src/index.ts:1-57`):

```typescript
import {
  beginTransaction,
  buildCreateMigration,
  buildSchemaGenerator,
} from '@payloadcms/drizzle'

import {
  columnToCodeConverter,
  countDistinct,
  createDatabase,
  createExtensions,
  execute,
  init,                  // ← 结构同步入口
  insert,
  requireDrizzleKit,
} from '@payloadcms/drizzle/postgres'

import { pgEnum, pgSchema, pgTable } from 'drizzle-orm/pg-core'
import pgDependency from 'pg'   // ← node-postgres 驱动
```

**Vercel Postgres 适配器** (`packages/db-vercel-postgres/src/index.ts:1-58`):

```typescript
// 从 @payloadcms/drizzle 导入 ← 完全相同
import {
  beginTransaction,
  buildCreateMigration,
  buildSchemaGenerator,
} from '@payloadcms/drizzle'

// 从 @payloadcms/drizzle/postgres 导入 ← 完全相同
import {
  columnToCodeConverter,
  countDistinct,
  createDatabase,
  createExtensions,
  execute,            // ← 复用同一个 execute
  init,              // ← 复用同一个 init
  insert,
  requireDrizzleKit,
} from '@payloadcms/drizzle/postgres'

import { pgEnum, pgSchema, pgTable } from 'drizzle-orm/pg-core'
// 没有导入 pg！使用 Vercel 驱动
```

#### 7.3.2 结构同步复用链路（完全相同）

**Postgres init** (`packages/drizzle/src/postgres/init.ts:11-45`):

```typescript
// 这个 init 函数同时被 Postgres 和 Vercel Postgres 使用
export const init: Init = async function init(this: BasePostgresAdapter) {
  this.rawRelations = {}
  this.rawTables = {}

  buildRawSchema({ adapter: this, setColumnID })

  await executeSchemaHooks({ type: 'beforeSchemaInit', adapter: this })

  if (this.payload.config.localization) {
    this.enums.enum__locales = this.pgSchema.enum(
      '_locales',
      this.payload.config.localization.locales.map(({ code }) => code) as [string, ...string[]],
    )
  }

  for (const tableName in this.rawTables) {
    buildDrizzleTable({ adapter: this, rawTable: this.rawTables[tableName] })
  }

  buildDrizzleRelations({ adapter: this })

  await executeSchemaHooks({ type: 'afterSchemaInit', adapter: this })

  this.schema = {
    pgSchema: this.pgSchema,
    ...this.tables,
    ...this.relations,
    ...this.enums,
  }
}
```

#### 7.3.3 继承边界总结

| 模块 | Postgres | Vercel Postgres | 差异原因 |
|------|----------|-----------------|---------|
| **init** | 导入自 @payloadcms/drizzle/postgres | 完全相同 | 结构同步逻辑与驱动无关 |
| **buildRawSchema** | 导入自 @payloadcms/drizzle/schema | 完全相同 | 完全抽象的 Schema 构建 |
| **buildDrizzleTable** | Postgres 方言转换 | 完全相同 | Postgres 方言统一 |
| **execute** | 导入自 @payloadcms/drizzle/postgres | 完全相同 | 执行逻辑与驱动无关 |
| **connect** | 本地实现（pg 连接池） | 自定义 | 不同的驱动（pg vs VercelPool） |
| **buildCreateMigration** | Postgres 策略（execute + 不转义） | 完全相同 | Postgres 方言统一 |
| **createExtensions** | 导入自 @payloadcms/drizzle/postgres | 完全相同 | 扩展创建与驱动无关 |

**结论：Vercel Postgres 在结构同步层面与 Postgres 完全一致，差异只在连接层。

---

## 八、连接层差异 vs 结构同步差异：代码证据

### 8.1 层次划分

```
┌──────────────────────────────────────────────────────────────────────┐
│                           层次划分                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  连接层（Connection Layer）                                    │    │
│  │  差异：驱动选择、连接池、读副本、WAL 等                          │    │
│  │  不影响结构同步结果                                            │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  结构同步层（Schema Sync Layer）                              │    │
│  │  差异：列类型映射、枚举实现、时间戳处理等                       │    │
│  │  影响表结构                                                   │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  抽象 Schema 层（Abstract Layer）                             │    │
│  │  无差异：buildRawSchema 完全统一                              │    │
│  │  统一生成 RawTable/RawColumn                                  │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.2 连接层差异（不影响结构同步结果）

#### 8.2.1 SQLite vs D1 SQLite 连接层差异

**SQLite 连接** (`packages/db-sqlite/src/connect.ts:10-76`):

```typescript
export const connect: Connect = async function connect(this: SQLiteAdapter, options) {
  const { hotReload } = options

  try {
    if (!this.client) {
      // 差异点 1: libsql 客户端创建
      this.client = createClient(this.clientConfig)

      // 差异点 2: SQLite 特定的 PRAGMA 配置（D1 不支持）
      if (this.busyTimeout > 0) {
        await this.client.execute(`PRAGMA busy_timeout = ${this.busyTimeout};`)
      }

      // 差异点 3: WAL 模式配置（D1 内部实现，用户无法配置）
      if (this.wal) {
        const result = await this.client.execute('PRAGMA journal_mode;')
        if (result.rows[0]?.journal_mode !== 'wal') {
          await this.client.execute(`PRAGMA journal_mode = WAL;`)
          await this.client.execute(`PRAGMA journal_size_limit = ${this.wal.journalSizeLimit};`)
        }
        await this.client.execute(`PRAGMA synchronous = ${this.wal.synchronous};`)
      }
    }

    // 差异点 4: libsql drizzle 驱动
    const logger = this.logger || false
    this.drizzle = drizzle(this.client, { logger, schema: this.schema })
    this.client = this.drizzle.$client
  }
  // ... pushDevSchema、migrate 等后续步骤完全相同
}
```

**D1 SQLite 连接** (`packages/db-d1-sqlite/src/connect.ts:9-73`):

```typescript
export const connect: Connect = async function connect(this: SQLiteD1Adapter, options) {
  const { hotReload } = options

  // 差异点 1: D1 binding 直接使用，不需要创建客户端
  this.schema = {
    ...this.tables,
    ...this.relations,
  }

  try {
    const logger = this.logger || false
    const readReplicas = this.readReplicas

    let binding = this.binding

    // 差异点 2: D1 特有：只读副本策略
    if (readReplicas && readReplicas === 'first-primary') {
      binding = this.binding.withSession('first-primary')
    }

    // 差异点 3: D1 drizzle 驱动
    this.drizzle = drizzle(binding, {
      logger,
      schema: this.schema,
    })

    this.client = this.drizzle.$client as any
  }
  // ... pushDevSchema、migrate 等后续步骤完全相同
}
```

**D1 SQLite 执行器差异** (`packages/db-d1-sqlite/src/execute.ts:40-67`):

```typescript
export const execute: Execute<any> = function execute({ db, drizzle, raw, sql: statement }) {
  const executeFrom: any = (db ?? drizzle)!

  const mapToLibSql = (query: SQLiteRaw<D1Result<unknown>>): any => {
    const execute = query.execute
    query.execute = async () => {
      const result: D1Result = await execute()

      // D1 特有：需要映射到 LibSQL 兼容格式
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

**SQLite vs D1 SQLite 连接层差异总结：

| 特性 | SQLite | D1 SQLite | 是否影响结构同步 |
|------|--------|-----------|---------------|
| **客户端创建** | `createClient(clientConfig)` | 直接使用 `binding` | ❌ 否 |
| **WAL 配置** | 可配置 | D1 内部实现 | ❌ 否 |
| **busy_timeout** | 可配置 | D1 内部实现 | ❌ 否 |
| **只读副本** | 无 | `withSession('first-primary') | ❌ 否 |
| **驱动** | `drizzle-orm/libsql` | `drizzle-orm/d1` | ❌ 否 |
| **结果格式** | LibSQL 原生 | 需要映射 | ❌ 否 |

**这些差异都只影响运行时行为，不影响表结构生成。

#### 8.2.2 Postgres vs Vercel Postgres 连接层差异

**Postgres 连接** (`packages/db-postgres/src/connect.ts:47-133`):

```typescript
export const connect: Connect = async function connect(this: PostgresAdapter, options) {
  const { hotReload } = options

  try {
    if (!this.pool) {
      // 差异点 1: pg 连接池
      this.pool = new this.pg.Pool(this.poolOptions)

      // 差异点 2: 自动重连逻辑
      await connectWithReconnect({ adapter: this, pool: this.pool })
    }

    const logger = this.logger || false
    this.drizzle = drizzle({ client: this.pool, logger, schema: this.schema })

    // 差异点 3: 读副本（使用 pg.Pool）
    if (this.readReplicaOptions) {
      this.primaryDrizzle = this.drizzle as any
      const readReplicas = this.readReplicaOptions.map((connectionString) => {
        const pool = new this.pg.Pool(options)
        return drizzle({ client: pool, logger, schema: this.schema })
      })
      const myReplicas = withReplicas(this.drizzle, readReplicas as any)
      this.drizzle = myReplicas
    }
  }
  // ... createExtensions、pushDevSchema、migrate 等后续步骤完全相同
}
```

**Vercel Postgres 连接** (`packages/db-vercel-postgres/src/connect.ts:12-115`):

```typescript
export const connect: Connect = async function connect(this: VercelPostgresAdapter, options) {
  const { hotReload } = options

  const connectionString = this.poolOptions?.connectionString ?? process.env.POSTGRES_URL

  try {
    let client: pg.Pool | VercelPool

    // 差异点 1: 本地开发自动降级到 pg
    if (
      !this.forceUseVercelPostgres &&
      connectionString &&
      ['127.0.0.1', 'localhost'].includes(new URL(connectionString).hostname)
    ) {
      client = new pg.Pool(this.poolOptions ?? { connectionString })
    } else {
      client = this.poolOptions ? new VercelPool(this.poolOptions) : sql
    }

    const logger = this.logger || false
    this.drizzle = drizzle({ client: client as pg.Pool, logger, schema: this.schema })

    // 差异点 2: 读副本（使用 VercelPool）
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
  // ... createExtensions、pushDevSchema、migrate 等后续步骤完全相同
}
```

**Postgres vs Vercel Postgres 连接层差异总结：

| 特性 | Postgres | Vercel Postgres | 是否影响结构同步 |
|------|----------|-------------------|---------------|
| **驱动** | `pg.Pool` (node-postgres) | `VercelPool` (@vercel/postgres) | ❌ 否 |
| **本地开发驱动** | 固定 pg | 自动降级到 pg | ❌ 否 |
| **自动重连** | `connectWithReconnect` | 无（Vercel 驱动内部实现 | ❌ 否 |
| **forceUseVercelPostgres** | 无 | 强制使用 Vercel 驱动 | ❌ 否 |
| **读副本** | pg.Pool 读副本 | VercelPool 读副本 | ❌ 否 |

**这些差异都只影响连接管理和运行时性能，不影响表结构生成。

### 8.3 结构同步层差异（影响表结构）

#### 8.3.1 SQLite vs Postgres 结构同步差异

**Postgres init 差异点** (`packages/drizzle/src/postgres/init.ts:22-27`):

```typescript
// Postgres 特有：创建枚举类型
if (this.payload.config.localization) {
  this.enums.enum__locales = this.pgSchema.enum(
    '_locales',
    this.payload.config.localization.locales.map(({ code }) => code) as [string, ...string[]],
  )
}
```

**SQLite init 差异点** (`packages/drizzle/src/sqlite/init.ts:18-20`):

```typescript
// SQLite：不支持枚举类型，locales 只用于 TEXT check 约束
if (this.payload.config.localization) {
  locales = this.payload.config.localization.locales.map(({ code }) => code)
}
```

**Postgres buildDrizzleTable 列类型映射：

```typescript
// 枚举：原生 ENUM
case 'enum':
  columns[key] = adapter.enums.enum__locales(column.name)

// 时间戳：原生 timestamp
case 'timestamp':
  let builder = timestamp(column.name, {
    mode: column.mode,
    precision: column.precision,
    withTimezone: column.withTimezone,
  })
  if (column.defaultNow) {
    builder = builder.defaultNow()
  }

// UUID：原生 uuid
case 'uuid':
  let builder = uuid(column.name)
  if (column.defaultRandom) {
    builder = builder.defaultRandom()  // gen_random_uuid()
  }

// 向量：pgvector 特有
case 'vector':
case 'halfvec':
case 'sparsevec':
case 'bit':
  columns[key] = vector(column.name, { dimensions: column.dimensions })
```

**SQLite buildDrizzleTable 列类型映射：

```typescript
// 枚举：TEXT + check 约束
case 'enum':
  columns[key] = text(column.name, { enum: locales as [string, ...string[]] })

// 时间戳：TEXT + strftime
case 'timestamp':
  let builder = text(column.name)
  if (column.defaultNow) {
    builder = builder.default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`)
  }

// UUID：TEXT + 应用层默认值
case 'uuid':
  let builder = text(column.name, { length: 36 })
  if (column.defaultRandom) {
    builder = builder.$defaultFn(() => uuidv4())  // 应用层生成
  }

// 向量：不支持
// 无 vector/halfvec/sparsevec/bit 处理
```

**Postgres vs SQLite 结构同步层差异总结：

| 特性 | Postgres | SQLite | 影响 |
|------|----------|--------|------|
| **枚举** | 原生 ENUM 类型 | TEXT + check 约束 | ✅ 表结构不同 |
| **布尔** | 原生 boolean | INTEGER + mode | ✅ 列类型不同 |
| **时间戳** | 原生 timestamp (支持时区) | TEXT + strftime | ✅ 列类型不同 |
| **UUID** | 原生 uuid + gen_random_uuid() | TEXT + 应用层默认值 | ✅ 列类型不同 |
| **向量** | pgvector 支持 | 不支持 | ✅ 功能差异 |
| **Schema 组装** | 包含 pgSchema + enums | 只有 tables + relations | ✅ 结构不同 |

**这些是真正影响表结构的差异，发生在结构同步层。

---

## 九、功能差异矩阵

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

---

## 十、统一接口如何收敛差异

### 10.1 迁移方法：默认实现 + 选择性覆盖

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

### 10.2 结构同步：抽象层 + 数据库特定转换

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

### 10.3 关键设计模式

| 设计模式 | 应用场景 | 代码位置 |
|---------|---------|---------|
| **策略模式** | 迁移语句生成 (`executeMethod`, `sanitizeStatements`) | `packages/drizzle/src/utilities/buildCreateMigration.ts` |
| **模板方法模式** | Schema 构建骨架 (`buildRawSchema`) | `packages/drizzle/src/schema/build.ts` |
| **抽象工厂** | 适配器创建 + 默认实现 | `packages/payload/src/database/createDatabaseAdapter.ts` |
| **适配器模式** | Drizzle 适配不同 SQL 方言 | `packages/drizzle/src/postgres/`, `packages/drizzle/src/sqlite/` |
| **桥接模式** | 抽象 Schema (`RawTable`) 与实现分离 | `packages/drizzle/src/types.ts` |
| **组合模式** | 派生适配器复用逻辑 (Vercel Postgres, D1 SQLite) | `packages/db-vercel-postgres/src/index.ts`, `packages/db-d1-sqlite/src/index.ts` |

---

## 十一、完整代码引用索引

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

---

## 十二、总结

### 12.1 四层架构总结

Payload CMS 通过四层架构实现了多数据库支持的差异收敛：

1. **接口层** (`BaseDatabaseAdapter`)：定义统一契约，提供默认实现
2. **抽象层** (`@payloadcms/drizzle` + `RawTable/RawColumn`)：SQL 系共享逻辑，数据库无关的 Schema 表示
3. **方言层** (Postgres/SQLite 方言)：处理列类型映射、枚举实现、时间戳处理等
4. **连接层**：处理驱动选择、连接池、读副本、WAL 配置等

### 12.2 核心主线：连接层差异不改变结构同步结果

**派生适配器的继承边界：

| 派生适配器 | 完全复用的模块 | 仅覆盖的模块 |
|-----------|---------------|-------------|
| **Vercel Postgres** | `init`, `buildRawSchema`, `buildDrizzleTable`, `execute`, `buildCreateMigration`, `createExtensions` | `connect`（驱动选择、本地降级、VercelPool） |
| **D1 SQLite** | `init`, `buildRawSchema`, `buildDrizzleTable`, `buildCreateMigration` | `connect`（D1 binding、只读副本策略）, `execute`（结果格式映射） |

**连接层差异（不影响表结构）：

- 驱动选择（pg vs VercelPool, libsql vs D1）
- 连接池配置
- WAL 模式
- busy_timeout
- 读副本策略
- 自动重连

**结构同步层差异（影响表结构）：

- Postgres vs SQLite：枚举实现（原生 ENUM vs TEXT + check）
- Postgres vs SQLite：列类型映射（timestamp vs TEXT, uuid vs TEXT, boolean vs INTEGER）
- Postgres vs SQLite：Schema 组装（包含 pgSchema/enums vs 仅 tables/relations）

### 12.3 关键收敛点

1. **迁移方法**：默认实现 + 选择性覆盖
2. **Schema 同步**：抽象 `RawTable` → 数据库特定转换
3. **预定义迁移**：统一加载机制，各适配器提供特定实现
4. **派生适配器**：通过组合模式复用逻辑，仅在连接层做最小覆盖
