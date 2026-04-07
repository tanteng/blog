---
title: "Superpowers + OpenSpec：AI 编程黄金搭档工作流"
date: 2026-03-15T10:00:00+08:00
draft: false
tags: ['ai', 'claude-code', 'agent', 'workflow']
categories: ['tech']
description: "深入解析 Superpowers 技能框架与 OpenSpec 规范驱动开发的互补关系，探讨如何构建高效可靠的 AI 编程闭环。"
---

AI 编程工具层出不穷，但真正的问题不是"AI 能不能写代码"，而是"如何让 AI 按照软件工程的标准流程开发"。2026 年，两个开源项目——Superpowers 和 OpenSpec——形成了一套被业内称为"黄金搭档"的开发范式。

<!--more-->

## 1. 背景：AI 编程的核心困境

很多团队引入 AI 编程后反而更累了，根本原因在于：

- **需求模糊**：用自然语言描述需求，AI 理解与预期不符
- **执行失控**：AI 直接跳到写代码，忽略测试和审查
- **变更不可追溯**：代码改了很多版，但没人记得为什么这样改
- **质量不稳定**：AI 生成的代码缺乏系统性验证

这时候需要的不只是"更强的 AI"，而是**规范约束 + 执行流程**。

## 2. Superpowers：让 AI 学会"怎么做"

### 2.1 是什么

Superpowers 是一款工作流插件，为 Claude Code、Cursor、Codex 等编程 Agent 提供一套标准化的软件开发流程。它的核心理念是：

> **不是让 AI 多会写代码，而是尽量让它少在错误的时机写代码。**

安装方式（Claude Code）：

```
/plugin install superpowers@claude-plugins-official
```

### 2.2 核心技能（Skills）

Superpowers 包含多个技能，分为几个大类：

| 技能 | 作用 |
|------|------|
| `brainstorming` | 苏格拉底式提问，深挖需求本质 |
| `writing-plans` | 创建详细实施计划，分解为 2-5 分钟的子任务 |
| `subagent-driven-development` | 子 Agent 分工开发 + 双重审查 |
| `test-driven-development` | RED-GREEN-REFACTOR 强制循环 |
| `systematic-debugging` | 四阶段根本原因分析 |
| `requesting-code-review` | 标准化代码审查流程 |
| `verification-before-completion` | 完成前强制验证 |
| `finishing-a-development-branch` | 分支收尾与归档 |

### 2.3 典型工作流

```
brainstorming → planning → TDD → code review
```

当你让 AI 开发功能时，它不会立刻敲代码，而是：

1. **追问需求**，提炼清晰的开发规格
2. **设计方案**，拆成小块确认
3. **制定计划**，每个任务 2-5 分钟，标注文件路径和验证步骤
4. **子 Agent 执行**，一个开发一个审查，互相校验
5. **全程 TDD**，测试先行
6. **自动验证**，清理开发环境

## 3. OpenSpec：让 AI 知道"做什么"

### 3.1 是什么

OpenSpec 是一款**规范驱动开发**（Spec-Driven Development，SDD）框架，通过"规范 → 执行 → 验证"的闭环，确保需求清晰、变更可追溯。

核心理念：

> 在写第一行代码之前，先把"要做什么"和"为什么做"用结构化的方式写清楚。

安装方式：

```bash
npm install -g @fission-ai/openspec@latest
```

### 3.2 目录结构

```
openspec/
├── specs/           # 当前系统规范（唯一事实源）
├── changes/         # 新功能变更提案
│   └── 新功能名称/
│       ├── proposal.md   # 变更提案
│       ├── specs/        # 需求与功能规范
│       ├── design.md     # 技术设计方案
│       └── tasks.md      # 可执行任务清单
└── config.yaml     # 项目配置
```

### 3.3 核心工作流

| 命令 | 作用 |
|------|------|
| `/openspec:proposal` | 发起新变更提案 |
| `/openspec:apply` | 执行已批准的变更 |
| `/openspec:archive` | 归档已完成的变更 |

## 4. 黄金搭档：互补而非冲突

### 4.1 定位互补

| 工具 | 核心定位 | 解决的问题 | 核心能力 |
|------|----------|------------|----------|
| **OpenSpec** | 规范驱动开发框架 | 需求模糊、变更不可追溯 | 提案管理、规范沉淀、变更追踪 |
| **Superpowers** | 技能流程框架 | 执行失控、质量低下 | 头脑风暴、TDD、代码审查、自动化验证 |

### 4.2 工作流闭环

```
OpenSpec 定方向：
  ├── propose      → 发起提案
  ├── refine       → 细化需求
  └── validate     → 验证规范

Superpowers 保执行：
  ├── brainstorm   → 追问需求
  ├── plan         → 制定计划
  ├── tdd          → 测试驱动
  └── review       → 代码审查

OpenSpec 管结果：
  └── archive      → 归档变更，更新规范
```

### 4.3 一句话总结

- **OpenSpec** 负责"做什么"——确保需求清晰、变更可追溯
- **Superpowers** 负责"怎么做"——保证执行规范、质量可靠

## 5. 实际应用场景

### 5.1 简单功能迭代

```
用户：我想要添加一个用户登录功能

OpenSpec：创建变更提案，写清需求和规范
Superpowers：brainstorming 追问细节 → planning 拆解任务 → tdd 实现
OpenSpec：归档，更新规范
```

### 5.2 复杂跨服务开发

```
OpenSpec：proposal → specs（需求文档）→ design（技术方案）→ tasks（任务清单）
Superpowers：按任务逐个执行，每个任务强制 TDD + review
OpenSpec：验证所有任务完成，archive 归档
```

### 5.3 Bug 修复

```
Superpowers：systematic-debugging 四阶段分析根因
Superpowers：tdd 先写测试验证 bug 存在
Superpowers：修复代码，验证测试通过
OpenSpec：归档变更，记录根因和解决方案
```

## 6. 支持的平台

两者都支持主流 AI 编程工具：

| 工具 | Superpowers | OpenSpec |
|------|-------------|----------|
| Claude Code | 内置插件市场 | 生成 `.claude/` 配置 |
| Cursor | 插件市场安装 | 生成 `.cursor/` 配置 |
| Codex/OpenCode | Fetch 安装 | 原生支持 |
| Gemini CLI | 扩展安装 | 集成支持 |

## 7. 相关工具对比

AI 编程规范驱动开发领域还有两个玩家：

| 工具 | 特点 |
|------|------|
| **Spec-Kit** | GitHub 官方出品，规范可执行化，82.5K Stars |
| **Kiro** | 强调编辑器深度集成 |

三者与 Superpowers 的关系：都是 AI 编程工作流规范的一部分，各有侧重。

## 8. 总结

2026 年的 AI 编程，不再是"拼模型参数"，而是"拼工程化能力"。

Superpowers + OpenSpec 的组合，本质上是：

> **OpenSpec 定义规范（Spec），Superpowers 执行规范（Do），两者共同形成"规划-执行-验证-归档"的完整闭环。**

如果你正在使用 Claude Code 或其他 AI 编程工具，建议同时安装这两个工具——它们会彻底改变你与 AI 的协作方式。

**参考链接：**

- Superpowers: [github.com/obra/superpowers](https://github.com/obra/superpowers)
- OpenSpec: [openspec.dev](https://openspec.dev)
