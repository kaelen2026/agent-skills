---
name: security-review
description: "安全审查技能。覆盖 OWASP Top 10、输入验证、认证授权、密钥管理、部署前检查清单。"
metadata:
  filePattern:
    - "**/.env.example"
    - "**/security*"
  bashPattern:
    - "npm audit|snyk|trivy|gitleaks"
  priority: 8
---

# Security Review

## When to Activate

- 实现认证或授权功能
- 处理用户输入或文件上传
- 创建新 API 端点
- 使用密钥或凭证
- 实现支付功能
- 存储或传输敏感数据
- 集成第三方 API
- 部署前安全检查

## Workflow

### 1. 识别安全范围

分析变更涉及的安全领域：

| 领域 | 触发条件 | 参考 |
|------|---------|------|
| 密钥管理 | 使用 API key、token、密码 | [references/secrets-and-input.md](references/secrets-and-input.md) |
| 输入验证 | 接收用户输入、文件上传 | [references/secrets-and-input.md](references/secrets-and-input.md) |
| 注入防护 | 数据库查询、HTML 渲染 | [references/injection-and-xss.md](references/injection-and-xss.md) |
| 认证授权 | 登录、权限、RLS | [references/auth-and-access.md](references/auth-and-access.md) |
| 部署安全 | 上线前检查 | [references/deployment-checklist.md](references/deployment-checklist.md) |

### 2. 执行审查（结论先行）

先输出安全问题总览：

```
## 安全审查结果

发现 N 个安全问题：

| 严重度 | 类别 | 问题 | 位置 |
|--------|------|------|------|
| CRITICAL | 密钥泄露 | 硬编码 API key | src/lib/api.ts:12 |
| HIGH | SQL 注入 | 字符串拼接查询 | src/api/users.ts:45 |
| MEDIUM | XSS | 未转义用户内容 | src/components/Comment.tsx:23 |
```

### 3. 提供修复方案

每个问题附带代码修复示例。

### 4. 部署前检查清单

- [ ] 无硬编码密钥，全部使用环境变量
- [ ] 所有用户输入已用 zod 验证
- [ ] 所有数据库查询使用参数化查询
- [ ] 用户提供的 HTML 已净化
- [ ] CSRF 保护已启用
- [ ] 认证使用 httpOnly cookie → 实现细节参见 [auth](../auth/SKILL.md)
- [ ] 授权检查在所有敏感操作前
- [ ] 所有 API 端点已启用速率限制（审计角色，实现 → 参见 [backend-patterns](../backend-patterns/SKILL.md)）
- [ ] HTTPS 在生产环境强制启用
- [ ] 安全头已配置（CSP, X-Frame-Options）
- [ ] 错误消息不泄露敏感信息
- [ ] 日志不记录密码/token/PII → 详细规范参见 [observability](../observability/SKILL.md)
- [ ] 依赖无已知漏洞（`npm audit`）
- [ ] 文件上传已验证（大小、类型）
- [ ] CORS 已正确配置
- [ ] RLS 已启用（如使用 Supabase）

## Output Format

```markdown
# Security Review: {审查范围}

## 安全问题总览
{问题表格，按严重度排序}

## 修复方案
{每个问题的修复代码示例}

## 部署前检查清单
{检查结果，标记通过/未通过}
```
