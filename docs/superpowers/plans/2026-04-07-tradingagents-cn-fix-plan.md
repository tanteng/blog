# TradingAgents-CN Skill 问题修复实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 tradingagents-cn-skill 的两个问题：1) 新闻摘要不显示 2) 交易计划不显示具体价格

**Architecture:** 方案3彻底修复：修改 trader_prompt 和 news_prompt 强制输出规范 + pdf_generator 防御性补全

**Tech Stack:** Python, WeasyPrint, LLM API

---

## 文件修改清单

| 文件 | 路径 |
|------|------|
| `news_prompt.md` | `/root/.openclaw/workspace/skills/tradingagents-cn-skill/references/news_prompt.md` |
| `trader_prompt.md` | `/root/.openclaw/workspace/skills/tradingagents-cn-skill/references/trader_prompt.md` |
| `SKILL.md` | `/root/.openclaw/workspace/skills/tradingagents-cn-skill/SKILL.md` |
| `pdf_generator.py` | `/root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts/pdf_generator.py` |

---

## Task 1: 修改 news_prompt.md

**Files:**
- Modify: `/root/.openclaw/workspace/skills/tradingagents-cn-skill/references/news_prompt.md`

- [ ] **Step 1: 读取当前文件内容**

Run: `ssh root@43.134.180.240 "cat /root/.openclaw/workspace/skills/tradingagents-cn-skill/references/news_prompt.md"`

- [ ] **Step 2: 修改输出格式，明确要求 JSON 结构**

将输出格式部分改为：

```markdown
## 输出格式（必须返回 JSON）

```json
{
  "news_list": [
    {"title": "新闻标题", "date": "2026-04-07", "source": "来源", "summary": "基于snippet生成的摘要（不超过50字）", "sentiment": "偏多/偏空/中性"},
    ...
  ],
  "sentiment": "整体新闻情绪（积极/中性/消极）",
  "sentiment_score": 0.7
}
```

**注意：**
- `summary` 字段必须填写，不能为空
- 如果有 snippet，基于 snippet 内容生成摘要
- 如果没有 snippet，基于 title 扩展描述
- 返回至少 3 条新闻
```

- [ ] **Step 3: 验证修改**

Run: `ssh root@43.134.180.240 "grep -A 5 '输出格式' /root/.openclaw/workspace/skills/tradingagents-cn-skill/references/news_prompt.md"`

- [ ] **Step 4: 提交**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git add references/news_prompt.md && git commit -m 'fix: 强化news_prompt输出JSON结构和summary要求'"`

---

## Task 2: 修改 trader_prompt.md

**Files:**
- Modify: `/root/.openclaw/workspace/skills/tradingagents-cn-skill/references/trader_prompt.md`

- [ ] **Step 1: 读取当前文件内容**

Run: `ssh root@43.134.180.240 "cat /root/.openclaw/workspace/skills/tradingagents-cn-skill/references/trader_prompt.md"`

- [ ] **Step 2: 修改 JSON 输出格式，添加参考价格字段**

找到输出格式示例部分，修改为：

```json
{
    "decision": "买入",
    "buy_price": 1.37,
    "target_price": 1.54,
    "stop_loss": 1.26,
    "reference_price": 1.40,
    "reference_target": 1.54,
    "reference_stop": 1.26,
    "position_size": "15%-20%",
    "entry_criteria": "价格回调至1.37元附近企稳后入场",
    "exit_criteria": "跌破1.26元止损或达到1.54元目标"
}
```

**注意：** `reference_price` 是当前价格，`reference_target` 和 `reference_stop` 是基于当前价格计算的理论目标/止损。即使是"观望"决策，参考价格字段也要填写。

- [ ] **Step 3: 更新决策说明表格**

| 字段 | 买入决策 | 观望/卖出 |
|------|---------|----------|
| buy_price | P × 0.98 | null |
| target_price | buy_price × 1.10~1.20 | null |
| stop_loss | buy_price × 0.92~0.95 | null |
| reference_price | current_price | current_price |
| reference_target | current_price × 1.10 | current_price × 1.10 |
| reference_stop | current_price × 0.95 | current_price × 0.95 |

- [ ] **Step 4: 验证修改**

Run: `ssh root@43.134.180.240 "grep -c 'reference_price' /root/.openclaw/workspace/skills/tradingagents-cn-skill/references/trader_prompt.md"`

Expected: 3

- [ ] **Step 5: 提交**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git add references/trader_prompt.md && git commit -m 'fix: trader_prompt添加参考价格字段'"`

---

## Task 3: 修改 pdf_generator.py - 防御性补全

**Files:**
- Modify: `/root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts/pdf_generator.py`

- [ ] **Step 1: 读取 _preprocess_data 方法**

Run: `ssh root@43.134.180.240 "sed -n '57,90p' /root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts/pdf_generator.py"`

- [ ] **Step 2: 修改 _preprocess_data 中的新闻摘要逻辑**

