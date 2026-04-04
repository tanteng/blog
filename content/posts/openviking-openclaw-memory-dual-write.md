---
title: "OpenViking × OpenClaw：给 AI Agent 装上长期记忆"
date: 2026-03-26
description: "深入解析字节跳动 OpenViking 插件如何接管 OpenClaw 的记忆生命周期，以及我为什么要给记忆系统加装「双写」保险机制。"
categories: ['tech']
tags: ['openclaw', 'openviking', 'ai', 'memory', 'context-engineering']
featured_image: "https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/0011f1b2f3102c5e3b03382d0b494545.png"
---

最近给 OpenClaw 装上了 OpenViking，顺手配了套记忆"双写"机制。折腾了一阵，记录下过程和使用心得，也深入梳理一下 OpenViking 插件的运行原理——它到底是怎么接管 OpenClaw 的记忆系统的。

<!--more-->

## OpenClaw 原生的记忆机制

OpenClaw 是个很强的 Agent 框架，能"看见"屏幕、能操作电脑，复杂任务自动化不在话下。但它有个致命短板——**健忘**。

OpenClaw 的核心理念是：**Memory = 文件**。所有记忆都以 Markdown 格式存在本地磁盘上，不依赖任何云服务。记忆分为两层：

- **`MEMORY.md`**：长期记忆，存储持久性事实、偏好、关键决策。每次 DM 会话开始时**自动加载**。
- **`memory/YYYY-MM-DD.md`**：每日笔记，记录当天的上下文和观察。系统**自动加载今天和昨天**的笔记。

OpenClaw 还有一个关键的**自动记忆刷新**机制：在执行对话**压缩（compact）之前**，系统会运行一个静默轮次，提示 Agent 把重要的上下文保存到记忆文件里，防止压缩时丢失信息。文件保存后，后台自动切片构建向量索引，支持语义搜索和关键词搜索，下次对话时按需召回。

## 为什么需要 OpenViking

