---
title: '从 Vibe Coding 到 SDD'
date: 2026-03-10T13:10:00+08:00
weight: 10
draft: false
tags: ['ai-programming', 'sdd', 'vibe-coding', 'codebuddy']
categories: ['tech']
description: '从 Vibe Coding 到 SDD（规范驱动开发），以 OpenSpec 框架和 CodeBuddy 的 Craft Spec、Skill/Rule 体系为例，看 AI 编程如何从"靠运气"走向"工业级可靠"。'
featured_image: 'https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/f6957061444057aefcf7e8bf708061af.png'
---

在 AI 辅助开发的浪潮中，"Vibe Coding" 虽然听起来很酷，但本质是依靠直觉和 AI 的模糊理解——你在和 AI "对暗号"，能不能跑通全靠运气。

为了让这种开发模式从「玄学」走向「工业级可靠」，**SDD（规范驱动开发）** 应运而生，而 **OpenSpec** 正是落地这一理念的开源框架。

如果你熟悉 **TDD（测试驱动开发）**，会发现 SDD 其实是 TDD 思想在 AI 时代的延伸：**先定义"什么是正确的"，再让 AI 去实现**。

<!--more-->

## AI Coding 的现状

当前主流 AI 编程工具的工作流，本质都是"需求 → 代码 → 试错"的循环：

{{< mermaid >}}
graph LR
    A[自然语言需求] --> B[AI生成代码]
    B --> C{运行测试}
    C -- 通过 --> D[交付]
    C -- 失败 --> E[调试]
    E --> B
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#bbf,stroke:#333,stroke-width:2px
{{< /mermaid >}}

这套流程的问题在于：**试错的次数决定了开发效率**。当需求复杂时，AI 生成的代码往往需要多次调试才能跑通，甚至会逐渐失控。

## 三种开发模式对比

| 模式 | 描述 | 核心问题 |
|:---|:---|:---|
| **Vibe Coding** | 自然语言 + AI 生成 + 试错修正 | 不可控，复杂项目易崩盘 |
| **TDD** | 先写测试，再写代码让测试通过 | 粒度只在函数级别 |
| **SDD** | 先定规范（Specification），再基于规范生成代码 | 覆盖系统级设计 |

**本质变化**：TDD 用测试用例约束代码，SDD 用规范文档约束整个系统的设计和实现。

## SDD 的三大支柱

在 OpenSpec / CodeBuddy 框架中，AI 的行为由三个核心概念定义：

| 概念 | 回答 | 类比 |
|:---|:---|:---|
| **Command** | 我要做什么 | 发动机的燃料 |
| **Rule** | 我必须怎么做 | 方向盘，约束边界 |
| **Skill** | 我能够怎么做 | 工具箱，扩展能力 |

### Rule：始终加载的静态约束

Rule 在**每次对话开始时**自动加载，适合高频、短小的约束：

- 项目命令（`npm run build`、`bun test`）
- 代码风格（ES 模块 vs CommonJS、命名规范）
- 项目结构约定（API 放在哪个目录）

### Skill：按需加载的动态能力

Skill **只在需要时**才加载，适合复杂、多步骤的操作：

- `/deploy`：执行部署工作流
- `/code-review`：运行审查流程
- 特定领域知识（数据库迁移、监控配置）

两者最核心的区别是**上下文窗口的利用效率**：Rule 始终占用上下文，Skill 按需加载。超过 50 行的详细指南应该做成 Skill，而不是堆在 Rule 里。

## SDD 工作流

在 OpenSpec 中，规范驱动开发遵循以下闭环：

{{< mermaid >}}
graph TD
    A[Proposal 提议] --> B[Spec 规范]
    B --> C[Tasks 任务拆解]
    C --> D[Design 设计]
    D --> E[Apply 实现]
    E --> F[Archive 归档]
    F -->|反馈| B
    style A fill:#f96,stroke:#333,stroke-width:2px
    style E fill:#6f9,stroke:#333,stroke-width:2px
{{< /mermaid >}}

- **Rule** 在规范阶段确保设计符合项目约束
- **Command** 将规范拆解为具体执行步骤
- **Skill** 提供执行所需的专业能力

## 工具实践：CodeBuddy Craft Spec

腾讯云 CodeBuddy IDE 的 **Craft Spec** 功能是 SDD 的产品化实现。开发者输入自然语言需求后，AI 会先输出结构化 PRD 和设计方案供确认，确认后才进入编码阶段：

| OpenSpec 阶段 | CodeBuddy 功能 |
|:---|:---|
| Proposal | 自然语言需求输入 |
| Spec | 自动生成 PRD 文档 |
| Design | 设计原型生成 |
| Tasks | 任务拆解 |
| Apply | Craft 模式逐步生成代码 |

关键差异在于**中间多了一步对齐**：

- Vibe Coding：`需求 → 代码`（两步，中间无确认）
- Craft Spec：`需求 → PRD/设计方案确认 → 代码`（三步，AI 先呈现理解，你确认后再动手）

这看似多了一步，实则大幅减少了返工。

## 进阶：MCP + SDD

将内部工具封装为 MCP（Model Context Protocol）后，AI 不仅能"看到"工具，还能真正执行操作：

**场景**：MySQL 数据库 + OpenSpec

1. 规范阶段：AI 调用 MySQL MCP 直接查询真实 schema
2. 设计阶段：AI 基于实际表结构设计新功能
3. 实现阶段：生成的代码自动匹配现有表风格

```text
用户：新增一个用户收藏功能
AI（OpenSpec）：先查库 → 生成 Spec（包含真实的 user_id 字段类型）
AI：根据 Spec 生成代码
```

**企业优先封装的 MCP 类型**：

| 类型 | MCP |
|:---|:---|
| 基础设施 | 数据库查询、日志查询、监控指标 |
| 研发流程 | 代码审查、CI/CD 触发、版本管理 |
| 业务能力 | 内部 API、消息推送、数据同步 |

## SDD 与传统开发方法对比

| 维度 | 传统瀑布 | 敏捷开发 | TDD | SDD |
|:---|:---|:---|:---|:---|
| **需求定义** | 详细文档 | User Story | 测试用例 | 规范文档 |
| **质量保障** | 后期测试 | 持续集成 | 单元测试 | 规范验证 |
| **适用场景** | 稳定需求 | 变化需求 | 单元级别 | 系统级 + AI |

**关键洞察**：SDD 不是要替代 TDD，而是弥补 TDD 在 AI 时代的断层——TDD 解决"如何确保代码正确"，SDD 解决"如何确保 AI 生成的系统设计正确"。

## 总结

| 层级 | 角色 | 回答 |
|:---|:---|:---|
| **Vibe Coding** | 出发点 | 简单快捷但不可控 |
| **OpenSpec/SDD** | 框架层 | 确保不偏航 |
| **Command** | 执行层 | 要做什么 |
| **Rule** | 约束层 | 要怎么做 |
| **Skill** | 能力层 | 会做什么 |

**实践建议**：从建立项目的 **Rule** 开始——这能立竿见影地减少 AI 乱写代码的情况。进一步可以在 [CodeBuddy IDE](https://www.codebuddy.cn/ide/) 中体验 Craft Spec 完整流程。

*参考资料：*
- *Fission-AI/OpenSpec GitHub 项目*
- *CodeBuddy IDE*
