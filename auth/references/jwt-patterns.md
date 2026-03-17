# JWT Patterns

## JWT 结构

JWT 由 3 个部分组成（以点号连接）：

```
header.payload.signature
```

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

| 算法       | 用途               | 推荐度 |
| ---------- | ------------------ | ------ |
| RS256      | 非对称密钥（推荐） | 推荐   |
| ES256      | 椭圆曲线           | 推荐   |
| HS256      | 对称密钥           | 不推荐 |

- RS256/ES256: 用私钥签名，用公钥验证。最适合微服务架构
- HS256: 同一密钥进行签名和验证。仅允许单一服务使用

### Payload（令牌载荷）

所有属性使用 **camelCase**：

```json
{
  "sub": "user_abc123",
  "role": "admin",
  "deviceType": "web",
  "iat": 1710000000,
  "exp": 1710000900
}
```

| 字段         | 类型   | 必须 | 说明                              |
| ------------ | ------ | ---- | --------------------------------- |
| `sub`        | string | Yes  | 用户 ID                           |
| `role`       | string | Yes  | 用户角色                          |
| `deviceType` | string | Yes  | `web` / `app` / `desktop`        |
| `iat`        | number | Yes  | 签发时间（Unix timestamp）        |
| `exp`        | number | Yes  | 过期时间（Unix timestamp）        |
| `jti`        | string | No   | 令牌唯一 ID（用于吊销管理）       |
| `permissions`| array  | No   | 细粒度权限（ABAC 使用时）         |

### Signature

```
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  privateKey
)
```

## Access Token + Refresh Token 模式

双令牌方式兼顾安全性与用户体验：

| 令牌           | 有效期   | 用途                   | 存储位置               |
| -------------- | -------- | ---------------------- | ---------------------- |
| Access Token   | <= 15 分钟 | API 请求认证          | 因设备类型而异         |
| Refresh Token  | <= 7 天  | 重新签发 Access Token  | 因设备类型而异         |

## 令牌存储位置（按设备类型）

根据 `X-Device-Type` 头决定存储位置：

### web: httpOnly Secure Cookie

```typescript
// Access Token
res.cookie("accessToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  path: "/",
  maxAge: 15 * 60 * 1000, // 15 分钟
});

// Refresh Token
res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  path: "/v1/auth/refresh", // 仅限刷新端点
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 天
});
```

### app: Secure Storage（Keychain / Keystore）

```typescript
// iOS: Keychain Services
// Android: Android Keystore
// React Native: react-native-keychain
// Flutter: flutter_secure_storage

// 客户端存储
await SecureStorage.set("accessToken", token);
await SecureStorage.set("refreshToken", refreshToken);

// 请求时通过 Authorization 头发送
headers: {
  Authorization: `Bearer ${await SecureStorage.get("accessToken")}`,
  "X-Device-Type": "app",
}
```

### desktop: OS Credential Store

```typescript
// Electron: safeStorage / keytar
// macOS: Keychain
// Windows: Credential Manager
// Linux: libsecret

import keytar from "keytar";

await keytar.setPassword("myapp", "accessToken", token);
await keytar.setPassword("myapp", "refreshToken", refreshToken);
```

## 令牌刷新流程

```
Client                          Server
  |                               |
  |-- API Request (Access Token) -->
  |                               |
  |<-- 401 TOKEN_EXPIRED ---------
  |                               |
  |-- POST /v1/auth/refresh ----->
  |   (Refresh Token)            |
  |                               |-- 验证 Refresh Token
  |                               |-- 使旧 Refresh Token 失效
  |                               |-- 签发新 Access Token
  |                               |-- 签发新 Refresh Token（轮换）
  |<-- New Access + Refresh ------
  |                               |
  |-- Retry API Request ---------->
  |   (New Access Token)          |
  |                               |
```

### 刷新端点实现

