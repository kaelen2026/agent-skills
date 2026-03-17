# Database Schema Design Patterns

## 命名规范

### 表名

- 复数形式 snake_case: `users`, `order_items`, `post_tags`
- 关联表: `{table_a}_{table_b}` 按字母顺序排列（`post_tags` 而非 `tag_posts`）

### 列名

- snake_case: `created_at`, `user_id`, `is_active`
- 主键: 始终为 `id`（推荐 UUID）
- 外键: `{singular_table}_id`（`user_id`, `order_id`）
- 布尔值: `is_` / `has_` 前缀（`is_active`, `has_verified`）
- 时间戳: `_at` 后缀（`created_at`, `updated_at`, `deleted_at`）

### 索引名

- 普通索引: `idx_{table}_{columns}`（`idx_users_email`）
- 唯一索引: `uniq_{table}_{columns}`（`uniq_users_email`）
- 复合索引: `idx_{table}_{col1}_{col2}`（`idx_orders_user_id_status`）

### ORM 映射

DB 的 snake_case 在应用层转换为 camelCase：

| DB Column      | App Property   |
| -------------- | -------------- |
| `created_at`   | `createdAt`    |
| `user_id`      | `userId`       |
| `is_active`    | `isActive`     |
| `deleted_at`   | `deletedAt`    |
| `order_items`  | `orderItems`   |

API 响应使用统一信封 `{ success, data?, meta?, error? }`，所有属性使用 camelCase。

## 标准审计列

**所有表必须包含**（无例外）：

```sql
id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
created_at  TIMESTAMPTZ NOT NULL    DEFAULT now(),
updated_at  TIMESTAMPTZ NOT NULL    DEFAULT now(),
deleted_at  TIMESTAMPTZ             -- 软删除，nullable
```

### updated_at 自动更新触发器

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 对每张表应用
CREATE TRIGGER trg_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

## 关联关系模式

### One-to-Many

外键放在子表一侧：

```sql
CREATE TABLE users (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT        NOT NULL,
  email       TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE TABLE posts (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID        NOT NULL REFERENCES users(id),
  title       TEXT        NOT NULL,
  body        TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
```

### Many-to-Many

使用关联表：

```sql
CREATE TABLE tags (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE TABLE post_tags (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id     UUID        NOT NULL REFERENCES posts(id),
  tag_id      UUID        NOT NULL REFERENCES tags(id),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  CONSTRAINT uniq_post_tags_post_id_tag_id UNIQUE (post_id, tag_id)
);

CREATE INDEX idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);
```

### One-to-One

外键 + 唯一约束：

```sql
CREATE TABLE user_profiles (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID        NOT NULL REFERENCES users(id),
  bio         TEXT,
  avatar_url  TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  CONSTRAINT uniq_user_profiles_user_id UNIQUE (user_id)
);
```

### 多态关联

**应避免**。改用关联表或独立外键：

```sql
-- BAD: 多态（无类型安全，无法使用外键约束）
CREATE TABLE comments (
  commentable_type TEXT,  -- 'post' | 'video'
  commentable_id   UUID
);

-- GOOD: 独立外键
CREATE TABLE comments (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id     UUID        REFERENCES posts(id),
  video_id    UUID        REFERENCES videos(id),
  body        TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  CONSTRAINT chk_comments_single_parent CHECK (
    (post_id IS NOT NULL AND video_id IS NULL) OR
    (post_id IS NULL AND video_id IS NOT NULL)
  )
);
```

## ENUM 处理

### PostgreSQL ENUM 类型

适用于少量固定值：

```sql
CREATE TYPE user_role AS ENUM ('admin', 'editor', 'viewer');

CREATE TABLE users (
  id    UUID      PRIMARY KEY DEFAULT gen_random_uuid(),
  role  user_role NOT NULL DEFAULT 'viewer',
  -- ...audit columns
);
```

### CHECK 约束

由于 ENUM 类型的修改（特别是删除值）较为困难，需要灵活性时使用 CHECK 约束：

```sql
CREATE TABLE orders (
  id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  status  TEXT NOT NULL DEFAULT 'pending',
  CONSTRAINT chk_orders_status CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
  -- ...audit columns
);
```

### 判断标准

| 条件                       | 推荐           |
| -------------------------- | -------------- |
| 值为 2-5 个且不会变更      | PostgreSQL ENUM |
| 值可能变更                 | CHECK 约束     |
| 值较多 / 动态              | 引用表         |

## JSON 列

### 适用场景

- 结构不固定的元数据（用户设置、外部 API 响应缓存）
- 嵌套较深、正规化成本较高的情况
- 不作为查询条件的数据

