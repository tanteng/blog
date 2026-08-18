# 在分类/标签列表页隐藏 note 类型文章 — 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `Section == "notes"` 的文章不出现在分类/标签列表页与"相关文章"区域，仅保留在首页合并流与 `/notes/` 列表。

**Architecture:** 在项目侧 `layouts/_default/` 与 `layouts/_partials/` 覆盖主题模板，循环层加 `where ... "Section" "ne" "notes"` 过滤；判定方式与 `layouts/home.html` 既有约定一致。

**Tech Stack:** Hugo（静态站点）、Ananke 主题、Pagefind

## Global Constraints

- 改动限定在 3 个项目侧模板文件，不动主题目录。
- 判定方式：`Section == "notes"`，**不动 front matter**。
- Hugo partial 必须放在 `layouts/_partials/`，**不是** `layouts/partials/`。
- 验收必须用 `npm run build`（含 pagefind 重建），不能用 `hugo server -D` 容错代替。
- 提交信息中文，遵循既有 `feat: / fix: / docs:` 前缀风格。

---

## File Structure

| 文件 | 类型 | 职责 |
|------|------|------|
| `layouts/_default/taxonomy.html` | 新建 | 覆盖主题同名模板，过滤 `/categories/<slug>/` 与 `/tags/<slug>/` |
| `layouts/_default/terms.html` | 修改 | 已有覆盖，过滤 `/categories/` 与 `/tags/` 索引页 |
| `layouts/_partials/related-posts.html` | 修改 | 排除 note 后再走原有渲染逻辑 |

每个文件单一职责；3 个任务彼此独立、可单独回退。

---

### Task 1: 覆盖分类/标签详情页（taxonomy.html）

**Files:**
- Create: `layouts/_default/taxonomy.html`

**Interfaces:**
- Consumes: Hugo 主题 ananke 的 `taxonomy.html` 模板结构
- Produces: 一个同名覆盖文件，由 Hugo 在分类/标签详情页优先选择项目侧版本

- [ ] **Step 1: 新建 `layouts/_default/taxonomy.html`**

写入以下内容：

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

关键差异（相对 `themes/ananke/layouts/taxonomy.html` 第 9 行）：
- 在 `range` 前用 `$pages := where .Pages "Section" "ne" "notes"` 过滤 note。
- `range $pages` 替代原 `range .Pages`。
- 其他结构、类名、文案保持与主题一致。

- [ ] **Step 2: 运行生产构建验证**

```bash
npm run build
```

预期：构建成功（无 template 错误），生成的 `public/categories/photo/index.html` 与 `public/tags/film/index.html` HTML 中不包含 note 标题"Pentax 17 更需要好的胶卷"或"claude-code-vs-openclaw-harness"。

检查方法：

```bash
grep -c "Pentax 17 更需要好的胶卷" public/categories/photo/index.html
grep -c "Pentax 17 更需要好的胶卷" public/tags/film/index.html
grep -c "claude-code-vs-openclaw-harness" public/categories/photo/index.html
grep -c "claude-code-vs-openclaw-harness" public/tags/film/index.html
```

预期：四条命令均输出 `0`。

- [ ] **Step 3: 提交**

```bash
git add layouts/_default/taxonomy.html
git commit -m "feat: 分类/标签详情页排除 note 类型"
```

---

### Task 2: 更新分类/标签索引页（terms.html）

**Files:**
- Modify: `layouts/_default/terms.html`（整文件替换）

**Interfaces:**
- Consumes: `.Data.Pages`（term 集合）、`$term.Pages`（每个 term 下的页面集合）
- Produces: 过滤后的 term 预览列表与"查看全部 N 篇"计数

- [ ] **Step 1: 整文件替换 `layouts/_default/terms.html`**

将 `layouts/_default/terms.html` 内容替换为：

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

关键差异（相对原文件第 10-19 行）：
- 新增 `$filtered := where $term.Pages "Section" "ne" "notes"` 中间变量。
- `range first 5 $filtered` 替代 `range first 5 $term.Pages`。
- `len $filtered` 替代 `len $term.Pages`，"查看全部 N 篇"的 N 与预览一致、不含 note。

- [ ] **Step 2: 运行生产构建验证**

```bash
npm run build
```

预期：构建成功。

检查索引页（注意：`photo` 分类同时挂载 posts 与 notes，`film` 标签同时挂载 posts 与 notes）：

```bash
grep -c "Pentax 17 更需要好的胶卷" public/categories/index.html
grep -c "Pentax 17 更需要好的胶卷" public/tags/index.html
grep -c "claude-code-vs-openclaw-harness" public/categories/index.html
grep -c "claude-code-vs-openclaw-harness" public/tags/index.html
```

预期：四条命令均输出 `0`。

再确认计数文本：

```bash
grep -oE "查看全部 [0-9]+ 篇" public/categories/index.html | head -5
grep -oE "查看全部 [0-9]+ 篇" public/tags/index.html | head -5
```

