---
name: tutorial
description: "新手引导技能。通过交互式练习，带用户体验 skill 的完整协作流程。三个难度级别：快速体验(4 skill)、标准流程(10 skill)、完整流程(19 skill)。"
metadata:
  priority: 2
---

# Tutorial — 交互式 Skill 引导

就像游戏的新手教程——不是读说明书，而是边玩边学。每一步都有明确目标和即时反馈。

## When to Activate

- 用户第一次使用 agent-skills
- 用户说"教程"、"tutorial"、"怎么开始"、"带我走一遍"
- 用户想了解 skill 之间如何协作

> **与 `docs/skill-usage-guide.md` 的区别**: usage-guide 是参考文档（查阅用），tutorial 是交互式引导（动手做）。

## Workflow

---

### Step 1: 选择难度

向用户展示三个路径，让用户选择：

| 级别 | 名称 | 涉及 Skill | 预计时间 | 适合谁 |
|------|------|-----------|---------|--------|
| 🟢 | Quick Start | prd → planning → api-design → database | 15min | 想快速了解核心流程 |
| 🟡 | Standard | 🟢 + auth, error-handling, backend-patterns, tdd-workflow, code-review, git-workflow | 45min | 想体验完整开发周期 |
| 🔴 | Full | 🟡 + grill-me, frontend-patterns, coding-standards, e2e-testing, security-review, ci-cd, observability, documentation, performance | 90min | 想掌握所有 skill |

如果用户犹豫，推荐 🟢 Quick Start——15 分钟就能体验核心价值。

### Step 2: 设定练习项目

默认练习项目：**待办事项 API**（Todo API）

- 足够简单，不会被业务逻辑分散注意力
- 足够完整，能触发所有 skill 的核心功能
- 技术栈：TypeScript + 任意后端框架

如果用户有自己的项目想法，也可以替换。确认后继续。

### Step 3: 引导式执行

按用户选择的难度路径，逐步引导。每一步遵循统一格式：

```
📍 当前步骤: [步骤名]
🎯 目标: [这一步要达成什么]
⚡ 激活 Skill: [skill 名称]
📝 操作: [用户需要做什么]
✅ 检查点: [怎么确认这一步完成了]
```

完成一步后，用户确认再进入下一步。不要一次性输出所有步骤。

---

### 🟢 Quick Start 路径 (4 Skills)

#### QS-1: 定义需求

```
📍 当前步骤: 需求定义
🎯 目标: 产出一份结构化的 PRD
⚡ 激活 Skill: prd
📝 操作: 告诉我"写一个待办事项 API 的 PRD"，prd skill 会引导你完成需求收集和文档生成
✅ 检查点: 生成了 PRD 文档，包含问题描述、目标用户、功能范围、非功能需求
```

#### QS-2: 制定计划

```
📍 当前步骤: 实施规划
🎯 目标: 将 PRD 拆解为可执行的实施计划
⚡ 激活 Skill: planning
📝 操作: 基于上一步的 PRD，让 planning skill 生成 spec.md 和 plan.md
✅ 检查点: 有了明确的任务拆解、依赖关系、实施顺序
```

#### QS-3: 设计 API

```
📍 当前步骤: API 设计
🎯 目标: 定义 RESTful API 端点和数据契约
⚡ 激活 Skill: api-design
📝 操作: 基于计划中的功能点，设计 Todo CRUD 的 API 端点
✅ 检查点: 有了 API 端点列表、请求/响应格式、统一响应信封 { success, data?, error? }
```

#### QS-4: 设计数据库

```
📍 当前步骤: 数据库设计
🎯 目标: 定义数据模型和表结构
⚡ 激活 Skill: database
📝 操作: 基于 API 设计，定义 Todo 表的 schema、索引、约束
✅ 检查点: 有了 schema 定义，包含主键、时间戳、索引策略
```

🎉 **Quick Start 完成！** 你已经体验了从需求到数据库设计的核心流程。想继续？进入 Standard 路径。

