# Custom Hooks Patterns

## 命名规则

- 必须以 `use` 开头
- 使用驼峰命名
- 名称描述功能: `useToggle`, `useDebounce`, `useLocalStorage`

## 状态管理 Hook

### useToggle

最简单的自定义 hook。像灯的开关，按一下开，再按一下关。

```typescript
export function useToggle(
  initialValue = false
): [boolean, () => void] {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue((v) => !v);
  }, []);

  return [value, toggle];
}

// 使用
const [isOpen, toggleOpen] = useToggle();
```

### useLocalStorage

把 state 持久化到 localStorage，页面刷新后数据还在。

```typescript
export function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      setStoredValue((prev) => {
        const nextValue = value instanceof Function ? value(prev) : value;
        window.localStorage.setItem(key, JSON.stringify(nextValue));
        return nextValue;
      });
    },
    [key]
  );

  return [storedValue, setValue];
}

// 使用
const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
```

## 数据获取 Hook

### useQuery（简易版）

```typescript
interface UseQueryResult<T> {
  data: T | null;
  error: Error | null;
  loading: boolean;
  refetch: () => Promise<void>;
}

export function useQuery<T>(
  key: string,
  fetcher: () => Promise<T>,
  options?: { enabled?: boolean }
): UseQueryResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(false);

  const refetch = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const result = await fetcher();
      setData(result);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [fetcher]);

  useEffect(() => {
    if (options?.enabled !== false) {
      refetch();
    }
  }, [key, refetch, options?.enabled]);

  return { data, error, loading, refetch };
}

// 使用
const { data: users, loading, error, refetch } = useQuery(
  "users",
  () => fetch("/api/users").then((r) => r.json())
);
```

> **注意**: 生产项目推荐使用 SWR 或 TanStack Query，而非自制 hook。此模式用于理解数据获取 hook 的核心原理。

## 副作用 Hook

### useDebounce

像电梯门——有人持续按按钮时不会马上关门，等一会儿没人按了才关。

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
const [query, setQuery] = useState("");
const debouncedQuery = useDebounce(query, 500);

useEffect(() => {
  if (debouncedQuery) {
    performSearch(debouncedQuery);
  }
}, [debouncedQuery]);
```

### useMediaQuery

```typescript
export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() =>
    typeof window !== "undefined"
      ? window.matchMedia(query).matches
      : false
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (event: MediaQueryListEvent) => setMatches(event.matches);

    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [query]);

  return matches;
}

// 使用
const isMobile = useMediaQuery("(max-width: 768px)");
```

## Context + Reducer 模式

适用于跨多个组件共享的复杂状态。像公司公告板——所有员工看到同一份信息，通过指定动作（action）修改。

```typescript
// types
interface State {
  items: Item[];
  selectedItem: Item | null;
  loading: boolean;
}

type Action =
  | { type: "SET_ITEMS"; payload: Item[] }
  | { type: "SELECT_ITEM"; payload: Item }
  | { type: "SET_LOADING"; payload: boolean };

// reducer（纯函数，不可变更新）
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "SET_ITEMS":
      return { ...state, items: action.payload };
    case "SELECT_ITEM":
      return { ...state, selectedItem: action.payload };
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    default:
      return state;
  }
}

// Context
const ItemContext = createContext<
  { state: State; dispatch: Dispatch<Action> } | undefined
>(undefined);

export function ItemProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, {
    items: [],
    selectedItem: null,
    loading: false,
  });

  return (
    <ItemContext.Provider value={{ state, dispatch }}>
      {children}
    </ItemContext.Provider>
  );
}

export function useItems() {
  const context = useContext(ItemContext);
  if (!context) throw new Error("useItems must be used within ItemProvider");
  return context;
}
```

### 规则

- Reducer 必须是纯函数（无副作用）
- 状态更新必须不可变（spread operator）
- 缺少 Provider 时抛出明确错误
- Action type 使用描述性名称

## Hook 设计规则

| 规则                                                      | 原因                               |
| --------------------------------------------------------- | ---------------------------------- |
| 每个 hook 只做一件事                                       | 职责单一，易复用                   |
| 返回值使用数组 `[value, setter]` 或对象 `{ data, error }` | 数组适合 2 个值，对象适合多个值    |
| 副作用在 `useEffect` 中执行，并返回清理函数                | 防止内存泄漏                       |
| 依赖数组要完整                                             | 避免 stale closure                 |
| 用 `useCallback` 稳定回调引用                              | 避免不必要的 effect 重新执行       |
