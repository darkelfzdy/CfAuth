# CfAuth — Cloudflare Workers 认证服务 PRD

## 1. 项目概述

**CfAuth** 是一个部署在 Cloudflare Workers 上的独立认证服务（基于 OpenAuth.js），为其他 Cloudflare Workers 提供统一的认证能力。支持两种集成方式：**标准 OAuth 2.0 HTTP 端点**（适用于外部服务）和 **Cloudflare Service Binding**（适用于同一账号下的 Workers，零网络开销）。

### 1.1 核心目标

- **独立部署**：作为一个独立的 Worker 运行，支持标准 OAuth 2.0 HTTP 端点与 Cloudflare Service Binding 两种集成方式
- **多提供商**：支持邮箱+密码、Google OAuth、GitHub OAuth 三种登录方式
- **D1 存储**：不使用 KV，全部持久化存储使用 Cloudflare D1（关系型数据库）
- **可测试**：即使没有其他消费者 Worker，也能独立完成测试验证
- **分步可测**：项目按模块拆分，每个模块可独立开发、测试、验收

### 1.2 参考基线

以 `openauth-template/` 为技术起点，在其基础上改造，实际代码落在 `app/` 目录。

---

## 2. 系统架构

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────┐
│                    CfAuth Worker                   │
│                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │  OAuth 2.0   │  │  Provider 层  │  │  UI 层   │ │
│  │  Token 端点   │  │ password     │  │ 登录/注册 │ │
│  │  /authorize  │  │ google       │  │ 选择页面  │ │
│  │  /token      │  │ github       │  │ 主题定制  │ │
│  └──────┬───────┘  └──────┬───────┘  └────┬─────┘ │
│         │                 │               │       │
│  ┌──────┴─────────────────┴───────────────┴──────┐ │
│  │              D1 Storage Adapter               │ │
│  │  (自实现，替换 CloudflareStorage/KV)           │ │
│  └──────────────────────┬────────────────────────┘ │
│                         │                           │
└─────────────────────────┼───────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │    Cloudflare D1       │
              │  ┌─────────────────┐   │
              │  │ user            │   │
              │  │ oauth_account   │   │
              │  │ verification_code│  │
              │  │ openauth_store  │   │
              │  │   (含 password  │   │
              │  │    hash、       │   │
              │  │    refresh      │   │
              │  │    token 等)    │   │
              │  └─────────────────┘   │
              └─────────────────────────┘
```

### 2.2 消费者集成方式

提供两种集成方式，根据消费者位置选择：

#### 方式 A：标准 OAuth 2.0 HTTP 端点（适用于外部服务 / 非 Workers 服务）

```
消费者 Worker                  CfAuth Worker              第三方 OAuth
     │                             │                          │
     │  GET /authorize?client_id   │                          │
     │  &redirect_uri&scope        │                          │
     │────────────────────────────>│                          │
     │                             │  (如选 Google/GitHub)     │
     │                             │─────────────────────────>│
     │                             │<─────────────────────────│
     │  redirect_uri?code=xxx      │                          │
     │<────────────────────────────│                          │
     │                             │                          │
     │  POST /token (code)         │                          │
     │────────────────────────────>│                          │
     │  {access_token, refresh}    │                          │
     │<────────────────────────────│                          │
     │                             │                          │
     │  GET /api/me (Bearer token) │                          │
     │────────────────────────────>│ (token 内置 subject 信息) │
```

- 消费者使用 `@openauthjs/openauth/client` 的 `createClient` + `verify` 验证 JWT
- 无需回调 CfAuth，因为 access_token 是自包含的 JWT，内含用户 subject 信息

#### 方式 B：Cloudflare Service Binding（推荐，适用于同一账号下的 Workers）

消费者 Worker 的 `wrangler.jsonc` 中绑定 CfAuth：

```json
{
  "services": [
    { "binding": "AUTH", "service": "cfauth" }
  ]
}
```

在代码中直接调用：

```typescript
import { createClient } from '@openauthjs/openauth/client'

