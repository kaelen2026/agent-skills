---
name: web-vitals
version: 1.0.0
description: Core Web Vitals 与前端性能优化参考
last_updated: 2026-03-17
---

# Core Web Vitals 与前端性能

## Core Web Vitals 指标

Google 定义的三大核心指标，直接影响搜索排名和用户体验。

### LCP（Largest Contentful Paint）— 最大内容绘制

衡量页面主要内容的加载速度。

| 评级 | 阈值 |
|------|------|
| 良好 | ≤ 2.5s |
| 需要改进 | 2.5s ~ 4.0s |
| 较差 | > 4.0s |

### INP（Interaction to Next Paint）— 交互到下次绘制

衡量页面对用户交互的响应速度（已取代 FID）。

| 评级 | 阈值 |
|------|------|
| 良好 | ≤ 200ms |
| 需要改进 | 200ms ~ 500ms |
| 较差 | > 500ms |

### CLS（Cumulative Layout Shift）— 累积布局偏移

衡量页面视觉稳定性，即元素在加载过程中是否发生意外移动。

| 评级 | 阈值 |
|------|------|
| 良好 | ≤ 0.1 |
| 需要改进 | 0.1 ~ 0.25 |
| 较差 | > 0.25 |

## LCP 优化

LCP 通常由以下元素决定：大图片、视频封面、大段文字块、带背景图的元素。

### 图片优化

```html
<!-- 对 LCP 图片使用 priority/fetchpriority -->
<img
  src="/hero.webp"
  alt="Hero image"
  width="1200"
  height="600"
  fetchpriority="high"
  decoding="async"
/>

<!-- Next.js: priority 属性会自动设置 fetchpriority="high" 并禁用懒加载 -->
<Image
  src="/hero.webp"
  alt="Hero image"
  width={1200}
  height={600}
  priority
/>
```

### 字体加载优化

字体阻塞会延迟文本渲染，从而拖慢 LCP。

```html
<!-- 预加载关键字体 -->
<link
  rel="preload"
  href="/fonts/inter-var.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

```css
/* font-display: swap 确保文字立即可见 */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-display: swap;
}
```

### 关键 CSS

将首屏渲染所需的 CSS 内联到 HTML 中，避免阻塞渲染。

```html
<head>
  <!-- 关键 CSS 内联 -->
  <style>
    /* 首屏必要样式 */
    .hero { display: flex; min-height: 60vh; }
    .hero-title { font-size: 3rem; font-weight: 700; }
  </style>
  <!-- 非关键 CSS 异步加载 -->
  <link rel="preload" href="/styles/main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
</head>
```

### SSR / SSG

服务端渲染（SSR）和静态生成（SSG）可以显著缩短 LCP，因为浏览器收到的是完整 HTML 而非空壳。

```typescript
// Next.js App Router: 默认即为服务端组件（SSR）
// 静态页面可使用 generateStaticParams 实现 SSG
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

## INP 优化

INP 衡量的是从用户交互到浏览器完成下一帧绘制的总时间。优化核心是减少主线程阻塞。

### 减少 JS 执行时间

```typescript
// 不良：一个长任务阻塞主线程
function processLargeList(items: Item[]) {
  items.forEach(item => heavyComputation(item)); // 可能阻塞 200ms+
}

// 良好：拆分为多个小任务，让出主线程
async function processLargeList(items: Item[]) {
  const CHUNK_SIZE = 50;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    chunk.forEach(item => heavyComputation(item));
    // 让出主线程，允许浏览器处理用户交互
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

### Web Workers

将计算密集型任务移到 Web Worker，完全避免阻塞主线程。

```typescript
// worker.ts
self.addEventListener('message', (event) => {
  const result = heavyComputation(event.data);
  self.postMessage(result);
});

// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url));
worker.postMessage(data);
worker.addEventListener('message', (event) => {
  updateUI(event.data);
});
```

### requestIdleCallback

在浏览器空闲时执行非关键任务。

```typescript
function scheduleNonCriticalWork(tasks: Array<() => void>) {
  function processNext(deadline: IdleDeadline) {
    while (deadline.timeRemaining() > 0 && tasks.length > 0) {
      const task = tasks.shift();
      task?.();
    }
    if (tasks.length > 0) {
      requestIdleCallback(processNext);
    }
  }
  requestIdleCallback(processNext);
}

