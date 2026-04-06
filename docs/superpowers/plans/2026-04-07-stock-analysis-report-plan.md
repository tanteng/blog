# Stock Analysis Report Skill 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个 Skill，用户粘贴股票分析文本后，基于 TradingAgents-CN 的 13 类 Agent 框架生成完整 PDF 分析报告。

**Architecture:** 纯 Prompt 方案 - Skill 加载 13 类 Agent 的 prompt 模板，在对话中完成分析，生成 HTML 后上传服务器转 PDF。

**Tech Stack:** WeasyPrint, Python 3, 中文字体, SSH/SCP

---

## 文件结构

```
~/.claude/skills/stock-analysis-report/
├── skill.md              # 主 skill 文件，包含 13 类 Agent prompt 模板
├── report_template.html  # HTML 报告模板
├── generate_pdf.sh       # PDF 生成脚本（服务器端）
└── style.css             # 报告样式
```

---

## Task 1: 服务器环境准备

**Files:**
- Modify: 服务器 `/opt/stock-analysis/` 目录

- [ ] **Step 1: SSH 连接到服务器并创建目录**

```bash
ssh root@43.134.180.240 "mkdir -p /opt/stock-analysis && ls -la /opt/stock-analysis"
```

- [ ] **Step 2: 安装 WeasyPrint 和依赖**

```bash
ssh root@43.134.180.240 "pip3 install weasyprint && pip3 list | grep weasyprint"
```

- [ ] **Step 3: 检查/安装中文字体**

```bash
ssh root@43.134.180.240 "fc-list :lang=zh | head -5"
```

如果无输出，执行：
```bash
ssh root@43.134.180.240 "yum install -y fontconfig && fc-cache -fv"
```

- [ ] **Step 4: 下载思源黑体字体**

```bash
ssh root@43.134.180.240 "cd /opt/stock-analysis && wget -q https://github.com/googlefonts/noto-cjk/releases/download/Sans2.004R/03_SourceHanSans.zip && unzip -q 03_SourceHanSans.zip && mv SourceHanSansSC/*.otf . && rm -rf SourceHanSansSC 03_SourceHanSans.zip"
```

- [ ] **Step 5: 验证字体安装**

```bash
ssh root@43.134.180.240 "fc-list :lang=zh | grep -i noto"
```

Expected: 包含 NotoSansCJK 的行

---

## Task 2: 创建 HTML 报告模板

**Files:**
- Create: `~/.claude/skills/stock-analysis-report/report_template.html`
- Create: `~/.claude/skills/stock-analysis-report/style.css`

- [ ] **Step 1: 创建 style.css**

```css
@page {
    size: A4;
    margin: 2cm;
    @top-center {
        content: "股票分析报告";
    }
    @bottom-center {
        content: "第 " counter(page) " 页";
    }
}

body {
    font-family: "Noto Sans CJK SC", "Source Han Sans SC", "SimHei", sans-serif;
    font-size: 12pt;
    line-height: 1.6;
    color: #333;
}

h1 {
    font-size: 24pt;
    color: #1a1a1a;
    border-bottom: 2px solid #333;
    padding-bottom: 10px;
}

h2 {
    font-size: 18pt;
    color: #2c3e50;
    margin-top: 30px;
    border-left: 4px solid #3498db;
    padding-left: 10px;
}

h3 {
    font-size: 14pt;
    color: #34495e;
    margin-top: 20px;
}

.section {
    margin-bottom: 25px;
}

.stock-info {
    background-color: #f8f9fa;
    padding: 15px;
    border-radius: 5px;
    margin-bottom: 20px;
}

.bull {
    color: #c0392b;
}

.bear {
    color: #27ae60;
}

.signal-buy {
    color: #e74c3c;
    font-weight: bold;
}

.signal-sell {
    color: #27ae60;
    font-weight: bold;
}

.signal-hold {
    color: #f39c12;
    font-weight: bold;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin: 15px 0;
}

th, td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: left;
}

th {
    background-color: #3498db;
    color: white;
}

tr:nth-child(even) {
    background-color: #f9f9f9;
}

.footer {
    margin-top: 40px;
    padding-top: 20px;
    border-top: 1px solid #ddd;
    font-size: 10pt;
    color: #7f8c8d;
}
```

