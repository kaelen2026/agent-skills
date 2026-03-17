# Agent Skills 完整使用指南：从0实现用户评论系统

> 本文档演示如何在一个真实需求中，串联使用 agent-skills 项目的 17 个 skill，从需求分析到代码交付的完整流程。

## 需求概述

**功能**: 用户评论系统（支持发表、回复、点赞、举报）
**技术栈**: Next.js + TypeScript + Prisma + PostgreSQL
**目标**: 演示每个阶段激活哪些 skill，以及它们如何协作

---

## 全局流程总览

```
需求 → planning → api-design → database → auth → error-handling
     → backend-patterns → frontend-patterns → coding-standards
     → tdd-workflow → e2e-testing → security-review → code-review
     → git-workflow → ci-cd → observability → documentation → performance
```

| 阶段 | 激活的 Skill | 输出物 |
|------|-------------|--------|
| 1. 需求规划 | `planning` | spec.md, plan.md |
| 2. API 设计 | `api-design` | openapi.yaml, API.md, 路由代码 |
| 3. 数据库设计 | `database` | schema.prisma, 迁移脚本 |
| 4. 认证授权 | `auth` | JWT 中间件, 权限控制 |
| 5. 错误处理 | `error-handling` | AppError 类, 错误处理中间件 |
| 6. 后端实现 | `backend-patterns` | Repository/Service/Controller 层 |
| 7. 前端实现 | `frontend-patterns` | React 组件, 自定义 hooks |
| 8. 编码规范 | `coding-standards` | 贯穿全程的质量检查 |
| 9. TDD 测试 | `tdd-workflow` | 单元测试, 集成测试 |
| 10. E2E 测试 | `e2e-testing` | Playwright 端到端测试 |
| 11. 安全审查 | `security-review` | 安全问题清单, 修复方案 |
| 12. 代码审查 | `code-review` | 审查报告, 问题修复 |
| 13. Git 工作流 | `git-workflow` | 分支策略, commit 规范 |
| 14. CI/CD | `ci-cd` | GitHub Actions 流水线 |
| 15. 可观测性 | `observability` | 日志, 监控, 告警 |
| 16. 文档 | `documentation` | README, ADR, CONTRIBUTING |
| 17. 性能优化 | `performance` | 性能预算, 优化方案 |

---

## 阶段 1: 需求规划

### 激活 Skill: `planning`

**触发条件**: 新功能设计 + 预计修改 3+ 文件 → HARD-GATE 触发

#### 步骤 1.1: 判断文档类型

本需求属于"大型功能"，需要生成全部三类文档：
- 功能规格书 (Spec) → 定义"做什么"
- 架构设计 → 定义"怎么做"
- 实施计划 (Plan) → 定义"分几步做"

#### 步骤 1.2: 收集需求

向用户确认（`planning` skill 的标准收集项）：

| 项目 | 值 |
|------|-----|
| 功能名称 | 用户评论系统 |
| 背景 | 产品需要用户互动功能，评论是核心互动形式 |
| 目标用户 | 已注册用户（发评论）、所有访客（看评论） |
| 范围 | 包含：评论 CRUD、回复、点赞、举报。不包含：评论审核后台、AI 内容过滤 |
| 约束 | 使用现有的 Next.js 项目，Prisma ORM，PostgreSQL |
| 非功能需求 | 评论列表加载 < 200ms，支持分页，防 XSS |

#### 步骤 1.3: 生成功能规格书

按 `planning/references/spec-template.md` 模板生成：

