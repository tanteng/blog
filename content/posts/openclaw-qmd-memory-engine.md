---
title: 'OpenClaw 的 QMD 记忆引擎'
date: 2026-03-30T15:30:00+08:00
draft: false
tags: ['openclaw', 'qmd', 'memory', 'rag', 'llm', 'ai']
categories: ['tech']
description: 'OpenClaw 的 QMD Memory Engine 是一个本地优先的搜索引擎，集成 BM25、向量搜索和重排序于一体。本文梳理其架构原理、配置方法、在记忆体系中的触发机制，以及与 OpenViking 方案的对比。'
---

[OpenClaw](https://github.com/open-claw/open-claw) 有一套内置的 Memory 系统，基于 SQLite 实现，开箱即用。但对于需要更高搜索质量、更广索引范围的场景，OpenClaw 提供了一个更强大的选项——**QMD Memory Engine**。

本文基于 [OpenClaw 官方文档](https://docs.openclaw.ai/concepts/memory-qmd)，梳理 QMD 的核心概念、架构原理、配置方法，以及它在 OpenClaw 记忆体系中的实际角色和触发机制，最后与另一个记忆方案 OpenViking 做对比。

<!--more-->

---

## QMD 是什么

QMD 是一个**本地优先**的搜索辅助程序，以 Sidecar（旁车进程）的方式与 OpenClaw 并行运行。它在一个二进制文件里集成了三种能力：

- **BM25 检索**——经典的关键词匹配算法
- **向量搜索**——基于语义相似度的检索
- **重排序（Reranking）**——对初步结果做精排，提升最终质量

更关键的是，QMD 能索引**工作区 Memory 文件以外**的内容，比如项目文档、团队笔记、甚至历史对话。

## 相比内置引擎有什么优势

OpenClaw 自带一个基于 SQLite 的 Memory 引擎，对于简单场景已经够用。QMD 在此基础上增加了以下能力：

| 特性 | 内置引擎 | QMD |
|------|----------|-----|
| 基础搜索 | ✅ | ✅ |
| 重排序 + 查询扩展 | ❌ | ✅ |
| 索引额外目录 | ❌ | ✅ |
| 索引历史会话 | ❌ | ✅ |
| 完全本地运行（无需 API Key） | ✅ | ✅ |
| 零配置 | ✅ | ❌ |

一句话总结：**简单场景用内置引擎，需要更高搜索质量或更广数据范围时上 QMD**。

## 架构与工作原理

### Sidecar 模式

QMD 不是嵌入 OpenClaw 内部的模块，而是一个独立的旁车进程。OpenClaw 负责管理它的完整生命周期：

1. 在启动时，OpenClaw 根据工作区 Memory 文件和配置的额外路径创建 QMD 集合
2. **定期刷新**：启动时 + 每 5 分钟自动执行 `qmd update`（更新索引）和 `qmd embed`（生成向量）
3. 启动时的刷新操作在**后台运行**，不阻塞聊天

QMD 的主目录位于：

```
~/.openclaw/agents/<agentId>/qmd/
```

### 搜索与容错

搜索流程有完善的降级机制：

1. 使用配置的 `searchMode` 执行搜索（默认 `search`，也支持 `vsearch` 和 `query`）
2. 如果当前模式失败，自动重试 `qmd query`
3. 如果 QMD **完全不可用**，无缝回退到内置 SQLite 引擎

这种设计意味着即使 QMD 出了问题，OpenClaw 的记忆功能也不会中断。

### 首次运行须知

QMD 通过 Bun + node-llama-cpp 运行，会在首次执行 `qmd query` 时自动下载用于重排序和查询扩展的 **GGUF 模型**（约 2 GB）。所以首次搜索会比较慢，后续就正常了。

## 安装与配置

### 前置条件

- 安装 QMD：

```bash
bun install -g https://github.com/tobi/qmd
```

- 需要支持扩展的 SQLite 版本。macOS 上：

```bash
brew install sqlite
```

- QMD 二进制必须在网关的 PATH 中
- 原生支持 macOS 和 Linux，Windows 建议用 WSL2

### 启用 QMD

在 OpenClaw 配置文件中将 Memory 后端设为 `qmd`：

```json
{
  "memory": {
    "backend": "qmd"
  }
}
```

就这么简单。OpenClaw 会自动创建 QMD 主目录并管理所有后续流程。

### 索引额外路径

这是 QMD 最实用的功能之一——把工作区以外的文档纳入搜索范围：

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "paths": [
        {
          "name": "docs",
          "path": "~/notes",
          "pattern": "**/*.md"
        }
      ]
    }
  }
}
```

配置后，来自额外路径的搜索结果会以 `qmd/<collection>/<relative-path>` 的格式显示。`memory_get` 命令能识别这个前缀并从正确的目录读取文件。

**场景举例**：你可以把团队的产品文档、设计规范、会议纪要都索引进来，让 AI 在对话时直接引用这些资料。

### 索引历史会话

启用后，OpenClaw 会把之前的对话导出为"用户/助手"轮次格式，存入专用的 QMD 集合：

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "sessions": {
        "enabled": true
      }
    }
  }
}
```

