---
title: "腾讯云 COS 存储桶 Policy 误配置导致未授权写入漏洞分析"
date: 2026-06-03T10:00:00+08:00
draft: false
tags: ['security', 'tencent-cloud', 'cos', 'black-hat-seo']
categories: ['tech']
description: "腾讯云 COS 存储桶被灰产扫描并注入赌博跳转页面，完整分析攻击原理、路径溯源，以及如何正确配置 COS 安全策略。"
---

6月某天下午，收到用户反馈：访问相册里的某张"图片"时，浏览器被跳转到了一个赌博网站。

这篇文章完整记录整个排查过程、根因定位（Bucket Policy 配置错误），以及后续的修复和安全配置建议。

<!--more-->

## 发现问题

一个用户反馈：访问 CDN 上某张图片时，浏览器被跳转到了赌博网站。

第一反应是：**这文件是怎么进到 COS 里的？**

关键信息：

| 属性 | 值 |
|------|-----|
| 桶名 | `photos-xxxxxx-1234567890`（示例格式） |
| 地域 | `ap-guangzhou` |
| CDN 域名 | `your-cdn-domain.com`（示例格式） |
| 文件路径 | `xxx/xxx/xxx`（随机生成） |
| 文件类型 | `text/html`（伪装成图片扩展名） |
| 上传时间 | 某日下午（非用户上传时间点） |

几个疑点立刻浮现：

1. **路径异常**：随机生成的层级目录，和正常相册路径结构完全不匹配
2. **Content-Type 伪装**：URL 以 `.png`/`.jpg` 结尾，但实际响应是 `text/html`
3. **上传时间**：这个时间点并没有上传任何文件

## 灰产页面分析

把这个 HTML 拉下来一看——就是一个**云端跳转劫持页**（俗称"寄生虫"或"快排"灰产页）。核心逻辑：

```html
<script src="https://rXXXX.com/Targeturlget/checkip?..."></script>
```

访问时，JS 会向外部发请求，判断当前访客是**搜索引擎爬虫**还是**真人**：

- **爬虫访问**：返回正常页面内容，养域名权重
- **真人访问**：执行 `window.location` 跳转至当下的赌博/诈骗站点

这是黑帽 SEO 的标准武器。灰产团队批量扫描全网有漏洞的 OSS/COS 桶，把这类页面塞进去，借你的域名权重给赌博网站引流。

## 排查路径：凭证泄露？

第一反应是检查：是不是代码泄露了 SecretId/SecretKey？

排查结果：

- 代码里没有硬编码密钥，均通过环境变量注入
- `.env` 文件没有被提交到 Git 仓库
- Git 历史里没有任何密钥明文
- 上传 API 路由有登录 session 鉴权，不是任意文件上传接口

**代码侧完全没有问题。**

## 决定性测试：匿名 PUT

排除代码问题后，剩下唯一可能：**桶本身的访问权限配置错了**。

对桶做了一个匿名 PUT 请求测试——不带任何密钥，看看能不能上传文件：

```bash
curl -X PUT "https://{bucket}.cos.ap-guangzhou.myqcloud.com/test-probe.txt" -d "test"
```

**返回 HTTP 200，文件写入成功。**

连测试文件的 DELETE 也匿名成功了——返回 204。

## 根因定位：Bucket Policy 的陷阱

这时候你可能觉得委屈：**桶明明设的是"公有读、私有写"，为什么还能被写入？**

问题不在桶 ACL，而在 **Bucket Policy（存储桶策略）**。

Bucket Policy 的优先级和粒度都高于桶 ACL。如果桶上绑定了这样一条 Policy，即使桶 ACL 显示"私有写"，Policy 仍然可以让"所有人"都能写——**Policy 直接架空了桶 ACL**。

以下是导致漏洞的那条 Policy（已做脱敏处理）：

```json
{
  "Statement": [{
    "Action": [
      "name/cos:HeadBucket",
      "name/cos:GetObject",
      "name/cos:HeadObject",
      "name/cos:OptionsObject",
      "name/cos:ListParts",
      "name/cos:PutObject",        // ← 任何人上传文件
      "name/cos:PostObject",       // ← 任何人上传文件
      "name/cos:DeleteObject",     // ← 任何人删除文件
      "name/cos:InitiateMultipartUpload",
      "name/cos:UploadPart",
      "name/cos:CompleteMultipartUpload",
      "name/cos:AbortMultipartUpload",
      "name/cos:GetBucketACL",
      "name/cos:GetObjectACL",
      "name/cos:PutBucketACL",    // ← 致命：任何人能改桶权限
      "name/cos:PutObjectACL"
    ],
    "Effect": "Allow",
    "Principal": {
      "qcs": ["qcs::cam::anyone:anyone"]  // ← 任何人
    },
    "Resource": [
      "qcs::cos:ap-guangzhou:uid/1234567890:{bucket}/*"
    ],
    "Sid": "costs-1733026935-xxxxxx-xxxxxx"
  }],
  "version": "2.0"
}
```