```markdown
# 功能规格书: 用户评论系统

## 概要
为平台添加用户评论功能，支持发表评论、回复评论、点赞和举报。

## 用户故事

### US-1: 发表评论
作为已登录用户，我希望在内容页面发表评论，以便分享我的想法。

### US-2: 回复评论
作为已登录用户，我希望回复其他用户的评论，以便进行讨论。

### US-3: 点赞评论
作为已登录用户，我希望给优质评论点赞，以便表达认同。

### US-4: 举报评论
作为已登录用户，我希望举报不当评论，以便维护社区秩序。

### US-5: 浏览评论
作为任意访客，我希望浏览内容页面的评论列表，以便了解他人看法。

## 功能需求

### MUST（必须实现）
- FR-1: 评论 CRUD（创建、读取、更新、删除）
- FR-2: 嵌套回复（支持一层回复）
- FR-3: 评论分页（cursor-based）
- FR-4: 点赞/取消点赞
- FR-5: 举报评论
- FR-6: 认证保护（发表/修改/删除需要登录）

### SHOULD（应该实现）
- FR-7: 评论排序（最新/最热）
- FR-8: 评论数统计
- FR-9: 点赞数统计

### COULD（可以实现）
- FR-10: 评论@提及通知
- FR-11: 富文本评论（Markdown）

## API 变更
详见 API 设计阶段

## DB 变更
详见数据库设计阶段

## 非功能需求
| 指标 | 目标值 |
|------|--------|
| 评论列表响应时间 | P95 < 200ms |
| 单次分页数量 | 20 条 |
| 评论内容长度 | 最大 2000 字符 |
| 嵌套深度 | 最大 1 层 |

## 验收标准 (GIVEN-WHEN-THEN)

### AC-1: 发表评论
GIVEN 用户已登录
WHEN 用户提交评论内容
THEN 评论创建成功，返回评论详情，评论出现在列表中

### AC-2: 未登录发表
GIVEN 用户未登录
WHEN 用户尝试发表评论
THEN 返回 401 Unauthorized

### AC-3: 删除他人评论
GIVEN 用户已登录
WHEN 用户尝试删除不属于自己的评论
THEN 返回 403 Forbidden

## 不包含（Out of Scope）
- 评论审核管理后台
- AI 内容过滤
- 评论导出功能
```

#### 步骤 1.4: 生成实施计划

按 `planning/references/plan-template.md` 模板：

```markdown
# 实施计划: 用户评论系统

## 阶段划分

### Phase 1: 基础设施（数据库 + 错误处理）
| Task | 描述 | 文件 | 依赖 |
|------|------|------|------|
| T-1 | 设计评论相关数据库表 | prisma/schema.prisma | - |
| T-2 | 创建 AppError 类和错误码 | src/lib/errors.ts | - |
| T-3 | 创建错误处理中间件 | src/middleware/error-handler.ts | T-2 |

### Phase 2: 核心 API
| Task | 描述 | 文件 | 依赖 |
|------|------|------|------|
| T-4 | 评论 Repository 层 | src/repositories/comment.ts | T-1 |
| T-5 | 评论 Service 层 | src/services/comment.ts | T-4, T-2 |
| T-6 | 评论 API 路由 | src/app/api/comments/route.ts | T-5, T-3 |
| T-7 | 点赞 API 路由 | src/app/api/comments/[id]/like/route.ts | T-5 |
| T-8 | 举报 API 路由 | src/app/api/comments/[id]/report/route.ts | T-5 |

### Phase 3: 前端组件
| Task | 描述 | 文件 | 依赖 |
|------|------|------|------|
| T-9 | CommentList 组件 | src/components/comments/CommentList.tsx | T-6 |
| T-10 | CommentForm 组件 | src/components/comments/CommentForm.tsx | T-6 |
| T-11 | CommentItem 组件 | src/components/comments/CommentItem.tsx | T-6 |
| T-12 | useComments hook | src/hooks/useComments.ts | T-6 |

### Phase 4: 测试与安全
| Task | 描述 | 文件 | 依赖 |
|------|------|------|------|
| T-13 | 单元测试 | src/**/*.test.ts | T-4~T-8 |
| T-14 | 集成测试 | src/app/api/**/*.test.ts | T-6~T-8 |
| T-15 | E2E 测试 | e2e/comments.spec.ts | T-9~T-12 |
| T-16 | 安全审查 | - | T-6~T-8 |

## 风险评估
| 风险 | 影响 | 应对 |
|------|------|------|
| N+1 查询导致性能问题 | HIGH | 使用 Prisma include 预加载 |
| XSS 攻击通过评论内容 | CRITICAL | 使用 DOMPurify 净化 |
| 刷赞/刷举报 | MEDIUM | 速率限制 + 唯一约束 |
```

