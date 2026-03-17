# Error Handling Patterns

## Express 中间件错误处理器

### 全局错误处理器

```typescript
import type { Request, Response, NextFunction } from "express";

function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  _next: NextFunction
): void {
  const requestId = req.headers["x-request-id"] as string ?? generateRequestId();

  if (err instanceof AppError) {
    // Operational 错误: 向用户返回错误信息
    if (!err.isOperational) {
      logger.error("Non-operational error", {
        requestId,
        error: err.message,
        stack: err.stack,
        code: err.code,
        path: req.path,
        method: req.method,
      });
    }

    res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.isOperational ? err.message : "An unexpected error occurred",
        ...(err.details ? { details: err.details } : {}),
      },
      meta: { requestId },
    });
    return;
  }

  // 未知错误（Programmer 错误）
  logger.error("Unhandled error", {
    requestId,
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  res.status(500).json({
    success: false,
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
    meta: { requestId },
  });
}
```

### 注册顺序

```typescript
// 在路由之后、最后注册
app.use("/v1/users", usersRouter);
app.use("/v1/orders", ordersRouter);

// 404 处理器
app.use((_req: Request, _res: Response, next: NextFunction) => {
  next(new NotFoundError("Route"));
});

// 全局错误处理器（必须放在最后）
app.use(errorHandler);
```

## Next.js API Route 错误处理

### App Router（Route Handlers）

```typescript
import { NextRequest, NextResponse } from "next/server";

function withErrorHandling(
  handler: (req: NextRequest) => Promise<NextResponse>
) {
  return async (req: NextRequest): Promise<NextResponse> => {
    const requestId = req.headers.get("x-request-id") ?? generateRequestId();

    try {
      return await handler(req);
    } catch (error) {
      if (error instanceof AppError) {
        if (!error.isOperational) {
          logger.error("Non-operational error", {
            requestId,
            error: error.message,
            stack: error.stack,
            code: error.code,
            path: req.nextUrl.pathname,
            method: req.method,
          });
        }

        return NextResponse.json(
          {
            success: false,
            error: {
              code: error.code,
              message: error.isOperational
                ? error.message
                : "An unexpected error occurred",
              ...(error.details ? { details: error.details } : {}),
            },
            meta: { requestId },
          },
          { status: error.statusCode }
        );
      }

      logger.error("Unhandled error", {
        requestId,
        error: error instanceof Error ? error.message : String(error),
        stack: error instanceof Error ? error.stack : undefined,
        path: req.nextUrl.pathname,
        method: req.method,
      });

      return NextResponse.json(
        {
          success: false,
          error: {
            code: "INTERNAL_ERROR",
            message: "An unexpected error occurred",
          },
          meta: { requestId },
        },
        { status: 500 }
      );
    }
  };
}

// 使用示例
  const users = await userService.findAll();
  return NextResponse.json({
    success: true,
    data: users,
    meta: { requestId: req.headers.get("x-request-id") ?? generateRequestId() },
  });
});
```

## Hono 错误处理

```typescript
import { Hono } from "hono";
import type { Context } from "hono";

const app = new Hono();

// 全局错误处理器
app.onError((err: Error, c: Context) => {
  const requestId = c.req.header("x-request-id") ?? generateRequestId();

  if (err instanceof AppError) {
    if (!err.isOperational) {
      logger.error("Non-operational error", {
        requestId,
        error: err.message,
        stack: err.stack,
        code: err.code,
        path: c.req.path,
        method: c.req.method,
      });
    }

    return c.json(
      {
        success: false,
        error: {
          code: err.code,
          message: err.isOperational
            ? err.message
            : "An unexpected error occurred",
          ...(err.details ? { details: err.details } : {}),
        },
        meta: { requestId },
      },
      err.statusCode as 400 | 401 | 403 | 404 | 409 | 422 | 429 | 500
    );
  }

  logger.error("Unhandled error", {
    requestId,
    error: err.message,
    stack: err.stack,
    path: c.req.path,
    method: c.req.method,
  });

  return c.json(
    {
      success: false,
      error: {
        code: "INTERNAL_ERROR",
        message: "An unexpected error occurred",
      },
      meta: { requestId },
    },
    500
  );
});

// 404 处理器
app.notFound((c: Context) => {
  const requestId = c.req.header("x-request-id") ?? generateRequestId();
  return c.json(
    {
      success: false,
      error: {
        code: "RESOURCE_NOT_FOUND",
        message: "Route not found",
      },
      meta: { requestId },
    },
    404
  );
});
```

