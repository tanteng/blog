---
title: "大模型是如何炼成的：从数据到预训练的完整技术栈"
date: 2026-06-28
draft: false
tags: ["ai", "machine-learning", "deep-learning", "llm", "transformer"]
categories: ["ai"]
description: "本文系统梳理从原始数据到千亿参数模型的完整训练 pipeline，涵盖数据工程、Tokenization、模型架构、预训练优化技术、以及 RLHF 等 post-training 关键步骤。"
---

大语言模型（LLM）的训练是一个涉及数据工程、分布式系统、GPU 架构、强化学习等多个领域的综合工程。本文把完整训练技术栈梳理清楚，让你对"模型是如何炼成的"有一个系统性认知。

<!--more-->

## 🏗️ 先看全局：大模型训练的四个阶段

```
数据收集 → 数据加工 → 预训练 → Post-training (SFT / RLHF / DPO)
```

| 阶段 | 核心目标 | 关键技术 |
| --- | --- | --- |
| **数据工程** | 把互联网原始数据变成"干净"的训练语料 | 去重、过滤、质量打分、毒性去除 |
| **Tokenization** | 把文字切成模型能处理的最小单位 | BPE / WordPiece / SentencePiece |
| **预训练** | 让模型学会"续写"文字 | Transformer + Next Token Prediction |
| **Post-training** | 让模型对齐人类意图、遵循指令 | SFT、RLHF、DPO |

接下来逐层拆解。

## 1️⃣ 数据工程：垃圾进，垃圾出

大模型训练数据量通常在万亿（Trillion） token 量级。以 GPT-3 为例，训练语料约 3000 亿 token。数据来源包括网页、书籍、代码、论文等。

**核心处理流程：**

```
原始数据 → 去重 → 质量过滤 → 安全过滤 → Tokenization → 训练语料
```

### 去重（Deduplication）

互联网上大量重复内容。如果不处理，模型会反复学习同一份数据，浪费算力且容易过拟合。常用 SimHash 或 MinHash 在文档级别做近似去重。

### 质量过滤（Quality Filtering）

- **规则过滤**：去除广告、垃圾文本、太短的页面
- **模型过滤**：用一个小模型（如 fastText）做二分类，预测文档质量分数，过滤低分样本
- **数学/代码检测**：GPT-4 的训练数据中对数学公式和代码片段有专门的质量筛选

### 安全过滤（Safety Filtering）

去除涉及暴力、仇恨言论、个人隐私（PII）等内容。这部分通常用关键词 + 分类模型双重过滤。

## 2️⃣ Tokenization：文字的数字翻译

Token 是模型处理文本的最小单元。Tokenization 就是把"今天天气真好"转换成 `[今天, 天气, 真好]` 这样的整数 ID 序列。

### 主流算法

| 算法 | 代表模型 | 特点 |
| --- | --- | --- |
| **BPE** (Byte Pair Encoding) | GPT-2、LLaMA | 从字节级别开始合并高频词对 |
| **WordPiece** | BERT | 优先保留完整词，遇到 OOV 再拆分 |
| **SentencePiece** | T5、Chinese models | 语言无关，把空格也当作 token |

### Tokenizer 的影响

- **词表大小**：LLaMA 使用 32K token 词表；GPT-4 据报道使用 100K+
- **中文效率**：中文在 BPE 下通常 1 个字 ≈ 1.5~2 个 token（英文 1 个词 ≈ 1.2 token）
- **推理成本**：token 越少，KV Cache 越小，推理越快

```python
# 用 tiktoken 体验 BPE Tokenization
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")  # GPT-4 同款
tokens = enc.encode("大语言模型是如何训练的？")
print(tokens)  # [123, 104, 112, 98, 58, 32, 85, 86, 82, 103, 101]
print(len(tokens), "tokens")  # 11 tokens
```

## 3️⃣ 预训练：Next Token Prediction

预训练的目标很简单：**给定前 N 个 token，预测第 N+1 个 token 是什么**。也叫 Causal Language Modeling（因果语言建模）。

### Transformer 架构是核心

2017 年 Google 提出 Transformer，之后几乎所有大模型都基于它。核心组件：

```
Input Tokens → [Embedding] → [Attention × N层] → [FFN] → ... → [Output Probs]
```

- **Self-Attention**：让每个 token "看到"上下文中的其他 token，计算它们之间的相关性权重
- **FFN（Feed-Forward Network）**：两层线性变换 + 激活函数，负责"非线性变换"
- **Residual Connection**：每层都有残差连接，缓解深层网络的梯度问题
- **LayerNorm**：稳定训练，移除非线性部分

### 混合精度训练（Mixed Precision）

FP32 全精度太慢、FP16 可能下溢。主流做法是：

- **权重存 BF16**（更宽的动态范围）
- **Forward/Backward 用 FP16 计算**
- **Optimizer 状态用 FP32**（防止精度崩塌）

这就是 NVIDIA Ampere 架构带来的重大突破，让 A100/H100 的算力能被充分用起来。

### 分布式训练：多卡协同

千亿参数单卡放不下，需要多卡甚至多机。主流方案：

| 方案 | 并行维度 | 代表框架 |
| --- | --- | --- |
| **Data Parallel（DP）** | 数据切分 | 常用，每个 GPU 跑不同 batch |
| **Tensor Parallel（TP）** | 单层权重切分 | Megatron-LM，单层矩阵乘法跨 GPU |
| **Pipeline Parallel（PP）** | 层切分 | 不同 GPU 跑不同层的序列 |
| **ZeRO** | 优化器状态分片 | DeepSpeed，显存优化 |

