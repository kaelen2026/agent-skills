# Migration Strategies & Patterns

## 迁移文件命名

### SQL 方式

```
{timestamp}_{description}.sql

示例:
20260317120000_create_users_table.sql
20260317120100_create_posts_table.sql
20260317120200_add_status_to_posts.sql
```

### ORM 方式

```
# Prisma: 自动生成
prisma/migrations/20260317120000_create_users_table/migration.sql

# Knex
knex migrate:make create_users_table
# -> migrations/20260317120000_create_users_table.ts

# Drizzle
drizzle-kit generate:pg --name create_users_table
# -> drizzle/0001_create_users_table.sql
```

## Forward-Only 迁移

**在生产环境中应避免 down 迁移**。原因：

1. 数据丢失风险（删除列 -> 数据消失）
2. 无法保证 down 迁移能正确运行
3. 在生产环境中几乎不会执行 down 迁移

取而代之，当出现问题时，通过**新的 forward 迁移**来修复。

```typescript
// BAD: down 迁移（在生产环境中危险）
export async function down(knex) {
  await knex.schema.dropTable("posts"); // 数据全部丢失
}

// GOOD: 如有问题，通过新的 forward 迁移修复
// 20260318_revert_posts_status_change.sql
ALTER TABLE posts ALTER COLUMN status TYPE TEXT;
```

## 安全迁移模式

### 添加列

**始终以 DEFAULT 值或 nullable 方式添加**。NOT NULL 且无 DEFAULT 会导致已有行报错：

```sql
-- GOOD: 以 nullable 方式添加
ALTER TABLE users ADD COLUMN phone TEXT;

-- GOOD: 带 DEFAULT 值添加
ALTER TABLE users ADD COLUMN is_verified BOOLEAN NOT NULL DEFAULT false;

-- BAD: NOT NULL 且无 DEFAULT（已有行违反约束）
ALTER TABLE users ADD COLUMN phone TEXT NOT NULL;
```

### 删除列（安全的 2 步法）

1. **步骤 1**: 从代码中移除对该列的读取并部署
2. **步骤 2**: 通过迁移删除该列

```sql
-- 步骤 2（代码部署后）
ALTER TABLE users DROP COLUMN legacy_field;
```

如果代码未先更新，部署期间引用该列的查询将会失败。

### 重命名列（安全的 4 步法）

直接 RENAME 有风险（部署期间应用仍引用旧名称）：

1. **添加新列** + 数据复制
2. **更新代码**: 同时写入新旧两列
3. **更新代码**: 仅读取新列
4. **删除旧列**

```sql
-- 步骤 1: 添加新列 + 数据复制
ALTER TABLE users ADD COLUMN display_name TEXT;
UPDATE users SET display_name = name;
ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;

-- 步骤 4（代码迁移完成后）: 删除旧列
ALTER TABLE users DROP COLUMN name;
```

### 添加索引

**使用 CONCURRENTLY**（PostgreSQL）。普通的 CREATE INDEX 会锁定表：

```sql
-- GOOD: 不锁表（PostgreSQL）
CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts(user_id);

-- BAD: 锁定整张表（阻塞写入）
CREATE INDEX idx_posts_user_id ON posts(user_id);
```

注意: `CONCURRENTLY` 不能在事务内使用。需要配置迁移工具：

```typescript
// Knex: 禁用事务
export const config = { transaction: false };

export async function up(knex) {
  await knex.raw(
    "CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts(user_id)"
  );
}
```

### 列类型变更

注意数据丢失风险。仅直接执行安全的变更：

```sql
-- 安全: 类型扩展（小 -> 大）
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12, 2);

-- 安全: TEXT -> VARCHAR（添加长度限制）
-- 但如果已有数据超过限制则报错
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(255);

-- 危险: VARCHAR -> INTEGER（需要数据转换）
-- 使用 添加新列 -> 数据转换 -> 删除旧列 的模式
```

## 零停机迁移（Expand-Contract 模式）

部署期间不产生停机的策略：

### Phase 1: Expand（扩展）

添加新结构，保留现有结构：

```sql
-- 添加新列（nullable，不影响现有代码）
ALTER TABLE orders ADD COLUMN total_cents BIGINT;
```

### Phase 2: Migrate（数据迁移）

在后台迁移数据：

