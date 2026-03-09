---
title: "exif-photo-blog 接入腾讯云 COS 实战"
date: 2026-03-03
draft: false
tags: ["tencent-cloud", "cos", "vercel", "next-js", "exif", "photo-blog"]
categories: ["Tech"]
description: "将 exif-photo-blog 开源照片博客从 Vercel Blob 迁移到腾讯云 COS 的完整教程"
---

## 为什么需要接入腾讯云 COS？

[exif-photo-blog](https://github.com/sambecker/exif-photo-blog) 是一个基于 Next.js 的开源照片博客项目，可以自动展示照片的 EXIF 信息（如相机型号、镜头参数、拍摄时间、地理位置等），这是它的一大特色。项目还支持 AI 标签生成、响应式设计等功能。

原项目默认使用 **Vercel Blob** 存储照片，但国内用户访问 Vercel 速度不稳定，且流量费用较高。因此考虑迁移到腾讯云 COS——对国内用户更友好，访问速度快，成本也更低。

原项目支持 AWS S3、Cloudflare R2、MinIO 等 S3 兼容存储，但没有直接支持腾讯云 COS。好在腾讯云 COS 兼容 S3 API，可以通过适配器接入。

<!--more-->

---

## 一、创建 COS 适配器

参考项目现有的 S3 适配器，创建 `tencent-cos.ts`：

```typescript
import { S3Client, CopyObjectCommand, DeleteObjectCommand, ListObjectsCommand, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// 配置从环境变量读取
const TENCENT_COS_BUCKET = process.env.NEXT_PUBLIC_TENCENT_COS_BUCKET ?? '';
const TENCENT_COS_REGION = process.env.NEXT_PUBLIC_TENCENT_COS_REGION ?? '';
const TENCENT_COS_SECRET_ID = process.env.TENCENT_COS_SECRET_ID ?? '';
const TENCENT_COS_SECRET_KEY = process.env.TENCENT_COS_SECRET_KEY ?? '';
const TENCENT_COS_APP_ID = process.env.NEXT_PUBLIC_TENCENT_COS_APP_ID ?? '';

// 腾讯云 COS 的 endpoint 格式
const TENCENT_COS_ENDPOINT = `cos.${TENCENT_COS_REGION}.myqcloud.com`;

export const TENCENT_COS_BASE_URL = 
  `https://${TENCENT_COS_BUCKET}-${TENCENT_COS_APP_ID}.${TENCENT_COS_ENDPOINT}`;

export const tencentCosClient = () => new S3Client({
  region: TENCENT_COS_REGION,
  endpoint: `https://${TENCENT_COS_ENDPOINT}`,
  credentials: {
    accessKeyId: TENCENT_COS_SECRET_ID,
    secretAccessKey: TENCENT_COS_SECRET_KEY,
  },
  forcePathStyle: false,
});
```

**关键点：**
- 使用 AWS SDK 访问腾讯云 COS（COS 兼容 S3 API）
- endpoint 格式：`cos.{region}.myqcloud.com`
- Bucket 名称格式：`{bucket}-{appId}`

---

## 二、修改配置检测

在 `src/app/config.ts` 中添加配置检测：

```typescript
export const HAS_TENCENT_COS_STORAGE_CLIENT =
  Boolean(process.env.NEXT_PUBLIC_TENCENT_COS_BUCKET) &&
  Boolean(process.env.NEXT_PUBLIC_TENCENT_COS_REGION) &&
  Boolean(process.env.NEXT_PUBLIC_TENCENT_COS_APP_ID);

export const HAS_TENCENT_COS_STORAGE =
  HAS_TENCENT_COS_STORAGE_CLIENT &&
  Boolean(process.env.TENCENT_COS_SECRET_ID) &&
  Boolean(process.env.TENCENT_COS_SECRET_KEY);
```

---

## 三、集成到存储系统

修改 `src/platforms/storage/index.ts`，在各个存储函数中添加 tencent-cos 的 case：

| 函数 | 说明 |
|------|------|
| `putFile` | 上传文件 |
| `copyFile` | 复制文件 |
| `deleteFile` | 删除文件 |
| `getSignedUrlForKey` | 获取预签名 URL |
| `getSignedUrlForUrl` | 获取预签名 URL（从 URL 解析）|

---

## 四、环境变量配置

在 Vercel 项目中配置以下环境变量：

### 公共变量（NEXT_PUBLIC_ 开头）

| 变量名 | 说明 | 示例 |
|--------|------|------|
| NEXT_PUBLIC_TENCENT_COS_BUCKET | Bucket 名称 | my-photo-blog |
| NEXT_PUBLIC_TENCENT_COS_REGION | 区域 | ap-guangzhou |
| NEXT_PUBLIC_TENCENT_COS_APP_ID | 腾讯云 AppID | 1234567890 |

### 私有变量

| 变量名 | 说明 | 示例 |
|--------|------|------|
| TENCENT_COS_SECRET_ID | SecretId | AKIDxxxxxx |
| TENCENT_COS_SECRET_KEY | SecretKey | xxxxxxxxxx |

### 可选变量

```
NEXT_PUBLIC_STORAGE_PREFERENCE = tencent-cos
```

### 区域对应表

| 区域 | Region 值 |
|------|-----------|
| 广州 | ap-guangzhou |
| 上海 | ap-shanghai |
| 北京 | ap-beijing |
| 香港 | ap-hongkong |
| 新加坡 | ap-singapore |

---

## 五、注意事项

1. **权限问题**：确保 COS API 密钥有对应的读写权限
2. **CORS 配置**：在 COS 控制台配置 CORS，允许你的域名访问
3. **公网访问**：确保 Bucket 设置为公网可访问，或者使用 CDN 加速
4. **数据迁移**：如果已有数据，需要自行迁移或配置跨 Bucket 复制

---

## 效果

部署完成后，照片上传、读取都通过腾讯云 COS 进行，完美兼容项目现有的所有功能。

项目已开源在我的 GitHub：https://github.com/tanteng/exif-photo-blog-v2

---

## 参考

- [腾讯云 COS S3 API 文档](https://cloud.tencent.com/document/product/436/41284)
- [exif-photo-blog 原项目](https://github.com/sambecker/exif-photo-blog)