```typescript
const refreshTokenHandler = async (req: Request, res: Response) => {
  const { refreshToken } = req.cookies; // web 的场景
  // app/desktop 的场景使用 req.body.refreshToken

  if (!refreshToken) {
    return res.status(401).json({
      success: false,
      error: {
        code: "AUTHENTICATION_REQUIRED",
        message: "Refresh token is required",
      },
      meta: { requestId: req.requestId },
    });
  }

  try {
    const payload = await verifyRefreshToken(refreshToken);

    // 令牌轮换: 使旧令牌失效
    await revokeRefreshToken(refreshToken);

    // 签发新令牌对
    const newAccessToken = generateAccessToken({
      sub: payload.sub,
      role: payload.role,
      deviceType: payload.deviceType,
    });
    const newRefreshToken = generateRefreshToken({
      sub: payload.sub,
      deviceType: payload.deviceType,
    });

    // 根据设备类型返回
    if (payload.deviceType === "web") {
      setTokenCookies(res, newAccessToken, newRefreshToken);
    }

    return res.json({
      success: true,
      data: {
        accessToken: newAccessToken,
        refreshToken: newRefreshToken,
        expiresAt: new Date(Date.now() + 15 * 60 * 1000).toISOString(),
      },
      meta: { requestId: req.requestId },
    });
  } catch (error) {
    // 刷新令牌无效 → 使所有会话失效（安全措施）
    if (error instanceof TokenReuseError) {
      await revokeAllUserTokens(error.userId);
    }

    return res.status(401).json({
      success: false,
      error: {
        code: "INVALID_TOKEN",
        message: "Refresh token is invalid or expired",
      },
      meta: { requestId: req.requestId },
    });
  }
};
```

## 令牌轮换

通过刷新令牌的重用检测来增强安全性：

```
1. 用户发起刷新 → 签发新令牌对，使旧刷新令牌失效
2. 攻击者使用被盗的旧刷新令牌 → 检测到重用
3. 使所有刷新令牌失效（用户需要重新登录）
```

### 令牌族管理

```typescript
interface RefreshTokenRecord {
  tokenHash: string;       // 令牌的哈希（禁止明文存储）
  userId: string;
  familyId: string;        // 令牌族 ID
  isRevoked: boolean;
  createdAt: Date;
  expiresAt: Date;
}

const rotateRefreshToken = async (oldToken: string): Promise<TokenPair> => {
  const record = await findRefreshToken(hashToken(oldToken));

  if (!record) {
    throw new InvalidTokenError();
  }

  // 重用检测: 已吊销的令牌被使用
  if (record.isRevoked) {
    await revokeTokenFamily(record.familyId);
    throw new TokenReuseError(record.userId);
  }

  // 使旧令牌失效
  await revokeRefreshToken(record.tokenHash);

  // 在同一族中签发新令牌
  const newRefreshToken = generateRefreshToken({
    sub: record.userId,
    familyId: record.familyId,
  });

  return { accessToken: generateAccessToken({ sub: record.userId }), refreshToken: newRefreshToken };
};
```

## 令牌吊销（Revocation）

### 黑名单方式

```typescript
// Redis 管理黑名单
const revokeToken = async (jti: string, expiresAt: number): Promise<void> => {
  const ttl = expiresAt - Math.floor(Date.now() / 1000);
  if (ttl > 0) {
    await redis.set(`blacklist:${jti}`, "1", "EX", ttl);
  }
};

const isTokenRevoked = async (jti: string): Promise<boolean> => {
  const result = await redis.get(`blacklist:${jti}`);
  return result !== null;
};
```

### 白名单方式

```typescript
// 在 DB 中管理活跃会话（更安全但负载较高）
const isTokenValid = async (jti: string): Promise<boolean> => {
  const session = await db.session.findUnique({ where: { tokenId: jti } });
  return session !== null && !session.isRevoked;
};
```

| 方式       | 优点                 | 缺点                     | 推荐场景                    |
| ---------- | -------------------- | ------------------------ | --------------------------- |
| 黑名单     | 验证轻量             | 存在吊销遗漏风险         | 一般 Web 应用               |
| 白名单     | 可靠的吊销管理       | DB 负载                  | 金融、医疗等高安全性场景    |

