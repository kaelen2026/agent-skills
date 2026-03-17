# Error Taxonomy

## 错误分类

### Operational 错误（运维错误）

可预期的运行时错误。可作为正常程序流程的一部分进行处理。

| 类别         | 说明                                 | 示例                                     |
| ------------ | ------------------------------------ | ---------------------------------------- |
| 验证         | 用户输入验证失败                    | 必填字段缺失、邮箱格式不正确             |
| 认证         | 认证信息缺失或无效                  | 令牌过期、无效令牌                       |
| 授权         | 权限不足                            | 访问管理员限定资源                       |
| 资源         | 请求的资源不存在                    | 使用不存在的用户 ID 进行查询             |
| 冲突         | 资源状态冲突                        | 重复邮箱注册、乐观锁失败                 |
| 频率限制     | 请求频率超限                        | API 调用次数达到上限                     |
| 外部服务     | 外部 API/DB 的暂时性故障            | 超时、连接拒绝                           |

### Programmer 错误（程序员错误）

代码 bug。无法恢复，需要修复。

| 类别         | 说明                           | 示例                                 |
| ------------ | ------------------------------ | ------------------------------------ |
| 类型错误     | 不正确的类型引用               | 访问 `undefined` 的属性              |
| 范围错误     | 不正确的值范围                 | 数组越界访问                         |
| 引用错误     | 引用未定义的变量               | 拼写错误的变量名                     |
| 断言         | 不变条件被违反                 | 不应为 NULL 的值为 NULL              |

**重要**: Programmer 错误不应被 catch 后正常化处理，而应让进程崩溃并记录日志。在生产环境中由 PM2/systemd 等进程管理器自动重启。

## HTTP 错误映射

### 4xx 客户端错误

| statusCode | 错误码                    | 说明                         | isOperational |
| ---------- | ------------------------- | ---------------------------- | ------------- |
| 400        | VALIDATION_ERROR          | 请求验证失败                 | true          |
| 400        | INVALID_DEVICE_TYPE       | X-Device-Type 不合法         | true          |
| 400        | MISSING_DEVICE_TYPE       | X-Device-Type 头部缺失       | true          |
| 400        | INVALID_REQUEST_BODY      | 请求体解析失败               | true          |
| 401        | AUTHENTICATION_REQUIRED   | 认证令牌缺失                 | true          |
| 401        | INVALID_TOKEN             | 令牌无效                     | true          |
| 401        | TOKEN_EXPIRED             | 令牌已过期                   | true          |
| 403        | INSUFFICIENT_PERMISSIONS  | 访问权限不足                 | true          |
| 404        | RESOURCE_NOT_FOUND        | 资源不存在                   | true          |
| 409        | RESOURCE_CONFLICT         | 资源冲突                     | true          |
| 409        | DUPLICATE_RESOURCE        | 尝试创建重复资源             | true          |
| 422        | BUSINESS_RULE_VIOLATION   | 不符合业务规则               | true          |
| 429        | RATE_LIMIT_EXCEEDED       | 请求频率超限                 | true          |

### 5xx 服务器错误

| statusCode | 错误码                    | 说明                         | isOperational |
| ---------- | ------------------------- | ---------------------------- | ------------- |
| 500        | INTERNAL_ERROR            | 内部服务器错误               | false         |
| 502        | UPSTREAM_SERVICE_ERROR    | 上游服务错误                 | false         |
| 503        | SERVICE_UNAVAILABLE       | 服务暂时不可用               | false         |
| 504        | GATEWAY_TIMEOUT           | 上游服务超时                 | false         |

## 自定义错误类层级

### 基类: AppError

```typescript
interface ErrorDetail {
  readonly field: string;
  readonly message: string;
  readonly code: string;
}

class AppError extends Error {
  readonly code: string;
  readonly statusCode: number;
  readonly isOperational: boolean;
  readonly details?: readonly ErrorDetail[];

  constructor(params: {
    readonly code: string;
    readonly message: string;
    readonly statusCode: number;
    readonly isOperational: boolean;
    readonly details?: readonly ErrorDetail[];
  }) {
    super(params.message);
    this.name = this.constructor.name;
    this.code = params.code;
    this.statusCode = params.statusCode;
    this.isOperational = params.isOperational;
    this.details = params.details;
    Error.captureStackTrace(this, this.constructor);
  }
}
```

### 子类

```typescript
class ValidationError extends AppError {
  constructor(message: string, details?: readonly ErrorDetail[]) {
    super({
      code: "VALIDATION_ERROR",
      message,
      statusCode: 400,
      isOperational: true,
      details,
    });
  }
}

class AuthenticationError extends AppError {
  constructor(
    message: string = "Authentication required",
    code: string = "AUTHENTICATION_REQUIRED"
  ) {
    super({
      code,
      message,
      statusCode: 401,
      isOperational: true,
    });
  }
}

class AuthorizationError extends AppError {
  constructor(message: string = "Insufficient permissions") {
    super({
      code: "INSUFFICIENT_PERMISSIONS",
      message,
      statusCode: 403,
      isOperational: true,
    });
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    const message = id
      ? `${resource} with id '${id}' not found`
      : `${resource} not found`;
    super({
      code: "RESOURCE_NOT_FOUND",
      message,
      statusCode: 404,
      isOperational: true,
    });
  }
}

class ConflictError extends AppError {
  constructor(
    message: string,
    code: string = "RESOURCE_CONFLICT"
  ) {
    super({
      code,
      message,
      statusCode: 409,
      isOperational: true,
    });
  }
}

class RateLimitError extends AppError {
  constructor(message: string = "Rate limit exceeded") {
    super({
      code: "RATE_LIMIT_EXCEEDED",
      message,
      statusCode: 429,
      isOperational: true,
    });
  }
}

class BusinessRuleError extends AppError {
  constructor(message: string, details?: readonly ErrorDetail[]) {
    super({
      code: "BUSINESS_RULE_VIOLATION",
      message,
      statusCode: 422,
      isOperational: true,
      details,
    });
  }
}

class InternalError extends AppError {
  constructor(message: string = "Internal server error") {
    super({
      code: "INTERNAL_ERROR",
      message,
      statusCode: 500,
      isOperational: false,
    });
  }
}
```

