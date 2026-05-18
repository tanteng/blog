---
title: "OpenClaw 多 Agent 架构详解"
date: 2026-05-17
draft: false
url: /2026/05/openclaw-multi-agent-architecture/
tags: ["ai", "openclaw", "agent", "architecture"]
categories: ["ai"]
description: "详解 OpenClaw 的多 Agent 架构，包括 Agent、Session、Sub-agent 的概念，以及它们之间的协作模式。"
---

OpenClaw 是一个强大的 AI 助手框架，其核心设计思想之一就是**多 Agent 协作**。很多人对 "Agent"、"Session"、"Sub-agent" 这些概念感到困惑，今天我们就用图解的方式彻底搞清楚。

<!--more-->

**Agent（智能体）** 是一个能够自主思考、决策、执行任务的 AI 程序。它不仅仅是回答问题，而是能够：

- 理解复杂任务
- 分解子任务
- 调用工具（tools）完成任务
- 与其他 Agent 协作

在 OpenClaw 中，每个 Agent 都有自己的：
- **上下文（Context）** - 记忆和工作区
- **工具（Tools）** - 读写文件、搜索、执行命令等
- **技能（Skills）** - 特定领域的专业能力

---

## Agent 的两种形态

在 OpenClaw 中，Agent 存在两种截然不同的形态，各有其用途和生命周期。

### 形态一：独立 Agent（Persistent Agent）

**独立 Agent** 是通过 `openclaw agents add` 命令创建的持久化 Agent，有固定的人设、记忆和工作区。

```bash
openclaw agents add travel_assistant
```

创建后会在 `~/.openclaw/agents/<agentId>/` 下生成完整的目录结构：

```
~/.openclaw/agents/travel_assistant/
├── agent/
│   ├── auth-profiles.json    # 独立凭证配置
│   └── ...
├── sessions/                 # 独立的会话历史
└── workspace/               # 独立的工作区
    ├── AGENTS.md            # Agent 行为规范
    ├── SOUL.md              # Agent 人设定义
    ├── USER.md              # 用户信息
    ├── MEMORY.md            # 长期记忆（累积）
    └── memory/
        └── YYYY-MM-DD.md    # 每日日志
```

**特点**：

| 特性 | 说明 |
|------|------|
| 持久性 | 长期存在，跨 session 保留 |
| 人设固化 | 通过 SOUL.md 定义固定人设 |
| 记忆累积 | MEMORY.md 会随使用不断丰富 |
| 独立凭证 | 可绑定独立的 API 密钥、渠道账号 |
| 固定角色 | 例如：旅行助手、股票分析师、客服机器人 |

**典型用途**：
- 固定角色的专业助手（旅行规划、代码审查）
- 多渠道 Bot（不同 Telegram Bot 对应不同 Agent）
- 需要长期记忆积累的助手

### 形态二：临时子 Agent（Ephemeral Sub-Agent）

**临时子 Agent** 是通过 `sessions_spawn` 动态创建的 Agent，执行完任务后立即消失，不保留任何记忆。

```javascript
sessions_spawn({
  task: "帮我查询深圳天气",
  mode: "run"
})
```

**特点**：

| 特性 | 说明 |
|------|------|
| 生命周期 | 本次任务执行完即结束 |
| 工作区 | 临时创建的独立目录 |
| 记忆 | MEMORY.md 为空，无累积 |
| 人设 | 在 task 参数中临时指定 |
| 隔离性 | 完全独立，不污染父 Agent |

**执行流程**：

{{< mermaid >}}sequenceDiagram
    participant Main as 主 Agent
    participant Sub as 临时子 Agent

    Main->>Sub: sessions_spawn 创建
    Sub->>Sub: 独立执行任务
    Sub-->>Main: 返回结果
    Note over Sub: 任务完成
    Note over Sub: Sub Agent 消失
{{< /mermaid >}}

### 对比总结

| 维度 | 独立 Agent | 临时子 Agent |
|------|------------|---------------|
| **创建方式** | `openclaw agents add` | `sessions_spawn` |
| **生命周期** | 持久常驻 | 临时，执行完消失 |
| **记忆累积** | ✅ MEMORY.md 长期积累 | ❌ 每次都是空的 |
| **人设** | SOUL.md 固化定义 | Prompt 中临时指定 |
| **凭证** | 独立配置 | 继承父 Agent |
| **适用场景** | 固定角色、长期助手 | 并行任务、一次性工作 |
| **工作区** | 固定 workspace | 临时独立 workspace |
| **下次调用** | 同一 Agent，有上下文 | 全新的空状态 |

