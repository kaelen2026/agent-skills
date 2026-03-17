# 测试标准与代码质量

## 测试模式

### AAA 模式（Arrange-Act-Assert）

```typescript
test("calculates similarity correctly", () => {
  // Arrange
  const vector1 = [1, 0, 0];
  const vector2 = [0, 1, 0];

  // Act
  const similarity = calculateCosineSimilarity(vector1, vector2);

  // Assert
  expect(similarity).toBe(0);
});
```

### 测试命名

用自然语言描述**行为**，而非实现细节：

```typescript
// GOOD: 描述行为
test("returns empty array when no markets match query", () => {});
test("throws error when API key is missing", () => {});
test("falls back to substring search when Redis unavailable", () => {});

// BAD: 含糊不清
test("works", () => {});
test("test search", () => {});
test("handles error", () => {});
```

### 测试结构

```
__tests__/
├── unit/              # 单元测试（纯函数、工具）
├── integration/       # 集成测试（API、数据库交互）
└── e2e/              # 端到端测试（用户流程）
```

### 测试原则

| 原则 | 说明 |
|------|------|
| 每个测试一个断言 | 一个测试验证一件事 |
| 独立运行 | 测试之间无依赖和顺序要求 |
| 快速执行 | 单元测试应在毫秒级完成 |
| 可重复 | 每次运行结果一致 |
| 覆盖边界 | 空值、边界值、异常路径 |

### Mock 规则

```typescript
// GOOD: 只 mock 外部依赖
vi.mock("@/lib/api/openai", () => ({
  generateEmbedding: vi.fn().mockResolvedValue([0.1, 0.2, 0.3]),
}));

// BAD: mock 被测试的内部函数
// 这意味着你测试的是 mock 而非真实代码
```

## 代码异味检测

### 1. 长函数（> 50 行）

```typescript
// BAD
function processMarketData() {
  // 100 行代码...
}

// GOOD: 拆分为子函数
function processMarketData() {
  const validated = validateData();
  const transformed = transformData(validated);
  return saveData(transformed);
}
```

### 2. 深嵌套（> 4 层）

```typescript
// BAD: 5 层嵌套
if (user) {
  if (user.isAdmin) {
    if (market) {
      if (market.isActive) {
        if (hasPermission) {
          // ...
        }
      }
    }
  }
}

// GOOD: early return
if (!user) return;
if (!user.isAdmin) return;
if (!market) return;
if (!market.isActive) return;
if (!hasPermission) return;
// ...
```

### 3. 上帝对象/函数

```typescript
// BAD: 一个函数做太多事
function handleSubmit() {
  validateForm();
  transformData();
  callApi();
  updateCache();
  showNotification();
  redirectUser();
  logAnalytics();
}

// GOOD: 每个函数单一职责，在上层编排
async function handleSubmit() {
  const data = validateAndTransform(formData);
  const result = await submitToApi(data);
  handlePostSubmit(result);
}
```

### 4. 布尔参数

```typescript
// BAD: 调用处无法理解含义
createUser(data, true, false);

// GOOD: 使用对象参数
createUser(data, { sendEmail: true, isAdmin: false });
```

### 5. 过长的导入列表

```typescript
// BAD: 从一个模块导入太多
import { a, b, c, d, e, f, g, h } from "./utils";

// GOOD: 模块拆分后各自导入
import { formatDate, parseDate } from "./utils/date";
import { validateEmail, validatePhone } from "./utils/validation";
```

## 代码异味速查表

| 异味 | 信号 | 解决方案 |
|------|------|----------|
| 长函数 | > 50 行 | 提取子函数 |
| 深嵌套 | > 4 层 | early return / guard clause |
| 大文件 | > 800 行 | 按职责拆分文件 |
| 魔法数字 | 无名数字字面量 | 提取为命名常量 |
| 重复代码 | 3+ 处相似代码 | 提取公共函数 |
| `any` 类型 | TypeScript 中使用 any | 定义明确类型 |
| 布尔参数 | 函数接收 boolean 参数 | 改用对象参数或拆分函数 |
| 注释掉的代码 | 被注释的旧代码 | 删除（git 有历史） |
| `console.log` | 生产代码中的 console.log | 使用结构化日志 |
| 回调地狱 | 多层嵌套回调 | async/await |

## 性能检查

### 数据库查询

```typescript
// GOOD: 只查询需要的列
const { data } = await supabase
  .from("markets")
  .select("id, name, status")
  .limit(10);

// BAD: 查询全部列
const { data } = await supabase.from("markets").select("*");
```

### Bundle 大小

- 使用动态 `import()` 延迟加载大模块
- 避免导入整个库（`import { debounce } from 'lodash'` 而非 `import _ from 'lodash'`）
- 定期检查 `next build` 的 bundle analyzer 输出

### 渲染性能

- 列表使用稳定的 `key`（不用 index）
- 大列表使用虚拟滚动（`@tanstack/react-virtual`）
- 避免在渲染路径中创建新对象/数组
