# React 模式与最佳实践

## 组件结构

### 函数组件 + 类型定义

```typescript
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  disabled?: boolean;
  variant?: "primary" | "secondary";
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = "primary",
}: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {children}
    </button>
  );
}
```

### 组件文件结构

一个组件文件的推荐顺序：

1. 导入
2. 类型/接口定义
3. 常量
4. 组件函数
5. 辅助函数（仅此组件使用）

## 自定义 Hooks

### 提取可复用逻辑

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// 使用
const debouncedQuery = useDebounce(searchQuery, 500);
```

### Hook 规则

- 命名以 `use` 开头
- 仅在组件或其他 Hook 顶层调用
- 不在条件语句/循环中调用
- 提取到 `hooks/` 目录便于复用

## 状态管理

### 函数式更新

```typescript
const [count, setCount] = useState(0);

// GOOD: 基于前值的函数式更新
setCount((prev) => prev + 1);

// BAD: 直接引用（异步场景下可能过期）
setCount(count + 1);
```

### 状态设计原则

| 原则 | 说明 |
|------|------|
| 最小化状态 | 能从其他状态派生的值不要存为状态 |
| 就近放置 | 状态放在最靠近使用它的组件 |
| 单一数据源 | 同一数据不要在多个状态中重复 |
| 不可变更新 | 始终创建新对象/数组 |

### 复杂状态使用 useReducer

```typescript
type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset"; payload: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "increment":
      return state + 1;
    case "decrement":
      return state - 1;
    case "reset":
      return action.payload;
  }
}
```

## 条件渲染

```typescript
// GOOD: 清晰的条件渲染
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// BAD: 三元嵌套地狱
{isLoading ? <Spinner /> : error ? <ErrorMessage /> : data ? <DataDisplay /> : null}
```

### 复杂条件提取为组件

```typescript
// GOOD: 状态映射
function StatusView({ status, data, error }: Props) {
  if (status === "loading") return <Spinner />;
  if (status === "error") return <ErrorMessage error={error} />;
  if (status === "empty") return <EmptyState />;
  return <DataDisplay data={data} />;
}
```

## 性能优化

### Memoization

```typescript
import { useMemo, useCallback } from "react";

// 缓存昂贵计算
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume);
}, [markets]);

// 缓存回调
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query);
}, []);
```

### 何时使用 useMemo/useCallback

| 场景 | 使用 | 原因 |
|------|------|------|
| 传递给 `React.memo` 子组件的 props | `useCallback` / `useMemo` | 防止不必要的重渲染 |
| 昂贵计算（排序、过滤大数组） | `useMemo` | 避免每次渲染重新计算 |
| 简单值/小数组 | 不使用 | 缓存本身有开销 |

### 懒加载

```typescript
import { lazy, Suspense } from "react";

const HeavyChart = lazy(() => import("./HeavyChart"));

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  );
}
```

## 组件模式

### 组合优于继承

```typescript
// GOOD: 通过 children 组合
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>;
}

// 使用
<Card>
  <CardHeader>Title</CardHeader>
  <p>Content</p>
</Card>
```

### Render Props（需要共享逻辑时）

```typescript
interface MouseTrackerProps {
  render: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);

  return <>{render(position)}</>;
}
```
