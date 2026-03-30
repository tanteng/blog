---
title: 'OpenClaw 的 QMD 记忆引擎'
date: 2026-03-30T15:30:00+08:00
draft: false
tags: ['openclaw', 'qmd', 'memory', 'rag', 'llm', 'ai']
categories: ['tech']
description: 'OpenClaw 的 QMD Memory Engine 是一个本地优先的搜索引擎，集成 BM25、向量搜索和重排序于一体。本文梳理其架构原理、配置方法、在记忆体系中的触发机制，以及与 OpenViking 方案的对比。'
---

[OpenClaw](https://github.com/open-claw/open-claw) 有一套内置的 Memory 系统，基于 SQLite 实现，开箱即用。但对于需要更高搜索质量、更广索引范围的场景，OpenClaw 提供了一个更强大的选项——**QMD Memory Engine**。

本文基于 [OpenClaw 官方文档](https://docs.openclaw.ai/concepts/memory-qmd)，梳理 QMD 的核心概念、架构原理、配置方法，以及它在 OpenClaw 记忆体系中的实际角色，最后与 OpenViking 方案做对比。

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

Sidecar 是微服务架构中的经典模式：主进程专注核心业务，辅助功能交给一个**独立进程**在旁边跑，两者通过本地通信协作。QMD 就是 OpenClaw 的搜索 Sidecar——OpenClaw 管聊天和记忆读写，QMD 专门负责搜索。

{{< mermaid >}}
graph TB
    subgraph OpenClaw["OpenClaw（主进程）"]
        Chat["对话 / Agent 调度"]
        MemRW["记忆读写<br/>MEMORY.md · memory/*.md"]
    end

    subgraph QMD["QMD（Sidecar 进程）"]
        BM25["BM25 关键词"]
        Vec["向量搜索"]
        Rerank["LLM 重排序"]
    end

    subgraph Data["数据源"]
        Workspace["工作区 Memory 文件"]
        Extra["额外目录<br/>~/notes · 项目文档"]
        Sessions["历史会话记录"]
    end

    Chat -->|"memory_search"| QMD
    QMD -->|"返回相关片段"| Chat
    QMD ---|"每 5 分钟自动索引"| Data
    Chat -.->|"QMD 不可用时降级"| SQLite["内置 SQLite 引擎"]

    style OpenClaw fill:#e8f4fd,stroke:#1a73e8
    style QMD fill:#fef3e0,stroke:#e8a017
    style Data fill:#e8f5e9,stroke:#34a853
{{< /mermaid >}}

这种设计带来三个好处：

- **解耦**：QMD 挂了不影响 OpenClaw 主功能，自动降级到内置引擎
- **独立生命周期**：QMD 在后台默默更新索引、生成向量，不和主进程抢资源
- **可替换**：今天用 QMD，明天换别的搜索引擎，只需改配置

OpenClaw 负责管理 QMD 的完整生命周期：

1. 启动时，根据工作区 Memory 文件和配置的额外路径创建 QMD 集合
2. **定期刷新**：启动时 + 每 5 分钟自动执行 `qmd update`（更新索引）和 `qmd embed`（生成向量），后台运行不阻塞聊天
3. 你不需要手动管理索引，也不需要 crontab

### 搜索与容错

搜索流程有完善的降级机制：

1. 使用配置的 `searchMode` 执行搜索（默认 `search`，也支持 `vsearch` 和 `query`）
2. 如果当前模式失败，自动重试 `qmd query`
3. 如果 QMD **完全不可用**，无缝回退到内置 SQLite 引擎

### 首次运行须知

QMD 通过 Bun + node-llama-cpp 运行，会在首次执行 `qmd query` 时自动下载用于重排序和查询扩展的 **GGUF 模型**（约 2 GB）。所以首次搜索会比较慢，后续就正常了。

## 安装与配置

### 前置条件

- 安装 QMD（两种方式均可）：

```bash
# 从 npm registry 安装（GitHub README 推荐）
npm install -g @tobilu/qmd

# 从 GitHub 直接安装（OpenClaw 官方文档写法）
bun install -g https://github.com/tobi/qmd
```

- macOS 上需要支持扩展的 SQLite：`brew install sqlite`
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

OpenClaw 会自动在 `~/.openclaw/agents/<agentId>/qmd/` 下创建主目录并管理所有后续流程。

### 索引额外路径

把工作区以外的文档纳入搜索范围：

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

配置后，来自额外路径的搜索结果会以 `qmd/<collection>/<relative-path>` 的格式显示，`memory_get` 能识别这个前缀并从正确的目录读取文件。

### 索引历史会话

启用后，OpenClaw 会把之前的对话导出为"用户/助手"轮次格式，存入专用的 QMD 集合：

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "sessions": { "enabled": true }
    }
  }
}
```

### 其他配置

- **搜索范围**：默认只在 DM（直接消息）会话中返回 QMD 搜索结果，群组和频道不会。可通过 `memory.qmd.scope` 修改。
- **引用标注**：`memory.citations` 设为 `auto` 或 `on` 时，搜索结果带 `Source: <path#line>` 来源标注。
- **超时**：`memory.qmd.limits.timeoutMs` 默认 4000ms，硬件较慢可调大。

