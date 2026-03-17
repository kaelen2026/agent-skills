# Git Worktree 并行开发

## 核心概念

Git worktree 就像在同一栋大楼里有多个独立办公室。每个办公室（worktree）可以同时处理不同项目（分支），不需要收拾桌面（stash）再换房间。

```
仓库 (.git)
├── 主工作目录 (main)           ← 你现在在这里
├── .git/worktrees/
│   ├── feat-auth/              ← worktree 1: 开发认证功能
│   └── hotfix-payment/         ← worktree 2: 紧急修复支付
```

**每个 worktree 是独立的目录**，共享同一个 `.git` 仓库。切换上下文 = 切换终端窗口，无需 stash、commit WIP 或 checkout。

## 何时使用 Worktree

| 场景                                         | 用 worktree？ | 替代方案                     |
| -------------------------------------------- | ------------- | ---------------------------- |
| 开发中途需要紧急修复 Bug                      | **是**        | `git stash`（易丢失、难管理） |
| 同时开发两个独立功能                          | **是**        | 两个 clone（浪费磁盘）       |
| 在另一个分支跑测试，同时继续开发               | **是**        | CI 等待（耗时）              |
| Code review 时需要本地运行别人的分支           | **是**        | checkout（中断当前工作）      |
| 简单的分支切换（无进行中的工作）               | 否            | `git checkout` 即可          |
| 一次性查看另一个分支的单个文件                 | 否            | `git show branch:file`       |

## 基本操作

### 创建 Worktree

```bash
# 从现有分支创建 worktree
git worktree add ../feat-auth feat/user-auth

# 创建新分支的 worktree（基于当前 HEAD）
git worktree add -b feat/new-feature ../feat-new-feature

# 创建新分支的 worktree（基于指定起点）
git worktree add -b hotfix/payment ../hotfix-payment main

# 推荐的目录结构：放在项目同级目录
# ~/projects/
# ├── myapp/                  ← 主工作目录
# ├── myapp-feat-auth/        ← worktree
# └── myapp-hotfix-payment/   ← worktree
```

### 查看所有 Worktree

```bash
git worktree list
# /Users/dev/myapp                  abc1234 [main]
# /Users/dev/myapp-feat-auth        def5678 [feat/user-auth]
# /Users/dev/myapp-hotfix-payment   ghi9012 [hotfix/payment]
```

### 删除 Worktree

```bash
# 删除 worktree（分支已合并时）
git worktree remove ../feat-auth

# 强制删除（有未提交的变更时）
git worktree remove --force ../feat-auth

# 清理已手动删除目录的 worktree 记录
git worktree prune
```

## 推荐工作流

### 场景一：开发中途紧急修复

```bash
# 1. 当前正在 feat/user-auth 上开发，突然需要修复生产 Bug
#    不需要 stash，直接创建 hotfix worktree
git worktree add -b hotfix/fix-crash ../myapp-hotfix main

# 2. 在新终端窗口中进入 hotfix worktree
cd ../myapp-hotfix

# 3. 安装依赖（worktree 不共享 node_modules）
npm install

# 4. 修复 Bug、测试、提交
git commit -m "fix: 修复订单页面崩溃"

# 5. 推送并创建 PR
git push -u origin hotfix/fix-crash
gh pr create --title "hotfix: 修复订单页面崩溃" --base main

# 6. PR 合并后，清理 worktree
cd ../myapp
git worktree remove ../myapp-hotfix
```

### 场景二：并行开发多个功能

```bash
# 创建两个独立的功能 worktree
git worktree add -b feat/auth ../myapp-auth main
git worktree add -b feat/dashboard ../myapp-dashboard main

# 每个 worktree 在独立终端中开发
# 终端 1: cd ../myapp-auth && npm install && npm run dev
# 终端 2: cd ../myapp-dashboard && npm install && npm run dev

# 完成后逐个合并、清理
git worktree remove ../myapp-auth
git worktree remove ../myapp-dashboard
```

### 场景三：Code Review 本地验证

