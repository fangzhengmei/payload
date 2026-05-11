# Payload 后台任务队列分析报告

## 一、任务接收与调度机制

### 1.1 任务接收

Payload 的任务队列系统通过 `payload.jobs.queue()` 方法接收任务。任务可以从多个位置入队：

- **Collection Hooks**：响应文档变更时入队
- **Field Hooks**：特定字段变更时入队
- **Custom Endpoints**：从 API 路由入队
- **Server Actions**：从 Next.js 服务端操作入队

任务入队的核心逻辑位于 `packages/payload/src/queues/localAPI.ts:52-223`：

```typescript
// 任务入队的核心流程
1. 检查访问控制权限
2. 确定目标队列（用户指定或工作流默认）
3. 准备任务数据（input, queue, waitUntil, workflowSlug/taskSlug 等）
4. 计算并发控制键（如果启用）
5. 处理 supersedes 逻辑（删除同键的旧待处理任务）
6. 通过 payload.db.create() 将任务写入 payload-jobs 集合
```

### 1.2 任务存储结构

任务存储在 `payload-jobs` 集合中（`packages/payload/src/queues/config/collection.ts:121-290`），核心字段包括：

| 字段 | 类型 | 说明 |
|------|------|------|
| `input` | JSON | 任务输入数据 |
| `workflowSlug` | select | 关联的工作流标识 |
| `taskSlug` | select | 关联的任务标识 |
| `queue` | text | 队列名称，默认 'default' |
| `waitUntil` | date | 延迟执行时间 |
| `processing` | checkbox | 是否正在处理中 |
| `completedAt` | date | 完成时间 |
| `totalTried` | number | 尝试次数 |
| `hasError` | checkbox | 是否有最终错误 |
| `error` | JSON | 错误详情 |
| `log` | array | 任务执行日志 |
| `concurrencyKey` | text | 并发控制键 |
| `taskStatus` | JSON（虚拟） | 任务执行状态摘要 |

### 1.3 定时调度链路（schedule 任务入队机制）

#### 1.3.1 调度配置

任务的 `schedule` 配置定义了自动入队规则：

```typescript
jobs: {
  tasks: [
    {
      slug: 'dailyDigest',
      schedule: [
        {
          cron: '0 8 * * *',  // 每天 8:00 AM
          queue: 'daily',     // 入队到 'daily' 队列
          hooks: {
            beforeSchedule: async ({ queueable, req }) => {
              // 可选：自定义入队前检查
              return { shouldSchedule: true }
            },
            afterSchedule: async ({ job, status }) => {
              // 可选：入队后回调
            },
          },
        },
      ],
      handler: async ({ input }) => { /* ... */ },
    },
  ],
}
```

#### 1.3.2 调度触发方式

定时调度通过以下方式触发：

**方式一：autoRun（Next.js 进程内）**

```typescript
// packages/payload/src/index.ts:712-770
async _initializeCrons() {
  if (this.config.jobs.enabled && this.config.jobs.autoRun && !isNextBuild()) {
    const cronJobs = typeof this.config.jobs.autoRun === 'function'
      ? await this.config.jobs.autoRun(this)
      : this.config.jobs.autoRun

    await Promise.all(
      cronJobs.map((cronConfig) => {
        const jobAutorunCron = new Cron(
          cronConfig.cron ?? DEFAULT_CRON,
          async () => {
            // 1. 先处理调度（入队）
            if (
              _internal_jobSystemGlobals.shouldAutoSchedule &&
              !cronConfig.disableScheduling &&
              this.config.jobs.scheduling
            ) {
              await this.jobs.handleSchedules({
                allQueues: cronConfig.allQueues,
                queue: cronConfig.queue,
              })
            }

            // 2. 然后运行已入队的任务
            if (_internal_jobSystemGlobals.shouldAutoRun) {
              await this.jobs.run({
                allQueues: cronConfig.allQueues,
                limit: cronConfig.limit ?? DEFAULT_LIMIT,
                queue: cronConfig.queue,
                silent: cronConfig.silent,
              })
            }
          },
          { protect: true },  // 防止重叠执行
        )
        this.crons.push(jobAutorunCron)
      }),
    )
  }
}
```

**方式二：Bin Script（独立进程）**

```bash
# 同时处理调度和运行
pnpm payload jobs:run --cron "*/5 * * * *" --queue myQueue --handle-schedules
```

**方式三：Endpoint（无服务器平台）**

```bash
GET /api/payload-jobs/handle-schedules?queue=daily
```

#### 1.3.3 调度执行流程（handleSchedules）

定时调度的核心逻辑位于 `packages/payload/src/queues/operations/handleSchedules/index.ts:25-123`：

