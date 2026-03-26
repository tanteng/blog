---
title: "OpenViking × OpenClaw：给 AI Agent 装上长期记忆"
date: 2026-03-26
description: "介绍字节跳动的 OpenViking 项目，如何与 OpenClaw 集成实现长程记忆，以及为记忆系统加装的「双写」保险机制。"
categories: ['tech']
tags: ['openclaw', 'openviking', 'ai', 'memory', 'context-engineering']
featured_image: "https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/0011f1b2f3102c5e3b03382d0b494545.png"
---

最近给 OpenClaw 装上了 OpenViking，顺手配了套记忆"双写"机制。折腾了一阵，记录下过程和使用心得，也顺便梳理一下这套系统的工作原理。

<!--more-->

## OpenClaw 的记忆机制

OpenClaw 是个很强的 Agent 框架，能"看见"屏幕、能操作电脑，复杂任务自动化不在话下。但它有个致命短板——**健忘**。

OpenClaw 的核心理念是：**Memory = 文件**。所有记忆都以 Markdown 格式存在本地磁盘上，不依赖任何云服务。记忆分为两层：每日日志（`memory/YYYY-MM-DD.md`）记录当天对话原始内容，长期记忆（`MEMORY.md`）沉淀重要决策和偏好。文件保存后，后台会自动切片构建向量索引，支持语义搜索和关键词搜索，下次对话时按需召回，而不是一股脑塞进上下文。

## 为什么需要 OpenViking

[OpenViking](https://github.com/volcengine/OpenViking) 是字节跳动 Viking 团队开源的**面向 AI Agent 的上下文数据库**，发布一个月斩获 4.5k GitHub Star。

核心特点：

- **虚拟文件系统范式**：用文件系统的方式组织记忆（`memory/`、`resources/`、`skills/`），告别碎片化
- **轻量高效**：分层上下文供给，只在必要时加载信息，从根本上解决 Token 消耗大的问题
- **插件化集成**：装个插件就能接 OpenClaw，不用改核心代码

底层依赖 VikingDB 向量数据库，支持混合检索（语义 + 关键词），万亿级向量毫秒级响应。

---

## 效果数据

火山引擎的测试（LoCoMo10 数据集，1540 条测试用例）：

| 实验组 | 任务完成率 | 输入 Token 总计 |
|--------|-----------|-----------------|
| OpenClaw (原生 memory-core) | 35.65% | 24,611,530 |
| OpenClaw + OpenViking Plugin | 51.23% | 2,099,622 |

接入 OpenViking 后：**任务完成率提升 43%，Token 成本降低 91%**。

---

## 我的部署：本地插件模式

我的 OpenClaw 跑在腾讯云 Lighthouse 上，接的是**本地插件模式**，安装了 OpenViking 的 memory plugin。

配置概览：

```yaml
# OpenViking 配置文件: ~/.openviking/ov.conf

server:
  host: "127.0.0.1"
  port: 1933

storage:
  workspace: "/root/.openviking/data"
  vectordb:
    backend: "local"

embedding:
  dense:
    backend: "volcengine"
    model: "doubao-embedding-vision-251215"
    dimension: 1024
```

---

## 双写机制：给记忆上保险

集成 OpenViking 之后，我又给它加了一层**双写保险**——同时写入两个地方：**本地文件**（`MEMORY.md` + `memory/YYYY-MM-DD.md`）和 **OpenViking 向量库**。向量库检索强但可能漏召或服务抖动，文件系统稳定可读但语义搜索弱——两者互补，才是真正的保险。我在 `AGENTS.md` 里固化了这个规则：存入记忆时必须同时写文件和存向量库。

![配置双写机制](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/c33867c3fca51998b201a132593d168b.png)

双写的好处：**不怕单点故障**、**可读性强**（文件随时能打开看）、**检索互补**（文件精确匹配，向量库语义相似）、**审计方便**（文件即日志）。

---

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

---

## 总结

OpenViking 解决了 Agent 的"健忘症"，让 OpenClaw 真正成为有记忆的助手。

加上双写机制，记忆系统稳如老狗——文件是锚，向量库是检索，两条腿走路。

如果你也在用 OpenClaw，强烈建议接上 OpenViking，然后给自己定制一套记忆管理规则。

---

## 参考

- [OpenViking GitHub](https://github.com/volcengine/OpenViking)
- [OpenViking × OpenClaw 官方集成文档](https://www.volcengine.com/docs/6396/2249500?lang=zh)
- [VikingDB 向量数据库](https://www.volcengine.com/product/vector-database)
