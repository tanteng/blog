---
title: "GFS 论文中英对照全文翻译（The Google File System）"
date: 2014-04-12T10:00:00+08:00
url: /2014/04/gfs-paper-cn-en/
draft: false
tags: ["gfs", "google", "distributed", "paper", "reliability"]
categories: ["tech"]
description: "GFS 论文 The Google File System 的完整中英对照翻译，共 9 节，含单 master 架构与 64 MB chunk、租约与主副本、原子记录追加、快照、延迟垃圾回收，以及松弛一致性模型的定义和对应用提出的约束。"
---

这是 Google File System 论文的完整中英对照翻译。原文 15 页，正文 9 节，参考文献 12 条。

体例：每段先列英文原文（引用块），紧接中文译文。图与表格按原文内容重绘；专业术语保留英文并附中文，原文的引用编号 [n] 对应文末参考文献。

<!--more-->

## 论文信息

| 项目 | 内容 |
|------|------|
| 标题 | The Google File System |
| 作者 | Sanjay Ghemawat, Howard Gobioff, Shun-Tak Leung |
| 机构 | Google |
| 发表 | SOSP '03（第 19 届 ACM 操作系统原理研讨会），2003-10-19 至 10-22，纽约 Bolton Landing |
| 编号 | ACM 1-58113-757-5/03/0010…$5.00 |
| 篇幅 | 15 页，正文 9 节，参考文献 12 条 |
| 核心机制 | 单 master、64 MB chunk、租约与主副本、原子记录追加、快照、延迟垃圾回收、松弛一致性模型 |

---

## Abstract · 摘要

{{% bilingual %}}
We have designed and implemented the Google File System, a scalable distributed file system for large distributed data-intensive applications. It provides fault tolerance while running on inexpensive commodity hardware, and it delivers high aggregate performance to a large number of clients.
<!--col-->
我们设计并实现了 Google File System（GFS），一个面向大规模分布式数据密集型应用的可扩展分布式文件系统。它在廉价通用硬件上运行的同时提供容错能力，并为大量客户端提供很高的聚合性能。
{{% /bilingual %}}

{{% bilingual %}}
While sharing many of the same goals as previous distributed file systems, our design has been driven by observations of our application workloads and technological environment, both current and anticipated, that reflect a marked departure from some earlier file system assumptions. This has led us to reexamine traditional choices and explore radically different design points.
<!--col-->
GFS 与之前的分布式文件系统共享很多相同的目标，但我们的设计是被对应用负载和技术环境的观察所驱动的——这些观察涵盖当下与可预见的情况，并且与早先文件系统的某些假设明显背离。这促使我们重新审视传统选择，并探索截然不同的设计点。
{{% /bilingual %}}

{{% bilingual %}}
The file system has successfully met our storage needs. It is widely deployed within Google as the storage platform for the generation and processing of data used by our service as well as research and development efforts that require large data sets. The largest cluster to date provides hundreds of terabytes of storage across thousands of disks on over a thousand machines, and it is concurrently accessed by hundreds of clients.
<!--col-->
这套文件系统成功地满足了我们的存储需求。它在 Google 内部被广泛部署，作为存储平台，既支撑我们各项服务的数据生成与处理，也支撑那些需要大数据集的研究与开发工作。迄今为止最大的一个集群在超过一千台机器、数千块磁盘上提供数百 TB 的存储，并被数百个客户端持续并发访问。
{{% /bilingual %}}

{{% bilingual %}}
In this paper, we present file system interface extensions designed to support distributed applications, discuss many aspects of our design, and report measurements from both micro-benchmarks and real world use.
<!--col-->
在本文中，我们给出为支持分布式应用而设计的文件系统接口扩展，讨论设计的多个方面，并报告来自微基准测试和真实生产环境的测量结果。
{{% /bilingual %}}

{{% bilingual %}}
**Categories and Subject Descriptors:** D [4]: 3—Distributed file systems

**General Terms:** Design, reliability, performance, measurement

**Keywords:** Fault tolerance, scalability, data storage, clustered storage
<!--col-->
**分类与主题描述：** D [4]: 3——分布式文件系统

**一般术语：** 设计、可靠性、性能、测量

**关键词：** 容错、可扩展性、数据存储、集群存储
{{% /bilingual %}}

## 1 Introduction · 引言

{{% bilingual %}}
We have designed and implemented the Google File System (GFS) to meet the rapidly growing demands of Google's data processing needs. GFS shares many of the same goals as previous distributed file systems such as performance, scalability, reliability, and availability. However, its design has been driven by key observations of our application workloads and technological environment, both current and anticipated, that reflect a marked departure from some earlier file system design assumptions. We have reexamined traditional choices and explored radically different points in the design space.
<!--col-->
我们设计并实现了 Google File System（GFS），以满足 Google 数据处理需求快速增长的要求。GFS 与之前的分布式文件系统共享许多相同目标：性能、可扩展性、可靠性和可用性。但它的设计是由对应用负载和技术环境的关键观察所驱动的——涵盖当下与可预见的情况，并且明显背离了早先文件系统的某些设计假设。我们重新审视了传统选择，并在设计空间中探索了截然不同的点。
{{% /bilingual %}}

{{% bilingual %}}
First, component failures are the norm rather than the exception. The file system consists of hundreds or even thousands of storage machines built from inexpensive commodity parts and is accessed by a comparable number of client machines. The quantity and quality of the components virtually guarantee that some are not functional at any given time and some will not recover from their current failures. We have seen problems caused by application bugs, operating system bugs, human errors, and the failures of disks, memory, connectors, networking, and power supplies. Therefore, constant monitoring, error detection, fault tolerance, and automatic recovery must be integral to the system.
<!--col-->
第一，组件故障是常态而非例外。这套文件系统由数百甚至数千台用廉价通用部件搭建的存储机器组成，并被数量相当的客户端机器访问。部件的数量之巨与质量之参差，几乎注定了在任何时刻都有一些不可用，也总有一些无法从当前的故障中恢复。我们见过由应用 bug、操作系统 bug、人为失误，以及磁盘、内存、连接器、网络和电源故障引发的问题。因此，持续监控、错误检测、容错和自动恢复必须成为系统不可分割的一部分。
{{% /bilingual %}}

{{% bilingual %}}
Second, files are huge by traditional standards. Multi-GB files are common. Each file typically contains many application objects such as web documents. When we are regularly working with fast growing data sets of many TBs comprising billions of objects, it is unwieldy to manage billions of approximately KB-sized files even when the file system could support it. As a result, design assumptions and parameters such as I/O operation and block sizes have to be revisited.
<!--col-->
第二，以传统标准衡量，文件非常巨大。多 GB 的文件是常态，每个文件通常包含许多应用对象，比如网页文档。当我们经常要处理由数十亿对象组成、总量达数 TB 且快速增长的数据集时，即使文件系统有能力支持，管理数十亿个约 KB 量级的文件也是笨重不堪的。因此，I/O 操作大小、块大小这类设计假设和参数都必须重新考虑。
{{% /bilingual %}}

{{% bilingual %}}
Third, most files are mutated by appending new data rather than overwriting existing data. Random writes within a file are practically non-existent. Once written, the files are only read, and often only sequentially. A variety of data share these characteristics. Some may constitute large repositories that data analysis programs scan through. Some may be data streams continuously generated by running applications. Some may be archival data. Some may be intermediate results produced on one machine and processed on another, whether simultaneously or later in time. Given this access pattern on huge files, appending becomes the focus of performance optimization and atomicity guarantees, while caching data blocks in the client loses its appeal.
<!--col-->
第三，多数文件是通过追加新数据来修改的，而不是覆盖已有数据。文件内部的随机写几乎不存在。文件一旦写完，就只被读取，而且往往只被顺序读取。多种数据都具备这些特征：有些是数据分析程序扫描的大型数据仓库；有些是运行中的应用持续产生的数据流；有些是归档数据；有些是在一台机器上产生、在另一台机器上处理的中间结果，可能同时处理，也可能延后处理。考虑到这种针对巨大文件的访问模式，追加成为性能优化和原子性保证的重点，而在客户端缓存数据块则失去了吸引力。
{{% /bilingual %}}

{{% bilingual %}}
Fourth, co-designing the applications and the file system API benefits the overall system by increasing our flexibility.
<!--col-->
第四，把应用与文件系统 API 放在一起协同设计，通过提高我们的灵活性而使整个系统受益。
{{% /bilingual %}}

{{% bilingual %}}
For example, we have relaxed GFS's consistency model to vastly simplify the file system without imposing an onerous burden on the applications. We have also introduced an atomic append operation so that multiple clients can append concurrently to a file without extra synchronization between them. These will be discussed in more details later in the paper.
<!--col-->
例如，我们放宽了 GFS 的一致性模型，从而在不对应用施加沉重负担的前提下大幅简化了文件系统。我们还引入了原子追加操作，使多个客户端可以并发向同一个文件追加，而彼此之间不需要额外的同步。这些内容将在后文详细讨论。
{{% /bilingual %}}

{{% bilingual %}}
Multiple GFS clusters are currently deployed for different purposes. The largest ones have over 1000 storage nodes, over 300 TB of disk storage, and are heavily accessed by hundreds of clients on distinct machines on a continuous basis.
<!--col-->
目前已有多个 GFS 集群按不同用途部署。其中最大的集群拥有超过 1000 个存储节点、超过 300 TB 磁盘存储，并被数百台不同机器上的客户端持续高强度访问。
{{% /bilingual %}}

## 2 Design Overview · 设计概览

### 2.1 Assumptions · 设计假设

{{% bilingual %}}
In designing a file system for our needs, we have been guided by assumptions that offer both challenges and opportunities. We alluded to some key observations earlier and now lay out our assumptions in more details.
<!--col-->
在设计满足我们需求的文件系统时，我们遵循了一些既带来挑战也带来机会的假设。前面已提到若干关键观察，现在更详细地列出我们的假设。
{{% /bilingual %}}

{{% bilingual %}}
• The system is built from many inexpensive commodity components that often fail. It must constantly monitor itself and detect, tolerate, and recover promptly from component failures on a routine basis.
<!--col-->
• 系统由大量廉价的通用部件构成，它们经常发生故障。系统必须持续自我监控，并在日常运行中及时发现、容忍组件故障并迅速从中恢复。
{{% /bilingual %}}

{{% bilingual %}}
• The system stores a modest number of large files. We expect a few million files, each typically 100 MB or larger in size. Multi-GB files are the common case and should be managed efficiently. Small files must be supported, but we need not optimize for them.
<!--col-->
• 系统存储数量适中但体积巨大的文件。我们预期有几百万个文件，每个通常 100 MB 或更大。多 GB 的文件是常见情况，必须被高效管理。小文件也必须支持，但不需要为它们做优化。
{{% /bilingual %}}

{{% bilingual %}}
• The workloads primarily consist of two kinds of reads: large streaming reads and small random reads. In large streaming reads, individual operations typically read hundreds of KBs, more commonly 1 MB or more. Successive operations from the same client often read through a contiguous region of a file. A small random read typically reads a few KBs at some arbitrary offset. Performance-conscious applications often batch and sort their small reads to advance steadily through the file rather than go back and forth.
<!--col-->
• 负载主要由两类读组成：大块流式读和小块随机读。在大块流式读中，单次操作通常读取数百 KB，更常见的是 1 MB 或更多；来自同一客户端的连续操作往往顺序读过一个连续的文件区域。小块随机读则通常在某个任意偏移处读取几 KB。关注性能的应用往往会把小块读批量收集并排序，以便稳步向前推进而不是来回跳转。
{{% /bilingual %}}

{{% bilingual %}}
• The workloads also have many large, sequential writes that append data to files. Typical operation sizes are similar to those for reads. Once written, files are seldom modified again. Small writes at arbitrary positions in a file are supported but do not have to be efficient.
<!--col-->
• 负载中也有大量向文件追加数据的大块顺序写操作，其典型操作大小与读相近。文件一旦写完，很少再被修改。对文件任意位置的小块写是支持的，但不必高效。
{{% /bilingual %}}

{{% bilingual %}}
• The system must efficiently implement well-defined semantics for multiple clients that concurrently append to the same file. Our files are often used as producer-consumer queues or for many-way merging. Hundreds of producers, running one per machine, will concurrently append to a file. Atomicity with minimal synchronization overhead is essential. The file may be read later, or a consumer may be reading through the file simultaneously.
<!--col-->
• 系统必须为多个客户端并发追加同一个文件这一场景，高效地实现语义明确的保证。我们的文件常被用作生产者—消费者队列，或用于多路归并。数百个生产者（每台机器跑一个）会并发追加同一个文件。以最小的同步开销实现原子性是必需的。文件可能稍后被读取，也可能同时有消费者正在读取。
{{% /bilingual %}}

{{% bilingual %}}
• High sustained bandwidth is more important than low latency. Most of our target applications place a premium on processing data in bulk at a high rate, while few have stringent response time requirements for an individual read or write.
<!--col-->
• 高持续带宽比低延迟更重要。我们的大多数目标应用看重的是以高速率批量处理数据，很少有应用对单次读或写的响应时间有严苛要求。
{{% /bilingual %}}

### 2.2 Interface · 接口

{{% bilingual %}}
GFS provides a familiar file system interface, though it does not implement a standard API such as POSIX. Files are organized hierarchically in directories and identified by pathnames. We support the usual operations to create, delete, open, close, read, and write files.
<!--col-->
GFS 提供了熟悉的文件系统接口，但没有实现 POSIX 这类标准 API。文件按目录分层组织，用路径名标识。我们支持创建、删除、打开、关闭、读、写这些常规操作。
{{% /bilingual %}}

{{% bilingual %}}
Moreover, GFS has snapshot and record append operations. Snapshot creates a copy of a file or a directory tree at low cost. Record append allows multiple clients to append data to the same file concurrently while guaranteeing the atomicity of each individual client's append. It is useful for implementing multi-way merge results and producer-consumer queues that many clients can simultaneously append to without additional locking. We have found these types of files to be invaluable in building large distributed applications. Snapshot and record append are discussed further in Sections 3.4 and 3.3 respectively.
<!--col-->
此外，GFS 还有快照（snapshot）和记录追加（record append）两个操作。快照以低成本创建文件或目录树的副本。记录追加允许多个客户端并发向同一个文件追加数据，同时保证每个客户端自身那一次追加的原子性。它很适合实现多路归并的结果，以及许多客户端可以同时追加、无需额外加锁的生产者—消费者队列。我们发现这类文件在构建大型分布式应用时非常宝贵。快照和记录追加分别在第 3.4 节和第 3.3 节进一步讨论。
{{% /bilingual %}}

### 2.3 Architecture · 架构

{{% bilingual %}}
A GFS cluster consists of a single master and multiple chunkservers and is accessed by multiple clients, as shown in Figure 1. Each of these is typically a commodity Linux machine running a user-level server process. It is easy to run both a chunkserver and a client on the same machine, as long as machine resources permit and the lower reliability caused by running possibly flaky application code is acceptable.
<!--col-->
一个 GFS 集群由单个 master 和多个 chunkserver 组成，并被多个客户端访问，如图 1 所示。它们中的每一个通常都是一台普通 Linux 机器，运行一个用户态服务进程。只要机器资源允许、并且可以接受运行可能不稳定的应用代码所带来的可靠性下降，在同一台机器上同时运行 chunkserver 和客户端是很容易的。
{{% /bilingual %}}

**图 1 重绘（GFS 架构）**

```
┌──────────────────────────────────────────────────────────┐
│                     Application                          │
│                                                          │
│   (file name, chunk index)      (chunk handle,           │
│             │                    chunk locations)        │
│             ▼                            │               │
│      ┌─────────────┐                     │               │
│      │ GFS client  │◀────────────────────┘               │
│      └─────────────┘                                     │
└───────────┬──────────────────────────────┬───────────────┘
            │ ①                            │ ③④
            ▼                              ▼
   ┌────────────────────┐        ┌──────────────────────┐
   │    GFS master      │        │   GFS chunkserver    │
   │  File namespace    │        │   Linux file system  │
   │      /foo/bar      │◀──────▶│   (chunk 2ef0, …)    │
   │  chunk 2ef0        │  ②     │  chunk data on disk  │
   └────────────────────┘        └──────────────────────┘

图例：──▶ 控制消息      ══▶ 数据消息
① 客户端向 master 请求 chunk 位置   ② master 下发指令、上报状态
③ 客户端向 chunkserver 请求数据     ④ chunkserver 返回 chunk 数据
```

{{% bilingual %}}
Files are divided into fixed-size chunks. Each chunk is identified by an immutable and globally unique 64 bit chunk handle assigned by the master at the time of chunk creation. Chunkservers store chunks on local disks as Linux files and read or write chunk data specified by a chunk handle and byte range. For reliability, each chunk is replicated on multiple chunkservers. By default, we store three replicas, though users can designate different replication levels for different regions of the file namespace.
<!--col-->
文件被划分为固定大小的 chunk。每个 chunk 由一个不可变、全局唯一的 64 位 chunk handle 标识，该 handle 在 chunk 创建时由 master 分配。Chunkserver 把 chunk 以 Linux 文件的形式存放在本地磁盘上，并按 "chunk handle + 字节区间" 的指定来读写 chunk 数据。为了可靠性，每个 chunk 会在多个 chunkserver 上复制。默认存储三份副本，不过用户可以为文件命名空间的不同区域指定不同的副本级别。
{{% /bilingual %}}

{{% bilingual %}}
The master maintains all file system metadata. This includes the namespace, access control information, the mapping from files to chunks, and the current locations of chunks. It also controls system-wide activities such as chunk lease management, garbage collection of orphaned chunks, and chunk migration between chunkservers. The master periodically communicates with each chunkserver in HeartBeat messages to give it instructions and collect its state.
<!--col-->
Master 维护全部文件系统元数据，包括命名空间、访问控制信息、文件到 chunk 的映射，以及 chunk 的当前位置。它还掌控若干系统级活动，例如 chunk 租约管理、孤儿 chunk 的垃圾回收，以及 chunkserver 之间的 chunk 迁移。Master 通过 HeartBeat 消息周期性地与每个 chunkserver 通信，向它下达指令并收集它的状态。
{{% /bilingual %}}

{{% bilingual %}}
GFS client code linked into each application implements the file system API and communicates with the master and chunkservers to read or write data on behalf of the application. Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunkservers. We do not provide the POSIX API and therefore need not hook into the Linux vnode layer.
<!--col-->
链接进每个应用的 GFS 客户端代码实现文件系统 API，并代表应用与 master 和 chunkserver 通信以读写数据。客户端与 master 的交互只用于元数据操作，而所有承载数据的通信都直接发往 chunkserver。我们不提供 POSIX API，因此也不需要挂接到 Linux 的 vnode 层。
{{% /bilingual %}}

