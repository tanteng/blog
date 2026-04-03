---
title: "图解 Transformer 训练过程"
date: 2026-04-04T00:00:00+08:00
tags: ["ai", "transformer", "deep-learning", "nlp", "tech"]
categories: ["ai"]
description: "通过交互式动画，深入理解 LLM 自回归训练的核心原理：分词、Masked Self-Attention、多样本生成、交叉熵损失与反向传播。"
layout: "wide"
---

## 什么是自回归训练？

**自回归（Autoregressive）** 是大语言模型（GPT 系列）的核心训练方式。它的思想很简单：**给定前面的 token，预测下一个最可能的 token**。

这就像完形填空：给你前半句，让你在空格处填入最合适的词。

<!--more-->

## 为什么需要分词？

模型不能直接处理文字，需要先将文本转换成数字 ID。这个过程叫**分词（Tokenization）**。

主流模型使用 **BPE（Byte Pair Encoding）** 或 **SentencePiece** 等子词分词算法：

| 原始文本 | 分词结果 |
|---------|---------|
| 我爱机器学习 | `["我爱", "机器", "学习"]` |
| ChatGPT | `["Chat", "GPT"]` |
| Transformer | `["Trans", "former"]` |

子词分词的好处是**兼顾词义和覆盖度**，既能处理常见词，又能应对生僻词。

## 一句话如何产生多个训练样本？

以 `"我爱机器学习和人工智能技术"` 为例，分词后得到 6 个 token：

```
["我爱", "机器", "学习", "和", "人工智能", "技术"]
```

**每个 token 位置都会产生一个训练样本**：

| 样本 | 输入（给定上文） | 目标（预测下一个） |
|------|----------------|------------------|
| 1 | `"我爱"` | `"机器"` |
| 2 | `"我爱 机器"` | `"学习"` |
| 3 | `"我爱 机器 学习"` | `"和"` |
| 4 | `"我爱 机器 学习 和"` | `"人工智能"` |
| 5 | `"我爱 机器 学习 和 人工智能"` | `"技术"` |

**5 个 token 产生 5 个训练样本**。这就是为什么大规模语料可以提供海量训练数据——一本 10 万字的书，分词后可能有几万个 token，就能产生几万个训练样本。

## 模型如何学习？

### 第一步：前向传播

输入序列经由 **Embedding 层**转换为向量，然后通过多层 **Self-Attention** 和 **FFN** 网络，最终输出每个位置对所有 token 的预测概率分布。

### 第二步：计算损失

对于每个样本，模型输出一个概率分布 $P$。损失函数为**交叉熵**：

```
Loss = -ln(P(正确答案))
```

- 猜对了，概率高 → Loss 小 → 参数小幅调整
- 猜错了，概率低 → Loss 大 → 参数大幅调整

### 第三步：反向传播

Loss 通过 **Backpropagation** 反向传回网络，用**梯度下降**更新所有权重矩阵，使得下次给相同输入时正确答案的概率提高。

## 训练与推理的区别

| 阶段 | 特点 | 效率 |
|------|------|------|
| **训练** | Teacher Forcing：每一步输入都是标准答案，可以并行计算所有位置 | 高（一个 forward pass 处理整个序列） |
| **推理** | 自回归生成：每一步的输入是上一步的输出，必须逐个 token 生成 | 低（需要 N 次 forward pass 生成 N 个 token） |

## 真实训练流程

下面的交互式演示展示了从一句话产生多个训练样本的完整过程：

<iframe src="/interactive/transformer-training.html" style="width: 100%; border: none; border-radius: 16px; overflow: hidden;" height="820" loading="lazy"></iframe>

---

## 关键公式汇总

### 交叉熵损失

```
Loss = -ln(P(target_token | input_sequence))
```

### 反向传播更新

```
θ_new = θ_old - η × ∇Loss(θ_old)
```

其中 $\eta$ 是学习率，$\nabla$ 表示梯度。

---

*参考资料：*
- *[Attention Is All You Need](https://arxiv.org/abs/1706.03762) (Vaswani et al., 2017)*
- *[The Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/) by Jay Alammar*
- *[GPT-2 Working Draft](https://openai.com/blog/gpt-2-1-model-card/)*