## 令牌验证中间件

```typescript
import jwt from "jsonwebtoken";
import { timingSafeEqual } from "crypto";

const authenticate = async (
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> => {
  // 根据设备类型获取令牌
  const deviceType = req.headers["x-device-type"] as string;
  let token: string | undefined;

  if (deviceType === "web") {
    token = req.cookies?.accessToken;
  } else {
    const authHeader = req.headers.authorization;
    token = authHeader?.startsWith("Bearer ") ? authHeader.slice(7) : undefined;
  }

  if (!token) {
    res.status(401).json({
      success: false,
      error: {
        code: "AUTHENTICATION_REQUIRED",
        message: "Authentication is required to access this resource",
      },
      meta: { requestId: req.requestId },
    });
    return;
  }

  try {
    const payload = jwt.verify(token, publicKey, {
      algorithms: ["RS256"],
      issuer: "your-app",
      audience: "your-api",
    }) as TokenPayload;

    // 黑名单检查
    if (payload.jti && await isTokenRevoked(payload.jti)) {
      res.status(401).json({
        success: false,
        error: {
          code: "INVALID_TOKEN",
          message: "Token has been revoked",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    // 确认设备类型匹配
    if (payload.deviceType !== deviceType) {
      res.status(401).json({
        success: false,
        error: {
          code: "INVALID_TOKEN",
          message: "Token device type mismatch",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    req.user = payload;
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      res.status(401).json({
        success: false,
        error: {
          code: "TOKEN_EXPIRED",
          message: "Access token has expired",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    res.status(401).json({
      success: false,
      error: {
        code: "INVALID_TOKEN",
        message: "Token is invalid",
      },
      meta: { requestId: req.requestId },
    });
  }
};
```

## 认证响应示例

### 登录成功

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIs...",
    "refreshToken": "dGhpcyBpcyBhIHJlZnJl...",
    "expiresAt": "2026-03-17T12:15:00Z",
    "user": {
      "id": "user_abc123",
      "email": "alice@example.com",
      "role": "admin",
      "lastLoginAt": "2026-03-17T12:00:00Z"
    }
  },
  "meta": {
    "requestId": "req_abc123"
  }
}
```

### 令牌刷新成功

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIs...",
    "refreshToken": "bmV3IHJlZnJlc2ggdG9r...",
    "expiresAt": "2026-03-17T12:30:00Z"
  },
  "meta": {
    "requestId": "req_def456"
  }
}
```

## 错误响应

### AUTHENTICATION_REQUIRED（401）

```json
{
  "success": false,
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Authentication is required to access this resource"
  },
  "meta": {
    "requestId": "req_err001"
  }
}
```

### INVALID_TOKEN（401）

```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Token is invalid or has been revoked"
  },
  "meta": {
    "requestId": "req_err002"
  }
}
```

### TOKEN_EXPIRED（401）

```json
{
  "success": false,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Access token has expired, please refresh"
  },
  "meta": {
    "requestId": "req_err003"
  }
}
```

## 令牌生成工具

```typescript
import jwt from "jsonwebtoken";
import { randomBytes } from "crypto";

interface AccessTokenPayload {
  sub: string;
  role: string;
  deviceType: "web" | "app" | "desktop";
}

const generateAccessToken = (payload: AccessTokenPayload): string => {
  return jwt.sign(
    {
      ...payload,
      jti: randomBytes(16).toString("hex"),
    },
    privateKey,
    {
      algorithm: "RS256",
      expiresIn: "15m",
      issuer: "your-app",
      audience: "your-api",
    }
  );
};

const generateRefreshToken = (payload: {
  sub: string;
  deviceType: string;
  familyId?: string;
}): string => {
  const token = randomBytes(32).toString("base64url");

  // 哈希后保存到 DB（禁止明文存储）
  // await saveRefreshToken({
  //   tokenHash: hashToken(token),
  //   userId: payload.sub,
  //   familyId: payload.familyId ?? randomBytes(16).toString("hex"),
  //   expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  // });

  return token;
};
```
