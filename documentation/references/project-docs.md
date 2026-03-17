---
name: project-docs
version: 1.0.0
description: 项目文档模板集合 — README、CONTRIBUTING、CHANGELOG 及 GitHub 模板
last_updated: 2026-03-17
---

# 项目文档模板

## README 模板

README 是项目的"第一印象"。目标：新人在 5 分钟内理解项目用途并成功运行。

```markdown
# 项目名称

> 一句话描述项目的核心价值和用途。

## 快速开始

三步启动项目：

1. 克隆仓库
   ```bash
   git clone https://github.com/your-org/your-project.git
   cd your-project
   ```

2. 安装依赖
   ```bash
   npm install
   ```

3. 启动开发服务器
   ```bash
   npm run dev
   ```

## 前置条件

- Node.js >= 20.0.0
- npm >= 10.0.0
- PostgreSQL >= 16（或其他依赖的服务）

## 安装

详细的安装步骤（如果快速开始不够用）：

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑环境变量
vim .env

# 安装依赖
npm install

# 初始化数据库
npm run db:migrate
npm run db:seed
```

## 使用示例

### 基本用法

```typescript
import { createClient } from "your-project";

const client = createClient({
  apiKey: process.env.API_KEY,
});

const result = await client.doSomething({
  input: "hello",
});

console.log(result);
```

### 进阶用法

```typescript
// 带配置的高级用例
const client = createClient({
  apiKey: process.env.API_KEY,
  timeout: 5000,
  retries: 3,
});
```

## 项目结构

```
your-project/
├── src/
│   ├── api/          # API 路由和控制器
│   ├── services/     # 业务逻辑
│   ├── models/       # 数据模型
│   ├── utils/        # 工具函数
│   └── index.ts      # 入口文件
├── tests/            # 测试文件
├── docs/             # 文档
├── .env.example      # 环境变量模板
├── package.json
└── README.md
```

## 贡献

欢迎贡献代码！请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解开发流程和规范。

## 许可证

[MIT](LICENSE)
```

### README 编写原则

- **快速开始不超过 3 步**：降低上手门槛
- **示例使用真实场景**：不要用 `foo` / `bar`
- **保持更新**：版本号、依赖版本、API 变更同步更新
- **项目结构用树形图**：一目了然

---

## CONTRIBUTING.md 模板

```markdown
# 贡献指南

感谢你对本项目的关注！以下是参与贡献的流程和规范。

## 开发环境搭建

```bash
# 1. Fork 并克隆仓库
git clone https://github.com/your-username/your-project.git
cd your-project

# 2. 安装依赖
npm install

# 3. 创建开发分支
git checkout -b feat/your-feature

# 4. 启动开发环境
npm run dev
```

## 分支命名规范

| 前缀 | 用途 | 示例 |
|------|------|------|
| `feat/` | 新功能 | `feat/user-authentication` |
| `fix/` | Bug 修复 | `fix/login-redirect-loop` |
| `refactor/` | 重构 | `refactor/extract-auth-service` |
| `docs/` | 文档 | `docs/api-usage-examples` |
| `test/` | 测试 | `test/payment-edge-cases` |
| `chore/` | 构建/工具 | `chore/upgrade-typescript` |

## 提交规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/) 格式：

```
<type>: <description>

<optional body>
```

类型说明：

- `feat`: 新功能
- `fix`: Bug 修复
- `refactor`: 重构（不改变行为）
- `docs`: 文档变更
- `test`: 测试相关
- `chore`: 构建工具或辅助变更
- `perf`: 性能优化
- `ci`: CI/CD 配置

示例：

```
feat: add cursor-based pagination to product list API

Replaces OFFSET/LIMIT pagination to resolve deep-page
performance degradation on tables with >5M rows.
```

## PR 流程

1. **确保测试通过**
   ```bash
   npm run test
   npm run lint
   ```

2. **创建 PR**
   - 标题简洁明了（70 字符以内）
   - 描述中说明"改了什么"和"为什么改"
   - 关联相关 Issue（`Closes #123`）

3. **代码审查**
   - 至少 1 位维护者 approve
   - 所有 CI 检查通过
   - 无未解决的 review 评论

4. **合并**
   - 使用 Squash and Merge
   - 合并后删除远程分支

## 代码风格

