# Authorization Patterns

## RBAC（Role-Based Access Control）

### 角色层级

上级角色继承下级角色的所有权限：

```
superAdmin > admin > manager > user > guest
```

| 角色         | 说明                                 |
| ------------ | ------------------------------------ |
| `superAdmin` | 全部权限 + 系统设置修改              |
| `admin`      | 所有资源的 CRUD + 用户管理           |
| `manager`    | 负责团队的资源管理                   |
| `user`       | 自己资源的 CRUD                      |
| `guest`      | 仅读取（限公开资源）                 |

### 权限矩阵

| 操作                 | superAdmin | admin | manager | user | guest |
| -------------------- | ---------- | ----- | ------- | ---- | ----- |
| 创建用户             | Yes        | Yes   | No      | No   | No    |
| 用户列表             | Yes        | Yes   | Yes     | No   | No    |
| 编辑自己的资料       | Yes        | Yes   | Yes     | Yes  | No    |
| 编辑其他用户         | Yes        | Yes   | No      | No   | No    |
| 删除用户             | Yes        | Yes   | No      | No   | No    |
| 创建资源             | Yes        | Yes   | Yes     | Yes  | No    |
| 查看资源             | Yes        | Yes   | Yes     | Yes  | Yes   |
| 编辑资源（自己的）   | Yes        | Yes   | Yes     | Yes  | No    |
| 编辑资源（他人的）   | Yes        | Yes   | Yes     | No   | No    |
| 删除资源             | Yes        | Yes   | Yes     | No   | No    |
| 系统设置             | Yes        | No    | No      | No   | No    |
| 查看审计日志         | Yes        | Yes   | No      | No   | No    |

### RBAC 中间件

```typescript
type Role = "superAdmin" | "admin" | "manager" | "user" | "guest";

const ROLE_HIERARCHY: Record<Role, number> = {
  superAdmin: 50,
  admin: 40,
  manager: 30,
  user: 20,
  guest: 10,
};

const authorize = (...allowedRoles: Role[]) => {
  return (req: Request, res: Response, next: NextFunction): void => {
    const userRole = req.user?.role as Role;

    if (!userRole) {
      res.status(401).json({
        success: false,
        error: {
          code: "AUTHENTICATION_REQUIRED",
          message: "Authentication is required",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    const hasPermission = allowedRoles.some(
      (allowed) => ROLE_HIERARCHY[userRole] >= ROLE_HIERARCHY[allowed]
    );

    if (!hasPermission) {
      res.status(403).json({
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "You do not have permission to perform this action",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    next();
  };
};

// 使用示例
router.get("/v1/users", authenticate, authorize("admin"), listUsers);
router.post("/v1/users", authenticate, authorize("admin"), createUser);
router.patch("/v1/users/:id", authenticate, authorize("user"), updateUser);
router.delete("/v1/users/:id", authenticate, authorize("admin"), deleteUser);
```

## ABAC（Attribute-Based Access Control）

### 策略规则结构

ABAC 通过 4 个属性类别判定访问权限：

| 类别          | 说明                           | 示例                               |
| ------------- | ------------------------------ | ---------------------------------- |
| Subject       | 请求者的属性                   | role, department, clearanceLevel   |
| Resource      | 目标资源的属性                 | ownerId, classification, status    |
| Action        | 执行的操作                     | read, write, delete, approve       |
| Environment   | 执行环境的属性                 | time, ipAddress, deviceType        |

### 策略定义