`Principal: anyone` 配上 `PutObject`/`DeleteObject`，灰产就可以直接写文件进来。`PutBucketACL` 更危险——**攻击者写完文件后还能把你的私有写权限改掉，等于把桶彻底接管。**

这条 Policy 通常是腾讯云 COS 控制台里选择"公有读写"预设模板时自动生成的。你以为选了"私有写"就安全了，实际上 Policy 层面的漏洞被忽略了。

## 正确的 Bucket Policy

如果你的场景确实是图片需要公网可读，匿名只应该有"读"，整条 Policy 应该只包含这三个 Action：

```json
{
  "Statement": [{
    "Action": [
      "name/cos:GetObject",
      "name/cos:HeadObject",
      "name/cos:OptionsObject"
    ],
    "Effect": "Allow",
    "Principal": {
      "qcs": ["qcs::cam::anyone:anyone"]
    },
    "Resource": [
      "qcs::cos:ap-guangzhou:uid/1234567890:{bucket}/*"
    ],
    "Sid": "anonymous-read-only"
  }],
  "version": "2.0"
}
```

写操作（上传照片）全部走**服务端用 SecretKey 签名**的方式——前台传图、看图都不受影响，灰产扫描器则无法再写入。

## 攻击链路总结

```
灰产扫描器（全网 Crawler）
    ↓ 发现桶 Policy 允许 anyone 写
    ↓ PUT 请求写入跳转 HTML
    ↓ 上传路径: xxx/xxx/xxx（随机目录批量写入）
    ↓ 通过 CDN 自定义域名访问
    ↓ 用户访问 → JS 判断 → 真人跳赌博站
```

跟代码、密钥、上传 API **一点关系都没有**——就是 Bucket Policy 配错了，扫描器扫到就直接写。

## 如何修复

### 1. 先堵 Policy 漏洞（最重要）

在腾讯云 COS 控制台 → 存储桶 → 权限管理 → 存储桶策略，把 Policy 替换成上面"正确的 Bucket Policy"那段的 JSON。只保留读操作，删掉所有写/删/改 ACL 的 Action。

### 2. 验证匿名写入已被封堵

```bash
curl -X PUT "https://{bucket}.cos.ap-guangzhou.myqcloud.com/verify-test.txt" -d "test"
# 应该返回 403 才算修好
```

### 3. 清理桶内非法文件

根据 `Last-Modified` 时间和异常路径（随机层级目录）筛出非正常上传文件，在 COS 控制台或用 SDK 批量删除。

### 4. 公开访问走 CDN 只读

桶改私有后，CDN 回源时需要用你的密钥签名访问 COS。控制台配置**回源鉴权**，前台用户体验不变，但灰产无法再写入。

### 5. 轮换一次密钥（善后建议）

虽然这次不是密钥泄露，但作为安全事件善后，建议在 CAM 控制台轮换一次 SecretId/SecretKey。

## 正确的 COS 安全配置

| 配置项 | 错误做法 | 正确做法 |
|--------|----------|----------|
| Bucket Policy | 给 `anyone` 开写权限 | 写操作走服务端签名，Policy 只开读 |
| 桶 ACL | 公有读写 | 私有读写 |
| 自定义域名/CDN | 直接绑定公可写桶 | CDN + 回源鉴权 |
| 密钥管理 | 代码硬编码 | 环境变量 / CAM 临时凭证 |

## 总结

这起事件的根因非常典型：**Bucket Policy 误配置给"任何人"开了写权限，被灰产扫描器发现后批量写入了跳转页面**。

腾讯云 COS 控制台的"公有读写"预设模板是重灾区——它自动生成的 Policy 把读和写同时开放给 `anyone`，开发者往往只看到桶 ACL 显示"私有写"就以为安全了，忽略了 Policy 层面的覆盖。

安全配置的核心原则很简单：**桶本身永远是私有读写（或只给匿名开读），公开写入全部走服务端签名路径**。做到这一点，就不会被这类自动化扫描攻击入侵。

---

*如果你也在用腾讯云 COS，建议立刻去控制台检查一下 Bucket Policy 配置——特别留意 `Principal` 里有没有 `anyone` 配了写操作。*