---

### 🟡 Standard 路径 (续接 Quick Start，+6 Skills)

#### ST-5: 认证授权

```
📍 当前步骤: 认证授权
🎯 目标: 为 API 添加身份验证和权限控制
⚡ 激活 Skill: auth
📝 操作: 设计 Todo API 的认证方案——谁能创建/编辑/删除待办事项？
✅ 检查点: 有了认证策略（JWT/Session）、权限矩阵（owner 才能编辑/删除）
```

#### ST-6: 错误处理

```
📍 当前步骤: 错误处理
🎯 目标: 建立统一的错误处理机制
⚡ 激活 Skill: error-handling
📝 操作: 定义 AppError 类、错误码体系、错误响应格式
✅ 检查点: 有了 AppError 基类、HTTP 状态码映射、用户友好的错误消息
```

#### ST-7: 后端实现

```
📍 当前步骤: 后端实现
🎯 目标: 用分层架构实现 API
⚡ 激活 Skill: backend-patterns
📝 操作: 按 Repository → Service → Controller 分层实现 Todo CRUD
✅ 检查点: 三层分离清晰，Service 层包含业务逻辑，Controller 只做请求/响应转换
```

#### ST-8: TDD 测试

```
📍 当前步骤: 测试驱动开发
🎯 目标: 用 TDD 方式为核心逻辑编写测试
⚡ 激活 Skill: tdd-workflow
📝 操作: 选一个 Service 方法，先写失败测试(RED)，再实现(GREEN)，再重构(IMPROVE)
✅ 检查点: 至少完成一个完整的 RED → GREEN → IMPROVE 循环，体验 TDD 节奏
```

#### ST-9: 代码审查

```
📍 当前步骤: 代码审查
🎯 目标: 用结构化清单审查已写代码
⚡ 激活 Skill: code-review
📝 操作: 对已实现的代码执行 code-review，检查命名、结构、错误处理、安全性
✅ 检查点: 生成了审查报告，CRITICAL/HIGH 问题已修复
```

#### ST-10: Git 工作流

```
📍 当前步骤: Git 提交
🎯 目标: 用规范的 Git 工作流提交代码
⚡ 激活 Skill: git-workflow
📝 操作: 创建 feature 分支，用 Conventional Commits 格式提交
✅ 检查点: 分支命名规范（feat/todo-api），commit message 格式正确（feat: add todo CRUD）
```

🎉 **Standard 完成！** 你已经走完了一个完整的开发周期。想挑战全部 skill？进入 Full 路径。

---

### 🔴 Full 路径 (续接 Standard，+9 Skills)

#### FL-11: 需求追问

```
📍 当前步骤: 需求追问
🎯 目标: 用结构化追问暴露需求盲区
⚡ 激活 Skill: grill-me
📝 操作: 回到 PRD，用 grill-me 对关键决策进行交叉询问
✅ 检查点: 发现了至少 2 个之前未考虑的边界情况或隐含假设
```

#### FL-12: 前端实现

```
📍 当前步骤: 前端实现
🎯 目标: 构建 Todo 列表的 UI 组件
⚡ 激活 Skill: frontend-patterns
📝 操作: 设计组件树、状态管理方案、用户交互流程
✅ 检查点: 组件职责清晰，状态提升合理，有 loading/error/empty 三态处理
```

#### FL-13: 编码规范

```
📍 当前步骤: 编码规范检查
🎯 目标: 确保代码符合项目编码标准
⚡ 激活 Skill: coding-standards
📝 操作: 对全部代码执行编码规范检查——命名、文件大小、嵌套深度、不可变性
✅ 检查点: 无 CRITICAL 违规，文件 < 800 行，函数 < 50 行，无 mutation
```

#### FL-14: E2E 测试

```
📍 当前步骤: 端到端测试
🎯 目标: 验证关键用户流程
⚡ 激活 Skill: e2e-testing
📝 操作: 为"创建待办 → 标记完成 → 删除"这个核心流程编写 E2E 测试
✅ 检查点: E2E 测试覆盖了核心 happy path，能在 CI 中运行
```

