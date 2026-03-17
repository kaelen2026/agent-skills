---
name: ci-cd
version: 1.0.0
description: CI/CD 流水线设计、部署策略与环境管理指南
last_updated: 2026-03-17
---

# CI/CD 技能

## When to Activate

以下场景触发本技能：

- 设置或修改 CI/CD 流水线（GitHub Actions、GitLab CI 等）
- 配置部署流程（staging、production）
- 管理环境变量与密钥
- 设计部署策略（蓝绿、金丝雀、滚动）
- 配置预览部署或特性分支部署
- 优化构建速度与缓存策略
- 设置 monorepo CI 流水线

## Workflow

### 1. 分析项目类型

确认项目的技术栈与部署目标：

- **运行时**: Node.js / Python / Go / Rust 等
- **框架**: Next.js / Remix / FastAPI / 等
- **部署平台**: Vercel / Railway / AWS / GCP 等
- **仓库结构**: 单仓库 / monorepo
- **包管理器**: npm / pnpm / yarn / bun

### 2. 设计流水线

根据项目类型设计 CI/CD 流水线：

1. **PR 检查流水线** — 每次 PR 触发 lint、类型检查、测试、构建
2. **Staging 部署流水线** — 合并到 main 后自动部署到 staging
3. **Production 部署流水线** — 打 tag 或创建 release 后部署到生产环境

参考 → [流水线设计](references/pipeline-design.md)

### 3. 配置环境

为每个环境层级配置正确的变量与密钥：

- local → dev → staging → production
- 确保环境一致性（staging 尽量接近 production）
- 密钥通过平台级别的密钥管理服务注入，绝不硬编码

参考 → [环境管理](references/environment-management.md)

### 4. 选择部署策略

根据风险容忍度与复杂度选择合适的部署策略：

| 策略 | 风险 | 回滚速度 | 复杂度 | 停机时间 |
| --- | --- | --- | --- | --- |
| 蓝绿部署 | 低 | 即时 | 中 | 零 |
| 金丝雀部署 | 最低 | 快 | 高 | 零 |
| 滚动部署 | 中 | 中 | 低 | 接近零 |
| 功能标志 | 最低 | 即时 | 中 | 零 |

参考 → [部署策略](references/deployment-strategies.md)

## 审查清单

流水线配置完成后，逐项确认：

### 流水线质量

- [ ] PR 检查覆盖 lint、类型检查、测试、构建
- [ ] 测试在隔离环境中运行，不依赖外部服务
- [ ] 构建产物被正确缓存，避免重复构建
- [ ] 并行执行无依赖的任务以缩短流水线时间
- [ ] 失败时有清晰的错误信息和日志

### 部署安全

- [ ] 生产部署需要手动审批或 tag 触发
- [ ] 密钥通过平台密钥管理注入，未硬编码在配置中
- [ ] 环境变量按环境层级正确分离
- [ ] 敏感分支有保护规则（branch protection）

### 回滚与监控

- [ ] 定义了明确的回滚流程
- [ ] 部署后有健康检查验证服务可用性
- [ ] 关键指标有监控告警（错误率、响应时间、可用性）
- [ ] 部署历史可追溯

### 环境一致性

- [ ] staging 环境配置尽量接近 production
- [ ] 数据库 schema 迁移在部署流水线中自动执行
- [ ] 预览部署可用于特性分支的测试验证

## Output Format

```markdown
# CI/CD: {流水线名称}

## 流水线构成
{阶段和任务的构成}

## 执行结果
{构建/测试/部署的结果}

## 设计审查
{审查结果清单，标记通过/未通过}
```
