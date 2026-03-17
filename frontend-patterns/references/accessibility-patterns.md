# Accessibility & Animation Patterns

## 键盘导航

可访问性就像建筑物的无障碍通道——不是额外功能，而是基本要求。每个用鼠标能操作的交互，必须也能用键盘完成。

### Dropdown 键盘导航

```typescript
export function Dropdown({
  options,
  onSelect,
}: {
  options: string[];
  onSelect: (option: string) => void;
}) {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(0);

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      switch (e.key) {
        case "ArrowDown":
          e.preventDefault();
          setActiveIndex((i) => Math.min(i + 1, options.length - 1));
          break;
        case "ArrowUp":
          e.preventDefault();
          setActiveIndex((i) => Math.max(i - 1, 0));
          break;
        case "Enter":
          e.preventDefault();
          onSelect(options[activeIndex]);
          setIsOpen(false);
          break;
        case "Escape":
          setIsOpen(false);
          break;
      }
    },
    [activeIndex, options, onSelect]
  );

  return (
    <div
      role="combobox"
      aria-expanded={isOpen}
      aria-haspopup="listbox"
      aria-activedescendant={isOpen ? `option-${activeIndex}` : undefined}
      onKeyDown={handleKeyDown}
      tabIndex={0}
    >
      <button onClick={() => setIsOpen((prev) => !prev)}>
        Select option
      </button>
      {isOpen && (
        <ul role="listbox">
          {options.map((option, index) => (
            <li
              key={option}
              id={`option-${index}`}
              role="option"
              aria-selected={index === activeIndex}
              onClick={() => {
                onSelect(option);
                setIsOpen(false);
              }}
            >
              {option}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### 必需的键盘操作

| 组件              | 键盘操作                                  |
| ----------------- | ----------------------------------------- |
| Button            | `Enter`, `Space` 触发 click               |
| Dropdown / Select | `ArrowUp/Down` 导航, `Enter` 选择, `Esc` 关闭 |
| Modal / Dialog    | `Esc` 关闭, `Tab` 在内部循环              |
| Tabs              | `ArrowLeft/Right` 切换, `Enter` 激活      |
| Menu              | `ArrowUp/Down` 导航, `Enter` 选择         |

## 焦点管理

### Modal 焦点陷阱

Modal 打开时，焦点必须被"锁"在 Modal 内部，关闭时回到触发元素。像进入一个房间——进去后只能在房间里走动，出来后回到门口。

```typescript
export function Modal({
  isOpen,
  onClose,
  children,
}: {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // 保存当前焦点
      previousFocusRef.current = document.activeElement as HTMLElement;
      // 聚焦到 Modal
      modalRef.current?.focus();
    } else {
      // 恢复焦点
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  // 焦点陷阱: Tab 键在 Modal 内循环
  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === "Escape") {
        onClose();
        return;
      }

      if (e.key !== "Tab" || !modalRef.current) return;

      const focusableElements = modalRef.current.querySelectorAll<HTMLElement>(
        'a[href], button:not([disabled]), textarea, input, select, [tabindex]:not([tabindex="-1"])'
      );
      const firstFocusable = focusableElements[0];
      const lastFocusable = focusableElements[focusableElements.length - 1];

      if (e.shiftKey) {
        if (document.activeElement === firstFocusable) {
          e.preventDefault();
          lastFocusable.focus();
        }
      } else {
        if (document.activeElement === lastFocusable) {
          e.preventDefault();
          firstFocusable.focus();
        }
      }
    },
    [onClose]
  );

  if (!isOpen) return null;

  return (
    <>
      <div className="modal-overlay" onClick={onClose} aria-hidden="true" />
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        tabIndex={-1}
        onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    </>
  );
}
```

## ARIA 属性速查

| 属性                    | 用途                              | 示例                                |
| ----------------------- | --------------------------------- | ----------------------------------- |
| `role`                  | 定义元素的语义角色                | `role="dialog"`, `role="tablist"`   |
| `aria-label`            | 不可见的文本标签                  | `aria-label="Close menu"`           |
| `aria-labelledby`       | 引用可见的标签元素                | `aria-labelledby="heading-id"`      |
| `aria-describedby`      | 引用描述性文本                    | `aria-describedby="error-id"`       |
| `aria-expanded`         | 展开/折叠状态                     | `aria-expanded={isOpen}`            |
| `aria-selected`         | 选中状态                          | `aria-selected={isActive}`          |
| `aria-invalid`          | 验证失败状态                      | `aria-invalid={!!error}`            |
| `aria-modal`            | 模态对话框标识                    | `aria-modal="true"`                 |
| `aria-live`             | 动态内容更新通知                  | `aria-live="polite"`                |
| `aria-hidden`           | 对辅助技术隐藏                    | `aria-hidden="true"`（装饰性元素） |

## Framer Motion 动画

### 列表动画

```typescript
import { motion, AnimatePresence } from "framer-motion";

export function AnimatedList({ items }: { items: Item[] }) {
  return (
    <AnimatePresence>
      {items.map((item) => (
        <motion.div
          key={item.id}
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -20 }}
          transition={{ duration: 0.3 }}
        >
          <ItemCard item={item} />
        </motion.div>
      ))}
    </AnimatePresence>
  );
}
```

### Modal 动画

```typescript
export function AnimatedModal({
  isOpen,
  onClose,
  children,
}: {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="modal-overlay"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.div
            className="modal-content"
            role="dialog"
            aria-modal="true"
            initial={{ opacity: 0, scale: 0.95, y: 20 }}
            animate={{ opacity: 1, scale: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.95, y: 20 }}
            transition={{ type: "spring", damping: 25, stiffness: 300 }}
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
}
```

### 动画规则

| 规则                                      | 原因                               |
| ----------------------------------------- | ---------------------------------- |
| 动画时长 200-400ms                        | 太短看不见，太长觉得卡             |
| 使用 `AnimatePresence` 处理退出动画        | 组件卸载前播放退出动画             |
| 尊重 `prefers-reduced-motion`              | 部分用户对动画敏感                 |
| 不在关键操作路径上阻塞                     | 动画不应延迟用户操作               |

### 尊重 Reduced Motion 偏好

```typescript
export function useReducedMotion(): boolean {
  return useMediaQuery("(prefers-reduced-motion: reduce)");
}

// 使用
const prefersReduced = useReducedMotion();

<motion.div
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: prefersReduced ? 0 : 0.3 }}
>
  Content
</motion.div>;
```

## 可访问性检查清单

- [ ] 所有交互元素可通过键盘操作
- [ ] 焦点指示器可见（不移除 `outline`）
- [ ] Modal/Dialog 实现焦点陷阱
- [ ] 图片有 `alt` 属性（装饰性图片用 `alt=""`）
- [ ] 表单字段有关联的 label
- [ ] 错误消息通过 `aria-describedby` 关联
- [ ] 颜色对比度满足 WCAG AA（4.5:1 正文，3:1 大文本）
- [ ] 动画尊重 `prefers-reduced-motion`
- [ ] 页面语义结构正确（heading 层级、landmark）
- [ ] 动态内容使用 `aria-live` 通知
