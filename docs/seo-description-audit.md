# tanteng.space 三站 SEO 体检 · 以 description 为核心

体检日期：2026-09-12  
范围：`tanteng.space`（枢纽站）、`blog.tanteng.space`（Hugo 274 篇）、`photos.tanteng.space`（Next.js 相册）

---

## 一、结论先讲

**三站的 description 问题不是"太短"，而是"没信息量"。**

现在的三句话都在**介绍"我有什么"**（我有一个博客、我有一个相册），没有一个在回答**"搜什么词能搜到你"**。

| 站点 | 现 description | 长度 | 判定 |
|---|---|---|---|
| tanteng.space | 谈腾（Tony）的个人主页。来自湖北，现居深圳。技术博客记录 AI、Golang、K8S、LLM 等文章；摄影作品记录日常和旅行。 | 65 字 | ⚠️ 及格。结构完整但**零搜索词**，且"个人主页"4 字浪费在开头黄金位 |
| blog.tanteng.space（首页） | 记录生活与成长的空间。热爱摄影、旅行、技术，这里有技术笔记与生活随想。 | 35 字 | ❌ 不合格。这句话可以贴到**任何**博客上，搜索引擎无法判断你讲什么 |
| blog.tanteng.space（关于页） | *（整页正文 209 字，未截断、未去格式）* | 209 字 | ❌ 严重。渲染成 description 会被腰斩，还带 emoji 和换行 |
| photos.tanteng.space | 滴，咔嚓！ | 5 字 | ❌ 严重。等于没有 description |

**一句判断**：`photos.tanteng.space` 的 "滴，咔嚓！" 是最贵的一处浪费——你的相册有 40+ 标签、5 个年份、深圳/香港/胶片/器材四条分类线，全都是长尾搜索的富矿，但首页在搜索结果里现在只显示四个字。

---

## 二、三站的分工：搜索意图分层

你在搜索引擎里的资产不是三个站，是**三类搜索意图**。description 要对准各自的意图，而不是互相抄同一句。

```
品牌词  ──→  tanteng.space（枢纽）
           "谈腾" "Tony老师" "tanteng" "谈腾 深圳"
           任务：把三个站 + 社交账号绑成一个"实体"

主题词  ──→  blog.tanteng.space
           "GraphRAG" "AI Agent" "期权 bull put spread" "胶片摄影"
           任务：让每篇文章吃自己的长尾

地理/场景词 ──→  photos.tanteng.space
           "深圳 摄影" "赤柱 攻略" "大澳 渔村" "香港 胶片"
           任务：地域词 + 器材词 + 场景词
```

---

## 三、改写方案（可直接粘贴）

基准：百度 PC 端约展示 **78 个汉字**（移动端约 68），Google 桌面端约 80 汉字。所以目标区间是 **70–80 字**，且**关键词必须落在前 40 字内**（超出部分常被截断）。

### 3.1 tanteng.space 首页

**现在**（65 字）
```
谈腾（Tony）的个人主页。来自湖北，现居深圳。技术博客记录 AI、Golang、K8S、LLM 等文章；摄影作品记录日常和旅行。
```

**改成**（78 字，推荐）
```
谈腾（Tony老师），腾讯后台开发工程师，常驻深圳。汇总我的技术博客与摄影相册：AI Agent、Golang、K8S 技术笔记，深圳、香港与旅行摄影作品。
```

改动逻辑：

| 改动 | 原因 |
|---|---|
| `谈腾（Tony）` → `谈腾（Tony老师）` | "Tony老师" 是你在站内、抖音、视频号到处用的别名，也是**唯一可能被主动搜索**的称呼。主页 description 不写它，等于放弃这条入口 |
| 开头删掉"的个人主页" | 4 个字占在最贵的位置，且"个人主页"搜索量近零 |
| 补 `腾讯后台开发工程师` | 身份实体信号。"腾讯 + 后台开发"是**招聘方和同行真实会搜**的词，也强化 Person 实体 |
| `LLM` → `AI Agent` | 你 2026 年的内容重心已经是 Agent（Harness 工程、Agent 设计模式、GraphRAG）。用最热的词替换最泛的词 |
| `摄影作品记录日常和旅行` → `深圳、香港与旅行摄影作品` | 加地名。**"深圳""香港"是地理长尾的入口**，且你确实常往返两地 |
| 保留 `Golang`、`K8S` | 这两个词你有真实存量内容（2018 年至今），是稳态流量 |