{{% bilingual %}}
Neither the client nor the chunkserver caches file data. Client caches offer little benefit because most applications stream through huge files or have working sets too large to be cached. Not having them simplifies the client and the overall system by eliminating cache coherence issues. (Clients do cache metadata, however.) Chunkservers need not cache file data because chunks are stored as local files and so Linux's buffer cache already keeps frequently accessed data in memory.
<!--col-->
客户端和 chunkserver 都不缓存文件数据。客户端缓存收益甚微，因为大多数应用是流式读取巨大文件，或者工作集大得无法缓存。不做客户端缓存还消除了缓存一致性问题，从而简化了客户端和整个系统。（不过客户端确实会缓存元数据。）Chunkserver 也无需缓存文件数据，因为 chunk 本身就是以本地文件形式存放的，Linux 的 buffer cache 已经把频繁访问的数据留在内存里了。
{{% /bilingual %}}

### 2.4 Single Master · 单一 master

{{% bilingual %}}
Having a single master vastly simplifies our design and enables the master to make sophisticated chunk placement and replication decisions using global knowledge. However, we must minimize its involvement in reads and writes so that it does not become a bottleneck. Clients never read and write file data through the master. Instead, a client asks the master which chunkservers it should contact. It caches this information for a limited time and interacts with the chunkservers directly for many subsequent operations.
<!--col-->
采用单一 master 极大地简化了我们的设计，并使 master 能够借助全局信息做出精细的 chunk 放置与副本决策。但我们必须尽量减少它参与读写，以免它成为瓶颈。客户端从不通过 master 读写文件数据，而是向 master 询问应该联系哪些 chunkserver。客户端会把这一信息缓存一段时间，在后续的许多操作中直接与 chunkserver 交互。
{{% /bilingual %}}

{{% bilingual %}}
Let us explain the interactions for a simple read with reference to Figure 1. First, using the fixed chunk size, the client translates the file name and byte offset specified by the application into a chunk index within the file. Then, it sends the master a request containing the file name and chunk index. The master replies with the corresponding chunk handle and locations of the replicas. The client caches this information using the file name and chunk index as the key.
<!--col-->
我们结合图 1 说明一次简单读取的交互过程。首先，客户端利用固定的 chunk 大小，把应用指定的文件名和字节偏移换算成该文件内的 chunk 索引。然后向 master 发送一个包含文件名和 chunk 索引的请求。Master 回复对应的 chunk handle 以及各副本的位置。客户端以"文件名 + chunk 索引"为键缓存这一信息。
{{% /bilingual %}}

{{% bilingual %}}
The client then sends a request to one of the replicas, most likely the closest one. The request specifies the chunk handle and a byte range within that chunk. Further reads of the same chunk require no more client-master interaction until the cached information expires or the file is reopened. In fact, the client typically asks for multiple chunks in the same request and the master can also include the information for chunks immediately following those requested. This extra information sidesteps several future client-master interactions at practically no extra cost.
<!--col-->
随后客户端向其中一个副本（很可能是最近的那个）发送请求，请求中指定 chunk handle 以及该 chunk 内的字节区间。对同一 chunk 的后续读取不再需要客户端与 master 交互，直到缓存信息过期或文件被重新打开。事实上，客户端通常会在一次请求中索要多个 chunk，而 master 也可以把紧随其后那些 chunk 的信息一并返回。这些额外信息以几乎为零的额外代价，省去了未来若干次客户端与 master 的交互。
{{% /bilingual %}}

### 2.5 Chunk Size · chunk 大小

{{% bilingual %}}
Chunk size is one of the key design parameters. We have chosen 64 MB, which is much larger than typical file system block sizes. Each chunk replica is stored as a plain Linux file on a chunkserver and is extended only as needed. Lazy space allocation avoids wasting space due to internal fragmentation, perhaps the greatest objection against such a large chunk size.
<!--col-->
Chunk 大小是最关键的设计参数之一。我们选择了 64 MB，远大于典型的文件系统块大小。每个 chunk 副本在 chunkserver 上就是一个普通的 Linux 文件，按需扩展。惰性空间分配（lazy space allocation）避免了因内部碎片而浪费空间——这也许是对如此大的 chunk 大小最主要的反对理由。
{{% /bilingual %}}

{{% bilingual %}}
A large chunk size offers several important advantages. First, it reduces clients' need to interact with the master because reads and writes on the same chunk require only one initial request to the master for chunk location information. The reduction is especially significant for our workloads because applications mostly read and write large files sequentially. Even for small random reads, the client can comfortably cache all the chunk location information for a multi-TB working set. Second, since on a large chunk, a client is more likely to perform many operations on a given chunk, it can reduce network overhead by keeping a persistent TCP connection to the chunkserver over an extended period of time. Third, it reduces the size of the metadata stored on the master. This allows us to keep the metadata in memory, which in turn brings other advantages that we will discuss in Section 2.6.1.
<!--col-->
大 chunk 带来几个重要优势。第一，它减少了客户端与 master 交互的需要，因为对同一个 chunk 的读写只需要最开始一次向 master 请求 chunk 位置信息。这一削减对我们的负载尤其显著，因为应用大多是顺序读写大文件。即使是小块的随机读，客户端也能轻松缓存下整个多 TB 工作集的 chunk 位置信息。第二，由于在大 chunk 上客户端更可能在同一个 chunk 上执行大量操作，它可以通过与 chunkserver 长时间保持一条持久 TCP 连接来降低网络开销。第三，它缩小了 master 上存储的元数据规模，这使我们能把元数据全部放在内存里，进而带来其他优势，我们将在 2.6.1 节讨论。
{{% /bilingual %}}

{{% bilingual %}}
On the other hand, a large chunk size, even with lazy space allocation, has its disadvantages. A small file consists of a small number of chunks, perhaps just one. The chunkservers storing those chunks may become hot spots if many clients are accessing the same file. In practice, hot spots have not been a major issue because our applications mostly read large multi-chunk files sequentially.
<!--col-->
另一方面，即使有惰性空间分配，大 chunk 也有缺点。一个小文件只由少数几个 chunk 组成，也许只有一个。如果许多客户端同时访问同一个文件，存放这些 chunk 的 chunkserver 就可能成为热点。实践中热点并未成为主要问题，因为我们的应用大多是顺序读取跨越多个 chunk 的大文件。
{{% /bilingual %}}

{{% bilingual %}}
However, hot spots did develop when GFS was first used by a batch-queue system: an executable was written to GFS as a single-chunk file and then started on hundreds of machines at the same time. The few chunkservers storing this executable were overloaded by hundreds of simultaneous requests. We fixed this problem by storing such executables with a higher replication factor and by making the batch-queue system stagger application start times. A potential long-term solution is to allow clients to read data from other clients in such situations.
<!--col-->
不过，当 GFS 最初被一个批处理队列系统使用时，热点确实出现了：一个可执行文件被作为单 chunk 文件写入 GFS，然后在数百台机器上同时启动。存放这个可执行文件的那少数几个 chunkserver 被数百个并发请求压垮。我们通过提高这类可执行文件的副本因子，并让批处理队列系统错开应用启动时间来修复了这个问题。一个潜在的长期方案是，在这种情形下允许客户端从其他客户端读取数据。
{{% /bilingual %}}

### 2.6 Metadata · 元数据

{{% bilingual %}}
The master stores three major types of metadata: the file and chunk namespaces, the mapping from files to chunks, and the locations of each chunk's replicas. All metadata is kept in the master's memory. The first two types (namespaces and file-to-chunk mapping) are also kept persistent by logging mutations to an operation log stored on the master's local disk and replicated on remote machines. Using a log allows us to update the master state simply, reliably, and without risking inconsistencies in the event of a master crash. The master does not store chunk location information persistently. Instead, it asks each chunkserver about its chunks at master startup and whenever a chunkserver joins the cluster.
<!--col-->
Master 存储三类主要元数据：文件与 chunk 的命名空间、文件到 chunk 的映射，以及每个 chunk 各副本的位置。所有元数据都保存在 master 的内存中。前两类（命名空间和文件到 chunk 的映射）还通过把变更记录到操作日志（operation log）来持久化——该日志存放在 master 的本地磁盘上，并复制到远程机器。使用日志使我们能够简单、可靠地更新 master 状态，并且在 master 崩溃时不会引入不一致。Master 不持久化 chunk 位置信息，而是在启动时、以及每当有 chunkserver 加入集群时，向各 chunkserver 询问其持有的 chunk。
{{% /bilingual %}}

#### 2.6.1 In-Memory Data Structures · 内存中的数据结构

{{% bilingual %}}
Since metadata is stored in memory, master operations are fast. Furthermore, it is easy and efficient for the master to periodically scan through its entire state in the background. This periodic scanning is used to implement chunk garbage collection, re-replication in the presence of chunkserver failures, and chunk migration to balance load and disk space usage across chunkservers. Sections 4.3 and 4.4 will discuss these activities further.
<!--col-->
由于元数据存放在内存中，master 的操作很快。而且，master 在后台周期性扫描自身全部状态既容易又高效。这种周期性扫描被用来实现 chunk 垃圾回收、chunkserver 故障时的重新复制，以及为均衡各 chunkserver 之间的负载与磁盘占用而做的 chunk 迁移。4.3 节和 4.4 节将进一步讨论这些活动。
{{% /bilingual %}}

{{% bilingual %}}
One potential concern for this memory-only approach is that the number of chunks and hence the capacity of the whole system is limited by how much memory the master has. This is not a serious limitation in practice. The master maintains less than 64 bytes of metadata for each 64 MB chunk. Most chunks are full because most files contain many chunks, only the last of which may be partially filled. Similarly, the file namespace data typically requires less then 64 bytes per file because it stores file names compactly using prefix compression.
<!--col-->
这种"只放内存"的方案有一个潜在顾虑：chunk 的数量、乃至整个系统的容量，受制于 master 拥有多少内存。实践中这并非严重限制。master 为每个 64 MB 的 chunk 维护的元数据不到 64 字节。多数 chunk 都是满的，因为多数文件包含许多 chunk，只有最后一个可能未写满。类似地，文件命名空间数据通常每个文件所需的字节数也不到 64 字节，因为它用前缀压缩来紧凑地存储文件名。
{{% /bilingual %}}

{{% bilingual %}}
If necessary to support even larger file systems, the cost of adding extra memory to the master is a small price to pay for the simplicity, reliability, performance, and flexibility we gain by storing the metadata in memory.
<!--col-->
如果要支持更大的文件系统，为 master 增加额外内存的代价，相对于我们把元数据放在内存中所获得的简洁、可靠、性能和灵活性而言，是微不足道的。
{{% /bilingual %}}

#### 2.6.2 Chunk Locations · chunk 位置

{{% bilingual %}}
The master does not keep a persistent record of which chunkservers have a replica of a given chunk. It simply polls chunkservers for that information at startup. The master can keep itself up-to-date thereafter because it controls all chunk placement and monitors chunkserver status with regular HeartBeat messages.
<!--col-->
Master 不保存"某个 chunk 的副本在哪些 chunkserver 上"的持久记录。它只是在启动时向 chunkserver 轮询这一信息。此后 master 能让自己保持最新，因为它掌控所有 chunk 放置，并通过定期的 HeartBeat 消息监控 chunkserver 状态。
{{% /bilingual %}}

{{% bilingual %}}
We initially attempted to keep chunk location information persistently at the master, but we decided that it was much simpler to request the data from chunkservers at startup, and periodically thereafter. This eliminated the problem of keeping the master and chunkservers in sync as chunkservers join and leave the cluster, change names, fail, restart, and so on. In a cluster with hundreds of servers, these events happen all too often.
<!--col-->
我们最初尝试在 master 上持久保存 chunk 位置信息，但后来认定：在启动时以及此后定期向 chunkserver 索取这些数据要简单得多。这消除了一个难题——当 chunkserver 加入或离开集群、改名、故障、重启等时，如何保持 master 与 chunkserver 之间的同步。在一个有数百台服务器的集群里，这些事件发生得实在太频繁了。
{{% /bilingual %}}

{{% bilingual %}}
Another way to understand this design decision is to realize that a chunkserver has the final word over what chunks it does or does not have on its own disks. There is no point in trying to maintain a consistent view of this information on the master because errors on a chunkserver may cause chunks to vanish spontaneously (e.g., a disk may go bad and be disabled) or an operator may rename a chunkserver.
<!--col-->
理解这一设计决策的另一种方式，是意识到 chunkserver 对自己磁盘上有什么、没有什么 chunk 拥有最终发言权。在 master 上维持这份信息的一致视图没有意义，因为 chunkserver 上的错误可能导致 chunk 自行消失（例如一块磁盘损坏并被停用），或者运维人员可能重命名某个 chunkserver。
{{% /bilingual %}}

#### 2.6.3 Operation Log · 操作日志

{{% bilingual %}}
The operation log contains a historical record of critical metadata changes. It is central to GFS. Not only is it the only persistent record of metadata, but it also serves as a logical time line that defines the order of concurrent operations. Files and chunks, as well as their versions (see Section 4.5), are all uniquely and eternally identified by the logical times at which they were created.
<!--col-->
操作日志包含关键元数据变更的历史记录，是 GFS 的核心。它不仅是元数据唯一的持久记录，还充当一条逻辑时间线，定义了并发操作的先后顺序。文件和 chunk 及其版本（见 4.5 节），都由它们被创建时的逻辑时间唯一且永久地标识。
{{% /bilingual %}}

{{% bilingual %}}
Since the operation log is critical, we must store it reliably and not make changes visible to clients until metadata changes are made persistent. Otherwise, we effectively lose the whole file system or recent client operations even if the chunks themselves survive. Therefore, we replicate it on multiple remote machines and respond to a client operation only after flushing the corresponding log record to disk both locally and remotely. The master batches several log records together before flushing thereby reducing the impact of flushing and replication on overall system throughput.
<!--col-->
由于操作日志至关重要，我们必须可靠地存储它，并且在元数据变更被持久化之前，不能让这些变更对客户端可见。否则即使 chunk 本身还在，我们实际上也会丢失整个文件系统或最近的客户端操作。因此，我们把日志复制到多台远程机器上，并且只有在把相应的日志记录同时刷写到本地和远程磁盘之后，才响应客户端操作。Master 会把若干条日志记录攒成一批再刷写，以此降低刷写和复制对整体系统吞吐的影响。
{{% /bilingual %}}

{{% bilingual %}}
The master recovers its file system state by replaying the operation log. To minimize startup time, we must keep the log small. The master checkpoints its state whenever the log grows beyond a certain size so that it can recover by loading the latest checkpoint from local disk and replaying only the limited number of log records after that. The checkpoint is in a compact B-tree like form that can be directly mapped into memory and used for namespace lookup without extra parsing. This further speeds up recovery and improves availability.
<!--col-->
Master 通过重放操作日志来恢复文件系统状态。为把启动时间降到最低，我们必须让日志保持较小。每当日志增长超过一定大小，master 就对自身状态做检查点（checkpoint），这样恢复时只需从本地磁盘载入最新的检查点，再重放其后数量有限的日志记录即可。检查点采用紧凑的类 B 树形式，可以直接映射进内存用于命名空间查找，无需额外解析。这进一步加速了恢复并提高了可用性。
{{% /bilingual %}}

**表 1 重绘（变更之后文件区域的状态）**

| 变更类型 | Write | Record Append |
|---|---|---|
| 串行成功 | defined | defined，其间夹杂 inconsistent |
| 并发成功 | consistent 但 undefined | inconsistent |
| 失败 | inconsistent | inconsistent |

{{% bilingual %}}
Because building a checkpoint can take a while, the master's internal state is structured in such a way that a new checkpoint can be created without delaying incoming mutations. The master switches to a new log file and creates the new checkpoint in a separate thread. The new checkpoint includes all mutations before the switch. It can be created in a minute or so for a cluster with a few million files. When completed, it is written to disk both locally and remotely.
<!--col-->
因为建立检查点可能耗时较久，master 的内部状态被组织成可以在不延迟新到变更的前提下创建新检查点的形式。Master 切换到新的日志文件，并在一个单独的线程中创建新检查点。新检查点包含切换之前的所有变更。对于有几百万个文件的集群，它大约一分钟就能建好。建成后，它会被写到本地和远程磁盘。
{{% /bilingual %}}

{{% bilingual %}}
Recovery needs only the latest complete checkpoint and subsequent log files. Older checkpoints and log files can be freely deleted, though we keep a few around to guard against catastrophes. A failure during checkpointing does not affect correctness because the recovery code detects and skips incomplete checkpoints.
<!--col-->
恢复只需要最新的完整检查点及其之后的日志文件。更早的检查点和日志文件可以随意删除，不过我们会保留少数几个以防灾难。检查点建立过程中的失败不影响正确性，因为恢复代码会检测并跳过不完整的检查点。
{{% /bilingual %}}

### 2.7 Consistency Model · 一致性模型

{{% bilingual %}}
GFS has a relaxed consistency model that supports our highly distributed applications well but remains relatively simple and efficient to implement. We now discuss GFS's guarantees and what they mean to applications. We also highlight how GFS maintains these guarantees but leave the details to other parts of the paper.
<!--col-->
GFS 有一个放宽的一致性模型，它能很好地支持我们高度分布式的应用，同时实现起来相对简单高效。下面讨论 GFS 提供哪些保证，以及这些保证对应用意味着什么。我们也会点出 GFS 如何维持这些保证，但把细节留到本文其他部分。
{{% /bilingual %}}

#### 2.7.1 Guarantees by GFS · GFS 提供的保证

{{% bilingual %}}
File namespace mutations (e.g., file creation) are atomic. They are handled exclusively by the master: namespace locking guarantees atomicity and correctness (Section 4.1); the master's operation log defines a global total order of these operations (Section 2.6.3).
<!--col-->
文件命名空间的变更（例如创建文件）是原子的。它们完全由 master 处理：命名空间加锁保证原子性和正确性（4.1 节）；master 的操作日志为这些操作定义了全局全序（2.6.3 节）。
{{% /bilingual %}}

{{% bilingual %}}
The state of a file region after a data mutation depends on the type of mutation, whether it succeeds or fails, and whether there are concurrent mutations. Table 1 summarizes the result. A file region is consistent if all clients will always see the same data, regardless of which replicas they read from. A region is defined after a file data mutation if it is consistent and clients will see what the mutation writes in its entirety. When a mutation succeeds without interference from concurrent writers, the affected region is defined (and by implication consistent): all clients will always see what the mutation has written. Concurrent successful mutations leave the region undefined but consistent: all clients see the same data, but it may not reflect what any one mutation has written. Typically, it consists of mingled fragments from multiple mutations. A failed mutation makes the region inconsistent (hence also undefined): different clients may see different data at different times. We describe below how our applications can distinguish defined regions from undefined regions. The applications do not need to further distinguish between different kinds of undefined regions.
<!--col-->
数据变更之后文件区域处于什么状态，取决于变更的类型、成功还是失败，以及是否存在并发变更。表 1 汇总了结果。如果无论从哪个副本读取，所有客户端看到的都是相同数据，那么这个文件区域就是 **consistent（一致的）**。如果某个区域在文件数据变更之后既一致、客户端又能看到该变更写入的全部内容，它就是 **defined（已定义的）**。当一次变更在不受并发写者干扰的情况下成功，受影响的区域就是 defined（因而也是一致的）：所有客户端总能看到该变更写入的内容。并发的多次成功变更会让该区域 undefined 但 consistent：所有客户端看到相同的数据，但它不一定反映任何单次变更写入的内容——通常它是来自多次变更的碎片混杂而成。失败的变更使该区域 inconsistent（因而也 undefined）：不同客户端在不同时刻可能看到不同数据。下面我们说明应用如何区分 defined 区域与 undefined 区域。应用不需要进一步区分不同种类的 undefined 区域。
{{% /bilingual %}}

