# Database Optimization

## SELECT 最小化

始终仅选取需要的列，避免 `SELECT *`。

```typescript
// WRONG: 选取全部列
const { data } = await supabase
  .from("markets")
  .select("*");

// CORRECT: 仅选取需要的列
const { data } = await supabase
  .from("markets")
  .select("id, name, status, volume")
  .eq("status", "active")
  .order("volume", { ascending: false })
  .limit(10);
```

Drizzle ORM:

```typescript
// WRONG
const rows = await db.select().from(markets);

// CORRECT
const rows = await db
  .select({
    id: markets.id,
    name: markets.name,
    status: markets.status,
    volume: markets.volume,
  })
  .from(markets)
  .where(eq(markets.status, "active"))
  .orderBy(desc(markets.volume))
  .limit(10);
```

## N+1 查询问题

### 问题识别

```typescript
// WRONG: N+1 查询（1 + N 次数据库查询）
const markets = await getMarkets();             // 1 次查询
for (const market of markets) {
  market.creator = await getUser(market.creatorId);  // N 次查询
}
```

### 解决方案: 批量查询 + Map

```typescript
// CORRECT: 2 次查询（固定）
const markets = await getMarkets();

// 提取唯一 ID
const creatorIds = [...new Set(markets.map((m) => m.creatorId))];

// 批量获取
const creators = await getUsersByIds(creatorIds);
const creatorMap = new Map(creators.map((c) => [c.id, c]));

// 映射（创建新对象，不修改原对象）
const enrichedMarkets = markets.map((market) => ({
  ...market,
  creator: creatorMap.get(market.creatorId) ?? null,
}));
```

### 解决方案: JOIN 查询

```typescript
// Supabase: 使用外键关联
const { data } = await supabase
  .from("markets")
  .select(`
    id, name, status, volume,
    creator:users!creator_id (id, name, avatar_url)
  `)
  .eq("status", "active");

// Drizzle: 使用 JOIN
const rows = await db
  .select({
    id: markets.id,
    name: markets.name,
    status: markets.status,
    creatorName: users.name,
    creatorAvatar: users.avatarUrl,
  })
  .from(markets)
  .innerJoin(users, eq(markets.creatorId, users.id))
  .where(eq(markets.status, "active"));
```

## 事务模式

### Drizzle ORM 事务

```typescript
async function createMarketWithPosition(
  marketData: CreateMarketDto,
  positionData: CreatePositionDto
): Promise<{ market: Market; position: Position }> {
  return db.transaction(async (tx) => {
    const [market] = await tx
      .insert(markets)
      .values(marketData)
      .returning();

    const [position] = await tx
      .insert(positions)
      .values({
        ...positionData,
        marketId: market.id,
      })
      .returning();

    return { market, position };
  });
}
```

### Supabase RPC 事务

```sql
-- SQL 函数（在 Supabase SQL Editor 中创建）
CREATE OR REPLACE FUNCTION create_market_with_position(
  p_market_data jsonb,
  p_position_data jsonb
)
RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
  v_market_id uuid;
  v_result jsonb;
BEGIN
  INSERT INTO markets (name, description, category, creator_id)
  VALUES (
    p_market_data->>'name',
    p_market_data->>'description',
    p_market_data->>'category',
    (p_market_data->>'creatorId')::uuid
  )
  RETURNING id INTO v_market_id;

  INSERT INTO positions (market_id, user_id, amount, side)
  VALUES (
    v_market_id,
    (p_position_data->>'userId')::uuid,
    (p_position_data->>'amount')::numeric,
    p_position_data->>'side'
  );

  SELECT jsonb_build_object(
    'marketId', v_market_id
  ) INTO v_result;

  RETURN v_result;
EXCEPTION
  WHEN OTHERS THEN
    RAISE;  -- 自动回滚
END;
$$;
```

```typescript
// TypeScript 调用
const { data, error } = await supabase.rpc("create_market_with_position", {
  p_market_data: marketData,
  p_position_data: positionData,
});

if (error) {
  throw mapDatabaseError(error);
}
```

## 索引策略

### 常见索引类型

