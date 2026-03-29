---
title: "用腾讯云数据万象替代 Next.js 图片优化"
date: 2026-03-29T21:00:00+08:00
draft: false
tags: ['next.js', 'performance', 'tencent-cloud', 'cos', 'image-optimization']
categories: ['tech']
description: "Next.js 内置图片优化在 2 核服务器上并发处理 80+ 张图片时 CPU 打满，响应时间飙升到 10 秒。通过自定义 Image Loader 将图片处理卸载到腾讯云 COS 数据万象，实现零服务端 CPU 消耗的图片优化"
---

在完成[迁移到腾讯云](/posts/vercel-to-tencent-cloud-migration/)和 [SSL/协议优化](/posts/photo-blog-performance-optimization/)之后，照片博客的整体性能已经很不错了。但有一个场景始终令人头疼：**CDN 缓存失效后的首次加载极慢**。深入排查后发现，问题出在 Next.js 的图片优化机制上。

<!--more-->

## 发现问题

浏览器 DevTools 里的数字触目惊心：

| 请求类型 | 数量 | 耗时 |
|----------|------|------|
| `/_next/image?url=...` 图片优化 | 84 个并发 | **9.7 - 10.7 秒** |
| RSC 数据请求 (`?_rsc=...`) | 4 个 | **6.4 - 6.5 秒** |

图片请求 10 秒才返回，RSC 也被拖慢到 6 秒——这意味着用户在缓存失效时需要等待近 **10 秒**才能看到页面上的照片。

## 定位根因

先看 Nginx 的上游响应时间（`urt`）：

```
GET /_next/image?url=...photo-T9in9OD6OCHkrZQ1.jpg&w=640&q=75  rt=10.728 urt=10.672
GET /_next/image?url=...photo-vGaPxkPfvs6nS3y9.jpg&w=640&q=75  rt=10.727 urt=10.692
```

`urt`（上游响应时间）和 `rt`（总请求时间）几乎相等，说明 **Nginx 没有排队**，时间全部花在等 Next.js 处理。

然后在服务器上做对比测试：

```bash
# 单个请求（缓存命中）
curl http://127.0.0.1:3000/_next/image?url=...&w=640&q=75
# → 0.01 秒

# 单个请求（首次处理）
curl http://127.0.0.1:3000/_next/image?url=...&w=640&q=75
# → 0.31 秒

# 并发 20 个不同图片
time (for p in ${photos[@]}; do curl ... & done; wait)
# → 最慢 3.2 秒

# 并发 84 个不同图片
time (for p in $photos; do curl ... & done; wait)
# → 最慢 10.5 秒 ← 和 EdgeOne 过来的 10.7 秒吻合
```

**本地并发 84 个 = 10.5 秒，经 EdgeOne 并发 84 个 = 10.7 秒**。差距仅 0.2 秒，问题 100% 在 Next.js 图片优化。

## Next.js Image Optimization 的工作原理

Next.js 的 `<Image>` 组件会把所有远程图片请求代理到 `/_next/image` 端点：

```
浏览器请求 <Image src="https://assets.tanteng.space/photo-xxx.jpg" width={640} />
    │
    ▼
Next.js 默认 loader 转换为：
    /_next/image?url=https%3A...photo-xxx.jpg&w=640&q=75
    │
    ▼
Next.js 服务端：
    1. 从 COS 下载原图（~200KB-2MB）
    2. sharp 解码 → 缩放到 640px → 转 webp → 压缩
    3. 返回优化后的图片（~30-150KB）
    4. 缓存到本地磁盘
```

**单个请求 0.3 秒看起来没问题，但问题在于并发**。我的照片页面一次加载 20-80 张图片缩略图，全部打到 `/_next/image`。服务器只有 2 核 CPU，sharp 的图片处理是 CPU 密集操作：

```
top - 20:54:26 up 9 days
%Cpu(s): 95.5 us,  4.5 sy,  0.0 ni,  0.0 id
```

CPU 满载，所有请求排队等待，最后一个请求要等前面 83 个处理完——于是 10 秒。更糟的是 RSC 请求本身只需 3-16ms，但因为 CPU 被 sharp 抢占，也被拖慢到 1.5-2 秒。

## 方案：COS 数据万象

腾讯云 COS 自带的**数据万象（Cloud Infinite, CI）**功能，可以在图片 URL 上追加参数实现实时图片处理，处理在云端完成，不消耗服务器资源：

```
# 原图
https://assets.tanteng.space/photo-xxx.jpg

# 加上数据万象参数 → 自动缩放 + 转 webp + 渐进加载
https://assets.tanteng.space/photo-xxx.jpg?imageMogr2/thumbnail/640x/format/webp/quality/75/interlace/1
```

测试速度：

| 场景 | 耗时 |
|------|------|
| 首次处理（云端） | **1-2 秒** |
| CDN 缓存命中后 | **0.14 秒** |

关键优势：处理发生在腾讯云的图片处理集群上，不管并发 10 个还是 1000 个，我的服务器 CPU 都是 **零负载**。

## 实现：Custom Image Loader