{{% bilingual %}}
Data mutations may be writes or record appends. A write causes data to be written at an application-specified file offset. A record append causes data (the "record") to be appended atomically at least once even in the presence of concurrent mutations, but at an offset of GFS's choosing (Section 3.3). (In contrast, a "regular" append is merely a write at an offset that the client believes to be the current end of file.) The offset is returned to the client and marks the beginning of a defined region that contains the record. In addition, GFS may insert padding or record duplicates in between. They occupy regions considered to be inconsistent and are typically dwarfed by the amount of user data.
<!--col-->
数据变更可以是写（write）也可以是记录追加（record append）。写是把数据写到应用指定的文件偏移处。记录追加则是把数据（即"记录"）以原子的方式追加——即使在存在并发变更的情况下也至少追加一次——但偏移由 GFS 自行选择（3.3 节）。（相比之下，"普通"追加只是在客户端认为的文件末尾偏移处做一次写。）该偏移会返回给客户端，并标记出包含该记录的 defined 区域的起点。此外，GFS 可能在中间插入填充或记录副本。它们所占的区域被视为 inconsistent，而通常相对于用户数据量来说微不足道。
{{% /bilingual %}}

{{% bilingual %}}
After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. GFS achieves this by (a) applying mutations to a chunk in the same order on all its replicas (Section 3.1), and (b) using chunk version numbers to detect any replica that has become stale because it has missed mutations while its chunkserver was down (Section 4.5). Stale replicas will never be involved in a mutation or given to clients asking the master for chunk locations. They are garbage collected at the earliest opportunity.
<!--col-->
在一系列成功的变更之后，被变更的文件区域保证是 defined 的，并且包含最后一次变更写入的数据。GFS 通过两点做到这一点：(a) 对同一个 chunk 在所有副本上以相同顺序施加变更（3.1 节）；(b) 用 chunk 版本号检测出那些因 chunkserver 宕机期间错过变更而变旧的副本（4.5 节）。过期副本永远不会参与变更，也不会被返回给向 master 询问 chunk 位置的客户端。它们会在最早的机会被垃圾回收。
{{% /bilingual %}}

{{% bilingual %}}
Since clients cache chunk locations, they may read from a stale replica before that information is refreshed. This window is limited by the cache entry's timeout and the next open of the file, which purges from the cache all chunk information for that file. Moreover, as most of our files are append-only, a stale replica usually returns a premature end of chunk rather than outdated data. When a reader retries and contacts the master, it will immediately get current chunk locations.
<!--col-->
由于客户端会缓存 chunk 位置，在这份信息被刷新之前，它们可能从过期副本读取。这个窗口受限于缓存条目的超时时间，以及下一次打开该文件——打开操作会从缓存中清除该文件的所有 chunk 信息。而且，由于我们大多数文件是只追加的，过期副本通常返回的是 chunk 提前结束，而不是过时的数据。当读取方重试并联系 master 时，会立刻拿到当前的 chunk 位置。
{{% /bilingual %}}

{{% bilingual %}}
Long after a successful mutation, component failures can of course still corrupt or destroy data. GFS identifies failed chunkservers by regular handshakes between master and all chunkservers and detects data corruption by checksumming (Section 5.2). Once a problem surfaces, the data is restored from valid replicas as soon as possible (Section 4.3). A chunk is lost irreversibly only if all its replicas are lost before GFS can react, typically within minutes. Even in this case, it becomes unavailable, not corrupted: applications receive clear errors rather than corrupt data.
<!--col-->
在一次成功的变更很久之后，组件故障当然仍可能损坏或销毁数据。GFS 通过 master 与所有 chunkserver 之间的定期握手识别失效的 chunkserver，并通过校验和检测数据损坏（5.2 节）。一旦问题浮现，数据会尽快从有效副本恢复（4.3 节）。只有当某个 chunk 的所有副本都在 GFS 来得及反应之前丢失（通常是几分钟之内），它才会不可逆地丢失。即使在这种情况下，它也是变得不可用，而不是被损坏：应用收到的是明确的错误，而不是损坏的数据。
{{% /bilingual %}}

#### 2.7.2 Implications for Applications · 对应用的约束

{{% bilingual %}}
GFS applications can accommodate the relaxed consistency model with a few simple techniques already needed for other purposes: relying on appends rather than overwrites, checkpointing, and writing self-validating, self-identifying records.
<!--col-->
GFS 的应用可以用几种本就因其他目的而需要的简单技术来适配这个放宽的一致性模型：依赖追加而非覆盖、做检查点，以及写入可自校验、可自识别的记录。
{{% /bilingual %}}

{{% bilingual %}}
Practically all our applications mutate files by appending rather than overwriting. In one typical use, a writer generates a file from beginning to end. It atomically renames the file to a permanent name after writing all the data, or periodically checkpoints how much has been successfully written. Checkpoints may also include application-level checksums. Readers verify and process only the file region up to the last checkpoint, which is known to be in the defined state. Regardless of consistency and concurrency issues, this approach has served us well. Appending is far more efficient and more resilient to application failures than random writes. Checkpointing allows writers to restart incrementally and keeps readers from processing successfully written file data that is still incomplete from the application's perspective.
<!--col-->
我们几乎所有应用都通过追加而非覆盖来修改文件。一种典型用法是：写者从头到尾生成一个文件，在写完所有数据之后把文件原子地重命名为一个永久名字，或者周期性地把"已成功写入多少"记录成检查点。检查点也可以包含应用层的校验和。读取方只校验并处理到最后一个检查点为止的文件区域——这段已知处于 defined 状态。无论一致性和并发问题如何，这套做法一直很好用。追加比随机写高效得多，对应用故障也更有韧性。检查点让写者可以增量重启，也让读取方不会去处理那些虽然写成功、但从应用角度看仍不完整的数据。
{{% /bilingual %}}

{{% bilingual %}}
In the other typical use, many writers concurrently append to a file for merged results or as a producer-consumer queue. Record append's append-at-least-once semantics preserves each writer's output. Readers deal with the occasional padding and duplicates as follows. Each record prepared by the writer contains extra information like checksums so that its validity can be verified. A reader can identify and discard extra padding and record fragments using the checksums. If it cannot tolerate the occasional duplicates (e.g., if they would trigger non-idempotent operations), it can filter them out using unique identifiers in the records, which are often needed anyway to name corresponding application entities such as web documents. These functionalities for record I/O (except duplicate removal) are in library code shared by our applications and applicable to other file interface implementations at Google. With that, the same sequence of records, plus rare duplicates, is always delivered to the record reader.
<!--col-->
另一种典型用法是：多个写者并发向同一个文件追加，以得到归并结果或用作生产者—消费者队列。记录追加的"至少追加一次"语义保住了每个写者的输出。读取方按如下方式处理偶发的填充和重复：写者准备的每条记录都带校验和之类的额外信息，以便验证其有效性。读取方可以利用校验和识别并丢弃多余的填充和记录碎片。如果它无法容忍偶发的重复（例如重复会触发非幂等操作），可以用记录中的唯一标识符把它们过滤掉——而为了命名诸如网页文档这类对应的应用实体，这些标识符通常本来就是必需的。用于记录 I/O 的这些功能（重复剔除除外）都放在我们各应用共享的库代码里，也适用于 Google 内部其他文件接口实现。这样一来，交给记录读取方的总是同一串记录，外加极少量的重复。
{{% /bilingual %}}

## 3 System Interactions · 系统交互

{{% bilingual %}}
We designed the system to minimize the master's involvement in all operations. With that background, we now describe how the client, master, and chunkservers interact to implement data mutations, atomic record append, and snapshot.
<!--col-->
我们在设计系统时尽量让 master 少参与各类操作。基于这一背景，下面描述客户端、master 和 chunkserver 如何交互，以实现数据变更、原子记录追加和快照。
{{% /bilingual %}}

### 3.1 Leases and Mutation Order · 租约与变更顺序

{{% bilingual %}}
A mutation is an operation that changes the contents or metadata of a chunk such as a write or an append operation. Each mutation is performed at all the chunk's replicas. We use leases to maintain a consistent mutation order across replicas. The master grants a chunk lease to one of the replicas, which we call the primary. The primary picks a serial order for all mutations to the chunk. All replicas follow this order when applying mutations. Thus, the global mutation order is defined first by the lease grant order chosen by the master, and within a lease by the serial numbers assigned by the primary.
<!--col-->
变更（mutation）是指改变某个 chunk 内容或元数据的操作，比如写或追加。每次变更都会在该 chunk 的所有副本上执行。我们用租约（lease）来在各副本之间维持一致的变更顺序。Master 把某个 chunk 的租约授予其中一个副本，我们称之为 **primary（主副本）**。由 primary 为该 chunk 的所有变更挑选一个串行顺序，所有副本在施加变更时都遵循这个顺序。因此，全局变更顺序首先由 master 选择的租约授予顺序决定，而在一个租约内部则由 primary 分配的序列号决定。
{{% /bilingual %}}

{{% bilingual %}}
The lease mechanism is designed to minimize management overhead at the master. A lease has an initial timeout of 60 seconds. However, as long as the chunk is being mutated, the primary can request and typically receive extensions from the master indefinitely. These extension requests and grants are piggybacked on the HeartBeat messages regularly exchanged between the master and all chunkservers. The master may sometimes try to revoke a lease before it expires (e.g., when the master wants to disable mutations on a file that is being renamed). Even if the master loses communication with a primary, it can safely grant a new lease to another replica after the old lease expires.
<!--col-->
租约机制的设计目的是把 master 的管理开销降到最低。租约的初始超时是 60 秒。但只要该 chunk 正在被变更，primary 就可以向 master 请求延长，而且通常能无限期地获得延长。这些延长请求和授予搭在 master 与所有 chunkserver 之间定期交换的 HeartBeat 消息上。Master 有时会试图在租约到期前撤销它（例如当 master 想要禁止对一个正在被重命名的文件做变更时）。即使 master 与某个 primary 失去联系，它也可以在旧租约到期后，安全地把新租约授予另一个副本。
{{% /bilingual %}}

{{% bilingual %}}
In Figure 2, we illustrate this process by following the control flow of a write through these numbered steps.

1. The client asks the master which chunkserver holds the current lease for the chunk and the locations of the other replicas. If no one has a lease, the master grants one to a replica it chooses (not shown).
2. The master replies with the identity of the primary and the locations of the other (secondary) replicas. The client caches this data for future mutations. It needs to contact the master again only when the primary becomes unreachable or replies that it no longer holds a lease.
3. The client pushes the data to all the replicas. A client can do so in any order. Each chunkserver will store the data in an internal LRU buffer cache until the data is used or aged out. By decoupling the data flow from the control flow, we can improve performance by scheduling the expensive data flow based on the network topology regardless of which chunkserver is the primary. Section 3.2 discusses this further.
4. Once all the replicas have acknowledged receiving the data, the client sends a write request to the primary. The request identifies the data pushed earlier to all of the replicas. The primary assigns consecutive serial numbers to all the mutations it receives, possibly from multiple clients, which provides the necessary serialization. It applies the mutation to its own local state in serial number order.
5. The primary forwards the write request to all secondary replicas. Each secondary replica applies mutations in the same serial number order assigned by the primary.
6. The secondaries all reply to the primary indicating that they have completed the operation.
7. The primary replies to the client. Any errors encountered at any of the replicas are reported to the client. In case of errors, the write may have succeeded at the primary and an arbitrary subset of the secondary replicas. (If it had failed at the primary, it would not have been assigned a serial number and forwarded.) The client request is considered to have failed, and the modified region is left in an inconsistent state. Our client code handles such errors by retrying the failed mutation. It will make a few attempts at steps (3) through (7) before falling back to a retry from the beginning of the write.
<!--col-->
我们用图 2 中一次写的控制流，按编号步骤说明这个过程。

1. 客户端询问 master：哪个 chunkserver 持有该 chunk 当前的租约，以及其他副本的位置。如果没有任何副本持有租约，master 就把租约授予它选中的某个副本（图中未画出）。
2. Master 回复 primary 的身份以及其他（secondary）副本的位置。客户端缓存这份数据供后续变更使用。只有当 primary 变得不可达、或者回复说自己不再持有租约时，客户端才需要再次联系 master。
3. 客户端把数据推送到所有副本，顺序不限。每个 chunkserver 会把数据暂存在内部的 LRU 缓冲区中，直到数据被使用或被淘汰。通过把数据流与控制流解耦，我们可以依据网络拓扑来调度开销更大的数据流，而不必在意哪个 chunkserver 是 primary，从而提高性能。3.2 节进一步讨论。
4. 等所有副本都确认收到数据之后，客户端向 primary 发送写请求。请求中标识出此前推送给所有副本的那份数据。Primary 为它收到的所有变更（可能来自多个客户端）分配连续的序列号，从而提供必要的串行化。它按序列号顺序把变更施加到自己的本地状态上。
5. Primary 把写请求转发给所有 secondary 副本。每个 secondary 副本按 primary 分配的相同序列号顺序施加变更。
6. 所有 secondary 都回复 primary，表示已完成该操作。
7. Primary 回复客户端。任何副本上遇到的错误都会报告给客户端。出现错误时，这次写可能已在 primary 和任意的 secondary 子集上成功。（如果它在 primary 上就失败了，就不会被分配序列号、也不会被转发。）此时客户端请求被视为失败，被修改的区域处于 inconsistent 状态。我们的客户端代码通过重试失败的变更来处理这类错误：它会先在步骤 (3) 到 (7) 上尝试几次，然后退回到从这次写的最开始重试。
{{% /bilingual %}}

**图 2 重绘（写的控制流与数据流）**

```
        ┌────────┐
        │ Client │
        └───┬────┘
      ① 询问租约│ ② 返回 primary 与副本位置
        ┌───────▼────────┐
        │     Master     │
        └────────────────┘
                 │③ 推送数据（可按任意顺序）
     ┌───────────┼───────────────┐
     ▼           ▼               ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Primary  │ │Secondary │ │Secondary │
│ Replica  │ │ Replica A│ │ Replica B│
└────┬─────┘ └────▲─────┘ └────▲─────┘
     │ ④ 写请求    │ ⑤ 转发      │
     └────────────┴─────────────┘
     ▲            │ ⑥ 执行完成
     │            ▼
     └── ⑦ 回复客户端

图例：──▶ 控制流      ══▶ 数据流
数据先按链路推送（③），控制再沿 client → primary → secondaries 流动（④⑤）
```

{{% bilingual %}}
If a write by the application is large or straddles a chunk boundary, GFS client code breaks it down into multiple write operations. They all follow the control flow described above but may be interleaved with and overwritten by concurrent operations from other clients. Therefore, the shared file region may end up containing fragments from different clients, although the replicas will be identical because the individual operations are completed successfully in the same order on all replicas. This leaves the file region in consistent but undefined state as noted in Section 2.7.
<!--col-->
如果应用的一次写很大，或者跨越了 chunk 边界，GFS 客户端代码会把它拆成多个写操作。它们都遵循上面描述的控制流，但可能与其他客户端的并发操作交错、并被其覆盖。因此，共享的文件区域最终可能包含来自不同客户端的碎片——不过各副本之间仍是一致的，因为每个单独操作都在所有副本上以相同顺序成功完成。如 2.7 节所述，这使该文件区域处于 consistent 但 undefined 的状态。
{{% /bilingual %}}

### 3.2 Data Flow · 数据流

{{% bilingual %}}
We decouple the flow of data from the flow of control to use the network efficiently. While control flows from the client to the primary and then to all secondaries, data is pushed linearly along a carefully picked chain of chunkservers in a pipelined fashion. Our goals are to fully utilize each machine's network bandwidth, avoid network bottlenecks and high-latency links, and minimize the latency to push through all the data.
<!--col-->
为了高效利用网络，我们把数据流与控制流解耦。控制从客户端流向 primary、再流向所有 secondary；而数据则沿一条精心挑选的 chunkserver 链路，以流水线方式线性推送。我们的目标是充分利用每台机器的网络带宽、避开网络瓶颈和高延迟链路，并把推送全部数据的时延降到最低。
{{% /bilingual %}}

{{% bilingual %}}
To fully utilize each machine's network bandwidth, the data is pushed linearly along a chain of chunkservers rather than distributed in some other topology (e.g., tree). Thus, each machine's full outbound bandwidth is used to transfer the data as fast as possible rather than divided among multiple recipients.
<!--col-->
为充分利用每台机器的网络带宽，数据沿一条 chunkserver 链线性推送，而不是按其他拓扑（例如树形）分发。这样每台机器的全部出向带宽都用于尽快传输数据，而不是被多个接收方瓜分。
{{% /bilingual %}}

{{% bilingual %}}
To avoid network bottlenecks and high-latency links (e.g., inter-switch links are often both) as much as possible, each machine forwards the data to the "closest" machine in the network topology that has not received it. Suppose the client is pushing data to chunkservers S1 through S4. It sends the data to the closest chunkserver, say S1. S1 forwards it to the closest chunkserver S2 through S4 closest to S1, say S2. Similarly, S2 forwards it to S3 or S4, whichever is closer to S2, and so on. Our network topology is simple enough that "distances" can be accurately estimated from IP addresses.
<!--col-->
为尽可能避开网络瓶颈和高延迟链路（交换机之间的链路往往二者兼具），每台机器把数据转发给网络拓扑中尚未收到该数据的"最近"机器。假设客户端要向 chunkserver S1 到 S4 推送数据，它先把数据发给最近的 chunkserver，记为 S1。S1 再把它转发给 S2 到 S4 中距离 S1 最近的那个，记为 S2。类似地，S2 把它转发给 S3 或 S4 中距离 S2 更近的那个，依此类推。我们的网络拓扑足够简单，可以从 IP 地址准确估算"距离"。
{{% /bilingual %}}

{{% bilingual %}}
Finally, we minimize latency by pipelining the data transfer over TCP connections. Once a chunkserver receives some data, it starts forwarding immediately. Pipelining is especially helpful to us because we use a switched network with full-duplex links. Sending the data immediately does not reduce the receive rate. Without network congestion, the ideal elapsed time for transferring B bytes to R replicas is B/T + RL where T is the network throughput and L is latency to transfer bytes between two machines. Our network links are typically 100 Mbps (T), and L is far below 1 ms. Therefore, 1 MB can ideally be distributed in about 80 ms.
<!--col-->
最后，我们通过在 TCP 连接上对数据传输做流水线化来把时延降到最低：chunkserver 一收到部分数据就立刻开始转发。流水线对我们尤其有用，因为我们使用的是带全双工链路的交换式网络——立刻转发并不会降低接收速率。在没有网络拥塞的情况下，把 B 字节传输到 R 个副本的理想耗时为：
{{% /bilingual %}}

