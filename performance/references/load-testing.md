---
name: load-testing
version: 1.0.0
description: 负载测试与性能预算参考
last_updated: 2026-03-17
---

# 负载测试与性能预算

## 工具概览

| 工具 | 语言 | 特点 | 适用场景 |
|------|------|------|---------|
| k6 | JavaScript | 开发者友好、CI 集成好、支持阈值断言 | 通用负载测试（推荐） |
| Artillery | YAML/JS | 配置驱动、场景编排能力强 | 复杂用户流程模拟 |
| autocannon | JavaScript | 极简、高性能、低开销 | 快速 HTTP 基准测试 |

## k6 脚本示例

### 基础脚本

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// 自定义指标
const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration', true);

// 测试配置
export const options = {
  stages: [
    { duration: '1m', target: 10 },   // 预热：1 分钟内爬升到 10 VU
    { duration: '3m', target: 50 },   // 加压：3 分钟内爬升到 50 VU
    { duration: '5m', target: 50 },   // 稳态：维持 50 VU 持续 5 分钟
    { duration: '1m', target: 0 },    // 冷却：1 分钟内降到 0 VU
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // P95 < 500ms, P99 < 1s
    errors: ['rate<0.01'],                           // 错误率 < 1%
    http_req_failed: ['rate<0.01'],                  // HTTP 失败率 < 1%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  // 模拟用户行为
  const responses = http.batch([
    ['GET', `${BASE_URL}/api/products`, null, { tags: { name: 'list_products' } }],
    ['GET', `${BASE_URL}/api/categories`, null, { tags: { name: 'list_categories' } }],
  ]);

  responses.forEach((res) => {
    check(res, {
      '状态码为 200': (r) => r.status === 200,
      '响应时间 < 500ms': (r) => r.timings.duration < 500,
      '响应体非空': (r) => r.body && r.body.length > 0,
    });

    errorRate.add(res.status !== 200);
    apiDuration.add(res.timings.duration);
  });

  // 模拟单个产品详情查询
  const productRes = http.get(`${BASE_URL}/api/products/1`, {
    tags: { name: 'get_product' },
  });

  check(productRes, {
    '产品查询成功': (r) => r.status === 200,
    '包含产品名称': (r) => JSON.parse(r.body).name !== undefined,
  });

  sleep(1); // 模拟用户思考时间
}
```

### 带认证的脚本

```javascript
import http from 'k6/http';
import { check } from 'k6';

export function setup() {
  // setup 阶段获取认证令牌
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: __ENV.TEST_PASSWORD,
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  const token = JSON.parse(loginRes.body).token;
  return { token };
}

export default function (data) {
  const headers = {
    'Authorization': `Bearer ${data.token}`,
    'Content-Type': 'application/json',
  };

  const res = http.get(`${BASE_URL}/api/user/profile`, { headers });

  check(res, {
    '认证请求成功': (r) => r.status === 200,
  });
}
```

## 负载测试类型

### 冒烟测试（Smoke Test）

验证系统在最小负载下能否正常工作。通常在每次部署后运行。

```javascript
export const options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};
```

**目的**：确认基本功能正常，无明显错误。

### 负载测试（Load Test）

模拟预期的正常流量，验证系统在日常负载下的表现。

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // 爬升到预期负载
    { duration: '30m', target: 100 },   // 维持预期负载
    { duration: '5m', target: 0 },      // 冷却
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};
```

**目的**：确认系统在正常负载下满足 SLA。

### 压力测试（Stress Test）

逐步增加负载超过正常水平，找到系统的极限和断裂点。

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },    // 正常负载
    { duration: '5m', target: 200 },    // 超出正常负载
    { duration: '5m', target: 400 },    // 继续加压
    { duration: '5m', target: 600 },    // 极限测试
    { duration: '2m', target: 0 },      // 恢复
  ],
};
```

**目的**：找到系统极限，验证优雅降级能力。

### 尖峰测试（Spike Test）

模拟突发的流量激增（如秒杀、热点事件）。

```javascript
export const options = {
  stages: [
    { duration: '1m', target: 10 },     // 正常流量
    { duration: '10s', target: 500 },   // 突发激增
    { duration: '3m', target: 500 },    // 维持高峰
    { duration: '10s', target: 10 },    // 突然回落
    { duration: '3m', target: 10 },     // 恢复期
  ],
};
```

**目的**：验证自动扩缩容能力和突发负载下的稳定性。

### 浸泡测试（Soak Test）

长时间运行稳定负载，检测内存泄漏、连接泄漏等随时间累积的问题。

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },     // 爬升
    { duration: '4h', target: 100 },     // 长时间维持
    { duration: '5m', target: 0 },       // 冷却
  ],
};
```

**目的**：检测内存泄漏、连接泄漏、日志膨胀等长期运行问题。

## 性能预算

性能预算是团队约定的性能指标上限，超出即为性能回归。

### 打包体积预算

| 资源 | 预算 | 说明 |
|------|------|------|
| 首屏 JS（压缩后） | < 150 KB | 包含框架和首屏逻辑 |
| 首屏 CSS（压缩后） | < 50 KB | 关键样式 |
| 单个路由 JS | < 50 KB | 代码分割后每个路由的体积 |
| 图片总量 | < 500 KB | 首屏所有图片 |
| 总传输量 | < 1 MB | 首次加载的所有资源 |

### API 响应时间预算

| 接口类型 | P50 | P95 | P99 |
|----------|-----|-----|-----|
| 简单查询（列表/详情） | < 50ms | < 200ms | < 500ms |
| 复杂查询（聚合/报表） | < 200ms | < 1s | < 3s |
| 写入操作 | < 100ms | < 500ms | < 1s |
| 搜索 | < 100ms | < 500ms | < 1s |