### 混合使用

两种形态可以组合使用，发挥各自优势：

{{< mermaid >}}graph TB
    subgraph Main["🤖 主 Agent（主 Session）"]
        M[任务分解
        结果汇总]
    end

    subgraph Persistent["👤 独立 Agent"]
        PA[travel_assistant<br/>旅行顾问 - 有记忆累积]
    end

    subgraph Ephemeral["⚡ 临时子 Agent（sessions_spawn）"]
        E1[天气查询]
        E2[活动搜索]
        E3[户外专项]
    end

    M -->|sessions_spawn| E1
    M -->|sessions_spawn| E2
    M -->|sessions_spawn| E3
    M -->|sessions_spawn
        agentId="travel_assistant"| PA
    E1 --> M
    E2 --> M
    E3 --> M
    PA --> M

    style Persistent fill:#e1f5fe
    style Ephemeral fill:#fff3e0
{{< /mermaid >}}

例如在一个旅行规划场景中：
- **临时子 Agent** 并行查询天气、景点、活动（快速、隔离）
- **独立 Agent** `travel_assistant` 基于用户的旅行偏好历史，提供个性化建议（记忆+人设）
- **主 Agent** 汇总所有结果，生成最终行程

> 📌 **关键理解**：无论是独立 Agent 还是临时子 Agent，核心的隔离性是一样的 —— 每个 Agent 都有自己独立的 workspace 和 memory，不会互相污染。区别只在于生命周期的长短和是否有持久记忆。

---

**Session（会话）** 是 OpenClaw 中最基本的运行单元，可以理解为一个"独立的工作空间"。

{{< mermaid >}}graph LR
    A[用户消息] --> B[Session A<br/>主会话]
    B --> C[Task 1]
    B --> D[Task 2]
    B --> E[Task 3]
    
    style B fill:#e1f5fe
    style C fill:#fff3e0
    style D fill:#fff3e0
    style E fill:#fff3e0{{< /mermaid >}}

### Session 的特性

| 特性 | 说明 |
|------|------|
| **隔离性** | 每个 Session 有独立的上下文 |
| **持久性** | 关闭后重新打开还能记住上下文 |
| **可派生** | 可以 spawn 新的子 Session |
| **可通信** | Session 之间可以互相发送消息 |


OpenClaw 的多 Agent 架构采用**主从模式**：

{{< mermaid >}}graph TB
    subgraph Gateway["🌐 Gateway（网关）"]
        G[消息入口<br/>路由决策<br/>Session 管理]
    end
    
    subgraph Main["🤖 Main Agent（主 Agent）"]
        M[任务分解<br/>协调调度<br/>结果汇总]
    end
    
    subgraph SubAgents["👥 Sub Agents（子 Agents）"]
        SA1[Agent A<br/>股票分析]
        SA2[Agent B<br/>新闻搜索]
        SA3[Agent C<br/>代码编写]
    end
    
    G --> M
    M --> SA1
    M --> SA2
    M --> SA3
    SA1 --> M
    SA2 --> M
    SA3 --> M
    
    style Gateway fill:#f3e5f5
    style Main fill:#e3f2fd
    style SubAgents fill:#e8f5e9{{< /mermaid >}}

### 核心组件

#### 1. Gateway（网关）

Gateway 是整个系统的入口，负责：
- 接收用户消息
- 路由到对应的 Session
- 管理 Session 的生命周期

#### 2. Main Agent（主 Agent）

Main Agent 是与用户直接对话的 Agent，它的职责是：
- 理解用户意图
- 将复杂任务分解为子任务
- 调度和协调各 Sub Agent
- 汇总结果，返回给用户

#### 3. Sub Agents（子 Agents）

Sub Agent 是执行具体任务的 Agent，可以是：
- 专业领域 Agent（股票分析、代码编写）
- 工具型 Agent（搜索、文件处理）
- 并行运行的多个同类 Agent


### 模式一：父子协作（Sessions Spawn）