```
handleSchedules 执行流程：

1. 获取所有带 schedule 配置的任务/工作流
   queuesWithSchedules = getQueuesWithSchedules(jobsConfig)

2. 读取调度状态全局（payload-jobs-stats）
   stats = payload.db.findGlobal('payload-jobs-stats')

3. 遍历每个调度配置，检查时间约束
   for each schedulable in schedules:
     queueable = checkQueueableTimeConstraints({
       scheduleConfig,
       stats,  // 包含 lastScheduledRun
     })

4. 检查是否应该入队（beforeSchedule 钩子）
   for each queueable:
     result = scheduleQueueable({ queueable, req, stats })
     - 调用 beforeSchedule（默认或自定义）
     - 如果 shouldSchedule=true，调用 payload.jobs.queue() 入队
     - 调用 afterSchedule 更新 lastScheduledRun

5. 返回结果：{ queued, skipped, errored }
```

#### 1.3.4 时间约束检查（checkQueueableTimeConstraints）

```typescript
// packages/payload/src/queues/operations/handleSchedules/index.ts:125-155
export function checkQueueableTimeConstraints({
  queue, scheduleConfig, stats, taskConfig, workflowConfig
}): false | Queueable {
  // 从 stats 获取上次调度时间
  const lastScheduledRun = taskConfig
    ? stats?.stats?.scheduledRuns?.queues?.[queue]?.tasks?.[taskConfig.slug]?.lastScheduledRun
    : stats?.stats?.scheduledRuns?.queues?.[queue]?.workflows?.[workflowConfig?.slug]?.lastScheduledRun

  // 基于 cron 和上次调度时间，计算下次应该调度的时间
  const nextRun = new Cron(scheduleConfig.cron).nextRun(lastScheduledRun ?? undefined)

  if (!nextRun) return false

  return {
    scheduleConfig,
    taskConfig,
    waitUntil: nextRun,  // 计算出的调度时间
    workflowConfig,
  }
}
```

**关键设计**：
- 使用 `payload-jobs-stats` 全局存储 `lastScheduledRun`
- 基于 cron 表达式和上次调度时间，计算下次应该调度的时间
- 如果 `nextRun` 是过去的时间（错过了调度窗口），该调度仍然会被触发

#### 1.3.5 调度状态全局（payload-jobs-stats）

```typescript
// packages/payload/src/queues/config/global.ts:11-30
export type JobStats = {
  stats?: {
    scheduledRuns?: {
      queues?: {
        [queueSlug: string]: {
          tasks?: {
            [taskSlug: string]: {
              lastScheduledRun: string  // 上次调度时间 ISO 字符串
            }
          }
          workflows?: {
            [workflowSlug: string]: {
              lastScheduledRun: string
            }
          }
        }
      }
    }
  }
}
```

#### 1.3.6 同队列防重复与 waitUntil 协同规则

**默认防重复逻辑**（`defaultBeforeSchedule`）：

```typescript
// packages/payload/src/queues/operations/handleSchedules/defaultBeforeSchedule.ts:5-20
export const defaultBeforeSchedule: BeforeScheduleFn = async ({ queueable, req }) => {
  // 统计同队列、同任务的"可运行"或"运行中"的任务
  const runnableOrActiveJobsForQueue = await countRunnableOrActiveJobsForQueue({
    onlyScheduled: true,           // 只统计调度系统创建的任务
    queue: queueable.scheduleConfig.queue,
    req,
    taskSlug: queueable.taskConfig?.slug,
    workflowSlug: queueable.workflowConfig?.slug,
  })

  return {
    input: {},
    shouldSchedule: runnableOrActiveJobsForQueue === 0,  // 只有 0 个才允许入队
    waitUntil: queueable.waitUntil,
  }
}
```

**可运行/运行中任务的查询条件**（`countRunnableOrActiveJobsForQueue`）：

```typescript
// packages/payload/src/queues/operations/handleSchedules/countRunnableOrActiveJobsForQueue.ts:13-76
export async function countRunnableOrActiveJobsForQueue({
  onlyScheduled = false, queue, req, taskSlug, workflowSlug
}): Promise<number> {
  const and: Where[] = [
    { queue: { equals: queue } },
    { completedAt: { exists: false } },    // 未完成
    { error: { exists: false } },          // 无最终错误（注意：不是 hasError）
  ]

  if (taskSlug) and.push({ taskSlug: { equals: taskSlug } })
  else if (workflowSlug) and.push({ workflowSlug: { equals: workflowSlug } })

  if (onlyScheduled) {
    and.push({ 'meta.scheduled': { equals: true } })  // 只统计调度系统创建的
  }

  const result = await req.payload.db.count({
    collection: jobsCollectionSlug,
    where: { and },
  })

  return result.totalDocs
}
```

**防重复与 waitUntil 的协同规则**：

