# 密钥管理与输入验证

## 密钥管理

### 绝对禁止

```typescript
// BAD: 硬编码密钥
const apiKey = "sk-proj-xxxxx";
const dbPassword = "password123";
```

### 正确方式

```typescript
// GOOD: 环境变量 + 启动时检查
const apiKey = process.env.OPENAI_API_KEY;
const dbUrl = process.env.DATABASE_URL;

if (!apiKey) {
  throw new Error("OPENAI_API_KEY not configured");
}
```

### 密钥检查清单

- [ ] 无硬编码 API key、token、密码
- [ ] 所有密钥通过环境变量读取
- [ ] `.env.local` 已加入 `.gitignore`
- [ ] git 历史中无密钥泄露
- [ ] 生产密钥存储在托管平台（Vercel、Railway、AWS Secrets Manager）
- [ ] 未设置的必需密钥在启动时立即 throw

### 密钥泄露应急

如果密钥已提交到 git：

1. **立即轮换密钥**（在提供商处生成新 key）
2. 更新环境变量
3. 使用 `git-filter-repo` 或 BFG 从历史中删除
4. 审查是否有未授权使用

## 输入验证

### 始终使用 zod 验证

```typescript
import { z } from "zod";

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150),
});

export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input);
    return await db.users.create(validated);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return {
        success: false,
        error: {
          code: "VALIDATION_ERROR",
          message: "Input validation failed",
          details: error.errors,
        },
        meta: { requestId: req.requestId },
      };
    }
    throw error;
  }
}
```

### 文件上传验证

```typescript
function validateFileUpload(file: File) {
  // 大小检查（5MB 上限）
  const MAX_SIZE = 5 * 1024 * 1024;
  if (file.size > MAX_SIZE) {
    throw new Error("File too large (max 5MB)");
  }

  // MIME 类型白名单
  const ALLOWED_TYPES = ["image/jpeg", "image/png", "image/gif"];
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new Error("Invalid file type");
  }

  // 扩展名检查
  const ALLOWED_EXTENSIONS = [".jpg", ".jpeg", ".png", ".gif"];
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0];
  if (!extension || !ALLOWED_EXTENSIONS.includes(extension)) {
    throw new Error("Invalid file extension");
  }

  return true;
}
```

### 输入验证规则

| 规则 | 说明 |
|------|------|
| 白名单优于黑名单 | 明确允许的值，拒绝其他一切 |
| 在系统边界验证 | API 入口处验证，内部函数可信任 |
| 不信任客户端验证 | 前端验证是 UX，后端验证是安全 |
| 错误消息不泄露内部信息 | 不返回数据库列名、文件路径、堆栈跟踪 |

## 敏感数据暴露

### 日志

```typescript
// BAD: 记录敏感数据
console.log("User login:", { email, password });
console.log("Payment:", { cardNumber, cvv });

// GOOD: 脱敏后记录
console.log("User login:", { email, userId });
console.log("Payment:", { last4: card.last4, userId });
```

### 错误消息

```typescript
// BAD: 暴露内部细节
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  );
}

// GOOD: 通用错误消息
catch (error) {
  console.error("Internal error:", error);
  return NextResponse.json(
    {
      success: false,
      error: { code: "INTERNAL_ERROR", message: "An error occurred" },
      meta: { requestId }
    },
    { status: 500 }
  );
}
```

### 禁止记录的数据

| 类别 | 示例 |
|------|------|
| 密码 | 明文密码、密码哈希 |
| Token | JWT、API key、refresh token |
| PII | 身份证号、手机号完整格式、信用卡号 |
| 内部路径 | 服务器文件路径、堆栈跟踪 |
