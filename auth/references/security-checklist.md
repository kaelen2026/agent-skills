# Auth Security Checklist

## 密码哈希

### 推荐算法

| 算法       | 推荐度   | 备注                     |
| ---------- | -------- | ------------------------ |
| argon2id   | 最推荐   | 内存硬化，最新标准       |
| bcrypt     | 推荐     | 成熟可靠，广泛支持       |
| scrypt     | 可接受   | 内存硬化                 |
| PBKDF2     | 不推荐   | 较旧，易受 GPU 攻击      |
| SHA-256    | 禁止     | 不适合密码哈希           |
| MD5        | 禁止     | 已被完全破解             |
| 明文       | 禁止     | 不可接受                 |

### 实现模式

```typescript
import bcrypt from "bcrypt";

const SALT_ROUNDS = 12; // 最低 10，推荐 12

const hashPassword = async (password: string): Promise<string> => {
  return bcrypt.hash(password, SALT_ROUNDS);
};

const verifyPassword = async (
  password: string,
  hash: string
): Promise<boolean> => {
  return bcrypt.compare(password, hash);
};
```

```typescript
// argon2 推荐配置
import argon2 from "argon2";

const hashPassword = async (password: string): Promise<string> => {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536, // 64 MB
    timeCost: 3,
    parallelism: 4,
  });
};

const verifyPassword = async (
  password: string,
  hash: string
): Promise<boolean> => {
  return argon2.verify(hash, password);
};
```

### 密码策略

```typescript
import { z } from "zod";

const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .max(128, "Password must not exceed 128 characters")
  .regex(/[A-Z]/, "Password must contain at least one uppercase letter")
  .regex(/[a-z]/, "Password must contain at least one lowercase letter")
  .regex(/[0-9]/, "Password must contain at least one number")
  .regex(/[^A-Za-z0-9]/, "Password must contain at least one special character");
```

## 暴力破解防御

### 登录速率限制

```typescript
import rateLimit from "express-rate-limit";

const loginRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 5,                    // 最多 5 次
  skipSuccessfulRequests: true,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    success: false,
    error: {
      code: "RATE_LIMIT_EXCEEDED",
      message: "Too many login attempts, please try again later",
    },
  },
  keyGenerator: (req) => {
    // 使用 IP + 邮箱地址的组合进行限制
    return `${req.ip}:${req.body?.email ?? "unknown"}`;
  },
});

router.post("/v1/auth/login", loginRateLimiter, loginHandler);
```

### 账户锁定

```typescript
const MAX_FAILED_ATTEMPTS = 5;
const LOCKOUT_DURATION_MS = 30 * 60 * 1000; // 30 分钟

const checkAccountLockout = async (
  userId: string
): Promise<{ isLocked: boolean; remainingMs: number }> => {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { failedLoginAttempts: true, lockedUntil: true },
  });

  if (!user) {
    return { isLocked: false, remainingMs: 0 };
  }

  if (user.lockedUntil && user.lockedUntil > new Date()) {
    return {
      isLocked: true,
      remainingMs: user.lockedUntil.getTime() - Date.now(),
    };
  }

  return { isLocked: false, remainingMs: 0 };
};

const recordFailedLogin = async (userId: string): Promise<void> => {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { failedLoginAttempts: true },
  });

  const attempts = (user?.failedLoginAttempts ?? 0) + 1;

  await db.user.update({
    where: { id: userId },
    data: {
      failedLoginAttempts: attempts,
      lockedUntil:
        attempts >= MAX_FAILED_ATTEMPTS
          ? new Date(Date.now() + LOCKOUT_DURATION_MS)
          : null,
    },
  });
};

const resetFailedLogin = async (userId: string): Promise<void> => {
  await db.user.update({
    where: { id: userId },
    data: { failedLoginAttempts: 0, lockedUntil: null },
  });
};
```

## CSRF 防御

### SameSite Cookie（推荐）

```typescript
res.cookie("accessToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "strict", // 或 "lax"（OAuth 回调场景）
  path: "/",
});
```

### CSRF 令牌（用于不支持 SameSite 的环境）

```typescript
import csrf from "csurf";

const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
  },
});

// CSRF 令牌获取端点
router.get("/v1/auth/csrf-token", csrfProtection, (req, res) => {
  res.json({
    success: true,
    data: { csrfToken: req.csrfToken() },
    meta: { requestId: req.requestId },
  });
});

// 对写入操作应用 CSRF 验证
router.post("/v1/auth/login", csrfProtection, loginHandler);
router.post("/v1/auth/logout", csrfProtection, logoutHandler);
```

### Double Submit Cookie 模式

