# Skill 解剖学

> SKILL.md 各章节的详细说明、规则与示例。

## 1. Frontmatter

YAML frontmatter 是技能的元数据，决定何时被注入到对话中。

### 字段说明

| 字段 | 必须 | 类型 | 说明 |
|------|:---:|------|------|
| `name` | Yes | string | kebab-case，与目录名一致 |
| `description` | Yes | string | 一句话描述，中文，用引号包裹 |
| `metadata.filePattern` | No | string[] | glob 模式，匹配文件路径时触发 |
| `metadata.bashPattern` | No | string[] | 正则模式，匹配 bash 命令时触发 |
| `metadata.priority` | Yes | number | 2-9，越大越优先注入 |

### filePattern 语法

```yaml
filePattern:
  - "**/*.ts"           # 所有 .ts 文件
  - "**/components/**/*" # components 目录下所有文件
  - "**/errors.ts"       # 特定文件名
  - "**/docs/adr/**/*"   # 特定子目录
```

匹配逻辑：先匹配完整路径，再匹配 basename，再逐级匹配后缀。

### bashPattern 语法

```yaml
bashPattern:
  - "git diff|git log.*--stat"    # 正则，匹配 git 命令
  - "express|hono|fastapi|中间件"  # 多关键词 OR
  - "throw|catch|Error|错误"       # 错误相关命令
```

### Priority 层级参考

```
9: prd, planning          — 流程入口，最先注入
8: auth, grill-me, security-review — 安全/质量关键
7: api-design, database   — 核心设计活动
6: code-review, tdd-workflow, e2e-testing, error-handling — 开发质量
5: backend-patterns, frontend-patterns, git-workflow, ci-cd — 实现模式
4: coding-standards, observability, performance — 横切关注点
3: documentation          — 辅助
2: tutorial               — 新手引导
```

注意：最多同时注入 3 个 skill（MAX_SKILLS=3），18KB 字节预算。高优先级 skill 会挤掉低优先级。

## 2. 类比开头

每个 SKILL.md 的第一段必须用日常比喻解释技能用途。

### 好的类比

| 技能 | 类比 |
|------|------|
| prd | "像记者采访一样——先深挖问题全貌，再写报道，最后拆成施工工序" |
| planning | "就像建筑师先画蓝图再动工一样，写代码前先创建 spec 和 plan" |
| grill-me | "就像律师交叉询问一样——不是为了否定你，而是为了找出论证的漏洞" |
| tutorial | "就像游戏的新手教程——不是看说明书，而是在实战中学习" |
| code-review | "就像编辑审稿一样——不是重写文章，而是找出逻辑漏洞和表达不清的地方" |

### 类比规则

- 1-2 句话，不超过 3 行
- 用"就像...一样"或"像...一样"句式
- 比喻对象必须是非技术人员也能理解的日常事物
- 比喻后不需要再解释比喻本身

## 3. When to Activate

列出触发技能的所有场景。

### 格式

```markdown
## When to Activate

以下场景自动激活本技能：

- 用户请求 {动作}（{具体描述}）
- 用户说"{触发短语}"
- 编辑 {文件类型} 文件时
- {其他触发条件}
```

### 规则

- 至少 3 个触发场景
- 包含用户可能使用的中文和英文关键词
- 包含文件操作触发（与 filePattern 对应）
- 包含命令触发（与 bashPattern 对应）

## 4. Workflow

分阶段的工作流是技能的核心。

### 结构模板

```markdown
### Phase 1: {阶段名称}

1.1 **{步骤名称}**

- {具体操作}
- {输入}: {什么}
- {输出}: {什么}

1.2 **{步骤名称}**

...

### Phase 2: {阶段名称}

...
```

### 规则

- 3-7 个 Phase，每个 Phase 2-5 个子步骤
- 每步有明确的动作动词开头（检查、生成、验证、创建）
- Phase 之间有明确的交付物（Phase 1 输出 = Phase 2 输入）
- 决策点用表格或条件列表表示
- 可并行的步骤明确标注

### 常见 Workflow 模式

| 模式 | 适用场景 | 阶段 |
|------|----------|------|
| 收集→设计→生成→审查 | 大多数技能 | 需求收集 → 方案设计 → 产出生成 → 质量审查 |
| 分析→分类→处理→报告 | 审查类技能 | 输入分析 → 问题分类 → 逐项处理 → 汇总报告 |
| 提问→深挖→总结→输出 | 对话类技能 | 初始提问 → 追问深挖 → 发现总结 → 结构化输出 |

## 5. 审查清单

每个技能必须有质量门禁。

### 格式

```markdown
## 审查清单

- [ ] {检查项 1}
- [ ] {检查项 2}
- [ ] {检查项 3}
```

### 必须覆盖的维度

| 维度 | 说明 | 示例 |
|------|------|------|
| 完整性 | 所有必要部分都存在 | "所有 API 端点都有错误响应定义" |
| 正确性 | 内容技术上正确 | "SQL 迁移可回滚" |
| 一致性 | 与项目约定一致 | "响应格式遵循统一信封" |
| 安全性 | 无安全隐患 | "无硬编码密钥" |

### 规则

- 至少 8 个检查项
- 用 `- [ ]` checkbox 格式
- 可分组（结构审查、内容审查、安全审查等）
- 检查项必须可客观验证（不是"代码质量好"，而是"函数不超过 50 行"）

## 6. References 链接

SKILL.md 末尾链接到 references/ 下的文件。

### 格式

```markdown
## References

- [描述](references/filename.md) — 一句话说明
- [描述](references/filename.md) — 一句话说明
```

### 交叉引用其他技能

```markdown
> 统一响应信封格式参见 [api-design](../api-design/SKILL.md)
```

规则：只引用 owner 技能，不引用也定义了相同内容的技能。