找到：
```python
# 1. 修复新闻摘要
news_analyst = result.get("parallel_analysis", {}).get("news_analyst", {})
news_list = news_analyst.get("news_list", [])
for news in news_list:
    summary = (news.get("summary") or "").strip()
    if not summary:
        # fallback: snippet > title
        summary = (news.get("snippet") or "").strip()
    if not summary:
        summary = news.get("title", "暂无摘要")
    news["summary"] = summary
```

修改为强化版：
```python
# 1. 修复新闻摘要（防御性补全）
news_analyst = result.get("parallel_analysis", {}).get("news_analyst", {})
news_list = news_analyst.get("news_list", [])
for news in news_list:
    summary = (news.get("summary") or "").strip()
    if not summary:
        # fallback 1: snippet
        summary = (news.get("snippet") or "").strip()
    if not summary:
        # fallback 2: title 扩展
        title = news.get("title", "")
        summary = title  # 直接用 title
    news["summary"] = summary if summary.strip() else "暂无摘要"
```

- [ ] **Step 3: 添加参考价格计算逻辑**

在 _preprocess_data 方法中，找到交易价格修复部分（约在 line 75），在其后添加：

```python
# 3. 计算参考价格（如果缺少决策价格但有当前价格）
current_price = result.get("current_price")
trading = result.get("trading_plan", {})
if current_price and trading:
    # 如果没有决策价格但有当前价格，计算参考价格
    if trading.get("buy_price") is None:
        trading["reference_price"] = current_price
        trading["reference_target"] = round(current_price * 1.10, 2)
        trading["reference_stop"] = round(current_price * 0.95, 2)
```

- [ ] **Step 4: 修改 _format_price 方法支持参考价格**

找到 _format_price 方法，修改为：

```python
@staticmethod
def _format_price(price, fallback: str = "N/A", reference: float = None) -> str:
    """格式化价格显示，兼容数字和字符串类型"""
    if price is None and reference is not None:
        return f"参考 ¥{reference:.2f}"
    if price is None:
        return fallback
    if isinstance(price, (int, float)):
        return f"¥{price:.2f}"
    # 字符串类型：尝试转为数字
    try:
        return f"¥{float(price):.2f}"
    except (ValueError, TypeError):
        # 非数字字符串（如 "现行价格"）直接返回
        return str(price) if str(price).strip() else fallback
```

- [ ] **Step 5: 修改 _generate_html 中的价格显示**

找到交易计划区域（约 line 160-180），修改价格卡片显示：

```html
<div class="target-prices">
    <div class="price-card">
        <div class="label">买入价格</div>
        <div class="value">{self._format_price(trading.get("buy_price"), "观望", trading.get("reference_price"))}</div>
    </div>
    <div class="price-card">
        <div class="label">目标价格</div>
        <div class="value">{self._format_price(trading.get("target_price"), "待定", trading.get("reference_target"))}</div>
    </div>
    <div class="price-card">
        <div class="label">止损价格</div>
        <div class="value">{self._format_price(trading.get("stop_loss"), "不适用", trading.get("reference_stop"))}</div>
    </div>
</div>
```

- [ ] **Step 6: 测试修改**

Run:
```bash
ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts && python3 -c \"
import sys
sys.path.insert(0, '.')
from pdf_generator import ReportGenerator

result = {
    'stock_code': 'TEST',
    'timestamp': '2026-04-07',
    'current_price': 1.40,
    'parallel_analysis': {
        'news_analyst': {
            'news_list': [
                {'title': '测试新闻', 'date': '2026-04-07', 'source': '测试', 'sentiment': '偏多'},
            ],
            'news_count': 1
        },
        'bull_analyst': {'analysis': ['测试']},
        'bear_analyst': {'analysis': ['测试']},
        'tech_analyst': {'analysis': ['测试']},
        'fundamentals_analyst': {'analysis': ['测试']},
        'social_analyst': {'sentiment_score': 0.5, 'platforms': []}
    },
    'trading_plan': {'decision': '观望', 'buy_price': None, 'target_price': None, 'stop_loss': None},
    'manager_decision': {'decision': '观望', 'rationale': '测试'},
    'risk_debate': {},
    'final_decision': {'final_recommendation': '观望', 'risk_level': '中等'},
    'debate': {'rounds': []}
}

gen = ReportGenerator()
processed = gen._preprocess_data(result)
news = processed['parallel_analysis']['news_analyst']['news_list'][0]
trading = processed['trading_plan']
print('summary:', news.get('summary'))
print('reference_price:', trading.get('reference_price'))
print('reference_target:', trading.get('reference_target'))
print('reference_stop:', trading.get('reference_stop'))
\"
```

Expected:
```
summary: 测试新闻
reference_price: 1.4
reference_target: 1.54
reference_stop: 1.33
```

- [ ] **Step 7: 提交**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git add scripts/pdf_generator.py && git commit -m 'fix: pdf_generator强化summary补全和参考价格显示'"`

