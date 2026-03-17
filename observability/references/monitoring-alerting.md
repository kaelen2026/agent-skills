---
name: monitoring-alerting
version: 1.0.0
description: 监控与告警最佳实践 — RED/USE 方法、SLO/SLI/SLA、仪表盘设计与告警策略
last_updated: 2026-03-17
---

# 监控与告警

## 核心指标

### RED 方法（面向服务）

用于衡量面向用户的服务健康状况：

| 指标 | 含义 | 度量示例 |
|------|------|----------|
| **R**ate（速率） | 每秒请求数 | `http_requests_total` |
| **E**rrors（错误） | 失败请求占比 | `http_errors_total / http_requests_total` |
| **D**uration（延迟） | 请求处理耗时 | `http_request_duration_seconds` |

每个服务必须收集这三项指标。RED 回答的核心问题是："我的服务对用户而言表现如何？"

### USE 方法（面向资源）

用于衡量基础设施资源的健康状况：

| 指标 | 含义 | 度量示例 |
|------|------|----------|
| **U**tilization（利用率） | 资源使用百分比 | CPU 使用率、内存占用率 |
| **S**aturation（饱和度） | 排队或等待的工作量 | 请求队列长度、线程池等待数 |
| **E**rrors（错误） | 资源错误事件数 | 磁盘 I/O 错误、网络丢包 |

每种关键资源（CPU、内存、磁盘、网络）都需要收集 USE 指标。

## SLO / SLI / SLA

### 定义

| 概念 | 说明 | 谁定义 |
|------|------|--------|
| **SLI**（服务等级指标） | 可测量的服务质量指标 | 工程团队 |
| **SLO**（服务等级目标） | SLI 的目标值 | 工程团队与产品团队 |
| **SLA**（服务等级协议） | 对外承诺，违反有后果 | 业务团队 |

### 示例

```
SLI：99 分位延迟
SLO：99 分位延迟 < 500ms
SLA：月度可用性 >= 99.9%，否则退还 10% 费用
```

### 错误预算

错误预算 = 1 - SLO 目标值

```
SLO = 99.9% 可用性
错误预算 = 0.1%
每月允许停机时间 = 30天 × 24小时 × 60分钟 × 0.1% = 43.2 分钟
```

**错误预算的使用规则**：

- 预算充足时：可以快速发布新功能，接受更高风险
- 预算紧张时：放慢发布节奏，优先修复稳定性问题
- 预算耗尽时：冻结功能发布，全力恢复可靠性

### SLO 配置示例

```yaml
slos:
  - name: "API 响应延迟"
    sli: "http_request_duration_seconds"
    target: 0.999          # 99.9% 的请求
    threshold: 0.5         # 在 500ms 内完成
    window: "30d"          # 滚动 30 天窗口

  - name: "API 可用性"
    sli: "http_requests_success_rate"
    target: 0.999          # 99.9% 成功率
    window: "30d"

  - name: "数据库查询延迟"
    sli: "db_query_duration_seconds"
    target: 0.99           # 99% 的查询
    threshold: 0.1         # 在 100ms 内完成
    window: "30d"
```

## 仪表盘设计

采用三层下钻结构：总览 → 服务 → 端点。

### 第一层：系统总览

展示全局健康状况，一目了然：

- 所有服务的 SLO 达标状态（绿/黄/红）
- 全局请求速率和错误率趋势
- 错误预算剩余百分比
- 活跃告警数量

### 第二层：服务详情

单个服务的深度视图：

- RED 三项指标的时序图
- 按端点分组的延迟分布（P50、P95、P99）
- 错误率按类型分类（4xx vs 5xx）
- 依赖服务的健康状态
- 资源使用情况（CPU、内存、连接池）

### 第三层：端点下钻

单个端点的诊断视图：

- 请求量和延迟的时序对比
- 慢请求的追踪链接
- 错误日志的关联查询
- 上下游依赖的调用拓扑

### 设计原则

- 最重要的信息放在左上角
- 使用一致的颜色编码：绿色=正常、黄色=警告、红色=异常
- 时间范围选择器全局生效
- 每个图表标注阈值线

