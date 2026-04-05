---
title: "Hugo 博客搜索方案对比与踩坑记录"
date: 2026-03-25
tags: ["hugo", "search", "pagefind", "blog"]
categories: ["tech"]
description: "对比 Fuse.js、Lunr.js、Pagefind 等 Hugo 博客搜索方案，详解踩坑经历与最佳实践"
featured_image: "/images/blog-cover.jpg"
---

之前给博客加了搜索功能，调研了几种方案，踩了不少坑，记录一下。

## 方案对比

| 方案 | 索引大小 | 搜索体验 | 配置复杂度 | 中文支持 |
|------|----------|----------|------------|----------|
| Fuse.js | ~50KB | 需手动实现 UI | 中 | 一般 |
| Lunr.js | ~50KB | 需手动实现 UI | 中 | 一般 |
| Pagefind | ~20KB（按需加载） | 自带 UI | 低 | 需配置 |
| Algolia | 免费额度内 OK | 极佳 | 高 | 好 |

<!--more-->

### 1. Fuse.js / Lunr.js（客户端方案）

最早用的方案。Hugo 生成 `index.json`，浏览器加载后用 Fuse.js 在前端搜索。

**优点：**
- 简单，Hugo 原生支持
- 无需服务端支持
- 开发模式可用

**缺点：**
- `index.json` 包含所有内容，流量开销大
- 搜索体验需自己实现
- 大站性能差

### 2. Pagefind（推荐）

静态搜索工具，Hugo 官方推荐。

**优点：**
- 构建时生成索引，体积小
- 自带搜索 UI
- 支持增量索引
- 纯静态，无需服务端

**缺点：**
- 需要在构建后运行命令
- 开发模式不生效，必须构建后才能测试
- 中文需要配置 `--force-language`

**配置：**

```yaml
# GitHub Actions
- name: Build search index
  run: npx pagefind --site public --force-language zh
```

### 3. Algolia

第三方搜索服务，适合大型站点。

**优点：**
- 搜索体验极佳
- 支持中文分词
- 有免费额度

**缺点：**
- 需要第三方账号
- 配置复杂
- 依赖外部服务

## 踩坑记录

### 1. Hugo Partial 路径问题

Hugo partials 必须放在 `layouts/_partials/` 目录，不是 `layouts/partials/`。

```bash
# ✅ 正确
layouts/_partials/site-navigation.html

# ❌ 错误
layouts/partials/site-navigation.html
```

开发模式（`hugo server`）可能有容错机制，看似能用。但生产构建（`hugo --minify`）是严格模式，会忽略错误路径。

**教训：修改后必须用 `npm run build` 检查实际输出，不能只靠开发服务器。**

### 2. Pagefind 索引重复

Hugo 默认生成多语言版本（zh 和 zh-cn），导致 Pagefind 索引了两遍。

**解决：** 强制指定语言：

```bash
npx pagefind --site public --force-language zh
```

### 3. 排除不需要的内容

搜索结果中出现"相关文章"区块的内容，导致搜索不准确。

**解决：** 使用 `data-pagefind-ignore` 属性：

```html
<div class="related-posts" data-pagefind-ignore>
  ...
</div>
```

同样需要排除的还有：
- 分页页（`/page/2/` 等）
- 标签页（`/tags/*/`）
- 分类页（`/categories/*/`）
- 首页

```html
<body {{ if or (.IsHome) (hasPrefix .RelPermalink "/page/") (hasPrefix .RelPermalink "/posts/page/") (hasPrefix .RelPermalink "/tags/") (hasPrefix .RelPermalink "/categories/") }}data-pagefind-ignore{{ end }}>
```

### 4. 搜索框不显示

部署到生产环境后搜索框不显示，但本地正常。

**原因：** 路径错误 + Hugo 版本差异。生产环境用 `hugo --minify`，行为更严格。

**解决：** 确保 partial 文件放在正确路径。

### 5. 搜索结果重复

分页页和列表页被索引，导致同一内容出现多次。

**解决：** 在 `baseof.html` 的 body 标签添加排除条件。

## 最终方案

综合体验和配置复杂度，我选择了 **Pagefind**：

1. 索引体积小（按需加载）
2. 自带 UI，体验好
3. 纯静态，零依赖

部署流程：

```yaml
# GitHub Actions
- name: Build Hugo
  run: hugo --minify

- name: Build search index
  run: npx pagefind --site public --force-language zh
```

## 总结

| 问题 | 解决方案 |
|------|----------|
| 索引重复 | `--force-language zh` |
| 内容重复 | `data-pagefind-ignore` + 排除分页/列表页 |
| 搜索框不显示 | Hugo partial 必须放在 `layouts/_partials/` |
| 中文分词 | Pagefind 原生不支持，但基本可用 |

如果你的博客需要更精准的中文搜索，Algolia + Hugo 的 `algolia-index` 可能是更好的选择。但对于大多数博客，Pagefind 已经足够。