```sql
-- 分批进行数据转换（避免一次性处理）
UPDATE orders
SET total_cents = ROUND(total_amount * 100)::BIGINT
WHERE total_cents IS NULL
LIMIT 1000;
```

### Phase 3: Code Deploy（代码部署）

部署兼容新旧两种结构的代码：

```typescript
// 同时写入新旧两列
const order = {
  totalAmount: amount,
  totalCents: Math.round(amount * 100),
};
```

### Phase 4: Contract（收缩）

删除旧结构：

```sql
-- 全部数据迁移完成后
ALTER TABLE orders DROP COLUMN total_amount;
ALTER TABLE orders ALTER COLUMN total_cents SET NOT NULL;
```

## 数据迁移 vs Schema 迁移

**分开处理**。原因：
- Schema 变更速度快，数据变更速度慢
- 回滚特性不同
- 数据迁移需要分批处理

```
migrations/
  20260317_001_add_total_cents_to_orders.sql       # Schema
  20260317_002_backfill_total_cents.sql             # 数据（独立文件）
  20260318_003_make_total_cents_not_null.sql        # Schema（迁移完成后）
```

### 数据迁移的批量处理

```sql
-- BAD: 一次性更新（表锁定、内存溢出）
UPDATE orders SET total_cents = ROUND(total_amount * 100)::BIGINT;

-- GOOD: 分批处理
DO $$
DECLARE
  batch_size INTEGER := 1000;
  rows_updated INTEGER;
BEGIN
  LOOP
    UPDATE orders
    SET total_cents = ROUND(total_amount * 100)::BIGINT
    WHERE id IN (
      SELECT id FROM orders
      WHERE total_cents IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    );

    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;

    PERFORM pg_sleep(0.1);  -- 降低负载
  END LOOP;
END $$;
```

## 种子数据

### 开发种子（Development Seeds）

用于测试的虚拟数据。不投入生产环境：

```typescript
// seeds/dev/users.ts
const devUsers = [
  { name: "Alice Developer", email: "alice@dev.example.com", role: "admin" },
  { name: "Bob Tester", email: "bob@dev.example.com", role: "viewer" },
];

export async function seed(db) {
  for (const user of devUsers) {
    await db("users")
      .insert({ ...user, id: crypto.randomUUID() })
      .onConflict("email")
      .ignore();
  }
}
```

### 生产参考数据（Production Reference Data）

主数据（国家代码、分类等）。作为迁移进行管理：

```sql
-- migrations/20260317_004_seed_categories.sql
INSERT INTO categories (id, name, slug) VALUES
  (gen_random_uuid(), 'Technology', 'technology'),
  (gen_random_uuid(), 'Science', 'science'),
  (gen_random_uuid(), 'Business', 'business')
ON CONFLICT (slug) DO NOTHING;
```

## 回滚策略

### 生产回滚流程

1. **立即**: 回滚有问题的代码部署
2. **评估**: 判断是否需要回滚迁移本身
3. **Forward Fix**: 通过新迁移修复（推荐）
4. **最终手段**: 时间点恢复（PITR）

### 迁移测试

在生产执行前务必测试：

```bash
# 1. 获取生产 DB 的快照（已脱敏）
pg_dump --schema-only production_db > schema.sql

# 2. 在测试 DB 上执行迁移
psql test_db < schema.sql
psql test_db < migrations/20260317_new_migration.sql

# 3. 运行应用测试套件
DATABASE_URL=test_db npm test
```

## 迁移检查清单

执行前确认所有项目：

### 安全性

- [ ] 确认不会发生表锁定（使用 `CONCURRENTLY`）
- [ ] 添加 `NOT NULL` 时是否有 `DEFAULT` 值
- [ ] 删除列前是否已从代码中移除引用
- [ ] 数据类型变更是否会导致数据丢失
- [ ] 大量数据更新是否采用分批处理

### 兼容性

- [ ] 部署期间旧版本代码是否能正常运行（零停机）
- [ ] 是否使用了 Expand-Contract 模式
- [ ] 回滚流程是否明确

### 测试

- [ ] 是否已在测试环境中执行迁移
- [ ] 迁移后应用测试套件是否通过
- [ ] 是否确认了对性能的影响（大表的情况）

### 运维

- [ ] 是否估算了迁移所需时间
- [ ] 是否计划在低流量时段执行（大规模变更的情况）
- [ ] DB 备份是否为最新
- [ ] 是否设置了监控告警
