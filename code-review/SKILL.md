---
name: code-review
description: 通用代码审查技能，适用于所有语言和框架的代码质量检查
metadata:
  filePattern:
    - "**/*.ts"
    - "**/*.tsx"
    - "**/*.js"
    - "**/*.jsx"
  bashPattern:
    - "git diff|git log.*--stat"
  priority: 6
---

# 代码审查技能

## When to Activate

以下场景自动激活本技能：

- 用户请求代码审查（code review）
- 用户请求 PR 审查（pull request review）
- 合并前的最终检查
- 用户使用 `/code-review` 命令

## Workflow

### 1. 收集变更

读取所有变更文件，理解变更范围和上下文。

```bash
# 查看变更文件列表
git diff --name-only HEAD~1

# 查看完整差异
git diff HEAD~1

# 查看暂存区差异
git diff --cached
```

如果是 PR 审查：

```bash
# 查看 PR 中所有变更
git diff main...HEAD --name-only
git diff main...HEAD
```

### 2. 应用审查清单

按照 `references/review-checklist.md` 中的清单逐项检查：

1. **正确性** — 逻辑错误、边界情况
2. **安全性** — 注入、认证绕过、密钥泄露
3. **性能** — N+1 查询、不必要的重渲染
4. **可维护性** — 命名、文件大小、函数大小
5. **类型安全** — any 使用、缺失类型
6. **错误处理** — 未处理的 Promise、缺失 try-catch
7. **测试覆盖** — 新逻辑缺少测试
8. **不可变性** — 直接变异操作

### 3. 输出审查结果

使用以下模板输出所有发现：

```markdown
# 代码审查报告

## 概要

- 审查文件数：X
- 变更行数：+Y / -Z
- 发现问题数：CRITICAL(X) / HIGH(X) / MEDIUM(X) / LOW(X)

## CRITICAL（合并前必须修复）

### [C-1] 问题标题
- **文件**: `path/to/file.ts:42`
- **问题**: 具体描述问题及其影响
- **建议修复**:
```修复代码```

## HIGH（应该修复）

### [H-1] 问题标题
- **文件**: `path/to/file.ts:78`
- **问题**: 具体描述
- **建议修复**:
```修复代码```

## MEDIUM（改进建议）

### [M-1] 问题标题
- **文件**: `path/to/file.ts:103`
- **建议**: 改进说明
- **参考**:
```改进代码```

## LOW（风格/偏好）

### [L-1] 问题标题
- **文件**: `path/to/file.ts:150`
- **建议**: 风格改进说明

## 审查结论

- [ ] 通过：0 CRITICAL + 0 HIGH
- [ ] 有条件通过：0 CRITICAL，存在 HIGH 问题需跟进
- [ ] 不通过：存在 CRITICAL 问题
```

### 4. 建议修复

对于每个 CRITICAL 和 HIGH 问题：

1. 解释**为什么**这是问题（用类比说明）
2. 提供**具体的修复代码**
3. 如果有多种修复方案，列出各方案的优劣

## 审查原则

- **结论先行**：先说审查结论（通过/不通过），再展开细节
- **建设性反馈**：指出问题的同时必须提供解决方案
- **解释原因**：每个问题都要说明"为什么"这样不好
- **区分严重级别**：不要把所有问题都标记为 CRITICAL
- **尊重风格差异**：LOW 级别的建议仅供参考，不阻碍合并

## 参考文档

- [审查清单](references/review-checklist.md) — 按类别的详细检查项
- [审查流程](references/review-process.md) — 审查流程和反馈模式

## Output Format

```markdown
# Code Review: {审查范围}

## 审查结果概要
{问题总数按严重程度分类}

## 问题清单
{按严重程度排序的问题列表，含文件路径和修复建议}

## 设计审查
{审查结果清单，标记通过/未通过}
```
