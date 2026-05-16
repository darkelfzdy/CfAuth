# OpenAuth 代理指南

本文件为AI代理（或开发者）提供OpenAuth在Cloudflare Workers上的关键配置信息。完整文档请参考：[OpenAuth官方文档](https://openauth.js.org/docs/)

## 支持的认证提供商

OpenAuth支持多种OAuth2提供商和密码认证，常见的有：

- `password`：邮箱+验证码（本项目已实现）
- `google`：Google OAuth2
- `github`：GitHub OAuth2
- 其他：如Microsoft、Apple、自定义OAuth2等

## 核心配置结构

在`src/index.ts`中，`issuer`函数接受以下主要参数：

- **storage**：持久化存储，必须实现`Storage`接口。本项目使用`CloudflareStorage`（基于KV）。
- **subjects**：定义客户端可用的主题类型（如`user`），用于类型安全的令牌验证。
- **providers**：一个对象，键为提供商名称，值为对应的Provider实例。可以同时启用多个。
- **theme**：自定义登录页面的UI（标题、颜色、logo等）。
- **success**：用户成功认证后调用的回调，返回主题（subject）并附加额外数据。

## 添加新提供商（以Google为例）

1. 安装对应的提供商包（如果尚未安装）：
   ```bash
   npm install @openauthjs/openauth-provider-google
   ```
2. 在`providers`对象中引入并配置：

   ```typescript
   import { GoogleProvider } from "@openauthjs/openauth/provider/google";

   // ...在providers内
   google: GoogleProvider({
     clientId: env.GOOGLE_CLIENT_ID,
     clientSecret: env.GOOGLE_CLIENT_SECRET,
     scopes: ["openid", "email", "profile"],
   }),
   ```

3. 在`wrangler.json`或通过wrangler secrets设置环境变量`GOOGLE_CLIENT_ID`和`GOOGLE_CLIENT_SECRET`。
4. 在成功回调中，根据提供的信息（如`value.email`）创建/查找用户。

## 主题定制示例

```typescript
theme: {
  title: "MyAuth",
  primary: "#0051c3",
  favicon: "https://example.com/favicon.ico",
  logo: { dark: "...", light: "..." },
}
```

## 客户端集成

任何支持OAuth2的客户端都可以与此Worker集成。认证流程：

1. 引导用户访问`https://your-worker.dev/authorize?client_id=...&redirect_uri=...&response_type=code`。
2. 用户登录并授权后，携带`code`重定向到`redirect_uri`。
3. 服务端通过`code`换取access token（使用`/token`端点）。

在Worker内，您可以使用`@openauthjs/openauth/client`创建类型安全的客户端。

## 注意事项

- 生产环境务必替换`sendCode`函数，使用真实的邮件服务（如Resend）。
- 确保D1数据库和KV命名空间已正确创建并绑定到Worker。
- 环境变量（如OAuth客户端密钥）应通过`wrangler secret`设置，勿硬编码。
- 可通过`wrangler tail`实时查看日志调试。

## 更多资源

- [OpenAuth文档](https://openauth.js.org/docs/)
- [本模板README](./README.md)
- [Cloudflare Workers文档](https://developers.cloudflare.com/workers/)