LLaMA 3 70B 用到了 **TP=4, PP=8** 的组合，配合 ZeRO-3 把 1400GB 权重分散到多卡。

### 优化器

大模型训练一般不用原生 SGD（随机梯度下降），而是使用**自适应学习率**的优化器：

- **AdamW**：带权重衰减的 Adam（实际训练 LLM 主流），每个参数有自己的自适应学习率
- **Adafactor**：比 AdamW 省显存，适合极长上下文
- **LAMB**：适合大 Batch Size 训练（如 2048+）

### 学习率调度（LR Schedule）

不是固定学习率，而是动态调整：

```
Warmup (0 → 峰值) → Cosine Annealing (峰值 → 0)
```

- **Warmup**：训练初期防止梯度爆炸，通常 2000~10000 步
- **Cosine Annealing**：平滑下降，让模型在训练末期更精细地收敛

### 梯度裁剪（Gradient Clipping）

大模型容易梯度爆炸。裁剪到 max norm = 1.0 是标准做法。

## 4️⃣ Post-training：从"续写"到"听话"

预训练模型本质上是一个"超级续写器"，但不一定听话。要让它学会遵循指令、回答问题，需要 Post-training。

### SFT：监督微调

用人工标注的"指令-回答"数据对，做有监督的 Fine-tuning。数据格式：

```
[System Prompt] 你是一个有帮助的助手
[User] 解释什么是光年
[Assistant] 光年是光在一年时间内走过的距离...
```

SFT 让模型知道"什么时候该回答，怎么回答"。

### RLHF：人类反馈强化学习

2022 年 InstructGPT 提出的三步流程：

1. **收集人类偏好数据**：对同一个问题，让模型生成多个回答，人类标注哪个更好
2. **训练 Reward Model**：用偏好数据训练一个奖励模型，学习"什么是好回答"
3. **PPO 强化学习**：用 Reward Model 的信号，通过 PPO 算法微调 SFT 模型

RLHF 是 ChatGPT 能对话、Claude 能遵循约束的关键技术。

### DPO：更简单的对齐方式

PPO 算法复杂、训练不稳定。2023 年出现的 **DPO（Direct Preference Optimization）** 用一个巧妙的方式绕过强化学习：

把"Reward Model + PPO"两步合并成一个**对比损失函数**，直接用人类偏好数据 Fine-tune。效果接近 RLHF，但实现简单得多。

### RLAIF：用 AI 反馈代替人类反馈

人类标注太贵太慢。用另一个大模型（如 Claude）来生成偏好标注，就是 RLAIF。

## 📊 大模型训练完整技术栈一览

{{< mermaid >}}
flowchart TB
    subgraph Data["📦 训练数据 Pipeline"]
        D1[🗑️ 去重<br/>SimHash / MinHash]
        D2[✅ 质量过滤<br/>规则 + fastText 分类]
        D3[🛡️ 安全过滤<br/>毒性 + PII 去除]
        D4[🔤 Tokenization<br/>BPE / SentencePiece]
        D1 --> D2 --> D3 --> D4
    end

    subgraph Arch["🏗️ 模型架构"]
        A1[Transformer]
        A2[Self-Attention<br/>Query/Key/Value]
        A3[FFN<br/>线性层 + 激活]
        A4[LayerNorm +<br/>Residual Connection]
        A1 --> A2 --> A3 --> A4
    end

    subgraph PreTrain["🚀 预训练"]
        P1[Next Token<br/>Prediction]
        P2[混合精度<br/>BF16 + FP32 Opt]
        P3[分布式训练<br/>TP / PP / DP + ZeRO]
        P4[AdamW 优化器]
        P5[Cosine LR +<br/>Warmup]
        P6[梯度裁剪<br/>max_norm=1.0]
        P1 --> P2 --> P3 --> P4 --> P5 --> P6
    end

    subgraph PostTrain["🎯 Post-training"]
        PT1[SFT<br/>监督微调]
        PT2[RLHF<br/>Reward Model + PPO]
        PT3[DPO<br/>直接偏好优化]
        PT4[RLAIF<br/>AI 反馈替代人类]
        PT1 --> PT2
        PT2 --> PT3
        PT3 --> PT4
    end

    Data --> Arch --> PreTrain --> PostTrain

    classDef stage fill:#1a1a2e,stroke:#5a67d8,color:#fff,stroke-width:2px,rx:10
    classDef tech fill:#2d3748,stroke:#718096,color:#a0aec0,stroke-width:1px,rx:6
    classDef highlight fill:#553c9a,stroke:#9f7aea,color:#fff,stroke-width:2px,rx:8
    classDef arrow color:#718096,stroke-width:2px

    class D1,D2,D3,D4,Arch,A1,A2,A3,A4,PreTrain,P1,P2,P3,P4,P5,P6,PT1,PT2,PT3,PT4 tech
    class Data,PostTrain highlight
{{< /mermaid >}}

## 🚀 总结

大模型训练是一个涉及数据工程、分布式系统、GPU 架构、强化学习等多个领域的综合工程。理解完整技术栈，对于系统性掌握 AI 工程化能力至关重要。

下次当你运行 `transformers` 库的 `Trainer` 训练模型时，背后其实是这整套系统在协同工作。
