---
title: '从 Vibe Coding 到规范驱动开发：AI 编程的工业级进化'
date: 2026-03-01T13:10:00+08:00
draft: false
tags: ['AI编程', 'OpenSpec', 'SDD', 'Vibe Coding', '开发方法论']
categories: ['技术思考']
description: 'Vibe Coding 听起来很酷，但本质上依靠直觉和 AI 的模糊理解。OpenSpec 和 SDD（规范驱动开发）让这种开发模式从"靠运气"变成"工业级可靠"。'
---

在 AI 辅助开发的浪潮中，"Vibe Coding"（氛围感编程）虽然听起来很酷，但其本质是依靠直觉和 AI 的模糊理解。为了让这种开发模式从"靠运气"变成"工业级可靠"，OpenSpec 和 SDD（规范驱动开发）应运而生。

<!--more-->

以下是围绕这些概念的核心梳理，以及 Command、Rule、Skill 这三者在实际执行层面的差异：

---

## 1. 核心理念：从 Vibe Coding 到 SDD

**Vibe Coding（氛围感编程）**：开发者通过自然语言（Tweet 式的描述）告诉 AI 一个模糊的想法，AI 生成代码，开发者发现错误后再不断 Debug 修正。缺点是不可控、代码质量不一、复杂项目易崩盘。

**SDD (Spec-Driven Development)**：借鉴了传统软件工程中的规范定义。要求在写代码前，先由 AI 和人共同确认一份"规范"（Specification）。

**OpenSpec**：这是一个具体的开源框架，它将 SDD 的思想落地，通过结构化的 Markdown 文件（如 proposal.md, tasks.md）来管理 AI 的上下文。

---

## 2. 执行三剑客：Command, Rule, Skill 的区别

在 OpenSpec 或类似的 AI Agent 框架中，这三个词定义了 AI 的边界和能力值：

| 概念 | 定义 | 作用 | 类比 |
|:---|:---|:---|:---|
| **Command** (指令) | 我要做什么 | 具体的、单次执行的任务描述。触发 AI 动作。通常是用户输入的 Prompt 或 tasks.md 里的一个条目 | 菜谱里的"炒肉"这一步骤 |
| **Rule** (规则) | 我必须怎么做 | 项目的约束条件、编码风格、安全底线。约束 AI 行为，确保生成代码的一致性 | 厨房里的卫生标准和调味禁忌 |
| **Skill** (技能) | 我能够怎么做 | AI 被赋予的特定工具调用能力或领域知识。扩展 AI 能力 | 厨师掌握的"刀工"或"火候控制"能力 |

---

## 3. OpenSpec、Rule 和 Skill 的关系

在 GitHub 的 Fission-AI/OpenSpec 项目中，OpenSpec、Rule 和 Skill 分别代表了流程框架、行为规范和执行能力。

它们的关系可以简单概括为：
- **OpenSpec** 是这一整套开发流程
- **Skill** 是 AI 执行这个流程的"技能包"
- **Rule** 是 AI 在执行过程中必须遵守的"家规"

### OpenSpec (流程 / 框架)

定义：OpenSpec 是整个**规范驱动开发（SDD）**的方法论和工具本体。它定义了从"提议 (Proposal)"到"规格 (Spec)"、"任务 (Tasks)"、"设计 (Design)"，最后到"实现 (Apply)"和"归档 (Archive)"的一整套标准工作流。

作用：它像是一个项目经理，规定了"我们要先写文档对齐需求，再写代码"这一核心原则，并提供了 CLI 工具来管理这些步骤。

### Skill (技能 / 能力)

定义：Skill 指的是 AI Agent 所具备的特定操作能力。在 OpenSpec 初始化时，它会在 .claude/skills/ 等目录下生成一系列文件。

作用：它像是赋予 AI 的"技能书"。有了 Skill，AI 才能听懂 `/opsx:new`、`/opsx:apply` 这种指令。

Skill 实际上是将 OpenSpec 的 CLI 命令封装成了 AI 可以调用的函数或工作流，让 AI 知道"当用户说创建新变更时，我需要去运行 openspec new 命令并读取输出"。

### Rule (规则 / 规范)

定义：Rule 是注入到 AI 上下文中的约束条件或编码规范。通常定义在 openspec/config.yaml 或专门的规则文件中。

作用：它像是公司的"员工手册"或"代码规范"。当 AI 使用 Skill 去生成某个文档时，Rule 会告诉 AI："必须使用中文"、"所有的常量都要大写"、"必须包含回滚计划"等。

层级：Rule 可以是全局的（对所有步骤生效），也可以是针对特定产物（Artifact）的。

---

## 4. 它们是如何协同工作的？

在 OpenSpec 的工作流中，这几个元素形成了一个闭环：

1. **OpenSpec/SDD (框架层)**：你告诉 AI "我要做一个支付功能"，AI 并不直接写代码，而是先生成一份 SDD 文档。

2. **Rule (约束层)**：AI 在构思文档时，会检查你的规则库，确保设计符合你的架构偏好。

3. **Command (执行层)**：文档确认后，拆解成一个个具体的 Command 分发给 AI Agent。

4. **Skill (支撑层)**：AI 使用它自带的 Skill（如阅读现有代码库、运行测试脚本）来确保 Command 执行成功。

---

## 5. 总结

- **Vibe Coding** 是出发点（简单快捷）
- **OpenSpec/SDD** 是导航仪（确保不偏航）
- **Command/Rule/Skill** 是发动机和方向盘（具体的执行与约束）

如果你正尝试从"随便写写"转向"专业级 AI 开发"，建议先从建立项目的 Rule 开始，这能立竿见影地减少 AI 乱写代码的情况。

---

*参考资料：*
- *Fission-AI/OpenSpec GitHub 项目*
- *GitHub Spec Kit vs OpenSpec 对比*
