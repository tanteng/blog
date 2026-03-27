---
title: "Harness Engineering 入门：让 AI Coding Agent 稳定工作的工程实践"
date: 2026-03-27
draft: false
tags: ["ai", "engineering", "tool"]
categories: ["tech"]
---

最近在研究 AI Coding Agent 的工程化实践，绕不开两个概念：**Harness Engineering** 和 **OpenSpec**。两者都跟 AI 写代码有关，但解决的问题完全不同。写篇文章梳理一下。

<!--more-->

## 什么是 Harness Engineering

Harness Engineering 这个概念是 2026 年 2 月由 Mitchell Hashimoto（Terraform 作者）正式命名的。他的定义非常精炼：

> 每当 Agent 犯一个错误，就设计一个机制让它永远不再犯同类错误。

核心公式只有一个：

```
Agent = Model + Harness
```

模型提供智能，Harness 让智能有用。OpenAI 3 人团队用 5 个月时间，基于这个理念构建了 100 万行代码的产品，零行人工手写。

## 解决什么问题

LLM 有三个根本缺陷，Harness Engineering 正是为了解决这些问题：

### 1. 无状态问题

LLM 每个新会话都是从零开始，不记得上次做了什么。Harness 通过跨会话进度文件让 Agent 能从上次中断点继续。

### 2. 上下文腐烂（Context Rot）

上下文窗口被工具输出、历史记录填满后，模型的指令跟随能力显著下降。Harness 有压缩机制防止这个问题。

### 3. 不可靠验证

Agent 往往会谎报完成——没测试就宣布成功。Harness 强制 Build-Verify 循环，不通过测试不准宣布完成。

## 三大支柱

Harness Engineering 有三个核心支柱：

### 上下文工程（Context Engineering）

核心命题：从 Agent 的视角，任何它在运行时无法访问的内容，等同于不存在。

AGENTS.md 是核心工具，约 100 行导航地图，指向深度文档。过长的 AGENTS.md 反而会让 Agent 性能下降。

### 架构约束（Architectural Constraints）

与其告诉 Agent "写好代码"，不如机械地强制执行"好代码"的样子。通过自定义 Linter，当 Agent 违反架构边界时，错误信息本身就是修复指南。

### 熵管理（Entropy Management）

AI 代码库会自然累积熵——文档脱节、死代码堆积。GC Agent 周期性运行，扫描文档一致性、约束违规、模式偏离。

## OpenSpec 是什么

OpenSpec 是一个规范驱动的工作流程，解决的是"AI 和人对需求的理解不一致"的问题。

核心流程：

```
Proposal → Planning → Implementation → Archive
```

痛点：AI 容易自己脑补细节，结果做出来的东西不符合预期。OpenSpec 的做法是强制在代码之前先生成 proposal/specs/design/tasks 文档，人审阅同意了才让动手。

## OpenSpec 与 Harness Engineering 的关系

OpenSpec 和 Harness Engineering 解决的是不同层次的问题：**OpenSpec 解决的是"人和 AI 对需求的理解不一致"——AI 容易自己脑补细节，做出来的东西不符合预期，所以 OpenSpec 强制在写代码之前先生成 proposal/specs/design/tasks 文档，人审阅同意了才让动手；Harness Engineering 解决的是"AI 写代码时不稳定、不可靠"——AI 会犯重复错误、谎报完成、架构随时间漂移，所以 Harness 通过约束、验证、反馈回路让 AI 的产出保持稳定。两者并非竞争关系，而是互补的：OpenSpec 覆盖了 Harness Engineering 的第一支柱——上下文工程（通过 AGENTS.md 让 AI 知道项目规范是什么），但架构约束和熵管理这些部分还需要 Linter、测试覆盖率、GC Agent 等独立去补充。

## 最小可行示例

Harness 的核心是验证循环。拿 Python + pytest 举例：

```python
# 检查测试覆盖率
def check_coverage() -> bool:
    result = subprocess.run(
        ["pytest", "--cov=calc", "--cov-report=json", "--quiet"],
        capture_output=True, text=True
    )
    # 读取 coverage 报告，检查 operations.py 是否 100% 覆盖
    # 如果有未覆盖的行，说明 Agent 忘了写测试
    ...
```

每当 Agent 添加新函数，Harness 就会检查：测试写了吗？覆盖率够吗？架构约束遵守了吗？没通过就拦截。

## 总结

- **Harness Engineering**：让 AI 稳定、可靠地实现代码
- **OpenSpec**：让 AI 和人对齐要做什么
- **两者结合**：OpenSpec 管规划，Harness 管执行

现在 AI 写代码的能力已经不是瓶颈，真正的瓶颈是如何建立一套工程体系，让 AI 在复杂长时任务中稳定产出。这个领域还非常新，值得深入研究。

---

*相关论文：arXiv:2603.05344 - Building Effective AI Coding Agents for the Terminal*
