---
title: "将 Hugo 博客从 Vercel 迁移到 GitHub Actions + 腾讯云 COS"
date: 2026-03-22
description: "借助 WorkBuddy (Claude Opus 4.6) 完成从 Vercel 到 GitHub Actions + 腾讯云 COS + EdgeOne 的迁移实践：从最初的第三方 Action 踩坑，到最终实现 coscli sync 增量同步 + EdgeOne 精准缓存清理的自动化部署流水线。"
categories: ['tech']
tags:
  - Hugo
  - GitHub Actions
  - 腾讯云
  - COS
  - EdgeOne
  - CI/CD
  - ai-coding
  - 博客
---

本文记录了将 Hugo 博客从 Vercel 迁移到 GitHub Actions + 腾讯云 COS + EdgeOne 的过程，最终实现了增量同步、并发保护、精准缓存清理的自动化部署流水线。整个 workflow 的编写和迭代主要借助 WorkBuddy（Claude Opus 4.6）完成，OpenClaw 做一些终端辅助。

<!--more-->

## 背景

之前博客托管在 Vercel 上，使用 Vercel 的自动构建功能。迁移前也预研过腾讯云 CloudBase 和 EdgeOne Pages，但这两个平台主要面向 Node.js 生态（Next.js、Nuxt 等），而 Hugo 是 Go 语言实现的静态站点生成器，构建产物就是纯静态文件，不需要 Node 运行时——用 COS 托管静态文件 + EdgeOne 做 CDN 加速反而是更轻量直接的方案。

最终选择 GitHub Actions + COS + EdgeOne，主要基于：

1. **国内访问速度** — COS + EdgeOne CDN 在国内有充足节点，访问体验好得多
2. **成本可控** — COS 存储 + EdgeOne 流量的组合成本很低
3. **流程可控** — 构建、部署、缓存清理全部可编排，出了问题能快速定位

下图展示了当前部署的完整流程：

{{< mermaid >}}flowchart TD
    A([🚀 Git Push to main]):::trigger --> B[⚙️ GitHub Actions<br/>并发控制]:::ci
    B --> C[📥 Checkout<br/>fetch-depth: 2]:::step
    C --> D[🔧 安装 Hugo 二进制]:::step
    D --> E[🏗️ hugo --minify 构建]:::build
    E --> F[☁️ coscli sync --delete<br/>增量同步到 COS]:::deploy
    F --> G{📋 git diff<br/>检测变更文章}:::check
    G -->|有变更文章| H[🔄 purge_url<br/>公共页面 + 变更文章]:::purge
    G -->|无文章变更| I[🔄 purge_url<br/>仅公共页面]:::purge
    H --> J([✅ 部署完成]):::done
    I --> J

    classDef trigger fill:#667eea,stroke:#5a67d8,color:#fff,stroke-width:2px
    classDef ci fill:#4a5568,stroke:#2d3748,color:#fff,stroke-width:2px
    classDef step fill:#edf2f7,stroke:#a0aec0,color:#2d3748,stroke-width:1px
    classDef build fill:#f6ad55,stroke:#dd6b20,color:#fff,stroke-width:2px
    classDef deploy fill:#4299e1,stroke:#2b6cb0,color:#fff,stroke-width:2px
    classDef check fill:#fefcbf,stroke:#d69e2e,color:#744210,stroke-width:2px
    classDef purge fill:#68d391,stroke:#38a169,color:#fff,stroke-width:2px
    classDef done fill:#48bb78,stroke:#2f855a,color:#fff,stroke-width:2px
{{< /mermaid >}}

## 部署方案演进

迁移经历了两个阶段，逐步迭代优化。

**初版**用 `peaceiris/actions-hugo` + `zkqiang/tencent-cos-action`，主要问题：

- 全量上传，无增量同步
- COS 残留旧文件
- 第三方 Action 的 Node.js 20 deprecation warning
- 部署后 CDN 缓存不刷新

