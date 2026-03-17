---
name: commit-conventions
version: 1.0.0
description: Conventional Commits 提交规范、commitlint 配置、husky + lint-staged 设置
last_updated: 2026-03-17
---

# 提交规范

## Conventional Commits 格式

```
<type>(<scope>): <description>

<optional body>

<optional footer>
```

### 各字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `type` | 是 | 提交类型，标识变更的性质 |
| `scope` | 否 | 影响范围，通常是模块或功能名 |
| `description` | 是 | 简短描述，不超过 72 个字符 |
| `body` | 否 | 详细说明变更的动机和方式 |
| `footer` | 否 | 破坏性变更声明或关联 Issue |

## 提交类型

| 类型 | 用途 | 版本影响 | 示例 |
|------|------|----------|------|
| `feat` | 新功能 | MINOR | `feat(auth): 添加 OAuth2 登录` |
| `fix` | Bug 修复 | PATCH | `fix(api): 修复分页参数越界` |
| `refactor` | 代码重构（不改功能） | 无 | `refactor(utils): 提取日期格式化函数` |
| `docs` | 文档更新 | 无 | `docs(readme): 更新安装说明` |
| `test` | 测试相关 | 无 | `test(auth): 添加登录集成测试` |
| `chore` | 杂项维护 | 无 | `chore(deps): 升级 TypeScript 到 5.4` |
| `perf` | 性能优化 | PATCH | `perf(query): 添加数据库查询索引` |
| `ci` | CI/CD 配置 | 无 | `ci: 添加 GitHub Actions 构建流水线` |
| `style` | 代码格式（不改逻辑） | 无 | `style: 统一缩进为 2 空格` |
| `build` | 构建系统变更 | 无 | `build: 迁移到 Vite 构建工具` |

## 破坏性变更（Breaking Changes）

有两种声明方式：

### 方式一：类型后加 `!`

```
feat!: 移除 v1 API 端点

BREAKING CHANGE: /api/v1/* 端点已全部移除，请迁移到 /api/v2/*
```

### 方式二：footer 中声明

```
feat(api): 重构用户认证接口

将认证方式从 session 迁移到 JWT。

BREAKING CHANGE: login API 的响应格式已变更，
返回 { token, refreshToken } 而非 { sessionId }。
迁移指南见 docs/migration-v3.md。
```

## 提交信息示例

### 正确示例

```bash
# 新功能
git commit -m "feat(auth): 添加 Google OAuth2 登录支持"

# Bug 修复（关联 Issue）
git commit -m "fix(cart): 修复商品数量为 0 时仍可下单的问题

Closes #234"

# 带详细说明的重构
git commit -m "refactor(database): 将原生 SQL 迁移到 QueryBuilder

使用 QueryBuilder 替代手写 SQL，提高可维护性和类型安全。
涉及 user、order、product 三个模块的数据访问层。"

# 破坏性变更
git commit -m "feat!(api): 将认证从 Cookie 迁移到 Bearer Token

BREAKING CHANGE: 所有 API 请求需在 Authorization header 中
携带 Bearer Token，不再支持 Cookie 认证。"
```

### 错误示例

```bash
# 错误：描述模糊
git commit -m "fix: 修复 bug"
git commit -m "update: 更新代码"

# 错误：缺少类型
git commit -m "添加用户认证功能"

# 错误：描述过长
git commit -m "feat: 添加了一个非常复杂的用户认证系统包含OAuth2和JWT和密码重置和邮件验证等功能"

# 错误：混合多个不相关变更
git commit -m "feat: 添加登录功能并修复样式和更新文档"
```

### 编写原则

1. **一次提交只做一件事** — 不要混合功能和修复
2. **描述用祈使句** — "添加 X" 而非 "添加了 X"
3. **不超过 72 字符** — 超出部分放到 body 中
4. **说明「为什么」** — body 部分解释动机，而非罗列「做了什么」

## commitlint 配置

### 安装

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
```

### 配置文件

创建 `commitlint.config.js`：

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    // 类型必须是以下之一
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "refactor",
        "docs",
        "test",
        "chore",
        "perf",
        "ci",
        "style",
        "build",
      ],
    ],
    // 类型必须小写
    "type-case": [2, "always", "lower-case"],
    // 描述不能为空
    "subject-empty": [2, "never"],
    // 描述不以句号结尾
    "subject-full-stop": [2, "never", "."],
    // header 不超过 72 字符
    "header-max-length": [2, "always", 72],
    // body 每行不超过 100 字符
    "body-max-line-length": [2, "always", 100],
  },
};
```

## husky + lint-staged 设置

### 安装

```bash
npm install -D husky lint-staged
```

### 初始化 husky

```bash
# 初始化 husky
npx husky init

# 添加 commit-msg 钩子（校验提交信息）
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg

# 添加 pre-commit 钩子（运行 lint-staged）
echo 'npx lint-staged' > .husky/pre-commit
```

### lint-staged 配置

在 `package.json` 中添加：

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ],
    "*.{css,scss}": [
      "prettier --write"
    ]
  }
}
```

### 完整的 package.json 示例

```json
{
  "scripts": {
    "prepare": "husky"
  },
  "devDependencies": {
    "@commitlint/cli": "^19.0.0",
    "@commitlint/config-conventional": "^19.0.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.0.0"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

## Interactive Rebase 指南

### 何时使用 Squash

| 场景 | 是否 Squash | 原因 |
|------|-------------|------|
| 多次 "WIP" 提交 | 是 | 中间状态对历史无意义 |
| 修复 review 意见的后续提交 | 是 | 应并入原始提交 |
| 逻辑独立的多个功能点 | 否 | 每个功能点值得独立追踪 |
| 合并前的 fixup 提交 | 是 | 用 `fixup` 而非 `squash` |

### 操作方法

```bash
# 将最近 3 次提交进行交互式变基
git rebase -i HEAD~3
```

在编辑器中将需要压缩的提交标记为 `squash` 或 `fixup`：

```
pick abc1234 feat(auth): 添加登录 API
squash def5678 WIP: 调试登录逻辑
squash ghi9012 fix: 修复 review 意见
```

### 注意事项

- **只对未推送的本地提交进行 rebase** — 已推送的提交禁止 rebase
- **推送到远程后需要 rebase 时**，与团队沟通后使用 `git push --force-with-lease`（优于 `--force`）
- **优先使用 `fixup`** — 当只需要合并代码而不需要保留提交信息时
- **保持原子性** — Squash 后的提交仍应只做一件事
