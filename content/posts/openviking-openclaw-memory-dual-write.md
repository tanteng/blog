---
title: "OpenViking × OpenClaw：给 AI Agent 装上「长期记忆」，顺便开启双写保险机制"
date: 2026-03-26
description: "介绍字节跳动的 OpenViking 项目，如何与 OpenClaw 集成实现长程记忆，以及为记忆系统加装的「双写」保险机制。"
categories: ['tech']
tags: ['openclaw', 'openviking', 'ai', 'memory', 'context-engineering']
---

<!--more-->

## 先说结论

给 OpenClaw 接上 OpenViking 之后：
- **任务完成率提升 43%**
- **Token 成本降低 91%**

这不是我说的，是火山引擎的官方测试数据。

---

## 为什么需要 OpenViking

OpenClaw 是个很强的 Agent 框架，能"看见"屏幕、能操作电脑，复杂任务自动化不在话下。但它有个致命短板——**健忘**。

社区里有人吐槽：*"刚告诉它我的 API 密钥，下次对话它就忘了。"*

这不是 bug，是设计取舍。单次对话内上下文窗口有限，跨会话的记忆管理需要额外组件。

原生 OpenClaw 的 memory-core 模块存在这几个问题：

| 问题 | 影响 |
|------|------|
| 任务完成率低 | 对话轮次多了容易"失忆"，回复跑偏 |
| 记忆碎片化 | 平铺式存储，检索效率低 |
| Token 成本激增 | 历史信息全塞上下文窗口，贵得离谱 |
| 跨场景协作难 | 多 Agent 间记忆孤岛，无法流转 |

OpenViking 就是来解决这个的。

---

## OpenViking 是什么

[OpenViking](https://github.com/volcengine/OpenViking) 是字节跳动 Viking 团队开源的**面向 AI Agent 的上下文数据库**，发布一个月斩获 4.5k GitHub Star。

核心特点：

- **虚拟文件系统范式**：用文件系统的方式组织记忆（`memory/`、`resources/`、`skills/`），告别碎片化
- **轻量高效**：分层上下文供给，只在必要时加载信息，从根本上解决 Token 消耗大的问题
- **插件化集成**：装个插件就能接 OpenClaw，不用改核心代码

底层依赖 VikingDB 向量数据库，支持混合检索（语义 + 关键词），万亿级向量毫秒级响应。

---

## OpenClaw 集成 OpenViking：效果数据

火山引擎的测试（LoCoMo10 数据集，1540 条测试用例）：

| 实验组 | 任务完成率 | 输入 Token 总计 |
|--------|-----------|-----------------|
| OpenClaw (原生 memory-core) | 35.65% | 24,611,530 |
| OpenClaw + OpenViking Plugin (-memory-core) | 52.08% | 4,264,396 |
| OpenClaw + OpenViking Plugin (+memory-core) | 51.23% | 2,099,622 |

结论：
- 任务完成率提升 **43%**
- Token 成本降低 **91%**

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
    backend: "local"  # 本地向量库

embedding:
  dense:
    backend: "volcengine"
    model: "doubao-embedding-vision-251215"
    dimension: 1024
```

OpenViking HTTP API 监听 **1933** 端口，AGFS 监听 **1833** 端口。

---

## 双写机制：给记忆上保险

集成 OpenViking 之后，我又给它加了一层**双写保险**。

### 什么是双写

双写 = **同时写入两个地方**：

1. **本地文件**（`MEMORY.md` + `memory/YYYY-MM-DD.md`）
2. **OpenViking 向量库**（语义检索）

### 为什么要双写

向量数据库虽好，但：
- 服务可能重启
- 数据可能损坏
- 检索可能漏召回

文件系统虽然慢，但**稳定、可读、永远能打开看**。

两者互补，才是真正的保险。

### 双写实现

```python
# 存入记忆时执行两步：
# 1. 写文件
write_to_file(text, "MEMORY.md")

# 2. 存向量库
memory_store(text=text, role="user")
```

我写了个 [skill](https://github.com/tanteng/blog) 来自动化这个流程：

```
skills/memory-dual-write/
├── SKILL.md        # 双写机制说明
└── dual_write.py  # Python 工具脚本
```

并在 `AGENTS.md` 里固化了这个规则：**存入记忆时必须同时写文件和存向量库**。

---

## 双写的好处

1. **不怕单点故障**：文件挂了有向量库，向量库挂了有文件
2. **可读性强**：文件随时能打开看，向量库的检索结果黑箱化
3. **检索互补**：文件搜索精确，向量库搜索语义相近
4. **审计方便**：文件即日志，回溯清晰

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
