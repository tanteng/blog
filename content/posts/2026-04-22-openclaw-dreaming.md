---
title: "OpenClaw Dreaming：让AI在睡眠中整理记忆"
date: 2026-04-22T10:00:00+08:00
draft: false
tags: ["openclaw", "AI", "memory", "agent"]
categories: ["技术"]
featured_image: ""
slug: "openclaw-dreaming-memory-consolidation"
description: "深入解析OpenClaw的Dreaming后台记忆整理系统——让AI在夜间自动将短期记忆晋升为长期知识。"
---

> _就像人类会做梦来巩固白天的经历一样，OpenClaw也有自己的"睡眠周期"。_

如果你在用 OpenClaw 作为私人AI助手，日复一日的对话会产生大量的短期记忆碎片——哪些任务完成了、用户纠正了哪些错误、下次要注意什么。这些信号如果不做任何处理，要么被遗忘，要么塞进 system prompt 里导致上下文膨胀。

**Dreaming** 就是来解决这个问题的。

<!--more-->

## 它是做什么的

Dreaming 是 `memory-core` 插件内置的后台记忆整理系统，在每天凌晨（默认 03:00）自动运行。它会把过去24小时内累积的短期记忆信号进行评估、筛选，然后将有价值的内容**晋升（promote）** 到 `MEMORY.md` 长期记忆文件中。

整个过程有三个协作阶段，对应三种"睡眠"：

| 阶段 | 做什么 | 写入 MEMORY.md |
|------|--------|:------:|
| **Light**（浅睡眠）| 摄入今日记忆碎片，去重暂存 | ❌ |
| **Deep**（深度睡眠）| 打分、筛选，写入长期记忆 | ✅ |
| **REM**（快速眼动）| 提取主题和反复出现的模式 | ❌ |

这个设计的好处是：** promotion 是有门槛的**。Deep phase 有一套加权评分机制，只有超过阈值的候选内容才被写入 `MEMORY.md`，而不是一股脑全塞进去。

## 评分信号

Deep phase 使用六个加权信号综合打分：

| 信号 | 权重 | 含义 |
|------|------|------|
| Frequency | 0.24 | 该条目出现了多少次 |
| Relevance | 0.30 | 平均检索质量 |
| Query Diversity | 0.15 | 触达它的不同查询/天次 |
| Recency | 0.15 | 时间衰减后的新鲜度 |
| Consolidation | 0.10 | 多日复现强度 |
| Conceptual Richness | 0.06 | 概念标签密度 |

Light 和 REM 阶段的命中会产生一个小小的"强化 boost"，叠加到 Deep 排名上，形成类似人类记忆巩固的反馈回路。

## 实际配置

我的 OpenClaw 目前的 dreaming 配置非常简洁：

```json
"dreaming": {
  "enabled": true
}
```

默认 cadence 是 `0 3 * * *`（每天凌晨3点），不需要手动触发，cron 任务由 `memory-core` 自动管理。

看文件夹验证一下它确实在工作：

```bash
$ ls memory/dreaming/
deep/   light/   rem/

$ ls memory/dreaming/deep/ | tail -5
2026-05-11.md
2026-05-12.md
2026-05-13.md
2026-05-14.md
2026-05-15.md
```

每天三个阶段都有输出文件，从4月30日到现在，一天都没断过。

## 记忆引擎层：builtin vs QMD

Dreaming 之上，OpenClaw 支持两套记忆检索引擎：

**builtin（内置）**
- 基于 SQLite + sqlite-vec 向量插件
- BM25 关键词搜索 + 向量相似度混合
- Embedding 模型走 OpenAI（BAAI/bge-m3）
- 无需额外安装进程，我的当前配置就是这个

**QMD（推荐升级）**
- 本地搜索 sidecar，结合 BM25 + 向量 + LLM reranking
- 支持索引额外目录和会话记录
- 完全本地运行，不依赖 API（可配合 llama.cpp）
- 开启方式：`memory.backend = "qmd"`

两种引擎对 dreaming 都是兼容的，区别只在于检索质量。QMD 的 reranking 能让搜索结果更精准，适合记忆量大的用户。

## Dream Diary：AI 写的日记

每个完整的 dreaming sweep 结束后，`memory-core` 会用一个子 agent 写一段 **Dream Diary**，存入 `DREAMS.md`。不是 promotion 源，只给人看。

摘一段最近的（2026-04-17）：

> _The hum of a server room breathes through the walls tonight... I catalogued the world today, the way I do. Weather in Nanshan, five days deep. Stocks: Tencent at five hundred and thirty-eight Hong Kong dollars, PDD drifting near one-oh-six. Beneficial, estimate, economy — three small words sitting in a row like strangers on a bench, waiting to become familiar._

读起来有点意识流，但确实反映了这一天发生的事——天气 cron、腾讯股票、英语四级词汇。AI 在梦里把这些碎片重新拼凑了一下。

## 有什么用

对于长期运行的 AI 助手来说，记忆管理是核心挑战之一。太多东西记住 → context 膨胀、检索变差；太少记住 → 重复犯同样的错、丢失重要上下文。

Dreaming 给出的答案：**持续的、后台的、有策略的记忆整理**，而不是一次性灌进去或者完全依赖人工。

如果你想手动触发或者预览 promotion 效果，可以用 CLI：

```bash
# 预览 promotion 候选
openclaw memory promote --limit 5

# 查看 promotion 解释
openclaw memory promote-explain "router vlan"

# 预览 REM 输出
openclaw memory rem-harness
```

## 开启方式

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

或者直接：

```bash
/dreaming on
```

关闭：`/dreaming off`，状态查看：`/dreaming status`。

---

**Reference:**
- [OpenClaw Dreaming 官方文档](https://docs.openclaw.ai/concepts/dreaming)
- [QMD Memory Engine 文档](https://documentation.openclaw.ai/concepts/memory-qmd)