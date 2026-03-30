---
title: "腾讯云 RAG 实践：原子引擎与 OpenViking 协同"
date: 2026-03-02
description: "当企业知识库遇上 AI Agent 记忆系统，腾讯云原子引擎管知识，OpenViking 管记忆。本文侧重原子引擎的实操经验，协同方案为探索性设想。"
categories: ['tech']
tags: ['ai', 'rag', 'openviking', 'agent', 'tencent-cloud']
featured_image: ""
---

当企业知识库遇上 AI Agent 记忆系统，一个管"知识"，一个管"记忆"，这可能是当前最务实的 AI 应用架构。

<!--more-->

## 背景

在构建企业级 AI 应用时，我们通常面对两类截然不同的数据管理需求：

- **静态知识**：产品文档、技术手册、FAQ、合同模板……这些内容相对固定，需要高质量的解析和精准检索。
- **动态上下文**：用户偏好、对话历史、Agent 积累的经验……这些数据随交互不断变化，需要长期记忆和智能管理。

用一套系统同时解决这两个问题，往往顾此失彼。更好的做法是：**让专业的工具做专业的事**。

本文介绍的方案是：**腾讯云知识引擎原子能力**负责企业知识库的 RAG 全链路，**字节跳动开源的 OpenViking** 负责 AI Agent 的上下文和记忆管理，两者协同工作。

## 一、腾讯云原子引擎：企业知识库的 RAG 全流程

### 1.1 什么是原子引擎

腾讯云知识引擎原子能力（LLM Knowledge Engine Basic API）是腾讯云提供的一套**解耦的 RAG 能力组件**。它不是一个黑盒产品，而是把 RAG 链路中的每个环节都暴露为独立的 API，开发者可以按需组装。

核心原子能力包括：

| 能力 | 说明 | API |
|------|------|-----|
| 文档解析（同步） | 多格式文件转 Markdown，支持表格、公式、图片 | `ReconstructDocumentSSE` |
| 文档解析（异步） | 同上，适合大文件，无耗时限制 | `CreateReconstructDocumentFlow` |
| 文档拆分 | 多级语义拆分，比传统正则切分回答完整性提升 20% | `CreateSplitDocumentFlow` |
| Embedding | 文本转向量，用于语义检索 | `GetEmbedding` |
| 多轮改写 | 对话中的指代消解和省略补全 | `QueryRewrite` |
| 重排序 | 对检索结果按相关性重新排序 | `RunRerank` |

此外，还提供了 **RAG 综合能力套件**，包含知识库创建、文档上传、状态查询、知识检索等一站式接口。

### 1.2 RAG 全流程详解

一个完整的 RAG 流程分为两个阶段：**离线索引**和**在线检索**。

#### 阶段一：离线索引（构建知识库）

{{< mermaid >}}
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#e3f2fd', 'primaryTextColor': '#1565c0', 'primaryBorderColor': '#1565c0', 'lineColor': '#90a4ae', 'secondaryColor': '#f5f5f5', 'tertiaryColor': '#fff9c4'}}}%%
flowchart TD
    A["📄 文档<br/>PDF/Word/网页等"] --> B["Step 1: 文档解析<br/>CreateReconstructDocumentFlow"]
    B --> C["Step 2: 语义拆分<br/>CreateSplitDocumentFlow"]
    C --> D["Step 3: 向量化<br/>GetEmbedding"]
    D --> E["Step 4: 入库存储<br/>向量数据库"]
{{< /mermaid >}}

#### 阶段二：在线检索（回答问题）

{{< mermaid >}}
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#e8f5e9', 'primaryTextColor': '#2e7d32', 'primaryBorderColor': '#2e7d32', 'lineColor': '#90a4ae', 'secondaryColor': '#f5f5f5', 'tertiaryColor': '#fff9c4'}}}%%
flowchart TD
    A["❓ 用户提问"] --> B["Step 5: 多轮改写<br/>QueryRewrite"]
    B --> C["Step 6: 查询向量化<br/>GetEmbedding"]
    C --> D["Step 7: 相似度检索<br/>SearchKnowledge"]
    D --> E["Step 8: 重排序<br/>RunRerank"]
    E --> F["Step 9: 生成回答<br/>LLM"]
{{< /mermaid >}}

#### 如果使用 RAG 综合套件

上述流程也可以简化为 4 步：

