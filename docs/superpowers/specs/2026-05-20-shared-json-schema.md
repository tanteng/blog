# TradingAgents-CN 共享数据文件规范

## 设计原则

1. **单一文件**：整个分析过程共用一个 JSON 文件，每个子步骤只写自己专属的字段
2. **中文键名**：所有数据分析结果的键使用中文，渲染层直接用中文
3. **原子写入**：每个步骤独立写入，完成后立即验证，不影响其他字段
4. **状态追踪**：用 `status` 字段记录每个步骤的完成状态

## 文件路径

```
{baseDir}/scripts/intermediate/{stock_code}_{YYYYMMDD}_{HHMMSS}.json
```

文件名示例：`PDD_20260520_092100.json`

## JSON 结构

```json
{
  "stock_code": "PDD",
  "stock_name": "拼多多",
  "current_price": 99.54,
  "timestamp": "2026-05-20T09:21:00+08:00",
  "news_data": [{"title": "...", "date": "...", "source": "...", "summary": "...", "sentiment": "..."}],
  "status": {
    "stock_data": "pending|done|failed",
    "news_data": "pending|done|failed",
    "多头分析": "pending|done|failed",
    "空头分析": "pending|done|failed",
    "技术分析": "pending|done|failed",
    "基本面分析": "pending|done|failed",
    "新闻分析": "pending|done|failed",
    "社交媒体分析": "pending|done|failed",
    "辩论过程": "pending|done|failed",
    "研究经理决策": "pending|done|failed",
    "交易计划": "pending|done|failed",
    "风险辩论": "pending|done|failed",
    "风险经理决策": "pending|done|failed"
  },
  "结果": {
    "股票数据": {
      "stock_code": "PDD",
      "stock_name": "拼多多",
      "current_price": 99.54,
      "涨跌幅": "+2.3%",
      "成交量": "850万股",
      "成交额": "8.5亿港元",
      "技术指标": {
        "MA5": 98.5,
        "MA10": 97.2,
        "MA20": 96.0,
        "MA60": 94.5,
        "RSI": 58,
        "MACD": "金叉",
        "KDJ": "多头",
        "BOLL_upper": 102.3,
        "BOLL_mid": 99.0,
        "BOLL_lower": 95.7
      },
      "基本面": {
        "PE": 18.5,
        "PB": 4.2,
        "ROE": "21%",
        "市值": "8500亿港元",
        "营收": "待获取",
        "净利润": "待获取"
      },
      "K线形态": "近5日缩量调整，均线多头排列"
    },
    "多头分析": {
      "core_logic": "核心逻辑（1-2句话）",
      "bull_case": ["论点1", "论点2", "论点3"],
      "risk_alert": "需要注意的风险",
      "confidenceindex": 0.75
    },
    "空头分析": {
      "core_logic": "核心逻辑（1-2句话）",
      "bear_case": ["论点1", "论点2", "论点3"],
      "valuation_risk": "估值风险",
      "downside_risk": "下行风险",
      "technical_alert": "技术面警示",
      "fundamental_concerns": "基本面担忧",
      "risk_events": "风险事件",
      "confidenceindex": 0.65
    },
    "技术分析": {
      "趋势判断": {"短期": "多头", "中期": "多头", "长期": "震荡"},
      "关键指标": {"MA5": "98.5", "MA10": "97.2", "RSI": "58", "MACD": "金叉"},
      "技术信号总结": "综合判断（50字以内）",
      "操作建议": {"支撑位": "95", "压力位": "105", "止损位": "92"}
    },
    "基本面分析": {
      "估值分析": {
        "当前市盈率（P/E）": "约18倍，处于历史中部",
        "同业比较": "较美股科技巨头折让30%",
        "PEG指标": "约0.9",
        "股息率": "约0.5%",
        "总结": "估值合理"
      },
      "盈利能力": {
        "毛利率": {"数值": "57%", "同比变化": "+1pct"},
        "净利率": {"数值": "34%", "同比变化": "+2pct"},
        "ROE": {"数值": "21%", "同比变化": "+2pct"}
      },
      "成长性": {
        "营收增速": "+9%",
        "净利润增速": "+11%",
        "PEG": "约0.9"
      },
      "财务健康": {
        "资产负债率": "约39%",
        "流动比率": "1.44",
        "经营现金流": "充裕"
      },
      "综合评价": "基本面优质，盈利能力行业领先"
    },
    "新闻分析": {
      "news_list": [
        {"title": "新闻标题", "date": "2026-05-18", "source": "来源", "summary": "摘要（≤50字）", "sentiment": "偏多"}
      ],
      "news_count": 5,
      "sentiment": "偏多"
    },
    "社交媒体分析": {
      "sentiment_score": 0.65,
      "platforms": [
        {"name": "雪球", "sentiment": "偏多", "heat": "高"},
        {"name": "东方财富", "sentiment": "中性", "heat": "中"}
      ]
    },
    "辩论过程": {
      "rounds": [
        {
          "round": 1,
          "bull_points": ["多头论点1", "多头论点2"],
          "bear_points": ["空头论点1", "空头论点2"]
        }
      ]
    },
    "研究经理决策": {
      "decision": "买入",
      "rationale": "核心逻辑（1-2句话）"
    },
    "交易计划": {
      "decision": "买入",
      "buy_price": 97.55,
      "target_price": 112.18,
      "stop_loss": 89.75,
      "position_size": "15%-20%",
      "entry_criteria": "回调至97.55元附近企稳入场",
      "exit_criteria": "跌破89.75元止损或达到112元目标"
    },
    "风险辩论": {
      "aggressive": {"position": "激进派", "position_size": "30%-40%", "target_return": "25%+", "stop_loss": "-12%"},
      "neutral": {"position": "中性派", "position_size": "15%-20%", "target_return": "10%-15%", "stop_loss": "-8%"},
      "conservative": {"position": "保守派", "position_size": "5%-10%", "target_return": "5%-8%", "stop_loss": "-5%"}
    },
    "风险经理决策": {
      "final_recommendation": "买入",
      "risk_level": "中",
      "investment_horizon": "6-12个月",
      "risk_assessment": {
        "市场风险": "中等",
        "流动性风险": "低",
        "波动性风险": "中等"
      },
      "suitable_investors": ["稳健型", "积极型"],
      "monitoring_points": ["季度财报", "行业政策", "技术面破位"]
    }
  }
}
```