```
B/T + R·L     T 为网络吞吐，L 为两台机器之间传输字节的单程时延
```

我们的网络链路通常是 100 Mbps（即 T），L 远小于 1 ms。因此理想情况下 1 MB 大约 80 ms 就能分发完成。

### 3.3 Atomic Record Appends · 原子记录追加

{{% bilingual %}}
GFS provides an atomic append operation called record append. In a traditional write, the client specifies the offset at which data is to be written. Concurrent writes to the same region are not serializable: the region may end up containing data fragments from multiple clients. In a record append, however, the client specifies only the data. GFS appends it to the file at least once atomically (i.e., as one continuous sequence of bytes) at an offset of GFS's choosing and returns that offset to the client. This is similar to writing to a file opened in O_APPEND mode in Unix without the race conditions when multiple writers do so concurrently.
<!--col-->
GFS 提供了一个原子追加操作，称为记录追加（record append）。在传统的写里，客户端指定数据要写到的偏移；对同一区域的并发写不可串行化，该区域最终可能包含来自多个客户端的数据碎片。而在记录追加里，客户端只指定数据本身；GFS 把它以原子的方式（即作为一段连续的字节序列）至少一次地追加到文件中，偏移由 GFS 自行选择，并把这个偏移返回给客户端。这类似于在 Unix 中向以 O_APPEND 模式打开的文件写入，区别在于多个写者并发执行时不会出现竞态。
{{% /bilingual %}}

{{% bilingual %}}
Record append is heavily used by our distributed applications in which many clients on different machines append to the same file concurrently. Clients would need additional complicated and expensive synchronization, for example through a distributed lock manager, if they do so with traditional writes. In our workloads, such files often serve as multiple-producer/single-consumer queues or contain merged results from many different clients.
<!--col-->
记录追加在我们的分布式应用中被大量使用——这些场景下，不同机器上的许多客户端会并发追加同一个文件。如果改用传统写来做，客户端就需要额外的、复杂且昂贵的同步机制，例如通过分布式锁管理器。在我们的负载中，这类文件常被用作多生产者/单消费者队列，或用于容纳来自许多不同客户端的归并结果。
{{% /bilingual %}}

{{% bilingual %}}
Record append is a kind of mutation and follows the control flow in Section 3.1 with only a little extra logic at the primary. The client pushes the data to all replicas of the last chunk of the file. Then, it sends its request to the primary. The primary checks to see if appending the record to the current chunk would cause the chunk to exceed the maximum size (64 MB). If so, it pads the chunk to the maximum size, tells secondaries to do the same, and replies to the client indicating that the operation should be retried on the next chunk. (Record append is restricted to be at most one-fourth of the maximum chunk size to keep worst-case fragmentation at an acceptable level.) If the record fits within the maximum size, which is the common case, the primary appends the data to its replica, tells the secondaries to write the data at the exact offset where it has, and finally replies success to the client.
<!--col-->
记录追加是一种变更，除 primary 上多一点额外逻辑外，它遵循 3.1 节的控制流。客户端把数据推送到该文件最后一个 chunk 的所有副本，然后向 primary 发送请求。Primary 检查把该记录追加到当前 chunk 是否会使 chunk 超过最大尺寸（64 MB）。如果是，它就把该 chunk 填充到最大尺寸，通知 secondaries 照做，并回复客户端表示该操作应在下一个 chunk 上重试。（记录追加被限制为不超过最大 chunk 尺寸的四分之一，以把最坏情况下的碎片控制在可接受水平。）如果记录放得下——这是常见情况——primary 就把数据追加到自己的副本，通知 secondaries 在它写入的完全相同偏移处写入数据，最后回复客户端成功。
{{% /bilingual %}}

{{% bilingual %}}
If a record append fails at any replica, the client retries the operation. As a result, replicas of the same chunk may contain different data possibly including duplicates of the same record in whole or in part. GFS does not guarantee that all replicas are bytewise identical. It only guarantees that the data is written at least once as an atomic unit. This property follows readily from the simple observation that for the operation to report success, the data must have been written at the same offset on all replicas of some chunk. Furthermore, after this, all replicas are at least as long as the end of record and therefore any future record will be assigned a higher offset or a different chunk even if a different replica later becomes the primary. In terms of our consistency guarantees, the regions in which successful record append operations have written their data are defined (hence consistent), whereas intervening regions are inconsistent (hence undefined). Our applications can deal with inconsistent regions as we discussed in Section 2.7.2.
<!--col-->
如果记录追加在任一副本上失败，客户端会重试该操作。结果，同一个 chunk 的各副本可能包含不同的数据，甚至可能整体或部分地包含同一条记录的多份拷贝。GFS 不保证所有副本逐字节相同，它只保证数据作为原子单元被至少写入一次。这一性质很容易从下面这个简单观察推出：要让操作报告成功，数据必须在某个 chunk 的所有副本上写到了相同的偏移。而且在此之后，所有副本的长度至少都到达该记录的末尾，因此即使稍后另一个副本成为 primary，未来的任何记录也会被分配到更高的偏移或另一个 chunk。就我们的一致性保证而言，成功的记录追加操作写入其数据的那些区域是 defined（因而 consistent）的，而夹在中间的区域是 inconsistent（因而 undefined）的。应用可以按 2.7.2 节所述的方式处理不一致区域。
{{% /bilingual %}}

### 3.4 Snapshot · 快照

{{% bilingual %}}
The snapshot operation makes a copy of a file or a directory tree (the "source") almost instantaneously, while minimizing any interruptions of ongoing mutations. Our users use it to quickly create branch copies of huge data sets (and often copies of those copies, recursively), or to checkpoint the current state before experimenting with changes that can later be committed or rolled back easily.
<!--col-->
快照操作几乎瞬间就能完成一个文件或目录树（即"源"）的复制，同时尽量减少对正在进行中的变更的打断。我们的用户用它来快速创建大数据集的分支副本（而且常常对副本再建副本，递归进行），或者在试验改动之前为当前状态做检查点——这些改动之后可以方便地提交或回滚。
{{% /bilingual %}}

{{% bilingual %}}
Like AFS [5], we use standard copy-on-write techniques to implement snapshots. When the master receives a snapshot request, it first revokes any outstanding leases on the chunks in the files it is about to snapshot. This ensures that any subsequent writes to these chunks will require an interaction with the master to find the lease holder. This will give the master an opportunity to create a new copy of the chunk first.
<!--col-->
与 AFS [5] 类似，我们用标准的写时复制（copy-on-write）技术实现快照。当 master 收到快照请求时，它首先撤销即将做快照的那些文件里各个 chunk 上一切尚未到期的租约。这保证了此后对这些 chunk 的任何写都必须与 master 交互以找到租约持有者，从而给 master 一个机会先创建该 chunk 的新副本。
{{% /bilingual %}}

{{% bilingual %}}
After the leases have been revoked or have expired, the master logs the operation to disk. It then applies this log record to its in-memory state by duplicating the metadata for the source file or directory tree. The newly created snapshot files point to the same chunks as the source files.
<!--col-->
租约被撤销或到期之后，master 把该操作记录到磁盘日志。然后它把这条日志记录应用到内存状态上：复制源文件或目录树的元数据。新建的快照文件与源文件指向相同的 chunk。
{{% /bilingual %}}

{{% bilingual %}}
The first time a client wants to write to a chunk C after the snapshot operation, it sends a request to the master to find the current lease holder. The master notices that the reference count for chunk C is greater than one. It defers replying to the client request and instead picks a new chunk handle C'. It then asks each chunkserver that has a current replica of C to create a new chunk called C'. By creating the new chunk on the same chunkservers as the original, we ensure that the data can be copied locally, not over the network (our disks are about three times as fast as our 100 Mb Ethernet links). From this point, request handling is no different from that for any chunk: the master grants one of the replicas a lease on the new chunk C' and replies to the client, which can write the chunk normally, not knowing that it has just been created from an existing chunk.
<!--col-->
快照操作之后，客户端第一次想要写 chunk C 时，会向 master 请求查询当前的租约持有者。Master 发现 chunk C 的引用计数大于一，于是暂缓回复该客户端请求，转而选一个新的 chunk handle C'，然后要求每个持有 C 当前副本的 chunkserver 创建一个名为 C' 的新 chunk。把新 chunk 创建在与原 chunk 相同的 chunkserver 上，可以保证数据是本地复制而非走网络（我们的磁盘速度大约是 100 Mb 以太网链路的三倍）。从此之后，请求处理与任何 chunk 别无二致：master 把新 chunk C' 的租约授予其中一个副本并回复客户端，客户端可以正常写入，并不知道这个 chunk 刚刚是从一个已有 chunk 派生出来的。
{{% /bilingual %}}

## 4 Master Operation · master 的操作

{{% bilingual %}}
The master executes all namespace operations. In addition, it manages chunk replicas throughout the system: it makes placement decisions, creates new chunks and hence replicas, and coordinates various system-wide activities to keep chunks fully replicated, to balance load across all the chunkservers, and to reclaim unused storage. We now discuss each of these topics.
<!--col-->
Master 执行所有命名空间操作。此外，它还管理系统中的 chunk 副本：做放置决策、创建新 chunk（及其副本），并协调各种系统级活动，以保持 chunk 副本齐全、均衡所有 chunkserver 之间的负载，以及回收未使用的存储。下面逐一讨论这些主题。
{{% /bilingual %}}

### 4.1 Namespace Management and Locking · 命名空间管理与加锁

{{% bilingual %}}
Many master operations can take a long time: for example, a snapshot operation has to revoke chunkserver leases on all chunks covered by the snapshot. We do not want to delay other master operations while they are running. Therefore, we allow multiple operations to be active and use locks over regions of the namespace to ensure proper serialization.
<!--col-->
许多 master 操作可能耗时很久：例如，快照操作必须撤销快照所覆盖的所有 chunk 上的 chunkserver 租约。我们不希望在它们运行期间延迟其他 master 操作。因此，我们允许同时有多个操作处于活动状态，并用命名空间区域上的锁来保证正确的串行化。
{{% /bilingual %}}

{{% bilingual %}}
Unlike many traditional file systems, GFS does not have a per-directory data structure that lists all the files in that directory. Nor does it support aliases for the same file or directory (i.e, hard or symbolic links in Unix terms). GFS logically represents its namespace as a lookup table mapping full pathnames to metadata. With prefix compression, this table can be efficiently represented in memory. Each node in the namespace tree (either an absolute file name or an absolute directory name) has an associated read-write lock.
<!--col-->
与许多传统文件系统不同，GFS 没有"每个目录一个数据结构、列出该目录下所有文件"这种设计，也不支持同一文件或目录的别名（即 Unix 意义上的硬链接或符号链接）。GFS 在逻辑上把命名空间表示成一张"完整路径名 → 元数据"的查找表。借助前缀压缩，这张表可以高效地放在内存里。命名空间树中的每个节点（无论是绝对文件名还是绝对目录名）都关联着一把读写锁。
{{% /bilingual %}}

{{% bilingual %}}
Each master operation acquires a set of locks before it runs. Typically, if it involves /d1/d2/.../dn/leaf, it will acquire read-locks on the directory names /d1, /d1/d2, ..., /d1/d2/.../dn, and either a read lock or a write lock on the full pathname /d1/d2/.../dn/leaf. Note that leaf may be a file or directory depending on the operation.
<!--col-->
每个 master 操作在运行之前会获取一组锁。典型情况下，如果它涉及 `/d1/d2/.../dn/leaf`，它会在目录名 `/d1`、`/d1/d2`、…、`/d1/d2/.../dn` 上获取读锁，并在完整路径名 `/d1/d2/.../dn/leaf` 上获取读锁或写锁。注意 leaf 视操作不同，可能是文件也可能是目录。
{{% /bilingual %}}

{{% bilingual %}}
We now illustrate how this locking mechanism can prevent a file /home/user/foo from being created while /home/user is being snapshotted to /save/user. The snapshot operation acquires read locks on /home and /save, and write locks on /home/user and /save/user. The file creation acquires read locks on /home and /home/user, and a write lock on /home/user/foo. The two operations will be serialized properly because they try to obtain conflicting locks on /home/user. File creation does not require a write lock on the parent directory because there is no "directory", or inode-like, data structure to be protected from modification. The read lock on the name is sufficient to protect the parent directory from deletion.
<!--col-->
下面举例说明这套加锁机制如何阻止下面这种情况：在把 `/home/user` 做快照到 `/save/user` 的同时创建文件 `/home/user/foo`。快照操作在 `/home` 和 `/save` 上加读锁，在 `/home/user` 和 `/save/user` 上加写锁。文件创建在 `/home` 和 `/home/user` 上加读锁，在 `/home/user/foo` 上加写锁。两个操作会被正确地串行化，因为它们试图在 `/home/user` 上获取相互冲突的锁。文件创建不需要在父目录上加写锁，因为并不存在需要防止被修改的"目录"或类 inode 数据结构；对目录名的读锁已足以保护父目录不被删除。
{{% /bilingual %}}

{{% bilingual %}}
One nice property of this locking scheme is that it allows concurrent mutations in the same directory. For example, multiple file creations can be executed concurrently in the same directory: each acquires a read lock on the directory name and a write lock on the file name. The read lock on the directory name suffices to prevent the directory from being deleted, renamed, or snapshotted. The write locks on file names serialize attempts to create a file with the same name twice.
<!--col-->
这套加锁方案有一个很好的性质：它允许同一目录内的并发变更。例如，多个文件创建可以在同一目录中并发执行：每个都获取目录名上的读锁和文件名上的写锁。目录名上的读锁足以防止该目录被删除、重命名或做快照；而文件名上的写锁则把"两次创建同名文件"的尝试串行化。
{{% /bilingual %}}

{{% bilingual %}}
Since the namespace can have many nodes, read-write lock objects are allocated lazily and deleted once they are not in use. Also, locks are acquired in a consistent total order to prevent deadlock: they are first ordered by level in the namespace tree and lexicographically within the same level.
<!--col-->
由于命名空间可能有很多节点，读写锁对象是惰性分配的，一旦不再使用就被删除。此外，锁按一致的全序获取以防止死锁：先按它们在命名空间树中的层级排序，同一层级内则按字典序排序。
{{% /bilingual %}}

### 4.2 Replica Placement · 副本放置

{{% bilingual %}}
A GFS cluster is highly distributed at more levels than one. It typically has hundreds of chunkservers spread across many machine racks. These chunkservers in turn may be accessed from hundreds of clients from the same or different racks. Communication between two machines on different racks may cross one or more network switches. Additionally, bandwidth into or out of a rack may be less than the aggregate bandwidth of all the machines within the rack. Multi-level distribution presents a unique challenge to distribute data for scalability, reliability, and availability.
<!--col-->
一个 GFS 集群在多个层面高度分布。它通常有数百个 chunkserver 分散在许多机架上，而这些 chunkserver 又可能被来自同一机架或不同机架的数百个客户端访问。不同机架上两台机器之间的通信可能要跨越一个或多个网络交换机。此外，进出某个机架的带宽可能小于该机架内所有机器带宽的总和。这种多层级分布，为兼顾可扩展性、可靠性和可用性的数据分布带来了独特的挑战。
{{% /bilingual %}}

{{% bilingual %}}
The chunk replica placement policy serves two purposes: maximize data reliability and availability, and maximize network bandwidth utilization. For both, it is not enough to spread replicas across machines, which only guards against disk or machine failures and fully utilizes each machine's network bandwidth. We must also spread chunk replicas across racks. This ensures that some replicas of a chunk will survive and remain available even if an entire rack is damaged or offline (for example, due to failure of a shared resource like a network switch or power circuit). It also means that traffic, especially reads, for a chunk can exploit the aggregate bandwidth of multiple racks. On the other hand, write traffic has to flow through multiple racks, a tradeoff we make willingly.
<!--col-->
Chunk 副本的放置策略有两个目的：最大化数据的可靠性与可用性，以及最大化网络带宽利用率。对这两者而言，只把副本分散到不同机器上是不够的——那只防住了磁盘或机器故障，也只利用了每台机器自身的网络带宽。我们还必须把 chunk 副本分散到不同机架上。这保证了即使整个机架损坏或离线（例如因为网络交换机、供电回路这类共享资源失效），某个 chunk 仍会有一部分副本存活并保持可用；这也意味着该 chunk 的流量（尤其是读）可以利用多个机架的聚合带宽。另一方面，写流量必须穿越多个机架——这是我们心甘情愿接受的取舍。
{{% /bilingual %}}

### 4.3 Creation, Re-replication, Rebalancing · 创建、重新复制与再均衡

{{% bilingual %}}
Chunk replicas are created for three reasons: chunk creation, re-replication, and rebalancing.
<!--col-->
Chunk 副本的创建有三个原因：chunk 创建、重新复制（re-replication）和再均衡（rebalancing）。
{{% /bilingual %}}

{{% bilingual %}}
When the master creates a chunk, it chooses where to place the initially empty replicas. It considers several factors. (1) We want to place new replicas on chunkservers with below-average disk space utilization. Over time this will equalize disk utilization across chunkservers. (2) We want to limit the number of "recent" creations on each chunkserver. Although creation itself is cheap, it reliably predicts imminent heavy write traffic because chunks are created when demanded by writes, and in our append-once-read-many workload they typically become practically read-only once they have been completely written. (3) As discussed above, we want to spread replicas of a chunk across racks.
<!--col-->
当 master 创建一个 chunk 时，它要选择把这些初始为空的副本放在哪里，会考虑几个因素。(1) 我们希望把新副本放在磁盘空间使用率低于平均值的 chunkserver 上，久而久之这会让各 chunkserver 的磁盘使用率趋于均衡。(2) 我们希望限制每个 chunkserver 上"近期"创建的数量。创建本身虽然便宜，但它能可靠地预示即将到来的大量写入流量——因为 chunk 是因写需求而创建的，而在我们"一次追加、多次读取"的负载模式下，chunk 一旦被完整写入，通常实际上就变成只读的了。(3) 如上所述，我们希望把同一个 chunk 的副本分散到不同机架上。
{{% /bilingual %}}