#### 步骤 1.5: 设计审查

`planning` skill 自动检查清单：
- [x] 功能范围明确（包含/不包含已定义）
- [x] 用户故事覆盖主要角色和场景（5 个故事）
- [x] API 变更包含完整端点（Phase 2 定义）
- [x] DB 变更包含表结构（Phase 1 定义）
- [x] 非功能需求有可量化目标值
- [x] 验收标准可测试
- [x] 实施阶段依赖关系清晰
- [x] 风险评估覆盖技术和业务风险
- [x] 文件清单完整
- [x] 集成点已识别

---

## 阶段 2: API 设计

### 激活 Skill: `api-design`

**触发条件**: 用户请求设计 REST API 端点

#### 步骤 2.1: 收集需求

| 项目 | 值 |
|------|-----|
| 资源名称 | comments |
| 操作 | CRUD + 点赞 + 举报 |
| 认证方式 | Bearer Token |
| 框架 | Next.js API Routes |
| OpenAPI 版本 | 3.1 |
| 分页 | cursor |
| 版本策略 | URL prefix (`/v1`) |
| 响应格式 | 统一信封 `{ success, data?, meta?, error? }` |

#### 步骤 2.2: 设计 API（结论先行）

```
## API 端点总览

| Method | Path                            | Description      | Auth |
| ------ | ------------------------------- | ---------------- | ---- |
| GET    | /v1/comments                    | 评论列表（分页） | No   |
| POST   | /v1/comments                    | 创建评论         | Yes  |
| GET    | /v1/comments/:id                | 获取评论详情     | No   |
| PATCH  | /v1/comments/:id                | 更新评论         | Yes  |
| DELETE | /v1/comments/:id                | 删除评论         | Yes  |
| POST   | /v1/comments/:id/like           | 点赞/取消点赞    | Yes  |
| POST   | /v1/comments/:id/report         | 举报评论         | Yes  |
| GET    | /v1/comments/:id/replies        | 获取回复列表     | No   |
```

#### 步骤 2.3: 生成输出物

**A. OpenAPI 规范 (`openapi.yaml`)**
遵循 `api-design/references/openapi-conventions.md`：
- 使用 PascalCase schema 命名: `Comment`, `CreateCommentRequest`, `CommentResponse`
- 属性 camelCase，布尔前缀 `is`/`has`，时间戳后缀 `At`
- 统一响应信封

**B. API 文档 (`API.md`)**
遵循 `api-design/references/documentation-template.md`：
- 包含认证说明、请求/响应示例、错误码、速率限制

**C. 路由代码**
遵循 `api-design/references/design-rules.md`：
- Next.js App Router 风格
- zod 请求体验证
- 统一错误响应

#### 步骤 2.4: 设计审查

`api-design` skill 检查清单：
- [x] URL 使用复数名词 (`/comments`)
- [x] 正确使用 HTTP 方法语义
- [x] 一致的错误响应格式
- [x] 统一响应信封 (`success`, `data?`, `meta?`, `error?`)
- [x] `meta.requestId` 始终存在
- [x] 所有响应属性使用 camelCase
- [x] `X-Device-Type` header 已定义和验证
- [x] 分页参数规范 (cursor-based)
- [x] 幂等性保证 (DELETE)
- [x] 请求体验证 (zod schema)
- [x] 无敏感信息泄露

---

## 阶段 3: 数据库设计

### 激活 Skill: `database`

**触发条件**: 新增数据表

#### 步骤 3.1: Schema 设计

