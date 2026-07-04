---
title: "大模型全栈拆解：从一堆数据到一个能对话的 AI"
date: 2026-06-28
draft: false
tags: ["ai", "machine-learning", "deep-learning", "llm", "transformer"]
categories: ["ai"]
description: "本文系统梳理从原始数据到千亿参数模型、再到线上推理服务的完整技术栈，涵盖数据工程、Tokenization、Transformer 架构、混合精度与分布式训练、RLHF/DPO 等 post-training，以及 KV Cache、量化、Flash Attention、投机解码等推理优化技术。"
---

大语言模型（LLM）的训练与部署，是一个横跨数据工程、分布式系统、GPU 架构、强化学习、推理服务等多个领域的综合工程。本文把从原始数据到线上推理的完整技术栈梳理清楚，让你对"模型是如何炼成的、又是如何跑起来的"有一个系统性认知。

<!--more-->

## 🏗️ 先看全局：从数据到服务的五个阶段

```
数据收集 → 数据加工 → 预训练 → Post-training (SFT / RLHF / DPO) → 推理部署
```

| 阶段 | 核心目标 | 关键技术 |
| --- | --- | --- |
| **数据工程** | 把互联网原始数据变成"干净"的训练语料 | 去重、过滤、质量打分、毒性去除 |
| **Tokenization** | 把文字切成模型能处理的最小单位 | BPE / WordPiece / Unigram（SentencePiece） |
| **预训练** | 让模型学会"续写"文字 | Transformer + Next Token Prediction |
| **Post-training** | 让模型对齐人类意图、遵循指令 | SFT、RLHF、DPO |
| **推理部署** | 让模型高效、低成本地对外服务 | KV Cache、量化、Flash Attention、投机解码 |

接下来逐层拆解。

## 1️⃣ 数据工程：垃圾进，垃圾出

大模型训练数据量通常在万亿（Trillion）token 量级。以 GPT-3 为例，训练语料约 3000 亿 token；到了 Llama 3 这一代，预训练语料已达约 15 万亿 token。数据来源包括网页、书籍、代码、论文等。

**为什么需要这么多数据？** 这背后有 **Scaling Law（缩放定律）** 支撑。DeepMind 的 Chinchilla 论文给出一个经验结论：在固定算力预算下，模型参数量和训练 token 数应当**同比例增长**，最优配比约为 **每 1 个参数配 ~20 个训练 token**。这解释了为什么一个 70B 的模型需要上万亿 token 才能"喂饱"。

**核心处理流程：**

```
原始数据 → 去重 → 质量过滤 → 安全过滤 → Tokenization → 训练语料
```

### 去重（Deduplication）

互联网上大量重复内容。如果不处理，模型会反复学习同一份数据，浪费算力且容易记忆化（memorization）。常用 **MinHash / SimHash** 在文档级别做近似去重，也会在子串级别（如 Suffix Array）做精确去重。

### 质量过滤（Quality Filtering）

- **规则过滤**：去除广告、垃圾文本、太短的页面、乱码
- **模型过滤**：用一个轻量分类器（如 fastText，或后来更常用的小型 LLM 打分器）预测文档质量分数，过滤低分样本
- **数学/代码专项**：现代模型（如 GPT-4、Llama 3）会对数学公式和代码片段做专门的质量筛选与配比调整

### 安全过滤（Safety Filtering）

去除涉及暴力、仇恨言论、个人隐私（PII）等内容。这部分通常用关键词 + 分类模型双重过滤。

## 2️⃣ Tokenization：文字的数字翻译

Token 是模型处理文本的最小单元。Tokenization 就是把"今天天气真好"转换成 `[今天, 天气, 真好]` 这样的整数 ID 序列。

### 主流算法