### 页面加载时间预算

| 指标 | 预算 | 说明 |
|------|------|------|
| LCP | < 2.5s | Core Web Vitals 良好阈值 |
| INP | < 200ms | Core Web Vitals 良好阈值 |
| CLS | < 0.1 | Core Web Vitals 良好阈值 |
| TTFB | < 600ms | 服务器首字节时间 |
| FCP | < 1.8s | 首次内容绘制 |
| TTI | < 3.5s | 可交互时间 |

### 在构建中强制执行预算

```javascript
// next.config.js — 实验性体积限制
module.exports = {
  experimental: {
    outputFileTracing: true,
  },
};

// webpack 性能提示
module.exports = {
  performance: {
    maxAssetSize: 150 * 1024,      // 单文件 150KB
    maxEntrypointSize: 300 * 1024,  // 入口 300KB
    hints: 'error',                 // 超出预算则构建失败
  },
};
```

```json
// bundlesize 配置（package.json）
{
  "bundlesize": [
    { "path": "dist/main.*.js", "maxSize": "150 kB" },
    { "path": "dist/vendor.*.js", "maxSize": "200 kB" },
    { "path": "dist/*.css", "maxSize": "50 kB" }
  ]
}
```

## 负载测试期间的监控

负载测试不仅关注 k6 的输出，还需要同时监控服务端指标。

### 关键监控指标

| 类别 | 指标 | 告警阈值 | 工具 |
|------|------|---------|------|
| CPU | 使用率 | > 80% 持续 5 分钟 | Grafana / CloudWatch |
| 内存 | 使用量和趋势 | > 85% 或持续增长 | Grafana / CloudWatch |
| 错误率 | HTTP 5xx 比率 | > 1% | 应用 APM |
| 延迟 P50 | 中位响应时间 | 超出预算 | k6 / APM |
| 延迟 P95 | 长尾响应时间 | 超出预算 | k6 / APM |
| 延迟 P99 | 极端情况 | 超出预算 | k6 / APM |
| 数据库连接 | 活跃连接数 | 接近池上限 | 数据库监控 |
| 队列深度 | 等待处理的任务数 | 持续增长 | 队列监控 |

### k6 + Grafana 实时监控

```bash
# 将 k6 指标输出到 InfluxDB，通过 Grafana 可视化
k6 run --out influxdb=http://localhost:8086/k6 script.js
```

## 结果分析模板

每次负载测试完成后，使用以下模板记录结果：

```markdown
# 负载测试报告

## 基本信息
- **测试日期**: YYYY-MM-DD
- **测试类型**: 冒烟 / 负载 / 压力 / 尖峰 / 浸泡
- **测试环境**: staging / production
- **测试时长**: Xm
- **最大虚拟用户数**: X VU

## 测试结果摘要

| 指标 | 预算 | 实际值 | 结果 |
|------|------|--------|------|
| P50 响应时间 | < 100ms | XXms | 通过/未通过 |
| P95 响应时间 | < 500ms | XXms | 通过/未通过 |
| P99 响应时间 | < 1s | XXms | 通过/未通过 |
| 错误率 | < 1% | X.XX% | 通过/未通过 |
| 吞吐量 | > X req/s | XX req/s | 通过/未通过 |

## 服务端指标

| 指标 | 峰值 | 均值 | 备注 |
|------|------|------|------|
| CPU 使用率 | XX% | XX% | |
| 内存使用 | XX MB | XX MB | 是否有增长趋势 |
| 数据库连接数 | XX | XX | 是否接近上限 |
| 错误日志 | XX 条 | — | 列举主要错误类型 |

## 发现的问题

1. **问题描述** — 严重程度（高/中/低）— 建议解决方案
2. ...

## 结论与建议

- 系统能否承受预期负载？
- 瓶颈在哪里？
- 下一步优化建议
```

## CI 集成（性能回归检测）

将负载测试集成到 CI 流程中，自动检测性能回归。

### GitHub Actions 集成

```yaml
# .github/workflows/performance.yml
name: Performance Test
on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # 每周一早上 6 点

jobs:
  smoke-test:
    runs-on: ubuntu-latest
    services:
      app:
        image: ${{ github.repository }}:${{ github.sha }}
        ports:
          - 3000:3000
    steps:
      - uses: actions/checkout@v4

      - name: 安装 k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6

      - name: 运行冒烟测试
        run: k6 run tests/performance/smoke.js
        env:
          BASE_URL: http://localhost:3000

      - name: 运行负载测试（仅 main 分支合并后）
        if: github.event_name == 'schedule'
        run: k6 run tests/performance/load.js
        env:
          BASE_URL: http://localhost:3000

      - name: 上传测试结果
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results/
```

### 打包体积回归检测

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size Check
on: [pull_request]

jobs:
  bundle-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build

      - name: 检查打包体积
        uses: siddharthkp/bundlesize2@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 性能预算门禁

```bash
#!/bin/bash
# scripts/check-performance-budget.sh
# 在 CI 中作为门禁使用

set -e

echo "=== 性能预算检查 ==="

# 检查 JS 打包体积
JS_SIZE=$(stat -f%z dist/main.*.js 2>/dev/null || stat -c%s dist/main.*.js)
JS_BUDGET=$((150 * 1024))

if [ "$JS_SIZE" -gt "$JS_BUDGET" ]; then
  echo "JS 体积超出预算: ${JS_SIZE} bytes > ${JS_BUDGET} bytes"
  exit 1
fi
echo "JS 体积检查通过: ${JS_SIZE} bytes"

# 运行 k6 冒烟测试（阈值失败会使 k6 返回非零退出码）
k6 run --quiet tests/performance/smoke.js

echo "=== 所有性能预算检查通过 ==="
```
