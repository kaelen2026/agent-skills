# E2E Test Patterns (Playwright)

## 基本用户流程测试

```typescript
import { test, expect } from "@playwright/test";

test("user can search and filter markets", async ({ page }) => {
  // Navigate
  await page.goto("/");
  await page.click('a[href="/markets"]');

  // Verify page loaded
  await expect(page.locator("h1")).toContainText("Markets");

  // Search
  await page.fill('input[placeholder="Search markets"]', "election");
  await page.waitForTimeout(600); // debounce

  // Verify results
  const results = page.locator('[data-testid="market-card"]');
  await expect(results).toHaveCount(5, { timeout: 5000 });

  // Verify relevance
  const firstResult = results.first();
  await expect(firstResult).toContainText("election", { ignoreCase: true });

  // Filter
  await page.click('button:has-text("Active")');
  await expect(results).toHaveCount(3);
});
```

## 表单提交测试

```typescript
test("user can create a new market", async ({ page }) => {
  await page.goto("/creator-dashboard");

  // Fill form
  await page.fill('input[name="name"]', "Test Market");
  await page.fill('textarea[name="description"]', "Test description");
  await page.fill('input[name="endDate"]', "2025-12-31");

  // Submit
  await page.click('button[type="submit"]');

  // Verify success
  await expect(
    page.locator("text=Market created successfully")
  ).toBeVisible();
  await expect(page).toHaveURL(/\/markets\/test-market/);
});
```

## 选择器优先级

1. `getByRole` — 语义角色（最佳）
2. `getByText` — 可见文本
3. `data-testid` — 专用测试属性
4. CSS selector — 最后手段（易碎）

## 注意事项

- 每个测试独立，不依赖前一个测试的状态
- 使用 `beforeEach` 设置初始状态
- 避免 `page.waitForTimeout` 硬编码等待，优先使用 `waitForSelector` 或 `expect` 自动重试
- 截图用于调试失败: `await page.screenshot({ path: "debug.png" })`

## 认证流程测试

E2E 测试中认证是最常见的前置条件。Playwright 支持通过 `storageState` 保存和复用登录状态，避免每个测试都重新登录。

### 全局 Setup 登录

```typescript
// auth.setup.ts — 全局认证 setup
import { test as setup, expect } from "@playwright/test";

const AUTH_FILE = "playwright/.auth/user.json";

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("test@example.com");
  await page.getByLabel("Password").fill("test-password-123");
  await page.getByRole("button", { name: "Sign in" }).click();

  // 等待登录完成（重定向到 dashboard）
  await page.waitForURL("/dashboard");
  await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();

  // 保存认证状态（cookies + localStorage）
  await page.context().storageState({ path: AUTH_FILE });
});
```

### 在 playwright.config.ts 中配置

```typescript
// playwright.config.ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  projects: [
    // 认证 setup 项目（先执行）
    { name: "setup", testMatch: /.*\.setup\.ts/ },

    // 需要认证的测试
    {
      name: "authenticated",
      use: { storageState: "playwright/.auth/user.json" },
      dependencies: ["setup"],
    },

    // 不需要认证的测试（登录页、注册页等）
    {
      name: "unauthenticated",
      use: { storageState: { cookies: [], origins: [] } },
      testMatch: /.*\.unauth\.spec\.ts/,
    },
  ],
});
```

### 多角色认证

```typescript
// 管理员和普通用户使用不同的 storageState
const ADMIN_FILE = "playwright/.auth/admin.json";
const USER_FILE = "playwright/.auth/user.json";

setup("admin auth", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("admin@example.com");
  await page.getByLabel("Password").fill("admin-password");
  await page.getByRole("button", { name: "Sign in" }).click();
  await page.waitForURL("/admin");
  await page.context().storageState({ path: ADMIN_FILE });
});

// 在测试中使用
test.use({ storageState: ADMIN_FILE });

test("admin can delete user", async ({ page }) => {
  await page.goto("/admin/users");
  // ...
});
```

## API Mock (MSW)

使用 MSW (Mock Service Worker) 在 E2E 测试中拦截网络请求，适用于第三方 API 不可控或需要模拟特定响应的场景。

