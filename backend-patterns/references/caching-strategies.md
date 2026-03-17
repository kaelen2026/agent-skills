# Caching Strategies

## 缓存装饰器模式

用装饰器包装 Repository，实现透明缓存。业务层无需感知缓存存在。

### Redis 缓存装饰器

```typescript
import type { Redis } from "ioredis";

class CachedMarketRepository implements Repository<Market, CreateMarketDto, UpdateMarketDto> {
  constructor(
    private readonly base: Repository<Market, CreateMarketDto, UpdateMarketDto>,
    private readonly redis: Redis,
    private readonly ttlSeconds: number = 300
  ) {}

  async findById(id: string): Promise<Market | null> {
    const cacheKey = `market:${id}`;

    // 1. 查缓存
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // 2. 缓存未命中 → 查数据库
    const market = await this.base.findById(id);

    // 3. 写入缓存
    if (market) {
      await this.redis.setex(cacheKey, this.ttlSeconds, JSON.stringify(market));
    }

    return market;
  }

  async findAll(filters?: Record<string, unknown>): Promise<Market[]> {
    // 列表查询通常不缓存，或使用短 TTL
    return this.base.findAll(filters);
  }

  async create(data: CreateMarketDto): Promise<Market> {
    const market = await this.base.create(data);
    // 写入缓存
    await this.redis.setex(
      `market:${market.id}`,
      this.ttlSeconds,
      JSON.stringify(market)
    );
    return market;
  }

  async update(id: string, data: UpdateMarketDto): Promise<Market> {
    const market = await this.base.update(id, data);
    // 更新缓存
    await this.redis.setex(
      `market:${id}`,
      this.ttlSeconds,
      JSON.stringify(market)
    );
    return market;
  }

  async delete(id: string): Promise<void> {
    await this.base.delete(id);
    // 删除缓存
    await this.redis.del(`market:${id}`);
  }
}
```

### Upstash Redis（Serverless 推荐）

```typescript
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Upstash 使用 HTTP，无需连接池管理
// 天然适合 Serverless 环境
```

## Cache-Aside Pattern

最常见的缓存模式。应用代码负责缓存的读写。

```
读取流程:
  1. 查缓存 → 命中则返回
  2. 缓存未命中 → 查数据库
  3. 将结果写入缓存
  4. 返回结果

写入流程:
  1. 写数据库
  2. 删除/更新缓存
```

### 删除 vs 更新缓存

| 策略     | 优点                   | 缺点                     | 适用场景         |
| -------- | ---------------------- | ------------------------- | ---------------- |
| 删除缓存 | 简单、无数据不一致风险 | 下次读取有一次缓存未命中 | 大多数场景       |
| 更新缓存 | 避免缓存未命中         | 并发写入可能导致数据不一致 | 读多写少的热数据 |

**推荐**: 默认使用删除策略，仅在测量到缓存未命中率过高时才切换为更新策略。

## 多级缓存

### 内存缓存 + Redis

```typescript
import { LRUCache } from "lru-cache";

class TwoLevelCache {
  private readonly l1: LRUCache<string, string>;
  private readonly l2: Redis;

  constructor(redis: Redis) {
    this.l1 = new LRUCache<string, string>({
      max: 1000,         // 最多 1000 个 key
      ttl: 30_000,       // L1: 30 秒（短 TTL）
    });
    this.l2 = redis;
  }

  async get<T>(key: string): Promise<T | null> {
    // L1: 内存缓存
    const l1Value = this.l1.get(key);
    if (l1Value) {
      return JSON.parse(l1Value);
    }

    // L2: Redis
    const l2Value = await this.l2.get(key);
    if (l2Value) {
      // 回填 L1
      this.l1.set(key, l2Value);
      return JSON.parse(l2Value);
    }

    return null;
  }

  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    const serialized = JSON.stringify(value);
    this.l1.set(key, serialized);
    await this.l2.setex(key, ttlSeconds, serialized);
  }

  async invalidate(key: string): Promise<void> {
    this.l1.delete(key);
    await this.l2.del(key);
  }
}
```

**注意**: 在 Serverless 环境中，L1 内存缓存仅在同一函数实例中有效。冷启动后 L1 为空。适合热路径上减少 Redis 延迟。

## HTTP 缓存头

### 静态资源

```typescript
// Next.js: 静态资源自动处理
// 手动设置: 不可变资源
return new Response(data, {
  headers: {
    "Cache-Control": "public, max-age=31536000, immutable",
  },
});
```

### API 响应

```typescript
// 可缓存的公共数据
return NextResponse.json(
  { success: true, data: markets, meta: { requestId } },
  {
    headers: {
      "Cache-Control": "public, s-maxage=60, stale-while-revalidate=300",
    },
  }
);

// 不可缓存的用户特定数据
return NextResponse.json(
  { success: true, data: profile, meta: { requestId } },
  {
    headers: {
      "Cache-Control": "private, no-cache",
    },
  }
);
```

### Cache-Control 指令速查

| 指令                     | 含义                                        |
| ------------------------ | ------------------------------------------- |
| `public`                 | CDN 和浏览器均可缓存                        |
| `private`                | 仅浏览器可缓存（用户特定数据）              |
| `no-cache`               | 必须向服务器验证后才能使用缓存              |
| `no-store`               | 完全不缓存                                  |
| `max-age=N`              | 浏览器缓存 N 秒                              |
| `s-maxage=N`             | CDN 缓存 N 秒（覆盖 max-age）               |
| `stale-while-revalidate` | 缓存过期后仍返回旧数据，同时后台刷新        |
| `immutable`              | 资源永不变更，浏览器不会发送条件请求        |

## 缓存 Key 设计

```typescript
// 单资源: {entity}:{id}
`market:${id}`

// 列表: {entity}:list:{hash(filters)}
`market:list:${hashFilters(filters)}`

// 用户特定: {entity}:{id}:user:{userId}
`market:${id}:user:${userId}`

// 版本化: {entity}:{id}:v{version}
`market:${id}:v${market.version}`
```

### Key 命名规范

- 使用冒号 `:` 分隔层级
- 全部小写
- 避免过长（Redis key 也消耗内存）
- 列表缓存的 filter 使用稳定哈希（排序后的 JSON 字符串）

## 缓存失效策略

| 策略             | 实现                        | 适用场景                   |
| ---------------- | --------------------------- | -------------------------- |
| TTL 过期         | `SETEX key ttl value`       | 对数据新鲜度要求不高       |
| 主动删除         | 写入时 `DEL key`            | 强一致性要求               |
| 标签失效         | 维护 tag → keys 映射并批量删除 | 关联资源的级联失效         |
| 版本号           | key 中包含版本号            | 无需删除旧缓存             |
