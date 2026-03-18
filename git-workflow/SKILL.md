---
name: git-workflow
description: "Git 工作流管理技能 — 分支策略、提交规范、PR 流程、版本发布、Pre-commit Hook 设置的完整指南"
metadata:
  filePattern:
    - "**/.husky/**/*"
    - "**/commitlint.config.*"
    - "**/.github/pull_request_template.md"
    - "**/.lintstagedrc"
    - "**/biome.json"
    - "**/biome.jsonc"
  bashPattern:
    - "git branch|git merge|git rebase|git worktree|commitlint|husky|lint-staged|biome"
  priority: 5
---

# Git 工作流技能

## When to Activate

当用户请求涉及以下场景时，激活此技能：

- **分支管理** — 创建、命名、保护分支，选择分支策略
- **PR 工作流** — 创建 Pull Request、设置审查规则、合并策略
- **提交规范** — 编写符合 Conventional Commits 的提交信息，配置 commitlint / husky
- **版本发布** — 语义化版本管理、打标签、生成 Changelog
- **变更日志** — 自动或手动维护 CHANGELOG.md
- **并行开发** — 使用 Git Worktree 同时在多个分支上工作

## Workflow

### 1. 确定分支策略

根据团队规模和发布频率选择合适的分支策略。

1. 阅读 [references/branching-strategy.md](references/branching-strategy.md) 了解三种主流策略的对比
2. **推荐大多数项目使用 GitHub Flow**（简洁、适合持续部署）
3. 确认分支命名规范和保护规则

### 2. 配置提交规范

确保每一次提交都清晰、可追溯。

1. 阅读 [references/commit-conventions.md](references/commit-conventions.md)
2. 在项目中配置 `commitlint` + `husky` + `lint-staged`
3. 向团队成员说明提交类型和格式要求

### 3. 设置 PR 规则

保证代码质量和团队协作效率。

1. 参照 [references/branching-strategy.md](references/branching-strategy.md) 中的 PR 模板
2. 配置分支保护规则（必须审查、CI 通过、禁止强制推送）
3. 确定合并策略：功能分支使用 Squash Merge，发布分支使用 Merge Commit

### 4. 配置发布流程

自动化版本管理和变更日志生成。

1. 阅读 [references/release-workflow.md](references/release-workflow.md)
2. 配置语义化版本和自动 Changelog 生成工具
3. 设置 GitHub Actions 自动发布流水线

### 5. 掌握 Worktree 并行开发

当需要同时在多个分支上工作时（紧急修复、并行功能开发、Code Review），使用 Git Worktree。

1. 阅读 [references/worktree-patterns.md](references/worktree-patterns.md)
2. 确认团队的 worktree 目录结构约定（同级目录 vs 子目录）
3. 了解依赖管理、端口分配等注意事项

## 审查清单

在完成 Git 工作流配置后，逐项确认：

- [ ] **分支策略已确定** — 团队成员理解并认同所选策略
- [ ] **分支命名规范已统一** — 使用 `feat/`、`fix/`、`refactor/` 等标准前缀
- [ ] **分支保护规则已启用** — 主分支禁止直接推送，要求审查和 CI 通过
- [ ] **提交规范已配置** — commitlint 和 husky 已安装并正常工作
- [ ] **PR 模板已创建** — `.github/pull_request_template.md` 已就位
- [ ] **合并策略已明确** — 功能分支 Squash、发布分支 Merge Commit
- [ ] **版本号遵循 SemVer** — MAJOR.MINOR.PATCH 规则无歧义
- [ ] **Changelog 自动生成** — 工具已配置，发布时自动更新
- [ ] **发布流水线已就绪** — 标签推送即可触发自动发布
- [ ] **过期分支清理机制** — 已合并分支定期清理，避免仓库膨胀
- [ ] **Worktree 使用规范** — 完成后及时清理，不手动删除目录
- [ ] **Worktree 目录约定** — 命名统一，位置一致（同级或子目录）
- [ ] **Pre-commit hook 已配置** — `.husky/pre-commit` 存在且可执行
- [ ] **lint-staged 已配置** — `.lintstagedrc` 存在，只处理暂存文件
- [ ] **Biome 同时处理 format 和 lint** — 不需要单独的 eslint
- [ ] **现有 Biome 配置未被覆盖** — 仅在不存在时创建

### 6. 设置 Pre-Commit Hooks

就像出门前的检查清单——钥匙、钱包、手机。Pre-commit hook 确保每次提交前代码都经过格式化、lint、类型检查和测试。

#### 6.1 检测包管理器

检查锁文件：`package-lock.json`（npm）、`pnpm-lock.yaml`（pnpm）、`yarn.lock`（yarn）、`bun.lockb`（bun）。默认 npm。

#### 6.2 安装依赖

作为 devDependencies 安装：

```
husky lint-staged @biomejs/biome
```

#### 6.3 初始化 Husky

```bash
npx husky init
```

创建 `.husky/` 目录，并在 package.json 添加 `prepare: "husky"`。

#### 6.4 创建 `.husky/pre-commit`

Husky v9+ 不需要 shebang：

```
npx lint-staged
npm run typecheck
npm run test
```

**适配**：将 `npm` 替换为检测到的包管理器。如果 package.json 没有 `typecheck` 或 `test` 脚本，省略对应行并告知用户。

#### 6.5 创建 `.lintstagedrc`

```json
{
  "*": "biome check --write --no-errors-on-unmatched --files-ignore-unknown=true"
}
```

`--files-ignore-unknown=true` 跳过 Biome 不支持的文件（图片等）。

#### 6.6 创建 `biome.json`（如不存在）

仅在没有 Biome 配置时创建：

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "formatter": {
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 120,
    "quoteStyle": "single"
  },
  "linter": {
    "enabled": true
  },
  "json": {
    "formatter": {
      "enabled": false
    }
  }
}
```

## 参考文档

| 文档 | 内容 |
|------|------|
| [分支策略](references/branching-strategy.md) | 分支模型对比、命名规范、PR 模板、保护规则 |
| [提交规范](references/commit-conventions.md) | Conventional Commits、commitlint、husky 配置 |
| [发布流程](references/release-workflow.md) | 语义化版本、Changelog、GitHub Actions 发布 |
| [Worktree 并行开发](references/worktree-patterns.md) | Worktree 操作、工作流、目录约定、清理策略 |

## Output Format

```markdown
# Git Workflow: {操作类型}

## 变更概要
{变更文件和内容摘要}

## 提交记录
{提交信息列表}

## PR 信息
{PR 标题、描述、审查要点}

## 设计审查
{审查结果清单，标记通过/未通过}
```