## 各步骤写入字段映射

| 步骤 | JSON 路径 | 关键约束 |
|------|----------|---------|
| Step 1B | `结果.股票数据` + `status.stock_data` | current_price 必须是数字 |
| Step 2 | `news_data` + `status.news_data` | summary ≤50字，sentiment 只能是"偏多/偏空/中性" |
| Step 3 | `结果.多头分析` + `status.多头分析` | confidenceindex 0-1 浮点数 |
| Step 4 | `结果.空头分析` + `status.空头分析` | 同上 |
| Step 5 | `结果.技术分析` + `status.技术分析` | 趋势判断键名固定为"短期/中期/长期" |
| Step 6 | `结果.基本面分析` + `status.基本面分析` | 估值分析键名必须用中文 |
| Step 7 | `结果.新闻分析` + `status.新闻分析` | news_list 每条必须有 summary |
| Step 8 | `结果.社交媒体分析` + `status.社交媒体分析` | sentiment_score 0-1 |
| Step 9A | `结果.辩论过程` + `status.辩论过程` | rounds 是数组 |
| Step 9B | `结果.研究经理决策` + `status.研究经理决策` | decision 只能是"买入/卖出/持有" |
| Step 10 | `结果.交易计划` + `status.交易计划` | buy_price/target_price/stop_loss 必须是数字或 null |
| Step 11A | `结果.风险辩论` + `status.风险辩论` | 三派结构固定 |
| Step 11B | `结果.风险经理决策` + `status.风险经理决策` | risk_level 只能是"低/中/高" |
| Step 12 | 读取整个文件，渲染 PDF | 只需读文件，不做任何解析 |

## 文件操作规范

### 初始化（Step 1 前）
```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
JSON_FILE="{baseDir}/scripts/intermediate/{stock_code}_${TIMESTAMP}.json"
mkdir -p {baseDir}/scripts/intermediate

# 创建初始结构
cat > $JSON_FILE << 'EOF'
{
  "stock_code": "{stock_code}",
  "stock_name": "",
  "current_price": null,
  "timestamp": "",
  "news_data": [],
  "status": {
    "stock_data": "pending", "news_data": "pending",
    "多头分析": "pending", "空头分析": "pending", "技术分析": "pending",
    "基本面分析": "pending", "新闻分析": "pending", "社交媒体分析": "pending",
    "辩论过程": "pending", "研究经理决策": "pending", "交易计划": "pending",
    "风险辩论": "pending", "风险经理决策": "pending"
  },
  "结果": {}
}
EOF
```

### 子步骤写入（每个步骤完成后）
```bash
# 使用 python3 原子写入指定字段（不覆盖其他字段）
python3 -c "
import json, sys

path = '$JSON_FILE'
step_key = '$STEP_KEY'  # 如 '多头分析'
step_data = json.loads(sys.stdin.read())

with open(path, 'r') as f:
    data = json.load(f)

data['结果']['$STEP_KEY'] = step_data
data['status']['$STEP_KEY'] = 'done'

with open(path, 'w') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
"
```

### 主 Agent 读取（Step 12）
```bash
# 直接读取文件内容
cat $JSON_FILE
```

## 优势

1. **无上下文传参**：每个步骤的结果直接写文件，主 Agent 最后一次性读
2. **字段隔离**：一个步骤失败不影响其他步骤的状态
3. **中文键名**：PDF 模板直接用中文键名，无需映射表
4. **原子操作**：Python JSON 读写保证数据一致性
5. **可追溯**：分析过程全程记录在单一文件内