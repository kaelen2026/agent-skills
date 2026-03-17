# REST API Design Rules

## URL 设计

### 资源命名

- 使用复数名词: `/users`, `/orders`, `/products`
- 使用 kebab-case: `/order-items`, `/user-profiles`
- 避免动词: `/users` 而非 `/getUsers`
- 嵌套资源最多 2 层: `/users/:id/orders`（不要 `/users/:id/orders/:oid/items/:iid`）

### 版本策略

```
/v1/users          # URL prefix（推荐）
/v2/users          # 大版本变更时递增
```

## HTTP 方法语义

| Method | 语义     | 幂等 | 安全 | 请求体 | 典型状态码       |
| ------ | -------- | ---- | ---- | ------ | ---------------- |
| GET    | 读取     | Yes  | Yes  | No     | 200              |
| POST   | 创建     | No   | No   | Yes    | 201              |
| PUT    | 全量替换 | Yes  | No   | Yes    | 200              |
| PATCH  | 部分更新 | No   | No   | Yes    | 200              |
| DELETE | 删除     | Yes  | No   | No     | 204              |

## HTTP 状态码

### 成功

| Code | 用途                  |
| ---- | --------------------- |
| 200  | 成功（有返回体）      |
| 201  | 创建成功              |
| 204  | 成功（无返回体）      |

### 客户端错误

| Code | 用途                        |
| ---- | --------------------------- |
| 400  | 请求体验证失败              |
| 401  | 未认证（缺少/无效 token）   |
| 403  | 已认证但无权限              |
| 404  | 资源不存在                  |
| 409  | 冲突（如重复创建）          |
| 422  | 语义错误（业务规则不满足）  |
| 429  | 请求频率超限                |

### 服务端错误

| Code | 用途           |
| ---- | -------------- |
| 500  | 服务器内部错误 |
| 503  | 服务不可用     |

## 统一响应格式

所有 API 响应使用统一信封（envelope），属性使用 **camelCase**：

```typescript
interface ApiResponse<T = unknown> {
  success: boolean;       // 始终存在
  data?: T;               // 成功时的业务数据
  meta?: Meta;            // 分页、请求追踪等元信息
  error?: ApiError;       // 失败时的错误信息
}

interface Meta {
  requestId: string;      // 始终存在，用于追踪和调试
  cursor?: string;        // 分页游标
  hasMore?: boolean;      // 是否有更多数据
  total?: number;         // 总数（offset 分页）
  offset?: number;
  limit?: number;
}
```

### 成功响应

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Alice",
    "email": "alice@example.com",
    "createdAt": "2026-01-01T00:00:00Z"
  },
  "meta": {
    "requestId": "req_abc123"
  }
}
```

### 列表响应（带分页）

```json
{
  "success": true,
  "data": [
    { "id": "uuid-1", "name": "Alice" },
    { "id": "uuid-2", "name": "Bob" }
  ],
  "meta": {
    "requestId": "req_abc123",
    "cursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

### 错误响应

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email format is invalid",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "invalid_string"
      }
    ]
  },
  "meta": {
    "requestId": "req_abc123"
  }
}
```

### 规则

- `success` 始终存在，`true` 或 `false`
- `meta.requestId` 始终存在，成功和失败均携带
- 成功时只有 `data`（和 `meta`），不含 `error`
- 失败时只有 `error`（和 `meta`），不含 `data`
- `meta` 用于分页、请求追踪等非业务数据
- 204 No Content 无响应体，是唯一例外
- **所有响应属性使用 camelCase**（`createdAt` 而非 `created_at`）

### 错误码命名

- 使用 UPPER_SNAKE_CASE（仅错误码本身，非属性名）
- 具有描述性: `RESOURCE_NOT_FOUND`, `DUPLICATE_EMAIL`, `RATE_LIMIT_EXCEEDED`
- 不暴露内部实现细节

## 分页

### Cursor-based（推荐）

```
GET /v1/users?cursor=eyJpZCI6MTAwfQ&limit=20

Response:
{
  "success": true,
  "data": [...],
  "meta": {
    "requestId": "req_abc123",
    "cursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

### Offset-based

```
GET /v1/users?offset=0&limit=20

Response:
{
  "success": true,
  "data": [...],
  "meta": {
    "requestId": "req_abc123",
    "total": 150,
    "offset": 0,
    "limit": 20
  }
}
```

## 筛选与排序

```
GET /v1/users?status=active&sort=-createdAt,name&fields=id,name,email
```

- 筛选: query parameter 直接对应字段名
- 排序: `sort` 参数，`-` 前缀表示降序
- 字段选择: `fields` 参数，逗号分隔

## 请求验证

使用 zod 进行输入验证：

```typescript
import { z } from "zod";

const createUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(["admin", "user"]).default("user"),
});

// 路由中使用
const validated = createUserSchema.parse(req.body);
```

## 必需请求头

### X-Device-Type

所有请求必须携带 `X-Device-Type` 头，标识客户端类型：

```
X-Device-Type: web | app | desktop
```

| 值        | 说明             |
| --------- | ---------------- |
| `web`     | 浏览器 Web 端    |
| `app`     | 移动端 App       |
| `desktop` | 桌面客户端       |

- 缺少该头: 400 `MISSING_DEVICE_TYPE`
- 值不合法: 400 `INVALID_DEVICE_TYPE`

### 认证

#### Bearer Token

```
Authorization: Bearer <token>
```

### 认证响应

- 缺少 token: 401 `AUTHENTICATION_REQUIRED`
- 无效 token: 401 `INVALID_TOKEN`
- 过期 token: 401 `TOKEN_EXPIRED`
- 无权限: 403 `INSUFFICIENT_PERMISSIONS`

## 速率限制

响应头包含速率限制信息：

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1620000000
```

超限时返回 429 + `Retry-After` 头。

## 安全规则

- 不在 URL 中传递敏感信息（token, password）
- 不在错误消息中暴露堆栈跟踪或内部路径
- 所有列表端点必须有分页限制（默认 20，最大 100）
- DELETE 操作使用软删除（设置 `deletedAt`），除非明确要求硬删除
- 写操作需要 CSRF 保护（适用于 cookie 认证场景）
