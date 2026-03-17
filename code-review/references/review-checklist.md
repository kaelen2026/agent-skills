---
name: review-checklist
version: 1.0.0
description: 代码审查清单，按类别分类的检查项及严重级别
last_updated: 2026-03-17
---

# 代码审查清单

## 正确性（Correctness）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 逻辑错误 | CRITICAL | 条件判断反转、运算符错误、逻辑短路 |
| Off-by-one 错误 | HIGH | 循环边界、数组索引、分页偏移 |
| Null/Undefined 处理 | HIGH | 可选链缺失、空值未检查、解构默认值 |
| 边界情况 | HIGH | 空数组、空字符串、零值、负数、超大数 |
| 异步竞态条件 | CRITICAL | 并发修改、缺少锁、Promise 顺序依赖 |
| 数据类型不匹配 | HIGH | 字符串与数字比较、隐式类型转换 |
| 死代码/不可达代码 | MEDIUM | return 后的代码、永假条件分支 |

### 检查要点

```typescript
// 常见逻辑错误：条件反转
// BAD
if (!user.isActive) {
  grantAccess(user); // 非活跃用户获得了访问权限！
}

// GOOD
if (user.isActive) {
  grantAccess(user);
}
```

```typescript
// 常见 Off-by-one
// BAD
for (let i = 0; i <= items.length; i++) { // <= 导致越界
  process(items[i]);
}

// GOOD
for (let i = 0; i < items.length; i++) {
  process(items[i]);
}
```

---

## 安全性（Security）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| SQL 注入 | CRITICAL | 字符串拼接 SQL、未使用参数化查询 |
| XSS 攻击 | CRITICAL | 未转义的用户输入直接渲染到 HTML |
| 认证绕过 | CRITICAL | 缺少认证中间件、权限检查遗漏 |
| 密钥泄露 | CRITICAL | 硬编码 API Key、密码、Token |
| 输入验证缺失 | HIGH | 用户输入未经 schema 验证直接使用 |
| CSRF 防护 | HIGH | 表单提交缺少 CSRF Token |
| 路径遍历 | HIGH | 文件路径未清理、用户可控制文件访问 |
| 敏感信息日志 | MEDIUM | 日志中包含密码、Token、个人信息 |
| CORS 配置 | MEDIUM | 过于宽松的跨域设置 |
| 依赖漏洞 | HIGH | 使用已知存在漏洞的依赖版本 |

### 检查要点

```typescript
// 密钥泄露
// BAD — CRITICAL
const apiKey = "sk-proj-abc123xyz";

// GOOD
const apiKey = process.env.API_KEY;
if (!apiKey) {
  throw new Error("API_KEY environment variable is not configured");
}
```

```typescript
// 输入验证
// BAD — HIGH
app.post("/users", (req, res) => {
  db.insert(req.body); // 未验证的用户输入
});

// GOOD
import { z } from "zod";

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150),
});

app.post("/users", (req, res) => {
  const validated = CreateUserSchema.parse(req.body);
  db.insert(validated);
});
```

---

## 性能（Performance）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| N+1 查询 | HIGH | 循环内发起数据库查询 |
| 不必要的重渲染 | MEDIUM | React 组件缺少 memo、依赖数组错误 |
| 缺少数据库索引 | HIGH | 高频查询字段未建索引 |
| 打包体积过大 | MEDIUM | 未使用 tree-shaking、整包导入大型库 |
| 内存泄漏 | HIGH | 未清理的定时器、事件监听器、订阅 |
| 不必要的同步操作 | MEDIUM | 可并行的操作串行执行 |
| 缺少缓存 | MEDIUM | 重复计算、重复请求未缓存 |
| 大数据集未分页 | HIGH | 一次性加载全部数据 |

### 检查要点

```typescript
// N+1 查询
// BAD — HIGH
const users = await db.query("SELECT * FROM users");
for (const user of users) {
  const orders = await db.query(
    "SELECT * FROM orders WHERE user_id = $1",
    [user.id]
  );
}

// GOOD
const usersWithOrders = await db.query(`
  SELECT u.*, json_agg(o.*) as orders
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id
  GROUP BY u.id
`);
```

```typescript
// 内存泄漏
// BAD — HIGH
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  // 未清理定时器！
}, []);

// GOOD
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  return () => clearInterval(interval);
}, []);
```

---

## 可维护性（Maintainability）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 命名不清晰 | MEDIUM | 变量名过短、函数名未反映行为 |
| 文件过大 | MEDIUM | 超过 400 行应考虑拆分，超过 800 行必须拆分 |
| 函数过长 | MEDIUM | 超过 50 行的函数应拆分 |
| 嵌套过深 | MEDIUM | 超过 4 层嵌套，使用提前返回或提取函数 |
| 重复代码（DRY） | MEDIUM | 相同逻辑出现 3 次以上应提取公共函数 |
| 魔法数字 | LOW | 未命名的数字常量 |
| 注释过时 | LOW | 注释与代码行为不一致 |
| 导入顺序混乱 | LOW | 未按约定排列导入语句 |
| 无用代码 | LOW | 注释掉的代码块、未使用的导入 |
| 关注点混合 | MEDIUM | 单个函数/文件承担多个职责 |

### 检查要点

```typescript
// 嵌套过深
// BAD — MEDIUM
function processOrder(order) {
  if (order) {
    if (order.items) {
      if (order.items.length > 0) {
        if (order.status === "pending") {
          // 逻辑...
        }
      }
    }
  }
}

// GOOD — 提前返回
function processOrder(order) {
  if (!order?.items?.length) return;
  if (order.status !== "pending") return;

  // 逻辑...
}
```