遵循 `database/references/schema-design.md` 命名规范：
- 表名: 复数 snake_case (`comments`, `comment_likes`, `comment_reports`)
- 列名: snake_case，主键 `id` (UUID)，外键 `{singular}_id`
- 布尔前缀 `is_`/`has_`，时间戳后缀 `_at`
- 标准审计列: `id`, `created_at`, `updated_at`, `deleted_at`

```prisma
// prisma/schema.prisma

model Comment {
  id        String   @id @default(cuid())
  content   String   @db.VarChar(2000)
  authorId  String   @map("author_id")
  targetId  String   @map("target_id")       // 被评论的内容 ID
  targetType String  @map("target_type")      // 被评论的内容类型
  parentId  String?  @map("parent_id")        // 回复的父评论 ID
  isDeleted Boolean  @default(false) @map("is_deleted")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  author    User      @relation(fields: [authorId], references: [id])
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentReplies")
  likes     CommentLike[]
  reports   CommentReport[]

  @@index([targetId, targetType, createdAt])
  @@index([parentId])
  @@index([authorId])
  @@map("comments")
}

model CommentLike {
  id        String   @id @default(cuid())
  commentId String   @map("comment_id")
  userId    String   @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")

  comment   Comment  @relation(fields: [commentId], references: [id])
  user      User     @relation(fields: [userId], references: [id])

  @@unique([commentId, userId])  // 防止重复点赞
  @@map("comment_likes")
}

model CommentReport {
  id        String   @id @default(cuid())
  commentId String   @map("comment_id")
  userId    String   @map("user_id")
  reason    String   @db.VarChar(500)
  createdAt DateTime @default(now()) @map("created_at")

  comment   Comment  @relation(fields: [commentId], references: [id])
  user      User     @relation(fields: [userId], references: [id])

  @@unique([commentId, userId])  // 防止重复举报
  @@map("comment_reports")
}
```

#### 步骤 3.2: 查询优化

遵循 `database/references/query-patterns.md`：
- 使用 Prisma `include` 预加载防止 N+1
- cursor-based 分页
- 选择性 `select` 避免 SELECT *

#### 步骤 3.3: 迁移策略

遵循 `database/references/migration-patterns.md`：
- 仅前进迁移（无 down migration）
- 新增表不需要 expand-contract（首次创建）
- 添加索引使用 `CREATE INDEX CONCURRENTLY`

---

## 阶段 4: 认证授权

### 激活 Skill: `auth`

**触发条件**: 实现认证或授权功能

#### 步骤 4.1: JWT 策略

遵循 `auth/references/jwt-patterns.md`：

```typescript
// JWT Payload 结构
interface JwtPayload {
  sub: string;        // userId
  role: string;       // user role
  deviceType: string; // web | app | desktop
  iat: number;
  exp: number;
}

// Access Token: ≤15 分钟
// Refresh Token: ≤7 天
// 存储: web → httpOnly Secure Cookie
```

#### 步骤 4.2: 授权规则

| 操作 | 权限要求 |
|------|---------|
| 浏览评论 | 无需登录 |
| 发表评论 | 已登录用户 |
| 编辑评论 | 仅作者本人 |
| 删除评论 | 作者本人 OR 管理员 |
| 点赞 | 已登录用户 |
| 举报 | 已登录用户 |

#### 步骤 4.3: 安全检查清单

遵循 `auth/references/security-checklist.md`：
- [x] 使用 httpOnly Secure Cookie 存储 Token
- [x] Token 过期时间合理（Access ≤15min）
- [x] 授权检查在所有敏感操作前执行
- [x] 资源所有权验证（编辑/删除需要验证 authorId）

---

## 阶段 5: 错误处理

### 激活 Skill: `error-handling`

**触发条件**: 新增 API 端点

#### 步骤 5.1: 错误分类

遵循 `error-handling/references/error-taxonomy.md`：