| 算法 | 代表模型 | 特点 |
| --- | --- | --- |
| **BPE** (Byte Pair Encoding) | GPT-2、GPT-4、Llama | 从字节级别开始，不断合并高频相邻字节/子词对 |
| **WordPiece** | BERT | 类似 BPE，但按"合并后能否提升语言模型似然"来选择合并对 |
| **Unigram** | T5、ALBERT | 从大词表出发，逐步删除对整体似然影响最小的子词 |

> ⚠️ 常见误区：**SentencePiece 不是一种算法，而是一个工具库**。它内部同时实现了 BPE 和 Unigram 两种算法，特点是把空格也编码成普通符号（`▁`），从而做到语言无关、无需预分词。所以严格来说，"SentencePiece" 与 "BPE/WordPiece" 不在同一层级。

### Tokenizer 的影响

- **词表大小**：Llama 2 使用 32K 词表，Llama 3 扩大到 128K；GPT-4（`cl100k_base`）约 10 万，GPT-4o（`o200k_base`）约 20 万
- **中文效率**：词表越大、对中文覆盖越好，同一句中文占用的 token 就越少（下面的实测对比很直观）
- **推理成本**：token 越少，KV Cache 越小、生成越快，API 计费也越低

```python
# 用 tiktoken 体验 BPE Tokenization（真实可复现的输出）
import tiktoken

s = "大语言模型是如何训练的？"

enc = tiktoken.get_encoding("cl100k_base")     # GPT-4 同款
tokens = enc.encode(s)
print(tokens)        # [27384, 73981, 78244, 54872, 25287, 21043, 30624, 99849, 10414, 255, 12774, 225, 9554, 11571]
print(len(tokens))   # 14 tokens

enc2 = tiktoken.get_encoding("o200k_base")     # GPT-4o 同款，词表更大、中文更省
print(len(enc2.encode(s)))   # 8 tokens
```

同一句中文，`cl100k` 需要 14 个 token，而词表更大的 `o200k` 只需 8 个——这就是词表大小对中文效率的直接影响。

## 3️⃣ 预训练：Next Token Prediction

预训练的目标很简单：**给定前 N 个 token，预测第 N+1 个 token 是什么**。也叫 Causal Language Modeling（因果语言建模），损失函数是对每个位置的交叉熵（Cross-Entropy）。

### Transformer 架构是核心

2017 年 Google 提出 Transformer，之后几乎所有大模型都基于它（准确说是其 Decoder-only 变体）。核心组件：

```
Input Tokens → [Embedding + 位置编码] → [ (Attention + FFN) × N层 ] → [Output Probs]
```

- **Self-Attention**：让每个 token "看到"上下文中的其他 token，通过 Query/Key/Value 计算它们之间的相关性权重。现代模型多用 **多头注意力（MHA）**，并演进出 **GQA（分组查询注意力）** 以减小 KV Cache
- **位置编码（Positional Encoding）**：Attention 本身对位置不敏感，必须显式注入位置信息。早期用绝对/正弦编码，现代主流是 **RoPE（旋转位置编码）**，它把位置信息编码进 Q/K 的旋转角度，天然支持相对位置与长度外推
- **FFN（Feed-Forward Network）**：两层线性变换 + 激活函数，负责逐位置的非线性变换。现代模型多用 **SwiGLU** 等门控激活替代传统 ReLU/GELU，效果更好
- **Residual Connection**：每层都有残差连接，缓解深层网络的梯度消失问题
- **LayerNorm / RMSNorm**：对激活值做归一化，稳定训练、加速收敛。现代 LLM 普遍采用 **Pre-LN**（归一化放在残差分支之前）而非原始的 Post-LN，训练更稳定；Llama 系列进一步用计算更省的 **RMSNorm** 替代 LayerNorm

### 混合精度训练（Mixed Precision）

FP32 全精度太慢、太占显存。主流做法是用 16 位精度做计算，同时保留一份 FP32 权重防止精度崩塌，标准配方是：

