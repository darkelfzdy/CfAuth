# CfAuth — Cloudflare OpenAuth 认证服务 PRD

## 1. 项目背景与目标

### 1.1 背景

当前项目内多个 Cloudflare Worker 各自实现认证逻辑，重复开发且维护成本高。需要一个统一的认证服务中心，所有 Worker 通过 OAuth 2.0 协议对接，实现认证逻辑的集中管理。

### 1.2 目标

- 在 Cloudflare Workers 上部署基于 `@openauthjs/openauth` 的独立认证服务（CfAuth）
- 提供邮箱验证码、Google、GitHub 三种登录方式
- 其他 Worker 通过 OAuth 2.0 标准流程对接 CfAuth，无需自行开发认证模块
- 使用 D1 作为全部存储后端，不依赖 KV

### 1.3 非目标

- 不提供用户管理后台（用户增删改查由业务 Worker 自行通过 API 管理）
- 不提供细粒度权限管理（RBAC 等由各业务 Worker 自行实现）
- 不提供社交账号信息同步（仅用 OAuth 获取邮箱）

## 2. 技术架构

### 2.1 整体架构

```
┌──────────────────┐     OAuth 2.0     ┌──────────────────────┐
│  业务 Worker A    │ ◄─────────────── │                      │
│  (资源服务)       │                   │    CfAuth Worker     │
├──────────────────┤                   │    (认证服务器)       │
│  业务 Worker B    │ ◄─────────────── │                      │
│  (资源服务)       │                   │  ┌────────────────┐  │
├──────────────────┤                   │  │  Providers:     │  │
│  业务 Worker C    │ ◄─────────────── │  │  ├─ Email Code  │  │
│  (资源服务)       │                   │  │  ├─ Google      │  │
└──────────────────┘                   │  │  └─ GitHub      │  │
                                       │  │                │  │
                                       │  │  Storage: D1   │  │
                                       │  └────────────────┘  │
                                       └──────────────────────┘
```

### 2.2 技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 运行时 | Cloudflare Workers | 边缘计算平台 |
| 认证框架 | `@openauthjs/openauth` v0.4.3 | OAuth 2.0 认证服务器 |
| 数据库 | Cloudflare D1 | 存储用户数据 + Token/Session |
| 数据校验 | `valibot` v1.2.0 | 类型安全的 schema 校验 |
| 语言 | TypeScript (strict) | 类型安全 |
| 兼容标志 | `nodejs_compat` | valibot 依赖 |

### 2.3 为什么不用 KV

openauth 官方默认使用 Cloudflare KV 存储 refresh token、code challenge 等临时数据。本项目要求全部使用 D1，原因：

- 减少基础设施种类，统一管理
- D1 支持事务和复杂查询，未来扩展更灵活
- 避免 KV 的最终一致性问题对认证流程的影响

通过实现自定义 `D1Storage` adapter 替代 `CloudflareStorage(KV)`。

## 3. 功能需求

### 3.1 认证方式

| 方式 | Provider | 说明 |
|------|----------|------|
| 邮箱验证码 | `PasswordProvider` | 输入邮箱 → 接收验证码 → 登录/注册。开发阶段 console.log 输出验证码，生产环境接入邮件服务 |
| Google 登录 | `GoogleProvider` | OAuth 2.0，获取用户邮箱 |
| GitHub 登录 | `GithubProvider` | OAuth 2.0，获取用户邮箱 |

### 3.2 认证流程

#### 3.2.1 邮箱验证码

```
注册:
  用户输入邮箱 → CfAuth 生成验证码 → console.log/邮件发送 → 用户输入验证码 → 设置密码 → 完成

登录:
  用户输入邮箱 → CfAuth 生成验证码 → console.log/邮件发送 → 用户输入验证码 → 完成
```

#### 3.2.2 Google/GitHub OAuth

```
用户点击 Google/GitHub 登录 → 跳转至 OAuth 授权页 → 用户授权 → 回调 → CfAuth 获取邮箱 → 完成
```

### 3.3 Token 管理

- 遵循 OAuth 2.0 标准
- 签发 access token + refresh token
- Access token 用于业务 Worker 验证用户身份
- Refresh token 用于无感续期
- Token 存储使用自定义 D1 Storage Adapter

