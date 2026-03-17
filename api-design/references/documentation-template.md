# API Documentation Template

## 文档结构

```markdown
# {Project} API Documentation

## Overview

{简要描述 API 用途和核心功能}

**Base URL**: `https://api.example.com/v1`

## Authentication

{认证方式说明}

### Request Header

\`\`\`
Authorization: Bearer <your-token>
\`\`\`

## Response Envelope

All responses use a unified envelope format:

\`\`\`json
{
  "success": true,
  "data": { ... },
  "meta": { ... }
}
\`\`\`

| Field     | Type    | Always | Description                          |
| --------- | ------- | ------ | ------------------------------------ |
| `success` | boolean | Yes    | `true` on success, `false` on error  |
| `data`    | any     | No     | Response payload (success only)      |
| `meta`    | object  | No     | Pagination, rate limiting metadata   |
| `error`   | object  | No     | Error details (failure only)         |

## Endpoints

### {Resource}

#### List {Resources}

\`\`\`
GET /v1/{resources}
\`\`\`

**Query Parameters**

| Parameter | Type    | Required | Default | Description         |
| --------- | ------- | -------- | ------- | ------------------- |
| cursor    | string  | No       | -       | Pagination cursor   |
| limit     | integer | No       | 20      | Items per page (1-100) |

**Response** `200 OK`

\`\`\`json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "Example",
      "created_at": "2026-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "cursor": "eyJpZCI6MTAwfQ",
    "has_more": true
  }
}
\`\`\`

---

#### Create {Resource}

\`\`\`
POST /v1/{resources}
\`\`\`

**Request Body**

\`\`\`json
{
  "name": "Example",
  "email": "user@example.com"
}
\`\`\`

**Response** `201 Created`

\`\`\`json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Example",
    "email": "user@example.com",
    "created_at": "2026-01-01T00:00:00Z"
  }
}
\`\`\`

---

#### Get {Resource}

\`\`\`
GET /v1/{resources}/:id
\`\`\`

**Path Parameters**

| Parameter | Type   | Description          |
| --------- | ------ | -------------------- |
| id        | string | Resource UUID        |

**Response** `200 OK`

\`\`\`json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Example",
    "email": "user@example.com",
    "created_at": "2026-01-01T00:00:00Z"
  }
}
\`\`\`

---

#### Update {Resource}

\`\`\`
PATCH /v1/{resources}/:id
\`\`\`

**Request Body** (partial update)

\`\`\`json
{
  "name": "Updated Name"
}
\`\`\`

**Response** `200 OK`

\`\`\`json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Updated Name",
    "email": "user@example.com",
    "created_at": "2026-01-01T00:00:00Z"
  }
}
\`\`\`

---

#### Delete {Resource}

\`\`\`
DELETE /v1/{resources}/:id
\`\`\`

**Response** `204 No Content`

---

## Error Responses

All errors follow the same envelope with `success: false`:

\`\`\`json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable description",
    "details": []
  }
}
\`\`\`

### Common Error Codes

| HTTP Status | Error Code              | Description              |
| ----------- | ----------------------- | ------------------------ |
| 400         | VALIDATION_ERROR        | Request validation failed |
| 401         | AUTHENTICATION_REQUIRED | Missing or invalid token |
| 403         | INSUFFICIENT_PERMISSIONS | No access to resource   |
| 404         | RESOURCE_NOT_FOUND      | Resource does not exist  |
| 409         | RESOURCE_CONFLICT       | Duplicate resource       |
| 429         | RATE_LIMIT_EXCEEDED     | Too many requests        |

## Rate Limiting

| Header                | Description              |
| --------------------- | ------------------------ |
| X-RateLimit-Limit     | Max requests per window  |
| X-RateLimit-Remaining | Remaining requests       |
| X-RateLimit-Reset     | Window reset timestamp   |
```

## 撰写规则

1. 每个端点必须包含: HTTP 方法、路径、参数表、请求/响应示例
2. 示例使用真实感的数据（不用 "foo", "bar"）
3. 响应示例中包含所有字段
4. 错误场景至少覆盖 400, 401, 404
5. 分页端点必须展示 pagination 对象