{{% bilingual %}}
The master re-replicates a chunk as soon as the number of available replicas falls below a user-specified goal. This could happen for various reasons: a chunkserver becomes unavailable, it reports that its replica may be corrupted, one of its disks is disabled because of errors, or the replication goal is increased. Each chunk that needs to be re-replicated is prioritized based on several factors. One is how far it is from its replication goal. For example, we give higher priority to a chunk that has lost two replicas than to a chunk that has lost only one. In addition, we prefer to first re-replicate chunks for live files as opposed to chunks that belong to recently deleted files (see Section 4.4). Finally, to minimize the impact of failures on running applications, we boost the priority of any chunk that is blocking client progress.
<!--col-->
一旦某个 chunk 的可用副本数低于用户指定的目标值，master 就会重新复制它。这可能有多种原因：某个 chunkserver 变得不可用、它报告自己的副本可能已损坏、它的某块磁盘因错误被停用，或者复制目标值被调高。每个需要重新复制的 chunk 都会按若干因素排定优先级。因素之一是该 chunk 距离复制目标值差多远——例如，我们会给丢了两份副本的 chunk 比只丢了一份的更高优先级。此外，我们倾向于优先重新复制属于活跃文件的 chunk，而不是属于最近被删除的文件的 chunk（见 4.4 节）。最后，为把故障对运行中应用的影响降到最低，我们会提升任何正在阻塞客户端推进的 chunk 的优先级。
{{% /bilingual %}}

{{% bilingual %}}
The master picks the highest priority chunk and "clones" it by instructing some chunkserver to copy the chunk data directly from an existing valid replica. The new replica is placed with goals similar to those for creation: equalizing disk space utilization, limiting active clone operations on any single chunkserver, and spreading replicas across racks. To keep cloning traffic from overwhelming client traffic, the master limits the numbers of active clone operations both for the cluster and for each chunkserver. Additionally, each chunkserver limits the amount of bandwidth it spends on each clone operation by throttling its read requests to the source chunkserver.
<!--col-->
Master 挑出优先级最高的 chunk 并"克隆"它：指示某个 chunkserver 直接从现有的有效副本复制 chunk 数据。新副本的放置目标与创建时类似：均衡磁盘空间使用率、限制任一 chunkserver 上的活跃克隆操作数，以及把副本分散到不同机架。为防止克隆流量压垮客户端流量，master 对集群整体和每个 chunkserver 各自限制了活跃克隆操作的数量。此外，每个 chunkserver 通过对源 chunkserver 的读请求做限流，限制自己花在每次克隆操作上的带宽。
{{% /bilingual %}}

{{% bilingual %}}
Finally, the master rebalances replicas periodically: it examines the current replica distribution and moves replicas for better disk space and load balancing. Also through this process, the master gradually fills up a new chunkserver rather than instantly swamps it with new chunks and the heavy write traffic that comes with them. The placement criteria for the new replica are similar to those discussed above. In addition, the master must also choose which existing replica to remove. In general, it prefers to remove those on chunkservers with below-average free space so as to equalize disk space usage.
<!--col-->
最后，master 周期性地对副本做再均衡：它检查当前的副本分布，并迁移副本来改善磁盘空间和负载的均衡。同样通过这个过程，master 会逐步填满一个新的 chunkserver，而不是瞬间用大量新 chunk 以及随之而来的沉重写流量把它压垮。新副本的放置标准与上面讨论的类似。此外，master 还必须选择移除哪个现有副本。总体上，它倾向于移除空闲空间低于平均值的 chunkserver 上的副本，以均衡磁盘空间使用。
{{% /bilingual %}}

### 4.4 Garbage Collection · 垃圾回收

{{% bilingual %}}
After a file is deleted, GFS does not immediately reclaim the available physical storage. It does so only lazily during regular garbage collection at both the file and chunk levels. We find that this approach makes the system much simpler and more reliable.
<!--col-->
文件被删除后，GFS 不会立即回收可用的物理存储，而是在文件和 chunk 两个层级上、于定期垃圾回收期间惰性地回收。我们发现这种做法让系统简单得多、也可靠得多。
{{% /bilingual %}}

#### 4.4.1 Mechanism · 机制

{{% bilingual %}}
When a file is deleted by the application, the master logs the deletion immediately just like other changes. However instead of reclaiming resources immediately, the file is just renamed to a hidden name that includes the deletion timestamp. During the master's regular scan of the file system namespace, it removes any such hidden files if they have existed for more than three days (the interval is configurable). Until then, the file can still be read under the new, special name and can be undeleted by renaming it back to normal. When the hidden file is removed from the namespace, its in-memory metadata is erased. This effectively severs its links to all its chunks.
<!--col-->
当应用删除一个文件时，master 会像对待其他变更一样立刻记录这次删除。但它不会立即回收资源，而只是把该文件重命名为一个包含删除时间戳的隐藏名字。在 master 定期扫描文件系统命名空间时，它会删除那些已经存在超过三天（该间隔可配置）的此类隐藏文件。在那之前，该文件仍可以用这个新的特殊名字读取，也可以通过重命名回正常名字来恢复。当隐藏文件从命名空间中被移除时，它的内存元数据被抹去，这实际上切断了它到所有 chunk 的链接。
{{% /bilingual %}}

{{% bilingual %}}
In a similar regular scan of the chunk namespace, the master identifies orphaned chunks (i.e., those not reachable from any file) and erases the metadata for those chunks. In a HeartBeat message regularly exchanged with the master, each chunkserver reports a subset of the chunks it has, and the master replies with the identity of all chunks that are no longer present in the master's metadata. The chunkserver is free to delete its replicas of such chunks.
<!--col-->
在对 chunk 命名空间类似的定期扫描中，master 找出孤儿 chunk（即无法从任何文件到达的 chunk），并抹去这些 chunk 的元数据。在与 master 定期交换的 HeartBeat 消息中，每个 chunkserver 上报自己持有的一部分 chunk，master 则回复那些已不在 master 元数据中的所有 chunk 的标识。Chunkserver 可以自由删除这些 chunk 的副本。
{{% /bilingual %}}

#### 4.4.2 Discussion · 讨论

{{% bilingual %}}
Although distributed garbage collection is a hard problem that demands complicated solutions in the context of programming languages, it is quite simple in our case. We can easily identify all references to chunks: they are in the file-to-chunk mappings maintained exclusively by the master. We can also easily identify all the chunk replicas: they are Linux files under designated directories on each chunkserver. Any such replica not known to the master is "garbage."
<!--col-->
尽管在编程语言的语境里，分布式垃圾回收是一个需要复杂解决方案的难题，但在我们这里它相当简单。我们可以轻松找出所有对 chunk 的引用：它们就在由 master 独家维护的文件到 chunk 的映射里。我们也能轻松找出所有 chunk 副本：它们就是各 chunkserver 指定目录下的 Linux 文件。任何 master 不知道的此类副本，就是"垃圾"。
{{% /bilingual %}}

{{% bilingual %}}
The garbage collection approach to storage reclamation offers several advantages over eager deletion. First, it is simple and reliable in a large-scale distributed system where component failures are common. Chunk creation may succeed on some chunkservers but not others, leaving replicas that the master does not know exist. Replica deletion messages may be lost, and the master has to remember to resend them across failures, both its own and the chunkserver's. Garbage collection provides a uniform and dependable way to clean up any replicas not known to be useful. Second, it merges storage reclamation into the regular background activities of the master, such as the regular scans of namespaces and handshakes with chunkservers. Thus, it is done in batches and the cost is amortized. Moreover, it is done only when the master is relatively free. The master can respond more promptly to client requests that demand timely attention. Third, the delay in reclaiming storage provides a safety net against accidental, irreversible deletion.
<!--col-->
相比立即删除，用垃圾回收来做存储回收有几个优势。第一，在一个组件故障频发的大规模分布式系统中，它简单而可靠。Chunk 创建可能在某些 chunkserver 上成功而在另一些上失败，从而留下 master 并不知道其存在的副本；副本删除消息可能丢失，而 master 必须记得在自身和 chunkserver 的故障之后重发它们。垃圾回收提供了一种统一且可靠的方式来清理所有已知无用的副本。第二，它把存储回收并入 master 的常规后台活动，例如定期扫描命名空间和与 chunkserver 握手。因此它是批量完成的，成本被摊薄；而且只在 master 相对空闲时进行，使 master 能更及时地响应那些需要及时处理的客户端请求。第三，回收存储的延迟为意外、不可逆的删除提供了一层安全网。
{{% /bilingual %}}

{{% bilingual %}}
In our experience, the main disadvantage is that the delay sometimes hinders user effort to fine tune usage when storage is tight. Applications that repeatedly create and delete temporary files may not be able to reuse the storage right away. We address these issues by expediting storage reclamation if a deleted file is explicitly deleted again. We also allow users to apply different replication and reclamation policies to different parts of the namespace. For example, users can specify that all the chunks in the files within some directory tree are to be stored without replication, and any deleted files are immediately and irrevocably removed from the file system state.
<!--col-->
根据我们的经验，主要的缺点是这种延迟有时会妨碍用户在存储紧张时精细调整用量：反复创建和删除临时文件的应用可能无法立刻复用这些存储。我们通过以下方式解决：如果被删除的文件被再次显式删除，就加速回收；同时允许用户对命名空间的不同部分施加不同的复制与回收策略。例如，用户可以指定某个目录树内所有文件的 chunk 都以无副本方式存储，并且任何被删除的文件都立即且不可逆地从文件系统状态中移除。
{{% /bilingual %}}

### 4.5 Stale Replica Detection · 过期副本检测

{{% bilingual %}}
Chunk replicas may become stale if a chunkserver fails and misses mutations to the chunk while it is down. For each chunk, the master maintains a chunk version number to distinguish between up-to-date and stale replicas.
<!--col-->
如果某个 chunkserver 发生故障、并在宕机期间错过了对该 chunk 的变更，那么 chunk 副本就可能变旧。对每个 chunk，master 维护一个 chunk 版本号，用于区分最新副本和过期副本。
{{% /bilingual %}}

{{% bilingual %}}
Whenever the master grants a new lease on a chunk, it increases the chunk version number and informs the up-to-date replicas. The master and these replicas all record the new version number in their persistent state. This occurs before any client is notified and therefore before it can start writing to the chunk. If another replica is currently unavailable, its chunk version number will not be advanced. The master will detect that this chunkserver has a stale replica when the chunkserver restarts and reports its set of chunks and their associated version numbers. If the master sees a version number greater than the one in its records, the master assumes that it failed when granting the lease and so takes the higher version to be up-to-date.
<!--col-->
每当 master 授予某个 chunk 一个新租约，它就递增该 chunk 的版本号，并通知最新的那些副本。Master 和这些副本都把新版本号记入各自的持久状态。这发生在通知任何客户端之前，因而是在该客户端可能开始向该 chunk 写入之前。如果另一个副本当前不可用，它的 chunk 版本号就不会被推进。当该 chunkserver 重启并上报自己持有的 chunk 集合及其对应版本号时，master 会发现它持有过期副本。如果 master 看到的版本号比它记录中的更大，master 就认定自己在授予租约时失败了，于是以更高的版本号为最新。
{{% /bilingual %}}

{{% bilingual %}}
The master removes stale replicas in its regular garbage collection. Before that, it effectively considers a stale replica not to exist at all when it replies to client requests for chunk information. As another safeguard, the master includes the chunk version number when it informs clients which chunkserver holds a lease on a chunk or when it instructs a chunkserver to read the chunk from another chunkserver in a cloning operation. The client or the chunkserver verifies the version number when it performs the operation so that it is always accessing up-to-date data.
<!--col-->
Master 在定期垃圾回收中移除过期副本。在那之前，当它回复客户端关于 chunk 信息的请求时，实际上完全把过期副本视为不存在。作为另一重保障，master 在告知客户端哪个 chunkserver 持有某 chunk 的租约时，或在克隆操作中指示某个 chunkserver 从另一个 chunkserver 读取该 chunk 时，都会附上 chunk 版本号。客户端或 chunkserver 在执行操作时校验版本号，从而保证访问的始终是最新数据。
{{% /bilingual %}}

## 5 Fault Tolerance and Diagnosis · 容错与诊断

{{% bilingual %}}
One of our greatest challenges in designing the system is dealing with frequent component failures. The quality and quantity of components together make these problems more the norm than the exception: we cannot completely trust the machines, nor can we completely trust the disks. Component failures can result in an unavailable system or, worse, corrupted data. We discuss how we meet these challenges and the tools we have built into the system to diagnose problems when they inevitably occur.
<!--col-->
设计这套系统时我们最大的挑战之一，就是应对频繁的组件故障。部件的质量与数量共同使这些问题成为常态而非例外：我们既不能完全信任机器，也不能完全信任磁盘。组件故障可能导致系统不可用，更糟的是导致数据损坏。下面讨论我们如何应对这些挑战，以及我们在系统中内置了哪些工具，用于在问题不可避免地发生时进行诊断。
{{% /bilingual %}}

### 5.1 High Availability · 高可用

{{% bilingual %}}
Among hundreds of servers in a GFS cluster, some are bound to be unavailable at any given time. We keep the overall system highly available with two simple yet effective strategies: fast recovery and replication.
<!--col-->
在一个 GFS 集群的数百台服务器中，任何时刻都必然有一些不可用。我们用两种简单却有效的策略让整个系统保持高可用：快速恢复与复制。
{{% /bilingual %}}

#### 5.1.1 Fast Recovery · 快速恢复

{{% bilingual %}}
Both the master and the chunkserver are designed to restore their state and start in seconds no matter how they terminated. In fact, we do not distinguish between normal and abnormal termination; servers are routinely shut down just by killing the process. Clients and other servers experience a minor hiccup as they time out on their outstanding requests, reconnect to the restarted server, and retry. Section 6.2.2 reports observed startup times.
<!--col-->
Master 和 chunkserver 都被设计成无论以何种方式终止，都能在数秒内恢复状态并启动。事实上，我们并不区分正常终止与异常终止；服务器例行就是用杀进程的方式关闭。客户端和其他服务器只会经历一次轻微的顿挫：它们尚未完成的请求超时，然后重新连接到重启后的服务器并重试。6.2.2 节报告了观察到的启动时间。
{{% /bilingual %}}

#### 5.1.2 Chunk Replication · chunk 复制

{{% bilingual %}}
As discussed earlier, each chunk is replicated on multiple chunkservers on different racks. Users can specify different replication levels for different parts of the file namespace. The default is three. The master clones existing replicas as needed to keep each chunk fully replicated as chunkservers go offline or detect corrupted replicas through checksum verification (see Section 5.2). Although replication has served us well, we are exploring other forms of cross-server redundancy such as parity or erasure codes for our increasing read-only storage requirements. We expect that it is challenging but manageable to implement these more complicated redundancy schemes in our very loosely coupled system because our traffic is dominated by appends and reads rather than small random writes.
<!--col-->
如前所述，每个 chunk 都会复制到不同机架上的多个 chunkserver。用户可以为文件命名空间的不同部分指定不同的副本级别，默认是三。当 chunkserver 下线，或通过校验和校验发现副本损坏时（见 5.2 节），master 会按需克隆现有副本来让每个 chunk 保持副本齐全。虽然复制一直很好用，但我们正在为日益增长的只读存储需求探索其他形式的跨服务器冗余，例如奇偶校验或纠删码。我们预计，在我们这种耦合非常松散的系统中实现这些更复杂的冗余方案虽有挑战，但仍在可控范围内，因为我们的流量以追加和读为主，而不是小的随机写。
{{% /bilingual %}}

#### 5.1.3 Master Replication · master 复制

{{% bilingual %}}
The master state is replicated for reliability. Its operation log and checkpoints are replicated on multiple machines. A mutation to the state is considered committed only after its log record has been flushed to disk locally and on all master replicas. For simplicity, one master process remains in charge of all mutations as well as background activities such as garbage collection that change the system internally. When it fails, it can restart almost instantly. If its machine or disk fails, monitoring infrastructure outside GFS starts a new master process elsewhere with the replicated operation log. Clients use only the canonical name of the master (e.g. gfs-test), which is a DNS alias that can be changed if the master is relocated to another machine.
<!--col-->
Master 状态为可靠性而做复制：它的操作日志和检查点被复制到多台机器上。对状态的变更只有在其日志记录被刷写到本地磁盘以及所有 master 副本的磁盘之后，才被视为已提交。为了简单，始终只有一个 master 进程负责所有变更，以及垃圾回收这类会改变系统内部状态的后台活动。它失败时可以几乎瞬时重启。如果它的机器或磁盘失效，GFS 之外的监控基础设施会带着复制过来的操作日志在别处启动一个新的 master 进程。客户端只使用 master 的规范名（例如 `gfs-test`），这是一个 DNS 别名，当 master 迁移到另一台机器时可以更改。
{{% /bilingual %}}

{{% bilingual %}}
Moreover, "shadow" masters provide read-only access to the file system even when the primary master is down. They are shadows, not mirrors, in that they may lag the primary slightly, typically fractions of a second. They enhance read availability for files that are not being actively mutated or applications that do not mind getting slightly stale results. In fact, since file content is read from chunkservers, applications do not observe stale file content. What could be stale within short windows is file metadata, like directory contents or access control information.
<!--col-->
此外，"影子"（shadow）master 即使在主 master 宕机时也能提供对文件系统的只读访问。它们是影子而非镜像——它们可能略微落后于主 master，通常是零点几秒。对于没有被活跃变更的文件，或者不介意拿到略微过期结果的应用，它们提升了读的可用性。事实上，由于文件内容是直接从 chunkserver 读取的，应用并不会观察到过期的文件内容；短时间内可能过期的只是文件元数据，例如目录内容或访问控制信息。
{{% /bilingual %}}

{{% bilingual %}}
To keep itself informed, a shadow master reads a replica of the growing operation log and applies the same sequence of changes to its data structures exactly as the primary does. Like the primary, it polls chunkservers at startup (and infrequently thereafter) to locate chunk replicas and exchanges frequent handshake messages with them to monitor their status. It depends on the primary master only for replica location updates resulting from the primary's decisions to create and delete replicas.
<!--col-->
为了让自己保持同步，影子 master 读取不断增长的操作日志的一个副本，并完全像主 master 那样把相同的变更序列施加到自己的数据结构上。与主 master 一样，它在启动时（以及此后的低频时间点）向 chunkserver 轮询以定位 chunk 副本，并与它们频繁交换握手消息以监控状态。它只在副本位置更新上依赖主 master——而那是由主 master 创建和删除副本的决策所导致的。
{{% /bilingual %}}

### 5.2 Data Integrity · 数据完整性

{{% bilingual %}}
Each chunkserver uses checksumming to detect corruption of stored data. Given that a GFS cluster often has thousands of disks on hundreds of machines, it regularly experiences disk failures that cause data corruption or loss on both the read and write paths. (See Section 7 for one cause.) We can recover from corruption using other chunk replicas, but it would be impractical to detect corruption by comparing replicas across chunkservers. Moreover, divergent replicas may be legal: the semantics of GFS mutations, in particular atomic record append as discussed earlier, does not guarantee identical replicas. Therefore, each chunkserver must independently verify the integrity of its own copy by maintaining checksums.
<!--col-->
每个 chunkserver 都用校验和来检测所存数据的损坏。考虑到一个 GFS 集群往往在数百台机器上拥有数千块磁盘，它经常遭遇磁盘故障，在读写两条路径上都可能造成数据损坏或丢失（其中一个成因见第 7 节）。我们可以借助其他 chunk 副本来从损坏中恢复，但靠跨 chunkserver 比较副本来检测损坏并不现实。而且副本之间存在差异可能是合法的：GFS 变更的语义——特别是前面讨论过的原子记录追加——并不保证副本完全一致。因此，每个 chunkserver 必须通过维护校验和来独立校验自己那份拷贝的完整性。
{{% /bilingual %}}

