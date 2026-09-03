# 后端文章系列（2015–2022）实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为博客撰写 18 篇后端技术文章，时间跨度 2015–2022，由浅入深，覆盖网络、数据库、缓存、并发、微服务、容器、可观测性、分布式系统八大主题。

**Architecture:** 一次性创建 worktree，按"年度+难度"分四个 Phase，每 Phase 由独立 subagent 并行撰写，每篇遵循现有博客的 frontmatter / 内容风格（深度原理 + 类比 + 表格 + Mermaid）。最后用 `hugo --minify` 验证构建。

**Tech Stack:**
- Hugo（静态站点生成器）
- Ananke 主题
- Mermaid（图表）
- 中文为主要语言

**Spec:** 用户原始需求：补充后端内容、覆盖 golang/k8s/redis/mysql 但不局限于此、2015–2022 由浅入深、用英文查证资料、避免过时知识。

---

## 全局约束（来自 CLAUDE.md）

1. **文件命名**：slug 不含日期，如 `tcp-three-way-handshake.md`，不是 `2015-03-tcp-three-way-handshake.md`
2. **front matter url**：含日期路径，如 `url: /2015/03/tcp-three-way-handshake/`
3. **front matter date**：ISO 8601 格式带时区，如 `date: 2015-03-15T10:00:00+08:00`
4. **标签**：使用已有英文 slug（参考 `content/tags/`），4–5 个；新标签需在 `content/tags/<slug>/_index.md` 建中文展示名
5. **分类**：技术文章一律 `categories: ["tech"]`
6. **Mermaid**：使用 ```` ```mermaid ```` 代码块，**不要使用** `{{< mermaid >}}` shortcode
7. **不写 featured_image**：保持主题默认
8. **Git 提交**：每个 Phase 一个 commit，message 用 `feat: 新增 Phase X 后端文章 N 篇`
9. **构建验证**：每个 Phase 完成后必须 `hugo --minify` 通过
10. **内容语言**：中文为主，技术名词保留英文
11. **知识时效性**：研究 2026 当前最佳实践撰写，但**避免假装讨论尚未发布的功能**（例如 2020 年文章不要提到 Go 1.22 的 range over func）；可在文章末尾加"更新记录"段落补充后话

---

## 文章系列（18 篇）

### Phase 1：基础篇（2015–2016，难度 ★☆☆）

| # | 日期 | Slug | 标题 | 标签 |
|---|------|------|------|------|
| 1 | 2015-03-15 | `tcp-three-way-handshake` | TCP 三次握手与四次挥手：为什么不是两次握手 | `tcp`, `networking`, `linux` |
| 2 | 2015-09-20 | `http-protocol-evolution` | HTTP 协议演进：从 1.0 到 2.0 的工程意义 | `protocol`, `http`, `performance-optimization` |
| 3 | 2016-04-10 | `mysql-innodb-buffer-pool` | MySQL InnoDB 存储引擎：Buffer Pool 与磁盘 I/O | `mysql`, `innodb`, `database` |
| 4 | 2016-08-25 | `linux-performance-observability` | Linux 性能观测：CPU、内存、I/O 基础 | `linux`, `performance-optimization` |
| 5 | 2016-12-05 | `process-thread-coroutine` | 进程、线程与协程：操作系统视角的本质区别 | `linux`, `concurrency`, `coroutine` |

### Phase 2：进阶篇（2017–2018，难度 ★★☆）

| # | 日期 | Slug | 标题 | 标签 |
|---|------|------|------|------|
| 6 | 2017-03-12 | `redis-persistence-rdb-aof` | Redis 持久化：RDB 与 AOF 的工程权衡 | `redis`, `database`, `performance-optimization` |
| 7 | 2017-08-30 | `consistent-hashing` | 一致性哈希算法：动态扩容的优雅解法 | `algorithm`, `distributed`, `architecture` |
| 8 | 2018-02-18 | `go-from-php-mindset` | Go 语言入门：从 PHP 到 Go 的思维转变 | `go`, `golang` |
| 9 | 2018-06-22 | `go-concurrency-goroutine-channel` | Go 并发编程：goroutine 与 channel 的工程实践 | `go`, `golang`, `concurrency`, `goroutine` |
| 10 | 2018-11-15 | `kafka-vs-rabbitmq-design` | 消息队列：Kafka 与 RabbitMQ 的设计哲学对比 | `kafka`, `rabbitmq`, `queue`, `architecture` |

### Phase 3：微服务篇（2019–2020，难度 ★★★）

| # | 日期 | Slug | 标题 | 标签 |
|---|------|------|------|------|
| 11 | 2019-04-08 | `microservices-evolution-path` | 微服务架构演进：从单体到分布式的路径 | `microservices`, `architecture`, `ddd` |
| 12 | 2019-09-25 | `grpc-internal-communication` | gRPC 实战：内部通信的协议之争 | `grpc`, `protocol`, `microservices` |
| 13 | 2020-05-10 | `docker-containerization-fundamentals` | Docker 容器化：从镜像到网络的完整指南 | `docker`, `containerization`, `linux` |
| 14 | 2020-10-20 | `kubernetes-control-data-plane` | Kubernetes 架构：控制平面与数据平面的协作 | `kubernetes`, `architecture`, `distributed` |

### Phase 4：分布式与可观测性篇（2021–2022，难度 ★★★★）

| # | 日期 | Slug | 标题 | 标签 |
|---|------|------|------|------|
| 15 | 2021-03-15 | `observability-three-pillars` | 可观测性三大支柱：Metrics、Logs、Traces 的工程落地 | `observability`, `architecture`, `performance-optimization` |
| 16 | 2021-08-08 | `prometheus-production-monitoring` | Prometheus 监控体系：从零搭建生产级 | `observability`, `prometheus`, `monitoring` |
| 17 | 2022-02-22 | `distributed-lock-comparison` | 分布式锁：Redis / etcd / ZooKeeper 三种实现的全对比 | `distributed`, `redis`, `etcd`, `zookeeper`, `concurrency` |
| 18 | 2022-09-18 | `distributed-transaction-tradeoffs` | 分布式事务：2PC、SAGA、TCC 的工程取舍 | `distributed`, `transaction`, `architecture`, `microservices` |

---

## 需要新建的标签

写入 `content/tags/<slug>/_index.md`，title 用中文：

- `tcp` (http)
- `networking` → "网络"
- `linux` → "Linux"
- `protocol` → "网络协议"
- `http` → "HTTP"
- `database` → "数据库"
- `innodb` → "InnoDB"
- `coroutine` → "协程"
- `algorithm` → "算法"
- `distributed` → "分布式系统"
- `rabbitmq` → "RabbitMQ"
- `queue` → "消息队列"
- `docker` → "Docker"
- `containerization` → "容器化"
- `observability` → "可观测性"
- `monitoring` → "监控"
- `prometheus` → "Prometheus"
- `etcd` → "etcd"
- `zookeeper` → "ZooKeeper"
- `transaction` → "事务"
- `grpc` (需要)
- `kafka` (需要)
- `redis` (存在)
- `mysql` (存在)
- `go` / `golang` (存在)
- `kubernetes` (存在)
- `concurrency` (存在)
- `microservices` (存在)
- `performance-optimization` (存在)
- `architecture` (存在)
- `ddd` (存在)
- `goroutine` (存在)

执行时先用 `ls content/tags/` 确认哪些已存在，避免重复创建。

---

## 文件结构

每个 Phase 创建一个目录组织文章：

```
content/posts/
├── (18 个 .md 文件，slug 见上表)
content/tags/
├── (新标签的 _index.md 文件)
```

**不修改任何其他文件**（不碰主题配置、不动 partials、不改 CLAUDE.md）。

---

## 执行策略

### 准备阶段（执行者负责）

- [ ] **Step 0：创建 worktree**

```bash
cd /Users/tanteng/Websites/blog
git worktree add .worktrees/backend-article-series -b feature/backend-article-series
cd .worktrees/backend-article-series
```

- [ ] **Step 1：核对已有标签**

```bash
ls content/tags/ | sort
```

对所有未存在的标签，在 `content/tags/<slug>/_index.md` 写入：

```markdown
---
title: "<中文名>"
---
```

完成后 commit：

```bash
git add content/tags/
git commit -m "chore: 新增后端系列所需标签"
```

### 写作阶段（按 Phase 并行 subagent）

每个 Phase 由一个独立 subagent 撰写该 Phase 的所有文章。Subagent 在 `general-purpose` 模式下工作，每个 subagent 的 prompt 模板如下：

```
你是一名资深后端工程师，正在为 Hugo 博客（Ananke 主题）撰写后端技术文章。
工作目录：/Users/tanteng/Websites/blog/.worktrees/backend-article-series/

