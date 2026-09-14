---
title: "将 Next.js 照片博客从 Vercel 迁移到腾讯云 Lighthouse 并持续优化"
date: 2026-03-22T08:00:00+08:00
draft: false
url: /2026/03/vercel-to-tencent-cloud-migration/
tags: ['next.js', 'tencent-cloud', 'migration', 'performance-optimization', 'nginx', 'http', 'ai-coding']
categories: ['tech']
description: "将基于 exif-photo-blog 的 Next.js 照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse 自托管，并完成图片优化卸载与 HTTP/3 升级的完整实践"
featured_image: 'https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/6f7afe43432ec45cb352d6196738e34e.png'
---

本文记录将基于 [exif-photo-blog](https://github.com/sambecker/exif-photo-blog) 的照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse 自托管的过程，以及迁移后围绕图片处理和回源协议做的两轮关键优化。整个过程借助 WorkBuddy（Claude Opus 4.6）和 OpenClaw（MiniMax-2.5）完成代码改造、脚本编写和问题排查。

<!--more-->

## 一、为什么迁移

我的站点 [photos.tanteng.space](https://photos.tanteng.space) 最初采用标准 Vercel 部署方案（Vercel Serverless + Cloudflare R2 + Neon PostgreSQL + Vercel Edge Network）。一键部署确实方便，但几个问题逐渐浮现：

1. **跨洋延迟**：Neon PostgreSQL 部署在 us-east-1，每次查询跨太平洋往返，构建和加载都很慢
2. **访问体验**：Vercel Edge Network 在国内没有节点，访问速度受限
3. **成本考量**：Vercel、Cloudflare R2、Neon 各自计费，免费额度有限
4. **可控性**：基础设施散落在多个平台，排障和调优都不方便

在确定自托管方案之前，也评估了腾讯云的几个 Serverless 产品：

- **CloudBase**：主要面向静态网站，这个相册服务包含 ISR、SSR 等特性，CloudBase 无法很好支持
- **EdgeOne Pages**：部分 Next.js 特性支持尚不完善

最终决定整体迁移到腾讯云 Lighthouse，把计算、存储、数据库全部集中到广州区域的一台 Lighthouse 实例上。

## 二、迁移核心流程

迁移围绕三个环节展开：存储、数据库、部署自动化。

### 2.1 存储：R2 → 腾讯云 COS

**为什么选 COS**：腾讯云 COS 兼容 S3 API，可以直接复用 `@aws-sdk/client-s3`，无需引入额外 SDK。服务器就在腾讯云上，内网传输免费且高速。

**数据搬迁**：在服务器上安装 rclone，配置好 R2（源）和 COS（目标），走 COS 内网 endpoint 传输：

```bash
rclone copy r2:pics cos:photos-1303209934 \
  --transfers 16 --checkers 32
```

362 个文件、476.9 MiB，走内网只用了 **1 分 22 秒**（平均 5.37 MB/s）。对比之前在 Mac 上通过公网传输，速度提升了约 370 倍。同时还有 19 张照片存在 Vercel Blob 上，一并迁移过来。

**代码改造：新增 COS 存储适配层**。exif-photo-blog 原生支持 Vercel Blob、Cloudflare R2、AWS S3、MinIO 四种存储后端，采用策略模式通过 `StorageType` 联合类型切换。需要新增一个 `tencent-cos` 类型。

这部分在 WorkBuddy 中完成：让它阅读整个存储抽象层代码，理解接口定义和各后端的实现方式，然后生成 COS 适配代码。Opus 4.6 对这种「理解现有架构 → 按相同模式扩展」的任务非常拿手，生成的代码风格和现有后端保持一致。关键点是 COS 的 S3 兼容 endpoint 格式为 `cos.<region>.myqcloud.com`，其他 put/copy/list/delete/presigned-url 操作与 AWS S3 几乎一致。

**数据库 URL 批量更新**。照片 URL 存在数据库里，需要从旧域名批量替换为新域名：

```sql
UPDATE photos
SET url = REPLACE(url, 'https://static.tanteng.space/', 'https://assets.tanteng.space/')
WHERE url LIKE '%static.tanteng.space%';
-- 404 rows affected
```

**补生成图片优化版本**。exif-photo-blog 在上传照片时会生成 `-sm`（小图）、`-md`（中图）、`-lg`（大图）三个优化版本用于响应式加载。但 R2 时期上传的老照片没有这些版本，构建时会 404 报错。

这个问题我在 WorkBuddy 中描述了需求：「COS 上有 400 多张原始照片，需要为每张生成 -sm、-md、-lg 三个优化版本，用 sharp 处理，上传回 COS」。WorkBuddy 直接生成了一个完整的 Node.js 批处理脚本——从 COS 拉取原图列表、用 sharp 按不同尺寸压缩、再上传回 COS，一轮对话搞定。手写这个脚本大概要半小时，AI 生成后微调几处参数就能跑。

### 2.2 数据库：Neon → 本地 PostgreSQL

最初考虑购买腾讯云 PostgreSQL 实例（PostgreSQL 14+），但评估后发现：

| 方案 | 月成本 | 延迟 | 备注 |
|------|--------|------|------|
| 腾讯云 PostgreSQL | ~¥200/月 | < 5ms | 需要额外配置内网连接 |
| 本地 PostgreSQL | 0（复用现有 Lighthouse）| < 1ms | 同一台实例，无需额外费用 |

照片数量只有 400+ 张，数据量很小，单独购买云数据库性价比不高。直接在 Lighthouse 实例上本地部署 PostgreSQL，从跨洋 200ms+ 降到本地 < 1ms。

**安装与数据导入**：

```bash
# 从 Neon 导出
pg_dump "postgres://user:pass@ep-xxx.us-east-1.aws.neon.tech/verceldb" > dump.sql

# 导入本地
psql -U tanteng -d verceldb < dump.sql
```

**后续调优**：

- **连接池参数**：原配置是为跨区域 Neon 设计的，超时很长、连接数很大。迁移到本地后收紧参数，同时关闭 SSL
- **自动备份**：配置 crontab 每天凌晨 3 点自动备份数据库，保留最近 30 份

### 2.3 部署：零停机部署脚本

告别 Vercel 一键部署后，需要自己搞定 CI/CD。这部分同样在 WorkBuddy 中完成——描述清楚「独立构建、原子替换、自动清 CDN 缓存」的需求，它生成了一套完整的部署脚本。核心设计：

- **独立构建目录**：在独立目录中构建，线上代码不受影响
- **原子替换**：`mv` 操作是文件系统原子操作，不存在中间状态
- **自动清缓存**：部署后通过腾讯云 API V3 调用 EdgeOne `purge_host` 清除全域缓存

## 三、迁移后的两项关键优化

迁移完成后功能正常，但有两个性能瓶颈需要在生产环境再解决：图片处理 CPU 占用过高、回源协议存在排队。

### 3.1 图片优化卸载到 COS 数据万象

**发现问题**。浏览器 DevTools 里的数字触目惊心：

| 请求类型 | 数量 | 耗时 |
|----------|------|------|
| `/_next/image?url=...` 图片优化 | 84 个并发 | **9.7 - 10.7 秒** |
| RSC 数据请求 (`?_rsc=...`) | 4 个 | **6.4 - 6.5 秒** |

图片请求 10 秒才返回，RSC 也被拖慢到 6 秒——用户缓存失效后需要等近 **10 秒**才能看到照片。

**定位根因**。先看 Nginx 上游响应时间：

```
GET /_next/image?url=...photo-T9in9OD6OCHkrZQ1.jpg&w=640&q=75  rt=10.728 urt=10.672
GET /_next/image?url=...photo-vGaPxkPfvs6nS3y9.jpg&w=640&q=75  rt=10.727 urt=10.692
```

`urt`（上游响应时间）≈ `rt`（总请求时间），说明 Nginx 没排队，时间全花在等 Next.js。

在服务器上做对比测试：

```bash
# 单个请求（缓存命中）
curl http://127.0.0.1:3000/_next/image?...   # 0.01 秒
# 单个请求（首次处理）
curl http://127.0.0.1:3000/_next/image?...   # 0.31 秒
# 并发 84 个不同图片
time (for p in $photos; do curl ... & done; wait)  # 最慢 10.5 秒
```

**本地并发 84 个 = 10.5 秒，经 EdgeOne 并发 84 个 = 10.7 秒**。差距仅 0.2 秒，问题 100% 在 Next.js 图片优化。

**Next.js Image Optimization 的工作原理**。`<Image>` 组件会把所有远程图片请求代理到 `/_next/image` 端点，服务端用 sharp 解码、缩放、转 webp：

```
浏览器 → /_next/image?url=...&w=640&q=75
   → 服务端从 COS 下载原图（~200KB-2MB）
   → sharp 解码 → 缩放到 640px → 转 webp → 压缩
   → 返回优化后的图片（~30-150KB）→ 缓存到本地磁盘
```

单个请求 0.3 秒看起来没问题，但问题是并发。照片页面一次加载 20-80 张缩略图，全部打到 `/_next/image`。服务器只有 2 核 CPU，sharp 处理是 CPU 密集操作：

```
top - 20:54:26 up 9 days
%Cpu(s): 95.5 us,  4.5 sy,  0.0 ni,  0.0 id
```

CPU 满载，所有请求排队，最后一个请求要等前面 83 个处理完——于是 10 秒。RSC 请求本身只需 3-16ms，但被 CPU 抢占也拖到 1.5-2 秒。

**方案：COS 数据万象**。腾讯云 COS 自带的**数据万象（Cloud Infinite, CI）**功能，在图片 URL 上追加参数就能实时处理，处理在云端完成，不消耗服务器资源：

```
# 原图
https://assets.tanteng.space/photo-xxx.jpg

# 加数据万象参数 → 自动缩放 + 转 webp + 渐进加载
https://assets.tanteng.space/photo-xxx.jpg?imageMogr2/thumbnail/640x/format/webp/quality/75/interlace/1
```

测试速度：

| 场景 | 耗时 |
|------|------|
| 首次处理（云端） | **1-2 秒** |
| CDN 缓存命中后 | **0.14 秒** |

关键优势：处理发生在腾讯云图片处理集群上，并发 10 个还是 1000 个，我的服务器 CPU 都是 **零负载**。

**实现：Custom Image Loader**。Next.js 支持通过 [Custom Loader](https://nextjs.org/docs/app/api-reference/next-config-js/images#loaderfile) 自定义图片 URL 生成逻辑。核心思路：**COS 图片走数据万象，非 COS 图片（如 QR 码）仍走 `/_next/image`**。

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
  src, width, quality,
}: ImageLoaderParams): string {
  const q = quality || 75;

  if (COS_CUSTOM_DOMAIN && src.includes(COS_CUSTOM_DOMAIN)) {
    return `${src}?imageMogr2/thumbnail/${width}x/format/webp/quality/${q}/interlace/1`;
  }

  return `/_next/image?url=${encodeURIComponent(src)}&w=${width}&q=${q}`;
}
```

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    loader: 'custom',
    loaderFile: './src/platforms/image-loader.ts',
  },
};
```

项目里除 `<Image>` 组件外，还有一些服务端代码（RSS feed 生成、blur 数据等）会手动构造 `/_next/image` URL，需要同步更新。三个文件改动，核心逻辑不到 20 行。

**效果**：

| 指标 | 优化前 (Next.js sharp) | 优化后 (COS 数据万象) |
|------|----------------------|---------------------|
| 图片处理位置 | 服务器 CPU (2 核) | 腾讯云端 |
| 84 张图片并发 | **10.5 秒** | **0.14 秒**（CDN 命中） |
| 服务器 CPU 负载 | **95.5%** | **接近 0** |
| RSC 响应时间 | 1.5-2 秒（被 CPU 争抢拖慢） | **3-16ms**（正常） |
| `/_next/image` 请求数 | 84 个 | **0 个** |

**踩坑：图片方向问题**。切换到数据万象后，部分 iPhone 拍摄的竖拍照片显示方向不对。原因是 **`imageMogr2` 默认不会根据 EXIF orientation 自动旋转图片**——之前走 `/_next/image` 时 sharp 会处理 orientation，切换到数据万象后这步丢失了。修复方式是加 `auto-orient` 参数：

```
# 修复后
?imageMogr2/auto-orient/thumbnail/640x/format/webp/quality/75/interlace/1
```

全库扫描 579 张照片，8 张带 EXIF orientation 的 JPEG 通过 `auto-orient` 解决。另有 2 张 iPhone ProRAW (DNG) 照片被存为 `.jpeg` 扩展名但内部实际是 TIFF 格式，`auto-orient` 对 TIFF 无效，这 2 张需要手动用 sharp 旋转像素数据后重新上传。

### 3.2 回源协议 + 用户侧 HTTP/3

图片走数据万象后，源站 CPU 压力消失。但测试中发现部署清缓存后 JS 静态文件加载异常缓慢——20+ 个 JS chunk 全部 MISS 回源，部分等了 8 秒。查 Nginx 日志发现了典型的 HTTP/1.1 排队：

```
# 同一秒内 12 个 JS 文件回源
前 6 个：rt=0.002s ~ 0.007s    ← 立即处理
后 6 个：rt=1.55s ~ 1.77s      ← 排队等待
```

HTTP/1.1 每个连接只能串行处理请求，EdgeOne 到源站建了约 6 个并发连接，前 6 个秒回，后 6 个要等。

**回源协议的三次演进**：

第一阶段（迁移初始）：双重 SSL。EdgeOne 到 Nginx 都走 HTTPS，SSL 被终止两次。虽然都在腾讯云体系内，但第二次 TLS 握手主要是额外开销。实测 TTFB 增加了 85-98ms。

第二阶段：去掉 Nginx SSL，HTTP 明文回源。TTFB 降低 26%-73%，运维也简化（不用管证书续期）。看似完美，但埋了隐患——并发回源会排队。

第三阶段：恢复 HTTPS，启用 HTTP/2 回源。EdgeOne 走 **HTTP/2 回源**——多路复用可以在单个连接上并行处理所有请求，消除排队：

```
用户 ──H2/H3──→ EdgeOne(SSL终止) ──HTTPS+H2──→ Nginx(:443) ──→ Next.js
```

虽然加回了一层 TLS，但内网 TLS 握手只增加几毫秒，而 HTTP/2 多路复用在并发回源时能节省秒级延迟。**权衡之下，H2 多路复用的收益远大于 TLS 的微小开销。**

**为什么不用 h2c？** 第二阶段理论上可以用 **h2c**（HTTP/2 cleartext）在不加密的情况下获得 HTTP/2 多路复用。Nginx 配了 `http2 on` 也支持 h2c，但实测 EdgeOne 仍走 HTTP/1.1。原因是 h2c 握手方式不兼容：

| 方式 | 过程 | Nginx 支持 |
|------|------|-----------|
| **Upgrade** | 先发 HTTP/1.1，服务端返回 101 后切换 | ❌ |
| **Prior Knowledge** | 直接发 HTTP/2 帧 | ✅ |

EdgeOne 用 Upgrade 方式，Nginx 只支持 Prior Knowledge，两边对不上。所以 h2c 走不通，最终选了 HTTPS + H2 回源。

**用户侧 HTTP/3 (QUIC)**。HTTP/2 解决回源问题，用户侧还要进一步升级。HTTP/3 基于 QUIC（UDP），相比 HTTP/2 有几个关键优势：

**1. 连接建立更快**。传统 HTTPS 要 2-3 个 RTT（TCP 三次握手 + TLS），QUIC 把传输层和加密层合并，首次连接 1 RTT，重连 **0-RTT**。

**2. 消除队头阻塞**。HTTP/2 虽然应用层多路复用，但底层仍是单个 TCP 连接——某个 TCP 包丢失，整个连接的所有流都要等重传，这就是**队头阻塞**。QUIC 的多路复用在传输层实现，每个流独立，一个流丢包不影响其他流。移动网络（丢包率高）上提升尤为明显。

**3. 连接迁移**。手机在 Wi-Fi 和 4G 之间切换时，TCP 连接会断开重建。QUIC 用 Connection ID 标识连接（而非 IP+端口），网络切换时连接无缝迁移。

在 EdgeOne 控制台开启 HTTP/2 和 HTTP/3 后，响应头出现 `alt-svc: h3=":443"; ma=2592000`，浏览器记住后自动升级到 HTTP/3。

有一次不小心在 EdgeOne 把 HTTP/2 关掉了（本意是关 HTTP/2 *回源*），结果 20+ 个 JS chunk 加载从 1 秒飙到 7-8 秒——这就是多路复用的差距。

## 四、最终架构与效果

经过迁移 + 两轮优化后，最终架构：

```mermaid
flowchart TB
    subgraph Client["🌐 用户"]
        A[浏览器<br/>HTTP/2 or HTTP/3]:::cdn
    end

    subgraph EdgeOne["⚡ EdgeOne CDN + WAF"]
        B[SSL 终止 + 缓存<br/>HTTP/3 升级]:::step
    end

    subgraph Lighthouse["🖥️ 腾讯云 Lighthouse 广州"]
        direction TB
        C[Nginx :443<br/>HTTPS + H2 回源]:::build
        D[Next.js + PM2<br/>127.0.0.1:3000]:::build
    end

    subgraph COS["☁️ 腾讯云 COS（数据万象）"]
        E[assets.tanteng.space<br/>图片云端处理]:::deploy
    end

    subgraph DB["🗄️ 本地 PostgreSQL"]
        F[localhost:5432]:::purge
    end

    A -->|H2/H3| B
    B -->|HTTPS+H2| C
    C --> D
    D -->|内网| E
    D -->|<1ms| F

    classDef cdn fill:#667eea,stroke:#5a67d8,color:#fff,stroke-width:2px
    classDef step fill:#edf2f7,stroke:#a0aec0,color:#2d3748,stroke-width:1px
    classDef build fill:#f6ad55,stroke:#dd6b20,color:#fff,stroke-width:2px
    classDef deploy fill:#4299e1,stroke:#2b6cb0,color:#fff,stroke-width:2px
    classDef purge fill:#68d391,stroke:#38a169,color:#fff,stroke-width:2px

    style EdgeOne fill:#f0f0ff,stroke:#667eea,stroke-width:2px,color:#2d3748
    style Lighthouse fill:#fff8f0,stroke:#f6ad55,stroke-width:2px,color:#2d3748
    style COS fill:#f0f8ff,stroke:#4299e1,stroke-width:2px,color:#2d3748
    style DB fill:#f0fff0,stroke:#68d391,stroke-width:2px,color:#2d3748
```

**性能对比**（迁移前 vs 最终状态）：

| 指标 | Vercel 时代 | 最终状态 |
|------|--------|---------|
| 数据库查询延迟 | ~200ms（跨太平洋） | < 1ms（localhost） |
| 页面构建时间 | 3-5 分钟 | < 1 分钟 |
| 首屏加载（国内） | 2-4s | < 1s |
| 84 张图片并发（缓存失效） | 10+ 秒 | 0.14 秒（CDN 命中） |
| JS chunk 回源 | HTTP/1.1 排队 | HTTP/2 多路复用 |

**可靠性与安全加固**：

- 数据库每日自动备份，保留 30 天
- 零停机部署（独立构建 + 原子替换）
- 部署后自动清除 CDN 缓存
- PostgreSQL 仅监听 localhost，`pg_hba.conf` 删除 `0.0.0.0/0`，防火墙关闭 5432
- SSH 禁密码登录、仅密钥；fail2ban 连续 3 次失败封禁 24 小时
- `.env` 权限收紧到 600，仅 root 可读写

**源站 IP 防护**。源站 443 暴露在公网，需防止绕过 CDN 直接访问。Nginx 中用 `default_server` 拦截：

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate /etc/letsencrypt/live/photos.tanteng.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/photos.tanteng.space/privkey.pem;

    return 444;
}
```

用 IP 直接访问会被 Nginx 断开连接（444），只有带正确 `Host` 头的 EdgeOne 回源请求才能命中站点配置。

## 五、踩坑总结

1. **COS S3 兼容 endpoint**：格式是 `cos.<region>.myqcloud.com`，不是 `s3` 开头
2. **老照片缺优化版本**：从 R2 迁过来的照片需要补生成 `-sm`/`-md`/`-lg` 版本
3. **EdgeOne 与 Next.js RSC 缓存冲突**：EdgeOne 没正确处理 `Vary: RSC` 响应头，导致首页偶发显示乱码，需在规则引擎中将 RSC 请求头加入自定义 Cache Key
4. **fonts.googleapis.cn 不靠谱**：应该用 `next/font/local` 自托管
5. **数据万象 EXIF orientation**：`imageMogr2` 默认不读 EXIF 旋转，必须加 `auto-orient`
6. **h2c 与 EdgeOne 不兼容**：握手方式不同，最终走 HTTPS + H2 回源

## 六、结语

整个过程的本质是**把对的工作交给对的服务**：

- Next.js 图片优化对 Vercel 弹性计算没问题，但 2 核固定规格遇上 80+ 并发就是灾难——交给腾讯云数据万象，CPU 零负载
- HTTP/2 多路复用解决回源排队——HTTPS 的额外开销在多路复用收益面前可以忽略
- 跨区域数据库延迟——本地化部署是终极方案

AI 编程工具对这类工作的加速效果明显。存储适配、图片批处理、部署自动化、CDN 排障——每个环节需要不同领域知识，在 WorkBuddy 里描述清楚需求，Opus 4.6 生成初版代码，review 微调就能用。选对工具和模型，效率翻倍。