{{< mermaid >}}sequenceDiagram
    participant User as 用户
    participant Main as Main Agent
    participant SA as Sub Agent A
    participant SB as Sub Agent B
    
    User->>Main: 分析腾讯股票并查询新闻
    Main->>Main: 任务分解
    Main-->>SA: 启动股票分析
    Main-->>SB: 启动新闻搜索
    SA-->>Main: 股票分析完成
    SB-->>Main: 新闻搜索完成
    Main->>User: 汇总结果回复{{< /mermaid >}}

**特点**：
- 主 Agent 派生子 Agent
- 子 Agent 独立执行任务
- 结果返回主 Agent 汇总

### 模式二：工具调用（Tools）

{{< mermaid >}}graph LR
    A[Main Agent] --> B[内置工具]
    A --> C[Skill 工具]
    A --> D[MCP 工具]
    
    B -->|read/write/exec| E[文件系统]
    C -->|web_search/tts| F[外部服务]
    D -->|stock_data| G[数据源]
    
    style A fill:#e3f2fd
    style B fill:#fff3e0
    style C fill:#fff3e0
    style D fill:#fff3e0{{< /mermaid >}}

**特点**：
- Agent 内置工具直接调用
- 通过 Skill 扩展专业能力
- 通过 MCP 连接外部系统

### 模式三：跨 Session 通信

{{< mermaid >}}graph LR
    A[Session A<br/>主会话] -->|sessions_send| B[Session B<br/>子会话]
    B -->|返回结果| A
    A -->|sessions_send| C[Session C<br/>另一个任务]
    C -->|返回结果| A{{< /mermaid >}}

**特点**：
- 通过 `sessions_send` 发送消息
- 等待目标 Session 处理完成
- 适合跨会话协作任务


很多人搞不清 Session 和 Subagent 的区别：

| | Session | Subagent |
|---|---|---|
| **创建方式** | `sessions_spawn` | `subagents` |
| **上下文** | 可选继承父 session | 独立隔离 |
| **适用场景** | 复杂多步骤任务 | 并行独立任务 |
| **生命周期** | 持久 | 任务级 |
| **通信方式** | `sessions_send` | 直接结果返回 |

### 选择建议

- **需要完整上下文** → 用 Session + `context:"fork"`
- **并行独立任务** → 用 Subagent
- **任务链** → 用 Session 序列


### 场景一：股票分析报告

{{< mermaid >}}flowchart TB
    A[用户请求：分析腾讯股票] --> B[Main Agent]
    B --> C[spawn 股票分析 Agent]
    B --> D[spawn 新闻搜索 Agent]
    B --> E[spawn 技术分析 Agent]
    
    C --> F[获取行情数据]
    D --> G[搜索最新新闻]
    E --> H[计算技术指标]
    
    F --> I[汇总分析报告]
    G --> I
    H --> I
    I --> J[返回用户]
    
    style A fill:#ffcdd2
    style J fill:#c8e6c9{{< /mermaid >}}

### 场景二：博客文章发布

{{< mermaid >}}flowchart LR
    A[用户：发布博客] --> B[Main Agent]
    B --> C[分析文章内容]
    C --> D[生成文件名]
    D --> E[渲染 Front Matter]
    E --> F[添加 more 标记]
    F --> G[Git 提交推送]
    G --> H[返回部署链接]
    
    style A fill:#ffcdd2
    style H fill:#c8e6c9{{< /mermaid >}}


### 1. 模块化设计
每个 Agent 可以独立开发、测试、部署，降低系统复杂度。

### 2. 并行处理
多个 Sub Agent 可以同时执行任务，大大提高处理效率。

### 3. 可扩展性
新增功能只需添加新的 Agent 或 Skill，无需修改核心架构。

### 4. 容错性
某个子任务失败不会影响整体流程，可以单独重试。


OpenClaw 的多 Agent 架构是一种**分层协作**的设计：

- **Gateway** 是入口，负责路由和会话管理
- **Main Agent** 是指挥家，负责分解任务和汇总结果
- **Sub Agents** 是乐手，负责演奏各自的部分
- **Session** 是工作空间，提供独立的上下文环境

理解这个架构，你就能更好地利用 OpenClaw 构建复杂的 AI 应用。


| 命令 | 说明 |
|------|------|
| `sessions_spawn` | 创建新的子 Session |
| `sessions_send` | 向其他 Session 发送消息 |
| `sessions_history` | 查看 Session 历史 |
| `subagents` | 管理子 Agent 任务 |

