---
name: deployment-strategies
version: 1.0.0
description: 部署策略详解 — 蓝绿、金丝雀、滚动部署与功能标志
last_updated: 2026-03-17
---

# 部署策略

## 策略对比总览

| 策略 | 风险等级 | 回滚速度 | 复杂度 | 停机时间 | 适用场景 |
| --- | --- | --- | --- | --- | --- |
| 蓝绿部署 | 低 | 即时（秒级） | 中 | 零 | 关键业务系统 |
| 金丝雀部署 | 最低 | 快（分钟级） | 高 | 零 | 高流量生产环境 |
| 滚动部署 | 中 | 中（分钟级） | 低 | 接近零 | 通用 Web 服务 |
| 功能标志 | 最低 | 即时（秒级） | 中 | 零 | 功能渐进发布 |

## 蓝绿部署

### 核心概念

蓝绿部署就像剧院的双舞台——一个舞台正在演出（蓝），另一个在幕后准备新剧目（绿）。准备完毕后，灯光瞬间切换到新舞台，观众感知不到任何中断。

### 工作原理

1. 当前版本运行在"蓝"环境，接收所有流量
2. 新版本部署到"绿"环境，不接收流量
3. 在绿环境验证新版本正确运行
4. 将负载均衡器/DNS 切换到绿环境
5. 蓝环境保留，作为即时回滚的保障

```
流量 ──→ [负载均衡器] ──→ [蓝环境 v1.0] ← 当前服务
                          [绿环境 v1.1] ← 就绪待切换

切换后:
流量 ──→ [负载均衡器] ──→ [绿环境 v1.1] ← 当前服务
                          [蓝环境 v1.0] ← 回滚备份
```

### 实现示例

```yaml
# GitHub Actions 蓝绿部署
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 部署到绿环境
        run: |
          deploy --target green --version ${{ github.sha }}

      - name: 验证绿环境
        run: |
          for i in $(seq 1 10); do
            status=$(curl -s -o /dev/null -w '%{http_code}' "$GREEN_URL/api/health")
            if [ "$status" = "200" ]; then
              echo "绿环境验证通过"
              exit 0
            fi
            sleep 5
          done
          echo "绿环境验证失败，终止部署"
          exit 1

      - name: 切换流量到绿环境
        if: success()
        run: switch-traffic --to green

      - name: 回滚（如果切换后检测到问题）
        if: failure()
        run: switch-traffic --to blue
```

### 优缺点

**优点**：零停机、即时回滚、完整的预发布验证环境

**缺点**：需要双倍基础设施资源、数据库迁移需要特殊处理

## 金丝雀部署

### 核心概念

金丝雀部署源自矿井中的金丝雀——先让少量流量试探新版本，如果出现问题，影响范围最小。确认安全后再逐步扩大流量。

### 工作原理

1. 新版本部署到一小部分实例
2. 将 5% 的流量导向新版本
3. 监控错误率、延迟、业务指标
4. 逐步增加流量比例：5% → 25% → 50% → 100%
5. 任何阶段发现问题立即回滚

```
阶段 1:  95% ──→ [v1.0]    5% ──→ [v1.1]
阶段 2:  75% ──→ [v1.0]   25% ──→ [v1.1]
阶段 3:  50% ──→ [v1.0]   50% ──→ [v1.1]
阶段 4:   0% ──→ [v1.0]  100% ──→ [v1.1]
```

### 实现示例

```yaml
# 金丝雀部署流水线
jobs:
  canary-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 部署金丝雀实例
        run: deploy --canary --version ${{ github.sha }}

      - name: 设置 5% 流量
        run: set-traffic-split --canary 5

      - name: 监控 10 分钟
        run: |
          monitor --duration 600 --thresholds \
            error_rate=0.01 \
            p99_latency=500ms

      - name: 扩展到 50%
        run: set-traffic-split --canary 50

      - name: 监控 10 分钟
        run: |
          monitor --duration 600 --thresholds \
            error_rate=0.01 \
            p99_latency=500ms

      - name: 全量发布
        run: set-traffic-split --canary 100

  canary-rollback:
    if: failure()
    needs: canary-deploy
    runs-on: ubuntu-latest
    steps:
      - name: 回滚金丝雀
        run: |
          set-traffic-split --canary 0
          destroy --canary
```

### 监控指标

金丝雀部署的成功依赖于有效的监控：

| 指标类别 | 具体指标 | 告警阈值示例 |
| --- | --- | --- |
| 错误率 | HTTP 5xx 比例 | > 1% |
| 延迟 | P99 响应时间 | > 500ms |
| 业务指标 | 转化率、订单量 | 偏差 > 10% |
| 资源 | CPU、内存使用率 | > 80% |

## 滚动部署

### 核心概念

滚动部署就像接力赛中逐个替换队员——每次替换一个实例，新实例就绪后再替换下一个，直到所有实例都运行新版本。

### 工作原理

1. 将一个实例从负载均衡器中移除
2. 更新该实例到新版本
3. 健康检查通过后加回负载均衡器
4. 重复直到所有实例完成更新

```
开始:   [v1.0] [v1.0] [v1.0] [v1.0]
步骤1:  [v1.1] [v1.0] [v1.0] [v1.0]
步骤2:  [v1.1] [v1.1] [v1.0] [v1.0]
步骤3:  [v1.1] [v1.1] [v1.1] [v1.0]
完成:   [v1.1] [v1.1] [v1.1] [v1.1]
```

### 实现示例（Kubernetes）

