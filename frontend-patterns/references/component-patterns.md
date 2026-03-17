# Component Patterns

## 组合模式（Composition）— 默认首选

组件就像乐高积木：每块积木独立成型，自由拼装组合。继承则像俄罗斯套娃，内外层紧密耦合。

```typescript
// Composition: 小积木自由拼装
interface CardProps {
  children: React.ReactNode;
  variant?: "default" | "outlined";
}

export function Card({ children, variant = "default" }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>;
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>;
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>;
}

// 使用: 自由组合
<Card>
  <CardHeader>Title</CardHeader>
  <CardBody>Content</CardBody>
</Card>
```

### 规则

- 优先使用 `children` 传递内容
- 每个子组件独立、可复用
- 避免在父组件中硬编码子组件结构

## Compound Components 模式

多个组件共享隐式状态，像一组遥控器按钮，每个按钮独立但都控制同一台电视。

```typescript
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | undefined>(undefined);

function useTabs() {
  const context = useContext(TabsContext);
  if (!context) throw new Error("Tab must be used within Tabs");
  return context;
}

export function Tabs({
  children,
  defaultTab,
}: {
  children: React.ReactNode;
  defaultTab: string;
}) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}

export function TabList({ children }: { children: React.ReactNode }) {
  return (
    <div role="tablist" className="tab-list">
      {children}
    </div>
  );
}

export function Tab({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  const { activeTab, setActiveTab } = useTabs();

  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      className={activeTab === id ? "active" : ""}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

export function TabPanel({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  const { activeTab } = useTabs();
  if (activeTab !== id) return null;

  return (
    <div role="tabpanel" aria-labelledby={id}>
      {children}
    </div>
  );
}
```

### 适用场景

- 一组紧密关联的组件需要共享状态（Tabs, Accordion, Dropdown）
- API 使用者需要灵活排列子组件

### 规则

- Context Provider 放在最外层容器组件中
- 缺少 Provider 时抛出明确错误
- 每个子组件通过自定义 hook 访问共享状态

## Render Props 模式

将渲染逻辑的控制权交给调用者。像餐厅提供食材和厨房，客人自己决定做什么菜。

```typescript
interface DataLoaderProps<T> {
  url: string;
  children: (
    data: T | null,
    loading: boolean,
    error: Error | null
  ) => React.ReactNode;
}

export function DataLoader<T>({ url, children }: DataLoaderProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then((res) => res.json())
      .then(setData)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return <>{children(data, loading, error)}</>;
}

// 使用
<DataLoader<User[]> url="/api/users">
  {(users, loading, error) => {
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <UserList users={users!} />;
  }}
</DataLoader>;
```

### 适用场景

- 复用数据获取/状态逻辑，但每次渲染 UI 不同
- 现代项目中通常被自定义 hook 替代，但在特定场景（如 slot 模式）仍有价值

## Error Boundary 模式

错误边界像防火门——一个房间着火不会烧到整栋楼。

```typescript
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends React.Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        this.props.fallback ?? (
          <div role="alert">
            <h2>Something went wrong</h2>
            <button onClick={() => this.setState({ hasError: false, error: null })}>
              Try again
            </button>
          </div>
        )
      );
    }

    return this.props.children;
  }
}
```

### 规则

- 在应用根级别和关键 UI 区域各放一个 Error Boundary
- 提供有意义的 fallback UI
- 记录错误到监控服务（通过 `onError` 回调）
- Error Boundary 只捕获渲染错误，不捕获事件处理器和异步错误

## 模式选择指南

| 场景                         | 推荐模式              |
| ---------------------------- | --------------------- |
| 通用 UI 组件                 | 组合模式              |
| 紧密关联的组件组             | Compound Components   |
| 复用逻辑 + 自定义渲染       | Render Props / Hook   |
| 错误隔离                     | Error Boundary        |
| 复杂状态逻辑提取             | 自定义 Hook           |
