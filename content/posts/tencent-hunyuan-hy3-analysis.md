---
title: "腾讯混元 Hy3 深度解析：295B MoE、快慢思考融合、Agent 能力跃升"
date: 2026-07-09T16:30:00+08:00
tags: ["ai", "llm", "agent", "china-ai"]
categories: ["ai"]
description: "腾讯混元 Hy3 从 Preview 到正式版全面解读：295B 总参数 / 21B 激活参数的 MoE 架构、快慢思考融合机制、ClawEval 68.5 登顶 Agent 榜单，以及它和 DeepSeek、Qwen、Kimi 到底有什么不同。"
---

2026 年 4 月 23 日，腾讯混元发布并开源了 **Hy3 preview** 语言模型。这是混元团队在 2026 年 2 月重建预训练与强化学习基础设施后的**第一个新模型**，也是迄今混元最强的 LLM。短短两个多月后的 7 月 6 日，正式版（GA）也正式上线，Agent 任务解决率直接跃升到 90%，在 ClawEval pass³ 上拿到 68.5，超过了 DeepSeek V4 Pro 的 62.4 和 Qwen 3.7 Max 的 65.2。

本文围绕 Hy3 的架构、亮点、性能、与同体量模型的差异，以及"姚顺雨空降腾讯半年到底交出了什么答卷"这几个问题，做一次系统梳理。

<!--more-->

## 一、背景：从"混元掉队"到 Hy3 杀回第一梯队

混元团队去年一度被外界认为"动作慢了"。转折点是 2025 年 12 月，前 OpenAI 研究员**姚顺雨**空降腾讯，出任首席 AI 科学家、接掌混元团队。

他的节奏非常快：

- **2026 年 2 月**：重建预训练与强化学习的基础设施，并定下模型追求实用性的三个原则
- **2026 年 4 月 23 日**：发布 Hy3 preview 并开源
- **2026 年 5 月 13 日**：在腾讯一季报里，Hy3 preview 自 4 月 28 日上线以来在 OpenRouter 的 token 使用量持续排第一
- **2026 年 7 月 6 日**：Hy3 正式版上线，定价 1 元/百万 tokens 输入、4 元/百万 tokens 输出、缓存命中 0.25 元/百万 tokens

整个过程不到 7 个月。Hy3 也成为 2026 年国产开源大模型里"逆袭剧本"最具代表性的一个。

## 二、核心架构：295B / 21B MoE + MTP

Hy3 的架构参数如下（来自官方 GitHub README）：

| 项目 | 数值 |
|------|------|
| 架构 | Mixture-of-Experts (MoE) |
| 总参数 | 295B |
| 激活参数 | 21B |
| MTP 层参数 | 3.8B |
| 层数（含 MTP） | 80 + 1 |
| 注意力头 | 64（GQA，8 个 KV 头，head dim 128） |
| 隐藏维度 | 4096 |
| 中间层维度 | 13312 |
| 上下文长度 | 256K |
| 词表大小 | 120832 |
| 专家数 | 192 个专家，top-8 激活 |
| 支持精度 | BF16 |

几个值得专门拎出来说的设计点：

### 1. 192 专家 / top-8 激活

相比 Kimi-K2（384 专家 / top-8）、DeepSeek-V3（256 专家 / top-8），Hy3 的 192 专家数量更克制，但配合更密集的中间层维度（13312），单次激活的"有效参数密度"反而更高。换句话说，它走的是**少而精**的路线。

### 2. MTP（Multi-Token Prediction）层

3.8B 的 MTP 层参数专门用于多 token 预测，这让它在 vLLM 部署时可以开启 `--speculative-config.method mtp` 投机解码，推理吞吐比普通 dense 模型高一个量级。

### 3. 256K 上下文 + GQA 注意力

256K 已经是 2026 年开源旗舰的标配。但 Hy3 用 GQA（Grouped Query Attention，64 个 Q 头只对应 8 个 KV 头），在长上下文场景里把 KV cache 占用压缩到原来的 1/8，对显存友好。

## 三、最重要的创新：快慢思考融合

