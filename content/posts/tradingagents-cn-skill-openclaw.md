---
title: "基于 TradingAgents 框架打造的股票分析 Skill"
date: 2026-04-11T00:30:00+08:00
draft: false
tags: ["ai", "agent", "llm", "investment", "openclaw", "tradingagents"]
categories: ["tech"]
description: "记录如何将 TradingAgents 框架的核心思想落地为 OpenClaw Skill，实现 4 位分析师 + 2 轮多空辩论 + 风控三方辩论 + 五级评级的完整中文股票分析流水线。"
---

TradingAgents-CN-Skill 是基于 TradingAgents 框架的中文股票分析 Skill。用户只需输入股票代码，自动完成 4 位分析师 + 2 轮多空辩论 + 风控三方辩论 + 五级评级，输出完整 PDF 报告。

<!--more-->

框架包含五个阶段：

1. **数据获取** - 结构化提取 + 新闻搜索
2. **四位分析师** - 技术、基本面、新闻、情绪分析
3. **多空辩论** - 两轮 Bull/Bear 对话式辩论
4. **研究管理 + 交易** - 管理者裁决 + 交易员制定计划
5. **风控辩论 + 评级** - 激进/保守/中立三方辩论，投资组合经理给出五级评级

整体流程：

{{< mermaid >}}
flowchart TD
    A[股票代码] --> B[数据获取]
    B --> C[四位分析师报告]
    C --> D[多空辩论 R1]
    D --> E[多空辩论 R2]
    E --> F[研究管理者裁决]
    F --> G[交易员计划]
    G --> H[风控三方辩论]
    H --> I[投资组合经理评级]
    I --> J[PDF 报告]
{{< /mermaid >}}

## 项目地址