【任务】撰写 Phase X 的 N 篇文章：
1. <slug> - <title> - 日期 YYYY-MM-DD - 标签 [...]
2. <slug> - <title> - 日期 YYYY-MM-DD - 标签 [...]
...

【风格要求】
- 中文为主，技术名词保留英文
- 长度 2000–3500 字
- 结构：引子（场景/痛点）→ 原理（含 Mermaid 图）→ 代码示例（Go/Python）→ 实战建议 → 小结
- 必须包含至少 1 个 Mermaid 图（流程图/时序图/对比表）
- 必须包含至少 1 个代码块（语法高亮）
- 类比开头（如"医院/快递/红绿灯"等贴近生活的比喻）
- 表格对比（如适用）

【front matter 模板】
---
title: "<标题>"
date: YYYY-MM-DDTHH:MM:SS+08:00
draft: false
url: /YYYY/MM/<slug>/
tags: ['tag1', 'tag2', 'tag3', 'tag4']
categories: ['tech']
description: "<一句话描述，140 字以内>"
---

【绝对不能做的事】
- 不要使用 `{{< mermaid >}}` shortcode
- 不要给文章添加 featured_image 字段
- 不要在文件名里加日期
- 不要写 2026 才发布的功能（如 2020 年文章不要提 Go 1.22 range over func）

