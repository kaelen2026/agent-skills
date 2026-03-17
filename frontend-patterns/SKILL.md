---
name: frontend-patterns
version: 1.0.0
description: "前端开发模式技能。React 组件设计、状态管理、性能优化、可访问性最佳实践。"
last_updated: 2026-03-17
---

# Frontend Patterns

## When to Activate

- 用户构建 React 组件（组合、props、渲染模式）
- 用户设计状态管理（useState, useReducer, Context, Zustand）
- 用户实现数据获取（SWR, React Query, Server Components）
- 用户优化性能（memoization, virtualization, code splitting）
- 用户处理表单（验证、受控输入、Zod schema）
- 用户构建可访问的 UI 组件
- 用户添加动画交互（Framer Motion, CSS transitions）

## Workflow

### 1. 现状分析

分析现有前端代码，确认以下内容：

| 项目             | 说明                                           | 确认方法                            |
| ---------------- | ---------------------------------------------- | ----------------------------------- |
| 框架             | React / Next.js / Remix 版本                   | 确认 `package.json`                 |
| 状态管理         | useState / useReducer / Zustand / Redux / Jotai | 搜索 store 定义和 Context           |
| 数据获取         | SWR / React Query / Server Components / fetch  | 搜索 hooks 和 fetch 调用            |
| 样式方案         | Tailwind / CSS Modules / styled-components     | 确认样式导入和配置                  |
| 组件模式         | 是否使用组合模式、compound components           | 分析现有组件结构                    |
| 表单处理         | React Hook Form / Formik / 原生受控组件        | 搜索表单相关 hooks                  |
| 测试框架         | Vitest / Jest / Testing Library                | 确认测试配置                        |

### 2. 组件设计（结论先行）

先输出组件层级总览，再展开细节：

```
## 组件层级总览

ComponentName/
├── ComponentName.tsx       # 主组件
├── ComponentName.test.tsx  # 测试
├── useComponentLogic.ts    # 自定义 hook（如有复杂逻辑）
├── types.ts                # 类型定义
└── index.ts                # 公共导出
```

### 3. 模式选择与实现

根据需求选择合适的模式：

#### A. 组件模式

- 参考 [references/component-patterns.md](references/component-patterns.md)
- 组合模式（默认首选）
- Compound Components（关联组件组）
- Render Props（灵活渲染逻辑）

#### B. 自定义 Hooks

- 参考 [references/hooks-patterns.md](references/hooks-patterns.md)
- 状态封装 hook
- 数据获取 hook
- Debounce / throttle hook

#### C. 性能优化

- 参考 [references/performance-patterns.md](references/performance-patterns.md)
- Memoization（useMemo, useCallback, React.memo）
- Code Splitting（lazy, Suspense）
- 虚拟化长列表

#### D. 可访问性与动画

- 参考 [references/accessibility-patterns.md](references/accessibility-patterns.md)
- 键盘导航
- 焦点管理
- ARIA 属性
- Framer Motion 动画

### 4. 设计审查

生成完成后，自动检查以下规则：

- [ ] 组合优于继承（Composition over Inheritance）
- [ ] Props 使用 TypeScript 接口定义
- [ ] 组件职责单一，不超过 200 行
- [ ] 自定义 hook 以 `use` 开头命名
- [ ] 状态不可变更新（spread operator, 不直接 mutate）
- [ ] 昂贵计算使用 `useMemo` 包裹
- [ ] 传递给子组件的回调使用 `useCallback` 包裹
- [ ] 大型列表使用虚拟化
- [ ] 重型组件使用 `lazy` + `Suspense` 按需加载
- [ ] 表单输入有验证（zod schema）
- [ ] 交互元素支持键盘操作
- [ ] 正确使用 ARIA 属性
- [ ] Modal/Dialog 有焦点管理
- [ ] Error Boundary 包裹关键 UI 区域
- [ ] 无 `any` 类型使用
- [ ] 无直接 DOM 操作（使用 ref）

## Output Format

```
# Frontend Pattern: {模式名称}

## 组件层级
{组件结构总览}

## 实现代码
{组件/hook 代码生成}

## 使用示例
{用法示例}

## 测试
{测试代码}

## 设计审查
{审查结果清单，标记通过/未通过}
```