```python
# 1. 创建知识库
CreateKnowledgeBase(Name="产品文档库")

# 2. 上传文档（系统自动完成解析→拆分→向量化→入库）
UploadDoc(KnowledgeBaseId="kb-xxx", FilePath="产品手册.pdf")

# 3. 查询文档处理状态
DescribeDoc(DocId="doc-xxx")  # 等待状态变为 "processed"

# 4. 检索知识
SearchKnowledge(KnowledgeBaseId="kb-xxx", Query="如何配置连接池")
```

综合套件把中间步骤全部封装了，适合快速上手；原子能力 API 适合需要精细控制每个环节的场景。

### 1.3 原子引擎的优势

- **文档解析质量高**：支持表格、公式、图片等复杂元素，不是简单的文本提取
- **语义拆分比正则切分强**：官方数据显示回答完整性提升 20%
- **内置重排序**：不需要自己搭 Reranker，直接调 API
- **多轮改写**：处理多轮对话的指代消解，不需要额外写 Prompt
- **灵活组装**：每个环节都是独立 API，可以只用你需要的部分

## 二、OpenViking：AI Agent 的上下文操作系统

### 2.1 什么是 OpenViking

OpenViking 是字节跳动火山引擎 Viking 团队开源的 **AI Agent 上下文数据库**。它不是一个传统的向量数据库，而是用**虚拟文件系统范式**来统一管理 Agent 所需的记忆（Memory）、资源（Resource）和技能（Skill）。

简单说，它是给 AI Agent 用的"文件系统 + 长期记忆 + 智能检索"一体化方案。

### 2.2 核心能力

| 能力 | 说明 |
|------|------|
| **虚拟文件系统（AGFS）** | 用 `viking://` 协议统一管理所有上下文数据 |
| **分层加载（L0/L1/L2）** | L0 摘要（~100 tokens）→ L1 概览（~2K tokens）→ L2 完整内容，按需加载 |
| **目录递归检索** | 先通过目录结构缩小范围，再做语义匹配，比扁平 RAG 更精准 |
| **记忆自进化** | 自动从对话中提取记忆，合并冗余信息，生成经验标签 |
| **可视化检索轨迹** | 检索过程完全透明，方便调试和优化 |

### 2.3 OpenViking 解决什么问题

传统 RAG 的痛点是：所有数据都扔进一个扁平的向量库，检索时只靠语义相似度匹配。当数据量大、类型杂时，召回率和精准度都会下降。

OpenViking 的做法不同——它给数据加上了**目录结构**，检索时先定位到相关目录，再在目录内做语义搜索。这就像你在电脑上找文件，不会全盘搜索，而是先进入相关的文件夹。

加上分层加载机制（先看摘要，需要时再加载全文），Token 消耗可以降低 60%-90%。

## 三、腾讯云原子引擎 + OpenViking：如何分工协同

> ⚠️ **说明**：本节探讨的是一种**概念架构设想**，腾讯云原子引擎已在实际项目中使用，但与 OpenViking 的协同方案尚未落地验证，仅作为探索性讨论。

### 3.1 定位差异

| 维度 | 腾讯云原子引擎 | OpenViking |
|------|--------------|------------|
| **核心定位** | 企业知识库的 RAG 工具链 | AI Agent 的上下文管理系统 |
| **数据类型** | 静态文档（PDF、Word、网页等） | 动态数据（记忆、会话、技能） |
| **数据特点** | 更新频率低，内容固定 | 持续演化，随交互积累 |
| **检索方式** | 扁平向量检索 + 重排序 | 目录结构 + 语义检索，分层加载 |
| **部署方式** | 云端 SaaS，开箱即用 | 本地/自建服务，数据自主可控 |
| **擅长场景** | 文档问答、知识检索 | 长期记忆、多轮对话、经验沉淀 |

### 3.2 协同架构

{{< mermaid >}}
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f3e5f5', 'primaryTextColor': '#7b1fa2', 'primaryBorderColor': '#7b1fa2', 'lineColor': '#90a4ae', 'secondaryColor': '#e1bee7', 'tertiaryColor': '#fff9c4'}}}%%
flowchart TD
    Q["❓ 用户提问"] --> OV1
    OV1["① OpenViking 理解上下文<br/>召回记忆"] --> OV2
    OV2["② OpenViking 输出改写后查询"] --> TE3
    TE3["③ 腾讯云原子引擎<br/>检索企业文档"] --> TE4
    TE4["④ 腾讯云原子引擎<br/>重排序精排"] --> LLM
    LLM["⑤ LLM 生成最终回答"] --> OV3
    OV3["⑥ OpenViking 记忆沉淀"]
    Q -->|"并行触发"| TE3
    OV2 -->|"提供上下文"| LLM
{{< /mermaid >}}
**协同流程：**