```typescript
// src/lib/errors.ts
export class AppError extends Error {
  constructor(
    public readonly code: string,
    public readonly statusCode: number,
    message: string,
    public readonly isOperational = true,
    public readonly details?: unknown
  ) {
    super(message);
  }
}

// 评论相关错误码 (UPPER_SNAKE_CASE)
export const CommentErrors = {
  COMMENT_NOT_FOUND: new AppError("COMMENT_NOT_FOUND", 404, "评论不存在"),
  COMMENT_CONTENT_EMPTY: new AppError("COMMENT_CONTENT_EMPTY", 400, "评论内容不能为空"),
  COMMENT_CONTENT_TOO_LONG: new AppError("COMMENT_CONTENT_TOO_LONG", 400, "评论内容超过 2000 字符"),
  COMMENT_NOT_OWNER: new AppError("COMMENT_NOT_OWNER", 403, "无权操作此评论"),
  COMMENT_ALREADY_LIKED: new AppError("COMMENT_ALREADY_LIKED", 409, "已经点赞过"),
  COMMENT_ALREADY_REPORTED: new AppError("COMMENT_ALREADY_REPORTED", 409, "已经举报过"),
  COMMENT_REPLY_DEPTH_EXCEEDED: new AppError("COMMENT_REPLY_DEPTH_EXCEEDED", 400, "不支持多层嵌套回复"),
} as const;
```

#### 步骤 5.2: 错误处理中间件

遵循 `error-handling/references/handling-patterns.md`：

```typescript
// src/middleware/error-handler.ts
export function handleApiError(error: unknown) {
  if (error instanceof AppError && error.isOperational) {
    return NextResponse.json(
      {
        success: false,
        error: {
          code: error.code,
          message: error.message,
          details: error.details,
        },
        meta: { requestId: generateRequestId() },
      },
      { status: error.statusCode }
    );
  }

  // 非预期错误 → 记录日志，返回通用错误
  console.error("Unexpected error:", error);
  return NextResponse.json(
    {
      success: false,
      error: {
        code: "INTERNAL_ERROR",
        message: "服务器内部错误",
      },
      meta: { requestId: generateRequestId() },
    },
    { status: 500 }
  );
}
```

---

## 阶段 6: 后端实现

### 激活 Skill: `backend-patterns`

**触发条件**: 后端架构设计

#### 步骤 6.1: 分层架构

遵循 `backend-patterns` 的 Repository → Service → Controller 模式：

```
Controller (API Route)
    ↓ 验证输入、调用 Service
Service (业务逻辑)
    ↓ 编排 Repository、应用业务规则
Repository (数据访问)
    ↓ Prisma 查询
Database (PostgreSQL)
```

#### 步骤 6.2: Repository 层

```typescript
// src/repositories/comment.ts — 纯数据访问，不含业务逻辑
export function createCommentRepository(prisma: PrismaClient) {
  return {
    async findById(id: string) {
      return prisma.comment.findUnique({
        where: { id, isDeleted: false },
        include: { author: { select: { id: true, name: true, avatar: true } } },
      });
    },

    async findByTarget(targetId: string, targetType: string, cursor?: string) {
      return prisma.comment.findMany({
        where: { targetId, targetType, parentId: null, isDeleted: false },
        include: {
          author: { select: { id: true, name: true, avatar: true } },
          _count: { select: { likes: true, replies: true } },
        },
        orderBy: { createdAt: "desc" },
        take: 21,  // 多取一条判断 hasMore
        ...(cursor ? { cursor: { id: cursor }, skip: 1 } : {}),
      });
    },

    async create(data: CreateCommentInput) {
      return prisma.comment.create({
        data,
        include: { author: { select: { id: true, name: true, avatar: true } } },
      });
    },
    // ... update, softDelete, like, unlike, report
  };
}
```

#### 步骤 6.3: Service 层

