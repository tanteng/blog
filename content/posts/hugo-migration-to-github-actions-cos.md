---
title: "将 Hugo 博客从 Vercel 迁移到 GitHub Actions + 腾讯云 COS"
date: 2026-03-03
categories:
  - 技术
tags:
  - Hugo
  - GitHub Actions
  - 腾讯云
  - COS
  - 博客
---

> 本文记录了将 Hugo 博客从 Vercel 构建迁移到 GitHub Actions + 腾讯云 COS 的完整过程，包括遇到的问题和解决方案。

<!--more-->

## 背景

之前博客托管在 Vercel 上，使用 Vercel 的自动构建功能。出于以下原因考虑，决定迁移到 GitHub Actions + 腾讯云 COS：

1. **国内访问更快** - COS + CDN 国内节点访问速度更好
2. **成本控制** - 腾讯云 COS 存储成本较低
3. **完全掌控** - 构建流程完全可控

## 迁移过程

### 1. 创建 GitHub Actions Workflow

在仓库的 `.github/workflows/` 目录下创建部署配置：

```yaml
name: Deploy to COS

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.146.4'
          extended: true

      - name: Build
        run: hugo

      - name: Upload to COS
        uses: zkqiang/tencent-cos-action@v0.1.0
        with:
          args: upload -r ./public/ /
          secret_id: ${{ secrets.TENCENT_SECRET_ID }}
          secret_key: ${{ secrets.TENCENT_SECRET_KEY }}
          bucket: ${{ secrets.TENCENT_BUCKET }}
          region: ${{ secrets.TENCENT_REGION }}
```

### 2. 配置腾讯云 Secrets

在 GitHub 仓库的 Settings → Secrets 中配置：

- `TENCENT_SECRET_ID` - 腾讯云 API 密钥 ID
- `TENCENT_SECRET_KEY` - 腾讯云 API 密钥 Key
- `TENCENT_BUCKET` - COS 存储桶名称
- `TENCENT_REGION` - 存储桶区域（如 ap-guangzhou）

### 3. 配置 COS 静态网站

1. 创建 COS 存储桶
2. 设置访问权限为「公有读私有写」
3. 开启「静态网站」功能
4. 配置自定义域名（如 blog.tanteng.space）
5. 可选：配置 EdgeOne CDN 加速

## 遇到的问题及解决方案

### 问题 1：schema.html 缺失

**现象**：构建时报错 `partial "schema.html" not found`

**解决**：在项目的 `layouts/partials/` 目录下创建 `schema.html`：

```html
{{- $title := .Site.Title -}}
{{- $url := .Site.BaseURL -}}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "{{ $title }}",
  "url": "{{ $url }}"
}
</script>
```

### 问题 2：大文件上传超时

**现象**：`mermaid.min.js`（3.3MB）上传失败，报 `UserNetworkTooSlow` 错误

**解决**：将 mermaid 改为 CDN 加载：

```html
<script type="module">
  import mermaid from 'https://cdn.jsdmirror.com/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'default' });
</script>
```

### 问题 3：ASCII 图表渲染异常

**现象**：文章中的 ASCII 图表被转成 SVG 后显示异常

**解决**：Hugo 0.146.4 与 `--minify` 参数有兼容性问题，去掉 minify 后正常

### 问题 4：coscmd 不支持 sync 命令

**现象**：尝试使用 `sync` 命令时报错

**解决**：改用 `upload` 命令，删除操作单独执行

## 最终配置

```yaml
name: Deploy to COS

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.146.4'
          extended: true

      - name: Build
        run: hugo

      - name: Upload to COS
        uses: zkqiang/tencent-cos-action@v0.1.0
        with:
          args: upload -r ./public/ /
          secret_id: ${{ secrets.TENCENT_SECRET_ID }}
          secret_key: ${{ secrets.TENCENT_SECRET_KEY }}
          bucket: ${{ secrets.TENCENT_BUCKET }}
          region: ${{ secrets.TENCENT_REGION }}
```

## 总结

迁移过程整体顺利，主要工作量在解决各种兼容性问题。GitHub Actions 构建比 Vercel 慢一些（主要慢在上传环节），但在国内访问速度更快，体验更好。

如果对访问速度有更高要求，建议配合腾讯云 EdgeOne CDN 使用，效果更佳。
