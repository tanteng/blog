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

```bash
# 设置环境变量，对齐 OpenClaw 的数据目录
export XDG_CONFIG_HOME=~/.openclaw/agents/main/qmd/xdg-config
export XDG_CACHE_HOME=~/.openclaw/agents/main/qmd/xdg-cache

# 添加 workspace 为索引集合
qmd collection add ~/.openclaw/workspace

# 更新索引
qmd update

# 生成向量嵌入
qmd embed
```

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
cpulimit -l 80 -- qmd embed
```

这样限制在 80% CPU，虽然慢一点，但服务器不会卡死。

### 3. Vulkan 编译失败

QMD 内置的 llama.cpp 默认尝试用 Vulkan GPU 加速。我的云服务器没有 GPU，编译 Vulkan 后端直接报错。不过 QMD 会自动回退到 CPU 模式，功能不受影响，只是速度慢。

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

折腾的过程本身也是学习——搞清楚了 OpenClaw 的 `contextEngine` 架构、插件 slot 机制、记忆搜索后端的切换方式。这些理解比用哪个工具更有价值。
