# 部署前安全检查清单

## 检查清单

在**任何生产部署**前逐项确认：

### 密钥管理

- [ ] 无硬编码密钥，全部使用环境变量
- [ ] `.env*` 文件已加入 `.gitignore`
- [ ] git 历史中无密钥泄露
- [ ] 生产密钥存储在托管平台的密钥管理服务
- [ ] 未设置的必需密钥在启动时 throw

### 输入与数据

- [ ] 所有用户输入使用 zod 验证
- [ ] 文件上传已限制大小和类型
- [ ] 所有数据库查询使用参数化查询
- [ ] 无 SQL 字符串拼接

### 认证与授权

- [ ] Token 存储在 httpOnly cookie
- [ ] Cookie 设置 Secure + SameSite=Strict
- [ ] 所有敏感操作前有授权检查
- [ ] RBAC / RLS 已启用
- [ ] 会话有过期时间

### XSS / CSRF

- [ ] 用户 HTML 使用 DOMPurify 净化
- [ ] CSP 头已配置
- [ ] 安全响应头已设置（X-Frame-Options, X-Content-Type-Options）
- [ ] CSRF 保护已启用
- [ ] 无未验证的 `dangerouslySetInnerHTML`

### API 安全

- [ ] 所有端点有速率限制
- [ ] 登录端点有暴力破解防护
- [ ] HTTPS 在生产环境强制
- [ ] CORS 仅允许信任的来源
- [ ] 错误消息不泄露内部信息
- [ ] 日志不记录敏感数据

### 依赖安全

- [ ] `npm audit` 无已知漏洞
- [ ] lock 文件已提交
- [ ] Dependabot / Renovate 已启用
- [ ] 无不必要的依赖

## 依赖安全命令

```bash
# 检查漏洞
npm audit

# 自动修复
npm audit fix

# 检查过期依赖
npm outdated

# CI 中使用 ci 而非 install（确保可重现构建）
npm ci
```

## 安全响应头模板

```typescript
// next.config.js
const headers = [
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=()" },
  {
    key: "Content-Security-Policy",
    value: [
      "default-src 'self'",
      "script-src 'self'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "connect-src 'self' https://api.example.com",
    ].join("; "),
  },
  {
    key: "Strict-Transport-Security",
    value: "max-age=63072000; includeSubDomains; preload",
  },
];
```

## 参考资源

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js Security](https://nextjs.org/docs/security)
- [Supabase Security](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)
