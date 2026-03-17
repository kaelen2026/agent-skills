---
name: backend-patterns
version: 1.0.0
description: "后端架构模式设计技能。生成分层架构代码（Repository / Service / Controller）、数据库查询优化、缓存策略、弹性模式（限流、后台任务、结构化日志）。"
last_updated: 2026-03-17
---

# Backend Patterns

## When to Activate

- 用户请求设计后端分层架构（Repository / Service / Controller）
- 用户请求数据库查询优化（N+1 问题、索引、连接池）
- 用户请求缓存策略设计（Redis、内存缓存、HTTP 缓存头）
- 用户请求限流（Rate Limiting）设计
- 用户请求后台任务/队列设计
- 用户请求结构化日志设计
- 用户请求中间件管道设计
- 用户请求事务模式设计

## Workflow

### 1. 现状分析

分析现有后端代码，确认以下信息：

| 项目           | 说明                                           | 确认方法                           |
| -------------- | ---------------------------------------------- | ---------------------------------- |
| 框架           | Express / Next.js App Router / Hono / FastAPI   | 检测 package.json / 项目结构        |
| 分层架构       | 是否存在 Repository / Service 分离              | 搜索项目目录结构                    |
| 数据库         | PostgreSQL / MySQL / MongoDB / Supabase         | 检测 ORM/客户端依赖                 |
| 缓存           | Redis / 内存缓存 / 无                           | 检测 Redis 客户端依赖               |
| 响应格式       | 是否遵循统一信封 `{ success, data?, meta?, error? }` | 确认现有 API 响应                  |
| 日志           | 是否使用结构化日志                              | 检测 logger 配置                    |

### 2. 架构设计（结论先行）

先输出架构分层总览表，再展开细节：

```
## 架构分层总览

| Layer        | 职责                          | 依赖             |
| ------------ | ----------------------------- | ---------------- |
| Controller   | 请求解析、响应构造、验证       | Service          |
| Service      | 业务逻辑、编排                 | Repository       |
| Repository   | 数据访问抽象                   | DB Client / ORM  |
| Middleware    | 横切关注点（认证、日志、限流） | 无状态           |
| Cache        | 缓存层装饰器                   | Repository       |
```

根据需求从以下 references 中选择适用的模式：

- 分层架构: [references/architecture-patterns.md](references/architecture-patterns.md)
- 数据库优化: [references/database-optimization.md](references/database-optimization.md)
- 缓存策略: [references/caching-strategies.md](references/caching-strategies.md)
- 弹性模式: [references/resilience-patterns.md](references/resilience-patterns.md)

### 3. 代码生成

按以下顺序生成输出物：

#### A. Repository 层

- 文件: `repositories/{resource}.repository.ts`
- 接口定义 + 具体实现
- 数据库错误映射
- 遵循 [references/architecture-patterns.md](references/architecture-patterns.md) 中的 Repository 模式

#### B. Service 层

- 文件: `services/{resource}.service.ts`
- 业务逻辑与数据访问分离
- 依赖注入（构造函数注入）
- 遵循 [references/architecture-patterns.md](references/architecture-patterns.md) 中的 Service 模式

#### C. 缓存层（如需要）

- 文件: `cache/{resource}.cache.ts`
- 装饰器模式包装 Repository
- 遵循 [references/caching-strategies.md](references/caching-strategies.md)

#### D. 中间件

- 文件: `middleware/{concern}.ts`
- 限流、日志、请求追踪
- 遵循 [references/resilience-patterns.md](references/resilience-patterns.md)

### 4. 设计审查

生成完成后，自动检查以下规则：

- [ ] Repository 接口与实现分离
- [ ] Service 层不直接访问数据库客户端
- [ ] Controller 不包含业务逻辑
- [ ] 统一响应信封 `{ success, data?, meta?, error? }`
- [ ] `meta.requestId` 始终存在
- [ ] 所有响应属性使用 camelCase
- [ ] `X-Device-Type` header 已验证
- [ ] N+1 查询问题已通过批量查询解决
- [ ] SELECT 仅选取需要的列（非 `SELECT *`）
- [ ] 事务中包含正确的回滚处理
- [ ] 缓存设置了合理的 TTL
- [ ] 缓存在数据变更时主动失效
- [ ] 限流策略已在关键端点上生效
- [ ] 结构化日志包含 requestId、timestamp、level
- [ ] 日志中不包含密码、令牌、PII
- [ ] 不可变模式（`readonly` 属性、新对象创建）
- [ ] 无硬编码配置值

## Output Format

```
# Backend Architecture: {项目名称}

## 架构分层总览
{分层表格}

## Repository 层
{接口定义 + 实现代码}

## Service 层
{业务逻辑代码}

## 缓存层
{缓存装饰器代码（如适用）}

## 中间件
{限流 / 日志 / 追踪代码}

## 数据库优化
{查询优化建议 + 索引建议}

## 设计审查
{审查结果清单，标记通过/未通过}
```