【可以/应该做的事】
- 可以引用 2026 当前最佳实践，但不要假装当时有
- 必要时可在文章末尾加"更新记录"段落补充后续发展
- 用英文 WebSearch / WebFetch 查证技术细节（如 RFC 文档、官方文档）

【写作流程】
1. WebSearch 关键事实（如 "TCP three-way handshake reason not two"）
2. WebFetch 官方文档核实细节
3. 起草文章
4. 写入 content/posts/<slug>.md
5. 验证 hugo 解析正确：`hugo --minify --quiet` 不报错

【输出】
完成后报告：
- 已写的文件列表
- 每篇字数
- 引用的资料链接
- 任何需要确认的事实争议
```

### 验证阶段

- [ ] **Step N：全量构建验证**

```bash
cd /Users/tanteng/Websites/blog/.worktrees/backend-article-series
hugo --minify 2>&1 | tail -20
```

预期：无 ERROR，所有 18 篇文章出现在 `public/YYYY/MM/<slug>/index.html`

- [ ] **Step N+1：本地预览**

```bash
hugo server -D
```

报告端口与所有 18 篇文章的预览 URL。

- [ ] **Step N+2：抽查内容**

抽样读 3 篇文章（Phase 1、Phase 3、Phase 4 各一篇），确认：
- 风格一致（深度原理 + 类比 + 表格 + Mermaid + 代码）
- 标签正确（每个标签都在 `content/tags/` 存在）
- 无过时知识

- [ ] **Step N+3：分阶段 commit**

每个 Phase 完成后单独 commit：

```bash
git add content/posts/<phase-slugs>
git commit -m "feat: 新增 Phase X 后端文章 N 篇"
```

最后总 commit（如需要）：

```bash
git commit -m "feat: 完成 2015–2022 后端文章系列"
```

---

## 任务分解

### Task 0：准备工作
- 创建 worktree
- 新建所有标签目录
- commit
- **可独立 review**

### Task 1：Phase 1 基础篇（5 篇，2015–2016）
- 5 个 subagent 并行（或 1 个串行）
- 涵盖：TCP、HTTP、MySQL InnoDB、Linux 性能、进程/线程/协程
- commit
- **可独立 review**

### Task 2：Phase 2 进阶篇（5 篇，2017–2018）
- 涵盖：Redis 持久化、一致性哈希、Go 入门、Go 并发、消息队列
- commit
- **可独立 review**

### Task 3：Phase 3 微服务篇（4 篇，2019–2020）
- 涵盖：微服务演进、gRPC、Docker、K8s
- commit
- **可独立 review**

### Task 4：Phase 4 分布式篇（4 篇，2021–2022）
- 涵盖：可观测性、Prometheus、分布式锁、分布式事务
- commit
- **可独立 review**

### Task 5：最终验证
- 全量 hugo 构建
- 本地预览
- 抽查内容
- **可独立 review**

---

## 自检清单（执行者完成每 Task 后核对）

- [ ] 没有 `{{< mermaid >}}` shortcode
- [ ] 没有 `featured_image` 字段
- [ ] 所有标签在 `content/tags/` 都有对应目录
- [ ] 所有 url 字段符合 `/YYYY/MM/<slug>/` 格式
- [ ] 所有 date 字段符合 ISO 8601
- [ ] `hugo --minify` 无报错
- [ ] 中文流畅，无翻译腔
- [ ] 每篇包含至少 1 个 Mermaid 图 + 1 个代码块
- [ ] 引用的技术事实有官方/RFC 来源

---

## 风险与回退

- **风险**：subagent 写入过期或错误信息
  - **回退**：抽样人工 review，发现问题立即让 subagent 重写
- **风险**：hugo 构建因 Mermaid 或 front matter 不通过
  - **回退**：每个 Phase 后立即构建，失败立即修
- **风险**：worktree 与主目录冲突
  - **回退**：始终在 `.worktrees/backend-article-series/` 内操作