预期：数字应与去掉 note 后每个 term 的文章数一致（人工对照原计数 - note 数）。

- [ ] **Step 3: 提交**

```bash
git add layouts/_default/terms.html
git commit -m "feat: 分类/标签索引页排除 note 类型并修正计数"
```

---

### Task 3: 更新相关文章（related-posts.html）

**Files:**
- Modify: `layouts/_partials/related-posts.html:1`

**Interfaces:**
- Consumes: `.Site.RegularPages.Related .`（Related 返回的 page 集合）
- Produces: 过滤掉 note 之后的 page 集合，供 `with` 块使用

- [ ] **Step 1: 修改第 1 行**

将原文件第 1 行：

```go-html-template
{{- $related := .Site.RegularPages.Related . | collections.First 5 -}}
```

替换为：

```go-html-template
{{- $related := where (.Site.RegularPages.Related . | collections.First 5) "Section" "ne" "notes" -}}
```

注意：`where` 包裹整个 `.Site.RegularPages.Related . | collections.First 5` 表达式——先取 top 5 再过滤，而不是先过滤全集再取 5（避免过滤后不足 5 篇）。文件其余部分（`{{ if not .Params.disableRelated }}`、`{{ with $related }}`、内部 HTML 结构）保持不变。

- [ ] **Step 2: 运行生产构建验证**

```bash
npm run build
```

预期：构建成功。

抽样检查 note 单页（pentax-17-needs-good-film 与 claude-code-vs-openclaw-harness）输出：

```bash
grep -c "相关文章" public/notes/pentax-17-needs-good-film/index.html
grep -c "相关文章" public/notes/claude-code-vs-openclaw-harness/index.html
```

预期：两条均输出 `0`（note 单页下无任何相关文章，整段不渲染）。

再抽一篇 posts 单页确认功能未回归：

```bash
grep -c "相关文章" public/posts/<some-post-slug>/index.html
```

预期：输出 ≥ `1`（posts 单页仍有"相关文章"区）。

- [ ] **Step 3: 提交**

```bash
git add layouts/_partials/related-posts.html
git commit -m "feat: 相关文章排除 note 类型"
```

---

### Task 4: 端到端浏览器复核

**Files:** 无（仅验证）

- [ ] **Step 1: 启动本地预览**

```bash
hugo server -D
```

读取启动输出中的本地端口（通常 `http://localhost:1313/`）。

- [ ] **Step 2: 浏览器复核清单**

依次访问下列页面，肉眼/快捷键 Ctrl+F 搜索 "Pentax" 与 "claude-code-vs-openclaw"：

| URL | 期望 |
|-----|------|
| `http://localhost:1313/categories/photo/` | 列表中无 note 标题 |
| `http://localhost:1313/tags/film/` | 列表中无 note 标题 |
| `http://localhost:1313/categories/` | 每个 term 预览与计数都不含 note |
| `http://localhost:1313/tags/` | 同上 |
| `http://localhost:1313/tags/ektar100/` | 若 term 仅挂在 note 上，预览区为空、无"查看全部 N 篇" |
| `http://localhost:1313/` | 首页合并流仍含 note |
| `http://localhost:1313/notes/` | notes 列表正常 |
| `http://localhost:1313/notes/pentax-17-needs-good-film/` | 单页正常；底部无"相关文章"区 |
| `http://localhost:1313/posts/<任意 posts 路径>/` | 单页底部"相关文章"区仍渲染 |

- [ ] **Step 3: 关闭预览**

回到终端，按 Ctrl+C 结束 `hugo server`。

- [ ] **Step 4: 整理最终提交（如有遗漏）**

若 Task 1-3 已分别提交，本步无操作；否则补一次提交：

```bash
git status
# 若有变更：
git add -A
git commit -m "chore: 端到端验证完成"
```

---

## Self-Review

**1. Spec coverage：**

| Spec 条款 | 对应任务 |
|----------|---------|
| 分类/标签详情页排除 note | Task 1 |
| 分类/标签索引页排除 note + 修正计数 | Task 2 |
| 相关文章排除 note | Task 3 |
| 不影响首页合并流、/notes/、/posts/ Section | Task 4 Step 2（验证清单） |
| 用生产构建 `npm run build` 验证 | Task 1-3 各 Step 2 |

无遗漏。

**2. Placeholder scan：** 全文无 "TBD" / "TODO" / "implement later"；每个代码块完整；无空泛描述。

**3. Type consistency：** `where ... "Section" "ne" "notes"` 在 3 个任务中使用完全一致；中间变量命名（`$pages` / `$filtered`）按各自文件约定，不冲突。

通过。

---

## 风险与回退

- Task 1 新增的 `layouts/_default/taxonomy.html` 若与未来升级的主题同名模板产生结构冲突，回退方式：`git revert` 对应 commit 或删除该文件即可恢复主题默认。
- Task 2/3 是修改，回退方式：`git revert` 对应 commit。
