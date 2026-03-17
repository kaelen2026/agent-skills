---
name: release-workflow
version: 1.0.0
description: 语义化版本管理、自动 Changelog 生成、GitHub Actions 发布流水线
last_updated: 2026-03-17
---

# 发布流程

## 语义化版本（Semantic Versioning）

版本号格式：`MAJOR.MINOR.PATCH`

### 版本递增规则

| 版本位 | 何时递增 | 示例 | 说明 |
|--------|----------|------|------|
| **MAJOR** | 存在破坏性变更 | `1.2.3` → `2.0.0` | API 不兼容的变更，用户需要修改代码 |
| **MINOR** | 新增向后兼容的功能 | `1.2.3` → `1.3.0` | 新功能，但旧代码无需修改 |
| **PATCH** | 向后兼容的 Bug 修复 | `1.2.3` → `1.2.4` | 修复已有功能的问题 |

### 预发布版本

```
1.0.0-alpha.1    # 内部测试
1.0.0-beta.1     # 公开测试
1.0.0-rc.1       # 发布候选
1.0.0            # 正式发布
```

### 版本判定流程图

```
提交包含 BREAKING CHANGE?
  ├── 是 → MAJOR 版本递增（次版本和补丁版本归零）
  └── 否 → 提交类型是 feat?
              ├── 是 → MINOR 版本递增（补丁版本归零）
              └── 否 → 提交类型是 fix 或 perf?
                          ├── 是 → PATCH 版本递增
                          └── 否 → 版本不变
```

## 自动 Changelog 生成

### 方案一：conventional-changelog

适合简单项目，基于提交历史自动生成。

#### 安装

```bash
npm install -D conventional-changelog-cli
```

#### 使用

```bash
# 根据上次标签以来的提交生成 Changelog
npx conventional-changelog -p angular -i CHANGELOG.md -s

# 首次生成完整 Changelog
npx conventional-changelog -p angular -i CHANGELOG.md -s -r 0
```

#### 添加到 package.json

```json
{
  "scripts": {
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s",
    "version": "npm run changelog && git add CHANGELOG.md"
  }
}
```

### 方案二：changesets（推荐 Monorepo）

适合 Monorepo 项目，支持多包独立版本管理。

#### 安装

```bash
npm install -D @changesets/cli
npx changeset init
```

#### 工作流程

```bash
# 1. 开发完成后，记录变更集
npx changeset

# 2. 根据交互式提示选择：
#    - 影响的包
#    - 版本递增类型（major/minor/patch）
#    - 变更描述

# 3. 发布时，消费所有变更集
npx changeset version

# 4. 发布到 npm（如需要）
npx changeset publish
```

#### 生成的 Changelog 示例

```markdown
# @myapp/core

## 2.1.0

### Minor Changes

- feat: 添加用户偏好设置 API

### Patch Changes

- fix: 修复时区转换错误
- Updated dependencies
  - @myapp/utils@1.3.1
```

## 发布流程

### 标准流程：标签 → Changelog → GitHub Release

```bash
# 1. 确保在 main 分支且代码最新
git checkout main
git pull origin main

# 2. 更新版本号
npm version minor  # 或 major / patch
# 该命令会自动：
#   - 更新 package.json 版本号
#   - 创建 git commit
#   - 创建 git tag

# 3. 生成 Changelog（如未使用 npm version 钩子）
npx conventional-changelog -p angular -i CHANGELOG.md -s
git add CHANGELOG.md
git commit -m "docs: 更新 CHANGELOG"

# 4. 推送代码和标签
git push origin main --follow-tags

# 5. 创建 GitHub Release
gh release create v1.3.0 \
  --title "v1.3.0" \
  --notes-file CHANGELOG.md \
  --latest
```

### 使用 GitHub CLI 快速发布

```bash
# 自动从提交历史生成发布说明
gh release create v1.3.0 --generate-notes

# 带自定义说明
gh release create v1.3.0 \
  --title "v1.3.0 - 用户认证增强" \
  --notes "$(cat <<'EOF'
## 新功能
- 添加 OAuth2 登录支持
- 添加双因素认证

## Bug 修复
- 修复 token 刷新竞态条件

## 破坏性变更
- 移除 v1 认证 API（迁移指南见 docs/migration.md）
EOF
)"
```

## 热修复（Hotfix）流程

当生产环境出现紧急 Bug，需要跳过正常开发流程快速修复。

### 流程

```
main (v1.3.0) ──────────────────────────────────► main (v1.3.1)
  │                                          ▲
  └── hotfix/fix-payment-crash ──── PR ──────┘
       (基于 v1.3.0 标签创建)
```

### 操作步骤

