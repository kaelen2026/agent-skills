---
name: performance
description: 性能优化技能 — Web Vitals、后端性能、负载测试与性能预算
metadata:
  filePattern:
    - "**/lighthouse*"
    - "**/k6/**/*"
    - "**/load-test*"
  bashPattern:
    - "lighthouse|k6 run|webpack-bundle-analyzer"
  priority: 4
---

# 性能优化技能

## When to Activate

以下场景应激活本技能：

- 性能优化请求（页面加载慢、API 响应慢）
- Core Web Vitals 指标不达标（LCP、INP、CLS）
- 负载测试规划与执行
- 缓存策略设计与实施
- 性能预算制定与监控
- 数据库查询优化
- 打包体积分析与优化

## Workflow

### 1. 测量基线

在优化之前，必须先获取当前性能数据作为基线。没有基线的优化是盲目的。

- **前端**: 使用 Lighthouse、WebPageTest 或 Chrome DevTools Performance 面板采集 Core Web Vitals
- **后端**: 采集 API 响应时间的 P50/P95/P99 分位数
- **数据库**: 使用 `EXPLAIN ANALYZE` 分析慢查询
- **整体**: 确认当前负载水平和错误率

```bash
# Lighthouse CLI 快速基线测量
npx lighthouse https://example.com --output=json --output-path=./baseline.json

# k6 快速冒烟测试
k6 run --vus 1 --duration 10s script.js
```

### 2. 识别瓶颈

根据基线数据，定位性能瓶颈的根本原因。

- **前端瓶颈**: 大体积 JS 包、未优化图片、布局偏移、长任务阻塞主线程
- **后端瓶颈**: 慢 SQL 查询、N+1 问题、缺少缓存、连接池耗尽
- **网络瓶颈**: 未启用压缩、缺少 CDN、过多请求
- **基础设施瓶颈**: CPU/内存不足、磁盘 I/O 瓶颈

### 3. 优化

根据瓶颈类型，参考对应的参考文档执行优化：

| 瓶颈类型 | 参考文档 |
|----------|---------|
| Core Web Vitals / 前端性能 | [references/web-vitals.md](references/web-vitals.md) |
| API / 数据库 / 缓存 | [references/backend-performance.md](references/backend-performance.md) |
| 负载能力 / 性能预算 | [references/load-testing.md](references/load-testing.md) |

**优化原则**：

1. 先优化影响最大的瓶颈（投入产出比最高的项目优先）
2. 每次只改一个变量，方便归因
3. 优化完成后立即验证效果

### 4. 验证改善

优化不以"感觉快了"为标准，必须用数据证明。

- 使用与基线相同的工具和条件重新测量
- 对比基线数据，确认指标改善
- 确保没有引入新的性能回归
- 记录优化结果，更新性能预算

```bash
# 优化后重新测量
npx lighthouse https://example.com --output=json --output-path=./after-optimization.json

# 对比前后数据
# LCP: 3.2s → 1.8s (改善 43%)
# INP: 320ms → 150ms (改善 53%)
```

## 审查清单

每次性能优化完成后，逐项确认：

### 前端性能

- [ ] LCP < 2.5s（良好阈值）
- [ ] INP < 200ms（良好阈值）
- [ ] CLS < 0.1（良好阈值）
- [ ] JS 打包体积在性能预算内
- [ ] 图片已优化（格式、尺寸、懒加载）
- [ ] 字体加载策略已配置（`font-display`、预加载、子集化）
- [ ] 关键 CSS 已内联或预加载

### 后端性能

- [ ] API P95 响应时间 < 500ms
- [ ] 无 N+1 查询问题（审计角色，实现 → 参见 [database](../database/SKILL.md) + [backend-patterns](../backend-patterns/SKILL.md)）
- [ ] 热点数据已缓存（合理的 TTL）
- [ ] 数据库索引已优化
- [ ] 响应压缩已启用（gzip/brotli）
- [ ] 分页策略正确（大数据集使用游标分页）

### 基础设施

- [ ] CDN 已配置并生效
- [ ] 连接池大小合理
- [ ] 耗时操作已异步化

### 流程保障

- [ ] 性能预算已定义并集成到 CI
- [ ] 基线数据已记录
- [ ] 优化前后数据对比已记录
- [ ] Lighthouse CI 或等效工具已集成

## Output Format

```markdown
# Performance Optimization: {优化目标}

## 基线数据
{优化前的性能指标}

## 瓶颈分析
{识别到的性能瓶颈及根因}

## 优化方案
{执行的优化措施}

## 验证结果
{优化后的性能指标，与基线对比}

## 审查清单
{审查结果清单，标记通过/未通过}
```
