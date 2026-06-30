---
title: "腾讯云 COS 存储桶 Policy 误配置导致未授权写入漏洞分析"
date: 2026-06-03T10:00:00+08:00
draft: false
tags: ['security', 'tencent-cloud', 'cos', 'black-hat-seo']
categories: ['tech']
description: "腾讯云 COS 存储桶被灰产扫描并注入赌博跳转页面，完整分析攻击原理、路径溯源，以及如何正确配置 COS 安全策略。"
---

相册的 CDN 域名在微信里打开时突然提示"禁止访问"，申请解封后，微信安全团队的驳回理由里给出了一个陌生的图片路径——点开一看，是一个伪装成图片的 HTML 跳转页。

这篇文章完整记录整个排查过程、根因定位（Bucket Policy 配置错误），以及后续的修复和安全配置建议。

<!--more-->

## 发现问题

微信打开相册链接被拦截，申请解封后收到驳回通知，附带了一个违规 URL。访问这个"图片路径"后，浏览器直接跳转到了赌博网站。

关键信息：

| 属性 | 值 |
|------|-----|
| 桶地域 | `ap-guangzhou` |
| CDN 域名 | `assets.your-domain.com`（示例） |
| 文件路径 | `xxx/xxx/xxx`（随机生成，非正常相册路径） |
| 文件类型 | `text/html`（伪装成图片扩展名） |
| 上传时间 | 非用户上传时间点 |

几个疑点：

1. **路径异常**：随机生成的层级目录，和相册正常路径结构完全不匹配
2. **Content-Type 伪装**：URL 看似图片扩展名，实际是 HTML
3. **上传时间**：这个时间点并没有上传任何文件

## 灰产页面分析

把这个 HTML 拉下来一看——就是一个**云端跳转劫持页**（黑帽 SEO 俗称"寄生虫"或"快排"）。核心逻辑：

```html
<script src="https://rXXXX.com/Targeturlget/checkip?..."></script>
```

访问时 JS 向外部发请求，判断访客是**搜索引擎爬虫**还是**真人**——爬虫看到正常内容养权重，真人直接跳转到赌博/诈骗站点。

灰产团队批量扫描全网有漏洞的 OSS/COS 桶，把这类页面塞进去，借你的域名权重做 SEO 引流。

## 排查路径

第一反应是检查凭证是否泄露：

- 代码里没有硬编码密钥，均通过环境变量注入
- `.env` 没有提交到 Git
- Git 历史没有密钥明文
- 上传 API 有登录 session 鉴权

代码侧没有问题。

## 决定性测试：匿名 PUT

对桶发了一个匿名 PUT 请求测试（不带任何密钥）：

```bash
curl -X PUT "https://{bucket}.cos.ap-guangzhou.myqcloud.com/test-probe.txt" -d "test"
```

**返回 HTTP 200，文件写入成功。** DELETE 也成功了——连删都不需要鉴权。

## 根因定位：这条 Policy 从哪来的？

问题确认：**桶允许匿名写入**。但桶访问权限明明设的是"公有读、私有写"——为什么还能被写？

答案在 **Bucket Policy**。

Bucket Policy 是腾讯云 COS 控制台里可以单独配置的一条策略，**优先级高于桶 ACL**，可以单独给"任何人"开放各种权限，而桶 ACL 本身不变。

这条有问题 Policy 大概是这样的（已脱敏）：

