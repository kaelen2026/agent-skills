---
name: planning
version: 1.0.0
description: "实现前规划技能。提供功能规格书、实施计划、架构设计的标准化模板。"
last_updated: 2026-03-17
---

# Planning

就像建筑师先画蓝图再动工一样，写代码前先创建 spec 和 plan。没有蓝图的建筑必然返工。

## When to Activate

以下场景自动激活本技能：

- 新功能设计（需要明确范围和需求）
- 实施计划制定（多阶段开发）
- 架构设计 / 技术选型（需要 trade-off 分析）
- HARD-GATE 触发（预计修改 3+ 文件）
- 用户请求编写 spec 或 plan 文档

## Workflow

### 1. 判断文档类型

根据需求确定要生成的文档类型：

| 文档类型 | 适用场景 | 参考模板 |
|---------|---------|---------|
| 功能规格书 (Spec) | 新功能定义、需求明确化 | `references/spec-template.md` |
| 实施计划 (Plan) | 多阶段开发、任务分解 | `references/plan-template.md` |
| 架构设计 | 技术选型、系统设计、trade-off 分析 | `references/architecture-design.md` |
| 全部 | 大型功能（spec → 架构 → plan 顺序生成） | 上述三个模板 |

### 2. 收集需求

向用户确认以下信息（未提供时主动询问）：

| 项目 | 说明 | 默认值 |
|------|------|--------|
| 功能名称 | 功能的简洁标识 | **必须** |
| 背景 | 为什么需要这个功能 | **必须** |
| 目标用户 | 谁会使用这个功能 | 所有用户 |
| 范围 | 包含什么、不包含什么 | 需确认 |
| 约束 | 技术限制、时间限制、依赖 | 无 |
| 非功能需求 | 性能、安全、可访问性目标 | 项目默认标准 |
| 现有系统 | 受影响的现有组件/模块 | 自动检测 |

### 3. 按模板生成文档

根据步骤 1 的判断，使用对应模板生成文档：

#### A. 功能规格书 (Spec)

- 输出路径: `docs/specs/{feature-name}.md`
- 模板: [references/spec-template.md](references/spec-template.md)
- 内容: 概要、用户故事、功能需求、API/DB 变更、非功能需求、验收标准

#### B. 实施计划 (Plan)

- 输出路径: `docs/plans/{feature-name}.md`
- 模板: [references/plan-template.md](references/plan-template.md)
- 内容: 阶段划分、任务列表、文件清单、依赖关系、风险评估

#### C. 架构设计

- 不生成独立文件，嵌入 spec 或 plan 的对应章节
- 参考: [references/architecture-design.md](references/architecture-design.md)
- 内容: 技术选型、trade-off 分析、C4 模型、架构模式选择

### 4. 设计审查

生成完成后，自动检查以下规则：

- [ ] 功能范围明确（包含/不包含已定义）
- [ ] 用户故事覆盖主要角色和场景
- [ ] API 变更包含完整的端点、请求/响应、错误码
- [ ] DB 变更包含表结构、索引、迁移策略
- [ ] 非功能需求有可量化的目标值
- [ ] 验收标准可测试（每条对应至少一个测试用例）
- [ ] 实施阶段之间的依赖关系清晰
- [ ] 风险评估覆盖技术风险和业务风险
- [ ] 文件清单完整（新建/修改文件已列出）
- [ ] 与现有系统的集成点已识别

## 与 /sync 命令兼容

生成的文档与 `/sync` 命令的文档同步功能兼容：

- `docs/specs/{feature-name}.md` 对应 `spec.md` 同步
- `docs/plans/{feature-name}.md` 对应 `prompt_plan.md` 同步

## Output Format

```
# Planning: {功能名称}

## 文档类型
{spec / plan / 架构设计 / 全部}

## 需求确认
{收集到的需求总览}

## 生成文档
{文档路径 + 文档内容}

## 设计审查
{审查结果清单，标记通过/未通过}
```