```bash
# 同事提了 PR，需要本地跑一下验证
git fetch origin
git worktree add ../myapp-review origin/feat/new-api

cd ../myapp-review
npm install
npm test
npm run dev  # 手动验证

# 验证完成后清理
cd ../myapp
git worktree remove ../myapp-review
```

## 目录结构约定

### 方案一：同级目录（推荐）

```
~/projects/
├── myapp/                      # 主工作目录（main）
├── myapp-feat-auth/            # worktree: feat/user-auth
├── myapp-hotfix-payment/       # worktree: hotfix/payment
└── myapp-review-pr-42/         # worktree: code review
```

**优点**: 每个 worktree 与主目录平级，IDE 可以直接打开为独立项目。

### 方案二：子目录（Claude Code 默认）

```
myapp/
├── .claude/worktrees/
│   ├── feat-auth/              # worktree
│   └── hotfix-payment/         # worktree
├── src/
└── package.json
```

**优点**: 所有 worktree 集中管理，不污染上级目录。

### 命名规则

```bash
# 格式: {项目名}-{分支类型}-{描述}
myapp-feat-auth
myapp-fix-crash
myapp-review-pr-42

# 或简写
myapp-auth
myapp-hotfix
myapp-review
```

## 注意事项

### 依赖管理

每个 worktree 有独立的 `node_modules`，需要分别安装：

```bash
cd ../myapp-feat-auth
npm install   # 必须在每个 worktree 中单独安装
```

> **提示**: 使用 pnpm 的项目受益于全局 store，重复安装速度很快。

### 端口冲突

多个 worktree 同时运行 dev server 时注意端口冲突：

```bash
# worktree 1
PORT=3000 npm run dev

# worktree 2
PORT=3001 npm run dev
```

### 不能 checkout 相同分支

**一个分支只能被一个 worktree checkout**。如果尝试在两个 worktree 中 checkout 同一分支，Git 会拒绝：

```bash
# 错误: 'feat/auth' 已在另一个 worktree 中被 checkout
fatal: 'feat/auth' is already checked out at '/Users/dev/myapp-auth'
```

### 锁定 Worktree

长期不用但暂时不想删除的 worktree 可以锁定，防止被 `prune` 清理：

```bash
git worktree lock ../myapp-experiment
git worktree unlock ../myapp-experiment
```

### 共享与隔离

| 共享                          | 隔离                       |
| ----------------------------- | -------------------------- |
| `.git` 仓库（提交历史、配置） | 工作目录文件               |
| 远程引用                      | `node_modules`             |
| Git hooks                     | 构建产物（`dist/`, `.next/`） |
| `.gitignore` 规则             | 本地环境变量（`.env.local`） |

## 清理策略

### 手动清理

```bash
# 查看所有 worktree
git worktree list

# 删除已合并分支的 worktree
git worktree remove ../myapp-feat-auth

# 清理无效的 worktree 记录（目录已手动删除的情况）
git worktree prune
```

### 定期清理脚本

```bash
#!/bin/bash
# cleanup-worktrees.sh — 清理已合并分支的 worktree

git worktree list --porcelain | grep "^worktree " | sed 's/^worktree //' | while read wt; do
  # 跳过主工作目录
  if [ "$wt" = "$(git rev-parse --show-toplevel)" ]; then
    continue
  fi

  branch=$(git -C "$wt" rev-parse --abbrev-ref HEAD 2>/dev/null)
  if [ -z "$branch" ]; then
    continue
  fi

  # 检查分支是否已合并到 main
  if git branch --merged main | grep -q "$branch"; then
    echo "清理已合并的 worktree: $wt ($branch)"
    git worktree remove "$wt"
  fi
done

git worktree prune
```

## Worktree 检查清单

- [ ] **创建前确认目标分支** — 明确要基于哪个分支创建
- [ ] **创建后安装依赖** — `npm install` / `pnpm install`
- [ ] **注意端口分配** — 多个 dev server 使用不同端口
- [ ] **完成后及时清理** — `git worktree remove` 并删除远程分支
- [ ] **定期检查 worktree 列表** — `git worktree list` 防止遗留
- [ ] **不要手动删除目录** — 使用 `git worktree remove`，否则留下残留记录