| 场景 | `defaultBeforeSchedule` 行为 | 说明 |
|------|-----------------------------|------|
| 同队列、同任务、无待处理任务 | `shouldSchedule: true` | 正常入队，`waitUntil` 设置为调度时间 |
| 同队列、同任务、有任务 `processing=true` | `shouldSchedule: false` | 跳过，不创建新任务 |
| 同队列、同任务、有任务 `waitUntil` 未到期 | `shouldSchedule: false` | 跳过，不创建新任务 |
| 同队列、同任务、有任务 `waitUntil` 已过期且未处理 | `shouldSchedule: false` | 跳过，不创建新任务 |
| 同队列、同任务、有任务失败但可重试（`hasError=false`） | `shouldSchedule: false` | 跳过，等待现有任务重试 |
| 同队列、同任务、有任务最终失败（`hasError=true`） | `shouldSchedule: true` | 允许入队新任务 |
| 同队列、不同任务 | `shouldSchedule: true` | 不互斥，可以同时存在 |

**关键发现**：
- `countRunnableOrActiveJobsForQueue` 查询条件是 `error: { exists: false }`，不是 `hasError: { not_equals: true }`
- 这意味着只要任务的 `error` 字段不存在（无论 `hasError` 是什么），都算作"可运行"
- 任务失败后，如果 `hasError=true` 且 `error` 字段被设置，该任务不会阻止新的调度入队
- **waitUntil 不会参与防重复检查**：即使有任务的 `waitUntil` 还未到期，只要它存在于队列中且未完成、无最终错误，新调度就会被跳过

**自定义防重复逻辑**：

用户可以通过 `hooks.beforeSchedule` 覆盖默认行为：

```typescript
schedule: [{
  cron: '0 8 * * *',
  queue: 'daily',
  hooks: {
    beforeSchedule: async ({ queueable, req, defaultBeforeSchedule }) => {
      // 可以调用默认逻辑
      const defaultResult = await defaultBeforeSchedule({ queueable, req })
      
      // 或者自定义检查
      const existingJobs = await req.payload.find({
        collection: 'payload-jobs',
        where: {
          and: [
            { taskSlug: { equals: queueable.taskConfig?.slug } },
            { queue: { equals: queueable.scheduleConfig.queue } },
            { completedAt: { exists: false } },
          ],
        },
      })

      // 可以修改 waitUntil
      return {
        shouldSchedule: existingJobs.totalDocs === 0,
        waitUntil: new Date(queueable.waitUntil.getTime() + 3600000), // 延迟 1 小时
      }
    },
  },
}]
```

### 1.4 任务执行调度机制

任务执行通过四种方式实现：

#### 方式一：Bin Script（推荐用于专用服务器）

```bash
# 基本用法 - 运行默认队列的任务
pnpm payload jobs:run

# 带自定义队列和限制
pnpm payload jobs:run --queue myQueue --limit 15

# 定时运行
pnpm payload jobs:run --cron "*/5 * * * *" --queue myQueue

# 同时处理调度（入队和运行）
pnpm payload jobs:run --cron "*/5 * * * *" --queue myQueue --handle-schedules
```

**优势**：
- 独立于 Next.js 进程运行
- 便于部署、扩展和独立管理
- 无 Next.js 开销，更轻量快速

#### 方式二：autoRun（专用服务器备选）

在 `jobs.autoRun` 配置中定义 cron 任务，在 Next.js 进程内自动执行：

```typescript
jobs: {
  autoRun: [
    {
      cron: '*/5 * * * *',  // 每5分钟检查一次
      queue: 'default',     // 处理 'default' 队列
      limit: 50,
    },
    {
      cron: '* * * * *',    // 每分钟检查一次
      queue: 'nightly',     // 处理 'nightly' 队列
      limit: 100,
    },
  ],
  shouldAutoRun: async (payload) => {
    return process.env.ENABLE_JOB_WORKERS === 'true'
  },
}
```

**注意**：`autoRun` 仅执行已入队的任务，不会自动入队新任务（除非未设置 `disableScheduling`）。

#### 方式三：Endpoint（无服务器平台）

通过 `/api/payload-jobs/run` 端点执行任务：

```typescript
await fetch('/api/payload-jobs/run?limit=100&queue=nightly', {
  method: 'GET',
  headers: {
    Authorization: `Bearer ${token}`,
  },
})
```

**Vercel Cron 示例**：

```json
{
  "crons": [
    {
      "path": "/api/payload-jobs/run",
      "schedule": "*/5 * * * *"
    }
  ]
}
```

#### 方式四：Local API（编程控制）

从服务端代码编程执行任务：

```typescript
// 运行所有任务
const results = await payload.jobs.run()

// 自定义队列和限制
await payload.jobs.run({ queue: 'nightly', limit: 100 })

// 运行所有队列的任务
await payload.jobs.run({ allQueues: true })

// 按 ID 运行单个任务
const results = await payload.jobs.runByID({ id: myJobID })
```

### 1.5 任务处理流程

任务执行的核心逻辑位于 `packages/payload/src/queues/operations/runJobs/index.ts:82-541`：

