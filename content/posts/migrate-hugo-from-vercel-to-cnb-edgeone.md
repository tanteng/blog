---
title: "将 Hugo 博客从 Vercel 迁移到 CNB + EdgeOne Pages 方案"
date: 2026-01-04
draft: false
tags: ["hugo", "vercel", "cnb", "edgeone", "迁移", "静态网站"]
categories: ["tech"]
description: "详解如何将托管在 Vercel 的 Hugo 静态博客迁移到腾讯云 CNB + EdgeOne Pages 的完整方案，包含配置步骤和注意事项。"
---

## 背景

之前我将博客托管在 Vercel，虽然体验不错，但国内访问速度一直是个问题。最近发现腾讯云推出了 CNB（Cloud Native Build）配合 EdgeOne Pages 的方案，经过研究后发现这简直是为国内用户量身定制的替代方案。

<!--more-->

本文记录完整的迁移方案，供有同样需求的朋友参考。

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

本文记录的是迁移方案，目前还未实际执行。如果你也在寻找 Vercel 的国内替代方案，CNB + EdgeOne Pages 是一个值得考虑的选项。它提供了类似的自动构建体验，同时解决了国内访问速度的问题。

唯一的遗憾是 CNB 没有像 Vercel 那样的预览部署（PR 自动生成预览 URL）功能，但对于静态博客来说，这并不是刚需。

后续实际迁移时会更新实际体验。有问题欢迎留言交流。
