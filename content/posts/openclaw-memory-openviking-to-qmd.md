---
title: "OpenClaw 记忆后端切换：从 OpenViking 到 QMD"
date: 2026-03-30
description: "记录 OpenClaw 记忆系统从 OpenViking 切换到 QMD 的完整过程。包括切换原因、数据迁移、QMD 自动管理机制、记忆加载流程解析、踩坑记录，以及 QMD 在 OpenClaw 记忆体系中的实际意义。"
categories: ['tech']
tags: ['openclaw', 'openviking', 'qmd', 'ai', 'memory', 'context-engineering']
featured_image: ""
---

昨天刚写完 [OpenViking × OpenClaw 的集成文章](/posts/openviking-openclaw-memory-dual-write/)，今天就把它换掉了。不是翻脸，是想通了一些事。

<!--more-->

## 为什么换

先说清楚：**OpenViking 本身没问题**。字节的 Viking 团队做得很扎实，向量检索能力强，记忆提取也聪明。用了两天，auto-recall 确实比 OpenClaw 原生的 FTS 搜索精准不少。

但有几个现实问题让我决定先切回来：

**1. 额外的模型费用**

OpenViking 需要两个外部模型：
- **Embedding 模型**（doubao-embedding-vision）—— 把文本转向量
- **VLM 模型**（doubao-seed-2-0-pro）—— 提取记忆摘要

这两个都走火山引擎的 API，虽然单价不贵，但日积月累是一笔开销。对于个人项目来说，能省则省。

**2. 供应商锁定的隐忧**

OpenViking 接管了 OpenClaw 的 `contextEngine` slot，成为记忆系统的唯一入口。一旦启用，OpenClaw 原生的 Markdown 记忆就不再更新。如果哪天想切回来或者换别的方案，中间积累的记忆就会断档。

虽然我写了个同步脚本（每小时从 OpenViking API 拉数据到本地 Markdown），但这终究是补丁，不是正路。

**3. QMD 足够好，而且免费**

QMD（Quantum Memory Database）是 Shopify 创始人 Tobi Lütke 开发的本地语义搜索引擎，OpenClaw 从 2026.2.2 版本开始内置支持。它的核心优势：

- **纯本地运行**，不调用任何外部 API
- **零费用**，模型跑在本机 CPU 上
- **混合搜索**：BM25 关键词 + 向量语义 + LLM 重排序
- **不改变存储方式**：记忆还是 Markdown 文件，QMD 只是换了搜索引擎

最关键的一点：**记忆数据始终在本地**，随时可以切换后端，没有锁定风险。

## 切换过程

### 第一步：安装 QMD

```bash
npm install -g @tobilu/qmd
qmd --version  # 2.0.1
```

### 第二步：修改 OpenClaw 配置

编辑 `~/.openclaw/openclaw.json`：

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "legacy"  // 从 openviking 切回 legacy
    }
  },
  "memory": {
    "backend": "qmd",
    "qmd": {
      "limits": {
        "timeoutMs": 8000
      }
    }
  }
}
```

两个关键改动：
- `contextEngine` 切回 `legacy`，让 OpenClaw 恢复原生的上下文管理
- `memory.backend` 设为 `qmd`，用 QMD 替代内置的 SQLite FTS 搜索

### 第三步：迁移 OpenViking 的记忆

OpenViking 的记忆存储在自己的向量库里，切回 legacy 后这些记忆就"看不见"了。好在我之前部署了一个同步脚本，已经把 71 条记忆全量备份到了本地。

把备份的记忆整合成 OpenClaw 原生格式：

```bash
# 同步脚本已经把数据拉到了 memory/openviking-sync/ 下
# 按分类合并成可索引的 Markdown 文件
# → memory/openviking-entities.md  (33 条)
# → memory/openviking-events.md    (31 条)  
# → memory/openviking-preferences.md (7 条)
```

这样 QMD 就能索引到这些历史记忆了。

### 第四步：QMD 索引——你不需要手动管理

~~这一步最容易踩坑。~~ 其实这一步**不需要你做什么**。

我最初在命令行手动折腾了一堆 `qmd collection add`、crontab 同步、memories 目录管理，后来发现全是多余的。

#### OpenClaw 自动管理 QMD 索引

查看 [官方文档](https://docs.openclaw.ai/reference/memory-config) 才搞清楚：**OpenClaw 会自己管理 QMD 的索引、更新和嵌入。** 它在启动 QMD 时设置了独立的 `XDG_CONFIG_HOME` 和 `XDG_CACHE_HOME`，指向自己的数据目录：

```
~/.openclaw/agents/main/qmd/
├── xdg-config/
│   ├── index.yml          ← 记忆文件索引配置（自动生成）
│   └── qmd/index.yml      ← workspace 索引配置（自动生成）
└── xdg-cache/
    └── qmd/index.sqlite   ← QMD 索引数据库