```
任务执行流程：
1. 权限检查
2. 构建查询条件：
   - completedAt 不存在（未完成）
   - hasError 为 false（无最终错误）
   - processing 为 false（未在处理中）
   - waitUntil 不存在或已过期
   - 队列匹配
   - 并发控制检查（如果启用）
3. 原子性更新：将符合条件的任务标记为 processing: true
4. 并发键去重（同一批次中同键任务只保留一个）
5. 执行任务（支持并行或串行）
6. 成功后可选删除任务（deleteJobOnComplete）
7. 返回执行结果
```

### 1.6 处理顺序控制

默认采用 FIFO（先进先出）顺序，可通过以下方式配置：

```typescript
// 全局配置
jobs: {
  processingOrder: '-createdAt',  // LIFO 后进先出
}

// 按队列配置
jobs: {
  processingOrder: {
    default: 'createdAt',       // FIFO
    queues: {
      nightly: '-createdAt',    // LIFO
    },
  },
}

// 动态函数配置
jobs: {
  processingOrder: ({ queue }) => {
    if (queue === 'myQueue') return '-createdAt'
    return 'createdAt'
  },
}
```

---

## 二、任务执行与主服务通信机制

### 2.1 基于数据库的状态同步

Payload 的任务队列采用**数据库作为中间层**的通信模式，任务执行器与主服务通过 `payload-jobs` 集合进行状态同步。

### 2.2 核心通信字段

任务状态通过以下字段进行协调：

| 字段 | 作用 | 状态转换 |
|------|------|----------|
| `processing` | 标记任务是否正在被处理 | false → true（任务被拾取）→ false（任务完成/失败） |
| `completedAt` | 任务完成时间 | null → 时间戳（成功完成） |
| `hasError` | 标记最终失败 | false → true（达到最大重试次数） |
| `waitUntil` | 下次执行时间 | null/过期时间 → 新的退避时间 |
| `totalTried` | 尝试次数 | 递增 |
| `log` | 执行日志 | 追加任务执行记录 |
| `taskStatus` | 任务状态摘要 | 从 log 动态计算 |

### 2.3 任务拾取的原子性

为避免多个 worker 拾取同一任务，采用原子性更新策略（`packages/payload/src/queues/operations/runJobs/index.ts:202-248`）：

```typescript
// 方式一：按 ID 运行单个任务
const job = await updateJob({
  id,
  data: { processing: true },
  req,
  returning: true,
})

// 方式二：批量更新待处理任务
const updatedDocs = await updateJobs({
  data: { processing: true },
  limit,
  req,
  returning: true,
  sort: processingOrder ?? defaultProcessingOrder,
  where: { and: [/* 条件 */] },
})
```

**关键设计**：
- 使用 `updateJob`/`updateJobs` 一次性完成"查询 + 标记"操作
- 利用数据库的原子更新特性，防止竞争条件
- 只有成功更新 `processing: true` 的任务才会被执行

### 2.4 独立执行上下文

任务执行时创建独立的请求上下文（`packages/payload/src/queues/operations/runJobs/index.ts:331`）：

```typescript
const jobReq = isolateObjectProperty(req, 'transactionID')
```

这确保：
- 任务执行与主请求事务隔离
- 每个任务有独立的数据库事务
- 任务失败不影响主服务事务

### 2.5 状态更新机制

任务状态通过 `updateJob` 工具函数更新（`packages/payload/src/queues/utilities/updateJob.ts:46-101`）：

```typescript
export async function updateJobs({
  id, data, limit, req, returning, sort, where
}: RunJobsArgs): Promise<Job[] | null> {
  // 1. 开启事务（非 MongoDB 时）
  const jobReq = {
    transactionID: req.payload.db.name !== 'mongoose'
      ? await req.payload.db.beginTransaction()
      : undefined,
  }

  // 2. 确保 updatedAt 更新
  if (typeof data.updatedAt === 'undefined') {
    data.updatedAt = getCurrentDate().toISOString()
  }

  // 3. 调用数据库层的 updateJobs
  const updatedJobs = await req.payload.db.updateJobs(args)

  // 4. 提交事务
  if (req.payload.db.name !== 'mongoose' && jobReq.transactionID) {
    await req.payload.db.commitTransaction(jobReq.transactionID)
  }

  return updatedJobs?.map(updatedJob => jobAfterRead({ config, doc: updatedJob }))
}
```

### 2.6 通信时序图

```
主服务/API 进程                          Worker 进程
      |                                      |
      |  1. payload.jobs.queue()             |
      |  ──────────────────────────────────> |
      |         (写入 payload-jobs)          |
      |                                      |
      |  2. 等待调度时机                      |
      |     (cron / autoRun / 端点调用)      |
      |                                      |
      |  3. Worker 开始查询可执行任务         |
      |  <────────────────────────────────── |
      |         (SELECT ... WHERE ...)       |
      |                                      |
      |  4. 原子性标记 processing=true        |
      |  <────────────────────────────────── |
      |         (UPDATE ... RETURNING)       |
      |                                      |
      |  5. Worker 执行任务逻辑               |
      |                                      |
      |  6. 更新任务状态                      |
      |  <────────────────────────────────── |
      |    (completedAt/hasError/waitUntil)  |
      |                                      |
      |  7. 主服务可查询任务状态              |
      |  (payload.findByID('payload-jobs'))  |
      |  ──────────────────────────────────> |
      |         (读取最新状态)                |
```

