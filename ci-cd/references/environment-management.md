---
name: environment-management
version: 1.0.0
description: 环境层级管理、密钥管理与环境一致性策略
last_updated: 2026-03-17
---

# 环境管理

## 环境层级

标准的环境层级从本地开发到生产环境依次为：

```
local → dev → staging → production
```

| 环境 | 用途 | 数据 | 访问范围 |
| --- | --- | --- | --- |
| local | 开发者本地开发与调试 | 模拟数据 / 本地数据库 | 仅开发者本人 |
| dev | 集成开发与联调 | 测试数据 | 开发团队 |
| staging | 预发布验证，尽量模拟生产 | 脱敏的生产数据副本 | 开发 + QA 团队 |
| production | 面向真实用户的生产环境 | 真实数据 | 所有用户 |

### 环境晋升流程

代码和配置通过以下流程逐级晋升：

```
功能分支 ──PR──→ main ──自动──→ staging ──审批──→ production
   │                              │                 │
   └─ 预览部署              自动部署           tag 触发
```

**关键原则**：代码只构建一次，同一构建产物逐级部署到不同环境。环境差异仅通过环境变量控制。

## 环境变量管理

### 按环境层级配置

每个环境维护独立的变量集合，绝不在代码中硬编码：

```
.env.local          ← 本地开发（git ignore）
.env.development    ← dev 环境默认值（可提交）
.env.staging        ← staging 环境（仅在 CI 中注入）
.env.production     ← production 环境（仅在 CI 中注入）
```

### 变量分类

| 类别 | 示例 | 敏感性 | 存储位置 |
| --- | --- | --- | --- |
| 公开配置 | `NEXT_PUBLIC_API_URL` | 低 | `.env` 文件或平台变量 |
| 服务配置 | `DATABASE_URL`、`REDIS_URL` | 高 | 平台密钥管理 |
| 第三方密钥 | `STRIPE_SECRET_KEY` | 极高 | 平台密钥管理 |
| 基础设施 | `AWS_REGION`、`CLUSTER_NAME` | 低 | 平台变量 |

### 本地开发配置

```bash
# .env.local（绝不提交到 Git）
DATABASE_URL=postgresql://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
API_KEY=dev_test_key_not_real

# .env.example（提交到 Git，作为模板）
DATABASE_URL=postgresql://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
API_KEY=your_api_key_here
```

### 启动时验证环境变量

```typescript
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  API_KEY: z.string().min(1),
  NODE_ENV: z.enum(["development", "staging", "production"]),
  PORT: z.coerce.number().default(3000),
});

function validateEnv() {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    const missing = result.error.issues
      .map((i) => `  ${i.path.join(".")}: ${i.message}`)
      .join("\n");
    throw new Error(`环境变量验证失败:\n${missing}`);
  }
  return result.data;
}

export const env = validateEnv();
```

## 密钥管理

### 核心原则

1. **绝不在代码中硬编码密钥** — 即使是测试环境
2. **绝不在日志中输出密钥** — 即使是 debug 级别
3. **定期轮换密钥** — 至少每 90 天
4. **最小权限原则** — 每个环境使用独立密钥，权限仅限所需

### GitHub Secrets

```yaml
# 在 GitHub Actions 中使用密钥
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 构建
        run: pnpm build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: 部署
        run: pnpm deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

按环境分离密钥：

```yaml
# 使用 GitHub Environments 功能
jobs:
  deploy-staging:
    environment: staging  # 使用 staging 环境的密钥
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.DATABASE_URL }}"
        # 这里的 DATABASE_URL 是 staging 环境特定的值

  deploy-production:
    environment: production  # 使用 production 环境的密钥
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.DATABASE_URL }}"
        # 这里的 DATABASE_URL 是 production 环境特定的值
```

### Vercel 环境变量

```bash
# 通过 Vercel CLI 设置不同环境的变量
vercel env add DATABASE_URL production
vercel env add DATABASE_URL preview
vercel env add DATABASE_URL development