{{% bilingual %}}
A chunk is broken up into 64 KB blocks. Each has a corresponding 32 bit checksum. Like other metadata, checksums are kept in memory and stored persistently with logging, separate from user data.
<!--col-->
一个 chunk 被拆成 64 KB 的块，每块对应一个 32 位校验和。与其他元数据一样，校验和保存在内存中，并与用户数据分开、通过日志持久化存储。
{{% /bilingual %}}

{{% bilingual %}}
For reads, the chunkserver verifies the checksum of data blocks that overlap the read range before returning any data to the requester, whether a client or another chunkserver. Therefore chunkservers will not propagate corruptions to other machines. If a block does not match the recorded checksum, the chunkserver returns an error to the requestor and reports the mismatch to the master. In response, the requestor will read from other replicas, while the master will clone the chunk from another replica. After a valid new replica is in place, the master instructs the chunkserver that reported the mismatch to delete its replica.
<!--col-->
对于读，chunkserver 在向请求方（无论是客户端还是另一个 chunkserver）返回任何数据之前，会校验与读取区间有重叠的那些数据块的校验和。因此 chunkserver 不会把损坏扩散到其他机器。如果某个块与记录的校验和不符，chunkserver 就向请求方返回错误，并把这次不匹配上报给 master。作为响应，请求方会改从其他副本读取，而 master 会从另一个副本克隆该 chunk。新的有效副本就位之后，master 指示那个上报不匹配的 chunkserver 删除自己的副本。
{{% /bilingual %}}

{{% bilingual %}}
Checksumming has little effect on read performance for several reasons. Since most of our reads span at least a few blocks, we need to read and checksum only a relatively small amount of extra data for verification. GFS client code further reduces this overhead by trying to align reads at checksum block boundaries. Moreover, checksum lookups and comparison on the chunkserver are done without any I/O, and checksum calculation can often be overlapped with I/Os.
<!--col-->
校验和对读性能影响很小，原因有几条。由于我们大多数读都至少跨越几个块，为校验而需要额外读取和计算校验和的数据量相对很小。GFS 客户端代码还通过尽量把读对齐到校验块边界来进一步降低这部分开销。此外，chunkserver 上的校验和查找与比较不需要任何 I/O，而校验和计算往往可以与 I/O 重叠进行。
{{% /bilingual %}}

{{% bilingual %}}
Checksum computation is heavily optimized for writes that append to the end of a chunk (as opposed to writes that overwrite existing data) because they are dominant in our workloads. We just incrementally update the checksum for the last partial checksum block, and compute new checksums for any brand new checksum blocks filled by the append. Even if the last partial checksum block is already corrupted and we fail to detect it now, the new checksum value will not match the stored data, and the corruption will be detected as usual when the block is next read.
<!--col-->
校验和计算针对"追加到 chunk 末尾的写"（相对于覆盖已有数据的写）做了大量优化，因为这类写在我们的负载中占主导。我们只需增量更新最后一个不完整校验块的校验和，并为这次追加所填满的全新校验块计算新的校验和。即使最后那个不完整的校验块已经损坏而我们这次没能发现，新的校验和也不会与存储的数据相符，于是在该块下次被读取时，损坏照常会被检测出来。
{{% /bilingual %}}

{{% bilingual %}}
In contrast, if a write overwrites an existing range of the chunk, we must read and verify the first and last blocks of the range being overwritten, then perform the write, and finally compute and record the new checksums. If we do not verify the first and last blocks before overwriting them partially, the new checksums may hide corruption that exists in the regions not being overwritten.
<!--col-->
相反，如果一次写覆盖了 chunk 中一段已有区间，我们必须先读取并校验被覆盖区间的首块和末块，然后执行写，最后计算并记录新的校验和。如果我们在部分覆盖首末两块之前不做校验，新的校验和就可能掩盖未被覆盖区域中已存在的损坏。
{{% /bilingual %}}

{{% bilingual %}}
During idle periods, chunkservers can scan and verify the contents of inactive chunks. This allows us to detect corruption in chunks that are rarely read. Once the corruption is detected, the master can create a new uncorrupted replica and delete the corrupted replica. This prevents an inactive but corrupted chunk replica from fooling the master into thinking that it has enough valid replicas of a chunk.
<!--col-->
在空闲时段，chunkserver 可以扫描并校验不活跃 chunk 的内容。这让我们能发现那些很少被读取的 chunk 中的损坏。一旦检出损坏，master 就可以创建一个新的未损坏副本并删除损坏的副本。这可以防止一个不活跃但已损坏的 chunk 副本骗过 master，让它以为某个 chunk 已经有足够多的有效副本。
{{% /bilingual %}}

### 5.3 Diagnostic Tools · 诊断工具

{{% bilingual %}}
Extensive and detailed diagnostic logging has helped immeasurably in problem isolation, debugging, and performance analysis, while incurring only a minimal cost. Without logs, it is hard to understand transient, non-repeatable interactions between machines. GFS servers generate diagnostic logs that record many significant events (such as chunkservers going up and down) and all RPC requests and replies. These diagnostic logs can be freely deleted without affecting the correctness of the system. However, we try to keep these logs around as far as space permits.
<!--col-->
广泛而详细的诊断日志在问题定位、调试和性能分析上帮助极大，而代价却微乎其微。没有日志，很难理解机器之间那些瞬时发生、无法复现的交互。GFS 服务器产生的诊断日志记录了许多重要事件（例如 chunkserver 上下线）以及所有 RPC 请求与应答。这些诊断日志可以随意删除而不影响系统正确性，不过在空间允许的前提下我们尽量把它们留着。
{{% /bilingual %}}

{{% bilingual %}}
The RPC logs include the exact requests and responses sent on the wire, except for the file data being read or written. By matching requests with replies and collating RPC records on different machines, we can reconstruct the entire interaction history to diagnose a problem. The logs also serve as traces for load testing and performance analysis.
<!--col-->
RPC 日志包含线上实际收发的请求与应答（读写的数据本身除外）。通过把请求与应答配对、并与不同机器上的 RPC 记录对照拼接，我们可以重建完整的交互历史来诊断问题。这些日志还可以作为负载测试和性能分析的追踪记录。
{{% /bilingual %}}

{{% bilingual %}}
The performance impact of logging is minimal (and far outweighed by the benefits) because these logs are written sequentially and asynchronously. The most recent events are also kept in memory and available for continuous online monitoring.
<!--col-->
日志对性能的影响很小（且远小于它带来的收益），因为这些日志是顺序、异步写入的。最近的事件还会保留在内存中，可用于持续的在线监控。
{{% /bilingual %}}

## 6 Measurements · 测量

{{% bilingual %}}
In this section we present a few micro-benchmarks to illustrate the bottlenecks inherent in the GFS architecture and implementation, and also some numbers from real clusters in use at Google.
<!--col-->
本节给出几个微基准测试，用以说明 GFS 架构与实现中固有的瓶颈，另外还给出 Google 内部在用的真实集群的一些数字。
{{% /bilingual %}}

### 6.1 Micro-benchmarks · 微基准测试

{{% bilingual %}}
We measured performance on a GFS cluster consisting of one master, two master replicas, 16 chunkservers, and 16 clients. Note that this configuration was set up for ease of testing. Typical clusters have hundreds of chunkservers and hundreds of clients.

All the machines are configured with dual 1.4 GHz PIII processors, 2 GB of memory, two 80 GB 5400 rpm disks, and a 100 Mbps full-duplex Ethernet connection to an HP 2524 switch. All 19 GFS server machines are connected to one switch, and all 16 client machines to the other. The two switches are connected with a 1 Gbps link.
<!--col-->
我们在一台 master、两台 master 副本、16 台 chunkserver 和 16 个客户端组成的 GFS 集群上测量性能。注意该配置是为了便于测试而搭建的，典型集群有数百台 chunkserver 和数百个客户端。

所有机器配置为双 1.4 GHz PIII 处理器、2 GB 内存、两块 80 GB 5400 rpm 磁盘，以及一条连到 HP 2524 交换机的 100 Mbps 全双工以太网连接。19 台 GFS 服务器机器接到一台交换机上，16 台客户端机器接到另一台上。两台交换机之间用 1 Gbps 链路相连。
{{% /bilingual %}}

#### 6.1.1 Reads · 读

{{% bilingual %}}
N clients read simultaneously from the file system. Each client reads a randomly selected 4 MB region from a 320 GB file set. This is repeated 256 times so that each client ends up reading 1 GB of data. The chunkservers taken together have only 32 GB of memory, so we expect at most a 10% hit rate in the Linux buffer cache. Our results should be close to cold cache results.
<!--col-->
N 个客户端同时从文件系统读取。每个客户端从 320 GB 的文件集合中随机选取一个 4 MB 区域读取，重复 256 次，最终每个客户端读取 1 GB 数据。各 chunkserver 合计只有 32 GB 内存，因此我们预期 Linux buffer cache 的命中率至多 10%，测量结果应接近冷缓存的情况。
{{% /bilingual %}}

{{% bilingual %}}
Figure 3(a) shows the aggregate read rate for N clients and its theoretical limit. The limit peaks at an aggregate of 125 MB/s when the 1 Gbps link between the two switches is saturated, or 12.5 MB/s per client when its 100 Mbps network interface gets saturated, whichever applies. The observed read rate is 10 MB/s, or 80% of the per-client limit, when just one client is reading. The aggregate read rate reaches 94 MB/s, about 75% of the 125 MB/s link limit, for 16 readers, or 6 MB/s per client. The efficiency drops from 80% to 75% because as the number of readers increases, so does the probability that multiple readers simultaneously read from the same chunkserver.
<!--col-->
图 3(a) 给出 N 个客户端的聚合读速率及其理论上限。上限在两台交换机之间的 1 Gbps 链路被打满时为 125 MB/s（聚合），或在客户端的 100 Mbps 网卡被打满时为每客户端 12.5 MB/s，取其中先达到的那个。仅有一个客户端在读时，观察到的读速率是 10 MB/s，即每客户端上限的 80%。16 个读者时，聚合读速率达到 94 MB/s，约为 125 MB/s 链路上限的 75%，即每客户端 6 MB/s。效率从 80% 降到 75%，是因为随着读者数量增加，多个读者同时从同一个 chunkserver 读取的概率也随之上升。
{{% /bilingual %}}

#### 6.1.2 Writes · 写

{{% bilingual %}}
N clients write simultaneously to N distinct files. Each client writes 1 GB of data to a new file in a series of 1 MB writes. The aggregate write rate and its theoretical limit are shown in Figure 3(b). The limit plateaus at 67 MB/s because we need to write each byte to 3 of the 16 chunkservers, each with a 12.5 MB/s input connection.
<!--col-->
N 个客户端同时向 N 个不同的文件写入。每个客户端以一系列 1 MB 的写把 1 GB 数据写入一个新文件。聚合写速率及其理论上限见图 3(b)。上限稳定在 67 MB/s，因为每个字节都要写入 16 台 chunkserver 中的 3 台，而每台的输入连接是 12.5 MB/s。
{{% /bilingual %}}

{{% bilingual %}}
The write rate for one client is 6.3 MB/s, about half of the limit. The main culprit for this is our network stack. It does not interact very well with the pipelining scheme we use for pushing data to chunk replicas. Delays in propagating data from one replica to another reduce the overall write rate.
<!--col-->
单个客户端的写速率是 6.3 MB/s，约为上限的一半。主要症结在我们的网络协议栈：它与我们用来向 chunk 副本推送数据的流水线方案配合得不太好，数据从一个副本传播到另一个副本的延迟拉低了整体写速率。
{{% /bilingual %}}

{{% bilingual %}}
Aggregate write rate reaches 35 MB/s for 16 clients (or 2.2 MB/s per client), about half the theoretical limit. As in the case of reads, it becomes more likely that multiple clients write concurrently to the same chunkserver as the number of clients increases. Moreover, collision is more likely for 16 writers than for 16 readers because each write involves three different replicas.
<!--col-->
16 个客户端时聚合写速率达到 35 MB/s（即每客户端 2.2 MB/s），约为理论上限的一半。与读的情况一样，随着客户端数量增加，多个客户端同时向同一个 chunkserver 写入的概率也上升。此外，16 个写者比 16 个读者更容易发生冲突，因为每次写都涉及三个不同的副本。
{{% /bilingual %}}

{{% bilingual %}}
Writes are slower than we would like. In practice this has not been a major problem because even though it increases the latencies as seen by individual clients, it does not significantly affect the aggregate write bandwidth delivered by the system to a large number of clients.
<!--col-->
写的速度比我们期望的慢。实践中这并未构成大问题，因为尽管它提高了单个客户端观察到的时延，但并不显著影响系统向大量客户端提供的聚合写带宽。
{{% /bilingual %}}

#### 6.1.3 Record Appends · 记录追加

{{% bilingual %}}
Figure 3(c) shows record append performance. N clients append simultaneously to a single file. Performance is limited by the network bandwidth of the chunkservers that store the last chunk of the file, independent of the number of clients. It starts at 6.0 MB/s for one client and drops to 4.8 MB/s for 16 clients, mostly due to congestion and variances in network transfer rates seen by different clients.
<!--col-->
图 3(c) 给出记录追加的性能。N 个客户端同时向单个文件追加。性能受限于存放该文件最后一个 chunk 的那些 chunkserver 的网络带宽，与客户端数量无关。它从单个客户端时的 6.0 MB/s 降到 16 个客户端时的 4.8 MB/s，主要原因是拥塞以及不同客户端所观察到的网络传输速率存在差异。
{{% /bilingual %}}

{{% bilingual %}}
Our applications tend to produce multiple such files concurrently. In other words, N clients append to M shared files simultaneously where both N and M are in the dozens or hundreds. Therefore, the chunkserver network congestion in our experiment is not a significant issue in practice because a client can make progress on writing one file while the chunkservers for another file are busy.
<!--col-->
我们的应用往往会并发产生多个这样的文件。换句话说，N 个客户端同时向 M 个共享文件追加，其中 N 和 M 都在几十到几百的量级。因此，实验中的 chunkserver 网络拥塞在实践中并不构成显著问题，因为当某个文件对应的 chunkserver 正忙时，客户端仍可以在另一个文件上取得进展。
{{% /bilingual %}}

**图 3 重绘（三张折线图的关键读数）**

原文图 3 是三条随客户端数 N（0、5、10、15）变化的聚合速率曲线，分别是 (a) 读、(b) 写、(c) 记录追加，各自叠加一条理论上限线。关键读数如下：

| 子图 | 理论上限 | N=1 | N=16 | 说明 |
|---|---|---|---|---|
| (a) 读 | 125 MB/s（链路）/ 12.5 MB/s（每客户端） | 10 MB/s | 94 MB/s（6 MB/s/客户端） | 效率由 80% 降至 75% |
| (b) 写 | 67 MB/s | 6.3 MB/s | 35 MB/s（2.2 MB/s/客户端） | 约为上限的一半 |
| (c) 记录追加 | 受末段 chunk 所在 chunkserver 带宽限制 | 6.0 MB/s | 4.8 MB/s | 与客户端数无关 |

### 6.2 Real World Clusters · 真实生产集群

{{% bilingual %}}
We now examine two clusters in use within Google that are representative of several others like them. Cluster A is used regularly for research and development by over a hundred engineers. A typical task is initiated by a human user and runs up to several hours. It reads through a few MBs to a few TBs of data, transforms or analyzes the data, and writes the results back to the cluster. Cluster B is primarily used for production data processing. The tasks last much longer and continuously generate and process multi-TB data sets with only occasional human intervention. In both cases, a single "task" consists of many processes on many machines reading and writing many files simultaneously.
<!--col-->
下面考察 Google 内部在用的两个集群，它们可以代表其他几个类似的集群。集群 A 被一百多名工程师日常用于研究与开发。一个典型任务由人类用户发起，运行数小时。它读取几 MB 到几 TB 的数据，对数据做变换或分析，并把结果写回集群。集群 B 主要用于生产数据处理。其任务持续时间长得多，持续生成并处理多 TB 的数据集，只需偶尔的人工介入。在两种情况下，一个"任务"都由分布在许多机器上的许多进程组成，同时读写许多文件。
{{% /bilingual %}}

**表 2 重绘（两个 GFS 集群的特征）**

| 项目 | 集群 A | 集群 B |
|---|---|---|
| Chunkserver 数 | 342 | 227 |
| 可用磁盘空间 | 72 TB | 180 TB |
| 已用磁盘空间 | 55 TB | 155 TB |
| 文件数 | 735 k | 737 k |
| 已删文件数（dead files） | 22 k | 232 k |
| chunk 数 | 992 k | 1550 k |
| chunkserver 上的元数据 | 13 GB | 21 GB |
| master 上的元数据 | 48 MB | 60 MB |

#### 6.2.1 Storage · 存储

{{% bilingual %}}
As shown by the first five entries in the table, both clusters have hundreds of chunkservers, support many TBs of disk space, and are fairly but not completely full. "Used space" includes all chunk replicas. Virtually all files are replicated three times. Therefore, the clusters store 18 TB and 52 TB of file data respectively.
<!--col-->
如表 2 前五行所示，两个集群都有数百台 chunkserver，支持许多 TB 磁盘空间，占用率较高但未满。"已用空间"包含所有 chunk 副本。几乎所有文件都复制三份，因此两个集群分别存储了 18 TB 和 52 TB 的文件数据。
{{% /bilingual %}}

{{% bilingual %}}
The two clusters have similar numbers of files, though B has a larger proportion of dead files, namely files which were deleted or replaced by a new version but whose storage have not yet been reclaimed. It also has more chunks because its files tend to be larger.
<!--col-->
两个集群的文件数相近，不过 B 中已删文件（即被删除或已被新版本替换、但存储尚未回收的文件）的比例更高。它的 chunk 也更多，因为其文件往往更大。
{{% /bilingual %}}

#### 6.2.2 Metadata · 元数据

{{% bilingual %}}
The chunkservers in aggregate store tens of GBs of metadata, mostly the checksums for 64 KB blocks of user data. The only other metadata kept at the chunkservers is the chunk version number discussed in Section 4.5.
<!--col-->
chunkserver 合计存储了数十 GB 的元数据，其中大部分是用户数据按 64 KB 分块的校验和。chunkserver 上保留的唯一其他元数据是 4.5 节讨论的 chunk 版本号。
{{% /bilingual %}}

{{% bilingual %}}
The metadata kept at the master is much smaller, only tens of MBs, or about 100 bytes per file on average. This agrees with our assumption that the size of the master's memory does not limit the system's capacity in practice. Most of the per-file metadata is the file names stored in a prefix-compressed form. Other metadata includes file ownership and permissions, mapping from files to chunks, and each chunk's current version. In addition, for each chunk we store the current replica locations and a reference count for implementing copy-on-write.
<!--col-->
master 上保存的元数据则小得多，只有数十 MB，平均每个文件约 100 字节。这与我们的假设一致：master 的内存大小在实践中并不限制系统容量。每个文件的元数据大部分是以前缀压缩形式存储的文件名。其他元数据包括文件属主与权限、文件到 chunk 的映射，以及每个 chunk 的当前版本。此外，对每个 chunk 我们还要存当前的副本位置，以及为实现写时复制而设的引用计数。
{{% /bilingual %}}