**备选（更偏内容，适合百度收录）**
```
谈腾（Tony老师）个人主页。深圳腾讯后台开发工程师，博客写 AI Agent、大模型、Golang、K8S 技术笔记与读书随笔，相册收录深圳、香港与旅行摄影作品。
```

---

### 3.2 blog.tanteng.space

#### (a) 首页 description

**现在**（35 字）
```
记录生活与成长的空间。热爱摄影、旅行、技术，这里有技术笔记与生活随想。
```

**改成**（82 字）
```
谈腾（Tony老师）的技术博客，累计 274 篇：AI Agent、GraphRAG、大模型、Golang、K8S 实战笔记，兼有读书笔记、期权投资与胶片摄影随笔。
```

改动逻辑：整句从"描述氛围"变成"**清单式承诺**"。"累计 274 篇"是**信任信号**（数字让摘要可信度显著提升），也顺手解决"这站值不值得点"的问题。"期权投资"和"胶片摄影"是从你 about 页和文章列表里挖出来的**差异化标签**——纯技术博客遍地都是，同时写期权和胶片的技术人很少。

> 想改成 35 字以内的短版本？那不如不要。短 description 会被搜索引擎**截取页面正文自动补全**，结果不可控。宁可自己写满。

#### (b) 首页 og:description（社交分享）

**现在**：`Hi，我是Tony老师` — 11 字，微信/Twitter 分享出去几乎是一片空白。

**改成**
```
后台开发工程师，写 AI Agent、大模型与 Golang；也拍胶片、研究期权。274 篇技术笔记与生活随笔。
```

`og:description` 和 `meta description` **不必相同**。meta 面向搜索引擎（要关键词），og 面向点开分享链接的人（要钩子）。

#### (c) 关于我页 description

**现在**：整页正文 209 字被硬塞进 description，含 emoji、换行、未去 Markdown。

**改成**（84 字）
```
谈腾（Tony老师），后台开发工程师，关注 Golang、K8S、AI 与 Agent。常驻深圳，来自湖北黄石。用索尼 A7M4、尼康 FM2 与宾得 17 记录旅途。
```

> "关于我"页往往是人名搜索的**最佳落地页**（比首页更像"这个人的档案"）。别浪费它。

---

### 3.3 photos.tanteng.space

**现在**（5 字）
```
滴，咔嚓！
```

**改成**（69 字）
```
Tony老师的相册：用索尼 A7M4 与胶片相机记录深圳、香港与旅行途中的人和风景。大澳渔村、赤柱海湾、浅水湾、小梅沙，按年份与地点归档。
```

改动逻辑：

| 元素 | 作用 |
|---|---|
| `Tony老师的相册` | 品牌词，和枢纽站/博客保持**同一个名字**（一致性是实体识别的关键） |
| `索尼 A7M4 与胶片相机` | 器材搜索是摄影站的隐藏流量池（"索尼 A7M4 样片""胶片 扫街"）。你站内已有 `a7m4`、`delta` 等器材标签 |
| `深圳、香港` | 地理长尾的主力 |
| `大澳渔村、赤柱海湾、浅水湾、小梅沙` | **具体地名 > 泛泛而谈**。现在你首页展示的就是这批照片，description 里点名，命中率立刻不同 |
| `按年份与地点归档` | 告诉搜索引擎站内结构，利于分类页被收录 |

**备选（偏胶片/器材，若想主打这条线）**
```
用索尼 A7M4、尼康 FM2 与宾得 17 记录的摄影相册：深圳、香港与旅行途中。大澳渔村、赤柱海湾、浅水湾，含胶片扫街与城市风光。
```

---

## 四、description 写作方法（可复用）

你自己写的时候，记住三段式。**顺序不能换**：