#### FL-15: 安全审查

```
📍 当前步骤: 安全审查
🎯 目标: 识别并修复安全漏洞
⚡ 激活 Skill: security-review
📝 操作: 对 API 端点执行 OWASP Top 10 检查——注入、认证、授权、数据暴露
✅ 检查点: 安全审查报告生成，CRITICAL 问题已修复
```

#### FL-16: CI/CD

```
📍 当前步骤: 持续集成/部署
🎯 目标: 配置自动化流水线
⚡ 激活 Skill: ci-cd
📝 操作: 设计 CI 流水线：lint → test → build → deploy
✅ 检查点: CI 配置文件就绪，覆盖了代码质量门禁
```

#### FL-17: 可观测性

```
📍 当前步骤: 可观测性
🎯 目标: 添加日志、监控、告警
⚡ 激活 Skill: observability
📝 操作: 添加结构化日志、requestId 追踪、关键指标监控
✅ 检查点: 日志格式统一（JSON），requestId 贯穿全链路
```

#### FL-18: 文档

```
📍 当前步骤: 文档编写
🎯 目标: 编写项目文档
⚡ 激活 Skill: documentation
📝 操作: 编写 README（快速开始、API 文档链接）和 ADR（关键技术决策记录）
✅ 检查点: README 能让新人 5 分钟内跑起项目，ADR 记录了至少 1 个关键决策
```

#### FL-19: 性能优化

```
📍 当前步骤: 性能优化
🎯 目标: 识别并优化性能瓶颈
⚡ 激活 Skill: performance
📝 操作: 对数据库查询做 EXPLAIN ANALYZE，检查 N+1 问题，设定性能预算
✅ 检查点: 关键查询有索引支撑，无 N+1，响应时间在预算内
```

🎉 **Full 路径完成！** 你已经体验了全部 19 个 skill 的协作流程。

---

### Step 4: 进度追踪

在引导过程中维护进度清单，每完成一步打勾：

**🟢 Quick Start**
- [ ] QS-1: prd — 需求定义
- [ ] QS-2: planning — 实施规划
- [ ] QS-3: api-design — API 设计
- [ ] QS-4: database — 数据库设计

**🟡 Standard** (续)
- [ ] ST-5: auth — 认证授权
- [ ] ST-6: error-handling — 错误处理
- [ ] ST-7: backend-patterns — 后端实现
- [ ] ST-8: tdd-workflow — TDD 测试
- [ ] ST-9: code-review — 代码审查
- [ ] ST-10: git-workflow — Git 工作流

**🔴 Full** (续)
- [ ] FL-11: grill-me — 需求追问
- [ ] FL-12: frontend-patterns — 前端实现
- [ ] FL-13: coding-standards — 编码规范
- [ ] FL-14: e2e-testing — E2E 测试
- [ ] FL-15: security-review — 安全审查
- [ ] FL-16: ci-cd — CI/CD
- [ ] FL-17: observability — 可观测性
- [ ] FL-18: documentation — 文档
- [ ] FL-19: performance — 性能优化

### Step 5: 总结回顾

全部（或所选路径）完成后，输出回顾总结：

1. **完成了哪些 skill** — 列出已体验的 skill 和各自的输出物
2. **核心模式** — 强调贯穿全程的模式：
   - 每个 skill 都有设计审查（checklist 验证）
   - skill 之间有输入/输出依赖（上游输出 = 下游输入）
   - 不可变性、结论先行、TDD 是不可协商的原则
3. **下一步建议** — 根据完成的级别推荐：
   - 🟢 完成 → 推荐尝试 🟡 Standard
   - 🟡 完成 → 推荐在真实项目中使用，遇到问题再尝试 🔴 Full
   - 🔴 完成 → 推荐阅读 `docs/skill-usage-guide.md` 深入理解协作细节
