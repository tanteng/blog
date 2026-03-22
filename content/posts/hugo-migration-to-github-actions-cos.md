---
title: "将 Hugo 博客从 Vercel 迁移到 GitHub Actions + 腾讯云 COS"
date: 2026-03-22
description: "借助 WorkBuddy (Claude Opus 4.6) 完成从 Vercel 到 GitHub Actions + 腾讯云 COS + EdgeOne 的迁移实践：从最初的第三方 Action 踩坑，到最终实现 coscli sync 增量同步 + EdgeOne 精准缓存清理的自动化部署流水线。"
categories:
  - 技术
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

> 本文记录了将 Hugo 博客从 Vercel 迁移到 GitHub Actions + 腾讯云 COS + EdgeOne 的完整过程。整个迁移中，我主要借助 [WorkBuddy](https://www.codebuddy.cn/)（底层模型为 Claude Opus 4.6）完成 workflow 编写和迭代优化，同时也用了 [OpenClaw](https://github.com/nicepkg/openclaw) 做一些终端辅助。从最初的简单 workflow 一步步演进到今天的方案——增量同步、并发保护、精准缓存清理，AI 工具让整个迭代过程高效了不少。

## 背景

之前博客托管在 Vercel 上，使用 Vercel 的自动构建功能。出于以下原因，决定迁移到 GitHub Actions + 腾讯云 COS：

1. **国内访问速度** — COS + EdgeOne CDN 在国内有充足节点，访问体验好得多
2. **成本可控** — COS 存储 + EdgeOne 流量的组合成本远低于其他方案
3. **流程可控** — 构建、部署、缓存清理全部可编排，出了问题能快速定位

<!--more-->

下图展示了当前部署的完整流程：

![Deploy 整体流程](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/fe7c0de03cf6d0a1473df227375bc667.png)

## 部署方案演进

整个迁移并不是一步到位的，而是经历了几个阶段的迭代。值得一提的是，这个演进过程本身就是一次 **AI 驱动的渐进式改造**——每次遇到问题，我都在 WorkBuddy 里描述现象和需求，Opus 4.6 生成改进方案，review 后直接应用。从第一版到当前版本，workflow 前后改了十几个 commit，但每次迭代基本都是「描述问题 → AI 给方案 → 微调 → 提交」的循环，效率非常高。

### 第一版：第三方 Action + 基础上传

最初的方案很简单——用 `peaceiris/actions-hugo` 安装 Hugo，用 `zkqiang/tencent-cos-action` 上传到 COS：

```yaml
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
```

这个方案能跑，但有几个明显的问题：

- **全量上传**：每次都把整个 `public/` 目录上传一遍，没有增量同步
- **残留文件**：删除或重命名文章后，COS 上的旧文件不会被清理
- **依赖第三方 Action**：`peaceiris/actions-hugo` 后来因为 Node.js 20 deprecation 问题开始报 warning
- **没有缓存清理**：部署完成后 EdgeOne CDN 还在返回旧内容

### 当前方案：直接下载 Hugo + coscli sync + EdgeOne 精准缓存清理

经过多轮迭代，当前的 workflow 解决了上述所有问题。这些改进并非一次性完成——比如从 `peaceiris/actions-hugo` 切换到直接下载二进制，是 WorkBuddy 在分析 Actions 日志中的 Node.js 20 deprecation warning 后建议的；`coscli sync --delete` 替换全量上传，也是在 WorkBuddy 中讨论「如何避免残留文件」时给出的方案。精准缓存清理的逻辑更复杂一些——涉及 git diff 检测变更、文件路径到 URL 的转换、JSON 数组拼接——这些 shell 脚本基本都是 Opus 4.6 一次生成的，我只调整了一些边界条件处理。

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

这是整个 workflow 中最有意思的部分，也是 AI 工具发挥最大价值的环节。

最初部署后我发现网站内容没更新，在 WorkBuddy 里问了一句「GitHub Actions 部署到 COS 后 EdgeOne CDN 缓存没更新怎么办」，Opus 4.6 给出了用 `tccli` 调用 EdgeOne API 清理缓存的方案。但第一版是 `purge_host` 全站清理，后来我觉得太粗暴了——又在 WorkBuddy 里提了「能不能只清理修改过的文章页面」，它就生成了下面这套 git diff + URL 转换 + 精准清理的完整逻辑。

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

通过 `git diff HEAD~1 HEAD` 比较最近两次提交的差异，精确找出哪些文章被修改了，然后把文件路径（如 `content/posts/my-post.md`）转换成对应的博客 URL（如 `https://blog.tanteng.space/posts/my-post/`）。

#### 按需清理缓存

```yaml
- name: Purge EdgeOne Cache
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

### 坑 3：peaceiris/actions-hugo 的 Node.js 20 问题

**现象**：GitHub Actions 日志中出现 Node.js 20 deprecation warning

**解决**：直接从 GitHub Releases 下载 Hugo 二进制，彻底绕开第三方 Action 的维护依赖。

### 坑 4：curl | tar 管道下载不稳定

**现象**：偶发 `curl | tar` 管道执行失败，错误信息是 tar 报错，实际是网络中断

**解决**：拆成两步——先 `curl` 下载到 `/tmp/hugo.tar.gz`，再 `tar` 解压。加上 `--retry 3 --retry-delay 5` 重试机制。

### 坑 5：git diff 在 shallow clone 下失败

**现象**：`git diff HEAD~1 HEAD` 报错，因为 GitHub Actions 默认 `fetch-depth: 1` 只拉取当前提交

**解决**：设置 `fetch-depth: 2`，并加上 fallback 逻辑——如果 `HEAD~1` 不存在（首次提交），则用 `git diff-tree` 列出当前提交的所有文件。

## 需要配置的 GitHub Secrets

在仓库的 **Settings → Secrets and variables → Actions** 中配置：

| Secret 名称 | 用途 |
|---|---|
| `TENCENT_SECRET_ID` | 腾讯云 API 密钥 ID |
| `TENCENT_SECRET_KEY` | 腾讯云 API 密钥 Key |
| `TENCENT_BUCKET` | COS 存储桶名称 |
| `TENCENT_REGION` | 存储桶区域（如 `ap-guangzhou`） |
| `EDGEONE_ZONE_ID` | EdgeOne 站点 ID |

## 总结

从最初的「Vercel 一键部署」到现在的 GitHub Actions + COS + EdgeOne 方案，表面上看复杂度增加了，但换来的是：

- **部署可控**：每个环节都能看到日志，出问题能快速定位
- **增量同步**：coscli sync 只上传变更文件，部署速度快
- **精准缓存**：不再粗暴地清全站缓存，只清理真正变化的页面
- **并发安全**：concurrency 机制确保不会出现部署冲突
- **零第三方依赖**：Hugo 安装、COS 上传、缓存清理全部用官方工具，不受第三方 Action 的维护状态影响

整个流水线跑一次大约 **30-40 秒**（视文件变更量而定），push 到 main 后基本一分钟内网站就更新了。

### AI 工具的使用体验

这次迁移再次印证了一个感受：**AI 编程工具对 CI/CD 类工作的加速效果尤其明显**。

CI/CD workflow 的特点是：迭代周期长（改一行 → commit → push → 等 Actions 跑完 → 看日志 → 再改），涉及的知识面杂（YAML 语法、shell 脚本、各种 CLI 工具的参数、云服务 API），且调试手段有限。传统做法是在文档和 Stack Overflow 之间反复横跳，而用 WorkBuddy 的体验则完全不同——把 Actions 日志贴进去，描述期望行为，Opus 4.6 就能精准定位问题并给出修复方案。

具体来说，这次 workflow 从初版到当前版本经历了 **十几次迭代**，每次的模式都很类似：

| 阶段 | 我做了什么 | WorkBuddy 做了什么 |
|------|-----------|-------------------|
| 初始搭建 | 描述需求：Hugo 博客部署到 COS | 生成完整的初版 workflow |
| 工具替换 | 贴上 Node.js 20 warning 日志 | 建议直接下载 Hugo 二进制，生成下载脚本 |
| 增量同步 | 问「怎么避免残留文件」 | 推荐 coscli sync --delete，给出完整配置 |
| 缓存清理 v1 | 说「部署后页面没更新」 | 生成 tccli purge_host 全站清理方案 |
| 缓存清理 v2 | 说「全站清理太粗暴了」 | 生成 git diff + URL 转换 + purge_url 精准清理的完整脚本 |
| 稳定性优化 | 贴上 curl \| tar 偶发失败的日志 | 建议拆分下载和解压，加 retry 机制 |

整个过程中我基本没有手写过完整的 shell 脚本——都是 Opus 4.6 生成初版，我 review 逻辑、调整参数、处理边界情况。这种「人负责决策和审查，AI 负责生成和迭代」的协作模式，对 CI/CD 场景来说特别合适。

对于个人博客来说，这套方案够用且省心。
