# Query Patterns & Optimization

## ORM 模式

### Prisma

#### Select with Relations

```typescript
const post = await prisma.post.findUnique({
  where: { id: postId },
  include: {
    user: { select: { id: true, name: true, email: true } },
    postTags: {
      include: { tag: true },
      where: { deletedAt: null },
    },
  },
});
```

#### Cursor-based Pagination

```typescript
const posts = await prisma.post.findMany({
  where: { deletedAt: null, status: "published" },
  take: limit + 1, // +1 用于判断 hasMore
  cursor: cursor ? { id: cursor } : undefined,
  orderBy: { createdAt: "desc" },
  include: { user: { select: { id: true, name: true } } },
});

const hasMore = posts.length > limit;
const data = hasMore ? posts.slice(0, -1) : posts;
const nextCursor = hasMore ? data[data.length - 1].id : null;

// API 响应（统一信封、camelCase）
// {
//   success: true,
//   data: [...],
//   meta: { requestId, cursor: nextCursor, hasMore }
// }
```

#### Offset-based Pagination

```typescript
const [data, total] = await Promise.all([
  prisma.post.findMany({
    where: { deletedAt: null },
    skip: offset,
    take: limit,
    orderBy: { createdAt: "desc" },
  }),
  prisma.post.count({ where: { deletedAt: null } }),
]);

// API 响应（统一信封、camelCase）
// {
//   success: true,
//   data: [...],
//   meta: { requestId, total, offset, limit }
// }
```

#### Transactions

```typescript
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { name, email, role: "viewer" },
  });

  const profile = await tx.userProfile.create({
    data: { userId: user.id, bio: "" },
  });

  return { user, profile };
});
```

#### Soft Delete Middleware

```typescript
// Prisma middleware: 自动应用 deleted_at 过滤
prisma.$use(async (params, next) => {
  if (params.action === "findMany" || params.action === "findFirst") {
    if (!params.args.where) {
      params.args.where = {};
    }
    if (params.args.where.deletedAt === undefined) {
      params.args.where.deletedAt = null;
    }
  }

  if (params.action === "delete") {
    params.action = "update";
    params.args.data = { deletedAt: new Date() };
  }

  if (params.action === "deleteMany") {
    params.action = "updateMany";
    if (!params.args.data) {
      params.args.data = {};
    }
    params.args.data.deletedAt = new Date();
  }

  return next(params);
});
```

### Drizzle

#### Select with Relations

```typescript
import { eq, isNull } from "drizzle-orm";

const result = await db.query.posts.findFirst({
  where: eq(posts.id, postId),
  with: {
    user: { columns: { id: true, name: true, email: true } },
    postTags: {
      with: { tag: true },
      where: isNull(postTags.deletedAt),
    },
  },
});
```

#### Cursor-based Pagination

```typescript
import { and, desc, isNull, lt } from "drizzle-orm";

const conditions = [isNull(posts.deletedAt), eq(posts.status, "published")];

if (cursor) {
  conditions.push(lt(posts.createdAt, cursorDate));
}

const result = await db
  .select()
  .from(posts)
  .where(and(...conditions))
  .orderBy(desc(posts.createdAt))
  .limit(limit + 1);
```

#### Transactions

```typescript
const result = await db.transaction(async (tx) => {
  const [user] = await tx
    .insert(users)
    .values({ name, email, role: "viewer" })
    .returning();

  const [profile] = await tx
    .insert(userProfiles)
    .values({ userId: user.id, bio: "" })
    .returning();

  return { user, profile };
});
```

### Knex

#### Select with Join

```typescript
const posts = await knex("posts")
  .select("posts.*", "users.name as user_name", "users.email as user_email")
  .join("users", "users.id", "posts.user_id")
  .where("posts.deleted_at", null)
  .orderBy("posts.created_at", "desc")
  .limit(limit)
  .offset(offset);
```

#### Transactions

```typescript
const result = await knex.transaction(async (trx) => {
  const [user] = await trx("users")
    .insert({ name, email, role: "viewer" })
    .returning("*");

  const [profile] = await trx("user_profiles")
    .insert({ user_id: user.id, bio: "" })
    .returning("*");

  return { user, profile };
});
```

## N+1 问题与解决方案

### 问题

```typescript
// BAD: N+1（1 次查询 + N 次关联查询）
const posts = await prisma.post.findMany();
for (const post of posts) {
  const user = await prisma.user.findUnique({ where: { id: post.userId } });
  // ... N 次额外查询
}
```

### 方案 1: Eager Loading（推荐）

```typescript
// GOOD: 1-2 次查询即可完成
const posts = await prisma.post.findMany({
  include: { user: true },
});
```

### 方案 2: DataLoader（适用于 GraphQL 和动态关联）

```typescript
import DataLoader from "dataloader";

const userLoader = new DataLoader(async (userIds: readonly string[]) => {
  const users = await prisma.user.findMany({
    where: { id: { in: [...userIds] } },
  });

  const userMap = new Map(users.map((u) => [u.id, u]));
  return userIds.map((id) => userMap.get(id) ?? null);
});

// 使用: 自动进行批量处理
const user = await userLoader.load(post.userId);
```

## 参数化查询

