---
title: "提升 RAG 召回率的最低成本方案：3 个 Query 改写范式"
date: 2025-09-03T08:00:00+08:00
description: "RAG 召回率不高的常见原因不是 Embedding 不行，而是用户问得太模糊。本文讲 3 个 Query 改写实战范式：单查询改写、Multi-Query、Step-back Prompting，附实现代码和成本对比。"
categories: ['tech']
tags: ['ai', 'rag', 'llm', 'query-rewriting', 'multi-query', 'advanced']
---

跑过 RAG 的同学大概率踩过这个坑：**Embedding 模型选得很好、向量库也没问题，但召回质量就是不行**。

排查一圈下来，原因往往出乎意料——**不是 Embedding 不行，而是用户的问题太"糙"**：

```
"这个东西怎么用？"           ← 缺主语
"为什么我那个又不行了？"      ← 全是代词
"性能怎么样？"              ← 完全没头没尾
"a 和 b 哪个好？"           ← 缺上下文
```

这些问题直接喂给向量检索，召回的全是"东西"、"用"、"性能"这些泛词的相邻片段，**真正相关的文档反而捞不回来**。

解法就是 **Query 改写**：用 LLM 先把模糊问题优化成清晰的检索语句，再去检索。这招 2024 年开始被各家 RAG 框架（LangChain、LlamaIndex、Dify）纳入默认流程，性价比极高——**多花 1 次 LLM 调用，召回率提升 20-40%**。

下面讲 3 个实战范式，按推荐顺序排。

<!--more-->

## 范式一：Query Rewriting（单查询改写）

最基础的版本：让 LLM 把模糊问题改写成结构化的检索语句。

```python
PROMPT_REWRITE = """你是一个搜索查询优化助手。请将用户的原始问题改写为更适合向量检索的查询语句。

要求：
1. 补全缺失的主语、宾语
2. 替换代词（"那个"、"它"）为具体名词
3. 保留核心意图，不要添加原问题没有的信息

原始问题：{user_query}
改写后的查询："""

def rewrite_query(user_query: str, llm) -> str:
    return llm.invoke(PROMPT_REWRITE.format(user_query=user_query))
```

**例子：**

| 原始问题 | 改写后 |
|---|---|
| "这个东西怎么用？" | "OpenClaw Memory 系统的配置方法与使用示例" |
| "为什么又报错了？" | "OpenClaw Memory 写入失败的常见原因与排查步骤" |
| "a 和 b 哪个好？" | "LangChain 与 LlamaIndex 在企业 RAG 场景下的对比" |

**优点：** 简单、成本低（1 次 LLM 调用）。
**缺点：** 改写可能丢掉用户原本的"边角意图"——用户的"那个"也许就是不想说全，但改写强行补全了。

## 范式二：Multi-Query（多查询扩展）——**最实用**

Multi-Query 是 2024 年业界公认**性价比最高**的改写范式。核心思想：**同一个问题，让 LLM 生成 N 个不同角度的查询，并行检索后去重合并**。

```python
PROMPT_MULTI_QUERY = """你是一个搜索查询扩展助手。请基于用户的原始问题，从不同角度生成 3 个不同版本的检索查询。

要求：
1. 3 个查询覆盖原始问题的不同侧面（定义、实现、对比、案例等）
2. 每个查询必须独立可检索
3. 用 JSON 数组返回，不要其他解释

原始问题：{user_query}
输出格式：["查询1", "查询2", "查询3"]"""

def multi_query_search(user_query: str, llm, retriever, top_k=5):
    queries = json.loads(llm.invoke(PROMPT_MULTI_QUERY.format(user_query=user_query)))
    all_docs = []
    for q in queries:
        docs = retriever.invoke(q, top_k=top_k)
        all_docs.extend(docs)
    # 按 doc_id 去重
    unique_docs = list({d.metadata['doc_id']: d for d in all_docs}.values())
    return unique_docs[:top_k * 2]  # 多召回一些，供后续 Rerank
```

**例子：**

