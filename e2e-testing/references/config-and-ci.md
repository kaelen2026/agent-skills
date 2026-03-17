# 配置与 CI/CD

## Playwright 配置

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,

  reporter: [
    ["html", { outputFolder: "playwright-report" }],
    ["junit", { outputFile: "playwright-results.xml" }],
    ["json", { outputFile: "playwright-results.json" }],
  ],

  use: {
    baseURL: process.env.BASE_URL || "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
    video: "retain-on-failure",
    actionTimeout: 10000,
    navigationTimeout: 30000,
  },

  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
    { name: "mobile-chrome", use: { ...devices["Pixel 5"] } },
  ],

  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

### 配置要点

| 配置项 | 本地 | CI | 说明 |
|--------|------|-----|------|
| `retries` | 0 | 2 | CI 中允许重试应对不稳定 |
| `workers` | auto | 1 | CI 中串行避免资源竞争 |
| `trace` | off | on-first-retry | 失败时保留 trace 用于调试 |
| `screenshot` | off | only-on-failure | 失败时自动截图 |
| `video` | off | retain-on-failure | 失败时保留视频 |
| `reuseExistingServer` | true | false | CI 中启动新服务 |

## 产物管理

### 截图

```typescript
// 手动截图
await page.screenshot({ path: "artifacts/after-login.png" });

// 全页截图
await page.screenshot({ path: "artifacts/full-page.png", fullPage: true });

// 元素截图
await page.locator('[data-testid="chart"]').screenshot({
  path: "artifacts/chart.png",
});
```

### Trace

```bash
# 运行时生成 trace
npx playwright test --trace on

# 查看 trace
npx playwright show-trace trace.zip
```

### 视频

```typescript
// playwright.config.ts
use: {
  video: "retain-on-failure",
}
```

### .gitignore

```
playwright-report/
playwright-results.*
artifacts/
test-results/
tests/.auth/
```

## GitHub Actions

```yaml
name: E2E Tests
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci

      - run: npx playwright install --with-deps

      - run: npx playwright test
        env:
          BASE_URL: ${{ vars.STAGING_URL }}

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: |
            playwright-report/
            artifacts/
          retention-days: 30
```

### 并行矩阵

```yaml
jobs:
  e2e:
    strategy:
      fail-fast: false
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      - run: npx playwright test --shard=${{ matrix.shard }}
```

## 测试报告模板

```markdown
# E2E Test Report

**Date:** YYYY-MM-DD HH:MM
**Duration:** Xm Ys
**Status:** PASSING / FAILING

## Summary

| Metric | Count |
|--------|-------|
| Total | X |
| Passed | Y (Z%) |
| Failed | A |
| Flaky | B |
| Skipped | C |

## Failed Tests

### test-name
- **File:** `tests/e2e/feature.spec.ts:45`
- **Error:** Expected element to be visible
- **Screenshot:** artifacts/failed.png
- **Recommended Fix:** [description]

## Artifacts
- HTML Report: playwright-report/index.html
- Screenshots: artifacts/*.png
- Videos: artifacts/videos/*.webm
- Traces: artifacts/*.zip
```

## 常用命令

```bash
# 运行全部测试
npx playwright test

# 运行指定文件
npx playwright test tests/e2e/auth/login.spec.ts

# 运行指定浏览器
npx playwright test --project=chromium

# UI 模式（交互式调试）
npx playwright test --ui

# 生成代码（录制操作）
npx playwright codegen http://localhost:3000

# 查看报告
npx playwright show-report
```