- **前向 / 反向计算用 16 位**：优先 **BF16**（指数位与 FP32 相同，动态范围大、几乎不溢出）；老硬件（如 V100）回退到 **FP16**，并需配合 Loss Scaling 防下溢
- **Optimizer 持有 FP32 master weights + FP32 优化器状态**：每步更新在 FP32 副本上进行，再 cast 回 BF16 供下一步计算，避免小梯度被"吃掉"

> 📌 澄清一个常见误解：**BF16 和 FP16 是两种互斥的半精度格式，不会"权重用 BF16、计算用 FP16"混着来**。一次训练要么走 BF16 路线、要么走 FP16 路线。此外，混合精度并非 Ampere 才有——它自 Volta（V100，2017）引入 Tensor Core 时就已可用；Ampere（A100）的真正贡献是**原生支持 BF16 和 TF32**，让 BF16 训练成为默认选择。

### 分布式训练：多卡协同

千亿参数单卡放不下，需要多卡甚至上万卡协同。主流并行维度：

| 方案 | 并行维度 | 说明 |
| --- | --- | --- |
| **Data Parallel（DP）** | 数据切分 | 每个 GPU 跑不同 batch，同步梯度；最基础的并行 |
| **Tensor Parallel（TP）** | 单层内切分 | 把一层的矩阵乘法按行/列切到多卡（Megatron-LM） |
| **Pipeline Parallel（PP）** | 按层切分 | 把模型按层切成多段，不同 GPU 持有不同层，像流水线一样传递激活 |
| **ZeRO / FSDP** | 状态分片 | 把参数、梯度、优化器状态分片到各卡，大幅省显存（DeepSpeed / PyTorch FSDP） |

以 **Llama 3.1 405B** 为例，Meta 官方采用了 **4D 并行**：FSDP + TP + PP + CP（上下文并行），在上万张 H100 上训练。作为直观对照：一个 70B 模型的 BF16 权重约 **140GB**（70B × 2 字节），已远超单卡 80GB 显存，因此必须切分。

### 优化器

大模型训练一般不用原生 SGD（随机梯度下降），而是使用**自适应学习率**的优化器：

- **AdamW**：带权重衰减的 Adam，是训练 LLM 的绝对主流；每个参数维护一阶/二阶动量，代价是优化器状态占 2× 模型大小的显存
- **Adafactor**：用低秩分解近似二阶动量，比 AdamW 省显存
- **LAMB**：为超大 Batch Size（如 数千+）训练设计

### 学习率调度（LR Schedule）

不是固定学习率，而是动态调整：

```
Warmup (0 → 峰值) → Cosine Annealing (峰值 → 最小值)
```

- **Warmup**：训练初期从 0 线性升到峰值，防止早期梯度不稳，通常 2000~10000 步
- **Cosine Annealing**：之后按余弦曲线平滑下降到一个较小值，让模型在训练末期更精细地收敛

### 梯度裁剪（Gradient Clipping）

大模型容易出现梯度爆炸。按全局范数裁剪到 **max norm = 1.0** 是标准做法。

## 4️⃣ Post-training：从"续写"到"听话"

预训练模型本质上是一个"超级续写器"，但不一定听话。要让它学会遵循指令、回答问题、符合价值观，需要 Post-training。

### SFT：监督微调

用人工标注的"指令-回答"数据对，做有监督的 Fine-tuning。数据格式：

```
[System Prompt] 你是一个有帮助的助手
[User] 解释什么是光年
[Assistant] 光年是光在一年时间内走过的距离...
```

SFT 让模型知道"什么时候该回答、怎么回答"，建立起对话/指令的基本行为模式。

### RLHF：人类反馈强化学习

2022 年 InstructGPT 提出的三步流程：

1. **收集人类偏好数据**：对同一个问题，让模型生成多个回答，人类标注哪个更好
2. **训练 Reward Model**：用偏好数据训练一个奖励模型，学习"什么是好回答"
3. **PPO 强化学习**：用 Reward Model 的信号，通过 PPO 算法微调 SFT 模型（同时用 KL 惩罚防止模型偏离 SFT 太远）

