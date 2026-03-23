---
title: "将 Next.js 照片博客从 Vercel 迁移到腾讯云 Lighthouse"
date: 2026-03-22T08:00:00+08:00
draft: false
tags: ['vercel', 'tencent-cloud', 'next.js', 'migration', 'lighthouse', 'ai-coding']
categories: ['tech']
description: "借助 AI 编程工具，将基于 exif-photo-blog 的 Next.js 照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse 自托管的完整实践"
---

本文记录了将基于 exif-photo-blog 的照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse（轻量应用服务器）自托管的完整过程。整个迁移中，我大量借助了 AI 编程工具，主力是 WorkBuddy（底层模型为 Claude Opus 4.6），同时也用了 OpenClaw 做一些终端交互——从代码改造、脚本编写到问题排查，AI 让两天的迁移工作变得异常顺滑。

<!--more-->

## 背景

exif-photo-blog 是一个优秀的 Next.js 开源照片博客项目，它能自动提取照片的 EXIF 数据（相机型号、镜头、光圈、快门、ISO 等），以简洁优雅的方式展示摄影作品。

我的站点 [photos.tanteng.space](https://photos.tanteng.space) 最初采用标准的 Vercel 部署方案：

| 组件 | 原方案 |
|------|--------|
| 计算 | Vercel Serverless Functions |
| 存储 | Cloudflare R2 + Vercel Blob |
| 数据库 | Neon PostgreSQL (us-east-1) |
| CDN | Vercel Edge Network |

这套方案开箱即用，但随着照片增长到 400+ 张，一些问题逐渐浮现：

1. **跨洋延迟**：Neon PostgreSQL 部署在 us-east-1，每次查询跨太平洋往返，构建和加载都很慢
2. **访问体验**：Vercel Edge Network 在国内没有节点，访问速度受限
3. **成本考量**：Vercel、Cloudflare R2、Neon 各自计费，免费额度有限
4. **可控性**：基础设施散落在多个平台，排障和调优都不方便

在确定自托管方案之前，我也评估了腾讯云的几个 Serverless 产品：

- **CloudBase**：主要面向静态网站，这个相册服务包含 ISR、SSR 等特性，CloudBase 无法很好支持
- **EdgeOne Pages**：部分 Next.js 特性支持尚不完善

最终决定整体迁移到腾讯云 Lighthouse，把计算、存储、数据库全部集中到广州区域的一台 Lighthouse 实例上。

## 迁移后的架构

{{< mermaid >}}flowchart TB
    subgraph CDN["EdgeOne CDN"]
        direction LR
        direction TB
        A[photos.tanteng.space]
    end

    subgraph Server["腾讯云 Lighthouse 广州"]
        direction TB
        B[Nginx<br/>反向代理 :80/443]
        C[Next.js<br/>PM2<br/>127.0.0.1:3000]
    end

    subgraph Storage["腾讯云 COS"]
        D[assets.tanteng.space<br/>CDN 加速]
    end

    subgraph Database["本地数据库"]
        E[PostgreSQL<br/>localhost:5432]
    end

    CDN --> B
    B --> C
    C --> E
    C --> D
{{< /mermaid >}}

| 组件 | 迁移后方案 |
|------|-----------|
| 计算 | 腾讯云 Lighthouse (广州四区) + PM2 + Nginx |
| 存储 | 腾讯云 COS + CDN 自定义域名 |
| 数据库 | 本地 PostgreSQL 15.16 |
| 站点 CDN | EdgeOne |
| 图片 CDN | 腾讯云 COS CDN |

全部在同一区域内网互通，数据库查询从跨洋 200ms+ 降到本地 < 1ms。

---

## AI 编程工具：迁移的加速器

在深入技术细节之前，先聊聊 AI 编程工具在这次迁移中扮演的角色。

迁移涉及存储适配层编写、数据库迁移、部署自动化、CDN 排障等多个环节，每个环节都需要不同领域的知识。如果全部手写手查，光是阅读 exif-photo-blog 的源码理解存储抽象层就要花不少时间。实际操作中，我主要用了两个 AI 编程工具：

- **WorkBuddy**（主力）：IDE 中的 AI 编程助手，底层模型选择的是 **Claude Opus 4.6**。这次迁移的核心工作——COS 存储适配层编写、图片批处理脚本生成、部署脚本、CDN 问题排查——基本都在 WorkBuddy 中完成。Opus 4.6 对复杂代码库的理解能力很强，能准确把握 exif-photo-blog 的存储策略模式，生成的代码质量很高，大多数时候微调几个参数就能直接用
- **OpenClaw**：一个开源的终端 AI 编程助手，类似 Claude Code。我搭配的模型是 **MiniMax-2.5**，主要用于终端环境下的一些辅助交互。坦率地说，和 WorkBuddy + Opus 4.6 相比，MiniMax-2.5 在复杂代码理解和生成上还是有明显差距——不过用来做一些简单的命令行辅助还是不错的

这两个工具在迁移过程中的使用分工：

| 场景 | 工具 | 模型 | 做了什么 |
|------|------|------|---------|
| 分析存储抽象层 | WorkBuddy | Claude Opus 4.6 | 阅读源码，梳理 StorageType 策略模式，给出新增 COS 类型的完整方案 |
| 编写 COS 适配代码 | WorkBuddy | Claude Opus 4.6 | 生成完整的 tencent-cos 存储后端代码 |
| 图片优化版本批量生成 | WorkBuddy | Claude Opus 4.6 | 编写 sharp 批处理脚本，一轮对话搞定 400+ 张照片 |
| 部署脚本 | WorkBuddy | Claude Opus 4.6 | 生成零停机部署脚本，包含构建、替换、缓存清除 |
| EdgeOne 缓存问题排查 | WorkBuddy | Claude Opus 4.6 | 分析 RSC 响应头，定位 Vary 头缓存串问题 |
| 终端辅助 | OpenClaw | MiniMax-2.5 | 命令行环境下的一些快速交互 |

下面进入具体的迁移步骤。

---

## 一、图片存储迁移：R2 → 腾讯云 COS

### 1.1 为什么选 COS

腾讯云 COS 兼容 S3 API，可以直接复用 `@aws-sdk/client-s3`，无需引入额外的 SDK。加上服务器就在腾讯云上，内网传输免费且高速。

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

exif-photo-blog 原生支持 Vercel Blob、Cloudflare R2、AWS S3、MinIO 四种存储后端，采用策略模式通过 `StorageType` 联合类型切换。我需要新增一个 `tencent-cos` 类型。

这部分工作在 WorkBuddy 中完成——让它阅读整个存储抽象层的代码，理解接口定义和各后端的实现方式，然后直接生成 COS 适配代码。Opus 4.6 对这种「理解现有架构 → 按相同模式扩展」的任务非常拿手，生成的代码风格和现有后端保持一致，几乎不需要修改。关键点是 COS 的 S3 兼容 endpoint 格式为 `cos.<region>.myqcloud.com`，其他 put/copy/list/delete/presigned-url 操作与 AWS S3 几乎一致。

### 1.4 数据库 URL 批量更新

照片 URL 存在数据库里，需要从旧域名批量替换为新域名：

```sql
UPDATE photos
SET url = REPLACE(url, 'https://static.tanteng.space/', 'https://assets.tanteng.space/')
WHERE url LIKE '%static.tanteng.space%';
-- 404 rows affected
```

### 1.5 补生成图片优化版本

exif-photo-blog 在上传照片时会生成 `-sm`（小图）、`-md`（中图）、`-lg`（大图）三个优化版本用于响应式加载。但 R2 时期上传的老照片没有这些版本，导致构建时 404 报错。

这个问题我在 WorkBuddy 中描述了需求：「COS 上有 400 多张原始照片，需要为每张生成 -sm、-md、-lg 三个优化版本，用 sharp 处理，上传回 COS」。WorkBuddy 直接生成了一个完整的 Node.js 批处理脚本——从 COS 拉取原图列表、用 sharp 按不同尺寸压缩、再上传回 COS，一轮对话搞定。手写这个脚本大概要半小时，AI 生成后微调几处参数就能跑。

---

## 二、数据库迁移：Neon → 本地 PostgreSQL

### 2.1 选型对比

最初考虑购买腾讯云 PostgreSQL 实例（PostgreSQL 14+），但评估后发现：

| 方案 | 月成本 | 延迟 | 备注 |
|------|--------|------|------|
| 腾讯云 PostgreSQL | ~¥200/月 | < 5ms | 需要额外配置内网连接 |
| 本地 PostgreSQL | 0（复用现有 Lighthouse）| < 1ms | 同一台实例，无需额外费用 |

照片数量只有 400+ 张，数据量很小，单独购买云数据库性价比不高。直接在 Lighthouse 实例上本地部署 PostgreSQL，既省钱延迟又最低。

### 2.2 迁移动机

之前的 Neon PostgreSQL 部署在 AWS us-east-1，从广州的服务器到美东的数据库，每次查询光网络往返就 200ms+。Next.js 构建时有几百次数据库查询，累积下来构建时间非常长。

### 2.3 安装与数据导入

在 Lighthouse 上安装 PostgreSQL 15.16，然后从 Neon 导出导入：

```bash
# 从 Neon 导出
pg_dump "postgres://user:pass@ep-xxx.us-east-1.aws.neon.tech/verceldb" > dump.sql

# 导入本地
psql -U tanteng -d verceldb < dump.sql
```

### 2.4 连接池参数调优

原来的连接池配置是为跨区域 Neon 设计的，超时很长、连接数很大。迁移到本地后需要调整参数，同时关闭 SSL。

### 2.5 自动备份

配置了 crontab 每天凌晨 3 点自动备份数据库，保留最近 30 份。

---

## 三、部署方案：零停机部署脚本

告别 Vercel 的一键部署后，需要自己搞定 CI/CD。这部分同样在 WorkBuddy 中完成——描述清楚「独立构建、原子替换、自动清 CDN 缓存」的需求，它就生成了一套完整的部署脚本。核心设计：

- **独立构建目录**：在独立目录中构建，线上代码不受影响
- **原子替换**：`mv` 操作是文件系统原子操作，不存在中间状态
- **自动清缓存**：部署后通过腾讯云 API V3 调用 EdgeOne purge_host 清除全域缓存

---

## 四、CDN 配置：EdgeOne 与 Next.js RSC 的坑

### 4.1 RSC 缓存串的问题

迁移后遇到了一个棘手的问题：**首页偶发显示乱码**，看起来像是 JSON 格式的数据流。

排查发现是 **Next.js RSC (React Server Components) 的响应被 CDN 错误缓存**。Next.js 通过 `Vary: RSC, Next-Router-Prefetch` 响应头告诉 CDN 区分缓存，但 **EdgeOne 没有正确处理 `Vary` 头**——导致 HTML 请求可能命中 RSC 的 JSON 缓存，页面就显示一堆乱码。

这个问题排查时，我把 curl 抓到的响应头贴给 WorkBuddy，Opus 4.6 很快就定位到 Vary 头和 CDN 缓存策略的冲突，并给出了解决方案。这种「贴一段日志让 AI 帮你分析」的场景，模型的推理能力越强效果越好。

### 4.2 解决方案：自定义 Cache Key

在 EdgeOne 规则引擎中，将 RSC 请求头加入自定义 Cache Key，从根本上解决了缓存串的问题。

---

## 五、图片 CDN：COS 自定义域名

图片通过腾讯云 COS CDN 加速，使用自定义域名 `assets.tanteng.space`。在 `next.config.ts` 中注册 COS 域名到 Next.js 的 `images.remotePatterns`，使 `<Image>` 组件可以优化 COS 上的图片。

---

## 六、迁移效果

### 性能对比

| 指标 | 迁移前 | 迁移后 |
|------|--------|--------|
| 数据库查询延迟 | ~200ms (跨太平洋) | < 1ms (localhost) |
| 页面构建时间 | 3-5 分钟 | < 1 分钟 |
| 首屏加载 (国内) | 2-4s | < 1s |

### 可靠性

- 数据库每日自动备份，保留 30 天
- 零停机部署（独立构建 + 原子替换）
- 部署后自动清除 CDN 缓存

---

## 七、踩坑总结

1. **COS S3 兼容 endpoint**：格式是 `cos.<region>.myqcloud.com`，不是 `s3` 开头
2. **rclone 内网传输**：Lighthouse 到 COS 用内网 endpoint，免费且快
3. **老照片缺优化版本**：从 R2 迁过来的照片需要补生成 `-sm`/`-md`/`-lg` 版本
4. **EdgeOne 不尊重 Vary 头**：需要手动将 RSC 请求头加入 Cache Key
5. **fonts.googleapis.cn 不靠谱**：应该用 `next/font/local` 自托管

---

## 结语

整个迁移花了约两天时间。最有价值的改变是 **把数据库搬到本地**——200ms 的查询延迟降到亚毫秒，对 Next.js 这种 SSR/SSG 密集查询的场景提升巨大。

另一个深刻感受是 **AI 编程工具对这类迁移工作的加速效果非常明显**。这次迁移涉及的技术面很广——存储 API 适配、图片批处理、部署自动化、CDN 缓存策略——每个环节都需要不同领域的知识。以前可能要查好几轮文档才能搞定的事，现在在 WorkBuddy 里描述清楚需求，Opus 4.6 生成初版代码，自己 review 一遍微调参数就能用。模型能力的差异在实际工程中体现得很明显：同样的任务，Claude Opus 4.6 对复杂代码库的理解深度和生成质量，确实比我在 OpenClaw 中用的 MiniMax-2.5 高出一截。选对工具和模型，真的能让效率翻倍。