```typescript
// src/services/comment.ts — 业务逻辑、权限检查
export function createCommentService(repo: CommentRepository) {
  return {
    async createComment(userId: string, input: CreateCommentInput) {
      // 业务规则: 回复只允许一层
      if (input.parentId) {
        const parent = await repo.findById(input.parentId);
        if (!parent) throw CommentErrors.COMMENT_NOT_FOUND;
        if (parent.parentId) throw CommentErrors.COMMENT_REPLY_DEPTH_EXCEEDED;
      }
      return repo.create({ ...input, authorId: userId });
    },

    async deleteComment(userId: string, commentId: string) {
      const comment = await repo.findById(commentId);
      if (!comment) throw CommentErrors.COMMENT_NOT_FOUND;
      if (comment.authorId !== userId) throw CommentErrors.COMMENT_NOT_OWNER;
      return repo.softDelete(commentId);
    },
    // ...
  };
}
```

#### 步骤 6.4: 设计审查

`backend-patterns` skill 检查清单：
- [x] Repository/Service 分离
- [x] Controller 不含业务逻辑
- [x] 统一响应信封
- [x] camelCase 属性
- [x] X-Device-Type 验证
- [x] N+1 预防（使用 include）
- [x] SELECT 优化（使用 select）
- [x] 软删除
- [x] 不可变性（spread operator 创建新对象）
- [x] 无硬编码配置

---

## 阶段 7: 前端实现

### 激活 Skill: `frontend-patterns`

**触发条件**: React 组件设计

#### 步骤 7.1: 组件层级

```
CommentSection (容器组件)
├── CommentForm (表单组件)
├── CommentList (列表组件)
│   ├── CommentItem (单条评论)
│   │   ├── CommentActions (点赞/回复/举报按钮)
│   │   └── ReplyList (回复列表)
│   │       └── CommentItem (回复项，复用)
│   └── LoadMoreButton (加载更多)
└── useComments (自定义 hook)
```

#### 步骤 7.2: 组合模式

遵循 `frontend-patterns` 的组合优先原则：
- 使用组合而非继承
- TypeScript 接口定义 props
- 单一职责
- 不可变状态更新
- Error Boundary 包裹

#### 步骤 7.3: 自定义 Hook

```typescript
// src/hooks/useComments.ts
export function useComments(targetId: string, targetType: string) {
  const [comments, setComments] = useState<Comment[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [hasMore, setHasMore] = useState(true);
  const [isLoading, setIsLoading] = useState(false);

  const fetchComments = useCallback(async () => {
    setIsLoading(true);
    try {
      const res = await fetch(
        `/api/v1/comments?targetId=${targetId}&targetType=${targetType}${cursor ? `&cursor=${cursor}` : ""}`
      );
      const data = await res.json();
      // 不可变更新: 创建新数组
      setComments((prev) => [...prev, ...data.data]);
      setCursor(data.meta.nextCursor);
      setHasMore(data.meta.hasMore);
    } catch (error) {
      console.error("Failed to fetch comments:", error);
      throw new Error("评论加载失败");
    } finally {
      setIsLoading(false);
    }
  }, [targetId, targetType, cursor]);

  return { comments, hasMore, isLoading, fetchComments };
}
```

---

## 阶段 8: 编码规范

### 激活 Skill: `coding-standards`

**触发条件**: 贯穿全程的质量守门

`coding-standards` 在每个阶段持续检查：

- [x] 命名清晰（变量/函数/文件）
- [x] 不可变性（spread operator，无 mutation）
- [x] 函数 < 50 行
- [x] 文件 < 800 行
- [x] 嵌套 ≤ 4 层
- [x] 无 `any` 类型
- [x] 无魔法数字
- [x] 完善的错误处理
- [x] 异步并行化（Promise.all）
- [x] 无 console.log
- [x] zod 输入验证
- [x] 测试使用 AAA 模式
- [x] 测试名描述行为

---

## 阶段 9: TDD 测试

### 激活 Skill: `tdd-workflow`

**触发条件**: 开发新功能

#### 步骤 9.1: RED — 先写失败测试