```typescript
// e2e/helpers/msw-server.ts
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

// 定义默认 handler
const handlers = [
  http.get("https://api.stripe.com/v1/prices", () => {
    return HttpResponse.json({
      data: [
        { id: "price_1", unit_amount: 2000, currency: "usd" },
        { id: "price_2", unit_amount: 5000, currency: "usd" },
      ],
    });
  }),

  http.post("https://api.stripe.com/v1/checkout/sessions", () => {
    return HttpResponse.json({
      id: "cs_test_123",
      url: "https://checkout.stripe.com/test",
    });
  }),
];

export const server = setupServer(...handlers);
```

### 在 Playwright 中使用路由拦截

Playwright 也提供原生的请求拦截，适用于浏览器端请求：

```typescript
test("应在支付 API 失败时显示错误", async ({ page }) => {
  // 拦截浏览器发出的请求
  await page.route("**/api/payment", (route) =>
    route.fulfill({
      status: 500,
      contentType: "application/json",
      body: JSON.stringify({ error: "Payment service unavailable" }),
    })
  );

  await page.goto("/checkout");
  await page.getByRole("button", { name: "Pay Now" }).click();

  await expect(page.getByText("Payment failed")).toBeVisible();
});

test("应正确显示商品价格", async ({ page }) => {
  await page.route("**/api/products", (route) =>
    route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify([
        { id: 1, name: "Pro Plan", price: 2000 },
        { id: 2, name: "Enterprise", price: 5000 },
      ]),
    })
  );

  await page.goto("/pricing");

  await expect(page.getByText("$20.00")).toBeVisible();
  await expect(page.getByText("$50.00")).toBeVisible();
});
```

## 数据库 Seed

E2E 测试需要可预测的初始数据。使用 seed 脚本确保每次测试运行前数据库状态一致。

```typescript
// e2e/helpers/seed.ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export async function seedTestData() {
  // 清除顺序：先删除有外键依赖的表
  await prisma.comment.deleteMany();
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();

  // 创建测试用户
  const testUser = await prisma.user.create({
    data: {
      email: "test@example.com",
      name: "Test User",
      role: "USER",
    },
  });

  // 创建测试文章
  await prisma.post.createMany({
    data: [
      { title: "First Post", content: "Hello", authorId: testUser.id, published: true },
      { title: "Draft Post", content: "WIP", authorId: testUser.id, published: false },
    ],
  });

  return { testUser };
}

export async function cleanupTestData() {
  await prisma.comment.deleteMany();
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
  await prisma.$disconnect();
}
```

### 在 global setup 中使用

```typescript
// playwright.config.ts
export default defineConfig({
  globalSetup: "./e2e/global-setup.ts",
  globalTeardown: "./e2e/global-teardown.ts",
});

// e2e/global-setup.ts
import { seedTestData } from "./helpers/seed";

export default async function globalSetup() {
  await seedTestData();
}

// e2e/global-teardown.ts
import { cleanupTestData } from "./helpers/seed";

export default async function globalTeardown() {
  await cleanupTestData();
}
```

## Page Object Model

将页面交互封装为可复用的类，避免在测试中重复编写选择器逻辑。测试代码只调用语义化方法。

```typescript
// e2e/pages/base-page.ts
import { Page, Locator, expect } from "@playwright/test";

export class BasePage {
  constructor(protected readonly page: Page) {}

  async navigateTo(path: string) {
    await this.page.goto(path);
  }

  async getToastMessage(): Promise<string> {
    const toast = this.page.locator('[role="alert"]');
    await expect(toast).toBeVisible();
    return toast.innerText();
  }

  async waitForLoadingComplete() {
    await this.page.locator('[data-testid="loading-spinner"]').waitFor({ state: "hidden" });
  }
}
```

```typescript
// e2e/pages/login-page.ts
import { Page, expect } from "@playwright/test";
import { BasePage } from "./base-page";

export class LoginPage extends BasePage {
  private readonly emailInput = this.page.getByLabel("Email");
  private readonly passwordInput = this.page.getByLabel("Password");
  private readonly submitButton = this.page.getByRole("button", { name: "Sign in" });
  private readonly errorMessage = this.page.getByTestId("login-error");

  async goto() {
    await this.navigateTo("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }
}
```

