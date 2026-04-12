# TradingAgents-CN Skill 问题修复设计

## 问题概述

1. **新闻摘要不显示**：PDF 中新闻列表只有标题、日期、来源，摘要(summary)为空
2. **交易计划不显示具体价格**：决策为"观望"时显示"观望/待定/不适用"而非参考价格

## 根因分析

### 问题1：新闻摘要
- SKILL.md 要求 AI 根据 `web_search` 返回的 `title` + `snippet` 生成 `summary`
- 但 AI 执行时未正确生成 summary 字段
- `pdf_generator.py` 的预处理有 fallback 逻辑，但在 `_preprocess_data` 中处理的是原始 `news_list` 的引用，而非用户传入的原始数据

### 问题2：交易计划价格
- `trader_prompt.md` 规定：非"买入"决策时，`buy_price/target_price/stop_loss` 设为 `null`
- 用户期望：即使观望/卖出决策，也应显示基于当前价格的参考价格供决策参考

## 修复方案

### 1. 修改 `references/news_prompt.md`

**目标：** 明确要求新闻分析师必须输出完整的 news_list 结构

在输出格式部分，明确要求返回结构化 JSON：
```json
{
  "news_list": [
    {"title": "...", "date": "...", "source": "...", "summary": "...", "sentiment": "..."}
  ],
  "sentiment": "..."
}
```

### 2. 修改 `references/trader_prompt.md`

**目标：** 无论决策如何，都提供参考价格

将价格分为两类：
- **决策价格**：实际执行用的价格（买入决策时有值，观望时为 null）
- **参考价格**：基于 current_price 计算的理论价格（始终显示）

```json
{
  "decision": "买入/观望/卖出",
  "buy_price": 1.37,        // 决策用买入价，观望时 null
  "target_price": 1.64,      // 决策用目标价，观望时 null
  "stop_loss": 1.26,         // 决策用止损价，观望时 null
  "reference_price": 1.40,   // 当前价格参考
  "reference_target": 1.54,  // 基于 current_price 的理论目标
  "reference_stop": 1.26,    // 基于 current_price 的理论止损
  "position_size": "15%",
  "entry_criteria": "...",
  "exit_criteria": "..."
}
```

### 3. 修改 `SKILL.md`

**目标：** 强化 AI 生成 summary 的要求

在 Step 2 的新闻处理部分，增加：
```
⚠️ 强制要求：
- summary 字段必须填写，不能为空
- 如果 snippet 不为空，summary 必须基于 snippet 内容生成（不超过50字）
- 如果 snippet 为空，summary 必须基于 title 扩展描述（不超过50字）
- 绝对禁止出现 summary 为空的情况
```

### 4. 修改 `pdf_generator.py`

**目标：** 防御性补全 + 显示参考价格

#### 4.1 新闻摘要补全

在 `_preprocess_data` 中强化逻辑：
```python
for news in news_list:
    summary = news.get("summary") or ""
    if not summary.strip():
        # fallback 1: snippet
        summary = news.get("snippet") or ""
    if not summary.strip():
        # fallback 2: title 扩展
        title = news.get("title", "")
        summary = title  # 直接用 title
    news["summary"] = summary if summary.strip() else "暂无摘要"
```

#### 4.2 参考价格显示

在 `_preprocess_data` 中：
```python
current_price = result.get("current_price")
trading = result.get("trading_plan", {})

# 如果没有决策价格但有当前价格，计算参考价格
if not trading.get("buy_price") and current_price:
    trading["reference_buy"] = round(current_price * 0.98, 2)
    trading["reference_target"] = round(current_price * 1.10, 2)
    trading["reference_stop"] = round(current_price * 0.95, 2)
```

在 `_format_price` 方法中支持参考价格：
```python
def _format_price(price, fallback="N/A", reference=None):
    if price is None and reference is not None:
        return f"参考价 ¥{reference:.2f}"
    # ... 原逻辑
```

#### 4.3 HTML 模板更新

在 `交易计划` 区域同时显示决策价格和参考价格：
```html
<div class="target-prices">
    <div class="price-card">
        <div class="label">买入价格</div>
        <div class="value">{buy_price or reference_buy}</div>
    </div>
    ...
</div>
```

## 文件修改清单

| 文件路径 | 修改类型 | 修改内容 |
|---------|---------|---------|
| `references/news_prompt.md` | 修改 | 明确 JSON 输出格式 |
| `references/trader_prompt.md` | 修改 | 添加参考价格字段 |
| `SKILL.md` | 修改 | 强化 summary 生成规则 |
| `scripts/pdf_generator.py` | 修改 | 防御性补全 + 参考价格显示 |

## 测试计划

1. 使用无 summary 的新闻数据测试摘要显示
2. 使用"观望"决策测试参考价格显示
3. 端到端测试完整流程生成 PDF

## 风险评估

- 修改 trader_prompt 可能影响现有决策逻辑，需监控测试
- 修改 SKILL.md 需确保 AI 理解新规则
- 改动集中在 4 个文件，风险可控