```typescript
interface Policy {
  name: string;
  description: string;
  effect: "allow" | "deny";
  conditions: {
    subject?: Record<string, unknown>;
    resource?: Record<string, unknown>;
    action?: string[];
    environment?: Record<string, unknown>;
  };
}

const policies: Policy[] = [
  {
    name: "owner-full-access",
    description: "资源所有者拥有完全访问权限",
    effect: "allow",
    conditions: {
      subject: { matchesResourceOwner: true },
      action: ["read", "write", "delete"],
    },
  },
  {
    name: "business-hours-only",
    description: "机密资源仅在工作时间内可访问",
    effect: "allow",
    conditions: {
      resource: { classification: "confidential" },
      action: ["read"],
      environment: { timeRange: { start: "09:00", end: "18:00" } },
    },
  },
  {
    name: "deny-external-delete",
    description: "拒绝来自外部网络的删除操作",
    effect: "deny",
    conditions: {
      action: ["delete"],
      environment: { isInternalNetwork: false },
    },
  },
];
```

### ABAC 策略引擎

```typescript
interface PolicyContext {
  subject: {
    id: string;
    role: string;
    department: string;
    clearanceLevel: number;
  };
  resource: {
    id: string;
    ownerId: string;
    classification: string;
    status: string;
  };
  action: string;
  environment: {
    ipAddress: string;
    deviceType: string;
    timestamp: Date;
    isInternalNetwork: boolean;
  };
}

const evaluatePolicy = (context: PolicyContext, policy: Policy): boolean => {
  const { conditions } = policy;

  if (conditions.action && !conditions.action.includes(context.action)) {
    return false;
  }

  if (conditions.subject?.matchesResourceOwner) {
    if (context.subject.id !== context.resource.ownerId) {
      return false;
    }
  }

  if (conditions.resource) {
    for (const [key, value] of Object.entries(conditions.resource)) {
      if (context.resource[key as keyof typeof context.resource] !== value) {
        return false;
      }
    }
  }

  if (conditions.environment) {
    for (const [key, value] of Object.entries(conditions.environment)) {
      if (context.environment[key as keyof typeof context.environment] !== value) {
        return false;
      }
    }
  }

  return true;
};

const evaluateAccess = (context: PolicyContext, policies: Policy[]): boolean => {
  // 先评估 Deny 策略（只要有一条 deny 就拒绝）
  const denyPolicies = policies.filter((p) => p.effect === "deny");
  const allowPolicies = policies.filter((p) => p.effect === "allow");

  for (const policy of denyPolicies) {
    if (evaluatePolicy(context, policy)) {
      return false;
    }
  }

  for (const policy of allowPolicies) {
    if (evaluatePolicy(context, policy)) {
      return true;
    }
  }

  // 默认: 拒绝
  return false;
};
```

### RBAC 与 ABAC 的选择标准

| 标准               | RBAC           | ABAC                   |
| ------------------ | -------------- | ---------------------- |
| 角色数量           | 少（< 10）     | 多或动态               |
| 权限粒度           | 粗              | 细                     |
| 环境条件（时间等） | 不需要         | 需要                   |
| 实现复杂度         | 低             | 高                     |
| 性能               | 快速           | 取决于策略数量         |
| 推荐场景           | 一般 Web 应用  | 金融、医疗、企业级应用 |

## 资源级权限（所有者检查）

确保用户只能操作自己的资源：

```typescript
const authorizeOwner = (resourceField: string = "userId") => {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const userId = req.user?.sub;
    const resourceId = req.params.id;

    if (!userId) {
      res.status(401).json({
        success: false,
        error: {
          code: "AUTHENTICATION_REQUIRED",
          message: "Authentication is required",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    const resource = await db.findById(resourceId);

    if (!resource) {
      res.status(404).json({
        success: false,
        error: {
          code: "RESOURCE_NOT_FOUND",
          message: "Resource not found",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    // admin 及以上可绕过
    const userRole = req.user?.role as string;
    const isAdmin = ROLE_HIERARCHY[userRole as Role] >= ROLE_HIERARCHY.admin;

    if (!isAdmin && resource[resourceField] !== userId) {
      res.status(403).json({
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "You can only access your own resources",
        },
        meta: { requestId: req.requestId },
      });
      return;
    }

    req.resource = resource;
    next();
  };
};

// 使用示例
router.patch(
  "/v1/posts/:id",
  authenticate,
  authorizeOwner("authorId"),
  updatePost
);
```