## 告警规则

### 严重程度分级

| 级别 | 响应时间 | 通知方式 | 示例 |
|------|----------|----------|------|
| **P1 - 紧急** | 5 分钟内 | 电话 + 短信 + 即时消息 | 服务完全不可用、数据丢失 |
| **P2 - 严重** | 30 分钟内 | 即时消息 + 邮件 | 错误率超过 5%、SLO 即将违约 |
| **P3 - 警告** | 4 小时内 | 即时消息 | 延迟升高、资源使用率超 80% |
| **P4 - 信息** | 下一工作日 | 邮件 | 证书即将过期、磁盘使用率超 70% |

### 告警路由与升级

```yaml
alerting:
  routes:
    - severity: P1
      notify:
        - channel: pagerduty
          team: on-call
        - channel: slack
          room: "#incidents"
      escalation:
        - after: 5m
          notify: on-call-primary
        - after: 15m
          notify: on-call-secondary
        - after: 30m
          notify: engineering-manager

    - severity: P2
      notify:
        - channel: slack
          room: "#alerts"
        - channel: email
          team: service-owners
      escalation:
        - after: 30m
          notify: on-call-primary

    - severity: P3
      notify:
        - channel: slack
          room: "#alerts"

    - severity: P4
      notify:
        - channel: email
          team: service-owners
```

### 告警规则示例

```yaml
alerts:
  - name: "高错误率"
    condition: "error_rate > 0.05 for 5m"
    severity: P2
    runbook: "runbooks/high-error-rate.md"

  - name: "高延迟"
    condition: "p99_latency > 2s for 10m"
    severity: P3
    runbook: "runbooks/high-latency.md"

  - name: "服务不可用"
    condition: "health_check_failed for 2m"
    severity: P1
    runbook: "runbooks/service-down.md"

  - name: "错误预算即将耗尽"
    condition: "error_budget_remaining < 20%"
    severity: P2
    runbook: "runbooks/error-budget-low.md"
```

## 告警疲劳防治

告警疲劳是监控系统最大的敌人。当团队对告警麻木时，真正的问题也会被忽略。

### 原则

1. **每条告警必须可操作** — 收到告警后必须有明确的处理步骤
2. **不可操作的告警必须删除** — 仅供参考的信息不应成为告警
3. **定期审查告警有效性** — 每月回顾一次，清理无效告警
4. **设置合理阈值** — 避免抖动导致反复触发

### 告警审查指标

- 每周告警总数（目标：不超过 20 条）
- 需要人工介入的告警占比（目标：> 80%）
- 告警到处理的平均时间
- 重复告警比例（目标：< 10%）

## 运行手册模板

每条告警必须关联运行手册：

```markdown
# [告警名称]

## 概述
简要说明此告警的含义和影响范围。

## 触发条件
告警的触发条件和阈值。

## 影响评估
- 受影响的用户群体
- 受影响的功能
- 业务影响程度

## 诊断步骤
1. 检查仪表盘：[仪表盘链接]
2. 查看相关日志：[日志查询语句]
3. 检查依赖服务状态
4. 检查最近的部署记录

## 修复步骤
1. 第一步...
2. 第二步...
3. 如果以上步骤无效，升级至 [团队/负责人]

## 回滚步骤
如需回滚最近的部署：
1. 回滚命令...
2. 验证步骤...

## 事后跟进
- 创建事后分析文档
- 更新此运行手册（如有新发现）
```

## 工具选型

| 工具 | 适用场景 | 特点 |
|------|----------|------|
| **Prometheus + Grafana** | 自建监控栈 | 开源免费，社区生态丰富，需要运维投入 |
| **Datadog** | 全栈可观测性 | 日志/指标/追踪统一平台，开箱即用，成本较高 |
| **Vercel Analytics** | Vercel 部署的前端项目 | 零配置，Web Vitals 内置，功能相对有限 |

### 选型建议

- **初创团队 / 小规模项目**：Vercel Analytics + 基础 CloudWatch
- **中等规模**：Datadog（减少运维负担，专注业务）
- **大规模 / 成本敏感**：Prometheus + Grafana + ELK（自建可控）
