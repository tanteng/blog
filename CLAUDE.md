# Claude Code 项目规范

## 项目概述

这是一个 Hugo 博客项目，使用 Hugo 静态站点生成器。

## 技术栈

- **框架**: Hugo
- **主题**: Ananke
- **部署**: GitHub Actions → 腾讯云 COS → 腾讯云 EdgeOne CDN
- **语言**: 中文为主

## 工作流程

### 文件命名规范

1. **文件名不带日期**：文章文件名不包含日期，如 `ddd-layered-architecture-dependency.md`，而非 `2021-04-01-ddd-layered-architecture-dependency.md`
2. **front matter 的 `url` 字段**中包含日期路径，如 `url: /2021/04/ddd-layered-architecture-dependency/`

| 分类 slug | 说明 |
|------|------|
| `tech` | 技术文章（编程、架构等） |
| `reading-notes` | 读书笔记、心得感悟 |
| `ai` | AI/机器学习相关 |
| `life` | 生活随想 |
| `investment` | 投资理财 |
| `photo` | 摄影 |
| `science` | 科学 |
| `art` | 艺术 |
| `technews` | 科技新闻 |

**规范：**
1. **slug 尽量用已有英文 slug**，不要新建中文或自造英文
2. **分类目录**：`content/categories/<slug>/_index.md`，front matter 的 `title` 字段设中文展示名

### 文章标签规范

1. **tag 使用已有英文 slug**，不要新建中文或自造英文 slug（如腾讯云用 `tencent-cloud`，不是 `tengxunyun`）
2. **标签数量控制在 4-5 个**
3. **标签目录**：`content/tags/<slug>/_index.md`，front matter 的 `title` 字段设中文展示名
4. **常用标签参考**：`ai`, `rag`, `openviking`, `agent`, `tencent-cloud`, `openclaw`, `hugo`, `life`, `reading`, `photography`, `investment` 等

### Mermaid 图表

使用 `{{< mermaid >}}...{{< /mermaid >}}` shortcode 渲染图表，**不要**使用 ` ```mermaid ` 代码块。Ananke 主题通过 shortcode 触发 `hasMermaid` 标记来条件加载 mermaid JS，代码块无法触发此机制。

### 文章修改规范

1. **Steve Jobs 系列文章**属于 `reading-notes`，不是 `tech`
2. **阅读清单系列**使用数字列表 + 文字链接格式
3. **翻译文章**保留英文原文，中文翻译紧随其后
4. **删除多余署名**，保持简洁
5. **控制 `<hr>` 换行线和段落内空行**，避免滥用，保持内容紧凑
6. **featured_image 非必填**，无需设置时不写该字段，使用主题默认封面

### Git 提交规范

```bash
# 提交格式
git add <files>
git commit -m "描述"

# 示例
git commit -m "fix: 修正文章分类标签"
```

### 部署流程

Push 到 GitHub → 触发 GitHub Actions → 同步到腾讯云 COS → 清除腾讯云 EdgeOne 缓存

### 本地预览

```bash
hugo server
# 访问 http://localhost:1313
```

## Superpowers 使用

本项目已安装 Superpowers 插件，开始任何任务前会先进行 brainstorming。

## 注意事项

- `.claude/` 和 `.workbuddy/` 目录不提交到 git
- 修改前先确认文章当前内容
- 引用书籍内容时使用原文，不要杜撰

### Hugo Partial 路径规范

**Hugo partials 必须放在 `layouts/_partials/` 目录**，不是 `layouts/partials/`。

- `layouts/_partials/` = Hugo 识别的正确路径
- `layouts/partials/` = 会被忽略（除非主题特殊处理）

**调试时必须同时测试生产构建：**

```bash
npm run build  # 构建到 public/，必须检查实际输出
hugo server -D # 开发模式可能有容错机制
```

本地开发时 `hugo server` 能"容错"跳过错误路径，但生产构建 `hugo --minify` 会严格遵循路径规范，导致修改不生效。