```yaml
# kubernetes deployment with rolling update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多同时多出 1 个 Pod
      maxUnavailable: 0   # 不允许不可用的 Pod
  template:
    spec:
      containers:
        - name: web
          image: app:v1.1
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
```

## 功能标志

### 核心概念

功能标志就像电灯开关——代码已部署但功能默认关闭，通过运行时的开关控制是否对用户可见，无需重新部署。

### 工作原理

1. 新功能的代码被包裹在功能标志判断中
2. 部署到生产环境，功能默认关闭
3. 通过管理后台逐步开启：内部用户 → Beta 用户 → 全量
4. 发现问题时即时关闭，无需回滚部署

### 实现示例

```typescript
// 功能标志服务接口
interface FeatureFlags {
  isEnabled(flag: string, context?: UserContext): boolean;
}

// 使用示例
function renderDashboard(flags: FeatureFlags, user: User) {
  const showNewChart = flags.isEnabled("new-analytics-chart", {
    userId: user.id,
    tier: user.plan,
  });

  if (showNewChart) {
    return renderNewChart();
  }
  return renderLegacyChart();
}
```

### 功能标志服务选型

| 服务 | 特点 | 适用规模 |
| --- | --- | --- |
| 环境变量 | 最简单，需重启生效 | 小型项目 |
| LaunchDarkly | 功能丰富，实时更新 | 中大型企业 |
| Unleash | 开源自托管 | 注重数据隐私 |
| PostHog | 与产品分析集成 | 需要 A/B 测试 |
| Vercel Edge Config | 边缘配置，极低延迟 | Vercel 用户 |

### 功能标志生命周期

```
创建 → 开发 → 灰度发布 → 全量发布 → 清理代码中的标志判断
```

**重要**：功能标志是临时的。全量发布后必须在下一个迭代中清理代码中的标志判断逻辑，避免技术债务积累。

## 回滚流程

### 回滚决策标准

满足以下任一条件应立即回滚：

- 错误率超过正常水平的 2 倍
- P99 延迟超过 SLA 要求
- 关键业务流程不可用
- 数据完整性受到威胁

### 自动回滚配置

```yaml
# GitHub Actions 中的自动回滚
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 部署新版本
        id: deploy
        run: deploy --version ${{ github.sha }}

      - name: 部署后验证
        id: verify
        run: |
          sleep 60  # 等待流量稳定
          error_rate=$(get-error-rate --duration 5m)
          if (( $(echo "$error_rate > 0.02" | bc -l) )); then
            echo "错误率过高: $error_rate"
            exit 1
          fi

      - name: 自动回滚
        if: steps.verify.outcome == 'failure'
        run: |
          echo "触发自动回滚"
          rollback --to-previous-version
          notify --channel ops --message "生产环境已自动回滚"
```

### 手动回滚检查清单

1. **确认问题** — 收集错误日志、监控截图
2. **通知团队** — 在 Slack/通信频道宣布回滚
3. **执行回滚** — 使用预定义的回滚命令或流水线
4. **验证恢复** — 确认服务恢复正常
5. **事后复盘** — 分析根因，更新流程

## 健康检查模式

### 基础健康检查

```typescript
// /api/health
app.get("/api/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});
```

### 深度健康检查

验证所有依赖服务的可用性：

```typescript
// /api/health/deep
app.get("/api/health/deep", async (req, res) => {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalAPI(),
  ]);

  const results = {
    database: checks[0].status === "fulfilled" ? "ok" : "error",
    redis: checks[1].status === "fulfilled" ? "ok" : "error",
    externalAPI: checks[2].status === "fulfilled" ? "ok" : "error",
  };

  const allHealthy = Object.values(results).every((s) => s === "ok");

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? "healthy" : "degraded",
    checks: results,
    timestamp: new Date().toISOString(),
  });
});
```

### 就绪探针与存活探针

| 探针类型 | 用途 | 失败后果 |
| --- | --- | --- |
| 就绪探针 (Readiness) | 判断能否接收流量 | 从负载均衡器移除 |
| 存活探针 (Liveness) | 判断进程是否存活 | 重启容器 |
| 启动探针 (Startup) | 判断初始化是否完成 | 等待完成后再执行其他探针 |

## 平台特定部署模式

### Vercel

```yaml
# Vercel 通过 Git 集成自动部署
# vercel.json
{
  "buildCommand": "pnpm build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "regions": ["icn1", "hnd1"],  # 指定部署区域
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" }
      ]
    }
  ]
}
```

Vercel 部署特点：
- 每次 PR 自动创建预览部署
- 合并到 main 自动部署到生产
- 内置边缘网络和自动 HTTPS
- 通过 `vercel --prod` 手动部署到生产

### Railway

```yaml
# railway.toml
[build]
builder = "nixpacks"
buildCommand = "pnpm build"

[deploy]
startCommand = "pnpm start"
healthcheckPath = "/api/health"
healthcheckTimeout = 300
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 3
```

Railway 部署特点：
- 支持 Docker 和 Nixpacks 自动构建
- 每个 PR 创建独立的预览环境（含独立数据库）
- 通过 Railway CLI 手动部署

### AWS（ECS + ALB）

```yaml
# GitHub Actions 部署到 AWS ECS
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - uses: aws-actions/amazon-ecr-login@v2

      - name: 构建并推送 Docker 镜像
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: 更新 ECS 服务
        run: |
          aws ecs update-service \
            --cluster production \
            --service web-app \
            --force-new-deployment
```

AWS 部署特点：
- 完全可定制的基础设施
- 支持蓝绿部署（CodeDeploy）
- 通过 ALB 实现健康检查和流量控制
- 需要管理更多基础设施组件