## QMD 在记忆体系中的角色

搞清楚一点很重要——**QMD 是按需语义搜索引擎，不是存储引擎，也不是每次对话都会被调用。**

OpenClaw 的记忆分两个阶段加载：

**启动时（不经过 QMD）：** 读取 `MEMORY.md`（长期记忆）+ `memory/今天.md` + `memory/昨天.md`，注入 system prompt。

**对话过程中（模型自行判断）：** 当模型认为需要从历史记忆中搜索时，调用 `memory_search`——**这里才触发 QMD**。

举个例子：你问"上个月讨论的数据库迁移方案是什么？"——上个月的信息不在已加载的文件里，模型会调用 `memory_search`，QMD 从所有历史日志中语义检索，返回最相关的片段。而如果你问的是今天刚聊过的东西，信息已经在上下文里了，QMD 不会介入。**不需要搜的时候就不搜，省 token。**

一句话概括：**启动时读小文件保证基础记忆，运行时靠 QMD 按需检索保证深度记忆。**

值得一提的是，如果你的记忆文件本来就很少，启动时加载的 3 个文件基本覆盖了所有内容，模型几乎不会触发 `memory_search`，QMD 就一直处于待命状态。**记忆少的时候 QMD 是保险，记忆多了才是刚需。**

## 与 OpenViking 的对比

OpenClaw 的记忆方案还有一个重量级选手——字节跳动开源的 [OpenViking](https://github.com/volcengine/OpenViking)。两者架构思路完全不同：

- **OpenViking** 接管了 OpenClaw 的 `contextEngine` 插槽，成为记忆系统的**唯一入口**，自己管存储和检索，OpenClaw 原生的 Markdown 记忆体系被绕过。
- **QMD** 只替换了**搜索引擎**，不改存储。记忆还是 Markdown 文件，QMD 只是让搜索从 SQLite FTS 升级到混合检索。

| | **QMD** | **OpenViking** |
|---|---|---|
| 搜索质量 | ⭐⭐⭐⭐ BM25+向量+重排序 | ⭐⭐⭐⭐⭐ 向量+VLM 摘要提取 |
| 费用 | 🆓 完全免费 | 💰 embedding + VLM API 持续计费 |
| 供应商锁定 | **无** — 数据始终是本地 Markdown | **有** — 切回需要数据迁移 |
| 中文支持 | ⚠️ EmbeddingGemma 对中文弱 | ✅ doubao-embedding 对中文好 |
| 硬件要求 | 较高（本地跑模型） | 低（计算在远端 API） |
| 数据所有权 | 本地 Markdown，你完全控制 | 存在 OpenViking 的向量库里 |

**怎么选**：个人项目、追求零成本和数据自主 → QMD；团队/企业、对中文检索质量要求高 → OpenViking；硬件受限的小机器 → OpenViking（计算在远端）。

## 实际使用注意事项

- **中文语义搜索效果有限**：QMD 默认的 EmbeddingGemma-300M 偏英文，纯中文语义查询效果不好。但 OpenClaw 会做 query expansion 缓解，也可以在记忆文件中中英双语并行。
- **CPU 资源消耗**：生成向量嵌入在 CPU 上跑推理，小规格云主机首次 embed 几百个文件可能打满 CPU，建议用 `cpulimit -l 50 -- qmd embed` 限速。
- **命令行和 OpenClaw 的 QMD 是隔离的**：同一个程序，不同的数据目录。终端里操作 QMD 不影响 OpenClaw，反之亦然。

## 总结

QMD 是 OpenClaw Memory 系统的"增强版"，核心价值在于：

1. **搜索质量更高** — BM25 + 向量 + 重排序
2. **数据范围更广** — 任何本地文件都能索引
3. **完全本地** — 不需要外部 API
4. **优雅降级** — 出问题自动回退到内置引擎
5. **无锁定** — 记忆始终是 Markdown 文件，随时可换后端

对于把 OpenClaw 作为日常工作助手的人来说，QMD 让 AI 的"记忆力"从"只看当前项目"扩展到了"你磁盘上的任何文件"——这是一个质的飞跃。

更多配置细节可参考 OpenClaw 官方的 [Memory 配置参考文档](https://docs.openclaw.ai/reference/memory-config)。