```typescript
// e2e/pages/dashboard-page.ts
import { Page, expect } from "@playwright/test";
import { BasePage } from "./base-page";

export class DashboardPage extends BasePage {
  private readonly heading = this.page.getByRole("heading", { level: 1 });
  private readonly createButton = this.page.getByRole("button", { name: "Create" });
  private readonly itemList = this.page.getByTestId("item-list");

  async goto() {
    await this.navigateTo("/dashboard");
  }

  async expectLoaded() {
    await expect(this.heading).toContainText("Dashboard");
  }

  async createItem(name: string) {
    await this.createButton.click();
    await this.page.getByLabel("Name").fill(name);
    await this.page.getByRole("button", { name: "Save" }).click();
  }

  async getItemCount(): Promise<number> {
    return this.itemList.locator("> *").count();
  }
}
```

```typescript
// e2e/tests/dashboard.spec.ts — Page Object 使用示例
import { test, expect } from "@playwright/test";
import { DashboardPage } from "../pages/dashboard-page";

test("用户可以创建新项目", async ({ page }) => {
  const dashboard = new DashboardPage(page);

  await dashboard.goto();
  await dashboard.expectLoaded();

  const countBefore = await dashboard.getItemCount();
  await dashboard.createItem("My New Project");

  const toast = await dashboard.getToastMessage();
  expect(toast).toContain("Created successfully");

  const countAfter = await dashboard.getItemCount();
  expect(countAfter).toBe(countBefore + 1);
});
```

## 视觉回归测试

Playwright 内置截图对比功能，可以检测 UI 变化。首次运行生成基线图片，后续运行自动对比。

```typescript
import { test, expect } from "@playwright/test";

test("首页视觉回归", async ({ page }) => {
  await page.goto("/");

  // 等待动态内容加载完成
  await page.locator('[data-testid="hero-section"]').waitFor();

  // 全页截图对比
  await expect(page).toHaveScreenshot("homepage.png", {
    fullPage: true,
    maxDiffPixelRatio: 0.01, // 允许 1% 像素差异
  });
});

test("组件视觉回归", async ({ page }) => {
  await page.goto("/components/button");

  // 单个元素截图
  const button = page.getByRole("button", { name: "Primary" });
  await expect(button).toHaveScreenshot("primary-button.png");
});

test("暗色模式视觉回归", async ({ page }) => {
  await page.emulateMedia({ colorScheme: "dark" });
  await page.goto("/");

  await expect(page).toHaveScreenshot("homepage-dark.png", {
    fullPage: true,
  });
});
```

### 更新基线

```bash
# 首次运行或 UI 变更后更新基线图片
npx playwright test --update-snapshots
```

## CI 集成

### playwright.config.ts（CI 优化配置）

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  timeout: 30_000,
  expect: { timeout: 5_000 },

  // CI 专用配置
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined, // CI 中单线程避免资源竞争
  reporter: process.env.CI
    ? [["html", { open: "never" }], ["github"]] // GitHub Actions 注解
    : [["html", { open: "on-failure" }]],

  // 全局设置
  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",
    trace: "on-first-retry",      // 失败重试时录制 trace
    screenshot: "only-on-failure", // 失败时自动截图
    video: "retain-on-failure",    // 失败时保留视频
  },

  // 启动开发服务器（CI 中使用）
  webServer: {
    command: "npm run start",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },

  projects: [
    { name: "setup", testMatch: /.*\.setup\.ts/ },
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"], storageState: "playwright/.auth/user.json" },
      dependencies: ["setup"],
    },
  ],
});
```

### GitHub Actions Workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        run: npx playwright test
        env:
          CI: true
          BASE_URL: http://localhost:3000
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14

      - name: Upload test traces
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces
          path: test-results/
          retention-days: 7
```

## 多浏览器测试

