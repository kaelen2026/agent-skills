# 认证、授权与访问控制

## Token 存储

```typescript
// BAD: localStorage（易受 XSS 攻击）
localStorage.setItem("token", token);

// GOOD: httpOnly cookie
res.setHeader(
  "Set-Cookie",
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`
);
```

### Cookie 安全标志

| 标志 | 作用 | 必须 |
|------|------|------|
| `HttpOnly` | JS 无法访问 cookie | 是 |
| `Secure` | 仅 HTTPS 传输 | 是（生产环境） |
| `SameSite=Strict` | 禁止跨站请求携带 | 是 |
| `Max-Age` | 过期时间 | 是 |
| `Path=/` | Cookie 作用路径 | 推荐 |

## 授权检查

### 操作前始终验证权限

```typescript
export async function deleteUser(userId: string, requesterId: string) {
  const requester = await db.users.findUnique({
    where: { id: requesterId },
  });

  if (requester.role !== "admin") {
    return NextResponse.json(
      {
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "Admin access required",
        },
        meta: { requestId },
      },
      { status: 403 }
    );
  }

  await db.users.delete({ where: { id: userId } });
}
```

### 资源所有权检查

```typescript
export async function updatePost(
  postId: string,
  userId: string,
  data: UpdatePostInput
) {
  const post = await db.posts.findUnique({ where: { id: postId } });

  if (!post) {
    return { success: false, error: { code: "RESOURCE_NOT_FOUND" } };
  }

  // 所有权检查：只有作者或管理员可以修改
  if (post.authorId !== userId && !isAdmin(userId)) {
    return { success: false, error: { code: "INSUFFICIENT_PERMISSIONS" } };
  }

  return db.posts.update({ where: { id: postId }, data });
}
```

## Row Level Security (Supabase)

```sql
-- 所有表启用 RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- 用户只能查看自己的数据
CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- 用户只能更新自己的数据
CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);

-- 管理员可以查看所有数据
CREATE POLICY "Admins view all data"
  ON users FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM user_roles
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );
```

## 授权检查清单

- [ ] Token 存储在 httpOnly cookie（非 localStorage）
- [ ] 所有敏感操作前有授权检查
- [ ] 资源操作验证所有权
- [ ] RBAC 已实现
- [ ] RLS 已在数据库层启用（如使用 Supabase）
- [ ] 会话管理安全（过期、续期、注销）

## 安全测试

### 认证测试

```typescript
test("requires authentication", async () => {
  const response = await fetch("/api/protected");
  expect(response.status).toBe(401);
});

test("requires admin role", async () => {
  const response = await fetch("/api/admin", {
    headers: { Authorization: `Bearer ${userToken}` },
  });
  expect(response.status).toBe(403);
});
```

### 输入验证测试

```typescript
test("rejects invalid input", async () => {
  const response = await fetch("/api/users", {
    method: "POST",
    body: JSON.stringify({ email: "not-an-email" }),
  });
  expect(response.status).toBe(400);
});
```

### 速率限制测试

```typescript
test("enforces rate limits", async () => {
  const requests = Array(101)
    .fill(null)
    .map(() => fetch("/api/endpoint"));

  const responses = await Promise.all(requests);
  const tooMany = responses.filter((r) => r.status === 429);
  expect(tooMany.length).toBeGreaterThan(0);
});
```
