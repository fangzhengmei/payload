# Payload 鉴权机制策略分析报告

## 1. 概述

Payload CMS 提供了一套灵活且可扩展的鉴权机制，支持多种认证策略，能够满足不同类型客户端（浏览器、移动端、第三方服务等）的需求。本报告深入分析 Payload 的鉴权架构、多种策略支持、JWT 与会话 Cookie 在不同客户端中的处理差异，以及中间件层的统一验证机制。

---

## 2. 鉴权策略架构

### 2.1 核心策略类型

Payload 支持以下三种主要的鉴权策略：

| 策略类型 | 策略名称 | 适用场景 |
|---------|---------|---------|
| **本地 JWT 策略** | `local-jwt` | 基于用户名/密码登录的标准应用 |
| **API Key 策略** | `{collection}-api-key` | 服务间通信、自动化任务、第三方集成 |
| **自定义策略** | 用户自定义 | 特殊认证需求（如 OAuth、双因素认证等） |

### 2.2 策略注册与优先级

策略在 Payload 初始化时按以下顺序注册（见 `packages/payload/src/index.ts:958-989`）：

```typescript
// 1. 首先添加用户自定义策略
if (collection.auth.strategies.length > 0) {
  authStrategies.push(...collection.auth.strategies)
}

// 2. 然后添加 API Key 策略（如果启用）
if (collection.auth?.useAPIKey) {
  authStrategies.push({
    name: `${collection.slug}-api-key`,
    authenticate: APIKeyAuthentication(collection),
  })
}

// 3. 最后添加 JWT 策略
if (jwtStrategyEnabled) {
  authStrategies.push({
    name: 'local-jwt',
    authenticate: JWTAuthentication,
  })
}
```

**优先级说明**：
- 自定义策略 > API Key 策略 > JWT 策略
- 执行时按顺序遍历，第一个成功验证的策略立即返回结果

---

## 3. JWT 与会话 Cookie 详解

### 3.1 JWT 令牌结构

JWT 令牌包含以下核心字段（见 `packages/payload/src/auth/getFieldsToSign.ts:116-141`）：

```typescript
{
  id: string,                    // 用户 ID
  collection: string,            // 所属集合名称
  email: string,                 // 用户邮箱
  sid?: string,                  // 会话 ID（仅当启用会话时）
  // 自定义字段（通过 saveToJWT 配置）
  roles?: string[],
  // ... 其他 saveToJWT 字段
}
```

**JWT 签名算法**：
- 使用 HS256（HMAC SHA-256）
- 密钥处理：用户配置的 `config.secret` 在 **Payload 初始化时** 进行 SHA-256 哈希后取前 32 个字符，存入 `payload.secret`。JWT 签名和验证时直接使用该已处理值，不再做额外转换
- 令牌默认有效期：2 小时（7200 秒）

**密钥处理流程**：

```
用户配置 config.secret
       ↓
Payload 初始化 (index.ts:844)
       ↓
crypto.createHash('sha256')
  .update(config.secret)
  .digest('hex')
  .slice(0, 32)
       ↓
存储为 payload.secret (32 字符十六进制字符串)
       ↓
JWT 签名/验证时直接使用 TextEncoder.encode(payload.secret)
```

**关键事实**：
- SHA-256 + 截断只发生 **一次**（初始化阶段），而非每次 JWT 操作
- `payload.secret` 始终是 **32 字符的十六进制字符串**（字符集：`[0-9a-f]`）
- JWT 签名/验证时通过 `TextEncoder.encode()` 转为 **32 字节 UTF-8 字节序列** 传入 HMAC-SHA256
- 从密码学熵角度：每个十六进制字符贡献 4 位熵，总计 **128 位有效熵**
- 外部服务验证 Payload JWT 时，需要对原始 secret 做相同的预处理（见 `docs/authentication/jwt.mdx`）

#### 3.1.1 此表述误判带来的安全判断偏差

> "32 字符十六进制字符串等价于 16 字节二进制数据"——这句看似合理的简化表述，实际会在三个关键维度上误导安全判断。

##### 偏差一：跨服务验签不一致

