---
name: auth
description: "认证与授权设计技能。涵盖 JWT、OAuth2、RBAC/ABAC、安全检查清单。"
metadata:
  filePattern:
    - "**/auth/**/*"
    - "**/middleware/auth*"
    - "**/lib/auth*"
  bashPattern:
    - "jwt|oauth|clerk|auth0|next-auth"
  priority: 8
---

# Auth

## When to Activate

- 用户请求设计认证（authentication）
- 用户请求设计授权（authorization）或权限管理
- 用户请求实现 JWT、OAuth2、登录、注册
- 用户请求实现 RBAC、ABAC、基于角色的访问控制
- 用户请求令牌管理（刷新、吊销、轮换）
- 用户请求实现 MFA（多因素认证）

## Workflow

### 1. 收集认证需求

向用户确认以下信息（未提供时主动询问）：

| 项目           | 说明                                            | 默认值                   |
| -------------- | ----------------------------------------------- | ------------------------ |
| 认证方式       | JWT / OAuth2 / Session / API Key                | JWT                      |
| 令牌存储位置   | Cookie / Secure Storage / OS Credential Store   | 根据设备类型自动选择     |
| 会话有效期     | 访问令牌的有效期                                | 15 分钟                  |
| 刷新策略       | Rotation / Sliding / Fixed                      | Rotation                 |
| MFA            | TOTP / SMS / Email / WebAuthn / None            | None                     |
| 提供商         | 自行实现 / Auth.js / Supabase Auth / Firebase Auth | 自动检测               |
| 框架           | Express / Next.js / FastAPI / Hono              | 自动检测                 |
| 授权模型       | RBAC / ABAC / 两者 / None                       | RBAC                     |

### 2. 设计认证架构（结论先行）

先输出认证流程概要，再展开细节：

```
## 认证架构概要

| 流程       | 端点                   | 说明                 | Auth |
| ---------- | ---------------------- | -------------------- | ---- |
| 注册       | POST /v1/auth/signup   | 用户注册             | No   |
| 登录       | POST /v1/auth/login    | 认证并签发令牌       | No   |
| 令牌刷新   | POST /v1/auth/refresh  | 重新签发访问令牌     | Yes  |
| 登出       | POST /v1/auth/logout   | 令牌失效处理         | Yes  |
| 修改密码   | POST /v1/auth/password | 修改密码             | Yes  |
```

### 3. 生成认证代码

按以下顺序生成：

#### A. 令牌管理

- JWT 签发与验证逻辑
- 刷新令牌轮换
- 遵循: [references/jwt-patterns.md](references/jwt-patterns.md)

#### B. 授权中间件

- 基于角色的访问控制
- 资源级权限检查
- 遵循: [references/authorization.md](references/authorization.md)

#### C. 安全防护

- 密码哈希
- 暴力破解防御
- CSRF 防护
- 遵循: [references/security-checklist.md](references/security-checklist.md)

### 4. 安全审查

生成完成后，自动检查以下项目（参见 [references/security-checklist.md](references/security-checklist.md)）：

- [ ] 密码使用 bcrypt 或 argon2 进行哈希
- [ ] 访问令牌有效期 <= 15 分钟
- [ ] 刷新令牌已启用轮换机制
- [ ] Cookie 已设置 httpOnly、secure、sameSite 标志
- [ ] 登录端点已启用速率限制
- [ ] 令牌验证使用时序安全比较
- [ ] 认证事件输出审计日志
- [ ] 错误消息未泄露内部信息
- [ ] 使用统一响应信封（`success`、`data?`、`meta?`、`error?`）
- [ ] `meta.requestId` 存在于所有响应中
- [ ] 所有响应属性使用 camelCase
- [ ] 已验证 `X-Device-Type` 头
- [ ] Cookie 认证场景中 CSRF 防护已启用
- [ ] OAuth2 公共客户端已应用 PKCE

## Output Format

```
# Auth Design: {项目名称}

## 认证流程概要
{流程表}

## 令牌管理
{JWT 签发、验证、刷新代码}

## 授权中间件
{RBAC/ABAC 中间件代码}

## 安全防护
{密码哈希、速率限制、CSRF 防护}

## 安全审查
{审查结果清单，标记通过/未通过}
```
