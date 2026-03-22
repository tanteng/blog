---
title: "从 Vercel 到腾讯云：一个照片博客的完整自托管迁移实录"
date: 2026-03-22T08:00:00+08:00
draft: false
tags: ['vercel', 'tencent-cloud', 'next.js', 'migration']
categories: ['tech']
description: "记录将基于 exif-photo-blog 的照片站点从 Vercel 全家桶迁移到腾讯云自托管的完整过程"
---

本文记录了将基于 [exif-photo-blog](https://github.com/sambecker/exif-photo-blog) 的照片站点从 Vercel 全家桶迁移到腾讯云自托管的完整过程，涵盖计算、存储、数据库、CDN、字体等各个层面的改造。

<!--more-->

## 背景

[exif-photo-blog](https://github.com/sambecker/exif-photo-blog) 是一个优秀的 Next.js 开源照片博客项目，它能自动提取照片的 EXIF 数据（相机型号、镜头、光圈、快门、ISO 等），以一种简洁优雅的方式展示你的摄影作品。

我的站点 [photos.tanteng.space](https://photos.tanteng.space) 最初就是标准的 Vercel 部署方案：

| 组件 | 原方案 |
|------|--------|
| 计算 | Vercel Serverless Functions |
| 存储 | Cloudflare R2 + Vercel Blob |
| 数据库 | Neon PostgreSQL (us-east-1) |
| CDN | Vercel Edge Network |
| 字体 | Google Fonts CDN |

这套方案开箱即用，但随着照片越来越多（400+ 张），一些问题逐渐浮现：

1. **跨洋延迟**：Neon PostgreSQL 部署在 us-east-1，每次数据库查询跨太平洋往返，构建时间长、页面加载慢
2. **访问体验**：Vercel Edge Network 在国内没有节点，访问速度受限
3. **成本考量**：Vercel、Cloudflare R2、Neon 各自计费，且免费额度有限制
4. **可控性**：散落在多个平台的基础设施，排障和调优都不方便

于是，我决定整体迁移到腾讯云，把计算、存储、数据库全部拉到广州区域的一台 CVM 上。

## 迁移后的架构

```
                         ┌────────────────────┐
                         │    EdgeOne CDN     │
                         │ photos.tanteng.space│
                         └────────┬───────────┘
                                  │
                         ┌────────▼───────────┐
                         │      Nginx         │
                         │  (反向代理 :80/443) │
                         └────────┬───────────┘
                                  │
                         ┌────────▼───────────┐
                         │   Next.js (PM2)   │
                         │   127.0.0.1:3000   │
                         └───┬────────────┬───┘
                             │            │
                    ┌────────▼──┐    ┌────▼──────────────┐
                    │PostgreSQL │    │   腾讯云 COS       │
                    │ localhost │    │ assets.tanteng.space│
                    │   :5432   │    │   (CDN 加速)       │
                    └───────────┘    └───────────────────┘
```

| 组件 | 迁移后方案 |
|------|-----------|
| 计算 | 腾讯云 CVM (广州四区) + PM2 + Nginx |
| 存储 | 腾讯云 COS + CDN 自定义域名 |
| 数据库 | 本地 PostgreSQL 15.16 |
| 站点 CDN | EdgeOne |
| 图片 CDN | 腾讯云 COS CDN |
| 字体 | next/font/local 自托管 |

全部在同一区域内网互通，数据库查询从跨洋 200ms+ 降到本地 < 1ms。

---

## 一、图片存储迁移：R2 → 腾讯云 COS

### 1.1 为什么选 COS

腾讯云 COS 兼容 S3 API，这意味着可以直接复用 `@aws-sdk/client-s3`，无需引入额外的 SDK。加上服务器就在腾讯云上，内网传输免费且高速。

### 1.2 数据搬迁

先在服务器上安装 rclone，配置好 R2（源）和 COS（目标）：

```bash
# 服务器上走 COS 内网 endpoint，速度拉满
rclone copy r2:pics cos:photos-1303209934 \
  --transfers 16 --checkers 32
```

362 个文件，476.9 MiB，走内网只用了 **1 分 22 秒**（平均 5.37 MB/s）。对比之前在 Mac 上通过公网传输，速度提升了约 370 倍。

同时还有 19 张照片存在 Vercel Blob 上，一并迁移过来。

### 1.3 代码改造：新增 COS 存储适配层

exif-photo-blog 原生支持 Vercel Blob、Cloudflare R2、AWS S3、MinIO 四种存储后端，采用策略模式通过 `StorageType` 联合类型切换。我只需要新增一个 `tencent-cos` 类型。

关键点：COS 的 S3 兼容 endpoint 格式是 `cos.<region>.myqcloud.com`，不是 `s3.<region>.amazonaws.com`。其他 put/copy/list/delete/presigned-url 操作的代码和 AWS S3 几乎一样。

### 1.4 数据库 URL 批量更新

照片 URL 存在数据库里，需要从旧域名批量替换成新域名：

```sql
UPDATE photos
SET url = REPLACE(url, 'https://static.tanteng.space/', 'https://assets.tanteng.space/')
WHERE url LIKE '%static.tanteng.space%';
-- 404 rows affected
```

### 1.5 补生成图片优化版本

exif-photo-blog 在上传照片时会生成 `-sm`（小图）、`-md`（中图）、`-lg`（大图）三个优化版本用于响应式加载。但 R2 时期上传的老照片没有这些优化版本，导致构建时访问 404 报错。

解决方案是用 sharp 批量生成所有优化版本。

---

## 二、数据库迁移：Neon → 本地 PostgreSQL

### 2.1 迁移动机

之前用的是 Neon PostgreSQL，部署在 AWS us-east-1。从广州的 CVM 到美东的数据库，每次查询光网络往返就 200ms+，构建时几百次查询累积下来非常痛苦。

### 2.2 安装与数据导入

在 CVM 上安装 PostgreSQL 15.16，然后从 Neon 导出导入：

```bash
# 从 Neon 导出
pg_dump "postgres://user:pass@ep-xxx.us-east-1.aws.neon.tech/verceldb" > dump.sql

# 导入本地
psql -U tanteng -d verceldb < dump.sql
```

### 2.3 连接池参数调优

原来的连接池配置是为跨区域 Neon 设计的，超时很长、连接数很大。迁移到本地后需要调整参数，同时关闭 SSL。

### 2.4 自动备份

配置了 crontab 每天凌晨 3 点自动备份数据库，保留最近 30 份。

---

## 三、字体国内化：Google Fonts → 自托管

### 3.1 问题

原项目使用 Google Fonts CDN 加载 IBM Plex Mono 字体。`fonts.googleapis.cn` 并非 Google 官方镜像，可靠性存疑。

### 3.2 方案：next/font/local

Next.js 的 `next/font/local` 是完美的自托管方案——字体文件打包进应用，零外部请求，且自动优化。

同时移除了 `<head>` 中的 Google Fonts preconnect link，减少了不必要的 DNS 查询和连接建立。

---

## 四、部署方案：零停机部署脚本

告别 Vercel 的一键部署后，需要自己搞定 CI/CD。核心设计：

- **独立构建目录**：在独立目录中构建，线上代码不受影响
- **原子替换**：`mv` 操作是文件系统原子操作，不存在中间状态
- **自动清缓存**：部署后通过腾讯云 API V3 调用 EdgeOne purge_host 清除全域缓存

---

## 五、CDN 配置：EdgeOne 与 Next.js RSC 的坑

### 5.1 RSC 缓存串的问题

迁移后遇到了一个问题：**首页偶发显示乱码**，看起来像是 JSON 格式的数据流。

排查发现是 **Next.js RSC (React Server Components) 的响应被 CDN 错误缓存**。Next.js 通过 `Vary: RSC, Next-Router-Prefetch` 响应头告诉 CDN 区分缓存，但 **EdgeOne 没有正确处理 `Vary` 头**。

### 5.2 解决方案：自定义 Cache Key

在 EdgeOne 规则引擎中，将 RSC 请求头加入自定义 Cache Key，从根本上解决了缓存串的问题。

---

## 六、图片 CDN：COS 自定义域名

图片通过腾讯云 COS CDN 加速，使用自定义域名 `assets.tanteng.space`。在 `next.config.ts` 中注册 COS 域名到 Next.js 的 `images.remotePatterns`，使 `<Image>` 组件可以优化 COS 上的图片。

---

## 七、迁移效果

### 性能对比

| 指标 | 迁移前 | 迁移后 |
|------|--------|--------|
| 数据库查询延迟 | ~200ms (跨太平洋) | < 1ms (localhost) |
| 页面构建时间 | 3-5 分钟 | < 1 分钟 |
| 首屏加载 (国内) | 2-4s | < 1s |
| 字体加载 | 依赖 Google CDN | 本地打包，零外部请求 |

### 可靠性

- 数据库每日自动备份，保留 30 天
- 零停机部署（独立构建 + 原子替换）
- 部署后自动清除 CDN 缓存

---

## 八、踩坑总结

1. **COS S3 兼容 endpoint**：格式是 `cos.<region>.myqcloud.com`，不是 `s3` 开头
2. **rclone 内网传输**：腾讯云 CVM 到 COS 用内网 endpoint，免费且快
3. **老照片缺优化版本**：从 R2 迁过来的照片需要手动补生成
4. **EdgeOne 不尊重 Vary 头**：需要手动将 RSC 请求头加入 Cache Key
5. **fonts.googleapis.cn 不靠谱**：应该用 `next/font/local` 自托管

---

## 结语

整个迁移花了约两天时间。大部分工作在数据搬迁和代码适配上。最有价值的改变是 **把数据库搬到本地**——200ms 的查询延迟降到亚毫秒，对 Next.js 这种 SSR/SSG 密集查询的场景提升巨大。

代码改动已经推送到 [GitHub 仓库](https://github.com/tanteng/exif-photo-blog-v2)，其中腾讯云 COS 存储支持是对原项目的功能扩展，有兴趣的朋友可以参考。
