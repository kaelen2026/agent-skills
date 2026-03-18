# Mocking Patterns

## 核心原则

- **只 mock 外部依赖**（数据库、缓存、第三方 API）
- **不 mock 被测模块的内部实现**
- 每个 mock 在 `afterEach` 中恢复

## Database Mock (Supabase)

```typescript
vi.mock("@/lib/supabase", () => ({
  supabase: {
    from: vi.fn(() => ({
      select: vi.fn(() => ({
        eq: vi.fn(() =>
          Promise.resolve({
            data: [{ id: 1, name: "Test Market" }],
            error: null,
          })
        ),
      })),
    })),
  },
}));
```

## Cache Mock (Redis)

```typescript
vi.mock("@/lib/redis", () => ({
  searchMarketsByVector: vi.fn(() =>
    Promise.resolve([{ slug: "test-market", similarity_score: 0.95 }])
  ),
  checkRedisHealth: vi.fn(() => Promise.resolve({ connected: true })),
}));
```

## AI SDK Mock

```typescript
vi.mock("ai", () => ({
  generateText: vi.fn(() =>
    Promise.resolve({
      text: "mocked response",
      usage: { promptTokens: 10, completionTokens: 20 },
    })
  ),
}));
```

## Embedding Mock

```typescript
vi.mock("@/lib/openai", () => ({
  generateEmbedding: vi.fn(() =>
    Promise.resolve(new Array(1536).fill(0.1)) // Mock 1536-dim embedding
  ),
}));
```

## HTTP Request Mock (msw)

```typescript
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  http.get("https://api.example.com/data", () => {
    return HttpResponse.json({ items: [{ id: 1 }] });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## 失败场景 Mock

```typescript
// 模拟网络错误
vi.mocked(fetchData).mockRejectedValueOnce(new Error("Network error"));

// 模拟超时
vi.mocked(fetchData).mockImplementationOnce(
  () => new Promise((_, reject) => setTimeout(() => reject(new Error("Timeout")), 5000))
);

// 模拟部分失败
vi.mocked(batchProcess).mockResolvedValueOnce({
  succeeded: [{ id: 1 }],
  failed: [{ id: 2, error: "Not found" }],
});
```

## 注意事项

- 使用 `vi.fn()` (Vitest) 而非 `jest.fn()` (Jest) — 根据项目配置选择
- Mock 返回值应符合实际数据结构
- 测试错误路径时使用 `mockRejectedValueOnce`
- 避免全局 mock 污染：使用 `Once` 后缀的方法

## Prisma Mock

Prisma 是最常见的 ORM，mock 时需要模拟整个 PrismaClient 实例。推荐创建一个共享的 mock 文件，所有测试复用。

```typescript
// __mocks__/prisma.ts
import { PrismaClient } from "@prisma/client";
import { vi } from "vitest";

// 深度 mock PrismaClient 的所有模型方法
const prisma = {
  user: {
    findMany: vi.fn(),
    findUnique: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
    count: vi.fn(),
  },
  post: {
    findMany: vi.fn(),
    findUnique: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
  },
  $transaction: vi.fn(),
  $disconnect: vi.fn(),
} as unknown as PrismaClient;

export { prisma };
```

```typescript
// user-service.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("@/lib/prisma", () => ({
  prisma: {
    user: {
      findMany: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    },
    $transaction: vi.fn(),
  },
}));

import { prisma } from "@/lib/prisma";
import { listUsers, createUser } from "@/services/user-service";

beforeEach(() => {
  vi.clearAllMocks();
});

describe("listUsers", () => {
  it("应返回分页用户列表", async () => {
    vi.mocked(prisma.user.findMany).mockResolvedValue([
      { id: "1", name: "Alice", email: "alice@test.com" },
      { id: "2", name: "Bob", email: "bob@test.com" },
    ]);

    const result = await listUsers({ page: 1, limit: 10 });

    expect(prisma.user.findMany).toHaveBeenCalledWith({
      skip: 0,
      take: 10,
      orderBy: { createdAt: "desc" },
    });
    expect(result).toHaveLength(2);
  });
});