| 段 | 内容 | 长度 | 例子（枢纽站） |
|---|---|---|---|
| ① 实体锚定 | 真名 + 别名 + 身份 + 地域 | 15–25 字 | 谈腾（Tony老师），腾讯后台开发工程师，常驻深圳 |
| ② 意图匹配 | 用户真会搜的主题词，逗号罗列 | 30–40 字 | AI Agent、Golang、K8S 技术笔记 |
| ③ 差异化钩子 | 具体地名 / 器材 / 数字，制造"只有你有" | 15–25 字 | 深圳、香港与旅行摄影作品 |

**关键词从哪来**（别靠猜）：

1. **Google Search Console → 效果 → 查询**：你已经在被搜的词，直接抄进 description
2. **百度搜索资源平台 → 搜索分析 → 关键词**：你已提交了 `baidu-site-verification`，数据现成
3. **Bing Webmaster Tools**：国内用户量小但数据干净
4. **站内搜索日志**：博客的 `/search/` 页有没有记录查询词？那里是**最真实的读者意图**

**三条禁令**：

- ❌ 不要在 description 里堆关键词（"博客,技术,AI,Go,K8S,摄影,旅行,深圳..."）——百度的关键词堆砌识别已经很成熟，而且会挤掉真正能打动人的句子
- ❌ 不要写成自我评价（"热爱生活、积极向上"）——搜不到，也不吸引点击
- ❌ 不要三站用同一句——会触发**重复内容**判断，三站互相稀释权重

---

## 五、连带发现的 8 个 SEO 问题（比 description 更影响收录）

### 🔴 P0 — sitemap 与 robots 全面失守

| 检查项 | 结果 |
|---|---|
| `tanteng.space/sitemap.xml` | **404**（且 `robots.txt` 里的 `Sitemap:` 行被注释掉了） |
| `blog.tanteng.space/robots.txt` | **404** — 返回的是 Hugo 的 404 HTML 页（5.4 KB） |
| `photos.tanteng.space/robots.txt` | **404** — 返回的是 Next.js 404 页（**68 KB**！） |
| `blog.tanteng.space/sitemap.xml` | ✅ 200（89 KB） |
| `photos.tanteng.space/sitemap.xml` | ✅ 200（104 KB） |

**为什么要紧**：爬虫每次请求 `/robots.txt` 都会拿到一整个 HTML 页面，白烧抓取预算；没有 robots.txt 就无法声明 sitemap，两个 sitemap 现在只能靠搜索引擎"碰巧"发现。

**修复**：

**(1) 新建 `blog.tanteng.space/robots.txt`** —— 放到 `blog/static/robots.txt`：
```
User-agent: *
Allow: /

Sitemap: https://blog.tanteng.space/sitemap.xml
```

**(2) 新建 `photos.tanteng.space/app/robots.ts`**：
```ts
import type { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: '*', allow: '/' },
    sitemap: 'https://photos.tanteng.space/sitemap.xml',
  };
}
```