这是 Hy3 在推理机制上最大的差异点。它没有像 o1 / DeepSeek-R1 那样把"慢思考"做成独立模型，也没有像 Qwen3 那样靠用户手动切换 thinking 模式，而是**把"快思考"和"慢思考"融合到同一个模型权重里**，通过 `reasoning_effort` 参数控制：

```python
extra_body={"chat_template_kwargs": {"reasoning_effort": "no_think"}}
```

- `no_think`（默认）：直接回答，类似传统 instruct 模型
- `low`：轻度思考，平衡速度和质量
- `high`：深度 CoT，复杂任务用

这背后的设计哲学是**一个模型覆盖两种推理预算**——用户根据场景按需调用，而不是在两个模型之间切换。对部署和成本都非常友好。

## 四、性能表现：小激活参数打平大模型

Hy3 的"以小博大"是它最有杀伤力的点。先看预训练基模对比（数据来自官方 README）：

| Benchmark | Hy3 preview-Base (21B 激活) | Kimi-K2 BASE (32B) | DeepSeek-V3 BASE (37B) | GLM-4.5 BASE (32B) |
|-----------|--------------------------|--------------------|-----------------------|--------------------|
| 总参数 | 295B | 1043B | 671B | 355B |
| MMLU-Pro | **65.76** | 65.98 | 63.98 | 63.67 |
| LiveCodeBench-v6 | **34.86** | 30.86 | 29.31 | 27.43 |
| GSM8K | **95.37** | 93.46 | 88.15 | 90.06 |
| MATH | **76.28** | 71.20 | 59.37 | 61.00 |
| CMath | **91.17** | 90.83 | 85.50 | 89.33 |
| MMMLU | **80.15** | 77.63 | 79.54 | 79.26 |

Hy3 preview-Base 总参数只有 DeepSeek-V3 的 44%、Kimi-K2 的 28%，激活参数更是少 30%~40%，但在 MATH、GSM8K、LiveCodeBench、CMath、MMMLU 等多个核心基准上**全部拿下第一**。

## 五、Agent 能力：ClawEval 68.5 登顶

如果说基础能力是"打平"，那 Agent 能力就是 Hy3 真正甩开对手的地方。

正式版公开的两个核心 Agent 基准：

| 基准 | Hy3 正式版 | DeepSeek V4 Pro | Qwen 3.7 Max |
|------|-----------|-----------------|--------------|
| ClawEval pass³ | **68.5** | 62.4 | 65.2 |
| SkillsBench | **55.3** | — | — |
| Agent 任务解决率 | **90%** | — | — |

ClawEval 是腾讯自己提出的 Agent 评测，模拟真实业务中"多轮工具调用 + 长程任务规划"的场景。Hy3 在这套评测上领先了 3~6 个绝对百分点，说明它不是刷榜出来的"偏科选手"，而是真能在生产环境里干活。

内部业务侧的数据更直观：

- **WorkBuddy**（腾讯的 AI 工作台）接入 Hy3 后：首次响应速度提升 54%，任务平均完成时间缩短 47%，任务成功率保持 99.99%
- **OpenRouter**：Hy3 preview 上线后 token 消耗量连续多周排第一
- 全栈产品部署：腾讯云、CodeBuddy、WorkBuddy、元宝、QQ、腾讯文档、ima

## 六、姚顺雨提的"三条原则"

在 Hy3 发布的技术博客里，混元团队明确列出了重建后的三个实用性原则。我觉得这是 Hy3 区别于之前混元模型、也区别于其他国产开源旗舰的关键：

1. **能力体系化**：不推崇"偏科"。即使是代码智能体这一个应用，也涉及推理、长文、指令、对话、代码、工具等多种能力的深度协同。所以 Hy3 在每一个维度都做对齐，而不是把模型做成"代码天才但写作文不行"。
2. **评测真实性**：规避传统榜单"刷分"陷阱。混元团队明确说"不追求某一个 benchmark 的极端分数"，而是把评测放在真实业务场景里。
3. **性价比**：Hy3 定价 1 元/百万 tokens 输入、4 元/百万 tokens 输出，缓存命中价 0.25 元。这个价格放到 DeepSeek-V3 面前还是有竞争力（DeepSeek-V3 缓存命中价约 0.5 元/百万 tokens）。