---

## 三、失败重试机制与跨服务协调

### 3.1 重试配置层次

重试配置可在三个层次定义（优先级从高到低）：

1. **任务级别**：单个 Task 的 `retries` 配置
2. **工作流级别**：Workflow 的 `retries` 配置
3. **默认值**：无配置时不重试

### 3.2 重试配置格式

```typescript
// 简单格式 - 仅指定重试次数
retries: 2

// 完整配置
retries: {
  attempts: 3,           // 最大重试次数
  backoff: {
    type: 'exponential', // 退避策略
    delay: 1000,         // 初始延迟（毫秒）
    maxDelay: 60000,     // 最大延迟
  },
  shouldRestore: true,   // 是否恢复已成功的任务
}
```

### 3.3 任务错误处理流程

任务失败时的处理逻辑位于 `packages/payload/src/queues/errors/handleTaskError.ts:14-188`：

```
任务错误处理流程：

1. 调用 taskConfig.onFail 回调（如果配置）
   
2. 确定最大重试次数：
   - 如果任务级 retries.attempts 已定义，使用该值
   - 否则继承工作流级 retries 配置
   - 都没有则 maxRetries = 0（不重试）

3. 检查是否达到最大重试次数：
   if (taskStatus.totalTried >= maxRetries):
     - 标记 hasError = true（最终失败）
     - 记录错误日志
     - 返回 hasFinalError = true
   else:
     - 继续重试逻辑

4. 计算任务级退避时间：
   taskWaitUntil = calculateBackoffWaitUntil(retriesConfig, taskStatus.totalTried)

5. 检查工作流级别重试：
   workflowRetry = getWorkflowRetryBehavior(job, workflowConfig.retries)

6. 合并退避时间（取较大值）：
   if (taskWaitUntil > job.waitUntil):
     job.waitUntil = taskWaitUntil
   if (workflowRetry.waitUntil > job.waitUntil):
     job.waitUntil = workflowRetry.waitUntil

7. 更新任务状态：
   - totalTried + 1
   - processing = false
   - waitUntil = 新的执行时间
   - hasError = workflowRetry.hasFinalError
   - 追加执行日志
```

### 3.4 工作流错误处理流程

工作流级别错误（非任务错误）的处理位于 `packages/payload/src/queues/errors/handleWorkflowError.ts:17-98`：

```
工作流错误处理流程：

1. 检查是否配置了工作流级重试：
   if (workflowConfig.retries === undefined):
     - 直接标记为最终失败
     - 返回 hasFinalError = true

2. 检查工作流重试次数：
   if (job.totalTried >= workflowRetries):
     - 标记为最终失败
     - 返回 hasFinalError = true

3. 计算工作流级退避时间：
   waitUntil = calculateBackoffWaitUntil(workflowConfig.retries, job.totalTried)

4. 更新任务状态：
   - totalTried + 1
   - processing = false
   - waitUntil = 退避时间
   - hasError = false（可重试）
   - 记录错误
```

### 3.5 退避时间计算

退避时间计算逻辑位于 `packages/payload/src/queues/errors/calculateBackoffWaitUntil.ts`：

```typescript
export function calculateBackoffWaitUntil({
  retriesConfig,
  totalTried,
}: {
  retriesConfig?: number | RetryConfig
  totalTried: number
}): Date {
  const now = getCurrentDate()
  
  if (!retriesConfig) {
    return now  // 无退避，立即重试
  }

  const attempts = typeof retriesConfig === 'object' 
    ? retriesConfig.attempts 
    : retriesConfig
  
  const backoff = typeof retriesConfig === 'object' 
    ? retriesConfig.backoff 
    : undefined

  if (!backoff) {
    return now  // 无退避配置
  }

  // 指数退避：delay * (2 ^ totalTried)
  const baseDelay = backoff.delay ?? 1000
  const exponentialDelay = baseDelay * Math.pow(2, totalTried)
  
  // 不超过最大延迟
  const actualDelay = backoff.maxDelay 
    ? Math.min(exponentialDelay, backoff.maxDelay)
    : exponentialDelay

  return new Date(now.getTime() + actualDelay)
}
```

### 3.6 重试行为决策

`getWorkflowRetryBehavior` 函数（`packages/payload/src/queues/errors/getWorkflowRetryBehavior.ts:11-63`）决定是否可重试：