{{% bilingual %}}
Each individual server, both chunkservers and the master, has only 50 to 100 MB of metadata. Therefore recovery is fast: it takes only a few seconds to read this metadata from disk before the server is able to answer queries. However, the master is somewhat hobbled for a period - typically 30 to 60 seconds - until it has fetched chunk location information from all chunkservers.
<!--col-->
每台单独的服务器（无论 chunkserver 还是 master）只有 50 到 100 MB 元数据。因此恢复很快：从磁盘读取这些元数据只需几秒，之后服务器就能应答查询。不过 master 会在一段时间内处于半瘫状态（通常是 30 到 60 秒），直到它从所有 chunkserver 取回 chunk 位置信息。
{{% /bilingual %}}

#### 6.2.3 Read and Write Rates · 读写速率

{{% bilingual %}}
Table 3 shows read and write rates for various time periods. Both clusters had been up for about one week when these measurements were taken. (The clusters had been restarted recently to upgrade to a new version of GFS.)
<!--col-->
表 3 给出不同时间粒度下的读写速率。测量时两个集群都已运行约一周。（这些集群为了升级到新版 GFS 于此前不久重启过。）
{{% /bilingual %}}

**表 3 重绘（两个 GFS 集群的性能指标）**

| 指标 | 集群 A | 集群 B |
|---|---|---|
| 读速率（最近 1 分钟） | 583 MB/s | 380 MB/s |
| 读速率（最近 1 小时） | 562 MB/s | 384 MB/s |
| 读速率（自重启以来） | 589 MB/s | 49 MB/s |
| 写速率（最近 1 分钟） | 1 MB/s | 101 MB/s |
| 写速率（最近 1 小时） | 2 MB/s | 117 MB/s |
| 写速率（自重启以来） | 25 MB/s | 13 MB/s |
| master 操作数（最近 1 分钟） | 325 Ops/s | 533 Ops/s |
| master 操作数（最近 1 小时） | 381 Ops/s | 518 Ops/s |
| master 操作数（自重启以来） | 202 Ops/s | 347 Ops/s |

{{% bilingual %}}
The average write rate was less than 30 MB/s since the restart. When we took these measurements, B was in the middle of a burst of write activity generating about 100 MB/s of data, which produced a 300 MB/s network load because writes are propagated to three replicas.
<!--col-->
自重启以来平均写速率低于 30 MB/s。测量时 B 正处于一波写活动的高峰，产生约 100 MB/s 的数据，由于写要传播到三个副本，这带来了 300 MB/s 的网络负载。
{{% /bilingual %}}

{{% bilingual %}}
The read rates were much higher than the write rates. The total workload consists of more reads than writes as we have assumed. Both clusters were in the middle of heavy read activity. In particular, A had been sustaining a read rate of 580 MB/s for the preceding week. Its network configuration can support 750 MB/s, so it was using its resources efficiently. Cluster B can support peak read rates of 1300 MB/s, but its applications were using just 380 MB/s.
<!--col-->
读速率远高于写速率。如我们假设的那样，总负载中读多于写。两个集群都处于大量读活动之中。特别是 A 在此前一周一直维持 580 MB/s 的读速率。它的网络配置可支持 750 MB/s，因此资源利用是高效的。集群 B 可支持 1300 MB/s 的峰值读速率，但其应用只用到 380 MB/s。
{{% /bilingual %}}

#### 6.2.4 Master Load · master 负载

{{% bilingual %}}
Table 3 also shows that the rate of operations sent to the master was around 200 to 500 operations per second. The master can easily keep up with this rate, and therefore is not a bottleneck for these workloads.
<!--col-->
表 3 还显示，发送给 master 的操作速率在每秒 200 到 500 次左右。master 可以轻松跟上这一速率，因此对这些负载而言它不是瓶颈。
{{% /bilingual %}}

{{% bilingual %}}
In an earlier version of GFS, the master was occasionally a bottleneck for some workloads. It spent most of its time sequentially scanning through large directories (which contained hundreds of thousands of files) looking for particular files. We have since changed the master data structures to allow efficient binary searches through the namespace. It can now easily support many thousands of file accesses per second. If necessary, we could speed it up further by placing name lookup caches in front of the namespace data structures.
<!--col-->
在早期版本的 GFS 中，master 偶尔会成为某些负载的瓶颈：它把大部分时间花在顺序扫描大目录（包含数十万个文件）以查找特定文件上。此后我们修改了 master 的数据结构，使命名空间支持高效的二分查找。现在它可以轻松支撑每秒数千次文件访问。如有必要，我们还可以在命名空间数据结构前面加一层名字查找缓存来进一步提速。
{{% /bilingual %}}

#### 6.2.5 Recovery Time · 恢复时间

{{% bilingual %}}
After a chunkserver fails, some chunks will become underreplicated and must be cloned to restore their replication levels. The time it takes to restore all such chunks depends on the amount of resources. In one experiment, we killed a single chunkserver in cluster B. The chunkserver had about 15,000 chunks containing 600 GB of data. To limit the impact on running applications and provide leeway for scheduling decisions, our default parameters limit this cluster to 91 concurrent clonings (40% of the number of chunkservers) where each clone operation is allowed to consume at most 6.25 MB/s (50 Mbps). All chunks were restored in 23.2 minutes, at an effective replication rate of 440 MB/s.
<!--col-->
某个 chunkserver 失效后，部分 chunk 会副本不足，必须克隆以恢复其副本数。恢复所有这类 chunk 所需的时间取决于可用资源量。在一次实验中，我们杀掉集群 B 中的一台 chunkserver。该 chunkserver 约持有 15,000 个 chunk、共 600 GB 数据。为限制对运行中应用的影响并为调度决策留出余地，我们的默认参数把该集群的并发克隆数限制为 91（即 chunkserver 数量的 40%），其中每个克隆操作最多可消耗 6.25 MB/s（50 Mbps）。所有 chunk 在 23.2 分钟内恢复完毕，有效复制速率达 440 MB/s。
{{% /bilingual %}}

{{% bilingual %}}
In another experiment, we killed two chunkservers each with roughly 16,000 chunks and 660 GB of data. This double failure reduced 266 chunks to having a single replica. These 266 chunks were cloned at a higher priority, and were all restored to at least 2x replication within 2 minutes, thus putting the cluster in a state where it could tolerate another chunkserver failure without data loss.
<!--col-->
在另一次实验中，我们杀掉了两台 chunkserver，每台约有 16,000 个 chunk、660 GB 数据。这次双重故障使 266 个 chunk 只剩单副本。这 266 个 chunk 以更高优先级被克隆，并在 2 分钟内全部恢复到至少 2 副本，从而使集群进入可以再容忍一台 chunkserver 失效而不丢数据的状态。
{{% /bilingual %}}

### 6.3 Workload Breakdown · 负载分解

{{% bilingual %}}
In this section, we present a detailed breakdown of the workloads on two GFS clusters comparable but not identical to those in Section 6.2. Cluster X is for research and development while cluster Y is for production data processing.
<!--col-->
本节详细分解两个 GFS 集群的负载，它们与 6.2 节的集群可比但并不相同。集群 X 用于研究与开发，集群 Y 用于生产数据处理。
{{% /bilingual %}}

#### 6.3.1 Methodology and Caveats · 统计方法与注意事项

{{% bilingual %}}
These results include only client originated requests so that they reflect the workload generated by our applications for the file system as a whole. They do not include interserver requests to carry out client requests or internal background activities, such as forwarded writes or rebalancing.
<!--col-->
这些结果只包含客户端发起的请求，以便反映我们的应用对整个文件系统施加的负载。它们不包含为执行客户端请求而产生的服务器间请求，也不包含内部后台活动，例如转发的写或再均衡。
{{% /bilingual %}}

{{% bilingual %}}
Statistics on I/O operations are based on information heuristically reconstructed from actual RPC requests logged by GFS servers. For example, GFS client code may break a read into multiple RPCs to increase parallelism, from which we infer the original read. Since our access patterns are highly stylized, we expect any error to be in the noise. Explicit logging by applications might have provided slightly more accurate data, but it is logistically impossible to recompile and restart thousands of running clients to do so and cumbersome to collect the results from as many machines.
<!--col-->
I/O 操作的统计基于从 GFS 服务器实际记录的 RPC 请求中启发式重建出的信息。例如，GFS 客户端代码可能为了增加并行度而把一次读拆成多个 RPC，我们据此反推原始的读。由于我们的访问模式高度程式化，我们预计误差都在噪声范围内。由应用显式打印日志也许能提供略更准确的数据，但为此重新编译并重启数千个正在运行的客户端在操作上不可能，从这么多机器上收集结果也极为繁琐。
{{% /bilingual %}}

{{% bilingual %}}
One should be careful not to overly generalize from our workload. Since Google completely controls both GFS and its applications, the applications tend to be tuned for GFS, and conversely GFS is designed for these applications. Such mutual influence may also exist between general applications and file systems, but the effect is likely more pronounced in our case.
<!--col-->
不应当从我们的负载过度外推。由于 Google 完全掌控 GFS 及其应用，应用往往会针对 GFS 调优，反之 GFS 也是为这些应用设计的。通用应用与文件系统之间也可能存在这种相互影响，但在我们这里这种效应大概更为显著。
{{% /bilingual %}}

**表 4 重绘（按操作大小分解的操作数占比 %）**

对读而言，"大小"指实际读出并传输的数据量，而非请求的量。

| 操作大小 | 读 X | 读 Y | 写 X | 写 Y | 记录追加 X | 记录追加 Y |
|---|---|---|---|---|---|---|
| 0K | 0.4 | 2.6 | 0 | 0 | 0 | 0 |
| 1B..1K | 0.1 | 4.1 | 6.6 | 4.9 | 0.2 | 9.2 |
| 1K..8K | 65.2 | 38.5 | 0.4 | 1.0 | 18.9 | 15.2 |
| 8K..64K | 29.9 | 45.1 | 17.8 | 43.0 | 78.0 | 2.8 |
| 64K..128K | 0.1 | 0.7 | 2.3 | 1.9 | < .1 | 4.3 |
| 128K..256K | 0.2 | 0.3 | 31.6 | 0.4 | < .1 | 10.6 |
| 256K..512K | 0.1 | 0.1 | 4.2 | 7.7 | < .1 | 31.2 |
| 512K..1M | 3.9 | 6.9 | 35.5 | 28.7 | 2.2 | 25.5 |
| 1M..inf | 0.1 | 1.8 | 1.5 | 12.3 | 0.7 | 2.2 |

#### 6.3.2 Chunkserver Workload · chunkserver 负载

{{% bilingual %}}
Table 4 shows the distribution of operations by size. Read sizes exhibit a bimodal distribution. The small reads (under 64 KB) come from seek-intensive clients that look up small pieces of data within huge files. The large reads (over 512 KB) come from long sequential reads through entire files.
<!--col-->
表 4 给出操作按大小的分布。读的大小呈双峰分布：小读（小于 64 KB）来自那些在巨大文件中定位小块数据的、以寻道为主的客户端；大读（大于 512 KB）来自贯穿整个文件的长时间顺序读。
{{% /bilingual %}}

{{% bilingual %}}
A significant number of reads return no data at all in cluster Y. Our applications, especially those in the production systems, often use files as producer-consumer queues. Producers append concurrently to a file while a consumer reads the end of file. Occasionally, no data is returned when the consumer outpaces the producers. Cluster X shows this less often because it is usually used for short-lived data analysis tasks rather than long-lived distributed applications.
<!--col-->
在集群 Y 中，相当数量的读完全没有返回数据。我们的应用——尤其是生产系统中的那些——常把文件用作生产者—消费者队列：生产者并发向文件追加，同时有消费者读取文件末尾。当消费者跑得比生产者快时，偶尔就会没有数据返回。集群 X 较少出现这种情况，因为它通常用于短命的数据分析任务，而非长命的分布式应用。
{{% /bilingual %}}

{{% bilingual %}}
Write sizes also exhibit a bimodal distribution. The large writes (over 256 KB) typically result from significant buffering within the writers. Writers that buffer less data, checkpoint or synchronize more often, or simply generate less data account for the smaller writes (under 64 KB).
<!--col-->
写的大小也呈双峰分布。大写（大于 256 KB）通常源自写者内部做了较多缓冲；而缓冲更少、更频繁地做检查点或同步、或本身产生数据更少的写者，则对应小写（小于 64 KB）。
{{% /bilingual %}}

{{% bilingual %}}
As for record appends, cluster Y sees a much higher percentage of large record appends than cluster X does because our production systems, which use cluster Y, are more aggressively tuned for GFS.
<!--col-->
至于记录追加，集群 Y 中大记录追加的占比远高于集群 X，因为使用集群 Y 的生产系统针对 GFS 做了更激进的调优。
{{% /bilingual %}}

{{% bilingual %}}
Table 5 shows the total amount of data transferred in operations of various sizes. For all kinds of operations, the larger operations (over 256 KB) generally account for most of the bytes transferred. Small reads (under 64 KB) do transfer a small but significant portion of the read data because of the random seek workload.
<!--col-->
表 5 给出不同大小的操作所传输的数据总量。对所有类型的操作而言，较大的操作（大于 256 KB）通常占所传输字节的大部分。由于存在随机寻道类负载，小读（小于 64 KB）确实也传输了读数据中一小块但不容忽视的部分。
{{% /bilingual %}}

**表 5 重绘（按操作大小分解的传输字节占比 %）**

对读而言，"大小"指实际读出并传输的数据量，而非请求的量。两者可能不同——如果读尝试越过文件末尾，而按设计这在我们的负载中并不罕见。

| 操作大小 | 读 X | 读 Y | 写 X | 写 Y | 记录追加 X | 记录追加 Y |
|---|---|---|---|---|---|---|
| 1B..1K | < .1 | < .1 | < .1 | < .1 | < .1 | < .1 |
| 1K..8K | 13.8 | 3.9 | < .1 | < .1 | < .1 | 0.1 |
| 8K..64K | 11.4 | 9.3 | 2.4 | 5.9 | 2.3 | 0.3 |
| 64K..128K | 0.3 | 0.7 | 0.3 | 0.3 | 22.7 | 1.2 |
| 128K..256K | 0.8 | 0.6 | 16.5 | 0.2 | < .1 | 5.8 |
| 256K..512K | 1.4 | 0.3 | 3.4 | 7.7 | < .1 | 38.4 |
| 512K..1M | 65.9 | 55.1 | 74.1 | 58.0 | 0.1 | 46.8 |
| 1M..inf | 6.4 | 30.1 | 3.3 | 28.0 | 53.9 | 7.4 |

#### 6.3.3 Appends versus Writes · 追加与写的对比

{{% bilingual %}}
Record appends are heavily used especially in our production systems. For cluster X, the ratio of writes to record appends is 108:1 by bytes transferred and 8:1 by operation counts. For cluster Y, used by the production systems, the ratios are 3.7:1 and 2.5:1 respectively. Moreover, these ratios suggest that for both clusters record appends tend to be larger than writes. For cluster X, however, the overall usage of record append during the measured period is fairly low and so the results are likely skewed by one or two applications with particular buffer size choices.
<!--col-->
记录追加被大量使用，在生产系统中尤其如此。对集群 X 而言，写与记录追加的比例按传输字节计为 108:1，按操作次数计为 8:1。对生产系统所使用的集群 Y，这两个比例分别为 3.7:1 和 2.5:1。此外，这些比例说明在两个集群中记录追加都倾向于比写更大。不过在集群 X 中，记录追加在测量期间的总使用量相当低，因此结果很可能被一两个缓冲大小选择特殊的应用带偏。
{{% /bilingual %}}

{{% bilingual %}}
As expected, our data mutation workload is dominated by appending rather than overwriting. We measured the amount of data overwritten on primary replicas. This approximates the case where a client deliberately overwrites previous written data rather than appends new data. For cluster X, overwriting accounts for under 0.0001% of bytes mutated and under 0.0003% of mutation operations. For cluster Y, the ratios are both 0.05%. Although this is minute, it is still higher than we expected. It turns out that most of these overwrites came from client retries due to errors or timeouts. They are not part of the workload per se but a consequence of the retry mechanism.
<!--col-->
如预期的那样，我们的数据变更负载以追加为主，而非覆盖。我们测量了主副本上被覆盖的数据量，这大致对应"客户端刻意覆盖此前写过的数据而不是追加新数据"的情形。对集群 X，覆盖占被变更字节的不到 0.0001%、占变更操作数的不到 0.0003%。对集群 Y，两个比例都是 0.05%。虽然极小，仍高于我们的预期。结果发现，这些覆盖大多来自客户端因错误或超时而做的重试——它们本身并不属于负载的一部分，而是重试机制的产物。
{{% /bilingual %}}

#### 6.3.4 Master Workload · master 负载

{{% bilingual %}}
Table 6 shows the breakdown by type of requests to the master. Most requests ask for chunk locations (FindLocation) for reads and lease holder information (FindLeaseLocker) for data mutations.
<!--col-->
表 6 给出发往 master 的请求按类型的分解。大多数请求是读时询问 chunk 位置（FindLocation），以及数据变更时询问租约持有者信息（原文此处写作 FindLeaseLocker，表 6 中该操作名为 FindLeaseHolder）。
{{% /bilingual %}}

**表 6 重绘（master 请求按类型分解 %）**

| 请求类型 | 集群 X | 集群 Y |
|---|---|---|
| Open | 26.1 | 16.3 |
| Delete | 0.7 | 1.5 |
| FindLocation | 64.3 | 65.8 |
| FindLeaseHolder | 7.8 | 13.4 |
| FindMatchingFiles | 0.6 | 2.2 |
| 其他所有合并 | 0.5 | 0.8 |

{{% bilingual %}}
Clusters X and Y see significantly different numbers of Delete requests because cluster Y stores production data sets that are regularly regenerated and replaced with newer versions. Some of this difference is further hidden in the difference in Open requests because an old version of a file may be implicitly deleted by being opened for write from scratch (mode "w" in Unix open terminology).
<!--col-->
集群 X 和 Y 的 Delete 请求数量差异显著，因为集群 Y 存放的是会被定期重新生成、替换为新版本的生产数据集。这一差异还有一部分隐藏在 Open 请求的差异中，因为一个文件的旧版本可能因被以从头写入的方式打开而隐式删除（用 Unix 的 open 术语说就是模式 "w"）。
{{% /bilingual %}}