// 使用示例：非关键分析和日志
scheduleNonCriticalWork([
  () => sendAnalytics('page_view'),
  () => prefetchNextPage(),
  () => updateLocalCache(),
]);
```

## CLS 优化

布局偏移的常见原因：图片/视频无尺寸、动态注入内容、Web 字体加载。

### 显式声明尺寸

```html
<!-- 始终为图片和视频声明 width/height -->
<img src="/photo.webp" alt="Photo" width="800" height="600" />
<video width="1280" height="720" poster="/poster.webp">
  <source src="/video.mp4" type="video/mp4" />
</video>

<!-- 或使用 aspect-ratio -->
<style>
  .responsive-image {
    width: 100%;
    aspect-ratio: 4 / 3;
    object-fit: cover;
  }
</style>
```

### font-display: swap 与后备字体匹配

```css
/* 使用 swap 并配置尺寸接近的后备字体，减少换字时的偏移 */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  /* 使用 size-adjust 让后备字体与自定义字体尺寸接近 */
}

body {
  font-family: 'CustomFont', -apple-system, BlinkMacSystemFont, sans-serif;
}
```

### 占位符策略

```tsx
// 为异步加载的内容预留空间
function ContentPlaceholder() {
  return (
    <div style={{ minHeight: '200px', background: '#f0f0f0' }}>
      {/* 骨架屏 */}
    </div>
  );
}

// 使用 Suspense + 骨架屏
function Page() {
  return (
    <Suspense fallback={<ContentPlaceholder />}>
      <AsyncContent />
    </Suspense>
  );
}
```

## 打包体积分析

打包体积直接影响加载速度。必须定期分析并控制。

### Next.js Bundle Analyzer

```bash
npm install @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // 其他 Next.js 配置
});
```

```bash
# 运行分析
ANALYZE=true next build
```

### source-map-explorer

```bash
# 通用项目的打包分析
npx source-map-explorer dist/main.js
npx source-map-explorer dist/main.js --html result.html
```

### 常见优化手段

```typescript
// 动态导入：按需加载大型组件
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,
});

// Tree shaking：具名导入代替整包导入
// 不良
import _ from 'lodash';
// 良好
import { debounce } from 'lodash-es';
```

## 图片优化

### 格式选择

| 格式 | 适用场景 | 压缩率 |
|------|---------|--------|
| WebP | 通用替代 JPEG/PNG | 比 JPEG 小 25-35% |
| AVIF | 现代浏览器优先选择 | 比 WebP 再小 20% |
| SVG | 图标、Logo、简单图形 | 矢量无损 |
| PNG | 需要透明通道的图片 | 无损 |

### Next.js Image 组件

```tsx
import Image from 'next/image';

// 自动优化：格式转换、尺寸适配、懒加载
<Image
  src="/product.jpg"
  alt="Product"
  width={600}
  height={400}
  sizes="(max-width: 768px) 100vw, 50vw"
  placeholder="blur"
  blurDataURL={blurHash}
/>
```

### 响应式图片

```html
<picture>
  <source srcset="/hero.avif" type="image/avif" />
  <source srcset="/hero.webp" type="image/webp" />
  <img
    src="/hero.jpg"
    alt="Hero"
    width="1200"
    height="600"
    loading="lazy"
    decoding="async"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 80vw, 1200px"
  />
</picture>
```

### 懒加载

```html
<!-- 原生懒加载：首屏以下的图片 -->
<img src="/below-fold.webp" alt="Content" loading="lazy" />

<!-- 首屏图片不应使用懒加载 -->
<img src="/hero.webp" alt="Hero" fetchpriority="high" />
```

## 字体优化

### font-display 策略

| 值 | 行为 | 适用场景 |
|----|------|---------|
| `swap` | 立即显示后备字体，加载完后替换 | 正文字体（推荐） |
| `optional` | 若未在短时间内加载完则放弃使用 | 非关键装饰字体 |
| `fallback` | 短暂空白后使用后备字体 | 标题字体 |

### 预加载与子集化

```html
<!-- 预加载关键字体文件 -->
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin />
```

```bash
# 字体子集化：只保留需要的字符
# 安装 glyphhanger
npm install -g glyphhanger

# 生成包含页面实际使用字符的子集
glyphhanger https://example.com --subset=fonts/full.woff2 --formats=woff2
```

### Next.js 字体优化

```typescript
// next/font 自动处理字体优化
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  // 自动内联 @font-face，零布局偏移
});
```

## Lighthouse CI 集成

将性能检测集成到 CI 流程，防止性能回归。

### 配置文件

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/products'],
      numberOfRuns: 3,
      startServerCommand: 'npm run start',
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-byte-weight': ['warning', { maxNumericValue: 500000 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

### GitHub Actions 集成

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          configPath: './lighthouserc.js'
          uploadArtifacts: true
```
