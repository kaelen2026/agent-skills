# Architecture Patterns

## Repository Pattern

数据访问逻辑的抽象层。隔离业务逻辑与具体数据库实现。

### 接口定义

```typescript
interface Repository<T, CreateDto, UpdateDto> {
  findAll(filters?: Record<string, unknown>): Promise<T[]>;
  findById(id: string): Promise<T | null>;
  create(data: CreateDto): Promise<T>;
  update(id: string, data: UpdateDto): Promise<T>;
  delete(id: string): Promise<void>;
}
```

### 具体实现（Supabase）

```typescript
import type { SupabaseClient } from "@supabase/supabase-js";

interface MarketFilters {
  readonly status?: string;
  readonly limit?: number;
  readonly cursor?: string;
}

class SupabaseMarketRepository implements Repository<Market, CreateMarketDto, UpdateMarketDto> {
  constructor(private readonly client: SupabaseClient) {}

  async findAll(filters?: MarketFilters): Promise<Market[]> {
    let query = this.client
      .from("markets")
      .select("id, name, status, volume, created_at");

    if (filters?.status) {
      query = query.eq("status", filters.status);
    }

    if (filters?.cursor) {
      query = query.gt("id", filters.cursor);
    }

    if (filters?.limit) {
      query = query.limit(filters.limit);
    }

    const { data, error } = await query.order("created_at", { ascending: false });

    if (error) {
      throw mapDatabaseError(error);
    }

    return data;
  }

  async findById(id: string): Promise<Market | null> {
    const { data, error } = await this.client
      .from("markets")
      .select("id, name, status, volume, created_at, updated_at")
      .eq("id", id)
      .single();

    if (error?.code === "PGRST116") {
      return null;
    }

    if (error) {
      throw mapDatabaseError(error);
    }

    return data;
  }

  async create(dto: CreateMarketDto): Promise<Market> {
    const { data, error } = await this.client
      .from("markets")
      .insert(dto)
      .select("id, name, status, volume, created_at, updated_at")
      .single();

    if (error) {
      throw mapDatabaseError(error);
    }

    return data;
  }

  async update(id: string, dto: UpdateMarketDto): Promise<Market> {
    const { data, error } = await this.client
      .from("markets")
      .update({ ...dto, updated_at: new Date().toISOString() })
      .eq("id", id)
      .select("id, name, status, volume, created_at, updated_at")
      .single();

    if (error) {
      throw mapDatabaseError(error);
    }

    return data;
  }

  async delete(id: string): Promise<void> {
    const { error } = await this.client
      .from("markets")
      .delete()
      .eq("id", id);

    if (error) {
      throw mapDatabaseError(error);
    }
  }
}
```

### 具体实现（Drizzle ORM）

```typescript
import { eq, and, gt, desc } from "drizzle-orm";
import type { PostgresJsDatabase } from "drizzle-orm/postgres-js";
import { markets } from "./schema";

class DrizzleMarketRepository implements Repository<Market, CreateMarketDto, UpdateMarketDto> {
  constructor(private readonly db: PostgresJsDatabase) {}

  async findAll(filters?: MarketFilters): Promise<Market[]> {
    const conditions = [];

    if (filters?.status) {
      conditions.push(eq(markets.status, filters.status));
    }

    if (filters?.cursor) {
      conditions.push(gt(markets.id, filters.cursor));
    }

    return this.db
      .select({
        id: markets.id,
        name: markets.name,
        status: markets.status,
        volume: markets.volume,
        createdAt: markets.createdAt,
      })
      .from(markets)
      .where(conditions.length > 0 ? and(...conditions) : undefined)
      .orderBy(desc(markets.createdAt))
      .limit(filters?.limit ?? 20);
  }

  async findById(id: string): Promise<Market | null> {
    const rows = await this.db
      .select()
      .from(markets)
      .where(eq(markets.id, id))
      .limit(1);

    return rows[0] ?? null;
  }

  async create(dto: CreateMarketDto): Promise<Market> {
    const rows = await this.db
      .insert(markets)
      .values(dto)
      .returning();

    return rows[0];
  }

  async update(id: string, dto: UpdateMarketDto): Promise<Market> {
    const rows = await this.db
      .update(markets)
      .set({ ...dto, updatedAt: new Date() })
      .where(eq(markets.id, id))
      .returning();

    if (rows.length === 0) {
      throw new NotFoundError("Market", id);
    }

    return rows[0];
  }

  async delete(id: string): Promise<void> {
    const rows = await this.db
      .delete(markets)
      .where(eq(markets.id, id))
      .returning({ id: markets.id });

    if (rows.length === 0) {
      throw new NotFoundError("Market", id);
    }
  }
}
```

## Service Layer Pattern

业务逻辑与数据访问分离。Service 依赖 Repository 接口，不直接访问数据库。

