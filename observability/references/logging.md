---
name: logging
version: 1.0.0
description: 结构化日志最佳实践 — 格式、级别、中间件、敏感数据过滤与日志聚合
last_updated: 2026-03-17
---

# 结构化日志

## 核心原则

日志是可观测性的基础。所有日志必须采用结构化 JSON 格式，便于机器解析和聚合查询。

## JSON 日志格式

每条日志必须包含以下标准字段：

```json
{
  "timestamp": "2026-03-17T10:30:00.000Z",
  "level": "info",
  "message": "用户登录成功",
  "requestId": "req-abc-123",
  "service": "auth-service",
  "context": {
    "userId": "user-456",
    "method": "POST",
    "path": "/api/auth/login"
  }
}
```

| 字段 | 说明 | 是否必需 |
|------|------|----------|
| `timestamp` | ISO 8601 格式的时间戳 | 必需 |
| `level` | 日志级别 | 必需 |
| `message` | 可读的事件描述 | 必需 |
| `requestId` | 请求关联 ID，用于跨服务追踪 | 必需 |
| `service` | 产生日志的服务名称 | 必需 |
| `context` | 业务上下文信息 | 推荐 |

## 日志级别

严格遵循以下级别定义，避免滥用：

| 级别 | 何时使用 | 示例 |
|------|----------|------|
| `error` | 需要立即处理的故障，影响用户功能 | 数据库连接失败、支付处理异常 |
| `warn` | 潜在问题，暂未影响用户但需要关注 | 重试成功、缓存未命中率升高、接近速率限制 |
| `info` | 正常业务事件，用于审计和运营 | 用户登录、订单创建、部署完成 |
| `debug` | 开发调试信息，生产环境默认关闭 | 函数入参、SQL 查询详情、缓存键 |

**关键规则**：

- 生产环境默认级别为 `info`
- `error` 级别日志必须触发告警
- `debug` 仅在排查问题时临时开启

## Logger 配置

推荐使用 pino，性能优于 winston，原生支持 JSON 输出。

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: [
      "password",
      "token",
      "authorization",
      "cookie",
      "*.password",
      "*.token",
      "*.ssn",
      "*.creditCard",
    ],
    censor: "[REDACTED]",
  },
});

export { logger };
```

### 创建子 Logger

为每个服务模块创建带上下文的子 Logger：

```typescript
const authLogger = logger.child({ service: "auth-service" });
const paymentLogger = logger.child({ service: "payment-service" });

authLogger.info({ userId: "user-456" }, "用户登录成功");
```

## 请求日志中间件

记录每个 HTTP 请求的关键信息：method、path、status、duration、requestId。

```typescript
import { randomUUID } from "node:crypto";

function requestLoggingMiddleware(req, res, next) {
  const requestId = req.headers["x-request-id"] || randomUUID();
  const startTime = Date.now();

  req.requestId = requestId;
  res.setHeader("X-Request-Id", requestId);

  const requestLogger = logger.child({ requestId });
  req.log = requestLogger;

  res.on("finish", () => {
    const duration = Date.now() - startTime;
    const logData = {
      method: req.method,
      path: req.originalUrl,
      statusCode: res.statusCode,
      duration,
      requestId,
      userAgent: req.headers["user-agent"],
    };

    if (res.statusCode >= 500) {
      requestLogger.error(logData, "请求处理失败");
    } else if (res.statusCode >= 400) {
      requestLogger.warn(logData, "客户端错误");
    } else {
      requestLogger.info(logData, "请求处理完成");
    }
  });

  next();
}
```

## 敏感数据过滤

**绝对禁止**记录以下信息：

- 密码和密码哈希
- API 令牌和密钥
- 个人身份信息（PII）：身份证号、信用卡号、社保号
- Session ID 和 Cookie 值
- Authorization 请求头的完整内容

通过 pino 的 `redact` 配置自动过滤（见上方 Logger 配置）。对于动态字段，使用序列化器：

```typescript
const logger = pino({
  serializers: {
    req(req) {
      return {
        method: req.method,
        url: req.url,
        headers: {
          "content-type": req.headers["content-type"],
          "user-agent": req.headers["user-agent"],
          // 不包含 authorization、cookie 等敏感头
        },
      };
    },
  },
});
```

## 日志聚合

根据基础设施选择合适的聚合方案：

| 工具 | 适用场景 | 特点 |
|------|----------|------|
| CloudWatch Logs | AWS 原生环境 | 与 AWS 服务深度集成，按量计费 |
| Datadog | 全栈可观测性 | 日志、指标、追踪统一平台，成本较高 |
| ELK Stack | 自建方案 | 灵活可控，需要运维投入 |

### 聚合配置建议

- 使用统一的日志管道（如 Fluent Bit）收集所有服务日志
- 按服务名称和环境建立索引
- 配置基于 `requestId` 的关联查询

## 日志轮转与保留策略

| 环境 | 保留周期 | 说明 |
|------|----------|------|
| 生产环境 | 90 天 | 满足合规和事故回溯需求 |
| 预发环境 | 30 天 | 用于发布前验证 |
| 开发环境 | 7 天 | 仅用于开发调试 |

**轮转规则**：

- 单文件大小上限：100MB
- 按天轮转
- 压缩归档旧日志（gzip）
- 超过保留周期自动删除

## 跨服务关联

`requestId` 是跨服务日志关联的核心。确保：

1. 入口服务生成 `requestId`（如果请求头中没有）
2. 每次服务间调用通过 `X-Request-Id` 头传递
3. 所有日志条目包含 `requestId` 字段
4. 日志查询平台支持按 `requestId` 过滤

```
用户请求 → API Gateway (requestId: req-abc-123)
  → Auth Service (requestId: req-abc-123)
  → Order Service (requestId: req-abc-123)
    → Payment Service (requestId: req-abc-123)
```

通过 `requestId` 可以在日志平台中一次性查询完整的请求链路。
