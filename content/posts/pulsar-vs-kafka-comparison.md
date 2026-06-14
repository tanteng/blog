---
title: "Pulsar 与 Kafka：两大消息流平台深度对比"
date: 2026-05-28
draft: false
tags: ["messaging", "kafka", "pulsar", "architecture"]
categories: ["tech"]
description: "全面对比 Apache Pulsar 和 Apache Kafka 的架构设计、消费模型、多租户、运维复杂度等核心差异，帮助技术选型。"
---

# Pulsar 与 Kafka：两大消息流平台深度对比

在分布式消息中间件领域，Apache Kafka 和 Apache Pulsar 是最被广泛讨论的两大流派。Kafka 诞生于 2011 年，已是流处理的事实标准；Pulsar 则是后起之秀，被称作"下一代云原生消息平台"。本文从架构设计到实际运维，对两者进行一次系统性的梳理对比。

<!--more-->

## 一、架构设计：计算与存储是否分离

**Kafka** 采用计算存储耦合的单体架构，每个 Broker 既负责消息的网络路由和处理，也存储分区日志数据。Broker 的扩展和故障恢复都需要数据同步，运维复杂度随集群规模增长而显著上升。

**Pulsar** 则采用了彻底的分层架构：Broker（计算层）负责消息的路由和生产消费交互，而持久化存储完全交给 Apache BookKeeper 处理。Broker 本身是无状态的，故障时可以快速替换，理论上支持百万级 Topic，且存储层和计算层可以独立扩缩容。

```
# Kafka 架构
Broker[计算+存储] — Broker[计算+存储] — Broker[计算+存储]
        ↓              ↓              ↓
      ZooKeeper       ZooKeeper      ZooKeeper

# Pulsar 架构
Broker — Broker — Broker（无状态）
   ↓         ↓         ↓
BookKeeper — BookKeeper — BookKeeper（分布式存储）
```

## 二、消息存储模型

Kafka 以 Topic 为单位，每个 Topic 划分为多个 Partition，每个 Partition 是有序的、不可变的日志文件，追加写入磁盘。数据存储在本地文件系统，依赖 OS Page Cache 实现高性能。

Pulsar 的存储粒度更细：Topic 被划分为 Segment（段），每个 Segment 存储在 BookKeeper 的 Ledger 中。Segment 是可以被均匀分布到不同 BookKeeper 节点的最小单位，这让负载均衡更加精细，也避免了 Kafka 中"热点 Partition"的问题。

## 三、消费模型

**Kafka** 的消费模型比较统一：每个 Partition 在同一个消费组内只会被一个消费者实例独占消费，支持手动 offset 管理和 exactly-once 语义。

**Pulsar** 在消费模型上更加丰富，提供了四种订阅模式：

| 模式 | 描述 |
|------|------|
| Exclusive（独占）| 只能有一个消费者，类似 Kafka |
| Failover（灾备）| 主备切换，主 consumer 故障则交给备用者 |
| Shared（共享）| 消息轮询分发，允许多个消费者并行消费 |
| Key_Shared | 按消息 key 决定分发到哪个消费者 |

Pulsar 的 Shared 模式天然适合 task queue 场景，多个 worker 可以并发消费消息，而 Kafka 实现类似功能通常需要额外设计。

## 四、多租户支持

这是 Pulsar 相对于 Kafka 最为突出的优势之一。

Kafka 的多租户能力很弱，通常依赖外部工具或额外的 ACL 配置实现租户隔离。

Pulsar 从设计之初就考虑了多租户场景，通过 **Tenant（租户）→ Namespace（命名空间）→ Topic** 三级层级实现天然的租户隔离。每个租户可以独立配置消息保留策略、配额、资源限制。不同团队/业务线共用同一个 Pulsar 集群时，彼此完全隔离，无需为每个业务线搭建独立 Kafka 集群。

## 五、运维复杂度

| 维度 | Kafka | Pulsar |
|------|-------|--------|
| 依赖组件 | 早期需要 ZooKeeper，KRpc 模式逐渐脱离 | 同样需要 ZooKeeper，但支持红黄切换减少对ZK的依赖 |
| Broker 扩缩容 | 涉及数据迁移，较复杂 | 有状态扩容，数据自动均衡 |
| Topic 数量 | 万级别以内表现较好 | 支持百万级 Topic |
| 故障恢复 | 需要同步 Replica，恢复时间较长 | 无状态 Broker + BookKeeper Ledger 快速恢复 |

## 六、适用场景小结

**选 Kafka 的场景：**
- 已有的成熟生态和团队经验
- 极致吞吐量追求，Topic 数量可控
- 生态工具链丰富（Kafka Connect、Flink、Spark）

**选 Pulsar 的场景：**
- 需要强多租户隔离的企业场景
- 超大规模 Topic（百万级）
- 需要丰富的订阅模式和灵活的消费逻辑
- 云原生架构，需要计算存储独立扩缩容

---

*如果你觉得这篇文章有帮助，欢迎在评论区交流你的选型经验。*