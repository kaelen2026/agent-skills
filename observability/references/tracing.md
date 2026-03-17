---
name: tracing
version: 1.0.0
description: 分布式追踪与健康检查 — OpenTelemetry 集成、链路追踪、健康检查端点与优雅降级
last_updated: 2026-03-17
---

# 分布式追踪与健康检查

## 分布式追踪核心概念

### Trace（追踪）

一个 Trace 代表一个完整请求在分布式系统中的生命周期。从用户发起请求到收到响应，经过的所有服务和操作构成一条 Trace。

### Span（跨度）

Span 是 Trace 中的最小工作单元。每个服务中的一次操作（HTTP 调用、数据库查询、缓存访问）对应一个 Span。Span 之间存在父子关系，形成树状结构。

```
Trace: 用户下单请求
├── Span: API Gateway (120ms)
│   ├── Span: Auth Service - 验证令牌 (15ms)
│   ├── Span: Order Service - 创建订单 (80ms)
│   │   ├── Span: Database - INSERT order (10ms)
│   │   └── Span: Payment Service - 处理支付 (55ms)
│   │       └── Span: External API - 银行接口 (40ms)
│   └── Span: Notification Service - 发送确认 (20ms)
```

### Context Propagation（上下文传播）

上下文传播确保 Trace ID 和 Span ID 在服务间正确传递。通过 HTTP 头（如 `traceparent`）实现跨服务关联。

## OpenTelemetry 配置

OpenTelemetry 是分布式追踪的行业标准。以下为 Node.js 环境配置：

### 安装依赖

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

### 初始化配置

此文件必须在应用程序入口处最先加载：

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
  ATTR_DEPLOYMENT_ENVIRONMENT_NAME,
} from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: new Resource({
    [ATTR_SERVICE_NAME]: process.env.SERVICE_NAME || "my-service",
    [ATTR_SERVICE_VERSION]: process.env.SERVICE_VERSION || "1.0.0",
    [ATTR_DEPLOYMENT_ENVIRONMENT_NAME]: process.env.NODE_ENV || "development",
  }),
  traceExporter: new OTLPTraceExporter({
    url:
      process.env.OTEL_EXPORTER_OTLP_ENDPOINT ||
      "http://localhost:4318/v1/traces",
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-fs": { enabled: false },
    }),
  ],
});

sdk.start();

process.on("SIGTERM", () => {
  sdk
    .shutdown()
    .then(() => process.exit(0))
    .catch(() => process.exit(1));
});
```

### 手动创建 Span

自动检测覆盖大多数场景，但业务逻辑需要手动创建 Span：

```typescript
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("order-service");