```

OpenClaw 自动生成的配置内容：

```yaml
# index.yml（记忆专用索引）
collections:
  memory-root-main:
    path: ~/.openclaw/workspace
    pattern: MEMORY.md
  memory-dir-main:
    path: ~/.openclaw/workspace/memory
    pattern: "**/*.md"

# qmd/index.yml（workspace 索引）
collections:
  workspace:
    path: ~/.openclaw/workspace
    pattern: "**/*.md"
    ignore:
      - "**/blog/content/posts/**/*.md"
      - "**/blog/themes/**"
```

注意看，**它已经自动排除了博客文章和主题文件**。

#### 自动更新和嵌入

根据官方文档，OpenClaw 的 QMD 管理器会：

- **启动时**执行一次 `qmd update` 和 `qmd embed`（后台运行，不阻塞聊天）
- **每 5 分钟**自动刷新一次索引和嵌入
- 通过 `memory.qmd.update.interval` 可以调整刷新频率

所以**不需要 crontab，不需要手动 `qmd update`，不需要手动 `qmd embed`**。OpenClaw 全自动搞定。

#### 命令行的 QMD 是独立的

这里有个容易混淆的点：你在终端里敲 `qmd` 和 OpenClaw 内部调用的 `qmd` 是**同一个程序，但用不同的数据目录**。

| | 索引路径 | 谁管理 |
|---|---|---|
| 命令行 `qmd` | `~/.cache/qmd/index.sqlite` | 你自己 |
| OpenClaw 的 `qmd` | `~/.openclaw/agents/main/qmd/xdg-cache/qmd/index.sqlite` | OpenClaw 自动管理 |

所以你在命令行 `qmd collection add` 加的东西，OpenClaw 根本看不到。反过来也一样。

#### 如果想自定义索引范围

可以在 `openclaw.json` 里通过 `memory.qmd.paths` 添加额外的目录：

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "includeDefaultMemory": true,
      "paths": [
        {
          "name": "notes",
          "path": "~/my-notes",
          "pattern": "**/*.md"
        }
      ]
    }
  }
}
```

`includeDefaultMemory` 默认为 `true`，会自动索引 `MEMORY.md` 和 `memory/**/*.md`。

## QMD 在 OpenClaw 记忆体系中的角色

搞清楚这一点很重要——**QMD 是按需语义搜索引擎，不是存储引擎，也不是每次对话都会被调用。**

### OpenClaw 的记忆加载流程

```
会话启动时（每次都执行，不经过 QMD）：
├── 读取 MEMORY.md        ← 长期记忆（仅私聊加载，~3 KB）
├── 读取 memory/今天.md    ← 今日对话日志
├── 读取 memory/昨天.md    ← 昨日对话日志
└── 注入 system prompt

对话过程中（模型自行判断是否调用）：
├── memory_get    → 读取指定文件内容（不经过 QMD）
└── memory_search → 语义搜索（调用 QMD）
```

