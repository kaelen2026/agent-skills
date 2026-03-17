---
name: api-docs
version: 1.0.0
description: API 文档编写标准 — 端点说明、错误码、认证、速率限制及版本迁移指南
last_updated: 2026-03-17
---

# API 文档标准

## 概述

API 文档是面向开发者的产品界面。好的 API 文档能让使用者在 10 分钟内完成首次调用。

类比：API 文档就像餐厅的菜单——顾客不需要知道厨房怎么运作，但需要清楚每道菜是什么、多少钱、有什么忌口。

> **注意**：OpenAPI 规范定义请参考 api-design 技能。本文档聚焦于 API 文档的编写标准和模板。

## 端点文档格式

每个端点的文档应包含以下部分：

### 标准模板

```markdown
## 端点名称

简要描述该端点的功能和用途。

### 请求

`METHOD /api/v1/resource`

#### 路径参数

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `id` | string | 是 | 资源的唯一标识 |

#### 查询参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `cursor` | string | 否 | - | 分页游标 |
| `limit` | integer | 否 | 20 | 每页数量（1-100） |

#### 请求头

| 请求头 | 必填 | 说明 |
|--------|------|------|
| `Authorization` | 是 | Bearer token |
| `Content-Type` | 是 | `application/json` |

#### 请求体

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 名称（1-100 字符） |
| `email` | string | 是 | 邮箱地址 |
| `role` | string | 否 | 角色，可选值：`admin` \| `member` |

### 请求示例

```bash
curl -X POST https://api.example.com/v1/users \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "张三",
    "email": "zhangsan@example.com",
    "role": "member"
  }'
```

### 响应

#### 成功响应（200 OK）

```json
{
  "data": {
    "id": "usr_abc123",
    "name": "张三",
    "email": "zhangsan@example.com",
    "role": "member",
    "created_at": "2026-03-17T10:30:00Z"
  }
}
```

#### 错误响应（422 Unprocessable Entity）

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求参数验证失败",
    "details": [
      {
        "field": "email",
        "message": "邮箱格式无效"
      }
    ]
  }
}
```
```

### 编写原则

- **请求和响应都给完整示例**：开发者可以直接复制修改
- **标注必填/选填**：避免使用者猜测
- **说明约束条件**：长度限制、取值范围、格式要求
- **使用真实数据**：不用 `foo`、`bar`、`test123`

---

## 错误文档

### 统一错误格式

所有 API 错误应返回统一格式：

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "面向用户的错误说明",
    "details": []
  }
}
```

### 错误码清单

文档中应列出所有可能的错误码：

```markdown
## 错误码参考

### 通用错误

| HTTP 状态码 | 错误码 | 说明 | 处理建议 |
|------------|--------|------|---------|
| 400 | `BAD_REQUEST` | 请求格式错误 | 检查请求体 JSON 格式 |
| 401 | `UNAUTHORIZED` | 未提供有效凭证 | 检查 Authorization 请求头 |
| 403 | `FORBIDDEN` | 无权访问该资源 | 确认账户权限 |
| 404 | `NOT_FOUND` | 资源不存在 | 检查资源 ID 是否正确 |
| 409 | `CONFLICT` | 资源冲突 | 该资源已存在，使用更新接口 |
| 422 | `VALIDATION_ERROR` | 请求参数验证失败 | 查看 details 字段修正参数 |
| 429 | `RATE_LIMITED` | 请求频率超限 | 等待后重试，参见速率限制章节 |
| 500 | `INTERNAL_ERROR` | 服务器内部错误 | 联系技术支持 |

### 业务错误

| HTTP 状态码 | 错误码 | 说明 | 处理建议 |
|------------|--------|------|---------|
| 402 | `INSUFFICIENT_BALANCE` | 账户余额不足 | 充值后重试 |
| 409 | `EMAIL_ALREADY_EXISTS` | 邮箱已被注册 | 使用其他邮箱或找回密码 |
| 423 | `ACCOUNT_LOCKED` | 账户已被锁定 | 联系管理员解锁 |
```

### 错误文档编写原则

- **每个错误码都提供处理建议**：告诉开发者怎么修，而非只说"出错了"
- **区分通用错误和业务错误**：方便查找
- **附带响应示例**：展示错误返回的完整 JSON
- **不泄露内部信息**：错误消息不包含堆栈、SQL、内部路径

---

## 认证章节

```markdown
## 认证

本 API 使用 Bearer Token 认证。

### 获取 API Key

1. 登录 [控制台](https://console.example.com)
2. 进入 **设置** > **API Keys**
3. 点击 **创建新密钥**

> ⚠️ API Key 仅在创建时显示一次，请妥善保存。

### 使用方式

在每个请求的 `Authorization` 请求头中携带 token：

```bash
curl https://api.example.com/v1/users \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### 安全建议

- **不要**在客户端代码中硬编码 API Key
- **不要**将 API Key 提交到版本控制
- **使用**环境变量存储 API Key
- **定期**轮换 API Key
- **限制** API Key 的权限范围（最小权限原则）

### Token 过期

| Token 类型 | 有效期 | 刷新方式 |
|-----------|--------|---------|
| API Key | 永不过期（可手动吊销） | 控制台重新生成 |
| Access Token | 1 小时 | 使用 refresh token 刷新 |
| Refresh Token | 30 天 | 重新登录 |
```

---

## 速率限制章节

```markdown
## 速率限制

