---
title: "RAG 召回率提升的另一条路：HyDE 让问题先伪装成答案"
date: 2026-03-03T08:00:00+08:00
description: "Query 改写从'问题侧'把模糊问题变清晰，HyDE 从'答案侧'反向生成。两者思路相反，效果互补。"
categories: ['ai']
tags: ['ai', 'rag', 'hyde', 'embedding', 'vector-search', 'query-rewriting']
featured_image: ""
---

跑过 RAG 的同学大概都踩过这个坑：用户问"我的订单怎么取消？"，向量库里明明有那段"如需取消订单，请前往'我的订单'页面……"，但召回就是捞不回来。

问题出在哪？**问题和答案在向量空间里隔得很远**——"取消"虽然两边都有，但问句的语气、词汇结构、隐含的主语省略，都让它和那段陈述句形态的答案距离不近。BM25 兜不住（关键词只有两个字），向量检索也兜不住（语义结构差太大），怎么解？

**HyDE（Hypothetical Document Embeddings）** 就是专门处理这道题的。

<!--more-->

## 核心思路：让 LLM 假装先回答一下

HyDE 是 2022 年 CMU 和滑铁卢大学团队（Luyu Gao, Xueguang Ma, Jimmy Lin, Jamie Callan）在论文 *Precise Zero-Shot Dense Retrieval without Relevance Labels* 中提出的方法。思路异常简单：

> 既然用户的问题"长得不像"答案，那就让 LLM 先按问题的语义**想象一段答案**，然后拿这个**和答案同形的文本**去做检索。整条流水线只有一步多了 LLM：

{{< mermaid >}}
flowchart TD
    A["<b>用户问题</b>"] --> B["<b>LLM 生成假想答案</b><br/>temperature = 0<br/>不用事实准确，形态对就行"]
    B --> C["<b>embed 假想答案</b>"]
    C --> D["<b>向量检索</b><br/>ANN 索引"]
    D --> E["<b>top-K 真实文档</b>"]
    
    style A fill:#e3f2fd,stroke:#1976d2
    style B fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style C fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style D fill:#e8f5e9,stroke:#2e7d32
    style E fill:#e8f5e9,stroke:#2e7d32
{{< /mermaid >}}

假想答案**不需要事实正确**——事实上它经常一本正经地胡说。但 HyDE 不在乎事实，它只在乎文本的形态和语义空间位置。看个例子：

```
用户问题:  "退款流程复杂吗？"

HyDE 生成: "退款流程通常包括提交申请、客服审核、退款到账三个步骤，
           整体相对简单，一般需要 3-5 个工作日完成……"

真实文档: "退款流程包含三个步骤：1. 提交申请  2. 客服审核  3. 退款到账。
           整体耗时 3-5 个工作日。请在'我的订单'页面发起退款申请。"
```

注意 HyDE 输出里出现的「流程」「退款申请」「3-5 个工作日」——这些全是真实文档的关键词。句式也从问句变成了陈述句，**形态和真实文档完全一致**。内容细节有点出入无伤大雅，反正后续 LLM 生成阶段会基于真实文档重新组织答案。

这就是 HyDE 的精髓：**用形态换匹配，用想象换召回**。

## 和 Query 改写正好相反，但能叠加

很容易把 HyDE 和之前讲过的 Query 改写搞混——两者都用 LLM 预处理用户问题，但思路完全相反：Query 改写是把**模糊的问题改清楚**，HyDE 是把**问题伪装成答案**。

| | Query 改写 | HyDE |
|---|---|---|
| 优化方向 | 把问题**改清楚** | 把问题**伪装成答案** |
| LLM 输出 | 优化后的 query | 假想的答案 |
| 解决的问题 | 问题太糙、缺主语 | 问题和答案的形态鸿沟 |

选型的简单法则：

- Query 太短、太模糊、太口语化 → 优先 Query 改写
- Query 形态和文档差异大（问句 vs 陈述句）→ 优先 HyDE
- 拿不准就两个都上，叠加使用在多数场景能再提升 10-15%

两者**不是替代关系，是互补关系**。

实操上 HyDE 的实现非常短：

```python
PROMPT_HYDE = """请回答以下问题。回答要详细、准确，使用规范的书面语，
就像在写一篇知识库的条目一样。

问题：{user_query}
回答："""

def hyde_search(user_query, embed_fn, vector_db, top_k=5):
    hypo = llm.invoke(PROMPT_HYDE.format(user_query=user_query), temperature=0)
    vec = embed_fn(hypo)
    return vector_db.search(vec, top_k=top_k), hypo
```

三个细节决定 HyDE 用得好不好：

- **`temperature=0`** —— HyDE 要求每次检索生成同样的假想文本，否则召回结果会飘
- **用便宜的模型就够** —— gpt-4o-mini、qwen-turbo 都行，假想答案不需要事实准确，只需要"形态对"
- **高频 query 可以缓存** —— 避免每次检索都调 LLM，省延迟省成本

## 何时用，何时别用

HyDE 的代价很清楚：多 200-500ms 延迟、多 $0.0001-0.001 / 次成本、多一个失败点。性价比本身不错，但下面三种情况**别用**：

- **真实文档高度专业化**（医学、法律、企业内部术语）——LLM 的通用知识猜不到准确表述，假想答案反而把检索带偏
- **FAQ 类短答案问题**（"你们支持国际版吗？"）—— HyDE 生成的又长又啰嗦，反而稀释关键信号
- **多跳推理问题** —— 单步假想不够用

**最适合用 HyDE 的场景**：知识库语言规范（白皮书、产品手册、FAQ）+ 用户问题偏口语化（客服、内部问答）。这种组合下，召回质量通常能提升 10-20%。

> **2026 年补充**：当下生产环境的 RAG 标配是 **Hybrid Search（BM25+向量）+ Rerank + 强 Embedding**（如 BGE-M3、Qwen3-Embedding），HyDE 退居可选增强位。新项目选型建议先上三件套，召回还有明显缺口时**再叠加** HyDE。详见[《RAG 召回率优化的主路线》](/posts/rag-retrieval-modern-stack-hybrid-rerank/)。

---

HyDE 是 RAG 召回阶段的"答案侧"优化——和 Query 改写思路相反但效果互补。在文档规范、用户口语化的场景里提升最明显，代价是多一次 LLM 调用。

**参考资料：**

1. Gao, L., et al. (2022). *Precise Zero-Shot Dense Retrieval without Relevance Labels*. [arXiv:2212.10496](https://arxiv.org/abs/2212.10496) — HyDE 原始论文。
2. Gao, Y., et al. (2024). *Retrieval-Augmented Generation for Large Language Models: A Survey*. [arXiv:2312.10997](https://arxiv.org/abs/2312.10997) — HyDE 在 RAG 全景中的位置。