## FastAPI 异常处理器

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import uuid

app = FastAPI()


class AppError(Exception):
    def __init__(
        self,
        code: str,
        message: str,
        status_code: int,
        is_operational: bool,
        details: list[dict] | None = None,
    ):
        super().__init__(message)
        self.code = code
        self.message = message
        self.status_code = status_code
        self.is_operational = is_operational
        self.details = details


class ValidationError(AppError):
    def __init__(self, message: str, details: list[dict] | None = None):
        super().__init__(
            code="VALIDATION_ERROR",
            message=message,
            status_code=400,
            is_operational=True,
            details=details,
        )


class NotFoundError(AppError):
    def __init__(self, resource: str, resource_id: str | None = None):
        message = (
            f"{resource} with id '{resource_id}' not found"
            if resource_id
            else f"{resource} not found"
        )
        super().__init__(
            code="RESOURCE_NOT_FOUND",
            message=message,
            status_code=404,
            is_operational=True,
        )


@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    request_id = request.headers.get("x-request-id", f"req_{uuid.uuid4().hex[:12]}")

    if not exc.is_operational:
        logger.error(
            "Non-operational error",
            extra={
                "request_id": request_id,
                "error": exc.message,
                "code": exc.code,
                "path": str(request.url.path),
                "method": request.method,
            },
        )

    error_body: dict = {
        "code": exc.code,
        "message": exc.message if exc.is_operational else "An unexpected error occurred",
    }
    if exc.details:
        error_body["details"] = exc.details

    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "error": error_body,
            "meta": {"requestId": request_id},
        },
    )


@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    request_id = request.headers.get("x-request-id", f"req_{uuid.uuid4().hex[:12]}")

    logger.error(
        "Unhandled error",
        extra={
            "request_id": request_id,
            "error": str(exc),
            "path": str(request.url.path),
            "method": request.method,
        },
    )

    return JSONResponse(
        status_code=500,
        content={
            "success": False,
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred",
            },
            "meta": {"requestId": request_id},
        },
    )
```

## 全局 Uncaught Exception / Unhandled Rejection 处理器

```typescript
// 进程级错误处理器（应用启动时注册一次）

process.on("uncaughtException", (error: Error) => {
  logger.error("Uncaught exception — shutting down", {
    error: error.message,
    stack: error.stack,
  });
  // 优雅关闭: 等待现有请求完成后退出
  gracefulShutdown(1);
});

process.on("unhandledRejection", (reason: unknown) => {
  logger.error("Unhandled promise rejection — shutting down", {
    error: reason instanceof Error ? reason.message : String(reason),
    stack: reason instanceof Error ? reason.stack : undefined,
  });
  // 优雅关闭
  gracefulShutdown(1);
});

function gracefulShutdown(exitCode: number): void {
  // 停止接受新请求
  server.close(() => {
    // 关闭 DB 连接等
    process.exit(exitCode);
  });

  // 超时后强制退出（10秒）
  setTimeout(() => {
    process.exit(exitCode);
  }, 10_000);
}
```

**重要**: `uncaughtException` 后的进程状态不确定。记录日志后，务必终止进程。由 PM2/systemd 等进程管理器负责重启。

## Async 错误包装器（Express 用）

Express 不会自动捕获 async 函数抛出的异常。通过包装器解决：

```typescript
type AsyncHandler = (
  req: Request,
  res: Response,
  next: NextFunction
) => Promise<void>;

function asyncHandler(fn: AsyncHandler) {
  return (req: Request, res: Response, next: NextFunction): void => {
    fn(req, res, next).catch(next);
  };
}

// 使用示例: 各路由无需 try-catch
router.get(
  "/users/:id",
  asyncHandler(async (req, res) => {
    const user = await userService.findById(req.params.id);
    if (!user) {
      throw new NotFoundError("User", req.params.id);
    }
    res.json({
      success: true,
      data: user,
      meta: { requestId: req.requestId },
    });
  })
);
```

## 重试模式: Exponential Backoff with Jitter

针对外部 API 调用暂时性故障的重试策略：

```typescript
interface RetryConfig {
  readonly maxRetries: number;
  readonly baseDelayMs: number;
  readonly maxDelayMs: number;
  readonly retryableStatusCodes: readonly number[];
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30_000,
  retryableStatusCodes: [408, 429, 500, 502, 503, 504],
};