## Row-Level Security（行级安全）

### PostgreSQL RLS

```sql
-- 启用 RLS
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- 所有用户仅可查看自己的帖子
CREATE POLICY posts_select_own ON posts
  FOR SELECT
  USING (user_id = current_setting('app.current_user_id')::uuid);

-- 所有用户仅可更新自己的帖子
CREATE POLICY posts_update_own ON posts
  FOR UPDATE
  USING (user_id = current_setting('app.current_user_id')::uuid);

-- admin 可查看所有帖子
CREATE POLICY posts_select_admin ON posts
  FOR SELECT
  USING (current_setting('app.current_user_role') = 'admin');

-- admin 可更新所有帖子
CREATE POLICY posts_update_admin ON posts
  FOR UPDATE
  USING (current_setting('app.current_user_role') = 'admin');
```

### Supabase RLS

```sql
-- 使用 Supabase 的 auth.uid()
CREATE POLICY "Users can view own posts"
  ON posts FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own posts"
  ON posts FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own posts"
  ON posts FOR UPDATE
  USING (auth.uid() = user_id);
```

### RLS 中间件（应用层）

```typescript
const setRLSContext = async (
  req: Request,
  _res: Response,
  next: NextFunction
): Promise<void> => {
  const userId = req.user?.sub;
  const userRole = req.user?.role;

  await db.$executeRaw`SELECT set_config('app.current_user_id', ${userId}, true)`;
  await db.$executeRaw`SELECT set_config('app.current_user_role', ${userRole}, true)`;

  next();
};

// 认证后设置 RLS 上下文
router.use(authenticate, setRLSContext);
```

## API 端点授权矩阵

| Method | Endpoint              | guest | user | manager | admin | superAdmin | 所有者检查 |
| ------ | --------------------- | ----- | ---- | ------- | ----- | ---------- | -------------- |
| POST   | /v1/auth/signup       | Yes   | Yes  | Yes     | Yes   | Yes        | No             |
| POST   | /v1/auth/login        | Yes   | Yes  | Yes     | Yes   | Yes        | No             |
| POST   | /v1/auth/refresh      | No    | Yes  | Yes     | Yes   | Yes        | No             |
| POST   | /v1/auth/logout       | No    | Yes  | Yes     | Yes   | Yes        | No             |
| GET    | /v1/users             | No    | No   | Yes     | Yes   | Yes        | No             |
| GET    | /v1/users/:id         | No    | Yes  | Yes     | Yes   | Yes        | Yes            |
| PATCH  | /v1/users/:id         | No    | Yes  | No      | Yes   | Yes        | Yes            |
| DELETE | /v1/users/:id         | No    | No   | No      | Yes   | Yes        | No             |
| GET    | /v1/posts             | Yes   | Yes  | Yes     | Yes   | Yes        | No             |
| POST   | /v1/posts             | No    | Yes  | Yes     | Yes   | Yes        | No             |
| PATCH  | /v1/posts/:id         | No    | Yes  | Yes     | Yes   | Yes        | Yes            |
| DELETE | /v1/posts/:id         | No    | No   | Yes     | Yes   | Yes        | Yes            |
| GET    | /v1/admin/audit-logs  | No    | No   | No      | Yes   | Yes        | No             |
| PATCH  | /v1/admin/settings    | No    | No   | No      | No    | Yes        | No             |

## 错误响应: INSUFFICIENT_PERMISSIONS（403）

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "You do not have permission to perform this action"
  },
  "meta": {
    "requestId": "req_err004"
  }
}
```

### 注意事项

- 403 响应中不要包含用户的角色或所需角色信息（防止信息泄露）
- 即使资源不存在，权限不足时也应返回 404 而非 403（防止推测资源是否存在）
- 所有授权操作均应记录到审计日志
