---
title: 在分类、标签列表页隐藏 note 类型文章
date: 2026-08-18
---

# 在分类、标签列表页隐藏 note 类型文章

## 背景

Hugo 博客使用 `Section == "notes"` 区分两类内容：

- `content/posts/`：长文章，正文流
- `content/notes/`：短动态（类微博），独立时间线

首页 `layouts/home.html` 显式合并 posts + notes 形成统一时间线。但当前实现里：

- 主题的 `taxonomy.html`（分类/标签详情页）使用 `{{ range .Pages }}`，会把带同一 term 的 note 也列出来
- 主题的 `terms.html` 同样会把 note 列在 term 索引页的预览里
- `_partials/related-posts.html` 用 `RegularPages.Related` 也会把 note 当成相关文章推荐

预期：note 只在首页合并流与 `/notes/` 列表页出现，不污染分类/标签页，也不出现在"相关文章"里。

## 目标

1. `/categories/<slug>/`、`/tags/<slug>/` 不再列出 note
2. `/categories/`、`/tags/` 索引页每个 term 的预览与"查看全部 N 篇"计数都不含 note
3. `related-posts.html` 不再把 note 列为相关文章
4. 不影响首页合并流、`/notes/` 列表、note 单页本身、`/posts/` Section 列表

## 方案选择

判定依据沿用项目既有约定：**`Section == "notes"`**（与 `layouts/home.html` 第 18 行 `where .Site.RegularPages "Section" "in" (slice "posts" "notes")` 一致），不动 front matter。

实现方式：在项目侧 `layouts/_default/` 与 `layouts/_partials/` 覆盖主题模板，循环层加 `where ... "Section" "ne" "notes"`。

未考虑 `front matter type: note` 或 helper 抽离——前者与既有判定不一致，后者 YAGNI。

## 改动清单

### 1. `layouts/_default/taxonomy.html`（新增）

完整覆盖 `themes/ananke/layouts/taxonomy.html`，仅在 `range` 前过滤。

```go-html-template
{{ define "main" }}
  <article class="cf pa3 pa4-m pa4-l">
    <div class="measure-wide-l center f4 lh-copy nested-copy-line-height nested-links {{ $.Param "text_color" | compare.Default "mid-gray" }}">
      <p>{{ lang.Translate "taxonomyPageList" . }}</p>
    </div>
  </article>
  <div class="mw8 center">
    <section class="flex-ns mt5 flex-wrap justify-around">
      {{ $pages := where .Pages "Section" "ne" "notes" }}
      {{ range $pages }}
        <div class="w-100 mb4 relative bg-white">
          {{ .Render "summary" }}
        </div>
      {{ end }}
    </section>
  </div>
{{ end }}
```

### 2. `layouts/_default/terms.html`（修改）

项目已有同名覆盖，需替换整文件。`$term.Pages` 过滤、计数同步走过滤后的集合。

```go-html-template
{{ define "main" }}
  <div class="mw8 center">
    <section class="ph4">
      {{ range $term := first 100 .Data.Pages }}
        <h2 class="f1">
          <a href="{{ $term.RelPermalink }}" class="link blue hover-black">
            {{ $.Data.Singular | inflect.Humanize }}: {{ $term.Title }}
          </a>
        </h2>
        {{ $filtered := where $term.Pages "Section" "ne" "notes" }}
        {{ range first 5 $filtered }}
          {{ .Render "summary" }}
        {{ end }}
        {{ if gt (len $filtered) 5 }}
          <p class="f5 ml4">
            <a href="{{ $term.RelPermalink }}" class="link blue hover-dark-blue">
              查看全部 {{ len $filtered }} 篇 →
            </a>
          </p>
        {{ end }}
      {{ end }}
    </section>
  </div>
{{ end }}
```

### 3. `layouts/_partials/related-posts.html`（修改）

```go-html-template
{{- $related := where (.Site.RegularPages.Related . | collections.First 5) "Section" "ne" "notes" -}}
{{ if not .Params.disableRelated }}
  {{ with $related }}
<div class="related-posts bg-light-gray pa3 nested-list-reset nested-copy-line-height nested-links" data-pagefind-ignore>
  <p class="f5 b mb3">相关文章</p>
  <ul class="pa0 list">
    {{ range . }}
      <li class="mb2">
        <a href="{{ .RelPermalink }}">{{ .Title }}</a>
      </li>
    {{ end }}
  </ul>
</div>
  {{ end }}
{{ end }}
```

## 边界与失败处理

- 某 term 仅挂在 note 上：taxonomy 页只渲染顶部介绍，正文区为空；terms 索引里该 term 预览空、不显示"查看全部 N 篇"。
- note 单页下"相关文章"为空：`with` 走 false 分支，整段不渲染。
- 不影响 `sitemap.xml`：note 单页 URL 仍出现，符合预期（note 仍可访问）。
- 不影响 RSS（项目当前未配置 RSS 输出）。
- 不影响 Pagefind：note 单页本身不在排除范围内（只 `/profile.html` 排除）。

## 验收

- `npm run build` 通过（含 pagefind 索引重建）
- `hugo server` 访问以下 URL，确认表现：
  - `/categories/photo/` —— 无 note
  - `/tags/film/` —— 无 note
  - `/categories/` —— 每个 term 预览与计数都不含 note
  - `/tags/ektar100/` —— 若 term 仅有 note，预览区为空
  - `/` —— notes 仍出现
  - `/notes/` —— notes 列表正常
  - 单个 note 单页 —— "相关文章"区不渲染

## 风险

- 覆盖了主题 `taxonomy.html`。若未来升级 ananke 主题带来该模板非循环段的变化（如顶部文案/类名），需重新对齐；本次改动刻意只动 `range` 前的赋值，结构与主题保持一致。