async function processOrder(order) {
  return tracer.startActiveSpan("processOrder", async (span) => {
    try {
      span.setAttribute("order.id", order.id);
      span.setAttribute("order.total", order.total);
      span.setAttribute("order.item_count", order.items.length);

      const result = await executeOrderLogic(order);

      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## Span 属性规范

遵循 OpenTelemetry 语义约定，确保 Span 属性统一：

### HTTP 相关

| 属性 | 说明 | 示例 |
|------|------|------|
| `http.method` | HTTP 方法 | `GET`、`POST` |
| `http.route` | 路由模板 | `/api/users/:id` |
| `http.status_code` | 响应状态码 | `200`、`500` |
| `http.url` | 完整请求 URL | `https://api.example.com/users/123` |

### 数据库相关

| 属性 | 说明 | 示例 |
|------|------|------|
| `db.system` | 数据库类型 | `postgresql`、`redis` |
| `db.statement` | 查询语句（需脱敏） | `SELECT * FROM users WHERE id = ?` |
| `db.operation` | 操作类型 | `SELECT`、`INSERT` |
| `db.name` | 数据库名称 | `orders_db` |

### 业务相关

| 属性 | 说明 | 示例 |
|------|------|------|
| `user.id` | 用户标识 | `user-456` |
| `order.id` | 订单标识 | `order-789` |
| `payment.method` | 支付方式 | `credit_card` |

## requestId 传播

`requestId` 通过 `X-Request-Id` 头在服务间传播，贯穿整个请求链路：

```typescript
import { randomUUID } from "node:crypto";

function propagateRequestId(req, res, next) {
  const requestId = req.headers["x-request-id"] || randomUUID();

  req.requestId = requestId;
  res.setHeader("X-Request-Id", requestId);

  next();
}
```

### 下游调用时传递

```typescript
async function callDownstreamService(requestId, path, body) {
  const response = await fetch(`https://downstream-service${path}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Request-Id": requestId,
    },
    body: JSON.stringify(body),
  });

  return response.json();
}
```

## 健康检查端点

### 存活检查（Liveness）

`/health` — 表示服务进程正在运行。探测失败时，编排系统（如 Kubernetes）会重启容器。

**规则**：仅检查进程本身是否存活，不检查外部依赖。

```typescript
app.get("/health", (req, res) => {
  res.status(200).json({
    status: "healthy",
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});
```

### 就绪检查（Readiness）

`/ready` — 表示服务可以接受流量。探测失败时，负载均衡器停止向该实例分发请求。

**规则**：检查所有关键依赖的连接状态。

```typescript
app.get("/ready", async (req, res) => {
  const checks = await runDependencyChecks();

  const isReady = Object.values(checks).every(
    (check) => check === "ok" || check === "degraded"
  );

  const status = isReady ? "healthy" : "unhealthy";
  const statusCode = isReady ? 200 : 503;

  res.status(statusCode).json({
    status,
    checks,
    timestamp: new Date().toISOString(),
  });
});
```

### 健康检查响应格式

```json
{
  "status": "healthy",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "external_api": "degraded"
  }
}
```

各检查项状态值：

| 状态 | 含义 |
|------|------|
| `ok` | 依赖正常 |
| `degraded` | 依赖可用但性能下降 |
| `unhealthy` | 依赖不可用 |

### 依赖检查实现

```typescript
async function runDependencyChecks() {
  const checks = {};

  checks.database = await checkWithTimeout(
    () => db.query("SELECT 1"),
    3000,
    "database"
  );

  checks.redis = await checkWithTimeout(
    () => redis.ping(),
    2000,
    "redis"
  );

  checks.external_api = await checkWithTimeout(
    () => fetch("https://api.example.com/health"),
    5000,
    "external_api"
  );

  return checks;
}

async function checkWithTimeout(checkFn, timeoutMs, name) {
  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);

    await checkFn();
    clearTimeout(timeout);
    return "ok";
  } catch (error) {
    console.error(`依赖检查失败 [${name}]:`, error.message);
    return "unhealthy";
  }
}
```

## 依赖健康监控

持续监控所有外部依赖的健康状况，而非仅在健康检查时探测：

```typescript
class DependencyMonitor {
  constructor() {
    this.statuses = new Map();
  }

  getStatus(name) {
    return this.statuses.get(name) || "unknown";
  }

  getAllStatuses() {
    return Object.fromEntries(this.statuses);
  }

  startMonitoring(name, checkFn, intervalMs = 30000) {
    const runCheck = async () => {
      const status = await checkWithTimeout(checkFn, 5000, name);
      this.statuses.set(name, status);
    };

    runCheck();
    setInterval(runCheck, intervalMs);
  }
}

const monitor = new DependencyMonitor();
monitor.startMonitoring("database", () => db.query("SELECT 1"));
monitor.startMonitoring("redis", () => redis.ping());
monitor.startMonitoring("payment-api", () =>
  fetch("https://payment.example.com/health")
);
```

## 优雅降级模式

当依赖出现故障时，服务应优雅降级而非完全失败。

### 断路器模式

```typescript
class CircuitBreaker {
  constructor(options) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeoutMs = options.resetTimeoutMs || 30000;
    this.state = "CLOSED";
    this.failureCount = 0;
    this.lastFailureTime = null;
  }

  async execute(fn, fallbackFn) {
    if (this.state === "OPEN") {
      if (Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.state = "HALF_OPEN";
      } else {
        return fallbackFn();
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      return fallbackFn();
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = "CLOSED";
  }

  onFailure() {
    this.failureCount += 1;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = "OPEN";
    }
  }
}
```

### 降级策略

| 策略 | 适用场景 | 示例 |
|------|----------|------|
| **返回缓存数据** | 数据更新不频繁的查询 | 商品列表使用缓存版本 |
| **返回默认值** | 个性化推荐等非关键功能 | 推荐服务不可用时展示热门商品 |
| **功能关闭** | 非核心功能 | 评论服务不可用时隐藏评论区 |
| **队列化处理** | 可异步完成的操作 | 通知服务不可用时将消息入队稍后重试 |

### 降级实现示例

```typescript
const recommendationBreaker = new CircuitBreaker({
  failureThreshold: 3,
  resetTimeoutMs: 60000,
});

async function getRecommendations(userId) {
  return recommendationBreaker.execute(
    () => recommendationService.getPersonalized(userId),
    () => recommendationService.getPopularItems()
  );
}
```