describe("createUser", () => {
  it("应创建用户并返回新记录", async () => {
    const newUser = { id: "3", name: "Charlie", email: "charlie@test.com" };
    vi.mocked(prisma.user.create).mockResolvedValue(newUser);

    const result = await createUser({ name: "Charlie", email: "charlie@test.com" });

    expect(prisma.user.create).toHaveBeenCalledWith({
      data: { name: "Charlie", email: "charlie@test.com" },
    });
    expect(result).toEqual(newUser);
  });
});
```

## 环境变量 Mock

测试中经常需要模拟不同的环境变量配置。Vitest 提供了 `vi.stubEnv` 方法，比手动操作 `process.env` 更安全。

```typescript
import { describe, it, expect, vi, afterEach } from "vitest";
import { getConfig } from "@/lib/config";

afterEach(() => {
  vi.unstubAllEnvs(); // 恢复所有环境变量
});

describe("getConfig", () => {
  it("应从环境变量读取数据库 URL", () => {
    vi.stubEnv("DATABASE_URL", "postgresql://localhost:5432/test");

    const config = getConfig();

    expect(config.databaseUrl).toBe("postgresql://localhost:5432/test");
  });

  it("环境变量缺失时应抛出错误", () => {
    vi.stubEnv("DATABASE_URL", ""); // 设为空字符串

    expect(() => getConfig()).toThrow("DATABASE_URL not configured");
  });

  it("应根据 NODE_ENV 切换配置", () => {
    vi.stubEnv("NODE_ENV", "production");
    vi.stubEnv("API_BASE_URL", "https://api.prod.example.com");

    const config = getConfig();

    expect(config.apiBaseUrl).toBe("https://api.prod.example.com");
  });
});
```

## Timer Mock

异步逻辑中经常使用 `setTimeout`/`setInterval`。使用 `vi.useFakeTimers` 可以精确控制时间流逝，避免测试中的真实等待。

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { debounce } from "@/utils/debounce";
import { scheduleRetry } from "@/lib/retry";

beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers(); // 必须恢复，否则影响后续测试
});

describe("debounce", () => {
  it("应在指定延迟后执行", () => {
    const callback = vi.fn();
    const debounced = debounce(callback, 300);

    debounced();
    expect(callback).not.toHaveBeenCalled(); // 还没到时间

    vi.advanceTimersByTime(299);
    expect(callback).not.toHaveBeenCalled(); // 差 1ms

    vi.advanceTimersByTime(1);
    expect(callback).toHaveBeenCalledTimes(1); // 刚好 300ms
  });

  it("连续调用应重置计时器", () => {
    const callback = vi.fn();
    const debounced = debounce(callback, 300);

    debounced();
    vi.advanceTimersByTime(200);
    debounced(); // 重置
    vi.advanceTimersByTime(200);
    expect(callback).not.toHaveBeenCalled(); // 从第二次调用起还没到 300ms

    vi.advanceTimersByTime(100);
    expect(callback).toHaveBeenCalledTimes(1);
  });
});

describe("scheduleRetry", () => {
  it("应按指数退避重试", async () => {
    const task = vi.fn()
      .mockRejectedValueOnce(new Error("fail"))
      .mockRejectedValueOnce(new Error("fail"))
      .mockResolvedValueOnce("success");

    const promise = scheduleRetry(task, { maxRetries: 3, baseDelay: 1000 });

    // 第一次重试: 1000ms
    await vi.advanceTimersByTimeAsync(1000);
    // 第二次重试: 2000ms（指数退避）
    await vi.advanceTimersByTimeAsync(2000);

    const result = await promise;
    expect(result).toBe("success");
    expect(task).toHaveBeenCalledTimes(3);
  });
});
```

