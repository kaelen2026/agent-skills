---
name: backend-performance
version: 1.0.0
description: 后端与 API 性能优化参考
last_updated: 2026-03-17
---

# 后端与 API 性能

## API 响应时间目标

所有 API 端点应满足以下分位数目标：

| 分位数 | 目标 | 说明 |
|--------|------|------|
| P50 | < 100ms | 一半请求应在 100ms 内完成 |
| P95 | < 500ms | 95% 的请求应在 500ms 内完成 |
| P99 | < 1s | 99% 的请求应在 1s 内完成 |

**关键原则**：始终关注 P95/P99 而非平均值。平均值会掩盖长尾延迟问题。

```typescript
// 在中间件中记录响应时间
function responseTimeMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const duration = Number(process.hrtime.bigint() - start) / 1_000_000; // ms
    metrics.recordResponseTime(req.path, req.method, duration);

    if (duration > 500) {
      logger.warn('Slow API response', {
        path: req.path,
        method: req.method,
        duration_ms: duration,
        status: res.statusCode,
      });
    }
  });

  next();
}
```

## 数据库查询优化

### EXPLAIN ANALYZE

在优化查询前，必须先用 `EXPLAIN ANALYZE` 了解查询的实际执行计划。

```sql
-- 查看查询的实际执行计划和耗时
EXPLAIN ANALYZE
SELECT u.id, u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 20;

-- 关注以下信息：
-- Seq Scan（全表扫描）→ 考虑添加索引
-- Nested Loop（嵌套循环）→ 检查是否可用 Hash Join
-- Sort（排序）→ 检查是否可用索引排序
-- actual time / rows → 实际耗时和行数
```

### 索引策略

```sql
-- 为高频查询条件创建索引
CREATE INDEX idx_users_created_at ON users (created_at);

-- 复合索引：列顺序应遵循"等值条件在前，范围条件在后"
CREATE INDEX idx_orders_user_status_date
ON orders (user_id, status, created_at);

-- 部分索引：只索引需要查询的子集
CREATE INDEX idx_orders_pending
ON orders (created_at)
WHERE status = 'pending';

-- 覆盖索引：索引包含查询所需的所有列，避免回表
CREATE INDEX idx_users_email_name
ON users (email) INCLUDE (name);
```

### N+1 查询检测与解决

N+1 是最常见的数据库性能问题。1 次查询获取列表，再为每条记录发起 1 次查询。

```typescript
// 不良：N+1 查询（1 次查询用户 + N 次查询订单）
const users = await db.query('SELECT * FROM users LIMIT 100');
for (const user of users) {
  user.orders = await db.query('SELECT * FROM orders WHERE user_id = $1', [user.id]);
}

// 良好：JOIN 一次查询完成
const usersWithOrders = await db.query(`
  SELECT u.*, json_agg(o.*) as orders
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id
  GROUP BY u.id
  LIMIT 100
`);

// 良好：使用 DataLoader 批量查询（适用于 GraphQL 场景）
const orderLoader = new DataLoader(async (userIds: string[]) => {
  const orders = await db.query(
    'SELECT * FROM orders WHERE user_id = ANY($1)',
    [userIds]
  );
  return userIds.map(id => orders.filter(o => o.user_id === id));
});
```

## 缓存层级

有效的缓存架构应采用多层级策略，越靠近用户命中率越高、延迟越低。

```
用户请求
  ↓
[CDN 缓存] ← 静态资源、API 响应缓存（延迟 ~10ms）
  ↓ 未命中
[应用层缓存] ← Redis/Memcached（延迟 ~1-5ms）
  ↓ 未命中
[数据库查询缓存] ← 数据库内置缓存 / 物化视图
  ↓ 未命中
[数据库] ← 实际查询（延迟 ~10-100ms）
```

### CDN 缓存

```typescript
// API 路由设置 CDN 缓存头
// s-maxage: CDN 缓存时间
// stale-while-revalidate: CDN 在后台更新期间继续提供旧数据
export async function GET(request: Request) {
  const data = await fetchData();

  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  });
}
```

