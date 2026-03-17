---
name: tdd-workflow
version: 1.0.0
description: "TDD 工作流技能。强制测试驱动开发，覆盖率 80%+ ，包含单元、集成、E2E 测试。"
last_updated: 2026-03-17
metadata:
  filePattern:
    - "**/*.test.ts"
    - "**/*.test.tsx"
    - "**/vitest.config.*"
    - "**/jest.config.*"
  bashPattern:
    - "vitest|jest|npm test|npm run test"
  priority: 6
---

# Test-Driven Development Workflow

像建筑师先画蓝图再动工一样，TDD 先用测试定义"正确行为"，再写代码实现。没有蓝图的房子迟早要返工。

## When to Activate

- 开发新功能或组件
- 修复 bug
- 重构现有代码
- 添加 API 端点
- 变更数据模型

## Core Principles

### 1. 测试先行（不可协商）

**先写测试，后写实现。** 实现后再补的测试只会写出"能通过的测试"，遗漏失败场景。

### 2. 覆盖率要求

- 最低 80% 覆盖率（unit + integration + E2E）
- 覆盖所有边界条件
- 覆盖错误场景
- 覆盖空值 / undefined / 极端输入

### 3. 测试类型

| 类型     | 测试对象                       | 工具              | 速度要求        |
| -------- | ------------------------------ | ----------------- | --------------- |
| Unit     | 函数、工具函数、组件逻辑       | Vitest / Jest     | 单个 < 50ms     |
| 集成     | API 端点、DB 操作、服务交互    | Vitest / Jest     | 单个 < 500ms    |
| E2E      | 关键用户流程、完整工作流       | Playwright        | 单个 < 30s      |

## Workflow

### 1. 编写用户故事

```
作为 [角色]，我希望 [操作]，以便 [收益]

示例:
作为用户，我希望按语义搜索市场，
以便即使不知道准确关键词也能找到相关结果。
```

### 2. 生成测试用例（RED）

为每个用户故事创建全面的测试用例：

```typescript
describe("Semantic Search", () => {
  it("returns relevant results for valid query", async () => {
    // Arrange → Act → Assert
  });

  it("returns empty array for no matches", async () => {
    // 边界: 无匹配
  });

  it("handles empty query gracefully", async () => {
    // 边界: 空输入
  });

  it("falls back to substring search when Redis unavailable", async () => {
    // 错误路径: 外部服务不可用
  });

  it("sorts results by similarity score descending", async () => {
    // 排序逻辑
  });
});
```

### 3. 运行测试（必须失败）

```bash
npm test
# 所有新测试应该失败 — 还没有实现
```

如果测试通过了，说明测试有问题（测试了已有行为或断言不正确）。

### 4. 最小实现（GREEN）

写刚好让测试通过的代码，不多不少：

```typescript
export async function searchMarkets(query: string): Promise<Market[]> {
  // 最小实现，只为通过测试
}
```

### 5. 再次运行测试

```bash
npm test
# 所有测试应该通过
```

### 6. 重构（IMPROVE）

保持测试绿色的前提下改善代码质量：

- 消除重复
- 改善命名
- 优化性能
- 提高可读性

### 7. 验证覆盖率

```bash
npm run test:coverage
# 确认 80%+ 覆盖率
```

## Testing Patterns

### Unit Test Pattern

```typescript
import { render, screen, fireEvent } from "@testing-library/react";
import { Button } from "./Button";

describe("Button Component", () => {
  it("renders with correct text", () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText("Click me")).toBeInTheDocument();
  });

  it("calls onClick when clicked", () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click</Button>);

    fireEvent.click(screen.getByRole("button"));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it("is disabled when disabled prop is true", () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByRole("button")).toBeDisabled();
  });
});
```

### API Integration Test Pattern

```typescript
import { NextRequest } from "next/server";
import { GET } from "./route";

describe("GET /api/markets", () => {
  it("returns markets successfully", async () => {
    const request = new NextRequest("http://localhost/api/markets");
    const response = await GET(request);
    const data = await response.json();

    expect(response.status).toBe(200);
    expect(data.success).toBe(true);
    expect(Array.isArray(data.data)).toBe(true);
  });

  it("rejects invalid query parameters", async () => {
    const request = new NextRequest(
      "http://localhost/api/markets?limit=invalid"
    );
    const response = await GET(request);

    expect(response.status).toBe(400);
  });

  it("handles database errors gracefully", async () => {
    // Mock database failure → 验证返回 500 + 友好错误信息
  });
});
```

### E2E Test Pattern (Playwright)

详见 [references/e2e-patterns.md](references/e2e-patterns.md)

## Test File Organization

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   └── Button.test.tsx          # Unit tests (colocated)
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts         # Integration tests (colocated)
└── e2e/
    ├── markets.spec.ts               # E2E tests (独立目录)
    ├── trading.spec.ts
    └── auth.spec.ts
```

**原则**: 单元测试和集成测试与源码同位放置，E2E 测试独立目录。

## Mocking 指南

详见 [references/mocking-patterns.md](references/mocking-patterns.md)

核心原则:

- **只 mock 外部依赖**（DB、Redis、第三方 API）
- **不 mock 被测代码的内部实现**
- 每个 mock 在 `afterEach` 中恢复

## Common Mistakes

### 测试实现细节（错误）

```typescript
// BAD: 测试内部状态
expect(component.state.count).toBe(5);

// GOOD: 测试用户可见行为
expect(screen.getByText("Count: 5")).toBeInTheDocument();
```

### 脆弱选择器（错误）

```typescript
// BAD: CSS class 会变
await page.click(".css-class-xyz");

// GOOD: 语义选择器
await page.click('button:has-text("Submit")');
await page.click('[data-testid="submit-button"]');
```

### 测试间耦合（错误）

```typescript
// BAD: 测试依赖执行顺序
test("creates user", () => {
  /* ... */
});
test("updates same user", () => {
  /* 依赖上个测试创建的用户 */
});

// GOOD: 每个测试独立
test("creates user", () => {
  const user = createTestUser();
  // ...
});
test("updates user", () => {
  const user = createTestUser();
  // ...
});
```

## Coverage Thresholds

```json
{
  "coverageThresholds": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

### 8. 设计审查

完成 TDD 循环后，自动检查以下规则：

- [ ] 所有测试通过（0 failures）
- [ ] 覆盖率 ≥ 80%
- [ ] 无 `skip` 或 `todo` 测试
- [ ] 单元测试总执行时间 < 30s
- [ ] E2E 测试覆盖关键用户流程
- [ ] Mock 只用于外部依赖
- [ ] 每个测试独立（无执行顺序依赖）
- [ ] 测试命名描述行为而非实现
- [ ] 错误路径和边界条件已覆盖
- [ ] 无 `console.log` 残留在测试中

## Output Format

```
# TDD Report: {功能名称}

## 用户故事
{用户故事列表}

## 测试用例
{测试文件路径 + 测试用例数量}

## RED → GREEN → IMPROVE
{每轮 TDD 循环的简要记录}

## 覆盖率报告
{覆盖率数据，标记是否达标}

## 设计审查
{审查结果清单，标记通过/未通过}
```
