---
name: write-a-skill
description: "Agent Skill 编写技能。从需求到完成的标准化工作流：结构设计、frontmatter 配置、references 编写、审查清单。"
metadata:
  filePattern:
    - "**/SKILL.md"
    - "**/skills/**/*"
  bashPattern:
    - "skill|SKILL|技能"
  priority: 7
---

# Write-a-Skill

就像写菜谱一样——先确定这道菜解决什么问题（饿了？宴客？减脂？），再列食材清单（references），最后写步骤（workflow）。好菜谱让任何厨师都能复现，好 skill 让任何 agent 都能执行。

## When to Activate

以下场景自动激活本技能：

- 用户要创建新的 agent skill
- 用户要修改现有 skill 的结构
- 用户说"写一个技能"、"新建 skill"、"添加技能"
- 编辑 `SKILL.md` 文件时

## Workflow

### Phase 1: 需求定义

1.1 **明确技能定位**

- 这个技能解决什么问题？（一句话）
- 目标用户是谁？（初级/中级/高级开发者）
- 触发场景是什么？（文件操作/bash 命令/用户指令）

1.2 **检查重叠**

- 查看 [Ownership Matrix](../README.md) 确认无重复
- 如果与现有技能有交叉，确定 owner vs reference 关系
- 原则：每个关注点只有一个 owner，其他技能引用

1.3 **确定优先级**

| 优先级 | 适用场景 | 示例 |
|--------|----------|------|
| 8-9 | 流程入口、架构决策 | prd, planning, auth |
| 6-7 | 核心开发活动 | api-design, database, code-review |
| 4-5 | 实现模式、运维 | backend-patterns, ci-cd, git-workflow |
| 2-3 | 辅助、文档 | documentation, tutorial |

### Phase 2: 结构设计

2.1 **编写 Frontmatter**

```yaml
---
name: {kebab-case 名称}
description: "{一句话描述，中文}"
metadata:
  filePattern:
    - "{glob 模式，匹配触发文件}"
  bashPattern:
    - "{正则模式，匹配触发命令}"
  priority: {2-9}
---
```

规则：
- `name`: kebab-case，与目录名一致
- `description`: 用引号包裹，中文，说明技能做什么
- `filePattern`: glob 语法（`*`, `**`, `?`），匹配文件路径触发
- `bashPattern`: 正则语法，匹配 bash 命令触发
- `priority`: 数字越大越优先注入（最多同时注入 3 个 skill）
- 纯对话型技能（如 grill-me、tutorial）可省略 filePattern/bashPattern

2.2 **设计 SKILL.md 主体**

必须包含的章节（按顺序）：

1. **标题 + 类比开头** — 用日常比喻解释技能用途（1-2 句）
2. **When to Activate** — 触发条件列表
3. **Workflow** — 分阶段工作流（Phase 1/2/3...）
4. **审查清单** — 质量门禁 checklist
5. **References 链接** — 指向 references/ 下的文件

2.3 **设计 References**

- 每个技能 2-4 个 reference 文件
- 文件命名：`{主题}-{类型}.md`（如 `schema-design.md`、`review-checklist.md`）
- 每个文件 100-300 行，聚焦单一主题
- 包含可直接使用的模板、代码示例、检查清单

### Phase 3: 编写内容

3.1 **SKILL.md 编写规范**

- **类比开头**：第一段用日常比喻，让非专家也能理解
- **结论先行**：每个章节先给结论，再展开细节
- **中文为主**：说明文字用中文，代码/命令用英文
- **Workflow 结构**：
  - 大阶段用 `### Phase N: 名称`
  - 子步骤用 `N.1`、`N.2` 编号
  - 每步有明确的输入和输出
- **审查清单**：用 `- [ ]` checkbox 格式，覆盖完整性、正确性、一致性

3.2 **References 编写规范**

- 模板文件用 `{占位符}` 标记可替换内容
- 代码示例用 TypeScript（除非技能特定于其他语言）
- 表格用于对比和决策矩阵
- 避免重复其他技能已定义的内容，用交叉引用代替

3.3 **交叉引用模式**

```markdown
<!-- 引用其他技能的内容 -->
> 统一响应信封格式参见 [api-design](../api-design/SKILL.md)
> 结构化日志标准参见 [observability](../observability/SKILL.md)
```

### Phase 4: 集成

4.1 **目录结构**

```
{skill-name}/
├── SKILL.md              # 主文件（200-400 行）
└── references/
    ├── {topic-a}.md      # 参考文档 1（100-300 行）
    ├── {topic-b}.md      # 参考文档 2（100-300 行）
    └── {topic-c}.md      # 参考文档 3（100-300 行）
```

4.2 **更新 README.md**

- 在 `## Skills (N)` 中更新数字
- 按优先级顺序添加技能描述段落
- 如有新的 ownership 关系，更新 Ownership Matrix

4.3 **安装验证**

```bash
# 验证 frontmatter 格式
head -15 {skill-name}/SKILL.md

# 验证 references 行数
cat {skill-name}/references/*.md | wc -l

# 验证符号链接（如已安装）
ls -la ~/.claude/skills/{skill-name}
```

## 审查清单

### 结构审查

- [ ] 目录名与 frontmatter `name` 一致（kebab-case）
- [ ] SKILL.md 包含所有必须章节（类比、When to Activate、Workflow、审查清单）
- [ ] References 文件 2-4 个，每个 100-300 行
- [ ] SKILL.md 总行数 200-400 行

### 内容审查

- [ ] 第一段有日常类比
- [ ] 结论先行，无冗长前言
- [ ] Workflow 步骤有明确输入/输出
- [ ] 审查清单覆盖完整性、正确性、一致性
- [ ] 中文说明 + 英文代码，无混杂

### 元数据审查

- [ ] Priority 合理（参考现有技能层级）
- [ ] filePattern glob 语法正确
- [ ] bashPattern 正则语法正确
- [ ] 无与现有技能的 pattern 冲突

### 集成审查

- [ ] README.md 已更新（数字 + 描述）
- [ ] Ownership Matrix 无冲突
- [ ] 交叉引用指向正确的技能
- [ ] 无重复定义（同一关注点只有一个 owner）

## 快速参考

### 现有技能层级

```
Tier 1 — 发现与规划 (priority 8-9)
  prd → planning → grill-me

Tier 2 — 架构与设计 (priority 7-8)
  api-design, database, auth, security-review

Tier 3 — 实现 (priority 5-6)
  backend-patterns, frontend-patterns, error-handling
  code-review, tdd-workflow, e2e-testing

Tier 4 — 运维与横切 (priority 4-5)
  git-workflow, ci-cd, coding-standards, observability, performance

Tier 5 — 辅助 (priority 2-3)
  documentation, tutorial, write-a-skill
```

### 新技能定位决策树

```
需要新技能吗？
├── 现有技能能覆盖吗？ → 不需要新技能，扩展现有 references
├── 与现有技能 >60% 重叠？ → 合并到现有技能
├── 是全新领域？ → 创建新技能
└── 是现有技能的子领域？ → 作为 reference 文件添加到现有技能
```

## References

- [Skill 解剖学](references/skill-anatomy.md) — SKILL.md 各章节详细说明与示例
- [项目最佳实践](references/best-practices.md) — 20 个技能总结出的模式与反模式