```typescript
// src/services/comment.test.ts
describe("CommentService", () => {
  describe("createComment", () => {
    it("creates a comment successfully", async () => {
      // Arrange
      const mockRepo = createMockCommentRepo();
      const service = createCommentService(mockRepo);
      const input = { content: "Great post!", targetId: "post-1", targetType: "post" };

      // Act
      const result = await service.createComment("user-1", input);

      // Assert
      expect(result.content).toBe("Great post!");
      expect(result.authorId).toBe("user-1");
    });

    it("rejects nested reply (depth > 1)", async () => {
      // Arrange: parent comment 已经是 reply
      const mockRepo = createMockCommentRepo({
        findById: vi.fn().mockResolvedValue({ id: "c-1", parentId: "c-0" }),
      });
      const service = createCommentService(mockRepo);

      // Act & Assert
      await expect(
        service.createComment("user-1", { content: "nested", parentId: "c-1", targetId: "post-1", targetType: "post" })
      ).rejects.toThrow("COMMENT_REPLY_DEPTH_EXCEEDED");
    });

    it("rejects empty content", async () => { /* ... */ });
    it("rejects content over 2000 chars", async () => { /* ... */ });
  });

  describe("deleteComment", () => {
    it("deletes own comment via soft delete", async () => { /* ... */ });
    it("rejects deleting other user's comment", async () => { /* ... */ });
    it("returns 404 for non-existent comment", async () => { /* ... */ });
  });
});
```

#### 步骤 9.2: 运行测试 — 全部失败 ✗

```bash
npm test
# FAIL  src/services/comment.test.ts
# ✗ creates a comment successfully
# ✗ rejects nested reply (depth > 1)
# ✗ rejects empty content
# ...
```

#### 步骤 9.3: GREEN — 最小实现

在 Service 层写刚好让测试通过的代码。

#### 步骤 9.4: 运行测试 — 全部通过 ✓

```bash
npm test
# PASS  src/services/comment.test.ts
# ✓ creates a comment successfully (3ms)
# ✓ rejects nested reply (depth > 1) (1ms)
# ...
```

#### 步骤 9.5: IMPROVE — 重构

保持测试绿色，改善命名、消除重复、优化结构。

#### 步骤 9.6: Mock 模式

遵循 `tdd-workflow/references/mocking-patterns.md`：
- 只 mock 外部依赖（Prisma Client）
- 不 mock 被测代码内部逻辑
- 每个 mock 在 afterEach 中恢复

#### 步骤 9.7: 覆盖率验证

```bash
npm run test:coverage
# Statements: 87%  ✓ (≥80%)
# Branches:   82%  ✓
# Functions:  91%  ✓
# Lines:      85%  ✓
```

#### 步骤 9.8: 设计审查

`tdd-workflow` skill 检查清单：
- [x] 所有测试通过（0 failures）
- [x] 覆盖率 ≥ 80%
- [x] 无 skip 或 todo 测试
- [x] 单元测试总执行时间 < 30s
- [x] Mock 只用于外部依赖
- [x] 每个测试独立
- [x] 测试命名描述行为
- [x] 错误路径和边界条件已覆盖
- [x] 无 console.log 残留

---

## 阶段 10: E2E 测试

### 激活 Skill: `e2e-testing`

**触发条件**: 关键用户流程测试

遵循 `tdd-workflow/references/e2e-patterns.md`：

```typescript
// e2e/comments.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Comment System", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/posts/test-post");
  });

  test("user can post and see a comment", async ({ page }) => {
    // 填写评论
    await page.getByRole("textbox", { name: /评论/ }).fill("This is a test comment");
    await page.getByRole("button", { name: /发表/ }).click();

    // 验证评论出现
    await expect(page.getByText("This is a test comment")).toBeVisible();
  });

  test("user can like a comment", async ({ page }) => {
    await page.getByRole("button", { name: /点赞/ }).first().click();
    await expect(page.getByText("1")).toBeVisible();
  });

  test("unauthenticated user cannot post comment", async ({ page }) => {
    // 未登录状态
    await page.getByRole("textbox", { name: /评论/ }).fill("test");
    await page.getByRole("button", { name: /发表/ }).click();
    await expect(page.getByText("请先登录")).toBeVisible();
  });
});
```

