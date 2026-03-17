# Resilience Patterns

## 限流（Rate Limiting）

### 滑动窗口限流器（Redis 实现）

```typescript
import type { Redis } from "ioredis";

interface RateLimitResult {
  readonly allowed: boolean;
  readonly remaining: number;
  readonly resetAt: number;
}

async function checkRateLimit(
  redis: Redis,
  identifier: string,
  maxRequests: number,
  windowMs: number
): Promise<RateLimitResult> {
  const key = `ratelimit:${identifier}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  // Lua 脚本保证原子性
  const script = `
    redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[1])
    local count = redis.call('ZCARD', KEYS[1])
    if count < tonumber(ARGV[2]) then
      redis.call('ZADD', KEYS[1], ARGV[3], ARGV[3])
      redis.call('PEXPIRE', KEYS[1], ARGV[4])
      return {1, tonumber(ARGV[2]) - count - 1}
    else
      return {0, 0}
    end
  `;

  const result = await redis.eval(
    script,
    1,
    key,
    String(windowStart),
    String(maxRequests),
    String(now),
    String(windowMs)
  ) as [number, number];

  return {
    allowed: result[0] === 1,
    remaining: result[1],
    resetAt: now + windowMs,
  };
}
```

### 限流中间件（Express）

```typescript
function rateLimitMiddleware(
  redis: Redis,
  config: { readonly maxRequests: number; readonly windowMs: number }
) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const identifier = req.headers["x-forwarded-for"] as string
      ?? req.socket.remoteAddress
      ?? "unknown";

    const result = await checkRateLimit(
      redis,
      identifier,
      config.maxRequests,
      config.windowMs
    );

    res.set({
      "X-RateLimit-Limit": String(config.maxRequests),
      "X-RateLimit-Remaining": String(result.remaining),
      "X-RateLimit-Reset": String(Math.ceil(result.resetAt / 1000)),
    });

    if (!result.allowed) {
      throw new RateLimitError();
    }

    next();
  };
}

// 使用: 每分钟 100 次请求
app.use("/api/v1", rateLimitMiddleware(redis, {
  maxRequests: 100,
  windowMs: 60_000,
}));
```

### 限流中间件（Next.js App Router）

```typescript
import { NextRequest, NextResponse } from "next/server";

function withRateLimit(
  handler: (req: NextRequest) => Promise<NextResponse>,
  config: { readonly maxRequests: number; readonly windowMs: number }
) {
  return async (req: NextRequest): Promise<NextResponse> => {
    const identifier = req.headers.get("x-forwarded-for")
      ?? req.ip
      ?? "unknown";

    const result = await checkRateLimit(
      redis,
      identifier,
      config.maxRequests,
      config.windowMs
    );

    if (!result.allowed) {
      return NextResponse.json(
        {
          success: false,
          error: {
            code: "RATE_LIMIT_EXCEEDED",
            message: "Too many requests. Please try again later.",
          },
          meta: {
            requestId: req.headers.get("x-request-id") ?? generateRequestId(),
          },
        },
        {
          status: 429,
          headers: {
            "Retry-After": String(Math.ceil(config.windowMs / 1000)),
            "X-RateLimit-Limit": String(config.maxRequests),
            "X-RateLimit-Remaining": "0",
            "X-RateLimit-Reset": String(Math.ceil(result.resetAt / 1000)),
          },
        }
      );
    }

    const response = await handler(req);
    response.headers.set("X-RateLimit-Limit", String(config.maxRequests));
    response.headers.set("X-RateLimit-Remaining", String(result.remaining));
    response.headers.set("X-RateLimit-Reset", String(Math.ceil(result.resetAt / 1000)));

    return response;
  };
}
```

## 后台任务模式

### Vercel 环境: waitUntil / after

```typescript
import { after } from "next/server";

export async function POST(req: NextRequest): Promise<NextResponse> {
  const requestId = req.headers.get("x-request-id") ?? generateRequestId();

  const market = await marketService.create(data);

  // 响应已发送后执行后台任务
  after(async () => {
    await indexService.updateSearchIndex(market.id);
    await notificationService.notifySubscribers(market);
    await analyticsService.trackMarketCreated(market);
  });

  return NextResponse.json(
    { success: true, data: market, meta: { requestId } },
    { status: 201 }
  );
}
```

### 简易内存队列（开发/小规模用）

```typescript
interface Job<T> {
  readonly id: string;
  readonly data: T;
  readonly createdAt: Date;
  readonly retryCount: number;
}

class SimpleQueue<T> {
  private readonly queue: Job<T>[] = [];
  private processing = false;
  private readonly maxRetries: number;

  constructor(
    private readonly handler: (data: T) => Promise<void>,
    options: { readonly maxRetries?: number } = {}
  ) {
    this.maxRetries = options.maxRetries ?? 3;
  }

  async enqueue(data: T): Promise<string> {
    const job: Job<T> = {
      id: generateId(),
      data,
      createdAt: new Date(),
      retryCount: 0,
    };

    this.queue.push(job);

    if (!this.processing) {
      this.processQueue();
    }

    return job.id;
  }

  private async processQueue(): Promise<void> {
    this.processing = true;

    while (this.queue.length > 0) {
      const job = this.queue.shift()!;

      try {
        await this.handler(job.data);
      } catch (error) {
        logger.error("Job failed", {
          jobId: job.id,
          retryCount: job.retryCount,
          error: error instanceof Error ? error.message : String(error),
        });

        if (job.retryCount < this.maxRetries) {
          // 重新入队（带退避延迟）
          const retryJob: Job<T> = {
            ...job,
            retryCount: job.retryCount + 1,
          };
          this.queue.push(retryJob);
        }
      }
    }

    this.processing = false;
  }
}

