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