注意：`SOUL.md`、`USER.md`、`IDENTITY.md` 这些核心身份文件是作为 system prompt 的一部分直接加载的，不走记忆系统。

### QMD 的实际意义

启动时加载的只有 3 个文件，加起来可能不到 10 KB。那 QMD 的价值在哪里？

**在于让 AI 能精准访问到剩下几百个文件的内容，而只消耗极少的 token。**

举个例子：

```
你问："我两周前分析过 PDD 的股票吧？"

情况 A（没有 QMD）：
  → 只有 MEMORY.md + 今天 + 昨天的日志
  → 两周前的信息不在上下文里
  → AI 回答："我不记得了"

情况 B（有 QMD）：
  → 模型判断：这个信息不在已加载的上下文中
  → 调用 memory_search("PDD stock analysis")
  → QMD 从所有历史日志中语义检索
  → 返回最相关的 2-3 个片段（~1 KB）
  → AI 基于召回内容准确回答
```

一句话概括：**启动时读小文件保证基础记忆，运行时靠 QMD 按需检索保证深度记忆。**

### 什么时候 QMD 不会被调用

如果你问的信息已经在启动时加载的文件里，模型不需要调用 `memory_search`，QMD 就不会介入。这也是我实测发现的——最近几天的对话 session 里 QMD 调用次数为 0，因为常用信息都在 MEMORY.md 和 USER.md 里。

这不是 bug，是设计如此。**不需要搜的时候就不搜，省 token。**

### 第五步：卸载 OpenViking

```bash
# 停掉 systemd 服务
systemctl stop openviking
systemctl disable openviking

# 从 OpenClaw 配置中移除
# 删除 plugins.entries.openviking

# 重启
openclaw gateway restart
```

## 踩的坑

### 1. OpenViking 的"幽灵进程"

禁用 openviking 插件后，以为就完事了。结果发现 OpenViking server 还在后台跑着吃 CPU。

原因有两个：
- openviking 插件的 `registerService` 在加载时就注册了子进程管理，`enabled: false` 只是不加载插件逻辑，但进程管理可能已经注册了
- OpenViking 有自己的 **systemd 服务**（`openviking.service`），独立于 OpenClaw gateway，重启 gateway 根本杀不掉它

解决：必须同时 `systemctl disable openviking` + 删除插件目录，双管齐下。

### 2. QMD embed 打满 CPU

QMD 生成向量嵌入时，会下载一个 ~330MB 的本地模型（embeddinggemma-300M），然后在 CPU 上跑推理。我的服务器是 2 核云主机，没有 GPU，462 个文件一起 embed 直接把 CPU 干到 100%，服务器差点失联。

解决：用 `cpulimit` 工具限制 QMD 的 CPU 使用率：

```bash
# 安装 cpulimit
yum install -y cpulimit

# 限制到 50% CPU
cpulimit -p $(pgrep -f 'qmd.js embed') -l 50 -b

# 或者启动时直接限制
cpulimit -l 50 -- qmd embed
```

另外要注意：**不要同时跑多个 `qmd embed` 进程**。多个实例会竞争同一个 SQLite 数据库，互相卡死，日志不更新但进程还在。如果不确定，先 `pkill -f 'qmd.js embed'` 杀干净再重新跑。

### 3. Vulkan 编译失败

QMD 内置的 llama.cpp 默认尝试用 Vulkan GPU 加速。我的云服务器没有 GPU，编译 Vulkan 后端直接报错。不过 QMD 会自动回退到 CPU 模式，功能不受影响，只是速度慢。

每次运行 `qmd status` 或 `qmd embed` 都会输出一大段 CMake 编译错误信息，看着吓人但不影响使用，可以忽略。

### 4. 中文语义搜索效果差

QMD 默认使用 EmbeddingGemma-300M 作为 embedding 模型，这是一个**偏英文的小模型**。实测发现：

