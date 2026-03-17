# Page Object Model

## 目录结构

```
tests/
├── e2e/
│   ├── auth/
│   │   ├── login.spec.ts
│   │   ├── logout.spec.ts
│   │   └── register.spec.ts
│   ├── features/
│   │   ├── browse.spec.ts
│   │   ├── search.spec.ts
│   │   └── create.spec.ts
│   └── api/
│       └── endpoints.spec.ts
├── pages/                  # Page Object Model
│   ├── LoginPage.ts
│   ├── SearchPage.ts
│   └── BasePage.ts
├── fixtures/
│   ├── auth.ts
│   └── data.ts
└── playwright.config.ts
```

## 基类

```typescript
import { Page } from "@playwright/test";

export class BasePage {
  constructor(protected readonly page: Page) {}

  async goto(path: string) {
    await this.page.goto(path);
    await this.page.waitForLoadState("networkidle");
  }

  async waitForApi(urlPattern: string) {
    await this.page.waitForResponse((resp) =>
      resp.url().includes(urlPattern)
    );
  }

  async screenshot(name: string) {
    await this.page.screenshot({ path: `artifacts/${name}.png` });
  }
}
```

## 页面类

```typescript
import { Page, Locator } from "@playwright/test";
import { BasePage } from "./BasePage";

export class ItemsPage extends BasePage {
  readonly searchInput: Locator;
  readonly itemCards: Locator;
  readonly createButton: Locator;
  readonly noResults: Locator;

  constructor(page: Page) {
    super(page);
    this.searchInput = page.locator('[data-testid="search-input"]');
    this.itemCards = page.locator('[data-testid="item-card"]');
    this.createButton = page.locator('[data-testid="create-btn"]');
    this.noResults = page.locator('[data-testid="no-results"]');
  }

  async goto() {
    await super.goto("/items");
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.waitForApi("/api/search");
    await this.page.waitForLoadState("networkidle");
  }

  async getItemCount() {
    return await this.itemCards.count();
  }
}
```

## POM 规则

### 元素定位优先级

| 优先级 | 方式 | 示例 |
|--------|------|------|
| 1 | `data-testid` | `[data-testid="submit-btn"]` |
| 2 | ARIA role + name | `getByRole('button', { name: 'Submit' })` |
| 3 | Text content | `getByText('Submit')` |
| 4 | CSS selector | `.submit-button`（避免使用） |
| 5 | XPath | 不使用 |

### 设计原则

- **一个页面/组件 = 一个 POM 类**
- 方法代表用户操作（`login()`、`search()`），不是 DOM 操作
- Locator 定义在构造函数中，作为 `readonly` 属性
- 复杂流程拆分为多个方法
- POM 不包含断言（断言在测试文件中）

### data-testid 命名

```
[data-testid="search-input"]          # 功能-元素类型
[data-testid="user-card"]             # 资源-元素类型
[data-testid="login-submit-btn"]      # 页面-操作-元素类型
[data-testid="error-message"]         # 状态-元素类型
```

## 认证 Fixture

```typescript
import { test as base, Page } from "@playwright/test";

type AuthFixtures = {
  authenticatedPage: Page;
};

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    await page.goto("/login");
    await page.locator('[data-testid="email-input"]').fill("test@example.com");
    await page.locator('[data-testid="password-input"]').fill("password123");
    await page.locator('[data-testid="login-submit-btn"]').click();
    await page.waitForURL("/dashboard");
    await use(page);
  },
});

export { expect } from "@playwright/test";
```

### 使用 storageState 复用认证

```typescript
// 全局 setup 保存认证状态
import { chromium } from "@playwright/test";

async function globalSetup() {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto("/login");
  await page.locator('[data-testid="email-input"]').fill("test@example.com");
  await page.locator('[data-testid="password-input"]').fill("password123");
  await page.locator('[data-testid="login-submit-btn"]').click();
  await page.waitForURL("/dashboard");

  await page.context().storageState({ path: "tests/.auth/user.json" });
  await browser.close();
}

export default globalSetup;
```

```typescript
// playwright.config.ts 中引用
export default defineConfig({
  projects: [
    { name: "setup", testMatch: /global-setup\.ts/ },
    {
      name: "chromium",
      dependencies: ["setup"],
      use: { storageState: "tests/.auth/user.json" },
    },
  ],
});
```
