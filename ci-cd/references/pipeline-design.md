---
name: pipeline-design
version: 1.0.0
description: CI/CD 流水线设计模式与 GitHub Actions 工作流示例
last_updated: 2026-03-17
---

# 流水线设计

## 流水线阶段

标准 CI/CD 流水线遵循以下阶段顺序：

```
lint → type-check → test → build → deploy
```

每个阶段的职责：

| 阶段 | 职责 | 失败影响 |
| --- | --- | --- |
| lint | 代码风格与规范检查 | 阻止合并 |
| type-check | 类型安全验证 | 阻止合并 |
| test | 单元测试与集成测试 | 阻止合并 |
| build | 编译与打包 | 阻止部署 |
| deploy | 部署到目标环境 | 触发回滚 |

## GitHub Actions 工作流示例

### PR 检查工作流（lint + test + build）

每次 PR 创建或更新时运行，确保代码质量：

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm type-check

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm test -- --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/

  build:
    runs-on: ubuntu-latest
    needs: [lint, type-check, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

### Staging 部署工作流（合并到 main 时触发）

合并到 main 分支后自动部署到 staging 环境：

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
          API_KEY: ${{ secrets.STAGING_API_KEY }}

      # 以 Vercel 为例
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}

      - name: 健康检查
        run: |
          for i in $(seq 1 10); do
            status=$(curl -s -o /dev/null -w '%{http_code}' "$STAGING_URL/api/health")
            if [ "$status" = "200" ]; then
              echo "健康检查通过"
              exit 0
            fi
            echo "等待服务启动... ($i/10)"
            sleep 10
          done
          echo "健康检查失败"
          exit 1
        env:
          STAGING_URL: ${{ vars.STAGING_URL }}
```

### Production 部署工作流（tag/release 触发）

创建版本 tag 后部署到生产环境，需要手动审批：

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    tags:
      - "v*"

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm test

  deploy:
    runs-on: ubuntu-latest
    needs: [test]
    environment: production  # 在 GitHub Settings 中配置需要审批
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
          API_KEY: ${{ secrets.PRODUCTION_API_KEY }}

      - name: 部署到生产环境
        run: pnpm deploy:production
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}

      - name: 健康检查
        run: |
          for i in $(seq 1 10); do
            status=$(curl -s -o /dev/null -w '%{http_code}' "$PRODUCTION_URL/api/health")
            if [ "$status" = "200" ]; then
              echo "生产环境健康检查通过"
              exit 0
            fi
            echo "等待服务启动... ($i/10)"
            sleep 15
          done
          echo "生产环境健康检查失败"
          exit 1
        env:
          PRODUCTION_URL: ${{ vars.PRODUCTION_URL }}

      - name: 创建部署记录
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.repos.createDeploymentStatus({
              owner: context.repo.owner,
              repo: context.repo.repo,
              deployment_id: context.payload.deployment?.id,
              state: 'success',
              environment_url: process.env.PRODUCTION_URL
            });
```

## 缓存策略

合理的缓存能显著缩短流水线执行时间。

### 依赖缓存（node_modules）

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: "pnpm"  # 自动缓存 pnpm store
```

自定义缓存路径：

```yaml
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      .pnpm-store
    key: deps-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: |
      deps-${{ runner.os }}-
```

### 构建缓存

```yaml
# Next.js 构建缓存
- uses: actions/cache@v4
  with:
    path: .next/cache
    key: nextjs-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('src/**') }}
    restore-keys: |
      nextjs-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-
      nextjs-${{ runner.os }}-
```

### Docker 层缓存

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE_TAG }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## 并行任务（Matrix 策略）

### 多版本并行测试

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
      fail-fast: false
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

### 多平台并行构建

```yaml
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - run: pnpm build
```

### 独立任务并行执行

lint、type-check、test 三个任务互不依赖，应并行执行：

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  type-check:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: [lint, type-check, test]  # 等待所有检查通过
    runs-on: ubuntu-latest
    steps: [...]
```

## 条件执行

### 路径过滤

仅当特定路径的文件发生变更时触发：

```yaml
on:
  pull_request:
    paths:
      - "src/**"
      - "package.json"
      - "pnpm-lock.yaml"
    paths-ignore:
      - "docs/**"
      - "*.md"
      - ".vscode/**"
```

### 分支过滤

```yaml
on:
  push:
    branches:
      - main
      - "release/*"
    branches-ignore:
      - "dependabot/**"
```

### 任务级条件

```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps: [...]

  notify:
    if: failure()  # 仅在失败时通知
    needs: [deploy]
    runs-on: ubuntu-latest
    steps:
      - uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "部署失败: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
```

## 产物与测试报告

### 上传构建产物

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7
```

### 测试报告

```yaml
- run: pnpm test -- --reporter=junit --outputFile=test-results.xml
- uses: dorny/test-reporter@v1
  if: always()
  with:
    name: 测试结果
    path: test-results.xml
    reporter: jest-junit
```

### 覆盖率报告

```yaml
- run: pnpm test -- --coverage
- uses: codecov/codecov-action@v4
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    files: coverage/lcov.info
    fail_ci_if_error: true
```

## Self-Hosted Runner 与 GitHub-Hosted Runner

| 特性 | GitHub-Hosted | Self-Hosted |
| --- | --- | --- |
| 维护成本 | 零 | 需要自行管理 |
| 费用 | 按分钟计费（公共仓库免费） | 仅基础设施费用 |
| 性能 | 标准配置 | 可定制高性能 |
| 安全 | 每次运行全新环境 | 需要自行加固 |
| 网络 | GitHub 公共 IP | 可访问内网资源 |
| 软件 | 预装常用工具 | 完全自定义 |

### 适用场景

**选择 GitHub-Hosted**：
- 标准 CI/CD 任务（lint、测试、构建）
- 公共开源项目
- 无特殊硬件或网络需求

**选择 Self-Hosted**：
- 需要访问内网资源（私有 npm registry、内部 API）
- 构建需要大量 CPU/内存（大型 monorepo）
- 需要 GPU（机器学习模型训练）
- 合规要求数据不离开私有网络

### Self-Hosted Runner 配置

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
      - run: pnpm build
```

## Monorepo CI

### Turborepo

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2  # turbo 需要对比差异
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile

      # Turborepo 远程缓存
      - run: pnpm turbo build lint test --filter="...[HEAD^1]"
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

### Nx

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # nx 需要完整 git 历史
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"
      - run: pnpm install --frozen-lockfile
      - uses: nrwl/nx-set-shas@v4

      # 仅对受影响的项目运行任务
      - run: pnpm nx affected -t lint test build
```

### Monorepo 变更检测

按包维度触发不同的工作流：

```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      api: ${{ steps.filter.outputs.api }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            web:
              - 'apps/web/**'
              - 'packages/shared/**'
            api:
              - 'apps/api/**'
              - 'packages/shared/**'
            shared:
              - 'packages/shared/**'

  test-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm --filter web test

  test-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm --filter api test
```