---

## Task 4: 修改 SKILL.md 强化 summary 要求

**Files:**
- Modify: `/root/.openclaw/workspace/skills/tradingagents-cn-skill/SKILL.md`

- [ ] **Step 1: 读取当前 SKILL.md 中关于 summary 的部分**

Run: `ssh root@43.134.180.240 "grep -n 'summary' /root/.openclaw/workspace/skills/tradingagents-cn-skill/SKILL.md | head -20"`

- [ ] **Step 2: 找到 Step 2 中关于 summary 的说明，替换为强化版本**

找到：
```markdown
**⚠️ 关键要求 - 新闻摘要（summary）**：
1. `web_search` 返回的是 `snippet` 字段，你**必须**基于 `title` + `snippet` 用自己的话**生成中文摘要**
2. 摘要不超过50字，概括新闻核心内容
3. **绝对不能**将 summary 留空、设为 None 或省略此字段
4. 如果 snippet 内容不足，至少根据 title 写一句摘要
5. 不要把 snippet 原文直接粘贴过来
```

替换为：
```markdown
**⚠️ 强制要求 - 新闻摘要（summary）**：
1. `web_search` 返回的是 `snippet` 字段，你**必须**基于 `title` + `snippet` 用自己的话**生成中文摘要**
2. 摘要不超过50字，概括新闻核心内容
3. **绝对不能**将 summary 留空、设为 None、空字符串或省略此字段
4. 如果 snippet 为空或内容不足，根据 title 扩展生成摘要
5. **每条新闻都必须有 summary**，不允许出现无摘要的新闻条目
6. 不要把 snippet 原文直接粘贴过来，要用自己的话概括
```

- [ ] **Step 3: 验证修改**

Run: `ssh root@43.134.180.240 "grep -c '绝对不能' /root/.openclaw/workspace/skills/tradingagents-cn-skill/SKILL.md"`

Expected: 1

- [ ] **Step 4: 提交**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git add SKILL.md && git commit -m 'fix: SKILL.md强化summary字段强制要求'"`

---

## Task 5: 端到端测试

- [ ] **Step 1: 运行完整测试**

Run:
```bash
ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts && python3 << 'PYEOF'
import sys
sys.path.insert(0, '.')
from analyst_multi import run_analysis

text_description = '''股票名称: 重庆钢铁 股票代码: 601005 当前价格: 1.40'''

news_data = [
    {'title': '重庆钢铁2025年年报：亏损收窄15%', 'date': '2026-04-05', 'source': '东方财富', 'sentiment': '偏多'},
    {'title': '钢铁行业景气度回升，基建需求带动钢价上涨', 'date': '2026-04-03', 'source': '我的钢铁网', 'sentiment': '偏多'},
]

result = run_analysis(
    stock_code='601005',
    stock_name='重庆钢铁',
    current_price=1.40,
    text_description=text_description,
    news_data=news_data,
)

print("=== 验证结果 ===")
news_list = result['analysis']['parallel_analysis']['news_analyst']['news_list']
for n in news_list:
    summary = n.get('summary', '')
    print(f"title: {n.get('title')}")
    print(f"  summary: '{summary}'")
    print(f"  summary OK: {bool(summary and summary.strip())}")

trading = result['analysis']['trading_plan']
print(f"\ntrading_plan:")
print(f"  buy_price: {trading.get('buy_price')}")
print(f"  target_price: {trading.get('target_price')}")
print(f"  stop_loss: {trading.get('stop_loss')}")
print(f"  reference_price: {trading.get('reference_price')}")
print(f"  reference_target: {trading.get('reference_target')}")
print(f"  reference_stop: {trading.get('reference_stop')}")

print(f"\nPDF: {result.get('pdf_path')}")
PYEOF"
```

Expected: 所有新闻都有非空 summary，且观望决策时 reference_* 字段有值

- [ ] **Step 2: 下载并检查 PDF**

PDF 路径: `/root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts/reports/`

下载命令:
```bash
scp root@43.134.180.240:/root/.openclaw/workspace/skills/tradingagents-cn-skill/scripts/reports/601005_*.pdf .
```

---

## Task 6: 合并提交

- [ ] **Step 1: 确认所有修改已提交**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git status"`

- [ ] **Step 2: 查看提交历史**

Run: `ssh root@43.134.180.240 "cd /root/.openclaw/workspace/skills/tradingagents-cn-skill && git log --oneline -6"`

Expected: 4 个新 commit

---

## 验证清单

- [ ] news_prompt.md 包含 JSON 输出格式要求
- [ ] trader_prompt.md 包含 reference_price/target/stop 字段
- [ ] SKILL.md 强化 summary 强制要求
- [ ] pdf_generator.py 防御性补全 summary
- [ ] pdf_generator.py 显示参考价格
- [ ] 端到端测试通过
- [ ] PDF 中新闻摘要显示正常
- [ ] PDF 中交易计划显示参考价格