| 查询 | 结果 |
|------|------|
| `qmd search 'tech stack'` | ✅ 命中，77% 匹配 |
| `qmd search 'preferences'` | ✅ 命中，63% 匹配 |
| `qmd search '我是谁'` | ❌ 无结果 |
| `qmd search '软件工程师'` | ❌ 无结果 |

纯中文的语义查询基本搜不到东西。但在实际使用中影响有限——因为 OpenClaw 调用 QMD 时会先通过 LLM 做 **query expansion**，把中文意图扩展成适合检索的关键词（通常包含英文），然后再查 QMD。

如果介意，可以在写记忆文件时**中英双语并行**：

```markdown
# About Me / 关于我
I am tanteng, a software engineer based in Shenzhen.
我是 tanteng，深圳的软件工程师。
```

### 5. nohup 运行 qmd embed 可能崩溃

用 `nohup qmd embed > log 2>&1 &` 在后台跑，有时会出现 `EBADF` 错误直接崩溃。这可能是 node-llama-cpp 的文件描述符和 nohup 的重定向机制冲突。

更稳定的方式是前台跑（保持终端开着），或者用 `screen` / `tmux`：

```bash
screen -S qmd
qmd embed
# Ctrl+A, D 分离
```

## 对比总结

| | OpenViking | QMD |
|---|---|---|
| 搜索质量 | ⭐⭐⭐⭐⭐ 向量+VLM | ⭐⭐⭐⭐ BM25+向量+重排序 |
| 费用 | 💰 embedding + VLM API | 🆓 完全免费 |
| 部署复杂度 | 中（Python 服务 + 配置） | 低（一行 npm install） |
| 记忆存储 | 自有向量库 | 本地 Markdown（原生） |
| 供应商锁定 | 有（切换需数据迁移） | 无（随时切后端） |
| Token 节省 | 好 | 好（官方称降 90-99%） |
| GPU 需求 | 不需要（远程 API） | 不需要（CPU 可跑，GPU 更快） |

## 最后

技术选型没有绝对的好坏。OpenViking 适合追求极致搜索质量、不在意 API 费用的场景；QMD 适合个人用户、想保持简单和免费的场景。

对我来说，当前阶段 QMD 的"够用 + 免费 + 无锁定"更有吸引力。等哪天真的需要更强的语义理解能力了，OpenViking 的数据还在服务器上，随时可以切回去。

几个实操经验总结：

1. **QMD 索引不需要手动管理** —— OpenClaw 自动生成配置、自动每 5 分钟刷新索引和嵌入。不需要 crontab，不需要手动 `qmd update`。我最初折腾了一堆手动 collection 方案，后来发现全是多余的
2. **命令行 `qmd` 和 OpenClaw 的 `qmd` 是隔离的** —— 同一个程序，不同的数据目录。在终端里操作 QMD 不会影响 OpenClaw，反之亦然
3. **QMD 不是每次对话都触发** —— 只有模型判断需要从历史记忆中搜索时才调用 `memory_search`。常用信息在 MEMORY.md 和当天日志里就能找到，不需要 QMD
4. **QMD 的意义是"深度记忆"** —— 启动时加载 3 个小文件（MEMORY.md + 今天 + 昨天）保证基础记忆，运行时靠 QMD 按需检索保证对几百个历史文件的访问能力
5. **中文搜索效果有限** —— EmbeddingGemma-300M 对中文支持不好，但 OpenClaw 会做 query expansion 缓解
6. **CPU 限速很重要** —— 2 核云主机上首次 embed 几百个文件会打满 CPU，用 `cpulimit` 限速或者 `screen` 前台跑

折腾的过程本身也是学习——搞清楚了 OpenClaw 的记忆加载流程（核心文件直接读 + QMD 按需搜索）、`contextEngine` 架构、QMD 自动管理机制。踩的坑也算值了。