**当前方案**解决了上述所有问题——直接下载 Hugo 二进制（不依赖第三方 Action）、`coscli sync --delete` 增量同步（自动清理残留）、git diff 检测变更 + `purge_url` 精准缓存清理。

下面是 GitHub Actions 的部署页面，每次 push 后会自动触发：

![GitHub Actions Deploy](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/fe7c0de03cf6d0a1473df227375bc667.png)

## 完整 Workflow 详解

### 1. 触发条件与并发控制

```yaml
name: Deploy to COS

on:
  push:
    branches: ["main"]
  workflow_dispatch:

concurrency:
  group: deploy-cos
  cancel-in-progress: true
```

- **push to main**：推送到 main 分支时自动触发
- **workflow_dispatch**：支持在 GitHub 界面手动触发，方便调试
- **concurrency**：同一时间只允许一个部署任务运行。如果正在部署时又有新 push 进来，会取消正在进行的任务、执行最新的——避免并发部署导致文件状态混乱

### 2. Checkout 与 Hugo 安装

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    submodules: recursive
    fetch-depth: 2

# env 中定义了 HUGO_VERSION: '0.146.4'
- name: Setup Hugo
  run: |
    HUGO_URL="https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
    curl -fsSL --retry 3 --retry-delay 5 -o /tmp/hugo.tar.gz "$HUGO_URL"
    tar -xzf /tmp/hugo.tar.gz -C /usr/local/bin hugo
    hugo version
```

几个值得注意的细节：

- **`fetch-depth: 2`**：只拉取最近 2 次提交，既节省时间，又能让后面的 `git diff HEAD~1 HEAD` 检测到文件变更
- **直接下载 Hugo 二进制**：放弃 `peaceiris/actions-hugo`，直接从 GitHub Releases 下载。加了 `--retry 3` 重试机制，避免 GitHub CDN 偶发的网络抖动导致下载失败
- **先下载再解压**：之前用管道 `curl | tar` 的写法，遇到网络中断时 tar 会报错且错误信息不直观，拆开两步更稳健

### 3. 构建

```yaml
- name: Build
  run: hugo --minify
```

使用 `--minify` 压缩输出的 HTML/CSS/JS。早期遇到过 minify 导致 ASCII 图表渲染异常的问题，升级 Hugo 版本后已解决。

### 4. 使用 coscli 增量同步

```yaml
- name: Install coscli
  run: |
    curl -fsSL https://cosbrowser.cloud.tencent.com/software/coscli/coscli-linux-amd64 -o /usr/local/bin/coscli
    chmod +x /usr/local/bin/coscli

- name: Upload to COS
  run: |
    coscli sync ./public/ cos://${{ secrets.TENCENT_BUCKET }}/ \
      -r \
      -e cos.${{ secrets.TENCENT_REGION }}.myqcloud.com \
      -i ${{ secrets.TENCENT_SECRET_ID }} \
      -k ${{ secrets.TENCENT_SECRET_KEY }} \
      --delete
```

这是相比初版最大的改进之一：

- **`sync` 命令**：只上传有变化的文件，未修改的文件跳过，大幅减少上传量和时间
- **`--delete` 参数**：自动删除 COS 上有、但本地 `public/` 中没有的文件——相当于保持 COS 和构建产物完全一致
- **coscli vs coscmd**：早期尝试过 Python 版的 coscmd，但它不支持 `sync` 命令。coscli 是腾讯云官方的 Go 实现，功能更完善、速度也更快

### 5. 精准 EdgeOne 缓存清理

这是整个 workflow 中最有意思的部分——精准清理变更页面的缓存，而不是粗暴地清全站。

#### 检测变更文章

```yaml
- name: Detect changed posts
  id: changed-posts
  run: |
    if git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
      CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
    else
      CHANGED_FILES=$(git diff-tree --no-commit-id --name-only -r HEAD)
    fi

    POSTS=$(echo "$CHANGED_FILES" | grep "^content/posts/.*\.md$" || true)

    if [ -n "$POSTS" ]; then
      # 将变更的文章路径转为博客 URL
      URLS_JSON="["
      FIRST=true
      for file in $POSTS; do
        slug=$(basename "$file" .md)
        url="https://blog.tanteng.space/posts/${slug}/"
        if [ "$FIRST" = true ]; then
          URLS_JSON="${URLS_JSON}\"${url}\""
          FIRST=false
        else
          URLS_JSON="${URLS_JSON},\"${url}\""
        fi
      done
      URLS_JSON="${URLS_JSON}]"

      echo "article_urls=${URLS_JSON}" >> "$GITHUB_OUTPUT"
      echo "has_articles=true" >> "$GITHUB_OUTPUT"
    else
      echo "has_articles=false" >> "$GITHUB_OUTPUT"
    fi