RLHF 是 ChatGPT 能对话、Claude 能遵循约束的关键技术。

### DPO：更简单的对齐方式

PPO 算法复杂、需要同时加载 4 个模型（策略、参考、奖励、Critic），训练不稳定且吃显存。2023 年出现的 **DPO（Direct Preference Optimization）** 用一个巧妙的推导绕过了显式强化学习：

它把"训练 Reward Model + PPO 优化"两步，合并成一个直接作用在偏好数据对上的**对比损失函数**，只需策略模型和参考模型两个网络。实现简单得多，效果接近 RLHF，已成为开源社区对齐的主流选择。

### RLAIF / RLVR：反馈信号的演进

- **RLAIF**：人类标注太贵太慢，改用另一个强大的 LLM 来生成偏好标注（Anthropic 的 Constitutional AI 是代表）
- **RLVR（可验证奖励的强化学习）**：在数学、代码等**有标准答案**的场景，直接用"答案是否正确"作为奖励信号，无需奖励模型。这是近来推理模型（如 o1、DeepSeek-R1 一类）训练的重要思路

## 5️⃣ 推理部署：让模型跑得起、跑得快

训练出模型只是开始，让它在线上**低延迟、高吞吐、低成本**地服务，是另一整套工程。LLM 推理分两个阶段：**Prefill（并行处理完整 prompt）** 和 **Decode（逐 token 自回归生成）**，后者是逐个 token 串行的，也是延迟的主要来源。

### KV Cache：自回归生成的基石

生成第 N+1 个 token 时，前 N 个 token 的 Key/Value 其实每步都一样。**KV Cache** 就是把它们缓存下来，避免重复计算，让每步生成从 O(N²) 降到 O(N)。代价是显存占用随上下文长度线性增长——这也是长上下文推理的主要瓶颈。GQA、MQA 等注意力变体正是为压缩 KV Cache 而生。

### PagedAttention 与 vLLM

传统 KV Cache 按最大长度预分配，碎片化严重、利用率低。**vLLM** 提出的 **PagedAttention** 借鉴操作系统虚拟内存分页思想，把 KV Cache 切成小块按需分配，显著提升显存利用率和并发吞吐，已成为主流推理引擎的标配（同类还有 TensorRT-LLM、SGLang 等）。

### Flash Attention：更快的注意力

**Flash Attention** 通过分块计算 + 融合算子，避免把巨大的 N×N 注意力矩阵写回显存（HBM），大幅减少访存开销。它是**训练和推理都受益**的核心优化，现已是事实标准。

### 量化（Quantization）：用更低精度换空间和速度

把权重（甚至激活）从 FP16 压到 INT8 / INT4，显存和带宽需求大幅下降：

- **权重量化**：GPTQ、AWQ 是最常用的 4-bit 训练后量化（PTQ）方案，几乎无损压缩到 INT4
- **KV Cache 量化**：把缓存也量化（如 FP8/INT8），进一步降低长上下文显存
- **代价**：极低比特会带来一定精度损失，需要在压缩率和质量间权衡

### 投机解码（Speculative Decoding）

用一个**小而快的草稿模型**一次性猜测多个 token，再让大模型**一次并行验证**这些猜测，接受正确的部分。由于验证是并行的，能在**不损失输出质量**的前提下把生成速度提升 2~3 倍。变体包括 Medusa、EAGLE 等。

### 连续批处理（Continuous Batching）

不同请求的生成长度不一。传统静态批处理要等最长的请求完成才能释放。**Continuous Batching**（又称 in-flight batching）在每一步动态地把已完成的请求换出、新请求换入，把 GPU 利用率拉满，是提升吞吐的关键调度技术。

## 🧩 补充：MoE —— 当今旗舰模型的架构选择