# 批量导入
vercel env pull .env.local  # 拉取开发环境变量到本地
```

Vercel 环境变量分类：

| 类型 | 可见范围 | 用途 |
| --- | --- | --- |
| Production | 仅生产部署 | 生产密钥 |
| Preview | PR 预览部署 | 测试密钥 |
| Development | `vercel dev` 本地开发 | 开发密钥 |

### AWS Secrets Manager

```typescript
import {
  SecretsManagerClient,
  GetSecretValueCommand,
} from "@aws-sdk/client-secrets-manager";

async function getSecret(secretName: string): Promise<string> {
  const client = new SecretsManagerClient({ region: "ap-northeast-2" });

  try {
    const response = await client.send(
      new GetSecretValueCommand({ SecretId: secretName })
    );
    if (response.SecretString === undefined) {
      throw new Error(`密钥 ${secretName} 的值为空`);
    }
    return response.SecretString;
  } catch (error) {
    throw new Error(`获取密钥 ${secretName} 失败: ${error}`);
  }
}

// 使用示例
const dbCredentials = JSON.parse(
  await getSecret("production/database")
);
```

AWS Secrets Manager 的优势：

- 支持自动密钥轮换
- 精细的 IAM 权限控制
- 完整的访问审计日志
- 跨区域复制

### 密钥管理工具对比

| 工具 | 适用场景 | 密钥轮换 | 费用 |
| --- | --- | --- | --- |
| GitHub Secrets | GitHub Actions 流水线 | 手动 | 免费 |
| Vercel 环境变量 | Vercel 部署 | 手动 | 免费 |
| AWS Secrets Manager | AWS 基础设施 | 自动 | 按使用量计费 |
| HashiCorp Vault | 混合云/多云 | 自动 | 开源版免费 |
| Doppler | 跨平台密钥同步 | 手动 | 免费层可用 |
| 1Password (Secrets Automation) | 小团队通用方案 | 手动 | 按用户计费 |

## 环境一致性

### 核心原则

staging 环境与 production 环境的差异越小，生产事故的风险越低。

**保持一致的方面**：
- 操作系统与运行时版本
- 数据库类型与版本
- 第三方服务（缓存、消息队列、搜索引擎）
- 网络拓扑（负载均衡、CDN 配置）
- 部署流程（使用相同的流水线和脚本）

**允许不同的方面**：
- 实例数量（staging 可以更少）
- 实例规格（staging 可以用更小规格）
- 域名与 SSL 证书
- 第三方服务的 API 密钥（使用各自环境的密钥）

### Docker 确保环境一致性

```dockerfile
# Dockerfile — 所有环境使用同一镜像
FROM node:20-alpine AS base
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

FROM base AS builder
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# 环境差异仅通过运行时环境变量注入
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### 基础设施即代码（IaC）

使用代码定义基础设施，确保环境之间的一致性：

```typescript
// Pulumi 示例 — 用代码定义基础设施
import * as aws from "@pulumi/aws";

function createEnvironment(name: string, config: EnvConfig) {
  const db = new aws.rds.Instance(`${name}-db`, {
    engine: "postgres",
    engineVersion: "16",
    instanceClass: config.dbInstanceClass,
    allocatedStorage: config.dbStorage,
  });

  const service = new aws.ecs.Service(`${name}-service`, {
    desiredCount: config.instanceCount,
    taskDefinition: taskDef.arn,
  });

  return { db, service };
}

// staging 和 production 使用相同的架构定义
const staging = createEnvironment("staging", {
  dbInstanceClass: "db.t3.medium",
  dbStorage: 20,
  instanceCount: 1,
});

const production = createEnvironment("production", {
  dbInstanceClass: "db.r6g.large",
  dbStorage: 100,
  instanceCount: 3,
});
```

## 数据库按环境分离策略

### 核心原则

每个环境使用独立的数据库实例，绝不共享：

```
local:      本地 PostgreSQL / SQLite
dev:        共享开发数据库
staging:    独立 staging 数据库（脱敏数据）
production: 生产数据库
```

### 数据库迁移流程

迁移脚本随代码一起晋升，在部署流水线中自动执行：

```yaml
# 部署流水线中的数据库迁移
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 运行数据库迁移
        run: pnpm db:migrate
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: 验证迁移结果
        run: pnpm db:status

      - name: 部署应用
        run: pnpm deploy
```