```bash
# 1. 从最新标签创建热修复分支
git checkout -b hotfix/fix-payment-crash v1.3.0

# 2. 修复 Bug
# ... 编码 ...

# 3. 提交修复
git commit -m "fix(payment): 修复支付金额精度丢失导致的崩溃

Closes #789"

# 4. 更新补丁版本
npm version patch  # v1.3.0 → v1.3.1

# 5. 推送并创建 PR
git push -u origin hotfix/fix-payment-crash
gh pr create \
  --title "hotfix: 修复支付崩溃问题" \
  --body "紧急修复：支付金额精度丢失导致部分用户支付失败。" \
  --base main

# 6. 审查通过后合并（使用 Squash Merge）
# 7. 创建补丁版本的 Release
gh release create v1.3.1 \
  --title "v1.3.1 - 紧急修复" \
  --notes "修复支付金额精度丢失导致的崩溃 (#789)"
```

### 热修复注意事项

- **热修复分支从标签创建**，而非从 `main` 的 HEAD 创建
- **只修复目标 Bug**，不要顺手做其他改动
- **必须走 PR 审查**，即使是紧急修复
- **合并后立即打标签发布**

## npm 发布流程

适用于发布 npm 包的项目。

### 手动发布

```bash
# 1. 确认登录状态
npm whoami

# 2. 运行测试和构建
npm test
npm run build

# 3. 检查将要发布的文件
npm pack --dry-run

# 4. 发布
npm publish

# 5. 发布带 scope 的包
npm publish --access public

# 6. 发布预发布版本
npm publish --tag beta
```

### .npmignore 或 package.json files 字段

```json
{
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ]
}
```

## GitHub Actions 自动发布

### 标签推送触发发布

创建 `.github/workflows/release.yml`：

```yaml
name: 自动发布

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: 安装 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: 安装依赖
        run: npm ci

      - name: 运行测试
        run: npm test

      - name: 构建
        run: npm run build

      - name: 生成发布说明
        id: changelog
        run: |
          # 获取上一个标签
          PREV_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
          if [ -z "$PREV_TAG" ]; then
            NOTES=$(git log --pretty=format:"- %s" HEAD)
          else
            NOTES=$(git log --pretty=format:"- %s" ${PREV_TAG}..HEAD)
          fi
          echo "notes<<EOF" >> $GITHUB_OUTPUT
          echo "$NOTES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: 创建 GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body: ${{ steps.changelog.outputs.notes }}
          draft: false
          prerelease: ${{ contains(github.ref, '-') }}

  publish-npm:
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4

      - name: 安装 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://registry.npmjs.org"
          cache: "npm"

      - name: 安装依赖
        run: npm ci

      - name: 构建
        run: npm run build

      - name: 发布到 npm
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 完全自动化流程（PR 合并触发）

创建 `.github/workflows/auto-release.yml`：

```yaml
name: 自动版本与发布

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: 安装 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: 安装依赖
        run: npm ci

      - name: 运行测试
        run: npm test

      - name: 判断版本递增类型
        id: version
        run: |
          # 获取上次标签以来的提交
          LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")

          if [ -z "$LAST_TAG" ]; then
            echo "bump=minor" >> $GITHUB_OUTPUT
            exit 0
          fi

          COMMITS=$(git log ${LAST_TAG}..HEAD --pretty=format:"%s")

          if echo "$COMMITS" | grep -qE "^feat(\(.+\))?!:|BREAKING CHANGE:"; then
            echo "bump=major" >> $GITHUB_OUTPUT
          elif echo "$COMMITS" | grep -qE "^feat(\(.+\))?:"; then
            echo "bump=minor" >> $GITHUB_OUTPUT
          elif echo "$COMMITS" | grep -qE "^(fix|perf)(\(.+\))?:"; then
            echo "bump=patch" >> $GITHUB_OUTPUT
          else
            echo "bump=none" >> $GITHUB_OUTPUT
          fi

      - name: 版本递增并打标签
        if: steps.version.outputs.bump != 'none'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          npm version ${{ steps.version.outputs.bump }} -m "chore(release): v%s"
          git push origin main --follow-tags

      - name: 创建 GitHub Release
        if: steps.version.outputs.bump != 'none'
        run: |
          TAG=$(git describe --tags --abbrev=0)
          gh release create "$TAG" --generate-notes --latest
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 发布清单

每次发布前逐项确认：

- [ ] **所有测试通过** — `npm test` 无失败用例
- [ ] **构建成功** — `npm run build` 无错误
- [ ] **版本号正确** — 遵循语义化版本规则
- [ ] **Changelog 已更新** — 包含本次发布的所有变更
- [ ] **无未提交的变更** — `git status` 工作区干净
- [ ] **分支与远程同步** — `git pull` 后无冲突
- [ ] **破坏性变更已标注** — 在 Changelog 和 Release Notes 中明确声明
- [ ] **迁移指南已提供** — 若有破坏性变更，提供迁移步骤
- [ ] **标签已推送** — `git push --follow-tags`
- [ ] **GitHub Release 已创建** — 包含发布说明