### 3.4 主题化 UI

使用 openauth 内置 UI，自定义品牌信息：

| 配置项 | 值 |
|--------|-----|
| title | CfAuth |
| primary | 品牌主色 |
| favicon | 自定义 favicon |
| logo | 自定义 logo（light/dark） |

### 3.5 用户管理

- 用户数据仅存储 `id`、`email`、`created_at`
- 登录时自动创建用户（upsert 模式）
- openauth 不直接提供用户管理功能，由业务 Worker 调用 D1 API 自行管理

## 4. 数据模型

### 4.1 D1 `user` 表

沿用 openauth-template 的迁移脚本：

```sql
CREATE TABLE IF NOT EXISTS user (
  id         TEXT PRIMARY KEY NOT NULL DEFAULT (lower(hex(randomblob(16)))),
  email      TEXT UNIQUE NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 4.2 D1 `auth_storage` 表

自定义 D1 Storage Adapter 所需，替代 KV 存储：

```sql
CREATE TABLE IF NOT EXISTS auth_storage (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  expires_at TIMESTAMP
);
```

存储数据类型：
- Password hash（密码哈希）
- Refresh token
- Authorization code
- Code challenge
- OAuth state
- Session data

## 5. 分步开发计划

整个项目拆分为 7 个步骤，每步可独立测试，降低风险。

### Step 1: 项目初始化

**内容：**
- 在 `app/` 下初始化项目：`package.json`、`tsconfig.json`、`wrangler.json`
- 配置 D1 数据库绑定 `AUTH_DB`
- 创建迁移脚本目录 `migrations/`
- 复制 `worker-configuration.d.ts` 模板
- 配置 `nodejs_compat` 兼容标志

**验证方式：**
- `npm run cf-typegen` 成功生成类型
- `wrangler d1 migrations apply AUTH_DB --remote` 成功在远程 D1 创建表
- `npm run check` 通过

---

### Step 2: 自定义 D1 Storage Adapter

**内容：**
- 创建 `src/storage.ts`
- 实现 `D1Storage` 类，满足 openauth `StorageAdapter` 接口
  - `get(key: string): Promise<string | undefined>`
  - `set(key: string, value: string, expires?: Date): Promise<void>`
  - `remove(key: string): Promise<void>`
  - `scan?(prefix: string): Promise<{ key: string; value: string }[]>`
- 创建迁移 `0002_create_auth_storage_table.sql`
- 自动清理过期数据（可通过 `scan` + `remove` 或 SQL WHERE 实现）

**验证方式：**
- 编写单元测试，mock D1 binding，验证 get/set/remove 行为
- `wrangler dev --remote` 部署后手动测试

---

### Step 3: 邮箱验证码登录

**内容：**
- 创建 `src/index.ts` Worker 入口
- 配置 `PasswordProvider` + `PasswordUI`
- `sendCode` 回调开发阶段使用 `console.log` 输出验证码
- 配置 `success` 回调，实现用户自动创建（upsert）
- 定义 `subjects`（user: { id: string }）
- 配置基础 UI 主题

**验证方式：**
- `wrangler dev --remote` 本地启动
- 浏览器访问 auth 页面
- 完成邮箱验证码注册/登录全流程
- `wrangler d1 execute AUTH_DB --remote --command "SELECT * FROM user"` 确认用户已创建

---

### Step 4: Google OAuth 接入

**内容：**
- 在 `providers` 中添加 `GoogleProvider`
- 创建 Google Cloud Console OAuth 2.0 凭据
- 凭据存入 `.dev.vars`（本地）和 `wrangler secret put`（生产）
- `scopes: ["openid", "email"]`

**验证方式：**
- `wrangler dev --remote` 启动
- 点击"Google 登录"按钮
- 完成 OAuth 授权全流程

---

### Step 5: GitHub OAuth 接入

**内容：**
- 在 `providers` 中添加 `GithubProvider`
- 创建 GitHub OAuth App 凭据
- 凭据存入 `.dev.vars` 和 `wrangler secret`
- `scopes: ["user:email"]`

**验证方式：**
- `wrangler dev --remote` 启动
- 点击"GitHub 登录"按钮
- 完成 OAuth 授权全流程

---

### Step 6: 测试其他 Worker 对接

**内容：**
- 在项目中创建测试用 `test-client/` 目录
- 实现一个简单的 Client Worker，使用 `createClient` 对接 CfAuth
- 验证 access token 签发、验证、refresh 完整流程

**验证方式：**
- 部署 test-client Worker
- 通过 test-client 登录 CfAuth
- 验证 token 可正常获取和使用

---

### Step 7: 邮件服务接入

**内容：**
- 选择并接入免费邮件服务（Resend / SendGrid / SMTP）
- 替换 `sendCode` 回调中的 `console.log` 为真实邮件发送
- 配置邮件模板

**验证方式：**
- 实际邮箱收到验证码
- 使用验证码完成登录

## 6. 开发与测试策略

### 6.1 本地开发模式

```bash
# 本地开发直接绑定远程 D1
npx wrangler dev --remote

