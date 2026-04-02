---
title: "一图看懂 Transformer 架构原理"
date: 2026-04-03T00:00:00+08:00
tags: ["ai", "transformer", "deep-learning", "nlp", "tech"]
categories: ["ai"]
description: "通过一张交互式架构图和核心公式，直观理解 Transformer 的完整原理。从 Encoder-Decoder 结构到 Multi-Head Attention，从位置编码到数据流全貌。"
---

Transformer 是当今大语言模型（GPT、BERT、T5 等）的基础架构，由 Google 在 2017 年论文 *"Attention Is All You Need"* 中提出。它彻底抛弃了 RNN 的递归结构，仅依靠**注意力机制**实现序列建模，在效果和效率上都带来了革命性突破。

本文通过一张架构图 + 核心公式 + 基础概念解释，帮你快速建立对 Transformer 的整体理解。

<!--more-->

## 先搞懂几个基础概念

在看架构图之前，先理解几个关键概念，后面会反复出现：

### Token（词元）

模型不直接处理文字，而是先把文本拆成小单元，叫 **Token**。比如 "我喜欢猫" 可能被拆成 `["我", "喜欢", "猫"]`，英文中 "unhappiness" 可能被拆成 `["un", "happiness"]`。每个 token 对应一个整数 ID。

### Embedding（嵌入）

Token ID 只是一个编号，没有语义信息。**Embedding** 是一个可学习的查找表，把每个 token ID 映射成一个高维向量（比如 512 维），这个向量就能表达语义。训练过程中，含义相近的词的向量会逐渐靠近。

### Softmax

Softmax 函数把一组任意实数转换成**概率分布**（所有值在 0~1 之间，且总和为 1）。比如输入 `[2.0, 1.0, 0.1]`，输出大约 `[0.66, 0.24, 0.10]`。在 Transformer 中，softmax 用来把注意力分数转化为权重。

### Q、K、V 的直觉理解

这是注意力机制的核心三元组，可以用**图书馆检索**来类比：

- **Query（查询）**：你想找什么——你带着一个问题去图书馆
- **Key（键）**：每本书的标签——图书馆里每本书都有一个描述标签
- **Value（值）**：书的实际内容——当标签匹配了你的查询，你就读到这本书的内容

具体来说：Query 和 Key 做点积来计算"匹配度"，匹配度高的 Key 对应的 Value 就会得到更大的权重。最终输出是所有 Value 的加权求和。

### Layer Normalization（层归一化）

神经网络训练时，每一层的输出数值范围可能差异很大，导致训练不稳定。**Layer Normalization** 对每个样本的所有特征做归一化（减均值、除标准差），让数值回到稳定范围，加速收敛。

## 架构总览

Transformer 采用经典的 **Encoder-Decoder（编码器-解码器）** 结构，下面这张图展示了完整的数据流：

<iframe src="/interactive/transformer-architecture.html" style="width: 100%; border: none; border-radius: 16px; overflow: hidden;" height="820" loading="lazy"></iframe>

## 核心公式

### 1. Scaled Dot-Product Attention（缩放点积注意力）

`Attention(Q, K, V) = softmax(QKᵀ / √d_k) · V`

- **Q**（Query）、**K**（Key）、**V**（Value）分别由输入经线性变换得到
- 点积 `QKᵀ` 衡量 Q 和 K 的相似度
- 除以 `√d_k` 防止数值过大导致 softmax 梯度消失
- softmax 归一化为权重，加权求和 V 得到输出

### 2. Multi-Head Attention（多头注意力）

`MultiHead(Q, K, V) = Concat(head₁, ..., headₕ) · Wᴼ`

其中 `headᵢ = Attention(QWᵢQ, KWᵢK, VWᵢV)`

将 Q、K、V 投影到 h 个不同的低维子空间，各自独立计算注意力，然后拼接后再做一次线性变换。**好处**：不同的头可以关注不同类型的关系（语法、语义、位置等）。