**误判逻辑**：认为"32 个十六进制字符 = 16 字节"，外部服务验证时可能会：
```typescript
// 错误做法：把十六进制字符串解码为二进制
const secret = crypto
  .createHash('sha256')
  .update(process.env.PAYLOAD_SECRET)
  .digest()           // 32 字节二进制 Buffer
  .slice(0, 16)       // 截断为 16 字节（因为"32 hex = 16 bytes"）
```

**真实情况**：Payload 不会解码十六进制，而是直接对字符串做 UTF-8 编码：
```typescript
// packages/payload/src/index.ts:844
this.secret = crypto
  .createHash('sha256')
  .update(this.config.secret)
  .digest('hex')      // 64 字符十六进制字符串
  .slice(0, 32)       // 32 字符十六进制字符串，例如 "a1b2c3d4..."

// packages/payload/src/auth/jwt.ts:12
const secretKey = new TextEncoder().encode(secret)
// 结果：32 字节 UTF-8，每个字符独立编码，例如 [0x61, 0x31, 0x62, 0x32, ...]
```

**后果**：
- 外部服务用"十六进制解码"得到的密钥与 Payload 内部使用的密钥完全不同
- JWT 验证永远失败，且难以排查（两边都声称"用了相同的 secret"）
- 运维团队可能反复检查 `PAYLOAD_SECRET` 环境变量，却忽略了编码方式差异

**正确做法**：
```typescript
// 外部服务必须完全复刻 Payload 的处理逻辑
const payloadSecret = crypto
  .createHash('sha256')
  .update(process.env.PAYLOAD_SECRET)
  .digest('hex')      // 十六进制字符串
  .slice(0, 32)       // 保留为字符串，不要 decode

// 然后直接用这个字符串验证 JWT（jose 等库会自动处理编码）
const { payload } = await jwtVerify(token, new TextEncoder().encode(payloadSecret))
```

---

#### 偏差二：密钥长度认知偏差

**误判逻辑**：
- "32 字符十六进制 = 16 字节" → 认为密钥太短，不满足 HS256 的推荐长度（≥ 32 字节）
- 或者反向：认为既然输入的 `config.secret` 可以很长，最终密钥熵也很高

**真实情况**：需要区分三个概念：

| 概念 | 数值 | 说明 |
|-----|------|-----|
| **原始 `config.secret`** | 用户配置 | 可以任意长度，但不决定最终密钥强度 |
| **`payload.secret` 字符串** | 32 字符 | 十六进制字符，取值范围 `[0-9a-f]` |
| **传入 HMAC 的字节数** | 32 字节 | `TextEncoder.encode()` 后，每个字符 1 字节 |
| **密码学有效熵** | **128 位** | 每个十六进制字符 4 位：32 × 4 = 128 位 |

**关键理解**：
- HS256 的密钥可以是任意长度字节序列，HMAC 会内部处理填充
- 但**密码学强度取决于熵**，而非字节数
- 128 位熵对大多数应用足够，但并非"工业级"的 256 位
- 无论原始 `config.secret` 多长（100 字节、1000 字节），经过 `sha256().digest('hex').slice(0,32)` 后，**有效熵固定为 128 位**

**安全影响**：
- 如果系统设计依赖"超长 secret 带来的安全边际"，这个假设不成立
- 128 位对抗量子计算的缓冲区比 256 位小
- 但配合短期 JWT（默认 2 小时），128 位在可预见未来仍然安全

---

#### 偏差三：排障方向误导

**误判场景**：JWT 验证失败，开发人员开始排查：

| 排查方向（基于误判） | 实际问题 | 浪费的时间 |
|-------------------|---------|-----------|
| "是不是 secret 长度不对？检查是不是 32 字节" | 外部服务用了 `Buffer.from(hex, 'hex')` 解码 | 数小时 |
| "是不是编码问题？试试 base64 / UTF-16" | 问题是十六进制字符串 vs 二进制解码 | 反复试错 |
| "是不是 Payload 版本差异？旧版本用了不同算法" | 问题是文档理解偏差，而非版本问题 | 查 changelog、回滚测试 |

**典型排障误区**：
```typescript
// 开发人员 A 的验证代码（错误）
const key1 = Buffer.from(payloadSecret.slice(0, 32), 'hex')  // 16 字节！

// 开发人员 B 的验证代码（错误）
const key2 = crypto.createHash('sha256').update(secret).digest().slice(0, 32)  // 32 字节二进制，不是十六进制

// Payload 实际使用的（正确）
const key3 = new TextEncoder().encode(payloadSecret)  // 32 字节 UTF-8 字符编码
```

