---
title: "将 Hugo 博客从 Vercel 迁移到 CNB + EdgeOne Pages 方案"
date: 2026-01-04
draft: false
tags: ["hugo", "vercel", "cnb", "edgeone", "静态网站", "博客"]
categories: ["tech"]
description: "详解如何将托管在 Vercel 的 Hugo 静态博客迁移到腾讯云 CNB + EdgeOne Pages 的完整方案，包含配置步骤和注意事项。"
---

## 背景

Vercel 是一个优秀的静态网站托管平台，提供了自动构建和全球 CDN 分发功能深受开发者喜爱。然而，由于服务器位于海外，国内访问速度一直不理想，而且国内域名需要备案才能绑定。

近期，腾讯云推出了 CNB（Cloud Native Build）配合 EdgeOne Pages 的方案，为国内用户提供了一个值得考虑的替代选择。

<!--more-->

本文详细介绍该方案的实现方式和配置步骤。

## 为什么选择 CNB + EdgeOne Pages

### Vercel 的问题

- 国内访问速度慢
- 国内域名需备案才能绑定

### CNB + EdgeOne Pages 的优势

| 特性 | Vercel | CNB + EdgeOne Pages |
|------|--------|-----|
| 自动构建 | ✅ | ✅ |
| 静态托管 | ✅ | ✅ |
| 全球 CDN | ✅ | ✅ |
| 国内访问 | 慢 | 快 |
| 域名备案 | 需要 | 需要（但访问快） |
| 价格 | 免费版有限制 | 免费额度够用 |

## 迁移方案

### 整体流程

```
Git Push → CNB 构建 Hugo → 部署到 EdgeOne Pages → 全球分发
```

### 准备工作

1. **EdgeOne 账号**
   - 需实名认证
   - 在控制台创建 Pages 项目，选择「Direct Upload」类型
   - 获取 API Token

2. **CNB 账号**
   - 使用微信登录 cnb.cool
   - 导入博客仓库

### 配置步骤

#### 1. 创建 CNB 仓库并导入博客

登录 CNB 后，创建新仓库并将本地博客推送到 CNB。

#### 2. 配置 .cnb.yml

在博客根目录创建 `.cnb.yml` 文件：

```yaml
# 推送时触发构建
push:
  - docker:
      image: klakegg/hugo
    stages:
      - hugo --gc --minify

# 部署到 EdgeOne Pages
- name: Deploy to EdgeOne Pages
  image: tencentcom/deploy-eopages:latest
  env:
    EDGEONE_PAGES_API_TOKEN: ${{ secrets.EDGEONE_PAGES_API_TOKEN }}
    EDGEONE_PAGES_PROJECT: your-project-name
  script: |
    edgeone pages deploy ./public -n your-project-name -t $EDGEONE_PAGES_API_TOKEN
```

#### 3. 配置环境变量

在 CNB 仓库设置中添加密文：
- `EDGEONE_PAGES_API_TOKEN`：你的 EdgeOne Pages API Token

#### 4. 绑定域名（可选）

在 EdgeOne Pages 控制台绑定自己的域名，开启 CDN 加速。

## 迁移后的体验

### 优点

- 🚀 构建速度更快
- 🌏 国内访问流畅
- 💰 免费额度足够个人博客使用
- 🔄 推送到 CNB 后自动构建部署

### 注意事项

- CNB 是独立账号体系，不是腾讯云原生集成
- EdgeOne Pages 必须选择「Direct Upload」类型才能使用此插件
- 域名仍需备案（这是国内要求，无法避免）

## 总结

该方案提供了类似 Vercel 的自动构建体验，同时解决了国内访问速度的问题。CNB + EdgeOne Pages 的组合适合正在寻找 Vercel 替代方案的用户，尤其是对国内访问速度有要求的博客主。

需要注意的是，CNB 目前没有像 Vercel 那样的预览部署（PR 自动生成预览 URL）功能，但对于纯静态博客来说，这并非刚需。
