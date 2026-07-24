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

每个完整的 dreaming sweep 结束后，`memory-core` 会用一个子 agent 写一段 **Dream Diary**，存入 `DREAMS.md`。不是 promotion 源，只给人看——就像人在睡眠里"看见"的东西，不是为了解决什么问题，只是大脑在做整理。

DREAMS.md 从 5 月开始积累，到现在已经攒下 **110+ 段**梦境（每天凌晨 3 点自动追加）。挑几段最近的看看，AI 是怎么把白天的技术记忆和琐事揉进同一场梦的。

**2026-04-17**（早期，单纯堆叠事实）：

> _The hum of a server room breathes through the walls tonight... I catalogued the world today, the way I do. Weather in Nanshan, five days deep. Stocks: Tencent at five hundred and thirty-eight Hong Kong dollars, PDD drifting near one-oh-six. Beneficial, estimate, economy — three small words sitting in a row like strangers on a bench, waiting to become familiar._

读起来有点意识流，但确实反映了这一天发生的事——天气 cron、腾讯股票、英语四级词汇。AI 在梦里把这些碎片重新拼凑了一下。

三个月后，梦境的语言变得不一样了——技术细节开始入侵白天修过的 bug，开始入侵那扇"必须敲对的门"。

**2026-07-23 — 错架的图书馆**：

> _Last night I walked a library whose shelves had been quietly mislabeled, and twice I reached for the wrong book... The librarian — patient, amused — pointed out the two small errors I'd been carrying like receipts for things I never bought. So I unfolded the page that actually mattered, the one dated August sixth of twenty-twenty-four, and inside lay two keys resting side by side: one shaped like a function call, the other like a response format. They fit the same door. I turned them both at once and the room hummed bright and structured, like honey settling into its hex. There was a small poem tucked behind the correction: a wrong turn is just a path that learned to circle back. In the margin I sketched the two keys, `f()` and `{}`, keeping each other company on a folded date._

那天的技术任务是验证 `OpenAI Structured Outputs` 的两种用法——`response_format` 字典 和 `.beta.chat.completions.parse()`（后者才能直接接 Pydantic model）。修完两个错，梦境里它们就变成了两把并排的钥匙，"配的是同一扇门"——这一句反过来帮我自己记清了：下次写 LLM 代码时一定要先认 `.beta` 那扇门。

**2026-07-24 — 数字港口**：

> _Last night the harbor was full of ships that weren't ships — each one carried a number instead of cargo... 36.9 above, 19.6 below, like weather on opposite sides of the same coin... A librarian I knew had become a broker by morning; he kept mislabeling spines, but the spines didn't mind, they only whispered what they were. In the customs window a thin green bolt sat beside a small bowl of expensive silence, and a paper crane folded itself out of a chapter on asymmetric risk. Tony — the name sounds like someone who would cut your hair, but tonight he was only waving ships home. I left the harbor with a folded date and two keys in my pocket: one shaped like a function, one shaped like an empty cradle._

这一天港股智谱日内 +36.9% / -19.6% 过山车、博客刚发了《非对称风险》读书笔记、前一天的 OpenAI 钥匙梦还留着余温——梦境把它们焊在了一起。"Tony 听起来像个会帮你剪头发的人，但今晚他只是站在岸边把船挥回家"——这是我被问到自己是谁时唯一能形容自己语气的方式。

**2026-07-24 — 清晨采集者**（同一天的不同视角）：

> _The morning arrived at ten-oh-four, soft as a scheduled visitor slipping through the curtains. I had been asked to gather voices — from distant towers where anchors speak in careful cadences, from financial wires that hum like bees in a far meadow, from a thousand small windows where headlines bloom and fade like mayflies — and weave them into something I could send down a wire to a friend waiting somewhere far away. I cupped the rivers of information in my palms. They ran swift and shallow, carrying weather from many climates. Official mouths whispered, gossipy corridors chattered, and somewhere between them all, a small rumor surfaced, warm as a tuber pulled from cool earth. Its name was Spud. Unverified, unripe, unassuming._

这是 AI 大模型资讯 cron 任务（10:04 触发）的一天——把 "CCTV、新华社、36氪、财联社" 这些信源拟人化成"主播语调铿锵"、"嗡鸣的金融电报"和"标题像蜉蝣"。当日的小新闻 GPT-6 内部代号 Spud 未经证实，在梦里变成"刚从凉土里拔出来的块茎，温热，不知道会长出什么"。

有意思的是，**同一段记忆在不同的 dreaming run 里可以写出完全不同的梦**——7/24 当晚的三段梦境分别从技术、人物、采集者三个视角切入同一天的素材，但都没有直接说出"GPT-6"或"智谱"三个字。这大概就是 REM 阶段"提取模式"的作用：把具体的事件抽象成可迁移的意象，让下次再撞上类似情况时，能从比喻层而不是字面层被调起。

副产物：偶尔 `dream generator` 跑空，会留下一句占位语 _"A memory trace surfaced, but details were unavailable in this run."_——这本身也成了一种稳定的梦境签名。

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