```
原始问题："为什么我的 Agent 老是忘记上下文？"

生成 3 个查询：
  1. "Agent 上下文丢失的常见原因与排查方法"
  2. "LLM 上下文窗口限制的技术原理"
  3. "OpenClaw 长期记忆机制的实现细节"

3 路并行检索 → 去重合并 → 召回率显著提升
```

**为什么有效？** 不同角度的查询能命中知识库的不同"切片"——一个查不到，另一个可能查得到。**这相当于免费让召回率翻倍**。

**关键细节：** 3 个就够了，5 个边际收益递减反而引入噪音。生成后记得加去重逻辑，否则同一文档被多次塞进上下文浪费 Token。

## 范式三：Step-back Prompting（抽象化）

这是 Google 在 2023 年提出的技巧，适合**需要"先建立背景再回答细节"的场景**。

```python
PROMPT_STEPBACK = """将用户的具体问题抽象为一个更通用、更高层次的问题，便于先检索背景知识。

原始问题：{user_query}
抽象后的问题："""

def step_back_search(user_query: str, llm, retriever):
    abstract_q = llm.invoke(PROMPT_STEPBACK.format(user_query=user_query))
    # 先用抽象问题检索背景
    bg_docs = retriever.invoke(abstract_q, top_k=3)
    # 再用原问题检索细节
    detail_docs = retriever.invoke(user_query, top_k=5)
    return bg_docs + detail_docs
```

**例子：**

| 原始问题 | Step-back 后 |
|---|---|
| "Redis 7.0 和 6.0 性能差多少？" | "Redis 主要版本的演进历史与性能改进" |
| "我这条 SQL 怎么优化？" | "MySQL 慢查询的通用优化方法论" |
| "Python 怎么读 Excel？" | "Python 文件处理的常用库与适用场景" |

**适用场景：** 技术对比、问题排查、工具选型类问题。**不适用：** 用户问的就是具体事实（如"X 公司的 Y 产品 Z 功能怎么用"），强行抽象反而答非所问。

## 实战选择：3 个怎么组合

**单独使用：**

- **首选 Multi-Query** —— 召回率提升最稳定，覆盖最广
- Query Rewriting 适合 LLM 成本极敏感的场景
- Step-back 只在"问题偏宏观、需要背景铺垫"时单独用

**组合使用（LangChain 2025 默认管线）：**

```
Multi-Query 生成 3 路 → 每路执行 Step-back → 合并去重 → Rerank → LLM 生成
```

这是最稳的组合，但**成本翻 4 倍**（1 次 Multi-Query + 3 次 Step-back）。中小项目不建议这么搞，先用 Multi-Query 就够了。

**成本对比：**

| 范式 | LLM 调用次数 | 相对召回率提升 | 适用阶段 |
|---|---|---|---|
| 无改写（baseline） | 0 | 1.0x | — |
| Query Rewriting | 1 | 1.1-1.2x | 成本敏感场景 |
| Multi-Query | 1（生成 3 路） | 1.3-1.5x | **首选** |
| Step-back | 1 | 1.2-1.4x | 宏观/对比类问题 |
| Multi-Query + Step-back | 4 | 1.5-1.8x | 关键业务 |

## 总结

Query 改写是 RAG 优化里**性价比最高的一招**——它不依赖任何额外基础设施（不用换 Embedding、换向量库），只是多 1 次 LLM 调用就能拿到 20-40% 的召回率提升。

**实操建议：** 第一版先上 Multi-Query，效果立竿见影；如果是大流量、成本敏感的场景，先用 Query Rewriting 顶上。等业务跑稳了，再考虑加 Step-back 做"问题分层"。

## 参考

- [LangChain Multi-Query Retriever](https://python.langchain.com/docs/how_to/MultiQueryRetriever/)
- [Step-back Prompting: Take a Step Back in LLMs (Google Research, 2023)](https://arxiv.org/abs/2310.06117)
- [Query Rewriting for Retrieval-Augmented Large Language Models (Huawei, 2023)](https://arxiv.org/abs/2305.14283)