## Spy 模式

当你只想**观察**某个方法的调用，而不想完全替换其实现时，使用 `vi.spyOn`。Spy 保留原始行为，同时记录调用信息。

```typescript
import { describe, it, expect, vi, afterEach } from "vitest";
import * as mathUtils from "@/utils/math";
import { calculateReport } from "@/services/report-service";

afterEach(() => {
  vi.restoreAllMocks(); // spy 必须 restore
});

describe("calculateReport", () => {
  it("应调用 sum 方法并传入正确参数", () => {
    // spy 保留原始实现
    const sumSpy = vi.spyOn(mathUtils, "sum");

    const report = calculateReport([10, 20, 30]);

    expect(sumSpy).toHaveBeenCalledWith([10, 20, 30]);
    expect(report.total).toBe(60); // 原始逻辑正常执行
  });

  it("可以临时覆盖 spy 的返回值", () => {
    vi.spyOn(mathUtils, "sum").mockReturnValue(999);

    const report = calculateReport([1, 2, 3]);

    expect(report.total).toBe(999); // 使用 mock 返回值
  });
});

// spy 也适用于对象方法
describe("console spy", () => {
  it("应在错误时输出警告", () => {
    const warnSpy = vi.spyOn(console, "warn").mockImplementation(() => {});

    processWithWarning(null);

    expect(warnSpy).toHaveBeenCalledWith("Invalid input: null");
  });
});
```

## Mock 验证最佳实践

### 调用验证

```typescript
// 验证调用次数
expect(mockFn).toHaveBeenCalledTimes(2);

// 验证调用参数
expect(mockFn).toHaveBeenCalledWith("arg1", { key: "value" });

// 验证最后一次调用的参数
expect(mockFn).toHaveBeenLastCalledWith("final-arg");

// 验证第 N 次调用（0-indexed）
expect(mockFn.mock.calls[0]).toEqual(["first-call-arg"]);
expect(mockFn.mock.calls[1]).toEqual(["second-call-arg"]);

// 验证从未被调用
expect(mockFn).not.toHaveBeenCalled();

// 验证调用顺序（多个 mock 之间）
expect(mockA).toHaveBeenCalledBefore(mockB);
```

### mockClear vs mockReset vs mockRestore

三者区别非常重要，混用会导致难以排查的测试 bug：

```typescript
const mockFn = vi.fn().mockReturnValue("hello");
mockFn("arg1");

// mockClear: 清除调用记录，保留实现
mockFn.mockClear();
expect(mockFn).not.toHaveBeenCalled(); // 调用记录清除
expect(mockFn()).toBe("hello");        // 实现保留

// mockReset: 清除调用记录 + 移除实现
mockFn.mockReset();
expect(mockFn()).toBeUndefined();      // 实现也被移除

// mockRestore: 恢复到原始实现（只对 vi.spyOn 有意义）
const spy = vi.spyOn(obj, "method");
spy.mockRestore();                      // 恢复到原始方法
```

| 方法 | 清除调用记录 | 移除实现 | 恢复原始 |
|------|:---:|:---:|:---:|
| `mockClear` | Yes | No | No |
| `mockReset` | Yes | Yes | No |
| `mockRestore` | Yes | Yes | Yes |

**建议**: `beforeEach` 中用 `vi.clearAllMocks()`，`afterEach` 中用 `vi.restoreAllMocks()`。

## Module Factory 模式

有时不同测试需要同一个 mock 返回不同值。使用 `vi.mocked()` 配合 `mockReturnValue` / `mockResolvedValue` 可以按测试动态控制。

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("@/lib/feature-flags", () => ({
  getFeatureFlag: vi.fn(),
}));

import { getFeatureFlag } from "@/lib/feature-flags";
import { renderDashboard } from "@/components/dashboard";

beforeEach(() => {
  vi.clearAllMocks();
});