- **Skill 仓库**：[github.com/tanteng/tradingagents-cn-skill](https://github.com/tanteng/tradingagents-cn-skill)
- **灵感来源**：[github.com/TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents)
- **原版论文**：[arXiv:2412.20138](https://arxiv.org/abs/2412.20138)

## 灵感来源

TradingAgents 是 Tauric Research 开源的多智能体股票分析框架。真实的交易公司不是一个人做决策，而是一个组织化的团队——分析师、研究员、交易员、风控各司其职，通过讨论和制衡来提高决策质量。TradingAgents 把这套组织动态搬到了 LLM 多智能体系统中。

原版用 Python + LangGraph 实现，通过 StateGraph 的条件分支控制辩论轮数。TradingAgents-CN-Skill 则用 OpenClaw SKILL.md 驱动，信息流对齐但实现方式不同：

| 维度 | 原版 | 本项目 |
|------|------|--------|
| 编排方式 | LangGraph | OpenClaw SKILL.md |
| 语言 | 英文 | 中文 |
| 输出 | 终端文本 | PDF 报告 |
| 辩论轮次 | N轮 | 2轮 |

## 深入源码：提取 Prompt

要做 Skill，首先要理解原版的 Prompt 在干什么。克隆了仓库，逐个文件读取，提取出关键 Prompt：

| 角色 | 文件 | 核心职责 |
|------|------|---------|
| 技术/市场分析师 | `market_analyst.py` | 从 12 种技术指标中选 8 种进行分析 |
| 基本面分析师 | `fundamentals_analyst.py` | 分析财报、估值、现金流 |
| 新闻分析师 | `news_analyst.py` | 区分公司新闻和宏观新闻 |
| 情绪分析师 | `social_media_analyst.py` | 分析社交媒体舆情和公众情绪 |
| 看多研究员 | `bull_researcher.py` | 构建买入论证，回应看空观点 |
| 看空研究员 | `bear_researcher.py` | 构建卖出论证，回应看多观点 |
| 研究管理者 | `research_manager.py` | 辩论裁判，"不要默认 Hold" |
| 交易员 | `trader.py` | 制定具体交易计划 |
| 激进型风控 | `aggressive_debator.py` | 高风险高回报视角 |
| 保守型风控 | `conservative_debator.py` | 资产保护视角 |
| 中立型风控 | `neutral_debator.py` | 平衡风险收益视角 |
| 投资组合经理 | `portfolio_manager.py` | 最终决策，五级评级 |

### 几个关键设计发现

**1. 双层辩论机制**

这是整个框架最有意思的设计。不只是 Bull vs Bear 的投资方向辩论，还有第二层——激进/保守/中立的风险偏好辩论。两层辩论覆盖了"方向"和"力度"两个维度。

**2. 对话式辩论 Prompt**

原版 Prompt 明确要求 "conversational style, engaging directly with the other analyst's points and debating effectively rather than just listing data"——不是各说各的，而是真正的辩论。

**3. 记忆/反思机制**

原版每个关键角色都有 `FinancialSituationMemory`，交易结果出来后会反思并存储教训，下次注入 Prompt。这形成了跨交易的学习闭环。

**4. 双模型分工**

`deep_think_llm` 用于研究管理者和投资组合经理（关键决策），`quick_think_llm` 用于分析师和辩论者（量大但不需要极深推理）。

## Prompt 的中文化改造

原版 Prompt 是英文的，直接翻译效果不好。做了几个关键改造：

- **指标本地化**：从美股指标（SMA/EMA/VWMA）换为 A 股/港股常用体系（MA5/MA10/MA20/MA30/MA60/MA120 + KDJ）
- **辩论输出纯文本**：分析师步骤输出 JSON，辩论步骤输出纯中文文本，让 LLM 自由辩论不受格式约束
- **交易员价格约束**：输出具体买入价、目标价、止损价（附计算公式），而非仅方向
- **五级评级中文化**：Buy/Overweight/Hold/Underweight/Sell → 买入/增持/持有/减持/卖出

## 工作流程

共 10 步，数据流线性推进：

1. **数据获取**：结构化提取 + 新闻搜索
2. **四位分析师**：技术 / 基本面 / 新闻 / 情绪各出一份报告
3. **多空辩论 R1**：看多 / 看空研究员第一轮辩论
4. **多空辩论 R2**：看多 / 看空研究员第二轮辩论
5. **裁决与交易**：研究管理者裁决 + 交易员制定计划
6. **风控辩论**：激进 / 保守 / 中立三方独立发言
7. **最终决策**：投资组合经理给出五级评级
8. **PDF 生成**：`generate_report.py` 读取 `report.json` 输出报告

每步 LLM 输出经 `validate_step.py` 验证后自动写入 `report.json`，最终合并为完整报告。

## 数据管线设计

每步 LLM 输出后通过 `validate_step.py` 验证，自动写入 `report.json`，最后由 `generate_report.py` 生成 PDF。设计原则是：

> **Prompt 定义 schema → validate 验证 → pdf_generator 读取**

三端对齐，确保数据一致性。

文件结构：

```
tradingagents-cn-skill/
├── references/          # 各角色 Prompt 定义和 JSON schema
│   ├── tech_prompt.md
│   ├── fundamentals_prompt.md
│   ├── bull_prompt.md
│   ├── bear_prompt.md
│   └── ...
└── scripts/
    ├── validate_step.py
    ├── generate_report.py
    └── pdf_generator.py
```

## 工程上的几个经验

**validate_step.py 是整个 Skill 的安全网。** 每次 LLM 调用后都通过它验证输出，失败则附带 hint 重试。五级评级有专门的白名单验证——"强烈推荐"这种不在五级里的评级会被拒绝。

**adapt_data.py 做了格式适配。** 新的数据结构（辩论文本、五级评级）通过适配层转换为 PDF 生成器能理解的格式，避免了重写 600 多行的 HTML 模板。

**辩论步骤不做 validate。** 这是刻意的。辩论的价值在于自由思辨，如果强制 JSON 格式，LLM 会把精力花在格式对齐而不是论证质量上。

## 效果

最终的 PDF 报告包含：

- 📊 执行摘要（评级 + 一句话结论）
- 📈 四维分析报告（技术/基本面/新闻/情绪）
- 🐂🐻 完整的 2 轮多空辩论记录
- 💹 具体价格的交易计划
- 🔴🟢🟡 风控三方辩论记录
- 👔 投资组合经理最终裁决

整个分析流程大约 3-5 分钟完成（取决于 LLM 响应速度），输出一份 10+ 页的中文 PDF 报告。

> ⚠️ 免责声明：本框架仅供研究和学习目的，不构成任何形式的投资建议。