```typescript
import { randomBytes } from "crypto";

const generateCsrfToken = (): string => {
  return randomBytes(32).toString("hex");
};

const setCsrfCookie = (res: Response, token: string): void => {
  // 可被 JavaScript 读取的 Cookie（httpOnly: false）
  res.cookie("csrfToken", token, {
    httpOnly: false,
    secure: true,
    sameSite: "strict",
    path: "/",
  });
};

const verifyCsrf = (req: Request, res: Response, next: NextFunction): void => {
  const cookieToken = req.cookies?.csrfToken;
  const headerToken = req.headers["x-csrf-token"] as string;

  if (!cookieToken || !headerToken || cookieToken !== headerToken) {
    res.status(403).json({
      success: false,
      error: {
        code: "CSRF_VALIDATION_FAILED",
        message: "CSRF token validation failed",
      },
      meta: { requestId: req.requestId },
    });
    return;
  }

  next();
};
```

## XSS 防御（认证流程）

```typescript
// 不要将令牌保存在 localStorage 中（会被 XSS 攻击窃取）
// WRONG: localStorage.setItem("accessToken", token);

// 保存在 httpOnly Cookie 中（JavaScript 无法访问）
// CORRECT: 在 Cookie 上设置 httpOnly 标志

// 响应头
const securityHeaders = (req: Request, res: Response, next: NextFunction): void => {
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("X-Frame-Options", "DENY");
  res.setHeader("X-XSS-Protection", "0"); // 依赖 CSP
  res.setHeader(
    "Content-Security-Policy",
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';"
  );
  res.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
  next();
};
```

## Secure Cookie 标志

| 标志         | 值         | 说明                                         |
| ------------ | ---------- | -------------------------------------------- |
| `httpOnly`   | `true`     | JavaScript 无法访问（XSS 防御）              |
| `secure`     | `true`     | 仅通过 HTTPS 发送                            |
| `sameSite`   | `"strict"` | 仅同站发送（CSRF 防御）                      |
| `path`       | `"/"`      | Cookie 有效路径（refresh 建议限定路径）      |
| `domain`     | 省略       | 限定为当前域名                               |
| `maxAge`     | 适当值     | Access: 15 分钟，Refresh: 7 天               |

```typescript
const COOKIE_OPTIONS_ACCESS = {
  httpOnly: true,
  secure: true,
  sameSite: "strict" as const,
  path: "/",
  maxAge: 15 * 60 * 1000, // 15 分钟
};

const COOKIE_OPTIONS_REFRESH = {
  httpOnly: true,
  secure: true,
  sameSite: "strict" as const,
  path: "/v1/auth/refresh",
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 天
};
```

## 令牌安全

| 项目                     | 推荐值                   | 原因                             |
| ------------------------ | ------------------------ | -------------------------------- |
| Access Token 有效期      | <= 15 分钟               | 最小化泄露时的损害期             |
| Refresh Token 有效期     | <= 7 天                  | 过长会增加重用风险               |
| 签名算法                 | RS256 / ES256            | 非对称密钥分离签名与验证         |
| 令牌大小                 | 最小化载荷               | 不包含不必要的信息               |
| Refresh Token 存储       | 哈希后存储到 DB          | 明文存储在泄露时会被立即利用     |
| jti（JWT ID）            | 所有令牌均须附带         | 吊销管理必需                     |

## OAuth2 安全

### PKCE（Proof Key for Code Exchange）

公共客户端（SPA、移动端）必须启用 PKCE：

```typescript
import { randomBytes, createHash } from "crypto";

const generateCodeVerifier = (): string => {
  return randomBytes(32).toString("base64url");
};

const generateCodeChallenge = (verifier: string): string => {
  return createHash("sha256").update(verifier).digest("base64url");
};

// 授权请求
const authorizationUrl = new URL("https://provider.com/authorize");
authorizationUrl.searchParams.set("response_type", "code");
authorizationUrl.searchParams.set("client_id", clientId);
authorizationUrl.searchParams.set("redirect_uri", redirectUri);
authorizationUrl.searchParams.set("scope", "openid profile email");
authorizationUrl.searchParams.set("state", generateState());
authorizationUrl.searchParams.set("code_challenge", generateCodeChallenge(verifier));
authorizationUrl.searchParams.set("code_challenge_method", "S256");
```

### State 参数

为防止 CSRF，必须使用 state 参数：

```typescript
const generateState = (): string => {
  const state = randomBytes(32).toString("hex");
  // 保存到会话中以便后续验证
  // req.session.oauthState = state;
  return state;
};

const verifyState = (receivedState: string, storedState: string): boolean => {
  if (!receivedState || !storedState) {
    return false;
  }
  return timingSafeCompare(receivedState, storedState);
};
```

## 会话固定攻击防御