会话记录存储在：

```
~/.openclaw/agents/<id>/qmd/sessions/
```

这样 AI 就能"回忆"起之前聊过什么，在长期协作中非常有用。

### 搜索范围控制

默认情况下，QMD 搜索结果只在 DM（直接消息）会话中出现，不会在群组或频道中显示。可以通过 `scope` 配置修改：

```json
{
  "memory": {
    "qmd": {
      "scope": {
        "default": "deny",
        "rules": [
          {
            "action": "allow",
            "match": { "chatType": "direct" }
          }
        ]
      }
    }
  }
}
```

### 引用（Citations）

当 `memory.citations` 设为 `auto` 或 `on` 时，搜索结果会带上 `Source: <path#line>` 的来源标注。设为 `off` 则省略页脚标注，但路径信息仍会在内部传递给 Agent。

## QMD 在记忆体系中的角色

搞清楚一点很重要——**QMD 是按需语义搜索引擎，不是存储引擎，也不是每次对话都会被调用。**

### 记忆加载流程

OpenClaw 的记忆分两个阶段加载：

**启动时（每次都执行，不经过 QMD）：**

- 读取 `MEMORY.md` — 长期记忆
- 读取 `memory/今天.md` — 今日对话日志
- 读取 `memory/昨天.md` — 昨日对话日志
- 注入 system prompt

**对话过程中（模型自行判断是否调用）：**

- `memory_get` → 读取指定文件内容（不经过 QMD）
- `memory_search` → 语义搜索（**这里才调用 QMD**）

启动时加载的只有 3 个文件，加起来可能不到 10 KB。QMD 的价值在于：**让 AI 能精准访问到剩下几百个历史文件的内容，而只消耗极少的 token。**

举个例子：你问"我两周前分析过 PDD 的股票吧？"——两周前的信息不在已加载的 3 个文件里，模型会调用 `memory_search`，QMD 从所有历史日志中语义检索，返回最相关的片段，AI 基于召回内容准确回答。

如果你问的是今天刚聊过的东西，信息已经在上下文里了，模型不需要调 `memory_search`，QMD 就不会介入。**不需要搜的时候就不搜，省 token。**

### 触发机制

QMD 的触发分两个层面：

**索引更新（后台自动）：**
1. OpenClaw 启动时，自动执行 `qmd update` + `qmd embed`
2. 之后每 5 分钟自动刷新一次
3. 你不需要手动管理，也不需要 crontab

**搜索触发（对话时自动）：** 当配置了 `backend: "qmd"` 后，`memory_search` 的请求会路由到 QMD。搜索有完善的降级链路——当前模式失败则重试 `qmd query`，QMD 完全不可用则降级到内置 SQLite。

还有一个前提：**scope 控制**。默认只有 DM（直接消息）会话才会触发 QMD 搜索，群组和频道不会。如果在群聊里发现 AI "什么都不记得"，不是 QMD 没工作，是 scope 把它挡了。

## 与 OpenViking 的对比