describe("renderDashboard", () => {
  it("新功能开启时应显示 beta 面板", () => {
    // 本测试中返回 true
    vi.mocked(getFeatureFlag).mockReturnValue(true);

    const result = renderDashboard();

    expect(result.showBetaPanel).toBe(true);
  });

  it("新功能关闭时应隐藏 beta 面板", () => {
    // 本测试中返回 false
    vi.mocked(getFeatureFlag).mockReturnValue(false);

    const result = renderDashboard();

    expect(result.showBetaPanel).toBe(false);
  });

  it("可以根据参数返回不同值", () => {
    vi.mocked(getFeatureFlag).mockImplementation((flag: string) => {
      const flags: Record<string, boolean> = {
        "beta-panel": true,
        "dark-mode": false,
        "new-nav": true,
      };
      return flags[flag] ?? false;
    });

    const result = renderDashboard();

    expect(result.showBetaPanel).toBe(true);
    expect(result.useDarkMode).toBe(false);
  });
});
```

## Mock 反模式

### 1. 过度 Mock（Over-mocking）

```typescript
// 反模式：mock 了所有东西，测试什么都没验证
vi.mock("@/utils/validate");
vi.mock("@/utils/format");
vi.mock("@/utils/transform");
vi.mock("@/lib/db");

it("should process data", async () => {
  vi.mocked(validate).mockReturnValue(true);
  vi.mocked(format).mockReturnValue("formatted");
  vi.mocked(transform).mockReturnValue("transformed");
  vi.mocked(db.save).mockResolvedValue({ id: 1 });

  // 这个测试只是验证 mock 的调用顺序，没有测试真正的逻辑
  await processData(input);
  expect(db.save).toHaveBeenCalled(); // 几乎没有价值
});

// 正确做法：只 mock 外部边界（数据库），保留内部逻辑
vi.mock("@/lib/db");

it("should validate, format, transform, and save", async () => {
  vi.mocked(db.save).mockResolvedValue({ id: 1 });

  const result = await processData({ name: "test", value: 42 });

  // 验证最终保存的数据（内部逻辑真正执行了）
  expect(db.save).toHaveBeenCalledWith({
    name: "TEST",           // validate + format 真正执行
    value: "42.00",         // transform 真正执行
    processedAt: expect.any(Date),
  });
});
```

### 2. Mock 内部实现细节

```typescript
// 反模式：测试与实现细节耦合
it("should call internal helper", () => {
  const spy = vi.spyOn(service, "_privateHelper"); // 私有方法
  service.publicMethod();
  expect(spy).toHaveBeenCalled(); // 重构后立即失败
});

// 正确做法：测试公开行为
it("should return processed result", () => {
  const result = service.publicMethod();
  expect(result).toEqual(expectedOutput); // 不关心内部如何实现
});
```

### 3. 忘记恢复 Mock

```typescript
// 反模式：mock 泄漏到其他测试
describe("suite A", () => {
  it("test 1", () => {
    vi.spyOn(Date, "now").mockReturnValue(1000);
    // 忘记 restore！Date.now 在后续所有测试中都返回 1000
  });
});

// 正确做法：始终在 afterEach 中恢复
afterEach(() => {
  vi.restoreAllMocks();
});
```

### 4. Mock 返回值与真实数据结构不匹配

```typescript
// 反模式：简化了返回结构，漏掉重要字段
vi.mocked(getUser).mockResolvedValue({ id: 1 }); // 缺少 name, email, role...

// 正确做法：使用工厂函数生成完整的测试数据
function createMockUser(overrides?: Partial<User>): User {
  return {
    id: "user-1",
    name: "Test User",
    email: "test@example.com",
    role: "member",
    createdAt: new Date("2025-01-01"),
    ...overrides,
  };
}

vi.mocked(getUser).mockResolvedValue(createMockUser({ role: "admin" }));
```