# 应用数据库迁移
npx wrangler d1 migrations apply AUTH_DB --remote

# 类型生成
npm run cf-typegen

# 类型检查 + 语法检查
npm run check
```

### 6.2 测试策略

| 阶段 | 方法 | 工具 |
|------|------|------|
| 开发验证 | 手动 + curl 测试 | wrangler dev --remote |
| 单元测试 | mock D1 binding 测试 Storage Adapter | Vitest / jest |
| 集成测试 | E2E 验证 OAuth 流程 | curl / Playwright |
| 数据验证 | 直接查询 D1 确认数据 | wrangler d1 execute |

### 6.3 环境变量与密钥

| 变量 | 说明 | 设置方式 |
|------|------|---------|
| `GOOGLE_CLIENT_ID` | Google OAuth Client ID | `.dev.vars` + `wrangler secret put` |
| `GOOGLE_CLIENT_SECRET` | Google OAuth Client Secret | `.dev.vars` + `wrangler secret put` |
| `GITHUB_CLIENT_ID` | GitHub OAuth Client ID | `.dev.vars` + `wrangler secret put` |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth Client Secret | `.dev.vars` + `wrangler secret put` |

`.dev.vars` 不提交到 Git，已在 `.gitignore` 中。

## 7. 项目文件结构

```
app/
├── src/
│   ├── index.ts              # Worker 入口，issuer 配置
│   └── storage.ts            # D1 Storage Adapter
├── migrations/
│   ├── 0001_create_user_table.sql
│   └── 0002_create_auth_storage_table.sql
├── test-client/               # 测试用对接 Worker（Step 6 创建）
│   └── src/
│       └── index.ts
├── package.json
├── tsconfig.json
├── wrangler.json
├── worker-configuration.d.ts
└── .dev.vars                  # 本地开发密钥（不提交）
```

## 8. 部署方案

### 8.1 生产部署

```bash
# 1. 设置密钥
npx wrangler secret put GOOGLE_CLIENT_ID
npx wrangler secret put GOOGLE_CLIENT_SECRET
npx wrangler secret put GITHUB_CLIENT_ID
npx wrangler secret put GITHUB_CLIENT_SECRET

# 2. 应用数据库迁移
npx wrangler d1 migrations apply AUTH_DB --remote

# 3. 部署
npm run deploy
```

### 8.2 其他 Worker 对接

业务 Worker 使用 `@openauthjs/openauth` 的 `createClient`：

```typescript
import { createClient } from '@openauthjs/openauth/client';

const client = createClient({
  clientID: 'my-worker',
  issuer: 'https://cf-auth.your-domain.workers.dev',
});

// 获取登录 URL
const { url } = await client.authorize();

// 验证 token
const verified = await client.verify(subjects, token);
```

## 9. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| D1 Storage Adapter 性能不满足 | 登录响应变慢 | 监控 D1 查询延迟，必要时加缓存层 |
| 无邮件服务无法验证码登录 | 邮箱登录不可用 | 开发阶段 console.log，上线前接邮件服务 |
| OAuth 凭据泄露 | 账号被盗 | 使用 wrangler secret，不在代码中硬编码 |
| D1 冷启动延迟 | 首次请求慢 | 可接受，后续请求 D1 连接池复用 |