### Staging 数据管理

```bash
# 从生产环境导出脱敏数据到 staging
pg_dump --no-owner production_db | \
  sed 's/real_email@/test_email@/g' | \
  psql staging_db

# 更好的方案：使用专用数据脱敏工具
# 例如 PostgreSQL Anonymizer 扩展
```

**注意事项**：
- staging 数据库绝不使用真实用户的敏感数据
- 定期从生产同步脱敏数据以保持数据结构一致
- 迁移脚本必须支持幂等执行（重复执行不报错）

## 特性分支预览部署

### 核心概念

每个特性分支（PR）自动创建独立的预览环境，便于代码审查和测试。

### Vercel 预览部署

Vercel 默认支持预览部署，无需额外配置：

```
PR #42 → https://my-app-pr-42.vercel.app
PR #43 → https://my-app-pr-43.vercel.app
```

自定义预览部署行为：

```json
// vercel.json
{
  "github": {
    "autoAlias": true,
    "autoJobCancelation": true
  }
}
```

### GitHub Actions 预览部署

```yaml
# .github/workflows/preview.yml
name: Preview Deployment

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 部署预览环境
        id: deploy
        run: |
          preview_url=$(deploy --preview --branch ${{ github.head_ref }})
          echo "url=$preview_url" >> "$GITHUB_OUTPUT"

      - name: 评论 PR
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `预览环境已就绪: ${{ steps.deploy.outputs.url }}`
            });

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: 清理预览环境
        run: destroy-preview --branch ${{ github.head_ref }}
```

### 预览环境的数据库处理

| 策略 | 优点 | 缺点 | 适用场景 |
| --- | --- | --- | --- |
| 共享 dev 数据库 | 简单 | 数据冲突风险 | 只读场景 |
| 独立数据库实例 | 完全隔离 | 费用高 | 写入密集型应用 |
| 数据库分支 (Neon/PlanetScale) | 隔离且快速 | 依赖特定服务商 | 推荐方案 |
| SQLite 内存数据库 | 零费用 | 功能受限 | 简单应用 |

### 使用 Neon 数据库分支

```yaml
- name: 创建数据库分支
  run: |
    neon branches create \
      --project-id ${{ vars.NEON_PROJECT_ID }} \
      --name "pr-${{ github.event.pull_request.number }}" \
      --parent main
  env:
    NEON_API_KEY: ${{ secrets.NEON_API_KEY }}
```

每个 PR 获得独立的数据库分支，合并后自动清理。

## 环境晋升流程

### 标准晋升流程

```
开发者本地验证
    ↓
PR 创建 → 预览部署 → 代码审查 + 自动化测试
    ↓
合并到 main → 自动部署到 staging
    ↓
QA 在 staging 验证
    ↓
创建 release tag → 触发 production 部署（需审批）
    ↓
部署后监控 → 确认稳定
```

### 晋升检查门禁

每次晋升前必须通过的检查：

```yaml
# staging → production 晋升条件
jobs:
  promote-to-production:
    runs-on: ubuntu-latest
    steps:
      - name: 检查 staging 测试结果
        run: |
          result=$(get-test-results --env staging)
          if [ "$result" != "pass" ]; then
            echo "staging 测试未通过，禁止晋升"
            exit 1
          fi

      - name: 检查 staging 运行稳定性
        run: |
          error_rate=$(get-error-rate --env staging --duration 24h)
          if (( $(echo "$error_rate > 0.01" | bc -l) )); then
            echo "staging 错误率过高: $error_rate，禁止晋升"
            exit 1
          fi

      - name: 等待人工审批
        uses: trstringer/manual-approval@v1
        with:
          approvers: tech-lead,devops-team
          minimum-approvals: 1

      - name: 部署到 production
        run: deploy --env production --version ${{ github.sha }}
```

### 紧急热修复流程

紧急情况下可跳过部分步骤，但必须事后补齐：

```
热修复分支 → PR（标记 hotfix） → 快速审查
    ↓
合并 → 同时部署 staging + production
    ↓
事后验证 → 补充测试 → 事后复盘
```

**注意**：即使是紧急热修复，也必须经过至少一人的代码审查和自动化测试。
