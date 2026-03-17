---
name: documentation
version: 1.0.0
description: 项目文档编写技能 — ADR、README、API 文档、变更日志等标准化模板与工作流
last_updated: 2026-03-17
---

# 文档编写技能

## 激活条件

以下场景自动激活本技能：

- 需要记录架构决策（ADR）
- 编写或更新项目文档（README、CONTRIBUTING）
- 生成或维护变更日志（CHANGELOG）
- 编写 API 文档
- 创建 GitHub issue / PR 模板

## 工作流

### 步骤 1：识别文档类型

根据需求确定要编写的文档类型：

| 文档类型 | 适用场景 | 参考模板 |
|---------|---------|---------|
| ADR | 技术选型、架构变更、重大权衡 | `references/adr-template.md` |
| README | 项目介绍、快速上手 | `references/project-docs.md` |
| CONTRIBUTING | 贡献指南、开发规范 | `references/project-docs.md` |
| CHANGELOG | 版本发布、变更记录 | `references/project-docs.md` |
| API 文档 | 接口说明、错误码、认证方式 | `references/api-docs.md` |
| GitHub 模板 | PR 模板、Issue 模板 | `references/project-docs.md` |

### 步骤 2：选用对应模板

根据文档类型加载对应的参考模板。模板提供标准结构，按实际需求填充内容。

### 步骤 3：编写文档

遵循以下原则：

- **结论先行**：先写核心信息，再展开细节
- **简洁明了**：避免冗余，每句话都有存在价值
- **示例驱动**：用代码示例代替抽象说明
- **保持一致**：同一项目内术语、格式统一

### 步骤 4：审查文档

使用下方审查清单逐项检查。

## 文档类型详解

### ADR（架构决策记录）

记录项目中重要的技术决策，包括背景、决策内容、后果和备选方案。详见 `references/adr-template.md`。

**何时编写 ADR：**

- 引入新技术或框架
- 变更系统架构
- 做出有重大权衡的技术选择
- 废弃或替换现有方案

### README

项目的"门面"。新人应在 5 分钟内理解项目用途并成功运行。详见 `references/project-docs.md`。

### API 文档

面向 API 消费者的完整参考。包含端点说明、请求/响应示例、错误码、认证方式和速率限制。详见 `references/api-docs.md`。

### CHANGELOG（变更日志）

遵循 [Keep a Changelog](https://keepachangelog.com/) 格式，按版本记录所有面向用户的变更。详见 `references/project-docs.md`。

### CONTRIBUTING（贡献指南）

降低贡献者门槛，明确开发流程、代码规范和 PR 要求。详见 `references/project-docs.md`。

## 审查清单

编写完成后，逐项确认：

- [ ] **准确性**：技术细节正确，代码示例可运行
- [ ] **完整性**：覆盖目标读者所需的全部信息
- [ ] **一致性**：术语、格式、命名风格统一
- [ ] **可读性**：结构清晰，段落简短，善用列表和表格
- [ ] **时效性**：版本号、日期、链接均为最新
- [ ] **示例质量**：示例贴近真实场景，可直接复制使用
- [ ] **无冗余**：删除重复内容，避免信息分散在多处
- [ ] **拼写与语法**：无错别字，语句通顺
