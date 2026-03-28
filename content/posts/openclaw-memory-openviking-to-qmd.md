---
title: "OpenClaw 记忆后端切换：从 OpenViking 到 QMD"
date: 2026-03-27
description: "记录 OpenClaw 记忆系统从 OpenViking 切换到 QMD 的完整过程，包括切换原因、数据迁移、踩坑记录。不是 OpenViking 不好，而是 QMD 更适合当前阶段。"
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

### 第四步：构建 QMD 索引

这一步最容易踩坑。直觉是"把整个 workspace 扔进去"，但实际上需要仔细想想**哪些内容该索引**。

#### 先说结论

不要把整个 `~/.openclaw/workspace` 加到 QMD。我的 workspace 下有 462 个 Markdown 文件，包括博客文章、技能文档、demo 项目等。全部索引进去会有两个问题：

1. **噪音太大** —— 搜索"我对 Go 怎么看"，可能返回一篇我转载的别人的观点
2. **稀释个人记忆** —— 真正有价值的记忆（身份信息、偏好、对话日志）被博客文章淹没

#### 最佳实践：按内容性质拆分 Collection

最终方案是创建两个精准的 collection：

```bash
# Collection 1: 个人记忆文件
# 在 /root/memories/ 目录下集中管理核心记忆
mkdir -p /root/memories

# 把 OpenClaw 核心身份文件复制过来
cp ~/.openclaw/workspace/{USER,SOUL,IDENTITY,MEMORY}.md /root/memories/

# 如果有自己写的笔记、分析报告也放这里
# cp ~/.openclaw/workspace/PDD_stock_analysis_*.md /root/memories/

qmd collection add /root/memories --name memories
# → 8 files indexed

# Collection 2: OpenClaw 每日对话日志
qmd collection add ~/.openclaw/workspace/memory --name oc-memory
# → 100 files indexed (每天的对话日志)
```

这样 QMD 里总共 108 个文件，全部是**属于我的**内容，没有博客转载文章和第三方技能文档的噪音。

#### 为什么不能用 ignore 排除

你可能想："直接索引 workspace，用 ignore 排除不需要的目录不就好了？"

我试过了。QMD 的 `qmd.yaml` 配置里有 `ignore` 字段，理论上支持 glob 模式排除。但实测（QMD 2.0.1）**collection 级别的 ignore 不生效**，`--glob` 参数也无法限制匹配范围——只要 `collection add` 指向一个目录，它就用 `**/*.md` 扫全部。

所以最可靠的方式还是：**用独立目录管理要索引的文件，只把需要的内容放进去。**

#### 生成向量嵌入

```bash
# 生成向量（CPU 模式，108 个文件约 11 分钟）
qmd embed
```

最终状态：

```
Documents
  Total:    108 files indexed
  Vectors:  397 embedded
Collections
  memories   → 8 files (个人记忆)
  oc-memory  → 100 files (对话日志)
```

### 第四步半：设置自动同步

QMD 不会监听文件变化，OpenClaw 更新了 `USER.md` 或新增了对话日志，QMD 索引不会自动刷新。需要一个 crontab 定时同步：

```bash
crontab -e
```

添加：

```
0 4 * * * bash -c 'source /root/.nvm/nvm.sh && cp /root/.openclaw/workspace/{USER,SOUL,IDENTITY,MEMORY}.md /root/memories/ 2>/dev/null && qmd update && qmd embed' >> /root/qmd-sync.log 2>&1
```

每天凌晨 4 点：
1. 把 OpenClaw 最新的核心文件同步到 memories 目录
2. `qmd update` 扫描文件变化
3. `qmd embed` 对新增/修改的内容生成向量

#### QMD 在 OpenClaw 记忆体系中的角色

搞清楚这一点很重要——**QMD 是搜索引擎，不是存储引擎**。

OpenClaw 每次对话的记忆加载流程：

```
固定读取（每次都读，不经过 QMD）
├── SOUL.md      → AI 身份
├── USER.md      → 用户信息
├── IDENTITY.md  → 身份配置
└── memory/最近几天.md → 近期对话上下文

按需检索（通过 QMD）
└── 当对话涉及的内容不在上述文件时
    → QMD 语义搜索 → 召回相关片段 → 注入上下文
```

核心文件 OpenClaw 每次直接读取，不依赖 QMD。QMD 的价值在于当记忆文件多到几百个时，帮 AI 从海量历史中**精准定位**相关内容，而不是把所有文件都塞进上下文窗口。

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

1. **不要把整个 workspace 扔进 QMD** —— 按内容性质拆分 collection，只索引真正属于你的记忆
2. **QMD 是搜索引擎，不是存储引擎** —— 核心记忆文件 OpenClaw 每次直接读取，QMD 只在需要从大量历史中检索时才介入
3. **一定要设自动同步** —— QMD 不监听文件变化，crontab 每天跑一次 `qmd update && qmd embed` 是必须的
4. **中文搜索先别指望太多** —— EmbeddingGemma-300M 对中文支持有限，但 OpenClaw 的 query expansion 能缓解这个问题
5. **CPU 限速很重要** —— 2 核云主机上不限速 embed 会把服务器跑挂

折腾的过程本身也是学习——搞清楚了 OpenClaw 的 `contextEngine` 架构、插件 slot 机制、记忆搜索后端的切换方式，以及 QMD 的 collection 管理和向量嵌入流程。这些理解比用哪个工具更有价值。