```

通过 diff 比较最近两次提交的差异，找出被修改的文章，将文件路径转换成对应的博客 URL。

#### 按需清理缓存

```yaml
- name: Purge EdgeOne Cache
  # HAS_ARTICLES / ARTICLE_INNER 等变量来自上一步 changed-posts 的 output
  # 此处省略了从 JSON 数组中提取内部元素的拼接逻辑
  run: |
    # 每次部署都清理的公共页面
    COMMON_URLS='["https://blog.tanteng.space/","https://blog.tanteng.space/posts/","https://blog.tanteng.space/categories/","https://blog.tanteng.space/tags/"]'

    # 如果有文章变更，追加到清理列表
    if [ "$HAS_ARTICLES" = "true" ]; then
      # 合并公共 URL 和文章 URL
      ALL_URLS="[${COMMON_INNER},${ARTICLE_INNER}]"
    else
      ALL_URLS="$COMMON_URLS"
    fi

    tccli teo CreatePurgeTask \
      --region ap-guangzhou \
      --ZoneId "$ZONE_ID" \
      --Type purge_url \
      --Targets "$ALL_URLS"
```

![精准 EdgeOne 缓存失效](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/4a89968718c94dc5c0b8e326339c38e9.png)

缓存清理策略分两层：

- **公共页面**（首页、文章列表、分类页、标签页）：每次部署都清理，因为新增或修改文章都会影响这些页面的内容
- **文章页面**：只清理实际被修改的文章对应的 URL

使用 `purge_url`（精准 URL 模式）而非 `purge_host`（全站清理），好处是：
- **避免缓存雪崩**：全站清理意味着所有页面瞬间回源，对 COS 源站产生压力突增
- **CDN 命中率更高**：未修改的文章继续享受缓存加速，用户体验不受影响
- **清理速度更快**：只清理几个 URL，EdgeOne 几乎秒级生效

## 踩坑记录

### 坑 1：schema.html 缺失

**现象**：构建时报错 `partial "schema.html" not found`

**解决**：主题需要但未提供的模板文件，在项目的 `layouts/partials/` 目录下补上即可：

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

### 坑 2：mermaid.min.js 上传失败

**现象**：`mermaid.min.js`（3.3MB）上传 COS 时报 `UserNetworkTooSlow` 错误

**解决**：将 mermaid 改为 CDN 加载，不再打包到构建产物中：

```html
<script type="module">
  import mermaid from 'https://cdn.jsdmirror.com/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'default' });
</script>
```

### 坑 3：curl | tar 管道下载不稳定

**现象**：偶发 `curl | tar` 管道执行失败，错误信息是 tar 报错，实际是网络中断

**解决**：拆成两步——先 `curl` 下载到 `/tmp/hugo.tar.gz`，再 `tar` 解压。加上 `--retry 3 --retry-delay 5` 重试机制。

## 总结

从 Vercel 一键部署到 GitHub Actions + COS + EdgeOne，复杂度增加了一些，但换来了部署可控、增量同步、精准缓存清理和零第三方依赖。整个流水线跑一次大约 **30-40 秒**，push 后基本一分钟内网站就更新了，够用且省心。
