# OpenAPI Conventions

## 文件结构

```yaml
openapi: "3.1.0"
info:
  title: "{Project} API"
  version: "1.0.0"
  description: "API description"

servers:
  - url: "https://api.example.com/v1"
    description: "Production"
  - url: "http://localhost:3000/v1"
    description: "Local development"

paths:
  # 端点定义

components:
  schemas:
    # 数据模型
  securitySchemes:
    # 认证方式
  responses:
    # 共享响应

security:
  - bearerAuth: []
```

## 命名规范

### Schema 命名

- 资源模型: PascalCase 单数 (`User`, `Order`)
- 请求体: `Create{Resource}Request`, `Update{Resource}Request`
- 响应体: `{Resource}Response`, `{Resource}ListResponse`
- 错误: `ErrorResponse`
- 信封: `ApiResponse`, `Meta`, `ApiError`

### 属性命名

- 使用 camelCase: `createdAt`, `userId`, `orderItems`
- 布尔字段使用 `is` / `has` 前缀: `isActive`, `hasPermission`
- 时间字段使用 `At` 后缀: `createdAt`, `updatedAt`, `deletedAt`
- ID 字段: `id`（主键）, `{resource}Id`（外键）

## Schema 定义模式

### 资源模型

```yaml
User:
  type: object
  required:
    - id
    - name
    - email
    - createdAt
  properties:
    id:
      type: string
      format: uuid
      description: "Unique identifier"
    name:
      type: string
      minLength: 1
      maxLength: 100
      description: "User's display name"
    email:
      type: string
      format: email
      description: "User's email address"
    role:
      type: string
      enum: [admin, user]
      default: user
    isActive:
      type: boolean
      default: true
    createdAt:
      type: string
      format: date-time
    updatedAt:
      type: string
      format: date-time
      nullable: true
```

### 请求体

```yaml
CreateUserRequest:
  type: object
  required:
    - name
    - email
  properties:
    name:
      type: string
      minLength: 1
      maxLength: 100
    email:
      type: string
      format: email
    role:
      type: string
      enum: [admin, user]
      default: user
```

### 统一响应信封

```yaml
ApiResponse:
  type: object
  required:
    - success
  properties:
    success:
      type: boolean
      description: "Whether the request was successful"
    data:
      description: "Response payload (present on success)"
    meta:
      $ref: "#/components/schemas/Meta"
    error:
      $ref: "#/components/schemas/ApiError"

Meta:
  type: object
  description: "Request tracking and pagination metadata"
  required:
    - requestId
  properties:
    requestId:
      type: string
      description: "Unique request identifier for tracing"
    cursor:
      type: string
      nullable: true
    hasMore:
      type: boolean
    total:
      type: integer
    offset:
      type: integer
    limit:
      type: integer

ApiError:
  type: object
  required:
    - code
    - message
  properties:
    code:
      type: string
      description: "Machine-readable error code"
    message:
      type: string
      description: "Human-readable error message"
    details:
      type: array
      items:
        type: object
        properties:
          field:
            type: string
          message:
            type: string
          code:
            type: string
```

### 单资源响应

```yaml
UserResponse:
  allOf:
    - $ref: "#/components/schemas/ApiResponse"
    - type: object
      properties:
        success:
          enum: [true]
        data:
          $ref: "#/components/schemas/User"
```

### 列表响应（带分页）

```yaml
UserListResponse:
  allOf:
    - $ref: "#/components/schemas/ApiResponse"
    - type: object
      properties:
        success:
          enum: [true]
        data:
          type: array
          items:
            $ref: "#/components/schemas/User"
        meta:
          $ref: "#/components/schemas/Meta"
```

### 错误响应

```yaml
ErrorResponse:
  allOf:
    - $ref: "#/components/schemas/ApiResponse"
    - type: object
      properties:
        success:
          enum: [false]
        error:
          $ref: "#/components/schemas/ApiError"
```

## 共享参数

```yaml
components:
  parameters:
    DeviceType:
      name: X-Device-Type
      in: header
      required: true
      description: "Client device type"
      schema:
        type: string
        enum: [web, app, desktop]
```

## 端点定义模式

```yaml
/users:
  get:
    summary: "List users"
    operationId: listUsers
    tags: [Users]
    parameters:
      - $ref: "#/components/parameters/DeviceType"
      - name: cursor
        in: query
        schema:
          type: string
      - name: limit
        in: query
        schema:
          type: integer
          minimum: 1
          maximum: 100
          default: 20
    responses:
      "200":
        description: "Successful response"
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/UserListResponse"
      "401":
        $ref: "#/components/responses/Unauthorized"

  post:
    summary: "Create a user"
    operationId: createUser
    tags: [Users]
    parameters:
      - $ref: "#/components/parameters/DeviceType"
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/CreateUserRequest"
    responses:
      "201":
        description: "User created"
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/UserResponse"
      "400":
        $ref: "#/components/responses/ValidationError"
      "409":
        $ref: "#/components/responses/Conflict"
```

## 共享响应

```yaml
components:
  responses:
    Unauthorized:
      description: "Authentication required"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    Forbidden:
      description: "Insufficient permissions"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    NotFound:
      description: "Resource not found"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    ValidationError:
      description: "Request validation failed"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    Conflict:
      description: "Resource conflict"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
```

## Security Schemes

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
```
