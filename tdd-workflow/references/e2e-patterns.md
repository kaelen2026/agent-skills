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
