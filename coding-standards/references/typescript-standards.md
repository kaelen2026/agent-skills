# TypeScript/JavaScript 编码规范

## 命名规范

### 变量命名

```typescript
// GOOD: 描述性名称
const marketSearchQuery = "election";
const isUserAuthenticated = true;
const totalRevenue = 1000;

// BAD: 含糊不清
const q = "election";
const flag = true;
const x = 1000;
```

### 函数命名

使用**动词-名词**模式：

```typescript
// GOOD
async function fetchMarketData(marketId: string) {}
function calculateSimilarity(a: number[], b: number[]) {}
function isValidEmail(email: string): boolean {}

// BAD: 不清晰或仅名词
async function market(id: string) {}
function similarity(a, b) {}
```

### 命名约定表

| 类型 | 约定 | 示例 |
|------|------|------|
| 变量/函数 | camelCase | `fetchUserData`, `isActive` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRIES`, `API_BASE_URL` |
| 类/接口/类型 | PascalCase | `UserProfile`, `ApiResponse` |
| 组件 | PascalCase | `Button`, `SearchInput` |
| Hook | camelCase + `use` 前缀 | `useAuth`, `useDebounce` |
| 文件（组件） | PascalCase | `Button.tsx`, `SearchInput.tsx` |
| 文件（工具） | camelCase | `formatDate.ts`, `parseQuery.ts` |
| 布尔变量 | `is`/`has`/`should`/`can` 前缀 | `isLoading`, `hasPermission` |

## 不可变性（关键）

```typescript
// GOOD: 始终使用 spread operator
const updatedUser = {
  ...user,
  name: "New Name",
};

const updatedArray = [...items, newItem];
const filteredArray = items.filter((item) => item.isActive);

// BAD: 直接修改
user.name = "New Name"; // mutation!
items.push(newItem); // mutation!
```

## 类型安全

### 禁止 `any`

```typescript
// GOOD: 明确类型
interface Market {
  id: string;
  name: string;
  status: "active" | "resolved" | "closed";
  createdAt: Date;
}

function getMarket(id: string): Promise<Market> {
  // ...
}

// BAD: 使用 any
function getMarket(id: any): Promise<any> {
  // ...
}
```

### 联合类型优于枚举

```typescript
// 推荐: 联合类型
type Status = "active" | "resolved" | "closed";

// 可选: const enum（如果需要反向映射）
const enum Status {
  Active = "active",
  Resolved = "resolved",
  Closed = "closed",
}
```

## 错误处理

```typescript
// GOOD: 全面的错误处理
async function fetchData(url: string) {
  try {
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json();
  } catch (error) {
    console.error("Fetch failed:", error);
    throw new Error("Failed to fetch data");
  }
}

// BAD: 无错误处理
async function fetchData(url: string) {
  const response = await fetch(url);
  return response.json();
}
```

## 异步最佳实践

### 并行执行独立操作

```typescript
// GOOD: Promise.all 并行执行
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats(),
]);

// BAD: 不必要的顺序执行
const users = await fetchUsers();
const markets = await fetchMarkets();
const stats = await fetchStats();
```

### 错误处理与 Promise.allSettled

```typescript
// 需要部分成功时使用 Promise.allSettled
const results = await Promise.allSettled([
  fetchUsers(),
  fetchMarkets(),
  fetchStats(),
]);

const successful = results
  .filter((r) => r.status === "fulfilled")
  .map((r) => r.value);
```

## 输入验证

在系统边界使用 zod 验证：

```typescript
import { z } from "zod";

const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1),
});

const validated = CreateMarketSchema.parse(input);
```

## 文件组织

### 项目结构

```
src/
├── app/                    # Next.js App Router
│   ├── api/               # API 路由
│   └── (auth)/            # 路由组
├── components/            # React 组件
│   ├── ui/               # 通用 UI 组件
│   ├── forms/            # 表单组件
│   └── layouts/          # 布局组件
├── hooks/                # 自定义 Hooks
├── lib/                  # 工具和配置
│   ├── api/             # API 客户端
│   ├── utils/           # 辅助函数
│   └── constants/       # 常量
├── types/                # TypeScript 类型
└── styles/              # 全局样式
```

### 文件大小限制

| 指标 | 上限 | 说明 |
|------|------|------|
| 文件行数 | 800 行 | 超过 400 行应考虑拆分 |
| 函数行数 | 50 行 | 超过就拆分为子函数 |
| 嵌套层级 | 4 层 | 使用 early return 减少嵌套 |
| 函数参数 | 3 个 | 超过就使用对象参数 |

## 注释规范

### 解释「为什么」而非「是什么」

```typescript
// GOOD: 解释原因
// 使用指数退避避免 API 服务中断时的过载
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000);

// 性能原因刻意使用 mutation（已通过 profiling 确认）
items.push(newItem);

// BAD: 陈述显而易见的事实
// 计数器加 1
count++;
```

### 公共 API 使用 JSDoc

```typescript
/**
 * 通过语义相似度搜索市场。
 *
 * @param query - 自然语言搜索查询
 * @param limit - 最大结果数（默认: 10）
 * @returns 按相似度排序的市场数组
 * @throws {Error} OpenAI API 失败或 Redis 不可用时抛出
 */
export async function searchMarkets(
  query: string,
  limit: number = 10
): Promise<Market[]> {
  // ...
}
```

## 魔法数字

```typescript
// BAD: 无解释的数字
if (retryCount > 3) {}
setTimeout(callback, 500);

// GOOD: 命名常量
const MAX_RETRIES = 3;
const DEBOUNCE_DELAY_MS = 500;

if (retryCount > MAX_RETRIES) {}
setTimeout(callback, DEBOUNCE_DELAY_MS);
```