```sql
-- B-tree（默认）: 等值查询、范围查询、ORDER BY
CREATE INDEX idx_markets_status ON markets (status);
CREATE INDEX idx_markets_created_at ON markets (created_at DESC);

-- 复合索引: 多列过滤
-- 列顺序: 等值条件列在前，范围条件列在后
CREATE INDEX idx_markets_status_created
  ON markets (status, created_at DESC);

-- 部分索引: 仅索引常用子集
CREATE INDEX idx_markets_active
  ON markets (created_at DESC)
  WHERE status = 'active';

-- GIN 索引: 全文搜索、JSONB
CREATE INDEX idx_markets_tags ON markets USING gin (tags);

-- 唯一索引: 业务唯一性约束
CREATE UNIQUE INDEX idx_users_email ON users (lower(email));
```

### 索引使用原则

| 场景                        | 建议                                 |
| --------------------------- | ------------------------------------ |
| WHERE 子句中的常用过滤列     | 添加 B-tree 索引                     |
| 高选择性列（唯一值多）       | 优先索引                             |
| 低选择性列（如 boolean）     | 使用部分索引或复合索引               |
| ORDER BY + LIMIT 组合        | 添加与排序匹配的索引                 |
| JOIN 连接列                  | 确保外键列有索引                     |
| JSONB 字段查询               | 使用 GIN 索引                        |
| 写入频繁的表                 | 减少索引数量，避免写入放大           |

## 连接池

### PostgreSQL 连接池配置

```typescript
import postgres from "postgres";

const sql = postgres(process.env.DATABASE_URL!, {
  max: 10,                    // 最大连接数
  idle_timeout: 20,           // 空闲超时（秒）
  connect_timeout: 10,        // 连接超时（秒）
  max_lifetime: 60 * 30,     // 连接最大生存时间（秒）
});
```

### Serverless 环境注意事项

Serverless 函数（Vercel Functions、AWS Lambda）中每次冷启动都会创建新连接。必须使用连接池代理：

```typescript
// Neon Serverless Driver（推荐用于 Vercel）
import { neon } from "@neondatabase/serverless";

const sql = neon(process.env.DATABASE_URL!);

// 自动使用 Neon 的 HTTP 代理，无需管理连接池
const markets = await sql`
  SELECT id, name, status, volume
  FROM markets
  WHERE status = ${status}
  ORDER BY created_at DESC
  LIMIT ${limit}
`;
```

## 分页模式

### Cursor 分页（推荐）

```typescript
interface CursorPage<T> {
  readonly data: readonly T[];
  readonly nextCursor: string | null;
  readonly hasMore: boolean;
}

async function findAllWithCursor(
  cursor?: string,
  limit: number = 20
): Promise<CursorPage<Market>> {
  const queryLimit = limit + 1;  // 多取一条判断是否还有下一页

  let query = db
    .select({ id: markets.id, name: markets.name, createdAt: markets.createdAt })
    .from(markets)
    .orderBy(desc(markets.createdAt))
    .limit(queryLimit);

  if (cursor) {
    query = query.where(lt(markets.createdAt, new Date(cursor)));
  }

  const rows = await query;
  const hasMore = rows.length > limit;
  const data = hasMore ? rows.slice(0, limit) : rows;

  return {
    data,
    nextCursor: hasMore ? data[data.length - 1].createdAt.toISOString() : null,
    hasMore,
  };
}
```

### Offset 分页（简单场景）

```typescript
interface OffsetPage<T> {
  readonly data: readonly T[];
  readonly total: number;
  readonly page: number;
  readonly pageSize: number;
  readonly totalPages: number;
}

async function findAllWithOffset(
  page: number = 1,
  pageSize: number = 20
): Promise<OffsetPage<Market>> {
  const offset = (page - 1) * pageSize;

  const [data, countResult] = await Promise.all([
    db
      .select({ id: markets.id, name: markets.name })
      .from(markets)
      .orderBy(desc(markets.createdAt))
      .limit(pageSize)
      .offset(offset),
    db
      .select({ count: sql<number>`count(*)` })
      .from(markets),
  ]);

  const total = countResult[0].count;

  return {
    data,
    total,
    page,
    pageSize,
    totalPages: Math.ceil(total / pageSize),
  };
}
```

**注意**: Offset 分页在大数据集上性能较差（`OFFSET 10000` 仍需扫描前 10000 行）。数据量 > 10,000 行时，改用 Cursor 分页。