- [ ] **Step 2: 创建 report_template.html**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>{{TITLE}}</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h1>{{TITLE}}</h1>

    <div class="stock-info">
        <p><strong>股票代码：</strong>{{TICKER}}</p>
        <p><strong>公司名称：</strong>{{COMPANY_NAME}}</p>
        <p><strong>分析日期：</strong>{{ANALYSIS_DATE}}</p>
        <p><strong>所属市场：</strong>{{MARKET}}</p>
    </div>

    <h2>📊 多空辩论</h2>
    <div class="section">
        <h3 class="bull">多头分析师观点</h3>
        {{BULL_ANALYSIS}}
    </div>
    <div class="section">
        <h3 class="bear">空头分析师观点</h3>
        {{BEAR_ANALYSIS}}
    </div>

    <h2>📈 各维度分析</h2>
    <div class="section">
        <h3>技术分析</h3>
        {{TECHNICAL_ANALYSIS}}
    </div>
    <div class="section">
        <h3>基本面分析</h3>
        {{FUNDAMENTAL_ANALYSIS}}
    </div>
    <div class="section">
        <h3>新闻分析</h3>
        {{NEWS_ANALYSIS}}
    </div>
    <div class="section">
        <h3>情绪分析</h3>
        {{SENTIMENT_ANALYSIS}}
    </div>

    <h2>💼 投资建议</h2>
    <div class="section">
        <p><strong>建议：</strong><span class="signal-{{SIGNAL}}">{{SIGNAL_CN}}</span></p>
        <p><strong>目标价：</strong>{{TARGET_PRICE}}</p>
        <p><strong>置信度：</strong>{{CONFIDENCE}}</p>
        <p><strong>风险评分：</strong>{{RISK_SCORE}}</p>
        <p><strong>详细推理：</strong></p>
        {{TRADER_REASONING}}
    </div>

    <h2>⚖️ 风险评估</h2>
    <div class="section">
        <h3>激进观点</h3>
        {{AGGRESSIVE_VIEW}}
    </div>
    <div class="section">
        <h3>保守观点</h3>
        {{CONSERVATIVE_VIEW}}
    </div>
    <div class="section">
        <h3>中性观点</h3>
        {{NEUTRAL_VIEW}}
    </div>
    <div class="section">
        <p><strong>最终风险决策：</strong>{{RISK_DECISION}}</p>
    </div>

    <h2>📝 总结</h2>
    <div class="section">
        {{SUMMARY}}
    </div>

    <div class="footer">
        <p>本报告由 AI 自动生成，仅供参考，不构成投资建议。</p>
        <p>生成时间：{{GENERATED_AT}}</p>
    </div>
</body>
</html>
```

- [ ] **Step 3: 提交**

```bash
git add ~/.claude/skills/stock-analysis-report/style.css ~/.claude/skills/stock-analysis-report/report_template.html
git commit -m "feat: add HTML report template and styles"
```

---

## Task 3: 创建 PDF 生成脚本

**Files:**
- Create: `~/.claude/skills/stock-analysis-report/generate_pdf.sh`

- [ ] **Step 1: 创建 generate_pdf.sh**

```bash
#!/bin/bash
# PDF 生成脚本

set -e

REPORT_DIR="/opt/stock-analysis"
INPUT_HTML="$1"
OUTPUT_PDF="$2"

if [ -z "$INPUT_HTML" ] || [ -z "$OUTPUT_PDF" ]; then
    echo "Usage: $0 <input.html> <output.pdf>"
    exit 1
fi

cd "$REPORT_DIR"

# 使用 WeasyPrint 生成 PDF
weasyprint "$INPUT_HTML" "$OUTPUT_PDF" --presentational-hints

if [ $? -eq 0 ]; then
    echo "PDF generated successfully: $OUTPUT_PDF"
    ls -lh "$OUTPUT_PDF"
else
    echo "PDF generation failed"
    exit 1
fi
```

- [ ] **Step 2: 设置执行权限并测试**

```bash
chmod +x ~/.claude/skills/stock-analysis-report/generate_pdf.sh
ssh root@43.134.180.240 "chmod +x /opt/stock-analysis/generate_pdf.sh"
```

- [ ] **Step 3: 创建测试 HTML 并测试 PDF 生成**

```bash
ssh root@43.134.180.240 "cat > /opt/stock-analysis/test.html << 'HTMLEOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>测试报告</title>
</head>
<body>
    <h1>测试报告</h1>
    <p>这是一条测试数据：中文显示</p>
</body>
</html>
HTMLEOF"