为保障服务稳定性，API 对请求频率进行了限制。

### 限制规则

| 接口类型 | 限制 | 时间窗口 |
|---------|------|---------|
| 读取接口（GET） | 1000 次 | 每分钟 |
| 写入接口（POST/PUT/DELETE） | 200 次 | 每分钟 |
| 搜索接口 | 100 次 | 每分钟 |
| 文件上传 | 50 次 | 每分钟 |

### 响应头

每个响应都包含速率限制相关的请求头：

| 请求头 | 说明 |
|--------|------|
| `X-RateLimit-Limit` | 时间窗口内的总配额 |
| `X-RateLimit-Remaining` | 剩余可用次数 |
| `X-RateLimit-Reset` | 配额重置的 Unix 时间戳 |

### 超限处理

超过限制时返回 `429 Too Many Requests`：

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "请求频率超限，请稍后重试",
    "details": {
      "retry_after": 30
    }
  }
}
```

### 最佳实践

- 实现指数退避重试（exponential backoff）
- 监控 `X-RateLimit-Remaining`，在耗尽前主动降速
- 批量操作使用批量接口而非循环单条请求
- 缓存不常变化的数据，减少不必要的请求
```

---

## SDK / 客户端库使用示例

```markdown
## SDK 使用

### 安装

```bash
# Node.js
npm install @your-org/sdk

# Python
pip install your-sdk

# Go
go get github.com/your-org/sdk-go
```

### Node.js

```typescript
import { Client } from "@your-org/sdk";

const client = new Client({
  apiKey: process.env.API_KEY,
});

// 创建用户
const user = await client.users.create({
  name: "张三",
  email: "zhangsan@example.com",
});

// 列表查询（自动分页）
for await (const user of client.users.list()) {
  console.log(user.name);
}
```

### Python

```python
from your_sdk import Client

client = Client(api_key=os.environ["API_KEY"])

# 创建用户
user = client.users.create(
    name="张三",
    email="zhangsan@example.com",
)

# 列表查询（自动分页）
for user in client.users.list():
    print(user.name)
```

### Go

```go
package main

import (
    "context"
    "os"

    sdk "github.com/your-org/sdk-go"
)

func main() {
    client := sdk.NewClient(os.Getenv("API_KEY"))

    // 创建用户
    user, err := client.Users.Create(context.Background(), &sdk.CreateUserParams{
        Name:  "张三",
        Email: "zhangsan@example.com",
    })
    if err != nil {
        log.Fatal(err)
    }
}
```
```

### SDK 文档编写原则

- **每种语言给完整可运行的示例**：从安装到调用
- **覆盖主要操作**：CRUD + 分页 + 错误处理
- **展示惯用写法**：遵循各语言的编码习惯
- **说明环境变量配置**：不硬编码密钥

---

## 版本管理文档

```markdown
## API 版本管理

### 版本策略

本 API 使用 URL 路径版本控制：

```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

### 版本生命周期

| 阶段 | 说明 | 持续时间 |
|------|------|---------|
| Current | 当前推荐版本，持续迭代 | 持续 |
| Deprecated | 仍可使用，但不再新增功能 | 12 个月 |
| Retired | 停止服务，请求返回 410 Gone | - |

### 当前版本状态

| 版本 | 状态 | 发布日期 | 预计下线日期 |
|------|------|---------|------------|
| v2 | Current | 2026-01-01 | - |
| v1 | Deprecated | 2025-01-01 | 2027-01-01 |
```

---

## 版本迁移指南

```markdown
## 从 v1 迁移到 v2

### 概述

v2 版本主要改进了分页方式、错误格式和认证机制。大部分端点保持
向后兼容，以下列出所有破坏性变更。

### 破坏性变更清单

#### 1. 分页方式变更

**v1（偏移量分页）：**

```bash
GET /v1/users?page=2&per_page=20
```

**v2（游标分页）：**

```bash
GET /v2/users?cursor=eyJpZCI6MTAwfQ&limit=20
```

**迁移方法：**

- 移除 `page` 和 `per_page` 参数
- 使用响应中的 `next_cursor` 字段获取下一页
- SDK 用户：升级到 v2 SDK，分页逻辑自动适配

#### 2. 错误响应格式变更

**v1：**

```json
{
  "error": "Not found",
  "status": 404
}
```

**v2：**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "请求的资源不存在",
    "details": []
  }
}
```

**迁移方法：**

- 更新错误处理逻辑，从 `error.code` 读取错误类型
- 使用 `error.details` 获取详细信息

#### 3. 日期格式统一为 ISO 8601

**v1：**

```json
{ "created_at": "2026-03-17 10:30:00" }
```

**v2：**

```json
{ "created_at": "2026-03-17T10:30:00Z" }
```

**迁移方法：**

- 更新日期解析逻辑以支持 ISO 8601 格式

### 迁移检查清单

- [ ] 更新 API 基础 URL 中的版本号（v1 -> v2）
- [ ] 替换偏移量分页为游标分页
- [ ] 更新错误处理逻辑
- [ ] 更新日期解析格式
- [ ] 升级 SDK 到最新版本
- [ ] 运行集成测试验证
- [ ] 在预发布环境验证

### 获取帮助

迁移过程中遇到问题：

- 查看 [常见问题](https://docs.example.com/migration-faq)
- 提交 [GitHub Issue](https://github.com/your-org/api/issues)
- 联系技术支持：api-support@example.com
```
