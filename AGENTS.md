# AGENTS.md

本项目基于 openauth-template 开发 auth 认证服务，部署到 Cloudflare Workers。`openauth-template/` 是官方模板参考，`app/` 是实际项目代码，`doc/` 存放 PRD 及 AI 辅助文档。

## 项目结构

```
app/                # 实际项目代码（主体）
openauth-template/  # Cloudflare 官方模板（只读参考）
doc/                # PRD、设计文档、AI 指令等
```

原则：以 `openauth-template/` 为起点，在其基础上按 `doc/` 中文档要求修改，代码落在 `app/`。

## 代码约定

- TypeScript 严格模式（`strict: true`）
- 2 空格缩进，不使用 Tab
- 必须加分号，字符串用单引号
- 优先使用命名导入（`import { x } from "y"`）
- Worker handler 使用 `satisfies ExportedHandler<Env>` 类型标注
- 数据库绑定名用大写：`AUTH_DB`、`AUTH_STORAGE`
- 变量/函数用 camelCase，类型/接口用 PascalCase，常量用 UPPER_CASE

## 关键约束

- **改完 wrangler.json 绑定后类型报错**：运行 `npm run cf-typegen` 重新生成
- **nodejs_compat 兼容标志必须开启**：`valibot` 依赖此标志
- **密钥/环境变量禁止硬编码**：通过 `wrangler secret put` 设置，本地开发用 `.dev.vars`
- **`.env` 和 `.dev.vars` 不提交**：已在 `.gitignore` 中