### 使用示例

```typescript
// 验证错误
throw new ValidationError("Invalid input", [
  { field: "email", message: "Invalid email format", code: "invalid_string" },
  { field: "age", message: "Must be at least 0", code: "too_small" },
]);

// 认证错误（令牌过期）
throw new AuthenticationError("Token has expired", "TOKEN_EXPIRED");

// 资源未找到
throw new NotFoundError("User", "usr_abc123");

// 冲突错误
throw new ConflictError("Email already registered", "DUPLICATE_RESOURCE");
```

## 统一错误响应格式

遵循 api-design 技能的统一信封。所有属性使用 **camelCase**：

### 响应结构

```typescript
interface ErrorResponse {
  readonly success: false;
  readonly error: {
    readonly code: string;      // UPPER_SNAKE_CASE
    readonly message: string;   // 面向用户的消息
    readonly details?: readonly ErrorDetail[];
  };
  readonly meta: {
    readonly requestId: string; // 始终存在
  };
}
```

### 验证错误示例

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "invalid_string"
      },
      {
        "field": "name",
        "message": "Required",
        "code": "invalid_type"
      }
    ]
  },
  "meta": {
    "requestId": "req_abc123"
  }
}
```

### 认证错误示例

```json
{
  "success": false,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Authentication token has expired"
  },
  "meta": {
    "requestId": "req_def456"
  }
}
```

### 内部错误示例（生产环境）

```json
{
  "success": false,
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred"
  },
  "meta": {
    "requestId": "req_ghi789"
  }
}
```

**重要**: 5xx 错误不应在响应中包含内部实现细节（堆栈跟踪、DB 错误消息等）。详细信息仅记录在服务器日志中。

## 错误码注册表

### 通用错误码

| Code                     | statusCode | Description                      | isOperational |
| ------------------------ | ---------- | -------------------------------- | ------------- |
| VALIDATION_ERROR         | 400        | 请求验证失败                     | true          |
| INVALID_DEVICE_TYPE      | 400        | X-Device-Type 头部不合法         | true          |
| MISSING_DEVICE_TYPE      | 400        | X-Device-Type 头部缺失           | true          |
| INVALID_REQUEST_BODY     | 400        | 请求体解析失败                   | true          |
| AUTHENTICATION_REQUIRED  | 401        | 认证令牌缺失                     | true          |
| INVALID_TOKEN            | 401        | 令牌无效                         | true          |
| TOKEN_EXPIRED            | 401        | 令牌已过期                       | true          |
| INSUFFICIENT_PERMISSIONS | 403        | 访问权限不足                     | true          |
| RESOURCE_NOT_FOUND       | 404        | 资源不存在                       | true          |
| RESOURCE_CONFLICT        | 409        | 资源冲突                         | true          |
| DUPLICATE_RESOURCE       | 409        | 尝试创建重复资源                 | true          |
| BUSINESS_RULE_VIOLATION  | 422        | 不符合业务规则                   | true          |
| RATE_LIMIT_EXCEEDED      | 429        | 请求频率超限                     | true          |
| INTERNAL_ERROR           | 500        | 内部服务器错误                   | false         |
| UPSTREAM_SERVICE_ERROR   | 502        | 上游服务错误                     | false         |
| SERVICE_UNAVAILABLE      | 503        | 服务暂时不可用                   | false         |
| GATEWAY_TIMEOUT          | 504        | 上游服务超时                     | false         |

### 领域特定错误码命名规则

添加项目特定错误码时，遵循以下命名规则：

```
{DOMAIN}_{ACTION}_{REASON}
```

例:

| Code                         | statusCode | Description                    |
| ---------------------------- | ---------- | ------------------------------ |
| ORDER_CANCEL_ALREADY_SHIPPED | 422        | 已发货订单不可取消             |
| PAYMENT_CHARGE_INSUFFICIENT  | 422        | 余额不足导致支付失败           |
| USER_REGISTER_EMAIL_TAKEN    | 409        | 邮箱地址已被使用               |

## 基于 isOperational 的分支处理

```typescript
function isOperationalError(error: unknown): boolean {
  if (error instanceof AppError) {
    return error.isOperational;
  }
  return false;
}
```

| isOperational | 处理策略                                                   |
| ------------- | ---------------------------------------------------------- |
| `true`        | 向用户返回错误消息。进程继续运行                           |
| `false`       | 返回通用消息。详细信息记录到日志。触发告警                 |