function calculateDelay(attempt: number, config: RetryConfig): number {
  // Exponential backoff: 2^attempt * baseDelay
  const exponentialDelay = Math.pow(2, attempt) * config.baseDelayMs;
  // 应用上限
  const cappedDelay = Math.min(exponentialDelay, config.maxDelayMs);
  // Full jitter: 0 到 cappedDelay 之间的随机值
  return Math.random() * cappedDelay;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  config: RetryConfig = DEFAULT_RETRY_CONFIG
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));

      const isRetryable =
        error instanceof AppError &&
        config.retryableStatusCodes.includes(error.statusCode);

      if (!isRetryable || attempt === config.maxRetries) {
        throw lastError;
      }

      const delay = calculateDelay(attempt, config);
      logger.warn("Retrying after transient error", {
        attempt: attempt + 1,
        maxRetries: config.maxRetries,
        delayMs: Math.round(delay),
        error: lastError.message,
      });

      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

// 使用示例
const result = await withRetry(
  () => externalApiClient.fetchData(params),
  {
    maxRetries: 3,
    baseDelayMs: 500,
    maxDelayMs: 10_000,
    retryableStatusCodes: [429, 500, 502, 503, 504],
  }
);
```

## 熔断器模式

连续失败时切断对外部服务的调用，防止系统整体级联故障：

```typescript
type CircuitState = "CLOSED" | "OPEN" | "HALF_OPEN";

interface CircuitBreakerConfig {
  readonly failureThreshold: number;  // 切换到 OPEN 的连续失败次数
  readonly resetTimeoutMs: number;    // 切换到 HALF_OPEN 前的等待时间
  readonly halfOpenMaxCalls: number;  // HALF_OPEN 状态下允许的调用数
}

interface CircuitBreakerState {
  readonly state: CircuitState;
  readonly failureCount: number;
  readonly lastFailureTime: number | null;
  readonly halfOpenCalls: number;
}

const DEFAULT_CIRCUIT_CONFIG: CircuitBreakerConfig = {
  failureThreshold: 5,
  resetTimeoutMs: 30_000,
  halfOpenMaxCalls: 1,
};

function createCircuitBreaker(config: CircuitBreakerConfig = DEFAULT_CIRCUIT_CONFIG) {
  let currentState: CircuitBreakerState = {
    state: "CLOSED",
    failureCount: 0,
    lastFailureTime: null,
    halfOpenCalls: 0,
  };

  function getNextState(
    prevState: CircuitBreakerState,
    event: "success" | "failure"
  ): CircuitBreakerState {
    if (event === "success") {
      return {
        state: "CLOSED",
        failureCount: 0,
        lastFailureTime: null,
        halfOpenCalls: 0,
      };
    }

    // event === "failure"
    const newFailureCount = prevState.failureCount + 1;
    if (newFailureCount >= config.failureThreshold) {
      return {
        state: "OPEN",
        failureCount: newFailureCount,
        lastFailureTime: Date.now(),
        halfOpenCalls: 0,
      };
    }

    return {
      ...prevState,
      failureCount: newFailureCount,
      lastFailureTime: Date.now(),
    };
  }

  async function execute<T>(fn: () => Promise<T>): Promise<T> {
    // 检查 OPEN 状态
    if (currentState.state === "OPEN") {
      const elapsed = Date.now() - (currentState.lastFailureTime ?? 0);
      if (elapsed >= config.resetTimeoutMs) {
        currentState = { ...currentState, state: "HALF_OPEN", halfOpenCalls: 0 };
      } else {
        throw new AppError({
          code: "SERVICE_UNAVAILABLE",
          message: "Circuit breaker is open — service temporarily unavailable",
          statusCode: 503,
          isOperational: true,
        });
      }
    }

    // 检查 HALF_OPEN 状态的调用限制
    if (currentState.state === "HALF_OPEN") {
      if (currentState.halfOpenCalls >= config.halfOpenMaxCalls) {
        throw new AppError({
          code: "SERVICE_UNAVAILABLE",
          message: "Circuit breaker is half-open — max concurrent calls reached",
          statusCode: 503,
          isOperational: true,
        });
      }
      currentState = {
        ...currentState,
        halfOpenCalls: currentState.halfOpenCalls + 1,
      };
    }

    try {
      const result = await fn();
      currentState = getNextState(currentState, "success");
      return result;
    } catch (error) {
      currentState = getNextState(currentState, "failure");
      throw error;
    }
  }

  return { execute };
}

