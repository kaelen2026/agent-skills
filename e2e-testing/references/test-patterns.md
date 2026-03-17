# 测试模式

## 测试结构

```typescript
import { test, expect } from "@playwright/test";
import { ItemsPage } from "../../pages/ItemsPage";

test.describe("Item Search", () => {
  let itemsPage: ItemsPage;

  test.beforeEach(async ({ page }) => {
    itemsPage = new ItemsPage(page);
    await itemsPage.goto();
  });

  test("should search by keyword", async ({ page }) => {
    await itemsPage.search("test");

    const count = await itemsPage.getItemCount();
    expect(count).toBeGreaterThan(0);
    await expect(itemsPage.itemCards.first()).toContainText(/test/i);
  });

  test("should handle no results", async () => {
    await itemsPage.search("xyznonexistent123");

    await expect(itemsPage.noResults).toBeVisible();
    expect(await itemsPage.getItemCount()).toBe(0);
  });
});
```

## 等待策略

### 禁止硬等待

```typescript
// BAD: 硬等待
await page.waitForTimeout(5000);

// GOOD: 等待特定条件
await page.waitForResponse((resp) => resp.url().includes("/api/data"));
await page.locator('[data-testid="result"]').waitFor({ state: "visible" });
await page.waitForLoadState("networkidle");
await page.waitForURL("/dashboard");
```

### 等待方式优先级

| 优先级 | 方式 | 适用场景 |
|--------|------|----------|
| 1 | Locator 自动等待 | `locator.click()` 自动等待可点击 |
| 2 | `waitForResponse` | 等待 API 返回 |
| 3 | `waitForURL` | 页面跳转 |
| 4 | `waitFor({ state })` | 等待元素出现/消失 |
| 5 | `waitForLoadState` | 页面加载完成 |
| 6 | 自定义轮询 | 复杂条件 |

### Locator vs page 方法

```typescript
// BAD: 直接调用 page 方法（无自动等待）
await page.click('[data-testid="button"]');

// GOOD: 使用 locator（自动等待元素可交互）
await page.locator('[data-testid="button"]').click();
```

## 不稳定测试（Flaky Tests）

### 标记与隔离

```typescript
// 已知不稳定，暂时跳过
test("flaky: complex search", async ({ page }) => {
  test.fixme(true, "Flaky - Issue #123");
  // ...
});

// CI 环境下跳过
test("environment-sensitive test", async ({ page }) => {
  test.skip(!!process.env.CI, "Flaky in CI - Issue #456");
  // ...
});
```

### 诊断命令

```bash
# 重复运行检测不稳定性
npx playwright test tests/search.spec.ts --repeat-each=10

# 带 retry 运行
npx playwright test tests/search.spec.ts --retries=3

# 生成 trace 用于调试
npx playwright test tests/search.spec.ts --trace on
```

### 常见原因与修复

| 原因 | 症状 | 修复 |
|------|------|------|
| 竞态条件 | 有时点击无效 | 使用 locator 自动等待 |
| 网络时序 | API 未返回就断言 | `waitForResponse` |
| 动画 | 元素移动中被点击 | `waitFor({ state: 'visible' })` + `networkidle` |
| 测试顺序依赖 | 单独通过，一起失败 | 每个测试独立 setup |
| 共享状态 | 修改了全局数据 | 使用独立测试数据 |
| 时区/日期 | 跨日边界失败 | Mock 系统时间 |

### 动画等待

```typescript
// BAD: 动画过程中操作
await page.locator('[data-testid="menu-item"]').click();

// GOOD: 等待元素稳定后再操作
const menuItem = page.locator('[data-testid="menu-item"]');
await menuItem.waitFor({ state: "visible" });
await page.waitForLoadState("networkidle");
await menuItem.click();
```

## API 请求拦截

### Mock API 响应

```typescript
test("should display data from API", async ({ page }) => {
  await page.route("**/api/users", (route) =>
    route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify({
        success: true,
        data: [{ id: "1", name: "Alice" }],
        meta: { requestId: "req_test123" },
      }),
    })
  );

  await page.goto("/users");
  await expect(page.locator('[data-testid="user-card"]')).toHaveCount(1);
});
```

### 模拟网络错误

```typescript
test("should show error on API failure", async ({ page }) => {
  await page.route("**/api/users", (route) =>
    route.fulfill({
      status: 500,
      contentType: "application/json",
      body: JSON.stringify({
        success: false,
        error: { code: "INTERNAL_ERROR", message: "Server error" },
        meta: { requestId: "req_err001" },
      }),
    })
  );

  await page.goto("/users");
  await expect(page.locator('[data-testid="error-message"]')).toBeVisible();
});
```

## 关键流程测试

### 表单提交

```typescript
test("should submit form successfully", async ({ page }) => {
  await page.goto("/create");

  await page.locator('[data-testid="name-input"]').fill("Test Item");
  await page.locator('[data-testid="description-input"]').fill("Description");
  await page.locator('[data-testid="submit-btn"]').click();

  await page.waitForResponse(
    (resp) => resp.url().includes("/api/items") && resp.status() === 201
  );

  await expect(page.locator('[data-testid="success-toast"]')).toBeVisible();
  await page.waitForURL(/\/items\/.+/);
});
```

### 生产环境保护

```typescript
test("critical: payment flow", async ({ page }) => {
  test.skip(
    process.env.NODE_ENV === "production",
    "Skip on production - real money"
  );
  // ...
});
```