Playwright 原生支持 Chromium、Firefox、WebKit 三大引擎，以及移动端设备模拟。

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  projects: [
    { name: "setup", testMatch: /.*\.setup\.ts/ },

    // 桌面浏览器
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
      dependencies: ["setup"],
    },
    {
      name: "firefox",
      use: { ...devices["Desktop Firefox"] },
      dependencies: ["setup"],
    },
    {
      name: "webkit",
      use: { ...devices["Desktop Safari"] },
      dependencies: ["setup"],
    },

    // 移动端模拟
    {
      name: "mobile-chrome",
      use: { ...devices["Pixel 7"] },
      dependencies: ["setup"],
    },
    {
      name: "mobile-safari",
      use: { ...devices["iPhone 14"] },
      dependencies: ["setup"],
    },

    // 自定义视口
    {
      name: "tablet",
      use: {
        viewport: { width: 1024, height: 768 },
        isMobile: false,
        hasTouch: true,
      },
      dependencies: ["setup"],
    },
  ],
});
```

### 按浏览器运行

```bash
# 只运行 chromium 测试
npx playwright test --project=chromium

# 运行所有移动端测试
npx playwright test --project=mobile-chrome --project=mobile-safari

# 运行全部浏览器
npx playwright test
```

## 不稳定测试修复

不稳定测试 (flaky tests) 是 E2E 测试最大的敌人。以下是常见原因和解决方案。

### 1. 竞态条件（最常见）

```typescript
// 反模式：硬编码等待
await page.waitForTimeout(2000); // 不可靠！

// 正确做法：等待具体条件
await expect(page.getByText("Data loaded")).toBeVisible();
await page.waitForResponse("**/api/data");
await page.locator('[data-testid="spinner"]').waitFor({ state: "hidden" });
```

### 2. 动画干扰点击

```typescript
// 反模式：动画进行中就点击
await page.getByRole("button", { name: "Submit" }).click(); // 可能点不到

// 正确做法：等待动画完成
await page.getByRole("button", { name: "Submit" }).click({ force: false }); // 默认等待 actionable
// 或者在 CSS 中禁用测试环境的动画
// playwright.config.ts
use: {
  // 注入 CSS 禁用动画
  launchOptions: {
    args: ["--force-prefers-reduced-motion"],
  },
}
```

### 3. 网络不确定性

```typescript
// 反模式：假设 API 总是快速响应
await page.goto("/dashboard");
await page.getByText("Welcome").click(); // 数据可能还没加载

// 正确做法：等待网络请求完成
await page.goto("/dashboard");
await page.waitForResponse((resp) =>
  resp.url().includes("/api/user") && resp.status() === 200
);
await page.getByText("Welcome").click();
```

### 4. 测试数据冲突

```typescript
// 反模式：使用固定数据，并行测试时冲突
test("create user", async ({ page }) => {
  await page.getByLabel("Email").fill("test@example.com"); // 另一个测试也用这个邮箱
});

// 正确做法：每个测试使用唯一数据
test("create user", async ({ page }) => {
  const uniqueEmail = `test-${Date.now()}-${Math.random().toString(36).slice(2)}@example.com`;
  await page.getByLabel("Email").fill(uniqueEmail);
});
```

### 5. Actionability 检查清单

Playwright 在操作元素前会自动检查以下条件（称为 actionability checks）：

| 操作 | 可见 | 稳定 | 可接收事件 | 已启用 | 可编辑 |
|------|:---:|:---:|:---:|:---:|:---:|
| `click` | Yes | Yes | Yes | Yes | - |
| `fill` | Yes | Yes | - | Yes | Yes |
| `check` | Yes | Yes | Yes | Yes | - |
| `hover` | Yes | Yes | Yes | - | - |
| `screenshot` | Yes | Yes | - | - | - |

利用这些自动检查，而不是手动添加等待逻辑。

### 调试不稳定测试的工具

```bash
# 使用 trace viewer 分析失败
npx playwright test --trace on
npx playwright show-trace test-results/trace.zip

# 重复运行检测不稳定性
npx playwright test --repeat-each 5

# 使用 UI 模式调试
npx playwright test --ui
```