```typescript
class MarketService {
  constructor(
    private readonly marketRepo: Repository<Market, CreateMarketDto, UpdateMarketDto>,
    private readonly userRepo: Repository<User, CreateUserDto, UpdateUserDto>
  ) {}

  async searchMarkets(query: string, limit: number = 10): Promise<Market[]> {
    const embedding = await generateEmbedding(query);
    const results = await this.vectorSearch(embedding, limit);

    // 批量获取完整数据（避免 N+1）
    const ids = results.map((r) => r.id);
    const markets = await this.marketRepo.findByIds(ids);

    // 按相似度排序
    const scoreMap = new Map(results.map((r) => [r.id, r.score]));
    return [...markets].sort((a, b) => {
      const scoreA = scoreMap.get(a.id) ?? 0;
      const scoreB = scoreMap.get(b.id) ?? 0;
      return scoreB - scoreA;
    });
  }

  async createMarket(dto: CreateMarketDto, creatorId: string): Promise<Market> {
    // 业务规则验证
    const creator = await this.userRepo.findById(creatorId);
    if (!creator) {
      throw new NotFoundError("User", creatorId);
    }

    if (creator.role !== "admin" && creator.role !== "moderator") {
      throw new ForbiddenError("Only admins and moderators can create markets");
    }

    return this.marketRepo.create({
      ...dto,
      creatorId,
      status: "draft",
    });
  }

  private async vectorSearch(
    embedding: readonly number[],
    limit: number
  ): Promise<ReadonlyArray<{ id: string; score: number }>> {
    // 向量搜索实现
    return [];
  }
}
```

## Controller Pattern（Next.js App Router）

Controller 仅负责请求解析、输入验证和响应构造。

```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

const createMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().max(2000).optional(),
  category: z.enum(["technology", "finance", "sports", "politics"]),
});

// GET /api/v1/markets
export async function GET(req: NextRequest): Promise<NextResponse> {
  const requestId = req.headers.get("x-request-id") ?? generateRequestId();
  const deviceType = req.headers.get("x-device-type");

  validateDeviceType(deviceType);

  const searchParams = req.nextUrl.searchParams;
  const filters: MarketFilters = {
    status: searchParams.get("status") ?? undefined,
    limit: Number(searchParams.get("limit") ?? 20),
    cursor: searchParams.get("cursor") ?? undefined,
  };

  const markets = await marketService.findAll(filters);

  return NextResponse.json({
    success: true,
    data: markets,
    meta: { requestId },
  });
}

// POST /api/v1/markets
export async function POST(req: NextRequest): Promise<NextResponse> {
  const requestId = req.headers.get("x-request-id") ?? generateRequestId();
  const deviceType = req.headers.get("x-device-type");

  validateDeviceType(deviceType);

  const body = await req.json();
  const validated = createMarketSchema.parse(body);

  const userId = await requireAuth(req);
  const market = await marketService.createMarket(validated, userId);

  return NextResponse.json(
    {
      success: true,
      data: market,
      meta: { requestId },
    },
    { status: 201 }
  );
}
```

## Middleware Pipeline Pattern

中间件按照单一职责原则设计，通过组合实现功能：

### Express

```typescript
import type { Request, Response, NextFunction } from "express";

// 中间件: requestId 注入
function requestIdMiddleware(req: Request, _res: Response, next: NextFunction): void {
  req.requestId = req.headers["x-request-id"] as string ?? generateRequestId();
  next();
}

// 中间件: 请求日志
function requestLoggerMiddleware(req: Request, res: Response, next: NextFunction): void {
  const start = Date.now();

  res.on("finish", () => {
    logger.info("Request completed", {
      requestId: req.requestId,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      durationMs: Date.now() - start,
    });
  });

  next();
}

// 组合管道
app.use(requestIdMiddleware);
app.use(requestLoggerMiddleware);
app.use(validateDeviceType);
app.use(authMiddleware);
```

### Next.js App Router（HOF 组合）

```typescript
type RouteHandler = (req: NextRequest) => Promise<NextResponse>;
type Middleware = (handler: RouteHandler) => RouteHandler;

function compose(...middlewares: readonly Middleware[]): Middleware {
  return (handler: RouteHandler): RouteHandler => {
    return middlewares.reduceRight(
      (acc, middleware) => middleware(acc),
      handler
    );
  };
}

// 使用
const withProtection = compose(
  withErrorHandling,
  withRequestId,
  withDeviceType,
  withAuth
);

export const GET = withProtection(async (req) => {
  // 已经过所有中间件处理
  const markets = await marketService.findAll();
  return NextResponse.json({ success: true, data: markets });
});
```

## 依赖注入

### 简单构造函数注入（推荐）

```typescript
// 组装（应用启动时）
function createContainer(config: AppConfig) {
  const db = createDatabaseClient(config.databaseUrl);

  const marketRepo = new DrizzleMarketRepository(db);
  const userRepo = new DrizzleUserRepository(db);

  const cachedMarketRepo = new CachedMarketRepository(
    marketRepo,
    createRedisClient(config.redisUrl)
  );

  const marketService = new MarketService(cachedMarketRepo, userRepo);
  const userService = new UserService(userRepo);

  return { marketService, userService } as const;
}

// 使用（路由中）
const { marketService, userService } = createContainer(config);
```

**重要**: 避免使用 DI 容器框架（InversifyJS 等），除非项目规模确实需要。构造函数注入足以满足大多数 Node.js 项目需求。
