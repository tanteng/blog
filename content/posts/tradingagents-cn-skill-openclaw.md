---
title: "从论文到 Skill：基于 TradingAgents 架构打造中文股票多智能体分析框架"
date: 2026-04-11T00:30:00+08:00
draft: false
tags: ["ai", "agent", "llm", "investment", "openclaw", "tradingagents"]
categories: ["tech"]
description: "记录如何将 TradingAgents 论文中的多智能体交易框架思想，落地为一个 OpenClaw Skill，实现 4 位分析师 + 2 轮多空辩论 + 风控三方辩论 + 五级评级的完整中文股票分析流水线。"
---

最近把 [TradingAgents](https://github.com/TauricResearch/TradingAgents) 这个多智能体交易框架的核心思想，做成了一个 OpenClaw Skill —— [TradingAgents-CN-Skill](https://github.com/tanteng/TradingAgents-CN-Skill)。本文记录整个过程：从理解论文、提取 Prompt、到最终落地为一个可运行的 Skill。

<!--more-->

## 灵感来源：TradingAgents 论文

[TradingAgents](https://arxiv.org/abs/2412.20138) 是 Tauric Research 在 2025 年发表的论文，核心思想很简单也很聪明：

> **用 LLM 多智能体模拟一家真实交易公司的组织架构。**

真实的交易公司不是一个人做决策，而是一个组织化的团队——分析师、研究员、交易员、风控各司其职，通过讨论和制衡来提高决策质量。TradingAgents 把这套组织动态搬到了 LLM 多智能体系统中。

它的架构分四层：

```
市场数据 → [4位分析师] 多维度分析
                ↓
         [研究员团队] 多空辩论 ← 核心创新：对抗性推理
                ↓
         [交易员] 综合决策
                ↓
         [风控+组合经理] 风险把关 → 最终决策
```

原版项目用 Python + LangGraph 实现，通过 StateGraph 的条件分支控制辩论轮数，是一个很典型的有向状态图编排。

## 深入源码：提取 12 个 Agent 的 Prompt

要做 Skill，首先要理解原版的每一个 Prompt 在干什么。我克隆了仓库，逐个文件读取，提取出了全部 12 个角色的完整 Prompt：

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

## 从 LangGraph 到 SKILL.md

原版用 LangGraph 的 `StateGraph` 编排智能体，通过条件边控制辩论轮数：

```python
# 原版的辩论循环控制
def should_continue_debate(self, state):
    if state["investment_debate_state"]["count"] >= 2 * self.max_debate_rounds:
        return "Research Manager"  # 辩论够了，交给裁判
    if state["current_response"].startswith("Bull"):
        return "Bear Researcher"   # 轮到对方
    return "Bull Researcher"
```

而 OpenClaw 的 Skill 是 Agent 驱动模式——SKILL.md 定义流程，Agent 按步骤串行调用 LLM。没有 LangGraph 的状态图，但可以通过 SKILL.md 的步骤编排实现同样的信息流。

**关键取舍：**

- ✅ 保留：四位分析师 → 多空辩论 → 交易员 → 风控辩论 → 最终决策的信息流
- ✅ 保留：2 轮 Bull/Bear 辩论、风控三方独立发言
- ✅ 保留：五级评级体系
- ✅ 新增：中文 PDF 报告输出
- ❌ 暂不实现：记忆/反思机制（需要持久化存储，后续迭代）
- ❌ 简化：分析师无独立数据工具，依赖 web_search 获取新闻

## 最终的 17 步工作流

```
Step 1A-1B: 数据获取和结构化
Step 2:     web_search 获取新闻
───── 四位分析师 ─────
Step 3-6:   技术/基本面/新闻/情绪分析师各出一份报告
───── 多空辩论（2轮）─────
Step 7:     🐂 看多研究员 Round 1
Step 8:     🐻 看空研究员 Round 1（回应看多）
Step 9:     🐂 看多研究员 Round 2（反驳看空）
Step 10:    🐻 看空研究员 Round 2（反驳看多）
───── 裁决与交易 ─────
Step 11:    ⚖️ 研究管理者裁决
Step 12:    💹 交易员制定计划
───── 风控三方辩论 ─────
Step 13:    🔴 激进型风控
Step 14:    🟢 保守型风控（回应激进派）
Step 15:    🟡 中立型风控（回应双方）
───── 最终决策 ─────
Step 16:    👔 投资组合经理（五级评级）
Step 17:    📄 生成 PDF 报告
```

共 14 次 LLM 调用，比原版的 12 步多了几步，但信息流更完整。

## Prompt 的中文化改造

原版的 Prompt 是英文的，直接翻译效果不好。我做了几个关键改造：

**1. 分析框架本地化**

技术分析师的指标库从原版的美股指标（SMA/EMA/VWMA）调整为 A 股/港股常用的均线体系（MA5/MA10/MA20/MA30/MA60/MA120），加入了 KDJ 等国内常用指标。

**2. 辩论步骤输出纯文本**

分析师步骤输出 JSON（方便验证和 PDF 渲染），但辩论步骤输出纯中文文本——让 LLM 自由地对话式辩论，不受 JSON 格式约束。这是一个关键的设计决策：辩论质量比格式规范更重要。

**3. 交易员的价格约束**

原版交易员只输出方向，但中国投资者更关心具体价格。交易员 Prompt 要求必须输出具体的数字（买入价、目标价、止损价），并附带公式（如 `buy_price = P × 0.98`）帮助 LLM 计算。

**4. 五级评级中文化**

```
Buy → 买入
Overweight → 增持
Hold → 持有
Underweight → 减持
Sell → 卖出
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

## 项目地址

- **Skill 仓库**：[github.com/tanteng/TradingAgents-CN-Skill](https://github.com/tanteng/TradingAgents-CN-Skill)
- **灵感来源**：[github.com/TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents)
- **原版论文**：[arXiv:2412.20138](https://arxiv.org/abs/2412.20138)

> ⚠️ 免责声明：本框架仅供研究和学习目的，不构成任何形式的投资建议。