const client = createClient({
  // Service Binding 方式：直接传入 binding
  fetch: (input, init) => env.AUTH.fetch(input, init),
})
```

**优点**：零网络延迟、无公网依赖、部署配置简单。

### 2.3 错误处理策略

#### 2.3.1 认证流程错误

| 错误场景 | HTTP 状态码 | 处理方式 |
|----------|------------|----------|
| authorization code 无效/过期 | `400 Bad Request` | 返回 `invalid_grant` 错误，客户端需重新发起授权 |
| access_token 过期 | `401 Unauthorized` | 客户端使用 refresh_token 刷新 |
| refresh_token 无效/过期 | `401 Unauthorized` | 返回 `invalid_grant`，客户端需重新登录 |
| 邮箱+密码登录，密码错误 | `401 Unauthorized` | 返回通用错误信息（不透露是邮箱不存在还是密码错误） |
| 验证码错误/过期 | `400 Bad Request` | 提示验证码无效，前端可请求重新发送 |
| D1 查询超时 | `502 Bad Gateway` | 捕获 `D1_ERROR`，返回结构化错误响应 |
| Provider OAuth 回调参数缺失 | `400 Bad Request` | 返回 `invalid_request` 错误 |

#### 2.3.2 异常处理原则

- 所有公开端点返回结构化 JSON 错误响应（`{ "error": "...", "error_description": "..." }`），遵循 OAuth 2.0 错误规范
- 服务端错误（500）使用 `try/catch` 捕获，通过 `console.error(JSON.stringify({...}))` 记录结构化日志
- 不暴露内部实现细节（如 D1 表名、堆栈信息）
- 密码验证错误不区分"邮箱不存在"和"密码错误"，防止枚举攻击
- 所有 Promise 必须 `await` / `return` / `ctx.waitUntil()`，不允许 floating promise

---

## 3. 技术选型

### 3.1 核心技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 运行时 | Cloudflare Workers | 免费额度充足，全球边缘网络 |
| 框架 | `@openauthjs/openauth` 0.4.3 | 基于 Hono，内置 OAuth 2.0 实现，锁定小版本 |
| 存储 | Cloudflare D1 | 替换原模板的 KV，关系型数据库更灵活 |
| 验证库 | valibot 1.2.0 | 标准 schema 验证，轻量 |
| 类型 | TypeScript strict | 严格模式 |
| 部署 | Wrangler CLI | `wrangler dev` / `wrangler deploy` |

### 3.2 关键决策

#### 3.2.1 为什么用 D1 替代 KV？

- 原模板使用 KV（`CloudflareStorage`）存储 refresh token、password hash 等
- D1 提供 SQL 查询能力，更利于后续扩展（如用户管理、账户关联、审计日志）
- 需**自实现 D1Storage Adapter**（实现 OpenAuth 的 Storage 接口），因为官方暂无 D1 适配器

#### 3.2.2 为何选择邮箱+密码为主，验证码仅辅助？

采用 **邮箱+密码登录** 为主，邮箱验证码仅用于注册时的邮箱验证和找回密码。理由：

- **邮件发送量受限**：使用的邮件免费套餐发送量有限，验证码仅用于低频场景（注册、找回密码）
- **用户体验**：日常登录无需等待邮件，输入密码即可
- **安全性**：密码存储在 D1（经 OpenAuth 自动 hash），验证码用于关键操作的双重验证
- OpenAuth PasswordProvider 支持配置 password 字段启用，同时保留 `sendCode` 回调用于验证码场景

#### 3.2.3 为什么本地开发绑定远程组件？

- Cloudflare Workers 本地模拟环境（`--local`）与生产环境存在差异
- 直接绑定远程 D1 可确保行为一致，避免"本地通过、部署失败"的问题
- 需要预先在 Cloudflare Dashboard 创建好 D1 数据库

### 3.3 Cloudflare Workers 最佳实践约束

本项目所有代码必须遵循 Cloudflare Workers 最佳实践。以下是核心约束：

| 规则 | 要求 |
|------|------|
| 配置格式 | 使用 `wrangler.jsonc`（支持注释），不使用 `wrangler.toml` |
| 兼容性日期 | `compatibility_date` 使用当前日期（`2026-05-17`），定期更新 |
| nodejs_compat | 必须开启（valibot 依赖） |
| 生成绑定类型 | 运行 `npm run cf-typegen`（即 `wrangler types`），不手写 `Env` 接口 |
| 密钥管理 | 通过 `wrangler secret put` 设置，本地用 `.dev.vars`，禁止硬编码 |
| 可观测性 | 开启 `observability` 配置，使用结构化 JSON 日志 |
| 无全局请求状态 | 不在模块级变量中存储请求级数据 |
| 无 floating promise | 每个 Promise 必须 `await` / `return` / `ctx.waitUntil()`，建议启用 `@typescript-eslint/no-floating-promises` 规则 |
| Web Crypto | 使用 `crypto.randomUUID()` 和 `crypto.getRandomValues()`，禁用 `Math.random()` |
| 显式错误处理 | 使用 `try/catch` + 结构化错误响应，禁用 `passThroughOnException()` |
| Service Binding | Worker-to-Worker 使用 Service Binding（`env.SERVICE.fetch()`），不走公网 HTTP |
| 测试 | 使用 `@cloudflare/vitest-pool-workers` 在 Workers 运行时内测试 |

---

## 4. 数据库设计

### 4.1 D1 表结构

```sql
-- 用户表（模板原有，扩展字段）
CREATE TABLE IF NOT EXISTS user (
  id TEXT PRIMARY KEY NOT NULL DEFAULT (lower(hex(randomblob(16)))),
  client_id TEXT NOT NULL,        -- 所属消费者标识（service-a, service-b ...）
  email TEXT NOT NULL,
  name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(client_id, email)        -- 每个消费者下 email 唯一
);

