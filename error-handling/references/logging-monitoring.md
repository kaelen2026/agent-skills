# Error Logging & Monitoring

## 结构化日志格式

所有日志条目以 JSON 格式输出。既可被机器解析，也便于人类阅读：

```typescript
interface LogEntry {
  readonly timestamp: string;    // ISO 8601 格式
  readonly level: LogLevel;      // "error" | "warn" | "info" | "debug"
  readonly message: string;      // 人类可读的消息
  readonly requestId?: string;   // 请求追踪用 ID
  readonly service: string;      // 服务名称
  readonly context?: LogContext; // 附加上下文
}

interface LogContext {
  readonly userId?: string;
  readonly deviceType?: string;  // X-Device-Type 头部值
  readonly path?: string;
  readonly method?: string;
  readonly statusCode?: number;
  readonly durationMs?: number;
  readonly error?: ErrorLogContext;
}

interface ErrorLogContext {
  readonly code: string;         // UPPER_SNAKE_CASE 错误码
  readonly message: string;
  readonly stack?: string;       // 仅记录在生产日志中（不包含在响应中）
  readonly isOperational: boolean;
}
```

### 日志输出示例

#### 错误日志

```json
{
  "timestamp": "2026-03-17T10:30:45.123Z",
  "level": "error",
  "message": "Database connection failed",
  "requestId": "req_abc123",
  "service": "user-api",
  "context": {
    "userId": "usr_def456",
    "deviceType": "web",
    "path": "/v1/users",
    "method": "GET",
    "error": {
      "code": "INTERNAL_ERROR",
      "message": "Connection to database timed out after 5000ms",
      "stack": "Error: Connection to database timed out...\n    at ...",
      "isOperational": false
    }
  }
}
```

#### 请求完成日志

```json
{
  "timestamp": "2026-03-17T10:30:45.456Z",
  "level": "info",
  "message": "Request completed",
  "requestId": "req_ghi789",
  "service": "user-api",
  "context": {
    "userId": "usr_jkl012",
    "deviceType": "app",
    "path": "/v1/users/usr_jkl012",
    "method": "GET",
    "statusCode": 200,
    "durationMs": 42
  }
}
```

## 日志级别

| Level   | 用途                                          | 示例                                       |
| ------- | --------------------------------------------- | ------------------------------------------ |
| `error` | 需要立即响应的故障                            | DB 连接失败、未捕获异常、5xx 响应           |
| `warn`  | 需要关注但服务可继续运行                      | 发生重试、接近频率限制、使用已弃用 API      |
| `info`  | 正常运维事件                                  | 请求完成、用户创建、批处理完成             |
| `debug` | 开发/排障用详细信息                           | 查询内容、外部 API 响应、缓存状态           |

### 各环境日志级别

| 环境        | 最低级别   | 说明                            |
| ----------- | ---------- | ------------------------------- |
| development | `debug`    | 输出所有级别                    |
| staging     | `info`     | 输出 debug 以外                 |
| production  | `info`     | 输出 debug 以外（推荐 warn 以上）|

## 日志中应包含 / 不应包含的内容

### 必须包含在日志中的内容

| 项目          | 说明                                             |
| ------------- | ------------------------------------------------ |
| requestId     | 跨请求追踪必需                                   |
| timestamp     | ISO 8601 格式。带时区（推荐 UTC）                |
| level         | error, warn, info, debug                         |
| message       | 简洁且具有描述性的消息                           |
| error.code    | UPPER_SNAKE_CASE 错误码                          |
| error.stack   | 堆栈跟踪（仅 error 级别）                       |
| path          | 请求路径                                         |
| method        | HTTP 方法                                        |
| statusCode    | 响应状态码                                       |
| durationMs    | 响应时间（毫秒）                                 |
| userId        | 已认证用户 ID（如存在）                          |
| deviceType    | X-Device-Type 头部值                             |
| service       | 服务/应用程序名称                                |

### 绝对不可包含在日志中的内容