**这三个密钥完全不同**，但开发人员可能坚信"逻辑是对的"，因为：
- 都对 `PAYLOAD_SECRET` 做了 SHA-256
- 都取了 32 的某种形式
- 文档里说"process secret using SHA-256 hash and takes the first 32 characters"

**快速自检方法**：
```typescript
// 验证你的处理逻辑是否正确
function verifyPayloadSecretProcessing(originalSecret: string): boolean {
  const processed = crypto
    .createHash('sha256')
    .update(originalSecret)
    .digest('hex')
    .slice(0, 32)
  
  // 检查：应该是 32 个字符，且都是十六进制
  return processed.length === 32 && /^[0-9a-f]{32}$/.test(processed)
}

// 然后直接用这个字符串，不要做任何解码
```

---

### 3.2 会话机制

Payload 支持可选的服务端会话管理（见 `packages/payload/src/auth/sessions.ts`）：

#### 会话启用配置
```typescript
auth: {
  useSessions: true  // 默认为 true
}
```

#### 会话生命周期
1. **登录时创建会话**：
   - 生成唯一的会话 ID（UUID v4）
   - 存储在用户文档的 `sessions` 数组中
   - 包含 `id`、`createdAt`、`expiresAt` 字段
   - JWT 令牌中包含 `sid` 声明

2. **验证时检查会话**：
   - 从 JWT 中提取 `sid`
   - 验证该会话 ID 是否存在于用户的 `sessions` 数组中
   - 检查会话是否过期

3. **登出时撤销会话**：
   - 从用户的 `sessions` 数组中移除对应的会话 ID

#### 会话验证逻辑（见 `packages/payload/src/auth/strategies/jwt.ts:104-114`）
```typescript
if (collection!.config.auth.useSessions) {
  const existingSession = (user.sessions || []).find(
    ({ id }) => id === decodedPayload.sid
  )
  
  if (!existingSession || !decodedPayload.sid) {
    return { user: null }
  }
  
  user._sid = decodedPayload.sid
}
```

### 3.3 JWT 提取策略

Payload 支持从多个位置提取 JWT 令牌，提取顺序可配置（见 `packages/payload/src/auth/extractJWT.ts`）：

#### 提取方法

| 方法 | 来源 | 格式 | 适用客户端 |
|-----|------|------|-----------|
| `JWT` | Authorization 头 | `Authorization: JWT <token>` | 自定义客户端 |
| `Bearer` | Authorization 头 | `Authorization: Bearer <token>` | 标准 OAuth 2.0 客户端 |
| `cookie` | HTTP Cookie | `{cookiePrefix}-token=<token>` | 浏览器客户端 |

#### 默认提取顺序
```typescript
jwtOrder: ['JWT', 'Bearer', 'cookie']  // 见 packages/payload/src/config/defaults.ts:38
```

#### Cookie 提取的 CSRF 防护

Cookie 提取包含严格的 CSRF 保护机制（见 `packages/payload/src/auth/extractJWT.ts:19-53`）：

```
请求存在 Origin 头 → 检查是否在 CSRF 白名单中
    ├── 在白名单中 → 允许
    └── 不在白名单中 → 拒绝

请求没有 Origin 头（非浏览器或同域请求）
    ├── 未配置 CSRF 白名单 → 允许
    └── 已配置 CSRF 白名单 → 检查 Sec-Fetch-Site
              ├── 'same-origin' / 'same-site' / 'none' → 允许
              └── 其他值 → 拒绝
```

---

## 4. 不同客户端类型的处理差异

### 4.1 浏览器客户端

**推荐方式**：Cookie + 会话

**特点**：
1. **自动管理**：浏览器自动处理 Cookie 的存储和发送
2. **HTTP Only**：Cookie 标记为 HttpOnly，防止 XSS 攻击窃取令牌
3. **安全标志**：支持 Secure、SameSite 等安全属性
4. **CSRF 防护**：内置多层次 CSRF 保护