选择器优先级: `getByRole` > `getByText` > `data-testid` > CSS selector

---

## 阶段 11: 安全审查

### 激活 Skill: `security-review`

**触发条件**: 创建新 API 端点 + 处理用户输入

#### 步骤 11.1: 安全问题扫描

```markdown
## 安全审查结果

发现 3 个需关注项:

| 严重度 | 类别 | 问题 | 位置 |
|--------|------|------|------|
| HIGH | XSS | 评论内容需要净化 | CommentItem.tsx |
| MEDIUM | 速率限制 | 发评论/点赞/举报需要限流 | API routes |
| MEDIUM | CSRF | POST 请求需要 CSRF 保护 | API routes |
```

#### 步骤 11.2: 修复方案

每个问题提供具体代码修复。

#### 步骤 11.3: 部署前检查清单

遵循 `security-review` 完整的 16 项检查清单逐项验证。

---

## 阶段 12: 代码审查

### 激活 Skill: `code-review`

**触发条件**: 代码编写完成

#### 步骤 12.1: 应用审查清单

遵循 `code-review/references/review-checklist.md`，按 8 个维度检查：
1. 正确性 2. 安全性 3. 性能 4. 可维护性 5. 类型安全 6. 错误处理 7. 测试覆盖 8. 不可变性

#### 步骤 12.2: 输出审查报告

```markdown
# 代码审查报告

## 概要
- 审查文件数: 15
- 变更行数: +850 / -0
- 发现问题数: CRITICAL(0) / HIGH(0) / MEDIUM(2) / LOW(3)

## 审查结论
✓ 通过（0 CRITICAL + 0 HIGH）
```

---

## 阶段 13-17: 补充 Skill

### `git-workflow` (阶段 13)
- 分支: `feat/comment-system`
- Commit 规范: Conventional Commits (`feat:`, `fix:`, `test:`)
- PR 模板: Summary + Test Plan

### `ci-cd` (阶段 14)
- GitHub Actions: lint → test → build → deploy
- 自动运行测试 + 覆盖率检查

### `observability` (阶段 15)
- 结构化日志 (pino)
- requestId 全链路传递
- 评论 API 响应时间监控

### `documentation` (阶段 16)
- ADR: "为什么选择 cursor 分页而非 offset"
- README 更新: 评论系统使用说明
- API 文档更新

### `performance` (阶段 17)
- 评论列表查询 EXPLAIN ANALYZE
- 索引优化验证
- 前端: 虚拟列表（长评论列表）、图片懒加载

---

## Skill 协作关系图

```
                    planning
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          api-design  database  auth
              │        │        │
              └────────┼────────┘
                       │
                 error-handling
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
       backend-    frontend-  coding-
       patterns    patterns   standards
              │        │        │
              └────────┼────────┘
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          tdd-     e2e-     security-
          workflow  testing  review
              │        │        │
              └────────┼────────┘
                       │
                  code-review
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          git-     ci-cd   observability
          workflow          │
                       ┌───┼───┐
                       ▼   ▼   ▼
                    docs perf  DONE ✓
```

---

## 总结: Skill 使用最佳实践

1. **总是从 `planning` 开始** — HARD-GATE 规则：3+ 文件变更必须先做计划
2. **结论先行** — 所有 skill 的输出都遵循"先结论后细节"原则
3. **设计审查是每个 skill 的最后一步** — 用清单验证质量
4. **Skill 之间有依赖关系** — planning 的输出是 api-design/database 的输入
5. **`coding-standards` 贯穿全程** — 每个阶段都要检查编码规范
6. **不可变性是核心原则** — 所有 skill 都强调不可变数据操作
7. **统一响应信封** — `{ success, data?, meta?, error? }` 是全局约定
8. **TDD 不可协商** — 先测试后实现，覆盖率 ≥ 80%