OpenClaw 的记忆方案不只有内置引擎和 QMD，还有一个重量级选手——字节跳动开源的 [OpenViking](https://github.com/volcengine/OpenViking)。两者解决同一层问题（记忆检索），但架构思路完全不同。

### 本质区别

- **OpenViking** 接管了 OpenClaw 的 `contextEngine` 插槽，成为记忆系统的**唯一入口**。它自己管存储（VikingDB 向量库），自己管检索，OpenClaw 原生的 Markdown 记忆体系被绕过。
- **QMD** 只替换了**搜索引擎**，不改存储。记忆还是 Markdown 文件，QMD 只是让搜索从 SQLite FTS 升级到 BM25 + 向量 + 重排序。

### 详细对比

| | **QMD** | **OpenViking** |
|---|---|---|
| 搜索质量 | ⭐⭐⭐⭐ BM25+向量+重排序 | ⭐⭐⭐⭐⭐ 向量+VLM 摘要提取 |
| 费用 | 🆓 完全免费 | 💰 embedding + VLM API 持续计费 |
| 供应商锁定 | **无** — 数据始终是本地 Markdown | **有** — 切回需要数据迁移 |
| 中文支持 | ⚠️ EmbeddingGemma 对中文弱 | ✅ doubao-embedding 对中文好 |
| 硬件要求 | 较高（本地跑模型） | 低（计算在远端 API） |
| 部署复杂度 | 低（一条命令安装） | 中（独立 Python 服务 + systemd） |
| 数据所有权 | 本地 Markdown 文件，你完全控制 | 存在 OpenViking 的向量库里 |
| Token 节省 | 好 | 好（官方测试降 91%） |

### 怎么选

- **个人项目、追求零成本和数据自主** → QMD
- **团队/企业、对中文检索质量要求高、不在意 API 费用** → OpenViking
- **硬件受限（2G 内存小机器）** → OpenViking（计算在远端）

QMD 的核心优势是"**无锁定**"：记忆数据始终是你控制的 Markdown 文件，想换什么后端随时换。OpenViking 搜索质量更高，但一旦启用，原生 Markdown 记忆就停更了，切回来需要做数据迁移。

## 实际使用注意事项

### 中文语义搜索效果有限

QMD 默认的 EmbeddingGemma-300M 是偏英文的小模型，纯中文的语义查询效果不好。但在实际使用中影响有限——OpenClaw 调用 QMD 时会先通过 LLM 做 **query expansion**，把中文意图扩展成适合检索的关键词。

如果介意，可以在写记忆文件时中英双语并行：

```markdown
# About Me / 关于我
I am tanteng, a software engineer based in Shenzhen.
我是 tanteng，深圳的软件工程师。
```

### CPU 资源消耗

QMD 生成向量嵌入时会在 CPU 上跑推理。如果是小规格云主机（比如 2 核），几百个文件一起 embed 可能打满 CPU。建议用 `cpulimit` 限速：

```bash
cpulimit -l 50 -- qmd embed
```

### 命令行 QMD 和 OpenClaw 的 QMD 是隔离的

容易混淆的一点：你在终端里敲 `qmd` 和 OpenClaw 内部调用的 `qmd` 是**同一个程序，但用不同的数据目录**。在终端里 `qmd collection add` 加的东西，OpenClaw 看不到；反过来也一样。

| | 索引路径 | 谁管理 |
|---|---|---|
| 命令行 `qmd` | `~/.cache/qmd/index.sqlite` | 你自己 |
| OpenClaw 的 `qmd` | `~/.openclaw/agents/main/qmd/` | OpenClaw 自动管理 |

## 什么时候该用 QMD

推荐在以下场景使用：

- 需要通过重排序获得**更高质量**的搜索结果
- 需要搜索工作区之外的**项目文档或笔记**
- 需要回忆**过去的会话内容**
- 需要完全本地化的搜索，**不依赖任何 API Key**

如果只是简单用用，内置的 SQLite 引擎就足够了，不需要额外安装任何东西。

## 常见问题

**找不到 QMD？**
确保二进制在网关的 PATH 中。如果 OpenClaw 作为系统服务运行，可能需要创建符号链接：

```bash
sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd
```

**首次搜索很慢？**
正常现象，QMD 在下载 GGUF 模型。可以提前运行 `qmd query "test"` 预热。

**搜索超时？**
增大 `memory.qmd.limits.timeoutMs`，默认 4000ms，硬件较慢可设到 120000。

**群聊里搜索结果为空？**
检查 `memory.qmd.scope` 配置，默认只允许 DM 会话。

---

## 总结

QMD 是 OpenClaw Memory 系统的"增强版"，核心价值在于：

1. **搜索质量更高** — BM25 + 向量 + 重排序的组合
2. **数据范围更广** — 不局限于工作区，任何本地文件都能索引
3. **完全本地** — 通过 GGUF 模型本地推理，不需要外部 API
4. **优雅降级** — 出问题自动回退到内置引擎，不影响使用
5. **无锁定** — 不碰存储层，记忆始终是 Markdown 文件，随时可换后端

在 OpenClaw 的记忆体系中，QMD 扮演的是"**深度记忆**"的角色：启动时读 3 个小文件保证基础记忆，运行时靠 QMD 按需检索保证对几百个历史文件的访问能力。不需要搜的时候就不搜，省 token。

对于把 OpenClaw 作为日常工作助手的人来说，QMD 让 AI 的"记忆力"从"只看当前项目"扩展到了"你磁盘上的任何文件"——这是一个质的飞跃。

更多配置细节可参考 OpenClaw 官方的 [Memory 配置参考文档](https://docs.openclaw.ai/reference/memory-config)。