// 使用示例
const paymentCircuit = createCircuitBreaker({
  failureThreshold: 3,
  resetTimeoutMs: 60_000,
  halfOpenMaxCalls: 1,
});

const result = await paymentCircuit.execute(() =>
  paymentService.charge(amount)
);
```

## Zod 验证错误转换

将 Zod 的 `ZodError` 转换为统一错误响应格式：

```typescript
import { ZodError, type ZodIssue } from "zod";

function zodIssueToDetail(issue: ZodIssue): ErrorDetail {
  return {
    field: issue.path.join("."),
    message: issue.message,
    code: issue.code,
  };
}

function handleZodError(error: ZodError): ValidationError {
  const details = error.issues.map(zodIssueToDetail);
  return new ValidationError("Request validation failed", details);
}

// 在中间件中使用（Express）
function validateBody<T>(schema: z.ZodSchema<T>) {
  return asyncHandler(async (req: Request, _res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        throw handleZodError(error);
      }
      throw error;
    }
  });
}

// 在路由中使用
router.post(
  "/users",
  validateBody(createUserSchema),
  asyncHandler(async (req, res) => {
    const user = await userService.create(req.body);
    res.status(201).json({
      success: true,
      data: user,
      meta: { requestId: req.requestId },
    });
  })
);
```

## 数据库错误映射

将 DB 特定错误转换为 AppError 子类：

```typescript
interface DatabaseErrorMap {
  readonly constraintCode: string;
  readonly toAppError: (error: Error) => AppError;
}

// PostgreSQL 错误码映射
const PG_ERROR_MAP: Record<string, (error: Error) => AppError> = {
  // 23505: unique_violation
  "23505": (error) =>
    new ConflictError(
      extractConstraintMessage(error) ?? "Resource already exists",
      "DUPLICATE_RESOURCE"
    ),
  // 23503: foreign_key_violation
  "23503": (error) =>
    new ValidationError(
      extractConstraintMessage(error) ?? "Referenced resource does not exist",
      [{ field: "unknown", message: "Foreign key constraint failed", code: "foreign_key" }]
    ),
  // 23502: not_null_violation
  "23502": (error) =>
    new ValidationError(
      "Required field is missing",
      [{ field: extractColumnName(error) ?? "unknown", message: "Cannot be null", code: "not_null" }]
    ),
  // 23514: check_violation
  "23514": (error) =>
    new ValidationError(
      extractConstraintMessage(error) ?? "Value constraint violated"
    ),
};

function mapDatabaseError(error: unknown): AppError {
  if (isPgError(error) && error.code in PG_ERROR_MAP) {
    return PG_ERROR_MAP[error.code](error);
  }
  // 未知 DB 错误转换为 InternalError
  return new InternalError("Database operation failed");
}

// 在仓库层使用
async function createUser(data: CreateUserInput): Promise<User> {
  try {
    return await db.insert(users).values(data).returning();
  } catch (error) {
    throw mapDatabaseError(error);
  }
}
```

## X-Device-Type 验证

对 api-design 技能要求的必需头部进行验证：

```typescript
const VALID_DEVICE_TYPES = ["web", "app", "desktop"] as const;

function validateDeviceType(
  req: Request,
  _res: Response,
  next: NextFunction
): void {
  const deviceType = req.headers["x-device-type"] as string | undefined;

  if (!deviceType) {
    throw new ValidationError("X-Device-Type header is required", [
      { field: "X-Device-Type", message: "Header is missing", code: "missing_header" },
    ]);
  }

  if (!VALID_DEVICE_TYPES.includes(deviceType as typeof VALID_DEVICE_TYPES[number])) {
    throw new ValidationError(
      `X-Device-Type must be one of: ${VALID_DEVICE_TYPES.join(", ")}`,
      [{ field: "X-Device-Type", message: "Invalid value", code: "invalid_header" }]
    );
  }

  next();
}
```