## 七、跟同类模型的差异化对比

最后把 Hy3 放到 2026 年国产开源旗舰里横向对比一下：

| 维度 | Hy3 正式版 | DeepSeek-V4 Pro | Qwen 3.7 Max | Kimi-K2 |
|------|-----------|-----------------|--------------|---------|
| 总参数 | 295B | — | — | 1043B |
| 激活参数 | 21B | ~50B | ~22B | 32B |
| 上下文 | 256K | 1M | 1M | 128K |
| 架构 | MoE + MTP | MoE + MTP | MoE | MoE |
| 思考模式 | 融合（reasoning_effort） | 双模型切换 | 双模式 | 单一 |
| Agent 评测 | **ClawEval 68.5** | 62.4 | 65.2 | — |
| 输入价 | **1 元/M** | 约 2 元/M | 约 2 元/M | 约 4 元/M |
| 开源 | ✅ | ✅ | ✅ | ✅ |

几个有意思的差异：

- **Hy3 比 Kimi-K2 小一个数量级，效果却更好**：Kimi-K2 走"大而全"路线，总参 1043B；Hy3 用 295B 就打出了更强的 Agent 能力，说明激活效率比总参数更重要。
- **Hy3 是唯一把"快慢思考融合"做成单模型的**：其他几家要么是独立思考模型（如 DeepSeek-R1 系），要么是用户手动切模式（Qwen3）。Hy3 的"一个权重、三档 effort"是部署和成本上的创新。
- **Hy3 价格更激进**：输入价 1 元/百万 tokens，比 DeepSeek 和 Qwen 都低，配合 OpenRouter 这种第三方平台，对中小开发者和 Agent 类应用非常友好。
- **上下文不是最长**：256K 比不上 DeepSeek-V4 Pro 和 Qwen 3.7 Max 的 1M。但 Hy3 强调的是"真实业务里 256K 已经够用"，而不是堆参数。

## 八、部署方式

官方推荐用 vLLM 或 SGLang，最低 8 张 H20-3e（或者显存更大的 GPU）就能跑起来。vLLM 启动命令：

```bash
vllm serve tencent/Hy3-preview \
  --tensor-parallel-size 8 \
  --speculative-config.method mtp \
  --speculative-config.num_speculative_tokens 1 \
  --tool-call
```

也兼容 OpenAI SDK 调用方式，本地代码可以无缝迁移：

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

response = client.chat.completions.create(
    model="hy3-preview",
    messages=[{"role": "user", "content": "Hello!"}],
    extra_body={"chat_template_kwargs": {"reasoning_effort": "high"}},
)
```

模型权重在 Hugging Face、ModelScope、GitCode 都可以下载。许可证是 Tencent Hy Community License（注意：欧盟、英国、韩国不适用）。

## 九、总结

Hy3 的发布，让 2026 年国产开源大模型的竞争格局彻底变了。它不是参数最大的，也不是上下文最长的，但它在 Agent 真实业务能力上的领先是实打实的——ClawEval 68.5、WorkBuddy 业务数据、OpenRouter 排名第一，这些都是外部可验证的指标。

如果让我一句话总结 Hy3 的特点，我会说：

> **用更少的总参数和激活参数，把"基础能力 + Agent 能力 + 性价比"三件事同时做到第一梯队。**

这也是姚顺雨带给混元团队最大的改变——从"刷榜型选手"变成"业务能落地"的工程化模型。

接下来值得关注的是 Hy3 的多模态版本（混元一直有图像、视频、3D 全模态布局），以及混元团队是否会把同样的思路应用到其他模态。

---

**参考链接**

- 官方 GitHub：https://github.com/Tencent-Hunyuan/Hy3-preview
- Hugging Face：https://huggingface.co/tencent/Hy3-preview
- 官方介绍页：https://hunyuan.tencent.com/
- 腾讯云控制台：https://console.cloud.tencent.com/