1. **OpenViking 负责理解上下文**：用户说"上次讨论的那个方案"，OpenViking 从长期记忆中召回 "PostgreSQL 方案"，并把模糊的指代转化为明确的查询。
2. **腾讯云原子引擎负责知识检索**：拿到明确的查询后，在企业文档库中做精准的 RAG 检索，返回最相关的文档片段。
3. **LLM 综合生成**：将记忆上下文和文档知识一起输入 LLM，生成完整的回答。
4. **OpenViking 沉淀新记忆**：这次交互的关键信息（用户关注什么、讨论了什么结论）被自动提取并存入长期记忆。

### 3.3 各自负责什么数据

```
腾讯云原子引擎（知识库）          OpenViking（上下文库）
├── 产品文档/                    ├── viking://user/memories/
│   ├── 用户手册.pdf             │   ├── 偏好: 喜欢用 PostgreSQL
│   ├── API 文档.md              │   ├── 习惯: 偏好命令行操作
│   └── 部署指南.docx            │   └── 历史: 3月1日讨论了数据库选型
├── 技术文档/                    ├── viking://agent/skills/
│   ├── 架构设计.pdf             │   ├── 代码审查技能
│   └── 最佳实践.md              │   └── 部署脚本生成
├── FAQ/                        ├── viking://session/
│   ├── 常见问题.xlsx            │   ├── 当前会话上下文
│   └── 故障排除.md              │   └── 历史会话摘要
└── 合规文档/                    └── viking://resources/
    └── 安全规范.pdf                 └── 用户上传的临时参考资料
```

### 3.4 什么时候调用谁

| 场景 | 调用谁 | 原因 |
|------|--------|------|
| "帮我查一下部署文档里关于 K8s 的配置" | 腾讯云原子引擎 | 明确的文档检索需求 |
| "我之前说过喜欢什么数据库来着？" | OpenViking | 用户长期记忆 |
| "继续上次的讨论" | OpenViking → 原子引擎 | 先召回记忆确定上下文，再检索知识 |
| "总结一下这个 PDF 的核心观点" | 腾讯云原子引擎 | 文档解析和摘要 |
| "记住：以后数据库统一用 PostgreSQL" | OpenViking | 用户偏好沉淀 |
| "根据我们的安全规范，这段代码有什么问题？" | 原子引擎 + OpenViking | 检索安全规范 + 结合历史代码审查经验 |

## 四、实际落地建议

### 4.1 先跑通单侧，再做集成

不要一上来就搭完整架构。建议：

1. **第一步**：先用腾讯云原子引擎把企业知识库的 RAG 跑通，验证检索质量
2. **第二步**：独立部署 OpenViking，给 Agent 加上长期记忆
3. **第三步**：在 Agent 的调度层做集成，按场景路由到不同的系统

### 4.2 模型选择

两个系统可以用不同的模型：

- **腾讯云原子引擎**：直接用内置的 Embedding 和 Rerank 模型，开箱即用
- **OpenViking**：可以配置火山引擎豆包、OpenAI、智谱等任意 Embedding 和 VLM 模型

### 4.3 成本考量

- 腾讯云原子引擎按 API 调用量计费，适合文档量大但查询频次可控的场景
- OpenViking 是开源自部署，计算成本取决于你选择的 Embedding/VLM 模型

## 总结

腾讯云原子引擎和 OpenViking 不是竞品，而是互补关系：

- **原子引擎** = 企业的"图书馆"，管理和检索固定的知识文档
- **OpenViking** = Agent 的"大脑"，管理动态的记忆、上下文和技能

把静态知识和动态记忆分开管理，既能保证知识检索的精准度，又能让 Agent 具备真正的长期记忆能力。这是当前构建企业级 AI 应用的一个务实且高效的架构选择。

---

*参考资料：*
- [腾讯云知识引擎原子能力 - 产品概述](https://cloud.tencent.com/document/product/1772/111122)
- [腾讯云知识引擎原子能力 - RAG 操作指南](https://cloud.tencent.com/document/product/1772/111236)
- [OpenViking GitHub 仓库](https://github.com/volcengine/OpenViking)
