---
name: error-handling
description: "错误处理设计技能。生成错误分类体系、处理模式、日志/监控指南。"
metadata:
  filePattern:
    - "**/errors.ts"
    - "**/errors/**/*"
    - "**/error-handler*"
  priority: 6
---

# Error Handling

## When to Activate

- 用户请求错误处理设计
- 用户询问错误分类体系（error taxonomy）
- 用户请求错误模式设计（retry, circuit breaker 等）
- 用户询问重试策略
- 用户请求错误日志/监控设计

## Workflow

### 1. 现状分析

分析现有错误处理代码，确认以下内容：

| 项目             | 说明                                         | 确认方法                        |
| ---------------- | -------------------------------------------- | ------------------------------- |
| 错误类           | 自定义错误类的有无及结构                     | 搜索项目中的 Error 类            |
| 错误响应         | 是否遵循统一信封 `{ success, error, meta }`  | 确认 API 响应格式                |
| 错误中间件       | 是否存在全局错误处理器                       | 确认框架中间件                   |
| 日志策略         | 是否使用结构化日志、requestId 追踪           | 确认日志输出位置                 |
| 重试策略         | 外部 API 调用时的重试实现                    | 确认 HTTP 客户端配置             |

### 2. 错误分类体系设计

根据 [references/error-taxonomy.md](references/error-taxonomy.md) 设计错误分类体系：

先输出错误码一览表，再展开细节：

```
## 错误码一览

| Code                    | statusCode | Description            | isOperational |
| ----------------------- | ---------- | ---------------------- | ------------- |
| VALIDATION_ERROR        | 400        | 请求验证失败           | true          |
| AUTHENTICATION_REQUIRED | 401        | 需要认证               | true          |
| INSUFFICIENT_PERMISSIONS| 403        | 权限不足               | true          |
| RESOURCE_NOT_FOUND      | 404        | 资源不存在             | true          |
| RESOURCE_CONFLICT       | 409        | 资源冲突               | true          |
| RATE_LIMIT_EXCEEDED     | 429        | 请求频率超限           | true          |
| INTERNAL_ERROR          | 500        | 内部服务器错误         | false         |
```

### 3. 错误处理代码生成

根据 [references/handling-patterns.md](references/handling-patterns.md) 生成以下内容：

#### A. 自定义错误类

- 文件: `errors.ts`（或 `errors/index.ts`）
- AppError 基类 + 各子类
- 不可变模式（`readonly` 属性）

#### B. 错误中间件

- 文件: 框架对应的错误处理器
- 统一信封格式的响应转换
- Zod 验证错误的转换

#### C. 重试/熔断器

- 外部 API 调用的重试包装器
- Exponential backoff with jitter
- 熔断器模式

#### D. 日志配置

- 根据 [references/logging-monitoring.md](references/logging-monitoring.md) 配置结构化日志
- 通过 requestId 进行关联追踪

### 4. 设计审查

生成完成后，自动检查以下规则：

- [ ] 所有错误均继承 AppError 基类
- [ ] 遵循统一信封 `{ success: false, error, meta }` 格式
- [ ] 所有错误响应中包含 `meta.requestId`
- [ ] 错误码使用 UPPER_SNAKE_CASE
- [ ] 响应属性使用 camelCase
- [ ] 已定义 `X-Device-Type` 头部的验证错误
- [ ] Operational 错误与 Programmer 错误明确分离
- [ ] 堆栈跟踪不会泄露到生产响应中
- [ ] 日志中不包含密码、令牌、PII
- [ ] 外部 API 调用已实现重试策略
- [ ] 存在全局 uncaught exception 处理器
- [ ] Zod 验证错误已转换为统一格式
- [ ] 日志包含 requestId, timestamp, level

## Output Format

```
# Error Handling Design: {项目名称}

## 错误码一览
{错误码表}

## 自定义错误类
{errors.ts 生成}

## 错误中间件
{框架对应处理器生成}

## 重试/熔断器
{外部 API 调用模式生成}

## 日志配置
{结构化日志配置生成}

## 设计审查
{审查结果清单，标记通过/未通过}
```