值得单独一提的是 **MoE（Mixture of Experts，混合专家）**。它把 FFN 层替换成多个"专家"网络，每个 token 只激活其中少数几个（由门控网络路由）。这样能在**总参数量巨大**的同时，保持**单次前向的激活参数量（计算量）可控**。GPT-4、Mixtral、DeepSeek-V3 等当代旗舰模型普遍采用 MoE 架构，是"参数越来越大、推理成本却没同比例暴涨"的重要原因。

## 📊 大模型完整技术栈一览

{{< mermaid >}}
flowchart TB
    subgraph Data["📦 数据 Pipeline"]
        D1[🗑️ 去重<br/>MinHash / SimHash]
        D2[✅ 质量过滤<br/>规则 + 分类器]
        D3[🛡️ 安全过滤<br/>毒性 + PII 去除]
        D4[🔤 Tokenization<br/>BPE / Unigram]
        D1 --> D2 --> D3 --> D4
    end

    subgraph Arch["🏗️ 模型架构"]
        A1[Transformer<br/>Decoder-only]
        A2[Self-Attention<br/>MHA / GQA + RoPE]
        A3[FFN<br/>SwiGLU]
        A4[Pre-LN / RMSNorm<br/>+ Residual]
        A1 --> A2 --> A3 --> A4
    end

    subgraph PreTrain["🚀 预训练"]
        P1[Next Token<br/>Prediction]
        P2[混合精度<br/>BF16 + FP32 Master]
        P3[分布式训练<br/>DP / TP / PP + FSDP]
        P4[AdamW 优化器]
        P5[Cosine LR +<br/>Warmup]
        P6[梯度裁剪<br/>max_norm=1.0]
        P1 --> P2 --> P3 --> P4 --> P5 --> P6
    end

    subgraph PostTrain["🎯 Post-training"]
        PT1[SFT<br/>监督微调]
        PT2[RLHF<br/>Reward Model + PPO]
        PT3[DPO<br/>直接偏好优化]
        PT4[RLAIF / RLVR<br/>AI / 可验证奖励]
        PT1 --> PT2 --> PT3 --> PT4
    end

    subgraph Infer["⚡ 推理部署"]
        I1[KV Cache]
        I2[PagedAttention<br/>vLLM]
        I3[Flash Attention]
        I4[量化<br/>GPTQ / AWQ / INT4]
        I5[投机解码 +<br/>连续批处理]
        I1 --> I2 --> I3 --> I4 --> I5
    end

    Data --> Arch --> PreTrain --> PostTrain --> Infer

    classDef tech fill:#2d3748,stroke:#718096,color:#a0aec0,stroke-width:1px,rx:6
    classDef highlight fill:#553c9a,stroke:#9f7aea,color:#fff,stroke-width:2px,rx:8

    class D1,D2,D3,D4,A1,A2,A3,A4,P1,P2,P3,P4,P5,P6,PT1,PT2,PT3,PT4,I1,I2,I3,I4,I5 tech
    class Data,Arch,PreTrain,PostTrain,Infer highlight
{{< /mermaid >}}

## 🚀 总结

大模型是一个横跨数据工程、分布式系统、GPU 架构、强化学习、推理服务的综合工程：

- **数据工程**决定了模型的知识上限（垃圾进，垃圾出）
- **Transformer + 位置编码 + 归一化**构成了模型的骨架
- **混合精度 + 分布式并行**让千亿参数训练在物理上成为可能
- **SFT / RLHF / DPO** 把"续写器"调教成"听话的助手"
- **KV Cache / 量化 / Flash Attention / 投机解码**让它以可接受的成本对外服务

理解这条完整链路，对于系统性掌握 AI 工程化能力至关重要。下次当你运行 `transformers` 的 `Trainer` 训练、或用 `vLLM` 起一个推理服务时，背后其实是这整套系统在协同工作。