```typescript
export function getWorkflowRetryBehavior({
  job,
  retriesConfig,
}: {
  job: Job
  retriesConfig?: number | RetryConfig
}): { hasFinalError: boolean; waitUntil?: Date } {
  
  const maxWorkflowRetries = typeof retriesConfig === 'object'
    ? retriesConfig.attempts
    : retriesConfig

  // 达到最大重试次数
  if (maxWorkflowRetries !== undefined && 
      job.totalTried >= maxWorkflowRetries) {
    return { hasFinalError: true, maxWorkflowRetries }
  }

  // 无重试配置
  if (!retriesConfig) {
    return { hasFinalError: false }
  }

  // 计算退避时间
  const waitUntil = calculateBackoffWaitUntil({
    retriesConfig,
    totalTried: job.totalTried ?? 0,
  })

  return { hasFinalError: false, maxWorkflowRetries, waitUntil }
}
```

### 3.7 跨服务协调机制

由于 Payload 采用**数据库中心化**的设计，重试机制天然支持跨服务协调：

#### 协调原理

```
服务实例 A                              服务实例 B                          数据库
     |                                      |                                |
     |  1. 查询待执行任务                    |                                |
     |  ───────────────────────────────────────────────────────────────────> |
     |         (WHERE processing=false AND waitUntil < now)                  |
     |                                      |                                |
     |  2. 原子性更新任务状态                |                                |
     |  <─────────────────────────────────────────────────────────────────── |
     |         (UPDATE processing=true WHERE ...)                            |
     |         (只返回被当前实例更新的行)     |                                |
     |                                      |                                |
     |  3. 执行任务（失败）                  |                                |
     |                                      |                                |
     |  4. 更新任务状态（可重试）            |                                |
     |  ───────────────────────────────────────────────────────────────────> |
     |         (processing=false, waitUntil=未来时间, totalTried++)           |
     |                                      |                                |
     |                                      |  5. 等待退避时间后，实例 B 查询  |
     |                                      |  <───────────────────────────── |
     |                                      |         (检查 waitUntil 是否过期)|
     |                                      |                                |
     |                                      |  6. 实例 B 拾取并重试该任务     |
     |                                      |  ─────────────────────────────> |
```

#### 关键协调点

1. **原子性拾取**：通过 `UPDATE ... RETURNING` 确保同一时刻只有一个 worker 能拾取任务
2. **退避时间同步**：`waitUntil` 字段存储在数据库中，所有 worker 都能看到
3. **重试计数共享**：`totalTried` 字段全局可见，确保跨服务的重试次数准确
4. **最终失败标记**：`hasError=true` 后，任何 worker 都不会再拾取该任务
5. **并发控制**：`concurrencyKey` 确保同键任务不会并行执行

### 3.8 取消机制

任务取消通过 `JobCancelledError` 实现（`packages/payload/src/queues/errors/index.ts`）：

```typescript
// 方式一：通过 Local API 取消
await payload.jobs.cancelByID({ id: jobId })
await payload.jobs.cancel({ where: { workflowSlug: { equals: 'createPost' } } })

// 方式二：在任务处理中主动抛出
throw new JobCancelledError('Job was cancelled')
```

取消后任务状态：
- `error.cancelled = true`
- `hasError = true`（不会再被重试）
- `processing = false`
- `waitUntil = null`

### 3.9 重试配置示例

```typescript
export default buildConfig({
  jobs: {
    tasks: [
      {
        slug: 'syncToThirdParty',
        retries: {
          attempts: 3,
          backoff: {
            type: 'exponential',
            delay: 1000,    // 第一次重试等 1 秒
            maxDelay: 60000, // 最多等 60 秒
          },
        },
        handler: async ({ input, req }) => {
          const response = await fetch('https://api.example.com/sync', {
            method: 'POST',
            body: JSON.stringify(input),
          })
          if (!response.ok) {
            throw new Error(`API error: ${response.status}`)
          }
          return { output: { synced: true } }
        },
      },
    ],
  },
})
```

**重试时序**：
- 第 0 次尝试（首次执行）：立即执行
- 第 1 次重试：失败后等 1 秒（1000 * 2^0）
- 第 2 次重试：再失败后等 2 秒（1000 * 2^1）
- 第 3 次重试：再失败后等 4 秒（1000 * 2^2）
- 第 4 次尝试：再失败后达到 maxRetries=3，标记为最终失败

---

## 四、并发控制机制

### 4.1 并发控制配置

```typescript
// 启用并发控制
jobs: {
  enableConcurrencyControl: true,
  
  tasks: [
    {
      slug: 'processPayment',
      concurrency: {
        key: ({ input }) => `order-${input.orderId}`,  // 相同订单的任务串行执行
        supersedes: true,  // 新任务取代旧任务
      },
      handler: async ({ input }) => { /* ... */ },
    },
  ],
}
```

### 4.2 并发控制执行流程

1. **任务入队时**：
   - 计算 `concurrencyKey`
   - 如果 `supersedes=true`，删除同键的待处理旧任务