**(3) 新建 `tanteng.space/sitemap.xml`**：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://tanteng.space/</loc>
    <lastmod>2026-09-12</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```
同时把 `robots.txt` 里那行注释打开：
```
Sitemap: https://tanteng.space/sitemap.xml
```

> ⚠️ `Sitemap:` 指令**只能声明同域**的 sitemap。三份 sitemap 要各自在自己域的 robots.txt 里声明，然后去 GSC / 百度资源平台**手动提交三次**。

---

### 🔴 P0 — blog 首页的 twitter 卡片是空的

线上实测：
```html
<meta name=twitter:title content>
<meta name=twitter:description content>
```
两个属性**内容为空**。

**根因**：`layouts/partials/twitter_cards.html` 用的是 `{{ .Title }}` 和 `{{ .Summary }}`，但 `content/_index.md` 里 `title: ""`（这是为了首页不渲染大标题），首页 `.Title` 因此为空，`.Summary` 也是空。

**修复** —— 改 `layouts/partials/twitter_cards.html`：
```html
<meta name="twitter:title" content="{{ if .IsPage }}{{ .Title }}{{ else }}{{ .Site.Title }}{{ end }}" />
<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{ if .IsPage }}{{ .Summary | plainify | truncate 160 }}{{ else }}{{ .Site.Params.description }}{{ end }}{{ end }}" />
```

---

### 🔴 P0 — 关于我页的 description 溢出

**根因**：`layouts/baseof.html` 第 9 行
```html
<meta name="description" content="{{ with .Description }}{{ . }}{{ else }}{{if .IsPage}}{{ .Summary }}{{ else }}{{ with .Site.Params.description }}{{ . }}{{ end }}{{ end }}{{ end }}">
```
`.Summary` **没有 `plainify`、没有 `truncate`**，所以整篇正文（含 Markdown 残留和 emoji）直接灌进去。

**修复**：
```html
<meta name="description" content="{{ with .Description }}{{ . }}{{ else }}{{ if .IsPage }}{{ .Summary | plainify | truncate 160 }}{{ else }}{{ .Site.Params.description }}{{ end }}{{ end }}">
```
同时给 `content/about.md` front matter 补一行 `description:`（用上文 3.2(c) 的文案），从源头解决。

---

### 🟠 P1 — 83 / 274 篇文章没有 description

占据总文章数 **30%**。这些页面现在全部落到 `.Summary` 兜底，摘要质量参差。

**建议**：优先补近两年 + 有流量的文章。历史上那批 `reading-list`（2018 年的周报式文章）价值低，可以不做，甚至考虑 `noindex`。

---

### 🟠 P1 — tanteng.space 首页是 thin content

首页可索引正文**仅约 165 个中文字符**，而且几乎全是导航文字和文章标题。H1 `Tony(谈腾)` 下面直接就是一句 tagline。

对个人枢纽页来说，这会导致：搜索引擎**没有足够文本判断这页讲什么**，也就无法为"谈腾"这类人名查询生成好的摘要。

**建议**：在 hero 和"最新文章"之间，加一段 **150–250 字的自我介绍正文**（不要用图片承载）。可以就从 about 页改写：

```html
<section class="intro">
  <p>我是谈腾（Tony），后台开发工程师，现居深圳，来自湖北黄石。
  平时关注 AI Agent、大模型、Golang 与 K8S，把技术沉淀写成博客，目前已有 270 余篇，
  涵盖 AI Agent 工程、GraphRAG、系统设计与读书笔记。</p>
  <p>工作之外带着索尼 A7M4、尼康 FM2 和宾得 17 到处拍照，
  深圳、香港和旅行途中的照片都收在相册里。也研究期权策略，把投资思考一并写在博客。</p>