Next.js 支持通过 [Custom Loader](https://nextjs.org/docs/app/api-reference/next-config-js/images#loaderfile) 自定义图片 URL 生成逻辑。核心思路是：**COS 图片走数据万象，非 COS 图片（如 QR 码）仍走 `/_next/image`**。

### 1. 创建 Loader 文件

```typescript
// src/platforms/image-loader.ts
const COS_CUSTOM_DOMAIN =
  process.env.NEXT_PUBLIC_TENCENT_COS_CUSTOM_DOMAIN || '';

interface ImageLoaderParams {
  src: string
  width: number
  quality?: number
}

export default function imageLoader({
  src,
  width,
  quality,
}: ImageLoaderParams): string {
  const q = quality || 75;

  // COS 图片 → 数据万象处理
  if (COS_CUSTOM_DOMAIN && src.includes(COS_CUSTOM_DOMAIN)) {
    return `${src}?imageMogr2/thumbnail/${width}x/format/webp/quality/${q}/interlace/1`;
  }

  // 非 COS 图片 → 走默认的 Next.js 图片优化
  return `/_next/image?url=${encodeURIComponent(src)}&w=${width}&q=${q}`;
}
```

### 2. 配置 next.config.ts

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    loader: 'custom',
    loaderFile: './src/platforms/image-loader.ts',
    // ... 其他配置保持不变
  },
};
```

### 3. 更新服务端图片请求

项目中除了 `<Image>` 组件，还有一些服务端代码（RSS feed 生成、blur 数据等）会手动构造 `/_next/image` URL。需要同步更新：

```typescript
// src/platforms/next-image.ts
export const getNextImageUrlForRequest = ({ imageUrl, size, quality, ... }) => {
  // COS 图片直接用数据万象
  if (COS_CUSTOM_DOMAIN && imageUrl.includes(COS_CUSTOM_DOMAIN)) {
    return `${imageUrl}?imageMogr2/thumbnail/${size}x/format/webp/quality/${quality}/interlace/1`;
  }

  // 非 COS 图片走原来的 /_next/image
  const url = new URL(`${baseUrl}/_next/image`);
  url.searchParams.append('url', imageUrl);
  url.searchParams.append('w', size.toString());
  url.searchParams.append('q', quality.toString());
  return url.toString();
};
```

**改动总共三个文件**，非常轻量。

## 效果

部署后立即验证：

```bash
# 检查页面中的图片 URL
curl -s https://photos.tanteng.space/ | grep -o 'imageMogr2[^"]*' | head -3
# imageMogr2/thumbnail/640x/format/webp/quality/75/interlace/1 1x
# imageMogr2/thumbnail/640x/format/webp/quality/75/interlace/1 1x
# imageMogr2/thumbnail/640x/format/webp/quality/75/interlace/1 1x

# 检查是否还有 _next/image 请求
curl -s https://photos.tanteng.space/ | grep '_next/image'
# （空 — 没有了）

# 检查 Nginx 日志
tail -20 /var/log/nginx/photo-blog.access.log | grep '_next/image' | wc -l
# 0
```

### 性能对比

| 指标 | 优化前 (Next.js sharp) | 优化后 (COS 数据万象) |
|------|----------------------|---------------------|
| 图片处理位置 | 服务器 CPU (2核) | 腾讯云端 |
| 84 张图片并发 | **10.5 秒** | **0.14 秒**（CDN 命中） |
| 服务器 CPU 负载 | **95.5%** | **接近 0** |
| RSC 响应时间 | 1.5-2 秒（被 CPU 争抢拖慢） | **3-16ms**（正常） |
| `/_next/image` 请求数 | 84 个 | **0 个** |

### 架构变化

```
优化前：
浏览器 → EdgeOne → Nginx → Next.js ─→ 从 COS 下载原图 → sharp 处理 → 返回
                                       (CPU 密集，2核被打满)

优化后：
浏览器 → COS CDN (assets.tanteng.space) → 数据万象云端处理 → 返回
                                          (服务器完全不参与)
```

图片请求不再经过我的服务器，直接从 COS CDN 节点返回。服务器只需要处理 HTML、RSC 和 JS 静态资源。

## 踩坑：DNG 格式的方向问题

切换到数据万象后发现有一张 iPhone 上传的照片方向不对。排查发现这张照片虽然扩展名是 `.jpeg`，但实际是 DNG（TIFF 容器）格式——iPhone 的 ProRAW 拍摄会生成 DNG 文件，上传流程中被保存为 `.jpeg` 扩展名但内部格式未变。

数据万象的 `auto-orient` 对 TIFF/DNG 格式的 EXIF orientation 处理存在兼容性问题，无法正确旋转。解决方式是用 sharp 手动旋转像素数据并转为真正的 JPEG 后重新上传。这属于个别特殊格式的边界情况，常规 JPEG 照片不受影响。

## 小结

这次优化的核心思路很简单：**别在自己的服务器上做云服务商更擅长的事情**。

Next.js 的图片优化对 Vercel 这种弹性计算平台来说没问题——请求多了自动扩容。但在固定规格的 2 核服务器上，80+ 张图片并发处理就是灾难。腾讯云 COS 数据万象本身就是为海量图片处理设计的，处理能力几乎无限，又和 COS CDN 天然集成，只需要在 URL 上加几个参数就能用。

整个改造只改了三个文件，核心逻辑不到 20 行代码。有时候最好的优化不是写更多代码，而是把工作交给对的人（或者说，对的服务）。