2. **任务拾取时**（`packages/payload/src/queues/operations/runJobs/index.ts:156-194`）：
   ```typescript
   // 查询正在运行的同键任务
   const runningJobs = await payload.db.find({
     collection: jobsCollectionSlug,
     where: {
       and: [
         { processing: { equals: true } },
         { concurrencyKey: { exists: true } }
       ],
     },
   })
   
   // 排除同键任务
   if (runningConcurrencyKeys.size > 0) {
     and.push({
       or: [
         { concurrencyKey: { exists: false } },
         { concurrencyKey: { not_in: [...runningConcurrencyKeys] } },
       ],
     })
   }
   ```

3. **同一批次去重**（`packages/payload/src/queues/operations/runJobs/index.ts:258-292`）：
   ```typescript
   const seenConcurrencyKeys = new Set<string>()
   const jobsToRun: Job[] = []
   const jobsToRelease: Job[] = []
   
   for (const job of jobs) {
     if (job.concurrencyKey) {
       if (seenConcurrencyKeys.has(job.concurrencyKey)) {
         jobsToRelease.push(job)  // 释放回待处理状态
       } else {
         seenConcurrencyKeys.add(job.concurrencyKey)
         jobsToRun.push(job)
       }
     } else {
       jobsToRun.push(job)
     }
   }
   
   // 释放重复任务
   if (jobsToRelease.length > 0) {
     await updateJobs({
       data: { processing: false },
       where: { id: { in: releaseIds } },
     })
   }
   ```

---

## 五、Worker 崩溃后的任务状态

### 5.1 代码证据分析

**关键发现：Payload 当前版本没有内置的 Worker 崩溃恢复机制**

#### 证据 1：任务拾取查询条件

```typescript
// packages/payload/src/queues/operations/runJobs/index.ts:112-142
const and: Where[] = [
  {
    completedAt: { exists: false },
  },
  {
    hasError: { not_equals: true },
  },
  {
    processing: { equals: false },  // ← 关键：只拾取 processing=false 的任务
  },
  {
    or: [
      { waitUntil: { exists: false } },
      { waitUntil: { less_than: getCurrentDate().toISOString() } },
    ],
  },
]
```

#### 证据 2：没有 heartbeat 或超时清理机制

搜索整个代码库，未发现以下机制：
- 没有 `heartbeat` / `lastAlive` 字段
- 没有根据 `updatedAt` 超时清理 `processing=true` 任务的逻辑
- 没有 `lockTimeout` 配置选项
- 没有后台清理进程扫描"卡住"的任务

#### 证据 3：测试代码中没有崩溃恢复测试

查看 `test/queues/int.spec.ts`，没有关于"worker 崩溃后任务恢复"的测试用例。

### 5.2 实际行为

**当 Worker 崩溃时**：

```
时间线：
T1: Worker A 拾取任务 J1 → processing=true
T2: Worker A 崩溃（进程被 kill / 机器宕机）
T3: Worker B 启动，查询可执行任务
    → 查询条件：processing=false
    → 任务 J1 仍为 processing=true
    → Worker B 无法拾取 J1

结果：
- 任务 J1 永远卡在 processing=true 状态
- 除非手动干预，否则不会被重试
```

### 5.3 手动恢复方法

**方案 1：手动更新数据库**

```sql
-- 将 processing=true 且未完成的任务重置为待处理状态
UPDATE payload-jobs 
SET processing = false 
WHERE processing = true 
  AND completedAt IS NULL;
```

**方案 2：通过 Payload API 重置**

```typescript
// 查询卡住的任务
const stuckJobs = await payload.find({
  collection: 'payload-jobs',
  where: {
    and: [
      { processing: { equals: true } },
      { completedAt: { exists: false } },
    ],
  },
})

// 重置为待处理状态
for (const job of stuckJobs.docs) {
  await payload.update({
    collection: 'payload-jobs',
    id: job.id,
    data: { processing: false },
  })
}
```

**方案 3：自定义清理任务**

```typescript
// 在应用启动时添加一个定时清理任务
jobs: {
  tasks: [
    {
      slug: 'cleanupStuckJobs',
      schedule: [{ cron: '*/10 * * * *', queue: 'system' }],  // 每 10 分钟
      handler: async ({ req }) => {
        const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000)
        
        const stuckJobs = await req.payload.find({
          collection: 'payload-jobs',
          where: {
            and: [
              { processing: { equals: true } },
              { completedAt: { exists: false } },
              { updatedAt: { less_than: fiveMinutesAgo.toISOString() } },
            ],
          },
        })

        for (const job of stuckJobs.docs) {
          await req.payload.update({
            collection: 'payload-jobs',
            id: job.id,
            data: { processing: false },
          })
          req.payload.logger.warn(`Reset stuck job: ${job.id}`)
        }

        return { output: { resetCount: stuckJobs.totalDocs } }
      },
    },
  ],
}
```

### 5.4 风险评估