-- OAuth 账户关联表（一个用户可绑定多个第三方登录）
CREATE TABLE IF NOT EXISTS oauth_account (
  client_id TEXT NOT NULL,
  provider TEXT NOT NULL,       -- 'google' | 'github'
  provider_user_id TEXT NOT NULL,
  user_id TEXT NOT NULL REFERENCES user(id),
  email TEXT,
  name TEXT,
  avatar_url TEXT,
  access_token TEXT,
  refresh_token TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (client_id, provider, provider_user_id)
  -- 同一消费者下，同一第三方账户只绑定一次
);

-- 验证码表（邮箱验证码临时存储，替代 KV 的 set/get 语义）
CREATE TABLE IF NOT EXISTS verification_code (
  id TEXT PRIMARY KEY NOT NULL DEFAULT (lower(hex(randomblob(16)))),
  email TEXT NOT NULL,
  code TEXT NOT NULL,
  purpose TEXT NOT NULL DEFAULT 'login',  -- 'login' | 'register' | 'change_password'
  expires_at TIMESTAMP NOT NULL,
  used INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- OpenAuth 存储表（替代 KV 的 key-value 存储）
-- 用于存储 refresh token、password hash 等 OpenAuth 内部数据
CREATE TABLE IF NOT EXISTS openauth_store (
  key TEXT PRIMARY KEY NOT NULL,
  value TEXT NOT NULL,
  expires_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 4.2 表说明

| 表 | 作用 |
|----|------|
| `user` | 本地用户主表。`client_id` 标识所属消费者（如 service-a、service-b），`UNIQUE(client_id, email)` 确保每个消费者下邮箱唯一，不同消费者间可存在相同邮箱的独立账号 |
| `oauth_account` | 关联第三方 OAuth 账户。`client_id` 作为联合主键的一部分，同一第三方账户可在不同消费者下绑定不同本地用户 |
| `verification_code` | 存储邮箱验证码，带过期时间和使用标记 |
| `openauth_store` | **核心**：实现 OpenAuth Storage 接口的 D1 实现，替代 KV 的 `get(key)/set(key,value)/delete(key)`。内部存储 password hash、refresh token、authorization code 等 OpenAuth 运行时数据 |

---

## 5. 认证方式详细设计

### 5.1 邮箱+密码（PasswordProvider + PasswordUI）

使用 OpenAuth PasswordProvider，配置启用 password 字段。验证码仅用于注册验证和找回密码：

- **注册**：输入邮箱 → 设置密码 → 发送验证码至邮箱 → 输入验证码验证 → 创建账户
- **登录**：输入邮箱 + 密码 → 验证通过 → 返回 token
- **找回密码**：输入邮箱 → 发送验证码 → 输入验证码 + 新密码 → 更新密码

验证码发送：开发阶段使用 `console.log` 输出至 Worker 日志（通过 `wrangler tail` 查看），生产环境对接 Resend / SendGrid 等邮件服务。

### 5.2 Google OAuth

```
GoogleProvider({
  clientId: env.GOOGLE_CLIENT_ID,
  clientSecret: env.GOOGLE_CLIENT_SECRET,
  scopes: ['openid', 'email', 'profile'],
})
```

在 Google Cloud Console 创建 OAuth 2.0 凭据，密钥通过 `wrangler secret put` 设置。

**此服务完全免费**，无需购买任何付费套餐。只需在 Google Cloud Console 中创建一个 OAuth 2.0 客户端 ID（Web 应用类型），填写授权的重定向 URI 即可获取凭据。

### 5.3 GitHub OAuth

```
GithubProvider({
  clientId: env.GITHUB_CLIENT_ID,
  clientSecret: env.GITHUB_CLIENT_SECRET,
  scopes: ['user:email'],
})
```

在 GitHub Developer Settings 创建 OAuth App，密钥通过 `wrangler secret put` 设置。

**此服务完全免费**，无需购买任何付费套餐。在 GitHub Developer Settings → OAuth Apps 中创建一个新应用，填写 Homepage URL 和 Authorization callback URL 即可获取 `client_id` 和 `client_secret`。

### 5.4 用户合并策略

每个消费者 Worker 拥有独立的用户命名空间（通过 `client_id` 隔离），因此合并策略作用域限定在**同一 client 内部**：

1. 在 `success` 回调中获取当前请求的 `client_id`（通过 `auth_flow` cookie 传递）
2. 按 `(client_id, email)` 查询 `user` 表是否存在该用户
3. 若存在，在 `oauth_account` 中绑定新的提供商记录（`PRIMARY KEY (client_id, provider, provider_user_id)` 确保不重复）
4. 若不存在，创建新 `user`（含 `client_id`）+ `oauth_account`

**跨消费者场景**：同一邮箱可在 Service A 和 Service B 各有一个独立账号。`oauth_account` 的联合主键 `(client_id, provider, provider_user_id)` 允许同一 Google 账号在不同消费者下绑定不同的本地用户。

---

## 6. D1Storage Adapter 设计

### 6.1 接口定义

> **注意**：以下接口基于 OpenAuth 0.4.3 版本。实现前请查阅 [OpenAuth 最新文档](https://openauth.js.org/docs/) 确认 Storage 接口签名是否有变化。

OpenAuth Storage 接口需要实现：

```typescript
interface StorageAdapter {
  get(key: string[]): Promise<Record<string, unknown> | undefined>;
  set(key: string[], value: Record<string, unknown>, expiry?: Date): Promise<void>;
  remove(key: string[]): Promise<void>;
  scan(prefix: string[]): Promise<{ key: string[]; value: Record<string, unknown> }[]>;
}
```

### 6.2 D1 实现方案

使用 `openauth_store` 表：

- `get(key)` → `SELECT value FROM openauth_store WHERE key = ? AND (expires_at IS NULL OR expires_at > CURRENT_TIMESTAMP)`
- `set(key, value, expiry?)` → `INSERT OR REPLACE INTO openauth_store (key, value, expires_at) VALUES (?, ?, ?)`
- `remove(key)` → `DELETE FROM openauth_store WHERE key = ?`
- `scan(prefix)` → `SELECT key, value FROM openauth_store WHERE key LIKE ? AND (expires_at IS NULL OR expires_at > CURRENT_TIMESTAMP)`
- key 使用分隔符（如 `:`）拼接数组

---

## 7. 本地开发策略

### 7.1 环境准备

```
# 1. 确保 Wrangler 已登录
wrangler login

# 2. 在 Cloudflare Dashboard 创建 D1 数据库（如 cfauth-db）
wrangler d1 create cfauth-db

# 3. 将 database_id 填入 app/wrangler.jsonc 的 d1_databases 配置

# 4. 应用数据库迁移（远程）
wrangler d1 migrations apply AUTH_DB --remote
```

### 7.2 开发流程

```bash
# 启动本地开发服务器（自动绑定远程 D1）
wrangler dev --remote

# 查看实时日志（用于查看验证码等 console.log 输出）
wrangler tail
```

### 7.3 密钥管理

- OAuth 密钥（Google/GitHub）通过 `wrangler secret put` 设置到远程
- 本地开发创建 `.dev.vars` 文件（不提交到 Git）设置临时密钥：

```
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxx
GITHUB_CLIENT_ID=xxx
GITHUB_CLIENT_SECRET=xxx
```

### 7.4 wrangler.jsonc 配置要点

```json
{
  "compatibility_date": "2026-05-17",
  "main": "src/index.ts",
  "name": "cfauth",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [
    {
      "binding": "AUTH_DB",
      "database_name": "cfauth-db",
      "database_id": "<从 wrangler d1 create 获取>"
    }
  ],
  "observability": {
    "enabled": true,
    "logs": { "head_sampling_rate": 1 },
    "traces": { "enabled": true, "head_sampling_rate": 0.01 }
  }
}
```

**注意：不包含 `kv_namespaces`**——本项目不使用 KV。

---

## 8. 测试策略

### 8.1 测试层次

| 层次 | 工具 | 范围 |
|------|------|------|
| 单元测试 | vitest | D1Storage adapter、工具函数 |
| 集成测试 | wrangler dev + curl/浏览器 | 完整 OAuth 流程、各 Provider |
| 手动验收 | 浏览器访问登录页面 | UI 展示、交互体验 |

### 8.2 测试方法（无消费者 Worker 场景）

由于当前没有其他 Workers 项目需要对接，使用以下方式独立测试：

#### 方法一：浏览器直接测试（推荐主要方式）

1. 启动 `wrangler dev --remote`
2. 浏览器访问 `http://localhost:8787/.well-known/oauth-authorization-server` 验证服务运行
3. 访问登录页面测试各 Provider 流程
4. 通过 URL 参数模拟 OAuth 客户端请求：

```
# 测试授权码流程
http://localhost:8787/authorize?client_id=test&redirect_uri=http://localhost:3000/callback&response_type=code

# 返回的 code 通过 curl 换 token
curl -X POST http://localhost:8787/token \
  -d "grant_type=authorization_code&code=<code>&client_id=test&redirect_uri=http://localhost:3000/callback"
```

#### 方法二：curl 脚本集成测试

编写 shell 脚本自动执行完整 OAuth 流程（需处理 Cookie 和重定向）。

#### 方法三：单元测试（D1Storage Adapter）

编写 vitest 针对 `D1Storage` adapter 进行测试，验证 CRUD 操作和时间戳过期逻辑。

### 8.3 vitest 配置

注意：vitest 需要在 `nodejs_compat` 兼容模式下运行（与运行时对齐），可使用 `@cloudflare/vitest-pool-workers` 或直接 mock D1 接口进行测试。

---

## 9. 分步开发计划

将项目拆分为 6 个阶段，每个阶段可独立开发、测试、验收：

### 阶段 0：项目初始化

**目标**：建立可运行的 Worker 骨架

- [ ] 在 `app/` 创建 `package.json`、`tsconfig.json`、`wrangler.jsonc`
- [ ] 安装依赖：`@openauthjs/openauth`、`valibot`
- [ ] 创建 Cloudflare D1 数据库（远程）
- [ ] 创建 `app/src/` 目录结构
- [ ] 编写最小化 `src/index.ts`（用 MemoryStorage 测试能否启动）
- [ ] 运行 `wrangler dev --remote` 验证基本服务可访问

**验收**：`npx tsc --noEmit` 编译通过，浏览器访问 `localhost:8787/.well-known/oauth-authorization-server` 返回 JSON

### 阶段 1：D1 Storage Adapter

**目标**：实现 D1 版本的 OpenAuth Storage，替换模板中的 KV

- [ ] 编写 D1 迁移脚本（user 含 client_id、openauth_store 表）
- [ ] 实现 `src/storage/d1.ts`：`D1Storage` 类，实现 StorageAdapter 接口
- [ ] 将 `openauth_store` 的 get/set/remove/scan 操作封装
- [ ] 过期的 key 自动清理或忽略
- [ ] 在 `src/index.ts` 中使用 `D1Storage` 替换 `CloudflareStorage`
- [ ] 验证 issuer 启动无报错

**验收**：`npx tsc --noEmit` 编译通过，使用 `D1Storage` 的 issuer 能正常启动，`/.well-known/oauth-authorization-server` 正常响应

### 阶段 2：邮箱+密码认证（PasswordProvider）

**目标**：实现邮箱+密码登录，邮箱验证码用于注册验证和找回密码

- [ ] 配置 `PasswordProvider` + `PasswordUI`（启用 password 字段）
- [ ] 实现 `sendCode` 回调（开发阶段 console.log，可通过 `wrangler tail` 查看）
- [ ] 实现密码 hash 验证逻辑（OpenAuth 内置）
- [ ] 实现 `success` 回调中的 `getOrCreateUser` 逻辑（按 client_id + email 查找/创建）
- [ ] 创建 `oauth_account`（含 client_id 联合主键）、`verification_code` 表的迁移脚本
- [ ] 自定义主题（`theme` 配置）
- [ ] 测试：浏览器打开登录页 → 输入邮箱+密码 → 登录成功获取 token
- [ ] 测试：注册流程（输入邮箱+密码 → 输入验证码 → 创建账户）

**验收**：`npx tsc --noEmit` 编译通过，邮箱+密码登录流程走通，注册验证码流程走通，能获取到 access_token

### 阶段 3：Google OAuth 认证

**目标**：添加 Google 登录

- [ ] 安装 `@openauthjs/openauth/provider/google`（或内置）
- [ ] 配置 Google Cloud OAuth 凭据
- [ ] 在 `providers` 中添加 `google: GoogleProvider(...)`
- [ ] 扩展 `success` 回调处理 `value.provider === 'google'` 场景
- [ ] 实现用户合并逻辑（同一邮箱多 Provider）
- [ ] 测试：浏览器点击 Google 登录按钮 → 完成 Google 授权 → 获取 token

**验收**：`npx tsc --noEmit` 编译通过，Google OAuth 流程走通，用户信息正确存入 D1

### 阶段 4：GitHub OAuth 认证

**目标**：添加 GitHub 登录

- [ ] 配置 GitHub OAuth App 凭据
- [ ] 在 `providers` 中添加 `github: GithubProvider(...)`
- [ ] 扩展 `success` 回调处理 `value.provider === 'github'` 场景
- [ ] 用户合并逻辑覆盖 GitHub 场景
- [ ] 测试：浏览器点击 GitHub 登录按钮 → 完成授权 → 获取 token

**验收**：`npx tsc --noEmit` 编译通过，GitHub OAuth 流程走通，用户信息正确存入 D1

### 阶段 5：集成验证与文档

**目标**：端到端验证 + 产出使用文档

- [ ] 编写完整 OAuth 流程的 curl 测试脚本
- [ ] 验证 token 校验（用 `@openauthjs/openauth/client` 的 verify 功能）
- [ ] 验证 refresh token 刷新流程
- [ ] 编写 `doc/INTEGRATION.md`：其他 Workers 如何对接到 CfAuth
- [ ] 编写 `doc/DEVELOPMENT.md`：本地开发环境搭建指南
- [ ] 代码清理、去除 console.log（生产路径）

**验收**：`npx tsc --noEmit` 编译通过，无 floating promise 警告，所有测试脚本通过，文档完整

### 阶段 6（可选）：单元测试

**目标**：补充自动化测试

- [ ] 配置 vitest + `@cloudflare/vitest-pool-workers`
- [ ] D1Storage adapter 单元测试
- [ ] 工具函数单元测试
- [ ] CI 集成（如 GitHub Actions）

---

## 10. 项目目录结构

```
app/
├── package.json
├── tsconfig.json
├── wrangler.jsonc
├── worker-configuration.d.ts   # 由 `npm run cf-typegen`（即 `wrangler types`）自动生成
├── .gitignore
├── migrations/
│   ├── 0001_create_user_table.sql
│   ├── 0002_create_oauth_account.sql
│   ├── 0003_create_verification_code.sql
│   └── 0004_create_openauth_store.sql
├── src/
│   ├── index.ts                # Worker 入口，issuer 配置
│   ├── subjects.ts             # Subject 定义（共享类型）
│   ├── storage/
│   │   └── d1.ts               # D1Storage adapter 实现
│   ├── providers/
│   │   └── password.ts         # 邮箱+密码认证相关配置
│   ├── db/
│   │   └── user.ts             # 用户 CRUD 操作
│   └── utils/
│       └── email.ts            # 邮件发送（未来对接真实服务）
└── test/
    ├── storage.test.ts         # D1Storage 单元测试
    └── oauth-flow.sh           # curl 集成测试脚本
```

---

## 11. 风险与注意事项

| 风险 | 应对 |
|------|------|
| D1 Storage Adapter 与 OpenAuth 内部依赖不兼容 | 阶段 1 就引入并验证，出现问题及时调整 |
| 邮件免费套餐发送量有限 | 验证码仅用于注册和找回密码，日常登录使用密码，大幅降低邮件用量 |
| OAuth 密钥泄露 | 使用 `wrangler secret` 管理，`.dev.vars` 加入 `.gitignore` |
| 远程 D1 开发产生费用 | D1 有免费额度（5GB 存储、每月 500 万行读取），开发阶段完全够用 |
| OpenAuth 版本更新导致 API 变更 | 锁定 `@openauthjs/openauth` 小版本号 |

---

## 12. 参考链接

- [OpenAuth 官方文档](https://openauth.js.org/docs/)
- [OpenAuth GitHub](https://github.com/toolbeam/openauth)
- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 文档](https://developers.cloudflare.com/d1/)
- [本模板参考 README](../openauth-template/README.md)