---

## 类型安全（Type Safety）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 使用 `any` 类型 | HIGH | 应使用具体类型或 `unknown` |
| 缺失返回类型 | MEDIUM | 公共函数应声明返回类型 |
| 不正确的泛型 | HIGH | 泛型约束缺失或过于宽松 |
| 类型断言滥用 | MEDIUM | 过度使用 `as`，应使用类型守卫 |
| 未处理的联合类型 | HIGH | switch/if 未覆盖所有联合类型分支 |
| `@ts-ignore` 使用 | HIGH | 应修复类型错误而非忽略 |

### 检查要点

```typescript
// any 类型
// BAD — HIGH
function parseResponse(data: any): any {
  return data.result;
}

// GOOD
interface ApiResponse {
  result: {
    id: string;
    name: string;
  };
}

function parseResponse(data: ApiResponse): { id: string; name: string } {
  return data.result;
}
```

---

## 错误处理（Error Handling）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 未处理的 Promise 拒绝 | HIGH | async 函数缺少 try-catch 或 .catch() |
| 系统边界缺少 try-catch | HIGH | API 调用、文件操作、数据库查询 |
| 错误信息泄露内部细节 | MEDIUM | 向用户暴露堆栈跟踪或系统路径 |
| 空 catch 块 | HIGH | 捕获异常后不做任何处理 |
| 错误未向上传播 | MEDIUM | 吞掉错误导致上层无法感知失败 |
| 缺少全局错误处理 | MEDIUM | 未配置全局 error boundary 或 handler |

### 检查要点

```typescript
// 空 catch 块
// BAD — HIGH
try {
  await saveData(data);
} catch (error) {
  // 什么都没做！错误被静默吞掉
}

// GOOD
try {
  await saveData(data);
} catch (error) {
  console.error("Failed to save data:", error);
  throw new Error("数据保存失败，请稍后重试");
}
```

---

## 测试覆盖（Testing）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 新逻辑缺少测试 | HIGH | 新增业务逻辑未编写对应测试 |
| 测试质量低 | MEDIUM | 只测试正常路径，未测试异常和边界 |
| 覆盖率缺口 | MEDIUM | 关键路径未覆盖 |
| 测试与实现耦合 | LOW | 测试依赖内部实现细节而非行为 |
| 缺少集成测试 | MEDIUM | 只有单元测试，缺少模块间交互测试 |
| 测试命名不清晰 | LOW | 无法从测试名理解测试意图 |

### 检查要点

```typescript
// 测试质量
// BAD — 只测试正常路径
test("creates user", async () => {
  const user = await createUser({ name: "Alice", email: "a@b.com" });
  expect(user.name).toBe("Alice");
});

// GOOD — 覆盖正常路径、异常和边界
describe("createUser", () => {
  test("creates user with valid input", async () => {
    const user = await createUser({ name: "Alice", email: "a@b.com" });
    expect(user.name).toBe("Alice");
  });

  test("throws on duplicate email", async () => {
    await createUser({ name: "Alice", email: "a@b.com" });
    await expect(
      createUser({ name: "Bob", email: "a@b.com" })
    ).rejects.toThrow("Email already exists");
  });

  test("throws on invalid email format", async () => {
    await expect(
      createUser({ name: "Alice", email: "not-an-email" })
    ).rejects.toThrow();
  });

  test("trims whitespace from name", async () => {
    const user = await createUser({ name: "  Alice  ", email: "a@b.com" });
    expect(user.name).toBe("Alice");
  });
});
```

---

## 不可变性（Immutability）

| 检查项 | 严重级别 | 说明 |
|--------|----------|------|
| 直接修改对象属性 | HIGH | `obj.prop = value` 而非创建新对象 |
| 数组变异方法 | HIGH | 使用 `push`、`splice`、`sort`（原地）而非不可变替代 |
| 函数参数变异 | HIGH | 修改传入的参数对象 |
| 共享状态变异 | CRITICAL | 修改全局或模块级共享对象 |
| React state 直接修改 | CRITICAL | 不通过 setter 直接修改 state |

### 检查要点

```typescript
// 对象变异
// BAD — HIGH
function updateUser(user, name) {
  user.name = name; // 变异原对象！
  return user;
}

// GOOD
function updateUser(user, name) {
  return { ...user, name };
}
```

```typescript
// 数组变异
// BAD — HIGH
function addItem(items, newItem) {
  items.push(newItem); // 变异原数组！
  return items;
}

// GOOD
function addItem(items, newItem) {
  return [...items, newItem];
}
```

```typescript
// React state 变异
// BAD — CRITICAL
const [items, setItems] = useState([]);
items.push(newItem); // 直接修改 state！
setItems(items);

// GOOD
const [items, setItems] = useState([]);
setItems(prev => [...prev, newItem]);
```

---

## 快速参考卡片

### 严重级别判断指南

| 级别 | 含义 | 典型影响 | 示例 |
|------|------|----------|------|
| **CRITICAL** | 必须在合并前修复 | 数据丢失、安全漏洞、系统崩溃 | SQL 注入、密钥泄露、竞态条件 |
| **HIGH** | 应该修复 | 功能缺陷、潜在 bug、维护困难 | 未处理异常、N+1 查询、any 类型 |
| **MEDIUM** | 改进建议 | 代码质量、可读性、性能优化 | 命名不清晰、缺少缓存、嵌套过深 |
| **LOW** | 风格/偏好 | 代码风格一致性 | 导入顺序、注释风格、魔法数字 |