| 项目                 | 原因                                         |
| -------------------- | -------------------------------------------- |
| 密码                 | 明文密码在任何环境下都禁止记录               |
| 访问令牌             | Bearer 令牌、JWT 的完整值                    |
| API 密钥             | 外部服务的认证密钥                           |
| 信用卡号             | 违反 PCI DSS                                 |
| 社会保障号           | PII（个人身份信息）                          |
| 会话令牌             | 存在会话劫持风险                             |
| 完整请求体           | 可能包含 PII 或敏感数据                      |
| DB 连接字符串        | 包含认证信息                                 |
| 环境变量值           | 可能包含密钥                                 |

### 脱敏实现

```typescript
const SENSITIVE_FIELDS = [
  "password",
  "token",
  "authorization",
  "apiKey",
  "secret",
  "creditCard",
  "ssn",
  "sessionId",
] as const;

function sanitizeLogData(data: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(data).map(([key, value]) => {
      const isSensitive = SENSITIVE_FIELDS.some(
        (field) => key.toLowerCase().includes(field.toLowerCase())
      );
      if (isSensitive) {
        return [key, "[REDACTED]"];
      }
      if (typeof value === "object" && value !== null && !Array.isArray(value)) {
        return [key, sanitizeLogData(value as Record<string, unknown>)];
      }
      return [key, value];
    })
  );
}
```

## 错误上下文 enrichment

将请求作用域的信息自动附加到日志条目：

### Express 中间件

```typescript
import { randomUUID } from "node:crypto";

function requestContext(req: Request, _res: Response, next: NextFunction): void {
  const requestId = (req.headers["x-request-id"] as string) ?? `req_${randomUUID().replace(/-/g, "").slice(0, 12)}`;
  const deviceType = req.headers["x-device-type"] as string;
  const userId = req.user?.id; // 由认证中间件设置

  // 将上下文附加到请求对象
  req.requestId = requestId;
  req.logContext = {
    requestId,
    deviceType,
    userId,
    path: req.path,
    method: req.method,
  };

  // 在响应头中包含 requestId
  _res.setHeader("x-request-id", requestId);

  next();
}
```

### 通过 AsyncLocalStorage 进行上下文传播

从中间件到服务层自动传播 requestId：

```typescript
import { AsyncLocalStorage } from "node:async_hooks";

interface RequestStore {
  readonly requestId: string;
  readonly userId?: string;
  readonly deviceType?: string;
  readonly path: string;
  readonly method: string;
}

const asyncLocalStorage = new AsyncLocalStorage<RequestStore>();

// 在中间件中设置 store
function contextMiddleware(req: Request, _res: Response, next: NextFunction): void {
  const store: RequestStore = {
    requestId: req.requestId,
    userId: req.user?.id,
    deviceType: req.headers["x-device-type"] as string,
    path: req.path,
    method: req.method,
  };

  asyncLocalStorage.run(store, () => next());
}

// 在日志工具中引用 store
function getRequestContext(): Partial<RequestStore> {
  return asyncLocalStorage.getStore() ?? {};
}

// 在 logger 中自动附加
const logger = {
  error(message: string, extra?: Record<string, unknown>): void {
    const context = getRequestContext();
    console.error(
      JSON.stringify({
        timestamp: new Date().toISOString(),
        level: "error",
        message,
        ...context,
        ...sanitizeLogData(extra ?? {}),
      })
    );
  },
  warn(message: string, extra?: Record<string, unknown>): void {
    const context = getRequestContext();
    console.warn(
      JSON.stringify({
        timestamp: new Date().toISOString(),
        level: "warn",
        message,
        ...context,
        ...sanitizeLogData(extra ?? {}),
      })
    );
  },
  info(message: string, extra?: Record<string, unknown>): void {
    const context = getRequestContext();
    console.info(
      JSON.stringify({
        timestamp: new Date().toISOString(),
        level: "info",
        message,
        ...context,
        ...sanitizeLogData(extra ?? {}),
      })
    );
  },
  debug(message: string, extra?: Record<string, unknown>): void {
    const context = getRequestContext();
    console.debug(
      JSON.stringify({
        timestamp: new Date().toISOString(),
        level: "debug",
        message,
        ...context,
        ...sanitizeLogData(extra ?? {}),
      })
    );
  },
};
```