## Redis 缓存模式

### Cache-Aside（旁路缓存）

最常用的模式。应用负责维护缓存的读写。

```typescript
async function getUser(userId: string): Promise<User> {
  const cacheKey = `user:${userId}`;

  // 1. 先查缓存
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. 缓存未命中，查数据库
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

  // 3. 写入缓存（设置 TTL）
  await redis.set(cacheKey, JSON.stringify(user), 'EX', 3600); // 1小时过期

  return user;
}

// 数据更新时清除缓存
async function updateUser(userId: string, data: Partial<User>): Promise<User> {
  const updatedUser = await db.query(
    'UPDATE users SET name = $1 WHERE id = $2 RETURNING *',
    [data.name, userId]
  );

  // 清除旧缓存，下次读取时重新填充
  await redis.del(`user:${userId}`);

  return updatedUser;
}
```

### Write-Through（穿透写入）

写入时同时更新缓存和数据库，保证缓存与数据库一致。

```typescript
async function updateUser(userId: string, data: Partial<User>): Promise<User> {
  const updatedUser = await db.query(
    'UPDATE users SET name = $1 WHERE id = $2 RETURNING *',
    [data.name, userId]
  );

  // 写入数据库后立即更新缓存
  await redis.set(`user:${userId}`, JSON.stringify(updatedUser), 'EX', 3600);

  return updatedUser;
}
```

### TTL 策略

TTL（生存时间）的设置需要在数据新鲜度和缓存命中率之间取得平衡。

| 数据类型 | 推荐 TTL | 理由 |
|----------|----------|------|
| 用户会话 | 30 分钟 | 安全性优先 |
| 用户个人信息 | 1 小时 | 变更不频繁 |
| 产品列表 | 5-15 分钟 | 需要较新数据 |
| 配置数据 | 24 小时 | 极少变更 |
| 统计数据/报表 | 1-6 小时 | 允许一定延迟 |
| 静态内容 | 7 天 | 几乎不变 |

```typescript
// 根据数据类型设置不同 TTL
const TTL = {
  SESSION: 1800,       // 30 分钟
  USER_PROFILE: 3600,  // 1 小时
  PRODUCT_LIST: 900,   // 15 分钟
  CONFIG: 86400,       // 24 小时
  REPORT: 21600,       // 6 小时
} as const;

await redis.set(key, value, 'EX', TTL.USER_PROFILE);
```

## 连接池

数据库连接创建开销大（TCP 握手、认证、协议协商），必须使用连接池复用连接。

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  // 连接池配置
  min: 5,              // 最小空闲连接数
  max: 20,             // 最大连接数
  idleTimeoutMillis: 30000,     // 空闲连接超时（30秒）
  connectionTimeoutMillis: 5000, // 获取连接超时（5秒）
});

// 监控连接池状态
setInterval(() => {
  logger.info('Pool status', {
    total: pool.totalCount,
    idle: pool.idleCount,
    waiting: pool.waitingCount,
  });
}, 60000);
```

**连接池大小经验公式**：`最大连接数 = CPU 核心数 * 2 + 磁盘数`。大多数场景下 20-50 就足够。

## 异步处理

耗时操作不应阻塞 API 响应。使用后台任务和消息队列。

### 后台任务

```typescript
// API 立即返回，耗时操作异步执行
export async function POST(request: Request) {
  const { email, data } = await request.json();

  // 将任务加入队列，立即返回
  await jobQueue.add('send-report', { email, data });

  return Response.json({ status: 'processing', message: '报告生成中，完成后将发送至邮箱' });
}
```

### 消息队列（BullMQ 示例）

```typescript
import { Queue, Worker } from 'bullmq';

// 生产者：添加任务
const reportQueue = new Queue('reports', {
  connection: { host: 'localhost', port: 6379 },
});