</section>
```

这段文字同时喂给三处：人类读者、搜索引擎摘要、`Person` 结构化数据的语义佐证。

---

### 🟠 P1 — og:image 体积偏大

| 文件 | 体积 | 问题 |
|---|---|---|
| `tanteng.space/og-image.png` | **677 KB** | 微信、Twitter 抓取 og 图有超时窗口，大于 300 KB 有失败风险 |
| `photos.tanteng.space/home-image` | **512 KB**（PNG） | 同上 |

**建议**：都压到 **150 KB 以内**，尺寸严格 1200×630。PNG 转 JPEG（og 图不需要透明通道）：

```bash
# 先看原图尺寸，再压
magick og-image.png -resize 1200x630^ -gravity center -extent 1200x630 -quality 85 og-image.jpg
```

---

### 🟡 P2 — blog 的结构化数据太薄

`layouts/partials/schema.html` 现在只输出一个 `WebSite`：
```json
{"@context":"https://schema.org","@type":"WebSite","name":"Tony老师的博客","url":"https://blog.tanteng.space/"}
```

对比之下，**tanteng.space 的 JSON-LD 做得非常好**（`WebSite` + `Person` + `ImageGallery` + `ImageObject` + `BlogPosting` + 导航），可以直接当模板抄过来。

**blog 端建议补**：
- `Person` 节点，并加 `sameAs` 指向 `tanteng.space`、`photos.tanteng.space`、Instagram —— **这是把三个站绑成同一个实体的关键**
- 文章页补 `BlogPosting`，带 `author: {@id: .../#person}`、`datePublished`、`headline`

**photos 端建议补**：`ImageGallery` + `Photograph`，把已有的人物/地点/标签映射成 `keywords`。

---

### 🟡 P2 — 三站实体信息不一致

| 字段 | tanteng.space | blog | about 页 |
|---|---|---|---|
| 名字 | Tony老师的个人空间 | Tony老师的博客 | 关于我 |
| 身份 | Developer / 后台开发工程师 | 未提 | 后台开发工程师 |
| 地域 | 深圳 | 未提 | 深圳（来自湖北黄石） |

搜索引擎建立"实体"靠的是**多点一致的信号**。建议统一为一套：

- **名字**：谈腾 / Tony / Tony老师 / tanteng（一致出现在三站）
- **身份**：后台开发工程师（**别再混用 "Software Developer" 和 "Developer"**，中英混用会削弱匹配）
- **地域**：深圳
- **`sameAs` 三站互链**：tanteng.space ↔ blog ↔ photos 三向 `sameAs`，形成闭环

---

## 六、落地清单（按优先级）

### 立刻做（1 小时内）
1. ✅ 改 `tanteng.space/index.html` 的 `<meta name="description">` 和 `og:description`
2. ✅ 改 `blog/content/_index.md` 加 `description:`，`config.toml` 里 `description` 和 `open_graph_description` 同步更新
3. ✅ 改 `blog/content/about.md` 加 `description:`
4. ✅ 相册站：`NEXT_PUBLIC_SITE_DESCRIPTION` 环境变量 / 后台 `/admin` 配置里的 `metaDescription`

### 本周做
5. ✅ 新建 `blog/static/robots.txt`、`photos/app/robots.ts`、`tanteng.space/sitemap.xml` 及 robots.txt 的 Sitemap 行
6. ✅ 修 `blog/layouts/baseof.html` 第 9 行（加 `plainify | truncate 160`）
7. ✅ 修 `blog/layouts/partials/twitter_cards.html`（首页 title/description 兜底）
8. ✅ 压缩两张 og 图到 150 KB 内
9. ✅ GSC + 百度资源平台**手动提交 3 份 sitemap**

### 这个月做
10. ⬜ tanteng.space 首页加 150–250 字自我介绍正文
11. ⬜ 补近两年文章的 description（优先有 GSC 曝光但 CTR 低的）
12. ⬜ blog 补 `Person` / `BlogPosting` 结构化数据，三站 `sameAs` 互链
13. ⬜ 相册站补 `ImageGallery` / `Photograph` 结构化数据
14. ⬜ 统一三站名字/身份/地域表述

### 观察 2–4 周
15. ⬜ GSC 看**展现量**是否上涨（description 改动的效果先体现在展现量，再体现到点击率）
16. ⬜ 百度资源平台看收录数变化
17. ⬜ 三站各自 CTR 变化（目标：改 description 后 CTR 提升 20%+）

---

## 附：本文所有改动的文案总表

| 位置 | 优化后文案 | 字数 |
|---|---|---|
| tanteng.space `<meta description>` | 谈腾（Tony老师），腾讯后台开发工程师，常驻深圳。汇总我的技术博客与摄影相册：AI Agent、Golang、K8S 技术笔记，深圳、香港与旅行摄影作品。 | 78 |
| tanteng.space `og:description` | 同上（或换更口语的版本） | 78 |
| blog 首页 `<meta description>` | 谈腾（Tony老师）的技术博客，累计 274 篇：AI Agent、GraphRAG、大模型、Golang、K8S 实战笔记，兼有读书笔记、期权投资与胶片摄影随笔。 | 82 |
| blog 首页 `og:description` | 后台开发工程师，写 AI Agent、大模型与 Golang；也拍胶片、研究期权。274 篇技术笔记与生活随笔。 | 55 |
| blog about `<meta description>` | 谈腾（Tony老师），后台开发工程师，关注 Golang、K8S、AI 与 Agent。常驻深圳，来自湖北黄石。用索尼 A7M4、尼康 FM2 与宾得 17 记录旅途。 | 84 |
| photos `<meta description>` | Tony老师的相册：用索尼 A7M4 与胶片相机记录深圳、香港与旅行途中的人和风景。大澳渔村、赤柱海湾、浅水湾、小梅沙，按年份与地点归档。 | 69 |
