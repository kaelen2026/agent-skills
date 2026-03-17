# Performance Patterns

## Memoization

### 何时使用

Memoization 就像缓存菜单上的推荐菜——只有食材变了才重新推荐，否则直接用上次的结果。

| Hook          | 用途                   | 使用场景                             |
| ------------- | ---------------------- | ------------------------------------ |
| `useMemo`     | 缓存计算结果           | 排序、过滤、复杂计算                 |
| `useCallback` | 缓存函数引用           | 传递给子组件的回调                   |
| `React.memo`  | 缓存组件渲染           | props 不变时跳过重渲染的纯展示组件   |

```typescript
// useMemo: 昂贵计算只在依赖变化时重新执行
const sortedItems = useMemo(() => {
  return [...items].sort((a, b) => b.score - a.score);
}, [items]);

// useCallback: 函数引用稳定，避免子组件不必要的重渲染
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query);
}, []);

// React.memo: props 不变时跳过整个组件的重渲染
export const ItemCard = React.memo<ItemCardProps>(({ item, onClick }) => {
  return (
    <div className="item-card" onClick={() => onClick(item.id)}>
      <h3>{item.name}</h3>
      <p>{item.description}</p>
    </div>
  );
});
```

### 避免过度优化

- 不要给每个组件都加 `React.memo` — 只用在确认有性能问题的地方
- 简单计算不需要 `useMemo` — 基本数组方法在小数据集上足够快
- 先用 React DevTools Profiler 定位问题，再优化

## Code Splitting & Lazy Loading

按需加载就像自助餐厅——不一次性把所有菜都摆出来，客人走到哪个区域才上那个区域的菜。

```typescript
import { lazy, Suspense } from "react";

// 重型组件按需加载
const HeavyChart = lazy(() => import("./HeavyChart"));
const AdminPanel = lazy(() => import("./AdminPanel"));

export function Dashboard() {
  return (
    <div>
      {/* 关键内容直接渲染 */}
      <DashboardHeader />

      {/* 图表区域按需加载，显示骨架屏等待 */}
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>

      {/* 管理面板按需加载 */}
      {isAdmin && (
        <Suspense fallback={<Spinner />}>
          <AdminPanel />
        </Suspense>
      )}
    </div>
  );
}
```

### 适用场景

- 大型第三方库组件（图表、编辑器、3D 渲染）
- 仅特定用户可见的功能（Admin, Pro features）
- 不在首屏的内容（below the fold）
- 路由级别的页面组件

## 虚拟化长列表

渲染 10,000 个 DOM 节点会卡死浏览器。虚拟化就像在窗户后面移动一幅长画卷——只渲染窗户里看得见的部分。

```typescript
import { useVirtualizer } from "@tanstack/react-virtual";

export function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80, // 每行预估高度
    overscan: 5, // 额外渲染行数（滚动平滑度）
  });

  return (
    <div
      ref={parentRef}
      style={{ height: "600px", overflow: "auto" }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: "relative",
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: "absolute",
              top: 0,
              left: 0,
              width: "100%",
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <ItemCard item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 适用场景

| 列表大小      | 推荐方案                     |
| ------------- | ---------------------------- |
| < 100 项      | 直接渲染                     |
| 100 - 500 项  | 分页或 "Load more"           |
| 500+ 项       | 虚拟化                       |

## 表单性能

### 受控组件 + Validation

```typescript
import { z } from "zod";

const formSchema = z.object({
  name: z.string().min(1, "Name is required").max(100),
  email: z.string().email("Invalid email format"),
  age: z.number().int().min(0).max(150),
});

type FormData = z.infer<typeof formSchema>;

export function CreateForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const [formData, setFormData] = useState<FormData>({
    name: "",
    email: "",
    age: 0,
  });
  const [errors, setErrors] = useState<Partial<Record<keyof FormData, string>>>({});

  const handleChange = useCallback(
    <K extends keyof FormData>(field: K, value: FormData[K]) => {
      setFormData((prev) => ({ ...prev, [field]: value }));
      // 清除该字段的错误
      setErrors((prev) => {
        const { [field]: _, ...rest } = prev;
        return rest;
      });
    },
    []
  );

  const handleSubmit = useCallback(
    (e: React.FormEvent) => {
      e.preventDefault();

      const result = formSchema.safeParse(formData);
      if (!result.success) {
        const fieldErrors: Partial<Record<keyof FormData, string>> = {};
        for (const issue of result.error.issues) {
          const field = issue.path[0] as keyof FormData;
          fieldErrors[field] = issue.message;
        }
        setErrors(fieldErrors);
        return;
      }

      onSubmit(result.data);
    },
    [formData, onSubmit]
  );

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          value={formData.name}
          onChange={(e) => handleChange("name", e.target.value)}
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? "name-error" : undefined}
        />
        {errors.name && <span id="name-error" role="alert">{errors.name}</span>}
      </div>
      <button type="submit">Submit</button>
    </form>
  );
}
```

### 规则

- 表单验证使用 zod schema（与后端共享）
- `onChange` 时清除对应字段错误
- 提交时验证全部字段
- 使用 `aria-invalid` 和 `aria-describedby` 关联错误信息

## 性能检查清单

| 检查项                           | 工具                    |
| -------------------------------- | ----------------------- |
| 不必要的重渲染                   | React DevTools Profiler |
| Bundle 大小                      | `npx next build` 输出  |
| 首屏加载时间 (LCP)              | Lighthouse              |
| 长列表性能                       | Performance tab         |
| 网络请求瀑布图                   | Network tab             |