// 使用
const indexQueue = new SimpleQueue<{ marketId: string }>(
  async (data) => {
    await indexService.updateSearchIndex(data.marketId);
  },
  { maxRetries: 3 }
);
```

**生产环境**: 使用 Vercel Queues、BullMQ（Redis-backed）或 Inngest 替代内存队列。

## 结构化日志

### Logger 实现

```typescript
type LogLevel = "debug" | "info" | "warn" | "error";

interface LogContext {
  readonly requestId?: string;
  readonly userId?: string;
  readonly method?: string;
  readonly path?: string;
  readonly durationMs?: number;
  readonly statusCode?: number;
  readonly [key: string]: unknown;
}

interface LogEntry {
  readonly timestamp: string;
  readonly level: LogLevel;
  readonly message: string;
  readonly [key: string]: unknown;
}

function createLogEntry(
  level: LogLevel,
  message: string,
  context?: LogContext
): LogEntry {
  return {
    timestamp: new Date().toISOString(),
    level,
    message,
    ...context,
  };
}

const logger = {
  debug(message: string, context?: LogContext): void {
    if (process.env.LOG_LEVEL === "debug") {
      console.debug(JSON.stringify(createLogEntry("debug", message, context)));
    }
  },

  info(message: string, context?: LogContext): void {
    console.info(JSON.stringify(createLogEntry("info", message, context)));
  },

  warn(message: string, context?: LogContext): void {
    console.warn(JSON.stringify(createLogEntry("warn", message, context)));
  },

  error(message: string, error: Error, context?: LogContext): void {
    console.error(
      JSON.stringify(
        createLogEntry("error", message, {
          ...context,
          error: error.message,
          stack: process.env.NODE_ENV === "production" ? undefined : error.stack,
        })
      )
    );
  },
} as const;
```

### 请求日志中间件

```typescript
function requestLogger(req: Request, res: Response, next: NextFunction): void {
  const start = Date.now();

  res.on("finish", () => {
    const durationMs = Date.now() - start;
    const level: LogLevel = res.statusCode >= 500 ? "error"
      : res.statusCode >= 400 ? "warn"
      : "info";

    logger[level]("Request completed", {
      requestId: req.requestId,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      durationMs,
      userAgent: req.headers["user-agent"],
      deviceType: req.headers["x-device-type"] as string,
    });
  });

  next();
}
```

### 日志安全规则

以下字段禁止出现在日志中：

| 禁止字段                          | 替代方案                     |
| --------------------------------- | ---------------------------- |
| 密码、令牌（password, token）     | `[REDACTED]`                 |
| API 密钥（api_key, secret）       | `***${last4}`                |
| PII（email, phone, SSN）          | 哈希值或 `***`               |
| 信用卡号                          | `****${last4}`               |
| 请求体中的敏感字段                | 仅记录 schema 验证后的字段名  |

```typescript
// 敏感字段过滤
const SENSITIVE_FIELDS = new Set([
  "password", "token", "secret", "apiKey", "api_key",
  "authorization", "cookie", "creditCard", "ssn",
]);

function sanitizeContext(context: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(context).map(([key, value]) => {
      if (SENSITIVE_FIELDS.has(key.toLowerCase())) {
        return [key, "[REDACTED]"];
      }
      return [key, value];
    })
  );
}
```

## 健康检查端点

```typescript
// GET /api/health
export async function GET(): Promise<NextResponse> {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
  ]);

  const results = {
    database: checks[0].status === "fulfilled" ? "healthy" : "unhealthy",
    redis: checks[1].status === "fulfilled" ? "healthy" : "unhealthy",
  };

  const allHealthy = Object.values(results).every((s) => s === "healthy");

  return NextResponse.json(
    {
      status: allHealthy ? "healthy" : "degraded",
      checks: results,
      timestamp: new Date().toISOString(),
    },
    { status: allHealthy ? 200 : 503 }
  );
}

async function checkDatabase(): Promise<void> {
  await db.execute(sql`SELECT 1`);
}

async function checkRedis(): Promise<void> {
  await redis.ping();
}
```

## 优雅关闭

```typescript
function setupGracefulShutdown(server: Server, resources: {
  readonly db?: { end: () => Promise<void> };
  readonly redis?: { quit: () => Promise<void> };
}): void {
  let shuttingDown = false;

  async function shutdown(signal: string): Promise<void> {
    if (shuttingDown) return;
    shuttingDown = true;

    logger.info(`Received ${signal}, starting graceful shutdown`);

    // 1. 停止接受新连接
    server.close(() => {
      logger.info("HTTP server closed");
    });

    // 2. 关闭外部资源
    try {
      if (resources.db) await resources.db.end();
      if (resources.redis) await resources.redis.quit();
      logger.info("All resources closed");
    } catch (error) {
      logger.error("Error during shutdown", error as Error);
    }

    process.exit(0);
  }

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));

  // 超时后强制退出
  process.on("SIGTERM", () => {
    setTimeout(() => {
      logger.error("Forced shutdown after timeout", new Error("Shutdown timeout"));
      process.exit(1);
    }, 10_000);
  });
}
```
