---
title: 'OpenClaw QMD Memory Engine：本地优先的 AI 记忆搜索引擎'
date: 2026-03-30T15:30:00+08:00
draft: false
tags: ['openclaw', 'qmd', 'memory', 'rag', 'llm', 'ai']
categories: ['tech']
description: 'OpenClaw 的 QMD Memory Engine 是一个本地优先的搜索引擎，集成 BM25、向量搜索和重排序于一体，支持索引工作区之外的文档和历史会话，完全离线运行无需 API Key。'
---

[OpenClaw](https://github.com/open-claw/open-claw) 有一套内置的 Memory 系统，基于 SQLite 实现，开箱即用。但对于需要更高搜索质量、更广索引范围的场景，OpenClaw 提供了一个更强大的选项——**QMD Memory Engine**。

本文基于 [OpenClaw 官方文档](https://docs.openclaw.ai/concepts/memory-qmd)，梳理 QMD 的核心概念、架构原理和配置方法。

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

QMD 是 OpenClaw Memory 系统的"增强版"，它的核心价值在于：

1. **搜索质量更高**——BM25 + 向量 + 重排序的组合
2. **数据范围更广**——不局限于工作区，任何本地文件都能索引
3. **完全本地**——通过 GGUF 模型本地推理，不需要外部 API
4. **优雅降级**——出问题自动回退到内置引擎，不影响使用

对于把 OpenClaw 作为日常工作助手的人来说，QMD 让 AI 的"记忆力"从"只看当前项目"扩展到了"你磁盘上的任何文件"——这是一个质的飞跃。

更多配置细节可参考 OpenClaw 官方的 [Memory 配置参考文档](https://docs.openclaw.ai/reference/memory-config)。
