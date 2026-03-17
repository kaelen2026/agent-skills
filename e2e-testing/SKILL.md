---
name: e2e-testing
version: 1.0.0
description: "Playwright E2E 测试技能。覆盖 Page Object Model、测试模式、CI/CD 集成、不稳定测试策略。"
last_updated: 2026-03-17
---

# E2E Testing

## When to Activate

- 用户请求编写或运行 E2E 测试
- 用户请求 Playwright 配置或模式
- 用户请求测试关键用户流程（登录、支付、表单提交）
- 用户报告不稳定测试（flaky tests）需要修复
- 用户请求 CI/CD 中集成 E2E 测试

## Workflow

### 1. 收集需求

| 项目 | 说明 | 默认值 |
|------|------|--------|
| 测试框架 | Playwright / Cypress | Playwright |
| 测试范围 | 关键用户流程列表 | **必须** |
| 浏览器 | Chromium / Firefox / WebKit / Mobile | Chromium |
| 环境 | 本地 / Staging / Production | 本地 |
| CI/CD | GitHub Actions / GitLab CI / None | GitHub Actions |
| 认证 | 需要登录的流程 | 自动检测 |

### 2. 设计测试（结论先行）

先输出测试计划总览：

```
## 测试计划

| 流程 | 测试文件 | 场景数 | 优先级 |
|------|---------|--------|--------|
| 用户登录 | auth/login.spec.ts | 4 | P0 |
| 搜索功能 | features/search.spec.ts | 3 | P1 |
| 创建资源 | features/create.spec.ts | 5 | P1 |
```

### 3. 生成输出物

按以下顺序生成：

#### A. Page Object Model

- 遵循 [references/page-object-model.md](references/page-object-model.md)
- 每个页面/组件一个 POM 类
- 使用 `data-testid` 定位元素

#### B. 测试用例

- 遵循 [references/test-patterns.md](references/test-patterns.md)
- 使用 AAA 模式（Arrange-Act-Assert）
- 覆盖正常路径和异常路径

#### C. 配置与 CI/CD

- 遵循 [references/config-and-ci.md](references/config-and-ci.md)
- Playwright 配置、GitHub Actions workflow
- 产物（截图、视频、trace）管理

### 4. 设计审查

- [ ] 使用 Page Object Model 封装页面交互
- [ ] 元素定位使用 `data-testid`（不用 CSS class / XPath）
- [ ] 无 `waitForTimeout` 硬等待（使用条件等待）
- [ ] 无 `page.click()` 直接调用（使用 `locator.click()`）
- [ ] 测试之间无依赖（可独立运行）
- [ ] 认证流程使用 fixture 复用
- [ ] 失败时自动截图 + trace
- [ ] CI 中有 retry 机制
- [ ] 不稳定测试已标记 `test.fixme()` 并关联 issue
