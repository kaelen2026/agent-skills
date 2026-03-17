---
name: coding-standards
version: 1.0.0
description: "通用编码规范技能。覆盖 TypeScript/JavaScript、React、Node.js 的命名、模式、反模式。"
last_updated: 2026-03-17
---

# Coding Standards

## When to Activate

- 开始新项目或模块时
- 代码质量审查和重构
- 执行命名、格式、结构一致性检查
- 设置 lint/format/type-check 规则
- 新成员了解项目编码规范

## 核心原则

| 原则 | 含义 | 关键规则 |
|------|------|----------|
| **不可变性** | 永远创建新对象，不修改原对象 | spread operator, 禁止 `.push()` 直接 mutation |
| **可读性优先** | 代码被读的次数远多于写 | 清晰命名 > 注释 > 聪明技巧 |
| **KISS** | 用最简单的方案解决问题 | 不过度工程化，不提前优化 |
| **DRY** | 不重复自己 | 提取公共逻辑，但避免过早抽象 |
| **YAGNI** | 不需要就不做 | 不构建未被请求的功能 |

## Workflow

### 1. 分析当前代码

检查现有代码是否遵循以下规范（参见 references/）：

- [references/typescript-standards.md](references/typescript-standards.md) — 命名、类型、不可变性、异步模式
- [references/react-patterns.md](references/react-patterns.md) — 组件结构、Hooks、状态管理、渲染模式
- [references/testing-and-quality.md](references/testing-and-quality.md) — 测试标准、代码异味检测、性能

### 2. 输出审查结果

按严重程度排序输出问题清单（结论先行）。

### 3. 设计审查清单

- [ ] 变量/函数命名清晰（动词-名词模式）
- [ ] 不可变性（无直接 mutation）
- [ ] 函数小于 50 行
- [ ] 文件小于 800 行
- [ ] 嵌套不超过 4 层（使用 early return）
- [ ] 无 `any` 类型
- [ ] 无魔法数字（使用命名常量）
- [ ] 适当的错误处理（try-catch + 有意义的消息）
- [ ] 并行执行独立的异步操作（`Promise.all`）
- [ ] 无 `console.log`（使用结构化日志）
- [ ] 输入验证使用 zod
- [ ] 公共 API 有 JSDoc 文档
- [ ] 测试遵循 AAA 模式（Arrange-Act-Assert）
- [ ] 测试名称描述行为而非实现
