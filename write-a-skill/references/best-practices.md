# 项目最佳实践

> 从 20 个技能的开发过程中总结出的模式与反模式。

## 核心原则

### 1. 单一 Owner 原则

每个关注点只有一个技能负责定义，其他技能引用。

```
✅ 正确: api-design 定义响应信封 → error-handling 引用 api-design
❌ 错误: api-design 和 error-handling 各自定义响应信封格式
```

当前 Ownership Matrix：

| 关注点 | Owner | 引用者 |
|--------|-------|--------|
| 统一响应信封 | api-design | auth, backend-patterns, error-handling |
| 结构化日志 | observability | backend-patterns, error-handling, security-review |
| N+1 查询 | database (schema) + backend-patterns (app) | performance (审计) |
| 限流实现 | backend-patterns | security-review (审计), auth (auth 端点) |
| 认证安全 | auth | security-review (通用 OWASP 审计) |
| Pre-commit hooks | git-workflow | — |

新增技能前必须检查此矩阵，避免重复定义。

### 2. 类比先行原则

技术概念用日常比喻开头，降低认知门槛。

```
✅ "就像医生先问诊再开药一样，先收集需求再写代码"
❌ "本技能提供需求收集的标准化工作流"
```

### 3. 结论先行原则

每个章节、每个步骤先给结论，再展开。

```
✅ "用 Server Actions 处理表单提交。因为它与缓存集成，支持渐进增强。"
❌ "考虑到 Next.js 的缓存机制和渐进增强需求...（10 行后）...所以用 Server Actions。"
```

### 4. 模板驱动原则

References 中提供可直接复制使用的模板，而非抽象描述。

```
✅ 提供带 {占位符} 的完整模板文件
❌ 描述"应该包含以下内容..."但不给模板
```

## 文件规模指南

| 文件类型 | 目标行数 | 最小 | 最大 |
|----------|----------|------|------|
| SKILL.md | 200-400 | 150 | 500 |
| Reference 文件 | 100-300 | 80 | 400 |
| 技能总 references 行数 | 400-800 | 300 | 1200 |

超过上限说明内容应拆分；低于下限说明内容不够充实。

## 命名约定

### 目录名

- kebab-case: `write-a-skill`, `error-handling`, `tdd-workflow`
- 与 frontmatter `name` 字段一致
- 简短但有描述性（2-3 个词）

### Reference 文件名

- kebab-case: `schema-design.md`, `review-checklist.md`
- 格式: `{主题}.md` 或 `{主题}-{类型}.md`
- 类型后缀: `-template`, `-patterns`, `-checklist`, `-strategies`, `-rules`

### 常见类型后缀

| 后缀 | 用途 | 示例 |
|------|------|------|
| `-template` | 可复制的模板 | `prd-template.md`, `spec-template.md` |
| `-patterns` | 代码模式集合 | `mocking-patterns.md`, `migration-patterns.md` |
| `-checklist` | 检查清单 | `review-checklist.md`, `security-checklist.md` |
| `-strategies` | 策略选择指南 | `caching-strategies.md`, `deployment-strategies.md` |
| `-rules` | 规则和约定 | `design-rules.md` |

## Workflow 设计模式

### 模式 A: 收集→设计→生成→审查（最常用）

适用于大多数生成型技能（prd、api-design、database 等）。

```
Phase 1: 需求收集（提问、分析现有代码）
Phase 2: 方案设计（架构决策、技术选型）
Phase 3: 产出生成（代码、文档、配置）
Phase 4: 质量审查（checklist 逐项验证）
```

### 模式 B: 分析→分类→处理→报告

适用于审查型技能（code-review、security-review 等）。

```
Phase 1: 输入分析（读取代码、理解上下文）
Phase 2: 问题分类（按严重度分级）
Phase 3: 逐项处理（给出修复建议）
Phase 4: 汇总报告（统计 + 优先级排序）
```

### 模式 C: 提问→深挖→总结→输出

适用于对话型技能（grill-me、tutorial 等）。

```
Phase 1: 初始提问（开放式问题）
Phase 2: 追问深挖（基于回答的针对性追问）
Phase 3: 发现总结（结构化整理发现）
Phase 4: 输出交付（文档/建议/行动项）
```

## 反模式

### 1. 万能技能

```
❌ 一个技能试图覆盖"全栈开发"
✅ 拆分为 frontend-patterns + backend-patterns + database
```

每个技能聚焦一个领域，通过交叉引用组合。

### 2. 空洞审查清单

```
❌ - [ ] 代码质量好
❌ - [ ] 性能可接受
✅ - [ ] 函数不超过 50 行
✅ - [ ] API 响应时间 < 200ms P95
```

检查项必须可客观验证，有明确的通过/失败标准。

### 3. 重复定义

```
❌ error-handling 和 api-design 都定义了错误响应格式
✅ api-design 定义格式（owner），error-handling 引用它
```

### 4. 过度抽象

```
❌ "根据项目需求选择合适的方案"
✅ 提供决策表格，列出条件→推荐方案
```

### 5. 缺少代码示例

```
❌ "应该使用参数化查询防止 SQL 注入"
✅ 给出 Prisma/Drizzle/Knex 的具体代码示例
```

### 6. Pattern 冲突

```
❌ 两个技能的 filePattern 都匹配 **/*.ts（浪费注入配额）
✅ 用更精确的 pattern: **/errors.ts vs **/components/**/*.tsx
```

注意 MAX_SKILLS=3 的限制。Pattern 过于宽泛会导致低优先级技能永远无法注入。

## 技能生命周期

### 创建

1. 确认需求和定位
2. 检查 Ownership Matrix
3. 编写 SKILL.md + references
4. 更新 README.md
5. 创建符号链接安装

### 维护

- 定期检查 references 是否过时（框架版本更新等）
- 根据使用反馈调整 Workflow 步骤
- 监控 pattern 匹配率，调整 filePattern/bashPattern

### 合并/拆分

- 两个技能 >60% 内容重叠 → 合并（如 write-a-prd + prd-to-spec → prd）
- 一个技能 >500 行且覆盖多个领域 → 拆分
- 合并时更新 Ownership Matrix 和 README

## 质量标准速查

| 维度 | 标准 |
|------|------|
| 类比 | 第一段有日常比喻 |
| 触发 | ≥3 个 When to Activate 场景 |
| 工作流 | 3-7 个 Phase，每步有输入/输出 |
| 审查 | ≥8 个可验证的 checkbox 项 |
| References | 2-4 个文件，每个 100-300 行 |
| 交叉引用 | 只引用 owner，不重复定义 |
| 语言 | 中文说明 + 英文代码 |
| 行数 | SKILL.md 200-400 行 |
