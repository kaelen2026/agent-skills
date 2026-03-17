---
name: api-design
version: 1.0.0
description: "REST API 设计技能。生成 OpenAPI 规范、Markdown 文档、路由代码。"
last_updated: 2026-03-17
---

# API Design

## When to Activate

- 用户请求设计 REST API 端点
- 用户请求生成 OpenAPI/Swagger 规范
- 用户请求 API 文档生成
- 用户请求 API 路由代码脚手架

## Workflow

### 1. 收集需求

向用户确认以下信息（未提供时主动询问）：

| 项目         | 说明                                        | 默认值                 |
| ------------ | ------------------------------------------- | ---------------------- |
| 资源名称     | API 管理的核心资源（如 users, orders）       | **必须**               |
| 操作         | CRUD 全部还是部分操作                        | CRUD 全部              |
| 认证方式     | Bearer Token / API Key / OAuth2 / None      | Bearer Token           |
| 框架         | Express / Next.js API Routes / FastAPI / Hono | 自动检测项目           |
| OpenAPI 版本 | 3.0 / 3.1                                   | 3.1                    |
| 分页         | cursor / offset / none                      | cursor                 |
| 版本策略     | URL prefix (`/v1`) / Header / None          | URL prefix             |
| 响应格式     | 统一信封 `{ success, data?, meta?, error? }` | 统一信封（不可更改）   |

### 2. 设计 API（结论先行）

先输出 API 端点总览表，再展开细节：

```
## API 端点总览

| Method | Path              | Description      | Auth |
| ------ | ----------------- | ---------------- | ---- |
| GET    | /v1/users         | 用户列表（分页） | Yes  |
| POST   | /v1/users         | 创建用户         | Yes  |
| GET    | /v1/users/:id     | 获取用户详情     | Yes  |
| PATCH  | /v1/users/:id     | 更新用户         | Yes  |
| DELETE | /v1/users/:id     | 删除用户         | Yes  |
```

### 3. 生成输出物

按以下顺序生成三个输出物：

#### A. OpenAPI 规范文件

- 文件: `openapi.yaml`（或 `openapi.json`）
- 遵循 [references/openapi-conventions.md](references/openapi-conventions.md) 中的规范
- 包含: paths, schemas, security, responses, error schemas

#### B. Markdown API 文档

- 文件: `API.md`
- 遵循 [references/documentation-template.md](references/documentation-template.md)
- 包含: 端点说明、请求/响应示例、错误码、认证说明

#### C. 路由代码

- 根据检测到的框架生成对应路由代码
- 遵循 [references/design-rules.md](references/design-rules.md)
- 包含: 路由定义、请求验证（zod）、错误处理、类型定义

### 4. 设计审查

生成完成后，自动检查以下规则（参见 [references/design-rules.md](references/design-rules.md)）：

- [ ] URL 使用复数名词（`/users` 而非 `/user`）
- [ ] 正确使用 HTTP 方法语义
- [ ] 一致的错误响应格式
- [ ] 统一响应信封（`success`, `data?`, `meta?`, `error?`）
- [ ] 分页参数规范
- [ ] 幂等性保证（PUT/DELETE）
- [ ] 适当的 HTTP 状态码
- [ ] 请求体验证（zod schema）
- [ ] 无敏感信息泄露

## Output Format

```
# API Design: {资源名称}

## 端点总览
{端点表格}

## OpenAPI 规范
{生成 openapi.yaml 文件}

## API 文档
{生成 API.md 文件}

## 路由代码
{生成框架对应的路由代码}

## 设计审查
{审查结果清单，标记通过/未通过}
```