**登录流程**：
```
1. 用户提交登录凭证
2. 服务端验证成功后：
   - 创建会话（sid）
   - 生成包含 sid 的 JWT
   - 通过 Set-Cookie 响应头发送给浏览器
3. 后续请求：
   - 浏览器自动携带 Cookie
   - 服务端验证 JWT + 会话有效性
```

**Cookie 配置选项**（见 `packages/payload/src/auth/types.ts:214-222`）：
```typescript
auth: {
  cookies: {
    domain?: string,
    sameSite?: 'Lax' | 'None' | 'Strict' | boolean,
    secure?: boolean
  }
}
```

### 4.2 移动端客户端（Native Apps）

**推荐方式**：Authorization Header (Bearer/JWT)

**特点**：
1. **手动管理**：需要应用代码存储和管理令牌
2. **灵活存储**：可选择 Keychain（iOS）/ Keystore（Android）等安全存储
3. **无 Cookie 限制**：不依赖浏览器 Cookie 机制
4. **无需 CSRF**：原生应用不存在 CSRF 风险

**请求示例**：
```typescript
fetch('https://api.example.com/data', {
  headers: {
    'Authorization': `Bearer ${token}`,
    // 或
    'Authorization': `JWT ${token}`
  }
})
```

**令牌管理建议**：
- 使用系统级安全存储
- 实现令牌刷新机制
- 检测设备安全状态（如越狱/root）

### 4.3 服务端客户端 / 第三方集成

**推荐方式**：API Key

**特点**：
1. **长期有效**：API Key 通常不会自动过期
2. **服务端存储**：仅在服务端环境使用，不暴露给客户端
3. **细粒度控制**：每个用户可有独立的 API Key
4. **简单集成**：适合脚本、CI/CD、微服务通信

**请求格式**：
```
Authorization: {collection-slug} API-Key <api-key>
```

**API Key 安全**（见 `packages/payload/src/auth/strategies/apiKey.ts`）：
- 使用 HMAC-SHA256（兼容 SHA-1 用于旧版本）
- 数据库中存储的是哈希后的索引值，而非原始密钥
- 每次请求实时计算哈希进行比对

### 4.4 单页应用（SPA）

**推荐方式**：Authorization Header + 短期令牌

**考虑因素**：
1. **XSS 风险**：JavaScript 可访问的存储（localStorage、sessionStorage）存在 XSS 风险
2. **刷新机制**：需要实现令牌刷新逻辑
3. **静默续期**：可使用 refresh token 或自动刷新机制

**对比选择**：

| 存储位置 | 安全性 | 自动发送 | XSS 防护 | CSRF 防护 |
|---------|-------|---------|---------|----------|
| HttpOnly Cookie | 高 | 是 | 是 | 需要配置 |
| localStorage | 中 | 否 | 否 | 不需要 |
| sessionStorage | 中 | 否 | 否 | 不需要 |
| Memory (变量) | 高 | 否 | 部分 | 不需要 |

---

## 5. 中间件层统一验证机制

### 5.1 核心执行流程

Payload 的统一验证入口位于 `packages/payload/src/auth/executeAuthStrategies.ts`：

```typescript
export const executeAuthStrategies = async (
  args: AuthStrategyFunctionArgs
): Promise<AuthStrategyResult> => {
  let result: AuthStrategyResult = { user: null }

  if (!args.payload.authStrategies?.length) {
    return result
  }

  for (const strategy of args.payload.authStrategies) {
    args.strategyName = strategy.name
    args.isGraphQL = Boolean(args.isGraphQL)
    args.canSetHeaders = Boolean(args.canSetHeaders)

    try {
      const authResult = await strategy.authenticate(args)
      // 合并响应头
      if (authResult.responseHeaders) {
        authResult.responseHeaders = mergeHeaders(
          result.responseHeaders || new Headers(),
          authResult.responseHeaders || new Headers()
        )
      }
      result = authResult
    } catch (err) {
      logError({ err, payload: args.payload })
    }

    // 第一个成功的策略立即返回
    if (result.user) {
      return result
    }
  }
  return result
}
```

### 5.2 策略函数接口

所有鉴权策略必须实现统一的接口（见 `packages/payload/src/auth/types.ts:169-199`）：