- 使用 ESLint + Prettier 保持一致
- 运行 `npm run lint` 检查
- 运行 `npm run format` 自动格式化
- 函数不超过 50 行
- 文件不超过 800 行
- 嵌套不超过 4 层
- 使用不可变模式（spread operator），禁止直接修改对象
```

---

## CHANGELOG 格式

遵循 [Keep a Changelog](https://keepachangelog.com/) 规范。

```markdown
# 变更日志

本项目的所有重要变更都会记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/)，
版本号遵循 [Semantic Versioning](https://semver.org/)。

## [Unreleased]

### Added（新增）
- 用户头像上传功能 (#234)

### Changed（变更）
- 列表接口迁移到游标分页 (#256)

## [1.2.0] - 2026-03-10

### Added（新增）
- 支持 OAuth2 第三方登录 (#200)
- 新增商品搜索过滤器 (#210)

### Changed（变更）
- 优化首页加载性能，减少 40% 请求数 (#215)

### Fixed（修复）
- 修复支付回调偶发超时问题 (#220)
- 修复移动端表格横向溢出 (#218)

## [1.1.0] - 2026-02-15

### Added（新增）
- 邮件通知功能 (#180)

### Deprecated（弃用）
- 旧版 webhook 接口，将在 2.0 移除 (#185)

### Removed（移除）
- 移除对 Node.js 16 的支持 (#175)

### Security（安全）
- 升级 jsonwebtoken 修复 CVE-2026-XXXX (#190)

[Unreleased]: https://github.com/your-org/your-project/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/your-org/your-project/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/your-org/your-project/releases/tag/v1.1.0
```

### 变更类型说明

| 类型 | 含义 | 使用场景 |
|------|------|---------|
| Added | 新增 | 新功能、新接口 |
| Changed | 变更 | 已有功能的修改 |
| Deprecated | 弃用 | 即将移除的功能 |
| Removed | 移除 | 已删除的功能 |
| Fixed | 修复 | Bug 修复 |
| Security | 安全 | 安全漏洞修复 |

### CHANGELOG 编写原则

- **面向用户**：记录用户能感知的变更，不记录内部重构
- **关联 Issue/PR**：每条记录附带编号，方便追溯
- **保持 Unreleased**：开发中的变更先放在 Unreleased 区域
- **发版时整理**：发版时将 Unreleased 内容移到新版本号下

---

## GitHub 模板

### PR 模板

文件路径：`.github/pull_request_template.md`

```markdown
## 概要

<!-- 用 1-3 句话描述这个 PR 做了什么 -->

## 变更类型

- [ ] 新功能（feat）
- [ ] Bug 修复（fix）
- [ ] 重构（refactor）
- [ ] 文档（docs）
- [ ] 测试（test）
- [ ] 构建/CI（chore/ci）

## 变更内容

<!-- 列出主要变更点 -->

-
-

## 测试计划

<!-- 如何验证这些变更 -->

- [ ] 单元测试通过
- [ ] 手动测试通过
- [ ] 关键场景覆盖

## 关联 Issue

<!-- Closes #123 -->

## 截图（如适用）

<!-- UI 变更请附截图 -->
```

### Bug 报告模板

文件路径：`.github/ISSUE_TEMPLATE/bug_report.md`

```markdown
---
name: Bug 报告
about: 提交 bug 帮助我们改进
title: "[Bug] "
labels: bug
assignees: ""
---

## 问题描述

<!-- 简明扼要描述这个 bug -->

## 复现步骤

1. 进入 '...'
2. 点击 '...'
3. 滚动到 '...'
4. 看到错误

## 期望行为

<!-- 描述你期望发生的事情 -->

## 实际行为

<!-- 描述实际发生的事情 -->

## 环境信息

- 操作系统：[例如 macOS 15.3]
- 浏览器：[例如 Chrome 130]
- 项目版本：[例如 1.2.0]

## 截图

<!-- 如果适用，添加截图说明问题 -->

## 补充信息

<!-- 其他有助于定位问题的信息 -->
```

### 功能请求模板

文件路径：`.github/ISSUE_TEMPLATE/feature_request.md`

```markdown
---
name: 功能请求
about: 提出新功能或改进建议
title: "[Feature] "
labels: enhancement
assignees: ""
---

## 问题描述

<!-- 描述你遇到的问题或痛点 -->

## 期望方案

<!-- 描述你期望的解决方案 -->

## 备选方案

<!-- 描述你考虑过的其他方案 -->

## 补充信息

<!-- 其他上下文、截图、参考链接等 -->
```
