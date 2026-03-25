---
title: "将 Next.js 照片博客从 Vercel 迁移到腾讯云 Lighthouse"
date: 2026-03-22T08:00:00+08:00
draft: false
tags: ['next.js', 'tencent-cloud', 'migration', 'ai-coding']
categories: ['tech']
description: "借助 AI 编程工具，将基于 exif-photo-blog 的 Next.js 照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse 自托管的完整实践"
featured_image: 'https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/6f7afe43432ec45cb352d6196738e34e.png'
---

本文记录了将基于 [exif-photo-blog](https://github.com/sambecker/exif-photo-blog) 的照片站点从 Vercel 全家桶迁移到腾讯云 Lighthouse 自托管的过程，迁移中借助 WorkBuddy（Claude Opus 4.6）和 OpenClaw（MiniMax-2.5）完成代码改造、脚本编写和问题排查。

<!--more-->

## 背景

我的站点 [photos.tanteng.space](https://photos.tanteng.space) 最初采用标准的 Vercel 部署方案（Vercel Serverless + Cloudflare R2 + Neon PostgreSQL + Vercel Edge Network），一键部署确实方便，但使用过程中一些问题逐渐浮现：

1. **跨洋延迟**：Neon PostgreSQL 部署在 us-east-1，每次查询跨太平洋往返，构建和加载都很慢
2. **访问体验**：Vercel Edge Network 在国内没有节点，访问速度受限
3. **成本考量**：Vercel、Cloudflare R2、Neon 各自计费，免费额度有限
4. **可控性**：基础设施散落在多个平台，排障和调优都不方便

在确定自托管方案之前，我也评估了腾讯云的几个 Serverless 产品：

- **CloudBase**：主要面向静态网站，这个相册服务包含 ISR、SSR 等特性，CloudBase 无法很好支持
- **EdgeOne Pages**：部分 Next.js 特性支持尚不完善

最终决定整体迁移到腾讯云 Lighthouse，把计算、存储、数据库全部集中到广州区域的一台 Lighthouse 实例上。迁移后的架构如下：

{{< mermaid >}}flowchart TB
    subgraph CDN["🌐 EdgeOne CDN"]
        A[photos.tanteng.space]:::cdn
    end

    subgraph Server["🖥️ 腾讯云 Lighthouse 广州"]
        direction TB
        B[Nginx<br/>反向代理 :80/443]:::step
        C[Next.js + PM2<br/>127.0.0.1:3000]:::build
    end

    subgraph Storage["☁️ 腾讯云 COS"]
        D[assets.tanteng.space<br/>CDN 加速]:::deploy
    end

    subgraph Database["🗄️ 本地数据库"]
        E[PostgreSQL<br/>localhost:5432]:::purge
    end

    CDN -->|HTTPS| B
    B -->|反向代理| C
    C -->|< 1ms| E
    C -->|内网访问| D

    classDef cdn fill:#667eea,stroke:#5a67d8,color:#fff,stroke-width:2px
    classDef step fill:#edf2f7,stroke:#a0aec0,color:#2d3748,stroke-width:1px
    classDef build fill:#f6ad55,stroke:#dd6b20,color:#fff,stroke-width:2px
    classDef deploy fill:#4299e1,stroke:#2b6cb0,color:#fff,stroke-width:2px
    classDef purge fill:#68d391,stroke:#38a169,color:#fff,stroke-width:2px

    style CDN fill:#f0f0ff,stroke:#667eea,stroke-width:2px,color:#2d3748
    style Server fill:#fff8f0,stroke:#f6ad55,stroke-width:2px,color:#2d3748
    style Storage fill:#f0f8ff,stroke:#4299e1,stroke-width:2px,color:#2d3748
    style Database fill:#f0fff0,stroke:#68d391,stroke-width:2px,color:#2d3748
{{< /mermaid >}}

全部在同一区域内网互通，数据库查询从跨洋 200ms+ 降到本地 < 1ms。

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

照片数量只有 400+ 张，数据量很小，单独购买云数据库性价比不高。直接在 Lighthouse 实例上本地部署 PostgreSQL，既省钱延迟又最低——从之前 Neon 的跨洋 200ms+ 降到本地 < 1ms。

### 2.2 安装与数据导入

在 Lighthouse 上安装 PostgreSQL 15.16，然后从 Neon 导出导入：

```bash
# 从 Neon 导出
pg_dump "postgres://user:pass@ep-xxx.us-east-1.aws.neon.tech/verceldb" > dump.sql

# 导入本地
psql -U tanteng -d verceldb < dump.sql
```

### 2.3 后续调优

- **连接池参数**：原来的配置是为跨区域 Neon 设计的，超时很长、连接数很大。迁移到本地后收紧参数，同时关闭 SSL
- **自动备份**：配置了 crontab 每天凌晨 3 点自动备份数据库，保留最近 30 份

---

## 三、部署方案：零停机部署脚本

告别 Vercel 的一键部署后，需要自己搞定 CI/CD。这部分同样在 WorkBuddy 中完成——描述清楚「独立构建、原子替换、自动清 CDN 缓存」的需求，它就生成了一套完整的部署脚本。核心设计：

- **独立构建目录**：在独立目录中构建，线上代码不受影响
- **原子替换**：`mv` 操作是文件系统原子操作，不存在中间状态
- **自动清缓存**：部署后通过腾讯云 API V3 调用 EdgeOne purge_host 清除全域缓存

---

## 四、EdgeOne 加速

站点接入 EdgeOne CDN，国内访问走就近节点，首屏加载从 2-4s 降到 1s 以内。图片通过 COS CDN 加速，使用自定义域名 `assets.tanteng.space`。

---

## 五、迁移效果

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

## 六、踩坑总结

1. **COS S3 兼容 endpoint**：格式是 `cos.<region>.myqcloud.com`，不是 `s3` 开头
2. **老照片缺优化版本**：从 R2 迁过来的照片需要补生成 `-sm`/`-md`/`-lg` 版本
3. **EdgeOne 与 Next.js RSC 缓存冲突**：EdgeOne 没正确处理 `Vary: RSC` 响应头，导致首页偶发显示乱码，需在规则引擎中将 RSC 请求头加入自定义 Cache Key
4. **fonts.googleapis.cn 不靠谱**：应该用 `next/font/local` 自托管

---

## 七、结语

AI 编程工具对这类迁移工作的加速效果非常明显。存储适配、图片批处理、部署自动化、CDN 排障——每个环节都需要不同领域的知识，在 WorkBuddy 里描述清楚需求，Opus 4.6 生成初版代码，review 微调就能用。OpenClaw 搭配 MiniMax-2.5 做终端辅助也不错，不过在写代码方面 Opus 还是更胜一筹。选对工具和模型，效率翻倍。

---

## 后记：安全加固

迁移完成后，我对服务器做了一轮安全加固。自托管意味着安全责任也转移到自己身上，以下是完成的几项措施：

### 1. PostgreSQL 关闭对外暴露

迁移初期为了方便调试，PostgreSQL 监听了所有网卡地址，`pg_hba.conf` 中也添加了 `0.0.0.0/0` 的访问规则。调试完成后立即收紧：

- `listen_addresses` 改为 `localhost`，数据库只监听本地回环地址
- `pg_hba.conf` 删除 `0.0.0.0/0` 规则，仅允许本地连接
- 防火墙关闭 5432 端口，从网络层彻底阻断外部访问

### 2. SSH 加固 + fail2ban

- `PermitRootLogin` 设为 `prohibit-password`，禁止 root 密码登录
- `PasswordAuthentication` 设为 `no`，全面禁用密码认证，仅允许密钥登录
- 安装并启用 fail2ban，SSH 连续 3 次认证失败即封禁 IP 24 小时

### 3. 敏感文件权限收紧

项目 `.env` 文件包含数据库密码、COS 密钥等敏感信息，权限从默认的 `644` 收紧到 `600`，仅 root 可读写。

这些都是自托管场景下的基本安全实践。数据库不对外、SSH 仅密钥登录、敏感配置严格权限——做到这几点，至少不会犯低级错误。