```typescript
type AuthStrategyFunctionArgs = {
  canSetHeaders?: boolean      // 是否可以设置响应头
  headers: Request['headers']  // 请求头
  isGraphQL?: boolean          // 是否为 GraphQL 请求
  payload: Payload             // Payload 实例
  strategyName?: string        // 策略名称（由执行器注入）
}

type AuthStrategyResult = {
  responseHeaders?: Headers    // 自定义响应头
  user: TypedUser | null       // 验证成功的用户对象
}

type AuthStrategyFunction = (
  args: AuthStrategyFunctionArgs
) => AuthStrategyResult | Promise<AuthStrategyResult>

type AuthStrategy = {
  authenticate: AuthStrategyFunction
  name: string
}
```

### 5.3 统一鉴权操作

顶层 `auth` 操作（见 `packages/payload/src/auth/operations/auth.ts`）：

```typescript
export const auth = async (args: Required<AuthArgs>): Promise<AuthResult> => {
  const { canSetHeaders, headers } = args
  const req = args.req as PayloadRequest
  const { payload } = req

  try {
    // 1. 执行所有鉴权策略
    const { responseHeaders, user } = await executeAuthStrategies({
      canSetHeaders,
      headers,
      payload,
    })

    // 2. 将用户绑定到请求对象
    req.user = user
    req.responseHeaders = responseHeaders

    // 3. 计算权限
    const permissions = await getAccessResults({ req })

    return {
      permissions,
      responseHeaders,
      user,
    }
  } catch (error) {
    await killTransaction(req)
    throw error
  }
}
```

### 5.4 中间件集成点

鉴权在请求处理流程中的调用时机：

```
HTTP 请求到达
    ↓
请求解析（Body、Query、Headers）
    ↓
[鉴权中间件] → executeAuthStrategies()
    ├── 尝试自定义策略
    ├── 尝试 API Key 策略
    └── 尝试 JWT 策略（Cookie / Header）
    ↓
req.user 被设置
    ↓
权限计算（getAccessResults）
    ↓
路由处理 / 操作执行
    ↓
响应返回
```

---

## 6. 自定义策略实现

### 6.1 自定义策略示例

以下是一个基于自定义 Header 的认证策略（参考 `test/auth/custom-strategy/config.ts`）：

```typescript
import type { AuthStrategyFunction } from 'payload'

const customAuthenticationStrategy: AuthStrategyFunction = async ({
  headers,
  payload,
}) => {
  // 从请求头获取自定义认证凭据
  const code = headers.get('code')
  const secret = headers.get('secret')

  if (!code || !secret) {
    return { user: null }
  }

  // 自定义验证逻辑
  const usersQuery = await payload.find({
    collection: 'users',
    where: {
      code: { equals: code },
      secret: { equals: secret },
    },
  })

  const user = usersQuery.docs[0]
  if (!user) return { user: null }

  return {
    // 可以设置自定义响应头
    responseHeaders: new Headers({
      'X-Custom-Auth': 'success',
    }),
    user: {
      ...user,
      _strategy: 'custom-strategy',
      collection: 'users',
    },
  }
}

// 配置使用
export default buildConfig({
  collections: [
    {
      slug: 'users',
      auth: {
        disableLocalStrategy: true,  // 可选：禁用内置策略
        strategies: [
          {
            name: 'custom-strategy',
            authenticate: customAuthenticationStrategy,
          },
        ],
      },
      // ...
    },
  ],
})
```

### 6.2 自定义策略使用场景

| 场景 | 说明 |
|-----|------|
| **OAuth 2.0 / OIDC** | 集成 Google、GitHub、Azure AD 等第三方登录 |
| **企业 SSO** | SAML、LDAP、Kerberos 等企业认证协议 |
| **双因素认证 (2FA)** | TOTP、SMS、硬件密钥等第二因素验证 |
| **无密码登录** | 魔法链接、WebAuthn / Passkeys |
| **IP 白名单** | 基于请求来源 IP 的访问控制 |
| **证书认证** | mTLS（双向 TLS）客户端证书认证 |

---

## 7. 安全最佳实践

### 7.1 JWT 安全

