---
name: database
description: "Database 设计技能。Schema 设计、迁移生成、查询优化、ORM 代码生成。"
metadata:
  filePattern:
    - "**/schema.prisma"
    - "**/drizzle.config.*"
    - "**/migrations/**/*"
    - "**/knexfile.*"
  bashPattern:
    - "prisma|drizzle|knex migrate|psql|mysql"
  priority: 7
---

# Database Design

## When to Activate

- 用户请求设计数据库 Schema
- 用户请求生成迁移
- 用户请求查询优化或索引策略
- 用户请求生成 ORM 模型/Schema 代码
- 用户请求审查表设计

## Workflow

### 1. 收集需求

向用户确认以下信息（未提供时主动询问）：

| 项目           | 说明                                              | 默认值               |
| -------------- | ------------------------------------------------- | -------------------- |
| 数据库类型     | PostgreSQL / MySQL / SQLite                       | PostgreSQL           |
| ORM            | Prisma / Drizzle / Knex / TypeORM / None (raw SQL) | 自动检测             |
| 命名规范       | DB: snake_case, App: camelCase                    | 统一（不可更改）     |
| 软删除         | `deleted_at` 列实现逻辑删除                       | 启用                 |
| 审计列         | `id`, `created_at`, `updated_at`, `deleted_at`    | 必须（不可更改）     |
| 多租户         | 行级隔离 / Schema 隔离 / DB 隔离 / None          | None                 |
| UUID 策略      | `gen_random_uuid()` / `uuid_generate_v4()` / 应用层生成 | `gen_random_uuid()` |
| 时区           | 使用 TIMESTAMPTZ                                  | 必须（不可更改）     |

### 2. Schema 设计（结论先行）

先输出表总览，再展开细节：

```
## 表总览

| Table          | Description      | Key Relations            |
| -------------- | ---------------- | ------------------------ |
| users          | 用户管理         | has_many: posts          |
| posts          | 文章管理         | belongs_to: users        |
| tags           | 标签管理         | many_to_many: posts      |
| post_tags      | 文章-标签关联表  | belongs_to: posts, tags  |
```

### 3. 生成输出物

按以下顺序生成三个输出物：

#### A. Schema 定义（SQL）

- 文件: `schema.sql` 或迁移文件
- 遵循: [references/schema-design.md](references/schema-design.md) 中的规范
- 包含: CREATE TABLE、索引、约束、触发器

#### B. ORM 模型代码

- 根据检测到的 ORM 生成对应模型/Schema 代码
- 遵循: [references/query-patterns.md](references/query-patterns.md)
- 包含: 模型定义、关联关系、验证、类型定义
- ORM 映射: DB `snake_case` <-> App `camelCase`

#### C. 迁移文件

- 遵循: [references/migration-patterns.md](references/migration-patterns.md)
- 包含: 安全的迁移步骤、回滚考量

### 4. 设计审查

生成完成后，自动检查以下规则（参见 [references/schema-design.md](references/schema-design.md)）：

- [ ] 表名使用复数形式 snake_case（`users` 而非 `user`）
- [ ] 列名使用 snake_case（`created_at` 而非 `createdAt`）
- [ ] 所有表包含 `id`, `created_at`, `updated_at`
- [ ] 启用软删除时，`deleted_at` 列存在
- [ ] 外键上存在索引
- [ ] 唯一约束已正确设置
- [ ] 主键使用 UUID（`gen_random_uuid()`）
- [ ] 时间戳使用 TIMESTAMPTZ
- [ ] ENUM 类型或 CHECK 约束使用得当
- [ ] ORM 查询已考虑 N+1 问题（schema/索引层 owner，应用层 → 参见 [backend-patterns](../backend-patterns/SKILL.md)）
- [ ] 迁移支持零停机部署
- [ ] 使用参数化查询（防止 SQL 注入）

## Output Format

```
# Database Design: {资源名称}

## 表总览
{表格}

## Schema 定义
{生成 schema.sql 或迁移文件}

## ORM 模型
{生成 ORM 对应的模型代码}

## 迁移
{生成迁移文件}

## 设计审查
{审查结果清单，标记通过/未通过}
```
