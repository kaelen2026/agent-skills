---
name: branching-strategy
version: 1.0.0
description: 分支策略对比、命名规范、PR 模板与分支保护规则
last_updated: 2026-03-17
---

# 分支策略

## 三种主流策略对比

| 维度 | Git Flow | GitHub Flow | Trunk-based |
|------|----------|-------------|-------------|
| **复杂度** | 高 | 低 | 中 |
| **主要分支** | `main` + `develop` + `release` + `hotfix` | `main` + 功能分支 | `main`（短命分支） |
| **发布方式** | 定期发布 | 持续部署 | 持续部署 |
| **适合团队** | 大型团队、版本化产品 | 中小团队、SaaS/Web 应用 | 高级团队、微服务 |
| **合并频率** | 低（按版本周期） | 高（功能完成即合并） | 极高（每日多次） |
| **学习成本** | 高 | 低 | 中（需要功能开关等配套） |
| **回滚难度** | 中 | 低（每次合并即一次部署） | 低 |
| **优点** | 版本管理清晰 | 简洁直观 | 集成冲突最少 |
| **缺点** | 分支过多、合并冲突频繁 | 不适合多版本并行维护 | 需要功能开关和强 CI |

## 推荐策略：GitHub Flow

**适用于绝大多数项目。** 只维护 `main` 分支和短命的功能分支，简洁高效。

### 核心流程

```
main ─────────────────────────────────────────────►
  │                                        ▲
  └── feat/user-auth ──── PR ──── Review ──┘
```

1. 从 `main` 创建功能分支
2. 在功能分支上进行开发和提交
3. 创建 Pull Request 并请求审查
4. CI 通过 + 审查通过后合并到 `main`
5. 合并后自动部署（如已配置）
6. 删除已合并的功能分支

## 分支命名规范

### 前缀规则

| 前缀 | 用途 | 示例 |
|------|------|------|
| `feat/` | 新功能 | `feat/user-authentication` |
| `fix/` | Bug 修复 | `fix/login-redirect-loop` |
| `refactor/` | 代码重构 | `refactor/extract-auth-service` |
| `docs/` | 文档更新 | `docs/api-endpoints` |
| `test/` | 测试相关 | `test/auth-integration` |
| `chore/` | 杂项维护 | `chore/upgrade-dependencies` |

### 命名规则

- 全部小写，使用连字符 `-` 分隔单词
- 简明扼要描述目的，不超过 4-5 个单词
- 可选：加入 Issue 编号，如 `feat/123-user-auth`
- 禁止使用个人名称作为分支名

```bash
# 正确
git checkout -b feat/user-authentication
git checkout -b fix/422-login-redirect

# 错误
git checkout -b my-branch
git checkout -b WIP
git checkout -b Feature_UserAuth
```

## PR 模板

在项目根目录创建 `.github/pull_request_template.md`：

```markdown
## 摘要

<!-- 用 1-3 句话描述这个 PR 做了什么，以及为什么 -->

## 变更类型

- [ ] 新功能 (feat)
- [ ] Bug 修复 (fix)
- [ ] 代码重构 (refactor)
- [ ] 文档更新 (docs)
- [ ] 测试 (test)
- [ ] 杂项维护 (chore)

## 变更内容

<!-- 用列表形式描述具体改动 -->

-
-
-

## 测试计划

<!-- 描述如何验证这些变更 -->

- [ ]
- [ ]

## 截图（如有 UI 变更）

<!-- 贴上变更前后的截图 -->

## 关联 Issue

<!-- 例如：Closes #123, Fixes #456 -->

## 审查重点

<!-- 告诉审查者哪些部分需要重点关注 -->
```

## PR 合并规则

### 合并策略选择

| 场景 | 合并方式 | 原因 |
|------|----------|------|
| 功能分支 → `main` | **Squash Merge** | 将多次提交压缩为一次，保持主分支历史清晰 |
| 发布分支 → `main` | **Merge Commit** | 保留完整的发布历史和合并记录 |
| 热修复 → `main` | **Squash Merge** | 单一修复提交，便于追踪 |

### Squash Merge 的提交信息格式

```
feat: 用户认证功能 (#42)

- 添加登录/注册 API
- 集成 JWT token 认证
- 添加密码加密存储
```

## 分支保护规则

### 必须配置的保护规则（`main` 分支）

```yaml
# GitHub 分支保护设置
branch_protection:
  branch: main
  rules:
    # 必须通过 PR 才能合并
    require_pull_request: true

    # 至少 1 人审查通过
    required_approving_review_count: 1

    # 推送新提交后，之前的审查自动失效
    dismiss_stale_reviews: true

    # CI 必须通过
    require_status_checks: true
    required_status_checks:
      - "build"
      - "test"
      - "lint"

    # 禁止强制推送
    allow_force_pushes: false

    # 禁止删除分支
    allow_deletions: false

    # 要求分支与 main 保持最新
    require_branches_up_to_date: true
```

### 使用 GitHub CLI 配置

```bash
# 启用分支保护
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["build","test","lint"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

## 过期分支清理

### 手动清理

```bash
# 查看已合并到 main 的远程分支
git branch -r --merged origin/main | grep -v 'main'

# 删除已合并的本地分支
git branch --merged main | grep -v 'main' | xargs -r git branch -d

# 删除已合并的远程分支
git branch -r --merged origin/main | grep -v 'main' | \
  sed 's/origin\///' | xargs -r -I {} git push origin --delete {}

# 清理本地的远程跟踪引用
git fetch --prune
```

### 自动清理（GitHub Actions）

```yaml
name: 清理过期分支

on:
  schedule:
    # 每周一凌晨 2 点执行
    - cron: "0 2 * * 1"

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 删除已合并的过期分支
        uses: actions/github-script@v7
        with:
          script: |
            const { data: branches } = await github.rest.repos.listBranches({
              owner: context.repo.owner,
              repo: context.repo.repo,
              per_page: 100,
            });

            for (const branch of branches) {
              if (branch.name === 'main') continue;

              try {
                const { data: comparison } = await github.rest.repos.compareCommits({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  base: branch.name,
                  head: 'main',
                });

                if (comparison.status === 'ahead' || comparison.status === 'identical') {
                  console.log(`删除已合并分支: ${branch.name}`);
                  await github.rest.git.deleteRef({
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    ref: `heads/${branch.name}`,
                  });
                }
              } catch (error) {
                console.log(`跳过分支 ${branch.name}: ${error.message}`);
              }
            }
```

### 建议

- **合并 PR 时启用自动删除分支**：在 GitHub 仓库设置中开启 "Automatically delete head branches"
- **定期审查长期未活跃的分支**：超过 30 天未更新的分支应评估是否仍然需要
- **使用 `git fetch --prune`**：每次拉取时清理已删除的远程分支引用