```json
{
  "Statement": [{
    "Action": [
      "name/cos:HeadBucket",
      "name/cos:GetObject",
      "name/cos:HeadObject",
      "name/cos:OptionsObject",
      "name/cos:ListParts",
      "name/cos:PutObject",          // ← 任何人可上传
      "name/cos:PostObject",         // ← 任何人可上传
      "name/cos:DeleteObject",        // ← 任何人可删除
      "name/cos:InitiateMultipartUpload",
      "name/cos:UploadPart",
      "name/cos:CompleteMultipartUpload",
      "name/cos:AbortMultipartUpload",
      "name/cos:GetBucketACL",
      "name/cos:GetObjectACL",
      "name/cos:PutBucketACL",       // ← 任何人可改桶权限（致命）
      "name/cos:PutObjectACL"
    ],
    "Effect": "Allow",
    "Principal": {
      "qcs": ["qcs::cam::anyone:anyone"]  // ← 任何人
    },
    "Resource": [
      "qcs::cos:ap-guangzhou:uid/1234567890:{bucket}/*"
    ],
    "Sid": "costs-xxxxxxxx-xxxxxx-xxxxxx"
  }],
  "version": "2.0"
}
```

`Principal: anyone` 配上 `PutObject`/`DeleteObject`，灰产扫描器可以直接写文件进来。`PutBucketACL` 更危险——**攻击者甚至能把你改好的私有写权限再改回去，等于桶被完全接管。**

这条 Policy 可能是当初在腾讯云控制台配置 CDN 回源时，不小心选了"公有读写"预设模板后自动生成的；也可能是某次网站优化时意外写入的——**其来源已经无法追溯，但它的存在把桶的写入口彻底开放给了所有人。**

## 修复方案：直接删掉这条 Policy

**Policy 不是必须的。** 相册的访问模型很简单：

- **读（看图）**：浏览器 / CDN 匿名访问 → 桶 ACL"公有读"已覆盖
- **写（传图）**：服务端用 `SecretId/SecretKey` 签名上传 → 走密钥身份，跟 Policy 无关

桶 ACL 的"公有读"已经满足公开读需求，**这条 Policy 完全是多余的——删掉它最干净，不留任何隐患。**

### 操作步骤

**1. 删除 Policy**

腾讯云 COS 控制台 → 存储桶 → 权限管理 → 存储桶策略 → **直接删除这条 Policy**。

**2. 验证修复结果**

```bash
# 匿名 GET 图片，应返回 200
curl -I "https://{bucket}.cos.ap-guangzhou.myqcloud.com/existing-photo.jpg"

# 匿名 PUT 上传，应返回 403
curl -X PUT "https://{bucket}.cos.ap-guangzhou.myqcloud.com/verify-test.txt" -d "test"
# 403 = 修好了
```

删除 Policy 后：
- ✅ 看图正常（桶 ACL 公有读管着）
- ✅ 服务端传图正常（密钥签名，跟 Policy 无关）
- ✅ 匿名写/删/改权限彻底关闭，灰产入口消失

### 清理桶内灰产文件

根据 `Last-Modified` 时间和异常路径（随机层级目录）筛出非正常上传的文件，在 COS 控制台或用 SDK 批量删除。

### 善后建议

- **微信解封**：清理完灰产文件后重新申请，并说明已修复安全配置
- **轮换密钥**：虽然这次不是密钥泄露，但作为安全事件善后，建议在 CAM 控制台轮换一次 SecretId/SecretKey

## 攻击链路

```
灰产扫描器（全网 Crawler）
    ↓ 发现桶 Policy 开放了 anyone 写
    ↓ PUT 请求写入跳转 HTML（伪装成图片）
    ↓ 通过 CDN 域名访问
    ↓ 微信扫描到违规内容 → 封禁域名
    ↓ 用户访问 → 真人跳赌博站
```

## 总结

这起事件的根因：**Bucket Policy 误给 `anyone` 开了写权限，被灰产扫描器发现后写入了跳转页面，并导致微信域名被封禁。**

Policy 的来源已经无法追溯，但其存在把本该"公有读私有写"的桶变成了"公有读写"，让灰产有了可乘之机。

**最优解：直接删掉这条 Policy。** 桶 ACL 的"公有读"已足够，服务端上传走密钥签名跟 Policy 完全无关。Policy 不是必须有的，多余的 Policy 只会增加配错风险。

---

*如果你也在用腾讯云 COS，建议立刻去控制台检查一下 Bucket Policy——发现任何给 `anyone` 开放写权限的 Policy，最优解是直接删掉。*