1. **使用强密钥（原始 secret 必须强壮）**
   - SHA-256 转换是**确定性映射**，不会增强弱密钥的安全性
   - `config.secret` 应使用密码学安全的随机数生成器（如 `crypto.randomBytes(32)`）
   - 建议原始长度至少 32 字节（虽然最终被压缩到 128 位熵）
   - **注意**：最终 `payload.secret` 是 32 字符十六进制字符串（128 位有效熵）

2. **合理设置过期时间**
   - 访问令牌：短期（15 分钟 - 2 小时）
   - 刷新令牌：可设置更长但需可撤销
   - 128 位密钥配合短期令牌在大多数场景下安全

3. **敏感信息不入 JWT**
   - JWT 是 Base64 编码，非加密
   - 不要在 JWT 中存储密码、信用卡号等敏感信息
   - 使用 `saveToJWT` 控制哪些字段进入令牌

4. **外部服务验证必须复刻预处理**
   - 外部服务验证 Payload JWT 时，必须先对原始 secret 做相同预处理
   - 流程：`SHA-256 → 十六进制字符串 → 取前 32 字符（保留为字符串，不要解码）`
   - 见 3.1.1 节"偏差一"中的代码示例和正确做法

### 7.2 会话安全

1. **启用会话**：
   - 提供即时撤销能力
   - 支持多设备管理
   - 可检测异常登录

2. **定期清理过期会话**：
   - Payload 会在添加新会话时清理过期会话
   - 建议实现定期的后台清理任务

### 7.3 Cookie 安全

```typescript
auth: {
  cookies: {
    secure: true,        // 仅通过 HTTPS 传输
    sameSite: 'Strict',  // 防止 CSRF
    domain: 'example.com' // 限制 Cookie 作用域
  }
}
```

### 7.4 API Key 安全

1. **仅服务端使用**：绝不在前端代码中暴露 API Key
2. **定期轮换**：建立 API Key 轮换机制
3. **最小权限**：为 API Key 用户分配最小必要权限
4. **审计日志**：记录 API Key 的使用情况

---

## 8. 配置参考

### 8.1 完整鉴权配置示例

```typescript
import type { CollectionConfig } from 'payload'

export const Users: CollectionConfig = {
  slug: 'users',
  auth: {
    // 令牌配置
    tokenExpiration: 7200,           // 2 小时（秒）
    removeTokenFromResponses: false, // 是否从响应中移除令牌
    
    // 会话配置
    useSessions: true,
    
    // Cookie 配置
    cookies: {
      domain: 'example.com',
      sameSite: 'Strict',
      secure: true,
    },
    
    // 登录配置
    maxLoginAttempts: 5,             // 最大登录尝试次数
    lockTime: 600000,                // 锁定时间（毫秒）
    loginWithUsername: false,        // 是否允许用户名登录
    
    // 邮箱验证
    verify: {
      generateEmailSubject: ({ token, user }) => 
        `Verify your email, ${user.email}`,
      generateEmailHTML: ({ token, user }) => 
        `<a href="/verify?token=${token}">Verify</a>`,
    },
    
    // 忘记密码
    forgotPassword: {
      expiration: 3600000,           // 1 小时
    },
    
    // API Key
    useAPIKey: true,
    
    // 自定义策略
    strategies: [
      {
        name: 'oauth-github',
        authenticate: githubOAuthStrategy,
      },
    ],
    
    // JWT 数据深度
    depth: 0,
    
    // 禁用本地策略（完全自定义）
    // disableLocalStrategy: true,
  },
  fields: [
    {
      name: 'roles',
      type: 'select',
      hasMany: true,
      options: ['admin', 'editor', 'user'],
      saveToJWT: true,  // 将此字段包含在 JWT 中
    },
  ],
}
```

### 8.2 全局 JWT 提取顺序配置

```typescript
import type { Config } from 'payload'

export default buildConfig({
  auth: {
    // 自定义 JWT 提取顺序
    jwtOrder: ['cookie', 'Bearer', 'JWT'],
  },
  // CSRF 白名单（配合 Cookie 使用）
  csrf: [
    'https://app.example.com',
    'https://admin.example.com',
  ],
  // ...
})
```

---

## 9. 架构总结

### 9.1 核心设计原则