{{% bilingual %}}
FindMatchingFiles is a pattern matching request that supports "ls" and similar file system operations. Unlike other requests for the master, it may process a large part of the namespace and so may be expensive. Cluster Y sees it much more often because automated data processing tasks tend to examine parts of the file system to understand global application state. In contrast, cluster X's applications are under more explicit user control and usually know the names of all needed files in advance.
<!--col-->
FindMatchingFiles 是支持 "ls" 及类似文件系统操作的模式匹配请求。与发给 master 的其他请求不同，它可能处理命名空间的一大部分，因而开销可能很大。集群 Y 中它出现得频繁得多，因为自动化的数据处理任务往往要查看文件系统的部分内容以理解应用的全局状态。相比之下，集群 X 的应用受用户更明确的控制，通常事先就知道所需全部文件的名字。
{{% /bilingual %}}

## 7 Experiences · 实践经验

{{% bilingual %}}
In the process of building and deploying GFS, we have experienced a variety of issues, some operational and some technical.
<!--col-->
在构建和部署 GFS 的过程中，我们遇到过各种各样的问题，有些是运维层面的，有些是技术层面的。
{{% /bilingual %}}

{{% bilingual %}}
Initially, GFS was conceived as the backend file system for our production systems. Over time, the usage evolved to include research and development tasks. It started with little support for things like permissions and quotas but now includes rudimentary forms of these. While production systems are well disciplined and controlled, users sometimes are not. More infrastructure is required to keep users from interfering with one another.
<!--col-->
最初，GFS 是被设想为生产系统的后端文件系统。随着时间推移，其用途逐渐扩展到研究与开发任务。它起初对权限、配额这类东西几乎没有支持，现在已包含这些功能的雏形。生产系统本身很有纪律、受控良好，但用户有时并非如此。需要有更多基础设施来防止用户之间互相干扰。
{{% /bilingual %}}

{{% bilingual %}}
Some of our biggest problems were disk and Linux related. Many of our disks claimed to the Linux driver that they supported a range of IDE protocol versions but in fact responded reliably only to the more recent ones. Since the protocol versions are very similar, these drives mostly worked, but occasionally the mismatches would cause the drive and the kernel to disagree about the drive's state. This would corrupt data silently due to problems in the kernel. This problem motivated our use of checksums to detect data corruption, while concurrently we modified the kernel to handle these protocol mismatches.
<!--col-->
我们最大的一些问题与磁盘和 Linux 有关。我们许多磁盘对 Linux 驱动声称支持一系列 IDE 协议版本，但实际上只对较新的那些版本可靠响应。由于各协议版本非常相似，这些盘大多能用，但偶尔这种不匹配会让磁盘与内核对磁盘状态的判断出现分歧，进而因内核中的问题静默地损坏数据。这个问题促使我们使用校验和来检测数据损坏，同时我们也修改了内核来处理这些协议不匹配。
{{% /bilingual %}}

{{% bilingual %}}
Earlier we had some problems with Linux 2.2 kernels due to the cost of fsync(). Its cost is proportional to the size of the file rather than the size of the modified portion. This was a problem for our large operation logs especially before we implemented checkpointing. We worked around this for a time by using synchronous writes and eventually migrated to Linux 2.4.
<!--col-->
更早的时候，由于 `fsync()` 的开销，我们在 Linux 2.2 内核上遇到过一些问题。它的开销与文件大小成正比，而不是与被修改部分的大小成正比。这对我们庞大的操作日志来说是个问题，尤其是在实现检查点之前。我们一度用同步写来绕过它，最终迁移到了 Linux 2.4。
{{% /bilingual %}}

{{% bilingual %}}
Another Linux problem was a single reader-writer lock which any thread in an address space must hold when it pages in from disk (reader lock) or modifies the address space in an mmap() call (writer lock). We saw transient timeouts in our system under light load and looked hard for resource bottlenecks or sporadic hardware failures. Eventually, we found that this single lock blocked the primary network thread from mapping new data into memory while the disk threads were paging in previously mapped data. Since we are mainly limited by the network interface rather than by memory copy bandwidth, we worked around this by replacing mmap() with pread() at the cost of an extra copy.
<!--col-->
另一个 Linux 问题是单把读写锁：地址空间中的任何线程在从磁盘换入页面（读锁）或在 `mmap()` 调用中修改地址空间（写锁）时都必须持有它。我们在轻负载下观察到系统出现瞬时超时，并费力排查资源瓶颈或偶发硬件故障。最终我们发现，在磁盘线程换入此前已映射的数据时，这把唯一的锁阻止了主网络线程把新数据映射进内存。由于我们的瓶颈主要在网卡而不是内存拷贝带宽，我们用在 `pread()` 替换 `mmap()` 绕过了它，代价是多一次拷贝。
{{% /bilingual %}}

{{% bilingual %}}
Despite occasional problems, the availability of Linux code has helped us time and again to explore and understand system behavior. When appropriate, we improve the kernel and share the changes with the open source community.
<!--col-->
尽管偶有问题，Linux 代码的可得性一次又一次地帮助我们探查和理解系统行为。在合适的时候，我们会改进内核并把改动分享给开源社区。
{{% /bilingual %}}

## 8 Related Work · 相关工作

{{% bilingual %}}
Like other large distributed file systems such as AFS [5], GFS provides a location independent namespace which enables data to be moved transparently for load balance or fault tolerance. Unlike AFS, GFS spreads a file's data across storage servers in a way more akin to xFS [1] and Swift [3] in order to deliver aggregate performance and increased fault tolerance.
<!--col-->
与其他大型分布式文件系统（如 AFS [5]）一样，GFS 提供位置无关的命名空间，使数据可以为负载均衡或容错而透明地迁移。与 AFS 不同的是，GFS 把文件数据分散到多台存储服务器上，方式更接近 xFS [1] 和 Swift [3]，以提供聚合性能并提高容错能力。
{{% /bilingual %}}

{{% bilingual %}}
As disks are relatively cheap and replication is simpler than more sophisticated RAID [9] approaches, GFS currently uses only replication for redundancy and so consumes more raw storage than xFS or Swift.
<!--col-->
由于磁盘相对便宜，而复制比更精细的 RAID [9] 方案简单，GFS 目前只用复制来做冗余，因此比 xFS 或 Swift 消耗更多的裸存储。
{{% /bilingual %}}

{{% bilingual %}}
In contrast to systems like AFS, xFS, Frangipani [12], and Intermezzo [6], GFS does not provide any caching below the file system interface. Our target workloads have little reuse within a single application run because they either stream through a large data set or randomly seek within it and read small amounts of data each time.
<!--col-->
与 AFS、xFS、Frangipani [12]、Intermezzo [6] 这类系统相比，GFS 在文件系统接口之下不提供任何缓存。我们的目标负载在单次应用运行中的复用率很低，因为它们要么流式扫过一个大得多的数据集，要么在数据集内随机定位、每次只读少量数据。
{{% /bilingual %}}

{{% bilingual %}}
Some distributed file systems like Frangipani, xFS, Minnesota's GFS [11] and GPFS [10] remove the centralized server and rely on distributed algorithms for consistency and management. We opt for the centralized approach in order to simplify the design, increase its reliability, and gain flexibility. In particular, a centralized master makes it much easier to implement sophisticated chunk placement and replication policies since the master already has most of the relevant information and controls how it changes. We address fault tolerance by keeping the master state small and fully replicated on other machines. Scalability and high availability (for reads) are currently provided by our shadow master mechanism. Updates to the master state are made persistent by appending to a write-ahead log. Therefore we could adapt a primary-copy scheme like the one in Harp [7] to provide high availability with stronger consistency guarantees than our current scheme.
<!--col-->
有些分布式文件系统（如 Frangipani、xFS、明尼苏达大学的 GFS [11] 和 GPFS [10]）去掉了中心化服务器，改用分布式算法来做一致性和管理。我们选择中心化方案，是为了简化设计、提高可靠性并获得灵活性。特别是，中心化的 master 让实现精细的 chunk 放置与复制策略容易得多，因为 master 本来就掌握大部分相关信息，并控制着这些信息如何变化。我们通过让 master 状态保持小体量、并完整复制到其他机器来解决容错。可扩展性和（读的）高可用目前由我们的影子 master 机制提供。对 master 状态的更新通过追加到预写日志来持久化。因此我们可以采用类似 Harp [7] 中主副本（primary-copy）方案的做法，来提供比当前方案一致性保证更强的高可用。
{{% /bilingual %}}

{{% bilingual %}}
We are addressing a problem similar to Lustre [8] in terms of delivering aggregate performance to a large number of clients. However, we have simplified the problem significantly by focusing on the needs of our applications rather than building a POSIX-compliant file system. Additionally, GFS assumes large number of unreliable components and so fault tolerance is central to our design.
<!--col-->
在向大量客户端提供聚合性能这一点上，我们处理的问题与 Lustre [8] 类似。但我们通过聚焦应用自身需求、而不是构建一个 POSIX 兼容的文件系统，大幅简化了这个问题。此外，GFS 假定存在大量不可靠组件，因此容错是我们设计的核心。
{{% /bilingual %}}

{{% bilingual %}}
GFS most closely resembles the NASD architecture [4]. While the NASD architecture is based on network-attached disk drives, GFS uses commodity machines as chunkservers, as done in the NASD prototype. Unlike the NASD work, our chunkservers use lazily allocated fixed-size chunks rather than variable-length objects. Additionally, GFS implements features such as rebalancing, replication, and recovery that are required in a production environment.
<!--col-->
GFS 最接近 NASD 架构 [4]。NASD 架构基于网络附加磁盘驱动器，而 GFS 像 NASD 原型那样用通用机器充当 chunkserver。与 NASD 的工作不同，我们的 chunkserver 使用惰性分配的固定大小 chunk，而不是变长对象。此外，GFS 实现了生产环境中必需的再均衡、复制和恢复等功能。
{{% /bilingual %}}

{{% bilingual %}}
Unlike Minnesota's GFS and NASD, we do not seek to alter the model of the storage device. We focus on addressing day-to-day data processing needs for complicated distributed systems with existing commodity components.
<!--col-->
与明尼苏达大学的 GFS 和 NASD 不同，我们并不试图改变存储设备的模型。我们关注的是用现有的通用组件，满足复杂分布式系统日常的数据处理需求。
{{% /bilingual %}}

{{% bilingual %}}
The producer-consumer queues enabled by atomic record appends address a similar problem as the distributed queues in River [2]. While River uses memory-based queues distributed across machines and careful data flow control, GFS uses a persistent file that can be appended to concurrently by many producers. The River model supports m-to-n distributed queues but lacks the fault tolerance that comes with persistent storage, while GFS only supports m-to-1 queues efficiently. Multiple consumers can read the same file, but they must coordinate to partition the incoming load.
<!--col-->
由原子记录追加所支持的生产者—消费者队列，处理的问题与 River [2] 中的分布式队列类似。River 使用分布在多台机器上的基于内存的队列，并有细致的数据流控制；而 GFS 使用一个持久化文件，可以被许多生产者并发追加。River 模型支持 m 对 n 的分布式队列，但缺少持久化存储带来的容错能力；而 GFS 只能高效支持 m 对 1 的队列。多个消费者可以读同一个文件，但它们必须自行协调以分摊到来的负载。
{{% /bilingual %}}

## 9 Conclusions · 结论

{{% bilingual %}}
The Google File System demonstrates the qualities essential for supporting large-scale data processing workloads on commodity hardware. While some design decisions are specific to our unique setting, many may apply to data processing tasks of a similar magnitude and cost consciousness.
<!--col-->
Google File System 展示了在通用硬件上支撑大规模数据处理负载所必需的品质。虽然有些设计决策是针对我们特定场景的，但许多决策也适用于同等规模、同样在意成本的数据处理任务。
{{% /bilingual %}}

{{% bilingual %}}
We started by reexamining traditional file system assumptions in light of our current and anticipated application workloads and technological environment. Our observations have led to radically different points in the design space. We treat component failures as the norm rather than the exception, optimize for huge files that are mostly appended to (perhaps concurrently) and then read (usually sequentially), and both extend and relax the standard file system interface to improve the overall system.
<!--col-->
我们从重新审视传统文件系统的假设出发，依据的是当前与可预见的应用负载和技术环境。这些观察把我们引向设计空间中截然不同的点。我们把组件故障视为常态而非例外；针对"主要被追加（可能是并发追加）、随后被读取（通常是顺序读取）"的巨大文件做优化；并同时扩展和放宽标准文件系统接口，以改进整个系统。
{{% /bilingual %}}

{{% bilingual %}}
Our system provides fault tolerance by constant monitoring, replicating crucial data, and fast and automatic recovery. Chunk replication allows us to tolerate chunkserver failures. The frequency of these failures motivated a novel online repair mechanism that regularly and transparently repairs the damage and compensates for lost replicas as soon as possible. Additionally, we use checksumming to detect data corruption at the disk or IDE subsystem level, which becomes all too common given the number of disks in the system.
<!--col-->
我们的系统通过持续监控、复制关键数据和快速自动恢复来提供容错。Chunk 复制让我们能容忍 chunkserver 故障。这类故障之频繁，促使我们设计了一套新颖的在线修复机制：定期、透明地修复损坏，并尽快补齐丢失的副本。此外，我们用校验和来检测发生在磁盘或 IDE 子系统层面的数据损坏——考虑到系统中磁盘的数量，这类损坏实在过于常见。
{{% /bilingual %}}

{{% bilingual %}}
Our design delivers high aggregate throughput to many concurrent readers and writers performing a variety of tasks. We achieve this by separating file system control, which passes through the master, from data transfer, which passes directly between chunkservers and clients. Master involvement in common operations is minimized by a large chunk size and by chunk leases, which delegates authority to primary replicas in data mutations. This makes possible a simple, centralized master that does not become a bottleneck. We believe that improvements in our networking stack will lift the current limitation on the write throughput seen by an individual client.
<!--col-->
我们的设计能为大量并发执行各类任务的读者和写者提供很高的聚合吞吐。做到这一点的方式，是把经过 master 的文件系统控制与在 chunkserver 和客户端之间直接进行的数据传输分离开。大 chunk 尺寸和 chunk 租约把数据变更中的权威下放给主副本，从而把 master 在常见操作中的参与度降到最低。这使得一个简单、中心化却不会成为瓶颈的 master 成为可能。我们相信，网络协议栈的改进将解除目前单个客户端所能达到的写吞吐上限。
{{% /bilingual %}}

{{% bilingual %}}
GFS has successfully met our storage needs and is widely used within Google as the storage platform for research and development as well as production data processing. It is an important tool that enables us to continue to innovate and attack problems on the scale of the entire web.
<!--col-->
GFS 成功地满足了我们的存储需求，并在 Google 内部被广泛用作研发与生产数据处理的存储平台。它是一件重要的工具，使我们能够持续创新，并解决整个互联网规模上的问题。
{{% /bilingual %}}

## Acknowledgments · 致谢

{{% bilingual %}}
We wish to thank the following people for their contributions to the system or the paper. Brain Bershad (our shepherd) and the anonymous reviewers gave us valuable comments and suggestions. Anurag Acharya, Jeff Dean, and David desJardins contributed to the early design. Fay Chang worked on comparison of replicas across chunkservers. Guy Edjlali worked on storage quota. Markus Gutschke worked on a testing framework and security enhancements. David Kramer worked on performance enhancements. Fay Chang, Urs Hoelzle, Max Ibel, Sharon Perl, Rob Pike, and Debby Wallach commented on earlier drafts of the paper. Many of our colleagues at Google bravely trusted their data to a new file system and gave us useful feedback. Yoshka helped with early testing.
<!--col-->
我们要感谢以下各位对系统或本文做出的贡献。Brain Bershad（我们的 shepherd）以及匿名审稿人给出了宝贵的意见与建议。Anurag Acharya、Jeff Dean 和 David desJardins 参与了早期设计。Fay Chang 负责跨 chunkserver 的副本比较工作。Guy Edjlali 负责存储配额。Markus Gutschke 负责测试框架和安全增强。David Kramer 负责性能优化。Fay Chang、Urs Hoelzle、Max Ibel、Sharon Perl、Rob Pike 和 Debby Wallach 对本文的早期草稿提出了意见。我们在 Google 的许多同事勇敢地把自己的数据托付给一套新文件系统，并给了我们有用的反馈。Yoshka 协助了早期测试。
{{% /bilingual %}}

## References · 参考文献

> 以下文献按原文编号列出，正文中的 [n] 对应此处。文献条目保留原文格式，不做翻译。

[1] Thomas Anderson, Michael Dahlin, Jeanna Neefe, David Patterson, Drew Roselli, and Randolph Wang. Serverless network file systems. In Proceedings of the 15th ACM Symposium on Operating System Principles, pages 109-126, Copper Mountain Resort, Colorado, December 1995.

[2] Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, and Kathy Yelick. Cluster I/O with River: Making the fast case common. In Proceedings of the Sixth Workshop on Input/Output in Parallel and Distributed Systems (IOPADS '99), pages 10-22, Atlanta, Georgia, May 1999.

[3] Luis-Felipe Cabrera and Darrell D. E. Long. Swift: Using distributed disk striping to provide high I/O data rates. Computer Systems, 4(4):405-436, 1991.

[4] Garth A. Gibson, David F. Nagle, Khalil Amiri, Jeff Butler, Fay W. Chang, Howard Gobioff, Charles Hardin, Erik Riedel, David Rochberg, and Jim Zelenka. A cost-effective, high-bandwidth storage architecture. In Proceedings of the 8th Architectural Support for Programming Languages and Operating Systems, pages 92-103, San Jose, California, October 1998.

[5] John Howard, Michael Kazar, Sherri Menees, David Nichols, Mahadev Satyanarayanan, Robert Sidebotham, and Michael West. Scale and performance in a distributed file system. ACM Transactions on Computer Systems, 6(1):51-81, February 1988.

[6] InterMezzo. http://www.inter-mezzo.org, 2003.

[7] Barbara Liskov, Sanjay Ghemawat, Robert Gruber, Paul Johnson, Liuba Shrira, and Michael Williams. Replication in the Harp file system. In 13th Symposium on Operating System Principles, pages 226-238, Pacific Grove, CA, October 1991.

[8] Lustre. http://www.lustre.org, 2003.

[9] David A. Patterson, Garth A. Gibson, and Randy H. Katz. A case for redundant arrays of inexpensive disks (RAID). In Proceedings of the 1988 ACM SIGMOD International Conference on Management of Data, pages 109-116, Chicago, Illinois, September 1988.

[10] Frank Schmuck and Roger Haskin. GPFS: A shared-disk file system for large computing clusters. In Proceedings of the First USENIX Conference on File and Storage Technologies, pages 231-244, Monterey, California, January 2002.

[11] Steven R. Soltis, Thomas M. Ruwart, and Matthew T. O'Keefe. The Global File System. In Proceedings of the Fifth NASA Goddard Space Flight Center Conference on Mass Storage Systems and Technologies, College Park, Maryland, September 1996.

[12] Chandramohan A. Thekkath, Timothy Mann, and Edward K. Lee. Frangipani: A scalable distributed file system. In Proceedings of the 16th ACM Symposium on Operating System Principles, pages 224-237, Saint-Malo, France, October 1997.