[OpenViking](https://github.com/volcengine/OpenViking) 是字节跳动 Viking 团队开源的**面向 AI Agent 的上下文数据库**，发布一个月斩获 4.5k GitHub Star。

核心特点：

- **虚拟文件系统范式**：用文件系统的方式组织记忆（`memory/`、`resources/`、`skills/`），告别碎片化
- **轻量高效**：分层上下文供给，只在必要时加载信息，从根本上解决 Token 消耗大的问题
- **插件化集成**：装个插件就能接 OpenClaw，不用改核心代码

底层依赖 VikingDB 向量数据库，支持混合检索（语义 + 关键词），万亿级向量毫秒级响应。

## OpenViking 插件的运行机制

这是本文的重点。很多人以为 OpenViking 只是个"记忆查询工具"，实际上它是一个**横跨 OpenClaw 整个生命周期的深度集成层**，同时扮演四个角色：

| 角色 | 职责 |
|---|---|
| **上下文引擎** | 组装上下文、轮次后处理、对话压缩 |
| **钩子层** | 接管 prompt 构建前、会话开始/结束等事件 |
| **工具提供者** | 注册记忆相关的 Agent 工具 |
| **运行时管理器** | 本地模式下启动并监控 OpenViking 子进程 |

### 生命周期钩子：全面接管

安装插件后，OpenViking 会接管 OpenClaw 的四个关键生命周期阶段：

**① before_prompt_build —— 自动记忆召回**

每次对话前，插件提取最新的用户输入，并行查询 `viking://user/memories` 和 `viking://agent/memories`，检索结果经过智能过滤和排序（考虑记忆层级、偏好记忆、事件记忆、词汇重叠度），转化为 `<memory>` 块插入到提示词最前面。

**② assemble —— 上下文组装**

不再是简单地从本地文件加载历史，而是从 OpenViking 读取会话上下文。OpenClaw 最终看到的是：

> 压缩的历史摘要 + 归档索引 + 活跃消息

而不是无限增长的原始文本。

**③ afterTurn —— 轮次后处理**

每轮对话结束后，新的对话增量被追加到 OpenViking 会话中（去除注入的 `<memory>` 块等噪音）。当 `pending_tokens` 超过阈值时，触发**异步提交**——归档生成和记忆提取在服务器端完成，不阻塞当前对话。

**④ compact —— 对话压缩**

这是一个严格的**同步边界**。调用 `commit(wait=true)` 阻塞直到完成，然后读取最新的归档概览。如果摘要太粗糙，模型可以调用 `ov_archive_expand` 工具重新打开特定归档。

### 流程全景图

下面这张图展示了 OpenViking 插件在一次完整对话中的运行流程：

{{< mermaid >}}flowchart TD
    A["👤 用户发送消息"] --> B["🔍 before_prompt_build"]
    B --> B1["提取用户输入关键词"]
    B1 --> B2["并行查询\nviking://user/memories\nviking://agent/memories"]
    B2 --> B3["智能过滤 & 排序\n记忆层级 / 偏好 / 词汇重叠"]
    B3 --> B4["生成 memory 块\n插入 prompt 最前面"]

    B4 --> C["🧩 assemble 上下文组装"]
    C --> C1["从 OpenViking 读取\n会话上下文"]
    C1 --> C2["拼接：压缩摘要\n+ 归档索引\n+ 活跃消息"]

    C2 --> D["🤖 模型推理生成回复"]

    D --> E["📝 afterTurn 轮次后处理"]
    E --> E1["追加对话增量\n到 OpenViking 会话"]
    E1 --> E2{"pending_tokens\n> 阈值？"}
    E2 -- 是 --> E3["异步 commit\n归档 + 记忆提取\n不阻塞对话"]
    E2 -- 否 --> E4["等待下一轮"]

    E3 --> F{"需要压缩？"}
    E4 --> F
    F -- 是 --> G["🗜️ compact 压缩"]
    G --> G1["commit wait=true\n同步阻塞"]
    G1 --> G2["读取归档概览"]
    G2 --> G3["可选：ov_archive_expand\n展开特定归档"]
    F -- 否 --> H["✅ 等待下一条消息"]
    G3 --> H

    style A fill:#e3f2fd
    style D fill:#fff3e0
    style H fill:#e8f5e9
{{< /mermaid >}}

### 新增的工具能力

除了自动行为，插件还为模型提供了四个显式操作记忆的工具：

| 工具 | 作用 |
|---|---|
| `memory_recall` | 显式搜索长期记忆 |
| `memory_store` | 立即写入记忆并触发提交 |
| `memory_forget` | 通过 URI 删除或搜索后移除匹配项 |
| `ov_archive_expand` | 摘要不够时，展开特定归档看原始消息 |

## 效果数据

火山引擎的测试（LoCoMo10 数据集，1540 条测试用例）：

| 实验组 | 任务完成率 | 输入 Token 总计 |
|--------|-----------|-----------------|
| OpenClaw (原生 memory-core) | 35.65% | 24,611,530 |
| OpenClaw + OpenViking Plugin | 51.23% | 2,099,622 |

接入 OpenViking 后：**任务完成率提升 43%，Token 成本降低 91%**。

## 装了 OpenViking 后，原生记忆文件还会自动维护吗？

这是我在使用过程中产生的一个核心疑问——给 OpenClaw 装了 OpenViking 之后，OpenClaw 原生的 `MEMORY.md` 和每日笔记文件还会被自动维护吗？

翻了 [OpenViking 的 openclaw-plugin 官方文档](https://github.com/volcengine/OpenViking/blob/main/examples/openclaw-plugin/README.md) 和 [OpenClaw 记忆机制官方文档](https://docs.openclaw.ai/concepts/memory)，答案是：

### **不会了。**

OpenViking 插件横跨了 OpenClaw 的 `assemble` / `afterTurn` / `compact` / `before_prompt_build` 等关键生命周期钩子，**全面接管了记忆的读写和维护**。具体对照如下：

| 原生行为 | 装了 OpenViking 后 |
|---|---|
| 压缩前自动刷新写入 `MEMORY.md` | ❌ 被 OpenViking 的 commit 流程替代 |
| 压缩前自动刷新写入每日笔记 | ❌ 同上 |
| 会话开始自动加载 `MEMORY.md` | ❌ 改为从 OpenViking 检索 `<memory>` 块注入 |
| 自动加载今天/昨天的每日笔记 | ❌ 被 OpenViking 上下文组装替代 |
| 用户说"记住这个"时写入文件 | ❌ 改为调用 `memory_store` 写入 OpenViking |

简单说：**OpenViking 是一次彻底的接管，而不是增量的增强。** 安装后，所有记忆存储和检索都转移到了 OpenViking 后端，原生的 Markdown 记忆文件将不再被自动维护。

这也是我决定做双写机制的直接原因。

## 双写机制：给记忆上保险

既然 OpenViking 会全面接管记忆系统，原生文件不再自动维护，那就存在一个隐患：**所有鸡蛋放在一个篮子里**。向量库检索强但可能漏召或服务抖动，一旦出问题就什么记忆都找不回来了。

所以我给它加了一层**双写保险**——同时写入两个地方：

1. **本地文件**（`MEMORY.md` + `memory/YYYY-MM-DD.md`）
2. **OpenViking 向量库**

我在 `AGENTS.md` 里固化了这个规则：存入记忆时必须同时写文件和存向量库。

![配置双写机制](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/c33867c3fca51998b201a132593d168b.png)

双写的好处：

- **不怕单点故障**：向量库挂了，文件还在
- **可读性强**：Markdown 文件随时能打开查看
- **检索互补**：文件精确匹配，向量库语义相似
- **审计方便**：文件即日志，git 可追踪变更

## 适用场景

**双写特别适合：**

- 重要的用户偏好（联系方式、时区、博客规范）
- 关键决策记录（为什么选了这个方案）
- 踩坑记录（防止重蹈覆辙）
- 项目上下文（跨会话的工作进度）

**不需要双写的：**

- 临时会话的中间结果
- 可以从外部重新获取的信息
- 敏感信息（密钥等）

## 总结

OpenViking 不只是给 OpenClaw 加了个记忆查询工具——它是一次**对记忆系统的彻底接管**，从上下文组装、轮次处理到对话压缩，全链路重写。接管后效果确实好：任务完成率提升 43%，Token 成本降低 91%。

但彻底接管也意味着原生的 Markdown 记忆文件不再被自动维护。这就是为什么我要加上双写机制——文件是锚，向量库是检索，两条腿走路，才是真正的保险。

如果你也在用 OpenClaw，强烈建议接上 OpenViking，然后给自己定制一套记忆管理规则。

## 参考

- [OpenViking × OpenClaw 插件文档](https://github.com/volcengine/OpenViking/blob/main/examples/openclaw-plugin/README.md)
- [OpenClaw 记忆机制官方文档](https://docs.openclaw.ai/concepts/memory)
