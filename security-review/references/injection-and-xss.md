# 注入防护与 XSS 防御

## SQL 注入

### 绝对禁止：字符串拼接

```typescript
// BAD: SQL 注入漏洞
const query = `SELECT * FROM users WHERE email = '${userEmail}'`;
await db.query(query);
```

### 正确方式：参数化查询

```typescript
// GOOD: ORM / Query Builder
const { data } = await supabase
  .from("users")
  .select("*")
  .eq("email", userEmail);

// GOOD: 参数化原始 SQL
await db.query("SELECT * FROM users WHERE email = $1", [userEmail]);

// GOOD: Prisma
const user = await prisma.user.findUnique({
  where: { email: userEmail },
});
```

### SQL 注入检查清单

- [ ] 所有数据库查询使用参数化查询
- [ ] 无 SQL 字符串拼接
- [ ] ORM / Query Builder 使用正确
- [ ] 动态表名/列名使用白名单验证

## XSS（跨站脚本）防护

### 净化用户 HTML

```typescript
import DOMPurify from "isomorphic-dompurify";

function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "p", "a"],
    ALLOWED_ATTR: ["href"],
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### React 内置防护

React 默认转义 JSX 中的变量，但以下情况例外：

```typescript
// 安全: React 自动转义
<p>{userInput}</p>

// 危险: 绕过了 React 的转义
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // 必须先净化！

// 危险: href 中的 javascript:
<a href={userInput}>Link</a>  // 必须验证 URL 协议
```

### URL 验证

```typescript
function isSafeUrl(url: string): boolean {
  try {
    const parsed = new URL(url);
    return ["http:", "https:"].includes(parsed.protocol);
  } catch {
    return false;
  }
}
```

### Content Security Policy (CSP)

```typescript
// next.config.js
const securityHeaders = [
  {
    key: "Content-Security-Policy",
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-eval' 'unsafe-inline'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' https://api.example.com",
    ].join("; "),
  },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=()" },
];
```

### XSS 检查清单

- [ ] 用户提供的 HTML 使用 DOMPurify 净化
- [ ] CSP 头已配置
- [ ] 无未验证的 `dangerouslySetInnerHTML`
- [ ] URL 协议已验证（防止 `javascript:` 注入）
- [ ] 安全响应头已配置（X-Frame-Options, X-Content-Type-Options）

## CSRF（跨站请求伪造）

### SameSite Cookie

```typescript
res.setHeader(
  "Set-Cookie",
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`
);
```

### CSRF Token

```typescript
import { csrf } from "@/lib/csrf";

export async function POST(request: Request) {
  const token = request.headers.get("X-CSRF-Token");

  if (!csrf.verify(token)) {
    return NextResponse.json(
      {
        success: false,
        error: { code: "INVALID_CSRF_TOKEN", message: "Invalid CSRF token" },
      },
      { status: 403 }
    );
  }

  // 处理请求...
}
```

### CSRF 检查清单

- [ ] 所有 Cookie 设置 `SameSite=Strict`
- [ ] 状态变更操作使用 CSRF token
- [ ] 使用 Double Submit Cookie 模式或 Synchronizer Token

## 速率限制

### 全局限制

```typescript
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 每窗口 100 次
  message: {
    success: false,
    error: { code: "RATE_LIMIT_EXCEEDED", message: "Too many requests" },
  },
});

app.use("/api/", limiter);
```

### 高代价操作限制

```typescript
// 搜索端点更严格的限制
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 分钟
  max: 10, // 每分钟 10 次
});

app.use("/api/search", searchLimiter);

// 登录端点防暴力破解
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 15 分钟内最多 5 次
});

app.use("/api/auth/login", loginLimiter);
```

### 速率限制检查清单

- [ ] 所有 API 端点有速率限制
- [ ] 高代价操作有更严格的限制
- [ ] 登录端点有防暴力破解限制
- [ ] 返回 `Retry-After` 头
