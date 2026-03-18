---
name: setup-pre-commit
version: 1.0.0
description: "Pre-commit hook 设置技能。Husky + lint-staged + Biome（format & lint）、型チェック、テスト を自動実行。"
last_updated: 2026-03-18
metadata:
  filePattern:
    - ".husky/**/*"
    - ".lintstagedrc"
    - "biome.json"
    - "biome.jsonc"
  bashPattern:
    - "husky"
    - "lint-staged"
    - "biome"
  priority: 5
---

# Setup Pre-Commit Hooks

就像出门前的检查清单——钥匙、钱包、手机。Pre-commit hook 确保每次提交前代码都经过格式化、lint、类型检查和测试。

## When to Activate

- 用户想添加 pre-commit hooks
- 用户说"设置 husky"、"pre-commit"、"提交前检查"
- 新项目需要代码质量自动化

## Workflow

### 1. 检测包管理器

检查锁文件：`package-lock.json`（npm）、`pnpm-lock.yaml`（pnpm）、`yarn.lock`（yarn）、`bun.lockb`（bun）。默认 npm。

### 2. 安装依赖

作为 devDependencies 安装：

```
husky lint-staged @biomejs/biome
```

### 3. 初始化 Husky

```bash
npx husky init
```

创建 `.husky/` 目录，并在 package.json 添加 `prepare: "husky"`。

### 4. 创建 `.husky/pre-commit`

Husky v9+ 不需要 shebang：

```
npx lint-staged
npm run typecheck
npm run test
```

**适配**：将 `npm` 替换为检测到的包管理器。如果 package.json 没有 `typecheck` 或 `test` 脚本，省略对应行并告知用户。

### 5. 创建 `.lintstagedrc`

```json
{
  "*": "biome check --write --no-errors-on-unmatched --files-ignore-unknown=true"
}
```

`--files-ignore-unknown=true` 跳过 Biome 不支持的文件（图片等）。

### 6. 创建 `biome.json`（如不存在）

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

### 7. 验证

- [ ] `.husky/pre-commit` 存在且可执行
- [ ] `.lintstagedrc` 存在
- [ ] package.json 的 `prepare` 脚本为 `"husky"`
- [ ] `biome.json` 存在
- [ ] 运行 `npx lint-staged` 确认正常工作

### 8. 提交

暂存所有变更文件，提交信息：`chore: add pre-commit hooks (husky + lint-staged + biome)`

此提交本身会触发新的 pre-commit hook——作为冒烟测试。

## Design Review

- [ ] 包管理器检测正确
- [ ] Biome 同时处理 format 和 lint（不需要单独的 eslint）
- [ ] lint-staged 只处理暂存文件（快速）
- [ ] typecheck 和 test 在 lint-staged 之后运行（全量）
- [ ] 现有 Biome 配置未被覆盖