| 场景 | 风险 | 建议 |
|------|------|------|
| 单 Worker 部署 | 任务可能永久卡住 | 务必实现自定义清理逻辑 |
| 多 Worker 部署 | 单个 Worker 崩溃的任务会卡住，但其他 Worker 继续工作 | 仍建议实现清理逻辑 |
| 使用 Bin Script | 每次新运行都可能拾取不到卡住的任务 | 建议在启动时清理 |
| 使用 autoRun | 定时任务可能永远跳过卡住的任务 | 实现 `shouldAutoRun` 检查 |

---

## 六、总结

Payload 的任务队列系统采用**数据库中心化**的架构设计，具有以下特点：

### 6.1 架构特点汇总

| 方面 | 实现方式 | 优势 | 注意事项 |
|------|----------|------|----------|
| **任务存储** | `payload-jobs` 集合 | 持久化、可查询、状态透明 | - |
| **调度方式** | Bin Script / autoRun / Endpoint / Local API | 灵活适配不同部署环境 | - |
| **定时调度** | `handleSchedules` + `payload-jobs-stats` 全局 | 无需额外调度服务 | 依赖 `lastScheduledRun` 记录 |
| **同队列防重复** | `defaultBeforeSchedule` 检查 | 防止同一任务重复入队 | `waitUntil` 不参与防重复检查 |
| **通信机制** | 数据库字段状态同步 | 跨服务天然协调，无额外基础设施 | - |
| **原子拾取** | UPDATE + RETURNING | 防止多 worker 重复执行 | - |
| **重试机制** | 任务级 + 工作流级双层配置 | 灵活控制重试策略 | - |
| **退避策略** | 指数退避（可配置最大延迟） | 避免瞬时故障导致的重试风暴 | - |
| **并发控制** | `concurrencyKey` + 运行时检查 | 确保同键任务串行执行 | - |
| **取消机制** | `JobCancelledError` + `hasError` 标记 | 支持主动取消和业务逻辑取消 | - |
| **崩溃恢复** | ❌ 无内置机制 | - | **需要手动实现清理逻辑** |

### 6.2 关键设计决策

#### 决策 1：数据库中心化 vs 消息队列

**选择**：使用数据库而非独立消息队列（如 Redis、RabbitMQ）

**理由**：
- 减少基础设施依赖
- 任务状态与业务数据在同一存储
- 事务一致性更好
- 查询和调试更方便

**代价**：
- 没有内置的消息确认（ACK）机制
- Worker 崩溃后任务可能卡住
- 需要手动实现清理逻辑

#### 决策 2：processing 标记而非锁机制

**选择**：使用 `processing: boolean` 字段标记任务状态

**理由**：
- 实现简单，数据库原生支持
- `UPDATE ... RETURNING` 原子性足够
- 易于理解和调试

**代价**：
- 没有 TTL（超时）机制
- 崩溃后任务状态不会自动恢复

#### 决策 3：防重复检查基于任务存在性

**选择**：`countRunnableOrActiveJobsForQueue` 检查同任务是否存在

**理由**：
- 简单直接
- 防止调度系统重复创建相同任务

**代价**：
- `waitUntil` 不参与检查
- 有任务"卡住"时新调度不会入队

### 6.3 生产环境建议

#### 必须做的

1. **实现崩溃恢复机制**：添加定时清理 `processing=true` 超过阈值的任务
2. **监控 `processing=true` 任务**：告警长时间处于处理中状态的任务
3. **使用独立 Worker 进程**：优先使用 Bin Script 而非 autoRun

#### 建议做的

1. **配置合理的重试策略**：根据任务特性设置 `attempts` 和 `backoff`
2. **使用队列隔离**：将不同优先级的任务放入不同队列
3. **配置 `deleteJobOnComplete`**：根据需要选择是否保留成功任务记录
4. **启用 `enableConcurrencyControl`**：对于需要串行执行的任务

#### 生产部署示例

```yaml
# docker-compose.yml
services:
  nextjs:
    command: pnpm start
    environment:
      - ENABLE_JOB_WORKERS=false  # 主服务不处理任务

  worker-default:
    command: pnpm payload jobs:run --cron "*/5 * * * *" --queue default
    restart: always  # 崩溃后自动重启

  worker-nightly:
    command: pnpm payload jobs:run --cron "* * * * *" --queue nightly --handle-schedules
    restart: always

  worker-cleanup:
    command: pnpm payload jobs:run --cron "*/10 * * * *" --queue system
    restart: always
```

### 6.4 待改进点（基于代码分析）

1. **缺少心跳机制**：没有 `lastHeartbeat` 字段来检测崩溃的 Worker
2. **缺少任务超时**：没有机制可以终止执行时间过长的任务
3. **缺少死信队列**：最终失败的任务没有专门的存储位置
4. **缺少任务优先级**：FIFO/LIFO 之外没有更细粒度的优先级控制
5. **缺少批量处理优化**：每个任务独立事务，大量小任务时效率较低