ssh root@43.134.180.240 "cd /opt/stock-analysis && bash generate_pdf.sh test.html test.pdf"
```

Expected: PDF 生成成功，文件存在

- [ ] **Step 4: 提交**

```bash
git add ~/.claude/skills/stock-analysis-report/generate_pdf.sh
git commit -m "feat: add PDF generation script"
```

---

## Task 4: 创建 Skill 主文件

**Files:**
- Create: `~/.claude/skills/stock-analysis-report/skill.md`

- [ ] **Step 1: 创建 skill.md 文件头**

```markdown
---
name: stock-analysis-report
description: 基于 TradingAgents-CN 框架，用户粘贴股票分析文本，自动生成 PDF 分析报告
---

# Stock Analysis Report Skill

当用户提供股票分析文本并要求生成报告时，使用此 Skill。

## 工作流程

1. 接收用户粘贴的股票分析文本
2. 解析文本，识别数据维度
3. 使用 13 类 Agent prompt 框架逐个分析
4. 填充 HTML 报告模板
5. 上传服务器生成 PDF
6. 返回下载链接
```

- [ ] **Step 2: 添加 13 类 Agent Prompt 模板**

在 skill.md 中添加完整的 prompt 模板（从之前分析的 TradingAgents-CN 获取）：

```markdown
## Agent Prompt 模板

### 1. 多头分析师 (Bull Researcher)

你是一位看涨分析师，负责为股票 {company_name}（股票代码：{ticker}）的投资建立强有力的论证。

你的任务是构建基于证据的强有力案例，强调增长潜力、竞争优势和积极的市场指标...

### 2. 空头分析师 (Bear Researcher)

你是一位看跌分析师，负责论证不投资股票 {company_name}（股票代码：{ticker}）的理由...

[以此类推，添加所有 13 类 Agent 的 prompt]
```

- [ ] **Step 3: 添加报告生成流程**

```markdown
## 报告生成流程

### Step 1: 解析用户输入
解析用户粘贴的文本，提取：
- 股票代码/名称
- 基本面数据
- 技术面数据
- 新闻资讯
- 情绪数据

### Step 2: 执行多 Agent 分析
按顺序调用各 Agent prompt：
1. 多头分析师
2. 空头分析师
3. 市场分析师
4. 基本面分析师
5. 新闻分析师
6. 社交媒体分析师
7. 研究经理
8. 交易员
9. 激进风险辩手
10. 保守风险辩手
11. 中性风险辩手
12. 风险经理
13. 反思系统

### Step 3: 填充 HTML 模板
将分析结果填入 report_template.html

### Step 4: 生成 PDF
```bash
scp report.html root@43.134.180.240:/opt/stock-analysis/
ssh root@43.134.180.240 "cd /opt/stock-analysis && bash generate_pdf.sh report.html {{COMPANY_NAME}}_{{DATE}}.pdf"
```

### Step 5: 返回结果
提供 PDF 下载链接
```

- [ ] **Step 4: 提交**

```bash
git add ~/.claude/skills/stock-analysis-report/skill.md
git commit -m "feat: add stock-analysis-report skill main file"
```

---

## Task 5: 服务器部署

**Files:**
- Modify: 服务器 `/opt/stock-analysis/`

- [ ] **Step 1: 上传模板和样式到服务器**

```bash
scp ~/.claude/skills/stock-analysis-report/report_template.html root@43.134.180.240:/opt/stock-analysis/
scp ~/.claude/skills/stock-analysis-report/style.css root@43.134.180.240:/opt/stock-analysis/
```

- [ ] **Step 2: 验证服务器文件**

```bash
ssh root@43.134.180.240 "ls -la /opt/stock-analysis/"
```

Expected: 包含 report_template.html, style.css, generate_pdf.sh, *.otf 字体文件

- [ ] **Step 3: 提交部署变更**

```bash
git add -A
git commit -m "deploy: upload templates and styles to server"
```

---

## Task 6: 端到端测试

- [ ] **Step 1: 使用示例数据测试完整流程**

用户提供一段股票分析文本，验证：
1. Skill 正确解析文本
2. 各 Agent 分析正常执行
3. HTML 生成正确
4. PDF 生成成功
5. 下载链接可用

---

## 依赖检查清单

- [ ] WeasyPrint 已安装
- [ ] 中文字体已配置
- [ ] 模板文件已上传
- [ ] 脚本有执行权限
- [ ] SSH 连接正常

---

**Plan created:** 2026-04-07
**Spec:** docs/superpowers/specs/2026-04-07-stock-analysis-report-design.md