await reportQueue.add('generate', {
  userId: '123',
  reportType: 'monthly',
}, {
  attempts: 3,              // 最多重试 3 次
  backoff: { type: 'exponential', delay: 1000 }, // 指数退避
  removeOnComplete: 1000,   // 保留最近 1000 个完成任务
  removeOnFail: 5000,       // 保留最近 5000 个失败任务
});

// 消费者：处理任务
const worker = new Worker('reports', async (job) => {
  const { userId, reportType } = job.data;
  const report = await generateReport(userId, reportType);
  await sendEmail(userId, report);
}, {
  connection: { host: 'localhost', port: 6379 },
  concurrency: 5,  // 并发处理 5 个任务
});
```

## 响应压缩

压缩可以显著减少网络传输量，特别是对 JSON API 响应效果显著。

```typescript
// Express 中间件
import compression from 'compression';

app.use(compression({
  filter: (req, res) => {
    // 不压缩已压缩的内容（图片、视频等）
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  },
  threshold: 1024, // 仅压缩大于 1KB 的响应
}));
```

```nginx
# Nginx 配置 Brotli 和 Gzip
# Brotli 比 Gzip 压缩率高 15-20%
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript text/xml;

gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript text/xml;
```

## API 分页

### 游标分页（推荐用于大数据集）

游标分页性能稳定，不受数据集大小影响。偏移量分页在深页时性能急剧下降。

```typescript
// 游标分页：O(1) 性能，与总数据量无关
async function getUsers(cursor?: string, limit: number = 20) {
  const query = cursor
    ? `SELECT * FROM users WHERE id > $1 ORDER BY id ASC LIMIT $2`
    : `SELECT * FROM users ORDER BY id ASC LIMIT $1`;

  const params = cursor ? [cursor, limit + 1] : [limit + 1];
  const rows = await db.query(query, params);

  const hasMore = rows.length > limit;
  const items = hasMore ? rows.slice(0, limit) : rows;
  const nextCursor = hasMore ? items[items.length - 1].id : null;

  return {
    items,
    nextCursor,
    hasMore,
  };
}
```

### 偏移量分页（简单场景可用）

```typescript
// 偏移量分页：简单但深页性能差 — O(offset + limit)
// 仅在总数据量小或不需要深翻页时使用
async function getUsers(page: number = 1, limit: number = 20) {
  const offset = (page - 1) * limit;

  const [items, countResult] = await Promise.all([
    db.query('SELECT * FROM users ORDER BY id LIMIT $1 OFFSET $2', [limit, offset]),
    db.query('SELECT COUNT(*) FROM users'),
  ]);

  return {
    items,
    page,
    totalPages: Math.ceil(countResult[0].count / limit),
    totalItems: countResult[0].count,
  };
}
```

**选择建议**：

| 场景 | 推荐 | 理由 |
|------|------|------|
| 无限滚动、消息流 | 游标分页 | 性能稳定，适合连续加载 |
| 管理后台表格 | 偏移量分页 | 需要跳转到指定页 |
| 数据量 > 10 万行 | 游标分页 | 偏移量分页深页性能不可接受 |
| 数据量 < 1 万行 | 偏移量分页 | 简单，性能影响可忽略 |

## 限流作为性能保护

限流不仅是安全措施，也是防止系统过载的性能保护机制。

```typescript
import rateLimit from 'express-rate-limit';

// 全局限流
const globalLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 分钟窗口
  max: 100,              // 每窗口最多 100 次请求
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: '请求过于频繁，请稍后再试' },
});

// 针对高成本 API 的严格限流
const heavyApiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10,               // 每分钟最多 10 次
  message: { error: '此接口请求频率受限' },
});

app.use('/api/', globalLimiter);
app.use('/api/reports/generate', heavyApiLimiter);
```

**限流策略选择**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| 固定窗口 | 每个时间窗口固定配额 | 简单场景 |
| 滑动窗口 | 更平滑的速率控制 | 通用 API |
| 令牌桶 | 允许突发流量 | 需要弹性的场景 |
| 漏桶 | 恒定处理速率 | 需要稳定吞吐的场景 |