## 告警阈值

### 基于错误率的告警

| 指标                       | 阈值            | 操作                               |
| -------------------------- | --------------- | ---------------------------------- |
| 5xx 错误率                 | > 1% / 5分钟    | WARNING 告警                       |
| 5xx 错误率                 | > 5% / 5分钟    | CRITICAL 告警 + 值班通知           |
| 5xx 错误突增               | 与前期相比 3倍  | WARNING 告警                       |
| 特定错误码连续发生         | > 10次 / 1分钟  | WARNING 告警                       |
| uncaughtException 发生     | > 0             | CRITICAL 告警（即时）              |
| unhandledRejection 发生    | > 0             | CRITICAL 告警（即时）              |
| 响应时间 P99               | > 5秒           | WARNING 告警                       |
| 响应时间 P99               | > 10秒          | CRITICAL 告警                      |
| 熔断器 OPEN                | 切换时          | WARNING 告警                       |

### 错误预算

```
月度允许错误率 = 1 - SLA 目标
例: SLA 99.9% → 错误预算 0.1%（月度约 43分钟停机时间）
```

当错误预算消耗率超过 80% 时，冻结功能发布，专注于可靠性改善。

## 通过 requestId 进行跨服务关联

### requestId 的生成与传播

```
Client → API Gateway → Service A → Service B → Database
         req_abc123    req_abc123   req_abc123
```

1. 在 API Gateway（或第一个服务）生成 `requestId`
2. 向下游服务请求时通过头部传播: `X-Request-Id: req_abc123`
3. 所有日志条目中包含 `requestId`
4. 错误响应的 `meta.requestId` 中也包含

### 关联日志搜索

```bash
# 按时间顺序搜索特定请求的所有日志
# CloudWatch Logs Insights
fields @timestamp, @message
| filter requestId = "req_abc123"
| sort @timestamp asc

# Elasticsearch / Kibana
GET /logs-*/_search
{
  "query": {
    "term": { "requestId.keyword": "req_abc123" }
  },
  "sort": [{ "@timestamp": "asc" }]
}
```

### requestId 格式

```typescript
import { randomUUID } from "node:crypto";

function generateRequestId(): string {
  // "req_" 前缀 + 12字符 hex
  return `req_${randomUUID().replace(/-/g, "").slice(0, 12)}`;
}

// 示例: req_a1b2c3d4e5f6
```

## 请求日志中间件

记录所有请求的开始和完成：

```typescript
function requestLogger(req: Request, res: Response, next: NextFunction): void {
  const startTime = Date.now();

  logger.info("Request started", {
    path: req.path,
    method: req.method,
    query: sanitizeLogData(req.query as Record<string, unknown>),
  });

  res.on("finish", () => {
    const durationMs = Date.now() - startTime;
    const level = res.statusCode >= 500 ? "error" : res.statusCode >= 400 ? "warn" : "info";

    logger[level]("Request completed", {
      statusCode: res.statusCode,
      durationMs,
    });
  });

  next();
}
```

## 监控仪表板推荐指标

| 类别         | 指标                              | 可视化           |
| ------------ | --------------------------------- | ---------------- |
| 错误率       | 5xx / 4xx 错误率 (%)             | 时序图           |
| 错误分布     | 按错误码分类的发生次数            | 堆叠柱状图       |
| 响应         | P50, P95, P99 响应时间            | 百分位线         |
| 吞吐量       | 请求/秒                          | 时序图           |
| 可用性       | 熔断器状态                        | 状态面板         |
| 重试         | 重试发生率                        | 时序图           |
| 进程         | uncaught / unhandled 发生数       | 计数器面板       |