```typescript
// 登录成功后重新生成会话 ID
const regenerateSession = (req: Request): Promise<void> => {
  return new Promise((resolve, reject) => {
    const oldSession = { ...req.session };
    req.session.regenerate((err) => {
      if (err) {
        reject(err);
        return;
      }
      // 仅将必要数据复制到新会话
      Object.assign(req.session, {
        userId: oldSession.userId,
        role: oldSession.role,
      });
      resolve();
    });
  });
};
```

## 时序安全比较

令牌比较必须使用时序安全函数（防止时序攻击）：

```typescript
import { timingSafeEqual, createHash } from "crypto";

const timingSafeCompare = (a: string, b: string): boolean => {
  const bufA = createHash("sha256").update(a).digest();
  const bufB = createHash("sha256").update(b).digest();
  return timingSafeEqual(bufA, bufB);
};

// WRONG: 普通比较（易受时序攻击）
// if (token === storedToken) { ... }

// CORRECT: 时序安全比较
// if (timingSafeCompare(token, storedToken)) { ... }
```

## 审计日志

### 应记录的事件

| 事件             | 日志级别 | 记录数据                             |
| ---------------- | -------- | ------------------------------------ |
| 登录成功         | INFO     | userId, ip, deviceType, timestamp    |
| 登录失败         | WARN     | email, ip, deviceType, reason        |
| 登出             | INFO     | userId, ip, timestamp                |
| 令牌刷新         | INFO     | userId, ip, deviceType               |
| 令牌吊销         | INFO     | userId, reason, revokedBy            |
| 修改密码         | WARN     | userId, ip, timestamp                |
| 重置密码         | WARN     | userId, ip, timestamp                |
| 权限变更         | WARN     | targetUserId, oldRole, newRole, changedBy |
| 账户锁定         | WARN     | userId, failedAttempts, ip           |
| MFA 启用/禁用    | WARN     | userId, mfaType, timestamp           |
| 非法令牌使用     | ERROR    | tokenHash, ip, deviceType            |
| 重用检测         | ERROR    | userId, tokenFamily, ip              |

### 审计日志实现

```typescript
interface AuditLog {
  eventType: string;
  userId: string | null;
  ip: string;
  deviceType: string;
  metadata: Record<string, unknown>;
  timestamp: Date;
}

const logAuthEvent = async (event: AuditLog): Promise<void> => {
  await db.auditLog.create({
    data: {
      eventType: event.eventType,
      userId: event.userId,
      ipAddress: event.ip,
      deviceType: event.deviceType,
      metadata: event.metadata,
      createdAt: event.timestamp,
    },
  });
};

// 使用示例
await logAuthEvent({
  eventType: "LOGIN_SUCCESS",
  userId: user.id,
  ip: req.ip,
  deviceType: req.headers["x-device-type"] as string,
  metadata: { userAgent: req.headers["user-agent"] },
  timestamp: new Date(),
});
```

## 常见漏洞列表

| 漏洞                           | 影响                             | 防御措施                                   |
| ------------------------------ | -------------------------------- | ------------------------------------------ |
| 明文存储密码                   | DB 泄露导致所有账户被攻破        | 使用 bcrypt/argon2 进行哈希                |
| JWT 保存在 localStorage 中     | XSS 攻击可窃取令牌              | 保存在 httpOnly Cookie 中                  |
| HS256 使用共享密钥             | 密钥泄露可伪造所有令牌           | 使用 RS256/ES256 非对称密钥                |
| 令牌有效期过长                 | 泄露时损害期扩大                 | Access <= 15 分钟，Refresh <= 7 天         |
| 刷新令牌未轮换                 | 被盗令牌可永久使用               | 令牌轮换 + 重用检测                        |
| 暴力破解无防护                 | 密码猜测导致账户被攻破           | 速率限制 + 账户锁定                        |
| 无 CSRF 防御                   | 跨站执行非法操作                 | SameSite Cookie + CSRF 令牌                |
| 时序不安全的比较               | 令牌字符串可被推测               | 使用 timingSafeEqual                       |
| 会话固定                       | 劫持其他用户的会话               | 登录后重新生成会话 ID                      |
| 错误消息泄露信息               | 确认用户存在、暴露内部结构       | 使用通用错误消息                           |
| OAuth2 未使用 PKCE             | 授权码拦截攻击                   | 公共客户端必须启用 PKCE                    |
| OAuth2 未验证 state            | CSRF 劫持账户关联                | 生成并验证 state 参数                      |
| 无审计日志                     | 无法检测和追踪非法访问           | 记录所有认证事件                           |
| Refresh Token 明文存储在 DB    | DB 泄露可重用令牌                | 哈希后存储                                 |
| 权限检查缺失                   | 水平/垂直权限提升                | 所有端点均应用授权中间件                   |
| 响应中泄露角色信息             | 向攻击者提供权限模型信息         | 403 响应中不包含详细信息                   |