### 3. Positional Encoding（位置编码）

`PE(pos, 2i) = sin(pos / 10000^(2i/d_model))`

`PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))`

- **pos** 是 token 在序列中的位置，**i** 是维度索引
- 交替使用 sin 和 cos，使模型能通过线性变换推断相对位置

### 4. 数据流一句话总结

```
输入 → Embedding + PE → [Self-Attn → Add&Norm → FFN → Add&Norm] × N → 输出
```

每一层：**注意力**（捕捉全局依赖）→ **残差+归一化**（稳定训练）→ **FFN**（逐位置非线性变换）→ **残差+归一化**。编码器和解码器各堆叠 N 层（论文中 N=6），解码器额外多一个交叉注意力子层连接编码器。

## 关键设计思想

### 为什么不用 RNN？

在 Transformer 之前，处理序列数据（文本、语音等）主要靠 RNN（循环神经网络）。RNN 像流水线一样逐个处理 token，前一个处理完才能处理下一个。这带来两个致命问题：

| 维度 | RNN | Transformer |
|------|-----|-------------|
| 并行度 | 必须按序处理，无法并行 | 所有位置同时计算 ✅ |
| 长距离依赖 | 信息经过多步传递会衰减（梯度消失） | 任意两个位置直接 Attention 连接 ✅ |
| 训练效率 | 慢，序列越长越慢 | 快，充分利用 GPU 并行能力 ✅ |

### Self-Attention 的直觉

想象你在读一句话："**猫**坐在垫子上，因为**它**很累"。

Self-Attention 的工作方式是：**序列中的每个词，都会同时看到所有其他词**，并计算与它们的相关程度。当处理"它"这个词时，它和"猫"的注意力权重会特别高——于是模型理解了"它"指的是"猫"。

注意区分：
- **Self-Attention**（编码器中使用）：每个位置能看到**所有**位置，是双向的
- **Masked Self-Attention**（解码器中使用）：每个位置只能看到**当前及之前**的位置，屏蔽了未来信息，保证生成时不会"作弊"

### 残差连接（Residual Connection）的作用

每个子层都有残差连接：`output = LayerNorm(x + Sublayer(x))`

简单说就是把子层的输入 `x` 直接"跳过"子层加到输出上。这有两个好处：
- **梯度直通**：即使子层很深，梯度也能通过跳跃连接直接回传，避免梯度消失
- **保底机制**：最坏情况下子层学到"什么都不做"（输出零），整体至少还有原始输入，不会更差

这个技巧来自 ResNet，是训练深层网络（6层+）的关键。

## 训练与推理的区别

理解 Transformer 还需要知道**训练**和**推理**时的行为差异：

**训练时**：编码器和解码器同时处理完整的输入/输出序列。解码器使用 **Teacher Forcing**——直接把正确答案（而非模型预测）作为下一步的输入，配合 Mask 屏蔽未来信息，这样所有位置可以并行计算，效率很高。

**推理（生成）时**：解码器只能**逐个 token 生成**。每次预测一个新 token，把它拼到已有输出后面，再输入解码器预测下一个。这就是为什么 ChatGPT 回答问题是"一个字一个字蹦出来"的原因。

## 后续发展

Transformer 催生了三大变体：

| 变体 | 代表模型 | 特点 |
|------|---------|------|
| 仅 Encoder | BERT | 双向理解，适合分类、NER 等 |
| 仅 Decoder | GPT 系列 | 自回归生成，当前 LLM 主流 |
| Encoder-Decoder | T5、BART | 翻译、摘要等 seq2seq 任务 |

---

*参考资料：*
- *[Attention Is All You Need](https://arxiv.org/abs/1706.03762) (Vaswani et al., 2017)*
- *[The Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/) by Jay Alammar*
- *[The Annotated Transformer](http://nlp.seas.harvard.edu/annotated-transformer/) by Harvard NLP*
