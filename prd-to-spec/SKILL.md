---
name: prd-to-spec
description: "PRD 拆解技能。将产品需求文档拆解为垂直切片的实施规格，输出为本地 Markdown。"
metadata:
  filePattern:
    - "**/docs/specs/**/*"
    - "**/docs/plans/**/*"
  priority: 8
---

# PRD to Spec

就像把建筑蓝图拆成施工工序一样——每道工序切穿所有楼层（地基→结构→水电→装修），而不是先做完所有地基再做所有结构。

## When to Activate

- 用户有现成 PRD（对话中或 GitHub Issue）需要拆解
- 用户说"拆 spec"、"实施规格"、"拆任务"
- 需要将需求文档转化为可执行的开发任务

## Workflow

### 1. 确认 PRD

确认 PRD 在上下文中：

- 对话中直接提供的文本
- GitHub Issue URL（用 `gh issue view` 读取）
- 本地文件路径

如果 PRD 不完整或模糊，建议用户先运行 `write-a-prd` 技能。

### 2. 探索代码库

用 Agent (Explore) 理解现有架构：

- 路由结构和页面组织
- 数据模型和 Schema
- 认证/授权模式
- 第三方集成边界
- 测试基础设施

目的：找到实施的着力点和约束。

### 3. 识别持久架构决策

从 PRD 和代码库中提取不会随实现变化的决策：

- **路由**: 新增/修改的路由路径
- **Schema**: 数据模型变更（表、字段、关系）
- **数据流**: 模块间的数据流向
- **认证**: 权限模型变更
- **第三方边界**: 外部 API/服务集成点

这些决策写入 spec，具体文件名/函数名不写。

### 4. 拆分垂直切片

将功能拆为垂直切片（tracer bullets），每个切片切穿所有集成层：

**切片结构**: Schema → API → UI → 测试

**分类**：
- **HITL** (Human-in-the-Loop): 需要人工决策或确认的切片
- **AFK** (Away-from-Keyboard): 可自主完成的切片

**原则**：
- 每个切片可独立演示或验证
- 切片间依赖关系明确（哪个必须先完成）
- 第一个切片是最小可验证路径（tracer bullet）
- 不含具体文件名/函数名，只含持久决策

### 5. 用户确认

向用户展示切片列表：

- 切片名称和范围
- HITL vs AFK 分类
- 依赖关系图（哪些可并行）
- 预估粒度是否合适

根据反馈调整切片粒度和顺序。

### 6. 输出

- 输出路径: `./docs/specs/{feature-name}.md`
- 使用 [references/spec-template.md](references/spec-template.md) 模板

## Design Review

输出后自动检查：

- [ ] 每个切片切穿所有集成层（Schema → API → UI → 测试）
- [ ] 每个切片可独立演示或验证
- [ ] HITL vs AFK 分类明确
- [ ] 依赖关系无循环
- [ ] 第一个切片是最小可验证路径
- [ ] 不含具体文件名/函数名
- [ ] 架构决策（路由、Schema、认证）已记录
- [ ] 验收标准描述外部行为