### 不适用场景

- 需要频繁过滤/排序的数据 -> 正规化
- 需要关联关系的数据 -> 独立表
- 需要数据完整性保证的数据 -> 拆分为独立列

```sql
-- 适用: 用户设置（结构不固定、不作为查询条件）
CREATE TABLE user_settings (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID        NOT NULL REFERENCES users(id),
  preferences JSONB       NOT NULL DEFAULT '{}',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  CONSTRAINT uniq_user_settings_user_id UNIQUE (user_id)
);

-- 如需按 JSONB 内的键过滤，使用 GIN 索引
CREATE INDEX idx_user_settings_preferences ON user_settings USING GIN (preferences);
```

## 索引策略

### 必须添加索引的情况

- **外键**: 直接影响关联查询性能
- **唯一字段**: `email`, `username`, `slug`
- **频繁过滤的列**: `status`, `type`, `role`
- **频繁排序的列**: `created_at`, `name`

### 复合索引的顺序（最左前缀规则）

```sql
-- 此索引对以下查询有效:
-- WHERE user_id = ? AND status = ?
-- WHERE user_id = ?
-- 对以下查询无效:
-- WHERE status = ?
CREATE INDEX idx_orders_user_id_status ON orders(user_id, status);
```

将基数（cardinality）较高的列放在前面。

### 软删除用部分索引

```sql
-- 仅对活跃记录建索引（排除已删除记录）
CREATE INDEX idx_users_email_active ON users(email)
  WHERE deleted_at IS NULL;

-- 软删除场景下的唯一约束
CREATE UNIQUE INDEX uniq_users_email_active ON users(email)
  WHERE deleted_at IS NULL;
```

### GIN 索引（JSON / 数组）

```sql
-- JSONB 列
CREATE INDEX idx_user_settings_preferences ON user_settings USING GIN (preferences);

-- 数组列
CREATE INDEX idx_posts_tag_ids ON posts USING GIN (tag_ids);

-- 仅对 JSONB 内的特定路径
CREATE INDEX idx_user_settings_theme ON user_settings USING GIN ((preferences -> 'theme'));
```

### 索引反模式

- 对所有列建索引 -> 写入性能下降
- 未使用的索引 -> 通过 `pg_stat_user_indexes` 确认后删除
- 低基数列的单独索引（如 `boolean`）-> 应包含在复合索引中

## 完整 Schema 示例: users + posts + tags

```sql
-- ========================================
-- updated_at 自动更新触发器函数
-- ========================================
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ========================================
-- ENUM 类型
-- ========================================
CREATE TYPE user_role AS ENUM ('admin', 'editor', 'viewer');
CREATE TYPE post_status AS ENUM ('draft', 'published', 'archived');

-- ========================================
-- users 表
-- ========================================
CREATE TABLE users (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT        NOT NULL,
  email       TEXT        NOT NULL,
  role        user_role   NOT NULL DEFAULT 'viewer',
  is_active   BOOLEAN     NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE UNIQUE INDEX uniq_users_email_active ON users(email)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role)
  WHERE deleted_at IS NULL;

CREATE TRIGGER trg_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ========================================
-- posts 表
-- ========================================
CREATE TABLE posts (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID        NOT NULL REFERENCES users(id),
  title       TEXT        NOT NULL,
  body        TEXT        NOT NULL,
  status      post_status NOT NULL DEFAULT 'draft',
  published_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_posts_user_id ON posts(user_id)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_posts_status ON posts(status)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_posts_published_at ON posts(published_at DESC)
  WHERE deleted_at IS NULL AND status = 'published';

CREATE TRIGGER trg_posts_updated_at
  BEFORE UPDATE ON posts
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ========================================
-- tags 表
-- ========================================
CREATE TABLE tags (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT        NOT NULL,
  slug        TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);

CREATE UNIQUE INDEX uniq_tags_slug_active ON tags(slug)
  WHERE deleted_at IS NULL;

CREATE TRIGGER trg_tags_updated_at
  BEFORE UPDATE ON tags
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ========================================
-- post_tags 关联表（Many-to-Many）
-- ========================================
CREATE TABLE post_tags (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id     UUID        NOT NULL REFERENCES posts(id),
  tag_id      UUID        NOT NULL REFERENCES tags(id),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  CONSTRAINT uniq_post_tags_post_id_tag_id UNIQUE (post_id, tag_id)
);

CREATE INDEX idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);

CREATE TRIGGER trg_post_tags_updated_at
  BEFORE UPDATE ON post_tags
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```