**绝对不使用字符串拼接** — 防止 SQL 注入：

```typescript
// BAD: SQL 注入漏洞
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// GOOD: 参数化查询
const result = await db.query("SELECT * FROM users WHERE email = $1", [email]);

// GOOD: ORM（自动参数化）
const user = await prisma.user.findUnique({ where: { email } });

// GOOD: Knex（自动参数化）
const user = await knex("users").where({ email }).first();
```

## 批量操作

### Bulk Insert

```typescript
// Prisma
await prisma.post.createMany({
  data: posts.map((p) => ({
    userId: p.userId,
    title: p.title,
    body: p.body,
    status: "draft",
  })),
  skipDuplicates: true,
});

// Knex
await knex("posts").insert(
  posts.map((p) => ({
    user_id: p.userId,
    title: p.title,
    body: p.body,
    status: "draft",
  }))
);
```

### Bulk Update

```typescript
// Knex: 条件批量更新
await knex("posts")
  .whereIn("id", postIds)
  .update({ status: "archived", updated_at: knex.fn.now() });

// Prisma
await prisma.post.updateMany({
  where: { id: { in: postIds } },
  data: { status: "archived" },
});
```

## Upsert 模式

```sql
-- PostgreSQL: INSERT ON CONFLICT
INSERT INTO user_settings (user_id, preferences)
VALUES ($1, $2)
ON CONFLICT (user_id)
DO UPDATE SET
  preferences = EXCLUDED.preferences,
  updated_at = now();
```

```typescript
// Prisma
await prisma.userSetting.upsert({
  where: { userId },
  create: { userId, preferences: defaultPrefs },
  update: { preferences: newPrefs },
});

// Drizzle
await db
  .insert(userSettings)
  .values({ userId, preferences: defaultPrefs })
  .onConflictDoUpdate({
    target: userSettings.userId,
    set: { preferences: newPrefs, updatedAt: new Date() },
  });
```

## 乐观锁

```sql
-- 添加 version 列
ALTER TABLE posts ADD COLUMN version INTEGER NOT NULL DEFAULT 1;

-- 更新时检查 version
UPDATE posts
SET title = $1, body = $2, version = version + 1, updated_at = now()
WHERE id = $3 AND version = $4;
-- rows affected = 0 表示发生冲突 -> 409 Conflict
```

```typescript
// Prisma 中的乐观锁
const updated = await prisma.post.updateMany({
  where: { id: postId, version: expectedVersion },
  data: { title, body, version: { increment: 1 } },
});

if (updated.count === 0) {
  throw new Error("OPTIMISTIC_LOCK_CONFLICT");
  // -> API: 409 Conflict
  // { success: false, error: { code: "OPTIMISTIC_LOCK_CONFLICT", message: "..." }, meta: { requestId } }
}
```

## 连接池

### 配置指南

| 参数                | 推荐值                            | 说明                    |
| ------------------- | --------------------------------- | ----------------------- |
| `pool.min`          | 2                                 | 最小连接数              |
| `pool.max`          | `max(2, CPU cores * 2 + 1)`      | 最大连接数              |
| `idleTimeoutMs`     | 10000 (10s)                       | 空闲连接超时            |
| `connectionTimeout` | 5000 (5s)                         | 获取连接超时            |

### Prisma 配置

```
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=5"
```

### Knex 配置

```typescript
const knex = require("knex")({
  client: "pg",
  connection: process.env.DATABASE_URL,
  pool: {
    min: 2,
    max: 10,
    idleTimeoutMillis: 10000,
    acquireTimeoutMillis: 5000,
  },
});
```

### Serverless 环境

在 Serverless 环境（Lambda, Vercel Functions）中使用连接池代理（PgBouncer, Supabase Pooler）：

```
# 直连（用于迁移）
DIRECT_URL="postgresql://user:pass@host:5432/db"

# 通过连接池代理（用于应用）
DATABASE_URL="postgresql://user:pass@pooler-host:6543/db?pgbouncer=true"
```

## 查询性能检查清单

### 必须确认项

- [ ] **EXPLAIN ANALYZE** 确认查询计划
- [ ] **避免 SELECT \*** — 仅选择所需列
- [ ] **验证索引使用** — 确认是否出现 Seq Scan
- [ ] **避免大 OFFSET** — 使用游标分页
- [ ] **限制结果集** — 始终设置 LIMIT
- [ ] **确认无 N+1 问题** — 使用 Eager Loading
- [ ] **保持事务简短** — 避免长时间锁定
- [ ] **检查未使用的索引** — 通过 `pg_stat_user_indexes` 确认

### EXPLAIN ANALYZE 的解读

```sql
EXPLAIN ANALYZE SELECT * FROM posts
WHERE user_id = '...' AND status = 'published'
ORDER BY created_at DESC
LIMIT 20;
```

确认要点：
- `Seq Scan` -> 需要索引（或索引不合适）
- `Index Scan` / `Index Only Scan` -> 良好
- `Nested Loop` + 大量行 -> 需要优化 JOIN
- `Sort` + 大量行 -> 需要排序用索引
- `actual time` vs `estimated rows` 偏差较大 -> 需要执行 ANALYZE

```sql
-- 更新表统计信息（提高规划器精度）
ANALYZE posts;
```