1. **策略模式**：将不同认证方式封装为独立策略，易于扩展
2. **责任链模式**：按顺序尝试多个策略，第一个成功即返回
3. **统一接口**：所有策略实现相同的 `AuthStrategyFunction` 接口
4. **多层提取**：JWT 可从多个来源提取，适应不同客户端
5. **可选会话**：服务端会话提供额外的撤销能力
6. **安全优先**：内置 CSRF 防护、HTTPOnly Cookie、密钥哈希等

### 9.2 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                      客户端请求                              │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ 浏览器     │  │ 移动应用     │  │ 第三方服务         │  │
│  │ (Cookie)  │  │ (Bearer)    │  │ (API Key)          │  │
│  └─────┬──────┘  └──────┬───────┘  └──────────┬─────────┘  │
└────────┼────────────────┼─────────────────────┼────────────┘
         │                │                     │
         ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   服务端鉴权层                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              extractJWT (JWT 提取)                   │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│  │  │ JWT      │  │ Bearer   │  │ Cookie (CSRF 检查)│   │  │
│  │  │ Header   │  │ Header   │  │                  │   │  │
│  │  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │  │
│  │       │             │                 │              │  │
│  │       └─────────────┼─────────────────┘              │  │
│  │                     ▼                                │  │
│  │              提取到的 JWT 令牌                        │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                   │
│  ┌──────────────────────▼───────────────────────────────┐  │
│  │         executeAuthStrategies (策略执行)             │  │
│  │  ┌──────────────┐  ┌──────────┐  ┌──────────────┐   │  │
│  │  │ 自定义策略    │  │ API Key  │  │ JWT + 会话   │   │  │
│  │  │ (优先)       │  │ 策略     │  │ 验证         │   │  │
│  │  └──────┬───────┘  └────┬─────┘  └──────┬───────┘   │  │
│  │         │               │                │           │  │
│  │         ▼               ▼                ▼           │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │         第一个成功的策略返回结果              │   │  │
│  │  └──────────────────────┬───────────────────────┘   │  │
│  └─────────────────────────┼───────────────────────────┘  │
│                            │                              │
│  ┌─────────────────────────▼───────────────────────────┐  │
│  │              getAccessResults (权限计算)            │  │
│  └─────────────────────────┬───────────────────────────┘  │
└────────────────────────────┼──────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  req.user 已设置 │
                    │  后续处理可用    │
                    └─────────────────┘
```

---

## 10. 关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `packages/payload/src/auth/index.ts` | 鉴权模块导出 |
| `packages/payload/src/auth/jwt.ts` | JWT 签名实现 |
| `packages/payload/src/auth/sessions.ts` | 会话管理 |
| `packages/payload/src/auth/extractJWT.ts` | JWT 提取逻辑 |
| `packages/payload/src/auth/cookies.ts` | Cookie 生成与解析 |
| `packages/payload/src/auth/types.ts` | 类型定义 |
| `packages/payload/src/auth/strategies/jwt.ts` | JWT 策略实现 |
| `packages/payload/src/auth/strategies/apiKey.ts` | API Key 策略实现 |
| `packages/payload/src/auth/executeAuthStrategies.ts` | 策略执行器 |
| `packages/payload/src/auth/operations/auth.ts` | 统一鉴权操作 |
| `packages/payload/src/auth/operations/login.ts` | 登录操作 |
| `packages/payload/src/auth/getFieldsToSign.ts` | JWT 字段收集 |
| `packages/payload/src/index.ts:958-989` | 策略注册逻辑 |

---

## 11. 结论

Payload 的鉴权机制设计体现了现代 Web 应用安全最佳实践：

1. **灵活性**：通过策略模式支持多种认证方式，从简单的用户名密码到复杂的企业 SSO
2. **安全性**：多层安全防护（CSRF、HTTPOnly、密钥哈希、会话撤销）
3. **适应性**：同时支持浏览器 Cookie 和移动端 Header 两种主流认证模式
4. **可扩展性**：自定义策略接口允许集成任何认证系统
5. **统一性**：中间件层的统一验证机制确保所有请求都经过一致的安全检查

对于开发团队，建议根据具体客户端类型选择合适的认证方式：
- **Web 应用**：优先使用 Cookie + 会话
- **移动应用**：使用 Authorization Header
- **服务集成**：使用 API Key
- **特殊需求**：实现自定义策略
