---
title: "MapReduce 论文中英对照全文翻译（Simplified Data Processing on Large Clusters）"
date: 2014-05-18T10:00:00+08:00
url: /2014/05/mapreduce-paper-cn-en/
draft: false
tags: ["mapreduce", "google", "distributed", "paper", "big-data"]
categories: ["tech"]
description: "MapReduce 论文 Simplified Data Processing on Large Clusters 的完整中英对照翻译，共 8 节加附录 A，含编程模型与类型签名、执行流程的七个步骤、主节点数据结构、容错语义与原子提交、局部性优化、任务粒度、Straggler 备份执行，以及 grep 与 sort 的实测数据和索引系统重写的经验数据。"
---

这是 MapReduce 论文的完整中英对照翻译。原文 13 页，正文 8 节加附录 A，参考文献 18 条。

体例：每段先列英文原文（引用块），紧接中文译文。图与表格按原文内容重绘；专业术语保留英文并附中文，原文的引用编号 [n] 对应文末参考文献。

<!--more-->

## 论文信息

| 项目 | 内容 |
|------|------|
| 标题 | MapReduce: Simplified Data Processing on Large Clusters |
| 作者 | Jeffrey Dean, Sanjay Ghemawat |
| 机构 | Google, Inc. |
| 发表 | OSDI 2004（第 6 届 USENIX 操作系统设计与实现研讨会），2004 年 12 月，旧金山 |
| 篇幅 | 13 页，正文 8 节加附录 A，参考文献 18 条 |
| 核心机制 | Map/Reduce 函数接口、M 个 map 任务与 R 个 reduce 任务、主节点调度、原子提交、局部性优化、combiner、备份执行 |
| 实测规模 | 1800 台机器的集群，grep 处理约 1 TB 用时约 150 秒，sort 处理约 1 TB 用时 891 秒 |

---

## Abstract · 摘要

> MapReduce is a programming model and an associated implementation for processing and generating large data sets. Users specify a map function that processes a key/value pair to generate a set of intermediate key/value pairs, and a reduce function that merges all intermediate values associated with the same intermediate key. Many real world tasks are expressible in this model, as shown in the paper.

MapReduce 是一个编程模型，也是一套配套实现，用于处理和生成大数据集。用户指定一个 map 函数，把一对 key/value 处理成一组中间 key/value 对；再指定一个 reduce 函数，把同一个中间 key 关联的所有中间 value 合并起来。许多现实任务都能用这个模型表达，本文给出了若干例子。

> Programs written in this functional style are automatically parallelized and executed on a large cluster of commodity machines. The run-time system takes care of the details of partitioning the input data, scheduling the program's execution across a set of machines, handling machine failures, and managing the required inter-machine communication. This allows programmers without any experience with parallel and distributed systems to easily utilize the resources of a large distributed system.

用这种函数式风格写出的程序会被自动并行化，并在一个由廉价机器组成的大集群上执行。运行时系统负责处理这些细节：切分输入数据、把程序执行调度到一批机器上、处理机器故障，以及管理所需的机器间通信。这让完全没有并行和分布式系统经验的程序员也能轻松用上大型分布式系统的资源。

> Our implementation of MapReduce runs on a large cluster of commodity machines and is highly scalable: a typical MapReduce computation processes many terabytes of data on thousands of machines. Programmers find the system easy to use: hundreds of MapReduce programs have been implemented and upwards of one thousand MapReduce jobs are executed on Google's clusters every day.

我们的 MapReduce 实现运行在一个由廉价机器组成的大集群上，可扩展性很高：一次典型的 MapReduce 计算会在数千台机器上处理数 TB 数据。程序员觉得这套系统好用：至今已有数百个 MapReduce 程序被实现，每天在 Google 的集群上执行的 MapReduce 作业超过一千个。

---

## 1 Introduction · 引言

> Over the past five years, the authors and many others at Google have implemented hundreds of special-purpose computations that process large amounts of raw data, such as crawled documents, web request logs, etc., to compute various kinds of derived data, such as inverted indices, various representations of the graph structure of web documents, summaries of the number of pages crawled per host, the set of most frequent queries in a given day, etc. Most such computations are conceptually straightforward. However, the input data is usually large and the computations have to be distributed across hundreds or thousands of machines in order to finish in a reasonable amount of time. The issues of how to parallelize the computation, distribute the data, and handle failures conspire to obscure the original simple computation with large amounts of complex code to deal with these issues.

过去五年里，本文作者以及 Google 的许多人实现了数百个专用计算，用来处理大量原始数据（比如抓取到的文档、Web 请求日志等），并算出各种派生数据，比如倒排索引、Web 文档图结构的各种表示、按主机统计的抓取页面数摘要、某一天出现频率最高的查询集合等。这类计算在概念上大多很直白。但输入数据通常很大，计算必须分布到数百甚至数千台机器上才能在可接受的时间内完成。于是「怎么并行」「怎么分发数据」「怎么处理故障」这三个问题纠缠在一起，用大量复杂代码把原本简单的计算遮蔽掉了。

> As a reaction to this complexity, we designed a new abstraction that allows us to express the simple computations we were trying to perform but hides the messy details of parallelization, fault-tolerance, data distribution and load balancing in a library. Our abstraction is inspired by the map and reduce primitives present in Lisp and many other functional languages. We realized that most of our computations involved applying a map operation to each logical "record" in our input in order to compute a set of intermediate key/value pairs, and then applying a reduce operation to all the values that shared the same key, in order to combine the derived data appropriately. Our use of a functional model with user-specified map and reduce operations allows us to parallelize large computations easily and to use re-execution as the primary mechanism for fault tolerance.

针对这种复杂性，我们设计了一个新的抽象：它让我们能表达想要完成的简单计算，同时把并行化、容错、数据分发和负载均衡这些麻烦细节藏进库里。这个抽象受 Lisp 等众多函数式语言中的 map 和 reduce 原语启发。我们意识到，大多数计算都可以归结为两件事：对输入中每一条逻辑「记录」施加一次 map 操作，算出一组中间 key/value 对；再对共享同一个 key 的所有 value 施加一次 reduce 操作，把派生数据恰当地合并起来。采用这种由用户提供 map 和 reduce 操作的函数式模型，使我们能轻松并行化大型计算，并以重新执行（re-execution）作为容错的主要机制。

> The major contributions of this work are a simple and powerful interface that enables automatic parallelization and distribution of large-scale computations, combined with an implementation of this interface that achieves high performance on large clusters of commodity PCs.

这项工作的主要贡献是：一个简单而强大的接口，能自动并行化并分发大规模计算；以及该接口的一套实现，能在廉价 PC 组成的大集群上取得高性能。

> Section 2 describes the basic programming model and gives several examples. Section 3 describes an implementation of the MapReduce interface tailored towards our cluster-based computing environment. Section 4 describes several refinements of the programming model that we have found useful. Section 5 has performance measurements of our implementation for a variety of tasks. Section 6 explores the use of MapReduce within Google including our experiences in using it as the basis for a rewrite of our production indexing system. Section 7 discusses related and future work.

第 2 节介绍基本编程模型并给出若干例子。第 3 节介绍一套面向我们集群计算环境定制的 MapReduce 接口实现。第 4 节介绍我们发现有用的若干编程模型改进。第 5 节给出该实现在多种任务上的性能测量。第 6 节探讨 MapReduce 在 Google 内部的使用情况，包括我们以它为基础重写生产索引系统的经验。第 7 节讨论相关工作与后续方向。

---

## 2 Programming Model · 编程模型

> The computation takes a set of input key/value pairs, and produces a set of output key/value pairs. The user of the MapReduce library expresses the computation as two functions: Map and Reduce.

计算接收一组输入 key/value 对，产出一组输出 key/value 对。MapReduce 库的使用者把计算表达成两个函数：Map 和 Reduce。

> Map, written by the user, takes an input pair and produces a set of intermediate key/value pairs. The MapReduce library groups together all intermediate values associated with the same intermediate key I and passes them to the Reduce function.

Map 由用户编写，接收一个输入对，产出一组中间 key/value 对。MapReduce 库会把关联到同一个中间 key I 的所有中间 value 归并到一起，交给 Reduce 函数。

> The Reduce function, also written by the user, accepts an intermediate key I and a set of values for that key. It merges together these values to form a possibly smaller set of values. Typically just zero or one output value is produced per Reduce invocation. The intermediate values are supplied to the user's reduce function via an iterator. This allows us to handle lists of values that are too large to fit in memory.

Reduce 函数同样由用户编写，接收一个中间 key I 和该 key 对应的一组 value。它把这些 value 合并成一个规模可能更小的 value 集合。通常每次 Reduce 调用只产出零个或一个输出值。中间 value 通过一个迭代器（iterator）交给用户的 reduce 函数，这让我们能处理大到放不进内存的 value 列表。

### 2.1 Example · 示例

> Consider the problem of counting the number of occurrences of each word in a large collection of documents. The user would write code similar to the following pseudo-code:

考虑这样一个问题：统计一大批文档中每个词出现的次数。用户会写出类似下面这样的伪代码：

```
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

> The map function emits each word plus an associated count of occurrences (just '1' in this simple example). The reduce function sums together all counts emitted for a particular word.

map 函数对每个词连同它出现次数（在这个简单例子里就是 `1`）做一次发射。reduce 函数把某个词对应的所有计数累加起来。

> In addition, the user writes code to fill in a mapreduce specification object with the names of the input and output files, and optional tuning parameters. The user then invokes the MapReduce function, passing it the specification object. The user's code is linked together with the MapReduce library (implemented in C++). Appendix A contains the full program text for this example.

此外，用户还要写代码填充一个 mapreduce 规格对象（specification object），写入输入输出文件名以及可选的调优参数。随后用户调用 MapReduce 函数并把这个规格对象传进去。用户的代码与 MapReduce 库（用 C++ 实现）链接在一起。完整的示例程序见附录 A。

### 2.2 Types · 类型

> Even though the previous pseudo-code is written in terms of string inputs and outputs, conceptually the map and reduce functions supplied by the user have associated types:

尽管上一段伪代码是按字符串输入输出来写的，但概念上，用户提供的 map 和 reduce 函数带有各自的类型：

```
map    (k1, v1)         → list(k2, v2)
reduce (k2, list(v2))   → list(v2)
```

> I.e., the input keys and values are drawn from a different domain than the output keys and values. Furthermore, the intermediate keys and values are from the same domain as the output keys and values.

也就是说，输入 key/value 所属的域与输出 key/value 所属的域不同；而中间 key/value 所在的域与输出 key/value 所在的域相同。

> Our C++ implementation passes strings to and from the user-defined functions and leaves it to the user code to convert between strings and appropriate types.

我们的 C++ 实现与用户自定义函数之间传递的都是字符串，字符串与合适类型之间的转换留给用户代码自己做。

### 2.3 More Examples · 更多示例

> Here are a few simple examples of interesting programs that can be easily expressed as MapReduce computations.

下面几个简单例子，都是能轻松表达成 MapReduce 计算的有趣程序。

> **Distributed Grep:** The map function emits a line if it matches a supplied pattern. The reduce function is an identity function that just copies the supplied intermediate data to the output.

分布式 Grep：如果某一行匹配给定的模式，map 函数就把它发射出去。reduce 函数是一个恒等函数，只把收到的中间数据原样复制到输出。

> **Count of URL Access Frequency:** The map function processes logs of web page requests and outputs ⟨URL, 1⟩. The reduce function adds together all values for the same URL and emits a ⟨URL, total count⟩ pair.

URL 访问频次统计：map 函数处理 Web 页面请求日志，输出 ⟨URL, 1⟩。reduce 函数把同一个 URL 的所有 value 相加，发射一个 ⟨URL, 总次数⟩ 对。

> **Reverse Web-Link Graph:** The map function outputs ⟨target, source⟩ pairs for each link to a target URL found in a page named source. The reduce function concatenates the list of all source URLs associated with a given target URL and emits the pair: ⟨target, list(source)⟩

反向 Web 链接图：对页面 source 中找到的每一条指向 target URL 的链接，map 函数输出 ⟨target, source⟩ 对。reduce 函数把与某个 target URL 关联的所有 source URL 拼成一个列表，发射 ⟨target, list(source)⟩ 对。

> **Term-Vector per Host:** A term vector summarizes the most important words that occur in a document or a set of documents as a list of ⟨word, frequency⟩ pairs. The map function emits a ⟨hostname, term vector⟩ pair for each input document (where the hostname is extracted from the URL of the document). The reduce function is passed all per-document term vectors for a given host. It adds these term vectors together, throwing away infrequent terms, and then emits a final ⟨hostname, term vector⟩ pair.

按主机的词项向量：词项向量（term vector）把一篇或一批文档中最重要的词概括成一组 ⟨词, 频次⟩ 对。map 函数对每篇输入文档发射一个 ⟨主机名, 词项向量⟩ 对（主机名从文档 URL 中抽取）。reduce 函数会收到某个主机的所有单篇文档词项向量，把它们相加、丢弃低频词，最后发射一个 ⟨主机名, 词项向量⟩ 对。

> **Inverted Index:** The map function parses each document, and emits a sequence of ⟨word, document ID⟩ pairs. The reduce function accepts all pairs for a given word, sorts the corresponding document IDs and emits a ⟨word, list(document ID)⟩ pair. The set of all output pairs forms a simple inverted index. It is easy to augment this computation to keep track of word positions.

倒排索引：map 函数解析每篇文档，发射一串 ⟨词, 文档 ID⟩ 对。reduce 函数接收某个词的全部配对，把对应的文档 ID 排序后发射 ⟨词, list(文档 ID)⟩ 对。所有输出对合起来就构成一个简单的倒排索引。要在此基础上跟踪词的位置也很容易。

> **Distributed Sort:** The map function extracts the key from each record, and emits a ⟨key, record⟩ pair. The reduce function emits all pairs unchanged. This computation depends on the partitioning facilities described in Section 4.1 and the ordering properties described in Section 4.2.

分布式排序：map 函数从每条记录中抽出 key，发射 ⟨key, record⟩ 对。reduce 函数把收到的所有对原样发射。这个计算依赖 4.1 节的分区机制和 4.2 节的排序保证。

---

## 3 Implementation · 实现

> Many different implementations of the MapReduce interface are possible. The right choice depends on the environment. For example, one implementation may be suitable for a small shared-memory machine, another for a large NUMA multi-processor, and yet another for an even larger collection of networked machines.

MapReduce 接口可以有很多种不同的实现，合适的选择取决于所处的环境。例如，某种实现可能适合小的共享内存机器，另一种适合大型 NUMA 多处理器，还有一种适合规模更大的联网机器集合。

> This section describes an implementation targeted to the computing environment in wide use at Google: large clusters of commodity PCs connected together with switched Ethernet [4]. In our environment:

本节介绍的实现面向 Google 内部广泛使用的那类计算环境：由交换式以太网 [4] 互联的、由廉价 PC 组成的大集群。在我们的环境里：

> (1) Machines are typically dual-processor x86 processors running Linux, with 2-4 GB of memory per machine.

（1）机器通常是运行 Linux 的双处理器 x86，每台内存 2–4 GB。

> (2) Commodity networking hardware is used - typically either 100 megabits/second or 1 gigabit/second at the machine level, but averaging considerably less in overall bisection bandwidth.

（2）使用廉价网络硬件，单机层面通常是 100 Mb/s 或 1 Gb/s，但整体对分带宽（bisection bandwidth）的平均值要低得多。

> (3) A cluster consists of hundreds or thousands of machines, and therefore machine failures are common.

（3）一个集群由数百到数千台机器组成，因此机器故障是常事。

> (4) Storage is provided by inexpensive IDE disks attached directly to individual machines. A distributed file system [8] developed in-house is used to manage the data stored on these disks. The file system uses replication to provide availability and reliability on top of unreliable hardware.

（4）存储由直接挂在各台机器上的廉价 IDE 磁盘提供。我们用自研的分布式文件系统 [8] 管理这些磁盘上的数据。该文件系统通过副本机制，在不可靠硬件之上提供可用性与可靠性。

> (5) Users submit jobs to a scheduling system. Each job consists of a set of tasks, and is mapped by the scheduler to a set of available machines within a cluster.

（5）用户把作业提交给调度系统。每个作业由一组任务组成，调度器把它映射到集群内一批可用的机器上。

### 3.1 Execution Overview · 执行流程概览

> The Map invocations are distributed across multiple machines by automatically partitioning the input data into a set of M splits. The input splits can be processed in parallel by different machines. Reduce invocations are distributed by partitioning the intermediate key space into R pieces using a partitioning function (e.g., `hash(key) mod R`). The number of partitions (R) and the partitioning function are specified by the user.

Map 调用的分发方式，是把输入数据自动切分成 M 个 split，这些输入 split 可以由不同机器并行处理。Reduce 调用的分发方式，是用一个分区函数（例如 `hash(key) mod R`）把中间 key 空间划成 R 份。分区数 R 和分区函数都由用户指定。

> Figure 1 shows the overall flow of a MapReduce operation in our implementation. When the user program calls the MapReduce function, the following sequence of actions occurs (the numbered labels in Figure 1 correspond to the numbers in the list below):

图 1 展示了我们这套实现中一次 MapReduce 操作的完整流程。当用户程序调用 MapReduce 函数时，会依次发生下列动作（图 1 中的编号与下面列表的编号对应）：

```
            ┌───────────────────────┐
   (1) fork │        Master         │ (1) fork
   ────────▶│ (2) assign map/reduce │◀────────
            └───────────┬───────────┘
                        │
        ┌───────────────┴────────────────┐
        ▼                                ▼
  ┌────────────┐   (3) read    ┌──────────────┐
  │Input files │──────────────▶│ map workers  │
  │split 0 .. 4│               │  (M tasks)   │
  └────────────┘               └──────┬───────┘
                                      │ (4) local write
                                      ▼
                               ┌──────────────┐
                               │ Intermediate │
                               │ files (local │
                               │    disks)    │
                               └──────┬───────┘
                                      │ (5) remote read
                                      ▼
                               ┌──────────────┐  (6)      ┌──────────────┐
                               │reduce workers│──write──▶│ Output files │
                               │  (R tasks)   │          │file 0 .. R-1 │
                               └──────────────┘          └──────────────┘
```

图 1：执行流程概览（Execution overview）。原文图 1 的六个编号步骤含义如下。

| 编号 | 动作 |
|------|------|
| (1) fork | 用户程序把 MapReduce 程序复制到集群中的一批机器上，其中一个是 master，其余是 worker |
| (2) assign | master 挑出空闲 worker，给每个 worker 分配一个 map 任务或一个 reduce 任务 |
| (3) read | 被分配 map 任务的 worker 读取对应输入 split 的内容，把 key/value 对交给用户的 Map 函数 |
| (4) local write | Map 函数产生的中间 key/value 对缓存在内存中，之后按分区函数周期性写到本地磁盘的 R 个区域 |
| (5) remote read | reduce worker 用远程过程调用从各 map worker 的本地磁盘读取缓冲数据，再按中间 key 排序 |
| (6) write | reduce worker 遍历已排序的中间数据，把每个 key 及其对应 value 集合交给用户的 Reduce 函数，结果追加到该 reduce 分区的最终输出文件 |

> 1. The MapReduce library in the user program first splits the input files into M pieces of typically 16 megabytes to 64 megabytes (MB) per piece (controllable by the user via an optional parameter). It then starts up many copies of the program on a cluster of machines.

1. 用户程序中的 MapReduce 库先把输入文件切成 M 片，每片通常 16 MB 到 64 MB（用户可以通过可选参数控制）。然后它在一批机器上启动这个程序的许多副本。

> 2. One of the copies of the program is special - the master. The rest are workers that are assigned work by the master. There are M map tasks and R reduce tasks to assign. The master picks idle workers and assigns each one a map task or a reduce task.

2. 这些副本中有一个是特殊的，即 master。其余的都是 worker，由 master 给它们分配工作。待分配的有 M 个 map 任务和 R 个 reduce 任务。master 挑选空闲的 worker，给每个 worker 分配一个 map 任务或一个 reduce 任务。

> 3. A worker who is assigned a map task reads the contents of the corresponding input split. It parses key/value pairs out of the input data and passes each pair to the user-defined Map function. The intermediate key/value pairs produced by the Map function are buffered in memory.

3. 被分配 map 任务的 worker 读取对应输入 split 的内容。它从输入数据中解析出 key/value 对，把每一对交给用户定义的 Map 函数。Map 函数产生的中间 key/value 对缓存在内存中。

> 4. Periodically, the buffered pairs are written to local disk, partitioned into R regions by the partitioning function. The locations of these buffered pairs on the local disk are passed back to the master, who is responsible for forwarding these locations to the reduce workers.

4. 这些缓冲的 key/value 对会周期性地写到本地磁盘，并按键分区函数分成 R 个区域。这些数据在本地磁盘上的位置会回传给 master，由 master 负责把位置转发给 reduce worker。

> 5. When a reduce worker is notified by the master about these locations, it uses remote procedure calls to read the buffered data from the local disks of the map workers. When a reduce worker has read all intermediate data, it sorts it by the intermediate keys so that all occurrences of the same key are grouped together. The sorting is needed because typically many different keys map to the same reduce task. If the amount of intermediate data is too large to fit in memory, an external sort is used.

5. reduce worker 收到 master 通知的这些位置后，用远程过程调用从 map worker 的本地磁盘读取缓冲数据。当 reduce worker 读完所有中间数据后，会按中间 key 排序，让同一个 key 的所有出现位置聚到一起。之所以需要排序，是因为通常有许多不同的 key 映射到同一个 reduce 任务上。如果中间数据量大到放不进内存，就改用外部排序。

> 6. The reduce worker iterates over the sorted intermediate data and for each unique intermediate key encountered, it passes the key and the corresponding set of intermediate values to the user's Reduce function. The output of the Reduce function is appended to a final output file for this reduce partition.

6. reduce worker 遍历已排序的中间数据，每遇到一个唯一的中间 key，就把该 key 及对应的中间 value 集合交给用户的 Reduce 函数。Reduce 函数的输出追加到该 reduce 分区的最终输出文件末尾。

> 7. When all map tasks and reduce tasks have been completed, the master wakes up the user program. At this point, the MapReduce call in the user program returns back to the user code.

7. 当所有 map 任务和 reduce 任务都完成后，master 唤醒用户程序。此时用户程序中的 MapReduce 调用返回用户代码。

> After successful completion, the output of the mapreduce execution is available in the R output files (one per reduce task, with file names as specified by the user). Typically, users do not need to combine these R output files into one file - they often pass these files as input to another MapReduce call, or use them from another distributed application that is able to deal with input that is partitioned into multiple files.

成功完成后，这次 mapreduce 执行的输出就在 R 个输出文件里（每个 reduce 任务一个，文件名由用户指定）。通常用户不需要把这 R 个输出文件合并成一个，他们一般会把这些文件作为输入传给下一次 MapReduce 调用，或者交给另一个能够处理「输入被切成多个文件」的分布式应用来使用。

### 3.2 Master Data Structures · 主节点数据结构

> The master keeps several data structures. For each map task and reduce task, it stores the state (idle, in-progress, or completed), and the identity of the worker machine (for non-idle tasks).

master 维护若干数据结构。对每个 map 任务和 reduce 任务，它记录状态（idle、in-progress 或 completed），以及（对非 idle 的任务）执行它的 worker 机器标识。

> The master is the conduit through which the location of intermediate file regions is propagated from map tasks to reduce tasks. Therefore, for each completed map task, the master stores the locations and sizes of the R intermediate file regions produced by the map task. Updates to this location and size information are received as map tasks are completed. The information is pushed incrementally to workers that have in-progress reduce tasks.

master 是中间文件区域位置从 map 任务传播到 reduce 任务的通道。因此，对每个已完成的 map 任务，master 都保存该任务产生的 R 个中间文件区域的位置和大小。这些位置与大小的更新随 map 任务的完成而到来，并会被增量地推送给那些正在执行 reduce 任务的 worker。

### 3.3 Fault Tolerance · 容错

> Since the MapReduce library is designed to help process very large amounts of data using hundreds or thousands of machines, the library must tolerate machine failures gracefully.

MapReduce 库的设计目标是用数百到数千台机器处理海量数据，因此它必须能优雅地容忍机器故障。

#### Worker Failure · Worker 失败

> The master pings every worker periodically. If no response is received from a worker in a certain amount of time, the master marks the worker as failed. Any map tasks completed by the worker are reset back to their initial idle state, and therefore become eligible for scheduling on other workers. Similarly, any map task or reduce task in progress on a failed worker is also reset to idle and becomes eligible for rescheduling.

master 周期性地 ping 每个 worker。如果在某个时间窗口内没有收到某个 worker 的响应，master 就把该 worker 标记为失败。这个 worker 已完成的 map 任务全部被重置回初始的 idle 状态，从而可以被重新调度到其他 worker 上。同理，失败 worker 上正在进行的 map 任务或 reduce 任务也被重置为 idle，并可以重新调度。

> Completed map tasks are re-executed on a failure because their output is stored on the local disk(s) of the failed machine and is therefore inaccessible. Completed reduce tasks do not need to be re-executed since their output is stored in a global file system.

已完成的 map 任务在失败时会被重新执行，因为它们的输出存在失败机器的本地磁盘上，已经不可访问。已完成的 reduce 任务不需要重新执行，因为它们的输出存在全局文件系统里。

> When a map task is executed first by worker A and then later executed by worker B (because A failed), all workers executing reduce tasks are notified of the re-execution. Any reduce task that has not already read the data from worker A will read the data from worker B.

当某个 map 任务先由 worker A 执行、之后（因为 A 失败）又由 worker B 执行时，所有正在执行 reduce 任务的 worker 都会收到这次重新执行的通知。任何尚未从 worker A 读取该数据的 reduce 任务，都会改为从 worker B 读取。

> MapReduce is resilient to large-scale worker failures. For example, during one MapReduce operation, network maintenance on a running cluster was causing groups of 80 machines at a time to become unreachable for several minutes. The MapReduce master simply re-executed the work done by the unreachable worker machines, and continued to make forward progress, eventually completing the MapReduce operation.

MapReduce 对大规模 worker 故障有恢复能力。例如，在一次 MapReduce 操作期间，运行中集群的网络维护导致每次有 80 台机器成组地失去联系数分钟。MapReduce master 只是把不可达机器做过的工作重新执行，继续向前推进，最终完成了这次 MapReduce 操作。

#### Master Failure · Master 失败

> It is easy to make the master write periodic checkpoints of the master data structures described above. If the master task dies, a new copy can be started from the last checkpointed state. However, given that there is only a single master, its failure is unlikely; therefore our current implementation aborts the MapReduce computation if the master fails. Clients can check for this condition and retry the MapReduce operation if they desire.

让 master 周期性地把上述数据结构写成检查点是很容易的。如果 master 任务挂掉，可以从最近一次检查点的状态启动一个新的副本。不过鉴于只有一个 master，它失败的概率很低，所以当前实现在 master 失败时直接中止整次 MapReduce 计算。客户端可以检测这一情况并自行重试。

#### Semantics in the Presence of Failures · 存在故障时的语义

> When the user-supplied map and reduce operators are deterministic functions of their input values, our distributed implementation produces the same output as would have been produced by a non-faulting sequential execution of the entire program.

当用户提供的 map 和 reduce 算子是输入值的确定性函数时，我们的分布式实现产出的输出，与整个程序无故障串行执行所产出的输出完全一致。

> We rely on atomic commits of map and reduce task outputs to achieve this property. Each in-progress task writes its output to private temporary files. A reduce task produces one such file, and a map task produces R such files (one per reduce task). When a map task completes, the worker sends a message to the master and includes the names of the R temporary files in the message. If the master receives a completion message for an already completed map task, it ignores the message. Otherwise, it records the names of R files in a master data structure.

我们依靠 map 和 reduce 任务输出的原子提交来实现这一性质。每个进行中的任务把自己的输出写到私有的临时文件里。一个 reduce 任务产生一个这样的文件，一个 map 任务产生 R 个这样的文件（每个 reduce 任务一个）。map 任务完成时，worker 向 master 发送一条消息，消息里带上这 R 个临时文件的名字。如果 master 收到的是一条针对已完成 map 任务的完成消息，就忽略它；否则就把这 R 个文件的名字记录到 master 的某个数据结构中。

> When a reduce task completes, the reduce worker atomically renames its temporary output file to the final output file. If the same reduce task is executed on multiple machines, multiple rename calls will be executed for the same final output file. We rely on the atomic rename operation provided by the underlying file system to guarantee that the final file system state contains just the data produced by one execution of the reduce task.

reduce 任务完成时，reduce worker 把它的临时输出文件原子地重命名成最终输出文件。如果同一个 reduce 任务在多台机器上执行，那么同一个最终输出文件会被多次重命名。我们依靠底层文件系统提供的原子重命名操作，来保证最终的文件系统状态里只包含该 reduce 任务某一次执行产出的数据。

> The vast majority of our map and reduce operators are deterministic, and the fact that our semantics are equivalent to a sequential execution in this case makes it very easy for programmers to reason about their program's behavior. When the map and/or reduce operators are non-deterministic, we provide weaker but still reasonable semantics. In the presence of non-deterministic operators, the output of a particular reduce task R1 is equivalent to the output for R1 produced by a sequential execution of the non-deterministic program. However, the output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program.

我们绝大多数 map 和 reduce 算子都是确定性的，而在这种情况下语义等价于串行执行，这一点让程序员非常容易推理自己程序的行为。当 map 或 reduce 算子是非确定性的，我们提供的是更弱但仍然合理的语义。存在非确定性算子时，某个特定 reduce 任务 R1 的输出，等价于对该非确定性程序做一次串行执行时 R1 的输出；但另一个 reduce 任务 R2 的输出，可能对应的是对该非确定性程序做另一次串行执行时 R2 的输出。

> Consider map task M and reduce tasks R1 and R2. Let e(Ri) be the execution of Ri that committed (there is exactly one such execution). The weaker semantics arise because e(R1) may have read the output produced by one execution of M and e(R2) may have read the output produced by a different execution of M.

考虑 map 任务 M 和 reduce 任务 R1、R2。记 e(Ri) 为 Ri 中真正提交的那次执行（这样的执行恰好只有一个）。更弱的语义之所以出现，是因为 e(R1) 读取的可能是 M 的某一次执行的输出，而 e(R2) 读取的可能是 M 的另一次执行的输出。

### 3.4 Locality · 局部性

> Network bandwidth is a relatively scarce resource in our computing environment. We conserve network bandwidth by taking advantage of the fact that the input data (managed by GFS [8]) is stored on the local disks of the machines that make up our cluster. GFS divides each file into 64 MB blocks, and stores several copies of each block (typically 3 copies) on different machines. The MapReduce master takes the location information of the input files into account and attempts to schedule a map task on a machine that contains a replica of the corresponding input data. Failing that, it attempts to schedule a map task near a replica of that task's input data (e.g., on a worker machine that is on the same network switch as the machine containing the data). When running large MapReduce operations on a significant fraction of the workers in a cluster, most input data is read locally and consumes no network bandwidth.

在我们的计算环境里，网络带宽是相对稀缺的资源。我们节省带宽的方式，是利用这样一个事实：输入数据（由 GFS [8] 管理）就存在构成集群的那些机器的本地磁盘上。GFS 把每个文件切成 64 MB 的块，并把每块的若干副本（通常 3 份）放在不同机器上。MapReduce master 会把输入文件的位置信息考虑进来，尽量把 map 任务调度到存有对应输入数据副本的那台机器上。做不到时，它会尽量把 map 任务调度到该任务输入数据副本的附近（比如与存有该数据的机器接在同一台网络交换机上的 worker）。当一次大型 MapReduce 操作占用了集群中相当大比例的 worker 时，大部分输入数据都是本地读取，不消耗网络带宽。

### 3.5 Task Granularity · 任务粒度

> We subdivide the map phase into M pieces and the reduce phase into R pieces, as described above. Ideally, M and R should be much larger than the number of worker machines. Having each worker perform many different tasks improves dynamic load balancing, and also speeds up recovery when a worker fails: the many map tasks it has completed can be spread out across all the other worker machines.

如上所述，我们把 map 阶段细分成 M 片、reduce 阶段细分成 R 片。理想情况下，M 和 R 应当远大于 worker 机器的数量。让每个 worker 执行许多不同的任务，既改善动态负载均衡，也加快 worker 失败后的恢复：它已完成的那许多 map 任务可以分散到其余所有 worker 机器上。

> There are practical bounds on how large M and R can be in our implementation, since the master must make O(M + R) scheduling decisions and keeps O(M ∗ R) state in memory as described above. (The constant factors for memory usage are small however: the O(M ∗ R) piece of the state consists of approximately one byte of data per map task/reduce task pair.)

在我们的实现里，M 和 R 能取多大有实际的边界：如上所述，master 需要做 `O(M + R)` 次调度决策，并在内存中保存 `O(M ∗ R)` 的状态。（不过内存占用的常数因子很小：`O(M ∗ R)` 这部分状态对每个 map/reduce 任务对大约只占一个字节。）

> Furthermore, R is often constrained by users because the output of each reduce task ends up in a separate output file. In practice, we tend to choose M so that each individual task is roughly 16 MB to 64 MB of input data (so that the locality optimization described above is most effective), and we make R a small multiple of the number of worker machines we expect to use. We often perform MapReduce computations with M = 200,000 and R = 5,000, using 2,000 worker machines.

此外，R 常常受用户限制，因为每个 reduce 任务的输出会落在各自独立的输出文件里。实践中我们倾向于这样选 M：让每个单独任务的输入数据大约在 16 MB 到 64 MB 之间（这样上面的局部性优化最有效）；而 R 取成预期使用机器数的一个小的倍数。我们经常用 M = 200,000、R = 5,000、2000 台 worker 机器来跑 MapReduce。

### 3.6 Backup Tasks · 备份任务

> One of the common causes that lengthens the total time taken for a MapReduce operation is a "straggler": a machine that takes an unusually long time to complete one of the last few map or reduce tasks in the computation.

拖长一次 MapReduce 操作总耗时的常见原因之一是「掉队者」（straggler）：某台机器完成计算中最后几个 map 或 reduce 任务之一的耗时异常地长。

> Stragglers can arise for a whole host of reasons. For example, a machine with a bad disk may experience frequent correctable errors that slow its read performance from 30 MB/s to 1 MB/s. The cluster scheduling system may have scheduled other tasks on the machine, causing it to execute the MapReduce code more slowly due to competition for CPU, memory, local disk, or network bandwidth. A recent problem we experienced was a bug in machine initialization code that caused processor caches to be disabled: computations on affected machines slowed down by over a factor of one hundred.

掉队者的成因很多。例如，一台磁盘有毛病的机器可能频繁出现可纠正错误，把读性能从 30 MB/s 拉低到 1 MB/s。集群调度系统可能在这台机器上排了别的任务，使它因为争抢 CPU、内存、本地磁盘或网络带宽而执行得更慢。我们最近遇到的一个问题是机器初始化代码里的 bug 导致处理器缓存被关闭：受影响机器上的计算慢了一百倍以上。

> We have a general mechanism to alleviate the problem of stragglers. When a MapReduce operation is close to completion, the master schedules backup executions of the remaining in-progress tasks. The task is marked as completed whenever either the primary or the backup execution completes. We have tuned this mechanism so that it typically increases the computational resources used by the operation by no more than a few percent. We have found that this significantly reduces the time to complete large MapReduce operations. As an example, the sort program described in Section 5.3 takes 44% longer to complete when the backup task mechanism is disabled.

我们有一个通用机制来缓解掉队者问题。当一次 MapReduce 操作接近完成时，master 会为剩余进行中的任务安排备份执行（backup execution）。只要主执行和备份执行中任意一个完成，该任务就被标记为已完成。我们调过这套机制，通常它带来的额外计算资源开销不超过几个百分点。我们发现它显著缩短了大型 MapReduce 操作的完成时间。举个例子：5.3 节描述的 sort 程序在关闭备份任务机制后，完成时间要长 44%。

---

## 4 Refinements · 改进

> Although the basic functionality provided by simply writing Map and Reduce functions is sufficient for most needs, we have found a few extensions useful. These are described in this section.

仅仅编写 Map 和 Reduce 函数所提供的基本功能，已经足以满足大多数需求，但我们发现有几个扩展很有用。本节逐一介绍。

### 4.1 Partitioning Function · 分区函数

> The users of MapReduce specify the number of reduce tasks/output files that they desire (R). Data gets partitioned across these tasks using a partitioning function on the intermediate key. A default partitioning function is provided that uses hashing (e.g. "hash(key) mod R"). This tends to result in fairly well-balanced partitions. In some cases, however, it is useful to partition data by some other function of the key. For example, sometimes the output keys are URLs, and we want all entries for a single host to end up in the same output file. To support situations like this, the user of the MapReduce library can provide a special partitioning function. For example, using "hash(Hostname(urlkey)) mod R" as the partitioning function causes all URLs from the same host to end up in the same output file.

MapReduce 的使用者指定自己想要的 reduce 任务／输出文件数量 R。数据通过一个作用在中间 key 上的分区函数分发到这些任务中。我们提供一个默认的哈希分区函数（例如 `hash(key) mod R`），它通常能给出相当均衡的分区。但有些情况下，用 key 的别的函数来做分区更合适。例如，有时输出 key 是 URL，而我们希望同一个主机的所有条目都落在同一个输出文件里。为支持这类场景，MapReduce 库的使用者可以提供一个特殊的分区函数。比如用 `hash(Hostname(urlkey)) mod R` 作为分区函数，就能让同一主机的所有 URL 落到同一个输出文件中。

### 4.2 Ordering Guarantees · 顺序保证

> We guarantee that within a given partition, the intermediate key/value pairs are processed in increasing key order. This ordering guarantee makes it easy to generate a sorted output file per partition, which is useful when the output file format needs to support efficient random access lookups by key, or users of the output find it convenient to have the data sorted.

我们保证：在给定的分区内，中间 key/value 对按 key 递增的顺序被处理。这个顺序保证使每个分区都能轻松生成已排序的输出文件，这在输出文件格式需要支持按 key 高效随机访问查找时很有用，或者在输出数据的使用者觉得排序好的数据更方便时很有用。

### 4.3 Combiner Function · Combiner 函数

> In some cases, there is significant repetition in the intermediate keys produced by each map task, and the user-specified Reduce function is commutative and associative. A good example of this is the word counting example in Section 2.1. Since word frequencies tend to follow a Zipf distribution, each map task will produce hundreds or thousands of records of the form `<the, 1>`. All of these counts will be sent over the network to a single reduce task and then added together by the Reduce function to produce one number. We allow the user to specify an optional Combiner function that does partial merging of this data before it is sent over the network.

有些情况下，每个 map 任务产生的中间 key 有大量重复，而用户指定的 Reduce 函数满足交换律和结合律。2.1 节的词频统计就是一个典型例子：由于词频大体服从 Zipf 分布，每个 map 任务都会产生成百上千条形如 `<the, 1>` 的记录。这些计数本来都要经网络送到同一个 reduce 任务，再由 Reduce 函数加起来得到一个数。我们允许用户指定一个可选的 Combiner 函数，在数据经网络发送之前先做部分合并。

> The Combiner function is executed on each machine that performs a map task. Typically the same code is used to implement both the combiner and the reduce functions. The only difference between a reduce function and a combiner function is how the MapReduce library handles the output of the function. The output of a reduce function is written to the final output file. The output of a combiner function is written to an intermediate file that will be sent to a reduce task.

Combiner 函数在每个执行 map 任务的机器上运行。通常 combiner 和 reduce 两个函数用同一份代码实现。二者唯一的差别在于 MapReduce 库如何处理函数的输出：reduce 函数的输出写到最终输出文件，combiner 函数的输出写到一份将被送往 reduce 任务的中间文件。

> Partial combining significantly speeds up certain classes of MapReduce operations. Appendix A contains an example that uses a combiner.

部分合并显著加速了某些类别的 MapReduce 操作。附录 A 给出了一个使用 combiner 的例子。

### 4.4 Input and Output Types · 输入与输出类型

> The MapReduce library provides support for reading input data in several different formats. For example, "text" mode input treats each line as a key/value pair: the key is the offset in the file and the value is the contents of the line. Another common supported format stores a sequence of key/value pairs sorted by key. Each input type implementation knows how to split itself into meaningful ranges for processing as separate map tasks (e.g. text mode's range splitting ensures that range splits occur only at line boundaries). Users can add support for a new input type by providing an implementation of a simple reader interface, though most users just use one of a small number of predefined input types.

MapReduce 库支持以若干不同格式读取输入数据。例如，`text` 模式把每一行当作一个 key/value 对：key 是文件中的偏移量，value 是该行的内容。另一种常见的支持格式存放的是按键排好序的 key/value 序列。每种输入类型的实现都知道如何把自己切成语义上有意义的区间，以便作为独立的 map 任务处理（例如 text 模式的区间切分保证切点只落在行边界上）。用户可以通过实现一个简单的 reader 接口来支持新的输入类型，不过大多数用户只会用到少数几种预定义输入类型。

> A reader does not necessarily need to provide data read from a file. For example, it is easy to define a reader that reads records from a database, or from data structures mapped in memory.

reader 不一定非要提供从文件读取的数据。例如，很容易定义一个从数据库读取记录的 reader，或者从内存映射的数据结构中读取的 reader。

> In a similar fashion, we support a set of output types for producing data in different formats and it is easy for user code to add support for new output types.

类似地，我们支持一组输出类型以产出不同格式的数据，用户代码也很容易添加对新输出类型的支持。

### 4.5 Side-effects · 副作用

> In some cases, users of MapReduce have found it convenient to produce auxiliary files as additional outputs from their map and/or reduce operators. We rely on the application writer to make such side-effects atomic and idempotent. Typically the application writes to a temporary file and atomically renames this file once it has been fully generated.

有些情况下，MapReduce 的使用者觉得在自己的 map 或 reduce 算子里额外产出一些辅助文件很方便。我们要求应用编写者自己保证这类副作用是原子的且幂等的。通常的做法是：应用先写临时文件，写完之后再原子地重命名。

> We do not provide support for atomic two-phase commits of multiple output files produced by a single task. Therefore, tasks that produce multiple output files with cross-file consistency requirements should be deterministic. This restriction has never been an issue in practice.

我们不支持对单个任务产出的多个输出文件做原子的两阶段提交。因此，如果一个任务要产出多个输出文件且它们之间存在跨文件一致性要求，该任务就应当是确定性的。这个限制在实践中从未成为问题。

### 4.6 Skipping Bad Records · 跳过坏记录

> Sometimes there are bugs in user code that cause the Map or Reduce functions to crash deterministically on certain records. Such bugs prevent a MapReduce operation from completing. The usual course of action is to fix the bug, but sometimes this is not feasible; perhaps the bug is in a third-party library for which source code is unavailable. Also, sometimes it is acceptable to ignore a few records, for example when doing statistical analysis on a large data set. We provide an optional mode of execution where the MapReduce library detects which records cause deterministic crashes and skips these records in order to make forward progress.

有时用户代码里的 bug 会让 Map 或 Reduce 函数在某些记录上确定性地崩溃。这类 bug 会使 MapReduce 操作无法完成。通常的处理方式是修掉这个 bug，但有时并不可行——比如 bug 出在没有源码的第三方库里。另外，有时候忽略少量记录是可以接受的，比如在对大数据集做统计分析时。我们提供一种可选的执行模式：MapReduce 库检测出哪些记录引发确定性崩溃，并跳过这些记录，以便继续向前推进。

> Each worker process installs a signal handler that catches segmentation violations and bus errors. Before invoking a user Map or Reduce operation, the MapReduce library stores the sequence number of the argument in a global variable. If the user code generates a signal, the signal handler sends a "last gasp" UDP packet that contains the sequence number to the MapReduce master. When the master has seen more than one failure on a particular record, it indicates that the record should be skipped when it issues the next re-execution of the corresponding Map or Reduce task.

每个 worker 进程都会安装一个信号处理器，捕获段错误和总线错误。在调用用户的 Map 或 Reduce 操作之前，MapReduce 库会把本次参数的序号存到一个全局变量里。如果用户代码触发了信号，信号处理器就向 MapReduce master 发送一个「临终」（last gasp）UDP 包，里面带上该序号。当 master 发现某条特定记录上出现了不止一次失败，它在下一次重新执行对应的 Map 或 Reduce 任务时，就会指明应当跳过这条记录。

### 4.7 Local Execution · 本地执行

> Debugging problems in Map or Reduce functions can be tricky, since the actual computation happens in a distributed system, often on several thousand machines, with work assignment decisions made dynamically by the master. To help facilitate debugging, profiling, and small-scale testing, we have developed an alternative implementation of the MapReduce library that sequentially executes all of the work for a MapReduce operation on the local machine. Controls are provided to the user so that the computation can be limited to particular map tasks. Users invoke their program with a special flag and can then easily use any debugging or testing tools they find useful (e.g. gdb).

调试 Map 或 Reduce 函数里的问题可能很棘手，因为真正的计算发生在分布式系统中，常常横跨数千台机器，工作分配又由 master 动态决定。为了便于调试、性能剖析和小规模测试，我们开发了 MapReduce 库的另一套实现：它在本地机器上把一次 MapReduce 操作的全部工作串行执行。我们向用户提供了控制手段，可以把计算限制在特定的 map 任务上。用户用一个特殊标志启动自己的程序，就可以随手使用任何他觉得有用的调试或测试工具（比如 gdb）。

### 4.8 Status Information · 状态信息

> The master runs an internal HTTP server and exports a set of status pages for human consumption. The status pages show the progress of the computation, such as how many tasks have been completed, how many are in progress, bytes of input, bytes of intermediate data, bytes of output, processing rates, etc. The pages also contain links to the standard error and standard output files generated by each task. The user can use this data to predict how long the computation will take, and whether or not more resources should be added to the computation. These pages can also be used to figure out when the computation is much slower than expected.

master 上跑着一个内部 HTTP 服务器，对外提供一组供人阅读的状态页。状态页展示计算进度，比如已完成多少任务、有多少任务在进行中、输入字节数、中间数据字节数、输出字节数、处理速率等。页面上还有指向各任务产生的标准错误、标准输出文件的链接。用户可以用这些数据预估计算还要多久，以及是否需要给这次计算加资源。这些页面也可以用来判断计算何时比预期慢得多。

> In addition, the top-level status page shows which workers have failed, and which map and reduce tasks they were processing when they failed. This information is useful when attempting to diagnose bugs in the user code.

此外，顶层状态页会显示哪些 worker 失败过，以及失败时它们正在处理哪些 map 和 reduce 任务。在排查用户代码里的 bug 时，这些信息很有用。

### 4.9 Counters · 计数器

> The MapReduce library provides a counter facility to count occurrences of various events. For example, user code may want to count total number of words processed or the number of German documents indexed, etc.

MapReduce 库提供一个计数器设施，用来统计各类事件的发生次数。例如用户代码可能想统计已处理词的总数，或者已建立索引的德语文档数量等。

> To use this facility, user code creates a named counter object and then increments the counter appropriately in the Map and/or Reduce function. For example:

要使用这个设施，用户代码先创建一个带名字的计数器对象，然后在 Map 或 Reduce 函数里适时递增。例如：

```
Counter* uppercase;
uppercase = GetCounter("uppercase");

map(String name, String contents):
    for each word w in contents:
        if (IsCapitalized(w)):
            uppercase->Increment();
        EmitIntermediate(w, "1");
```

> The counter values from individual worker machines are periodically propagated to the master (piggybacked on the ping response). The master aggregates the counter values from successful map and reduce tasks and returns them to the user code when the MapReduce operation is completed. The current counter values are also displayed on the master status page so that a human can watch the progress of the live computation. When aggregating counter values, the master eliminates the effects of duplicate executions of the same map or reduce task to avoid double counting. (Duplicate executions can arise from our use of backup tasks and from re-execution of tasks due to failures.)

各台 worker 机器上的计数器值会周期性地传给 master（搭在 ping 响应上顺带带回）。master 汇总成功完成的 map 和 reduce 任务的计数器值，并在 MapReduce 操作结束时把它们返回给用户代码。当前计数器值也会显示在 master 状态页上，方便人观察实时计算的进度。汇总计数器值时，master 会消除同一个 map 或 reduce 任务重复执行带来的影响，避免重复计数。（重复执行可能来自我们启用的备份任务，也可能来自因故障而重新执行的任务。）

> Some counter values are automatically maintained by the MapReduce library, such as the number of input key/value pairs processed and the number of output key/value pairs produced.

有些计数器值由 MapReduce 库自动维护，比如已处理的输入 key/value 对数量和已产出的输出 key/value 对数量。

> Users have found the counter facility useful for sanity checking the behavior of MapReduce operations. For example, in some MapReduce operations, the user code may want to ensure that the number of output pairs produced exactly equals the number of input pairs processed, or that the fraction of German documents processed is within some tolerable fraction of the total number of documents processed.

用户发现计数器设施在给 MapReduce 操作做正确性检查时很有用。例如在某些 MapReduce 操作里，用户代码可能想确保产出的输出对数量恰好等于已处理的输入对数量，或者确保已处理的德语文档占比落在已处理文档总数可容忍的比例之内。

---

## 5 Performance · 性能

> In this section we measure the performance of MapReduce on two computations running on a large cluster of machines. One computation searches through approximately one terabyte of data looking for a particular pattern. The other computation sorts approximately one terabyte of data.

本节测量 MapReduce 在一个大型机器集群上跑两个计算时的性能。一个计算在约 1 TB 数据中搜索某个特定模式，另一个计算对约 1 TB 数据排序。

> These two programs are representative of a large subset of the real programs written by users of MapReduce - one class of programs shuffles data from one representation to another, and another class extracts a small amount of interesting data from a large data set.

这两个程序代表了 MapReduce 用户所写真实程序中的很大一部分：一类程序把数据从一种表示搬成另一种表示，另一类程序从大数据集里抽取少量有趣的数据。

### 5.1 Cluster Configuration · 集群配置

> All of the programs were executed on a cluster that consisted of approximately 1800 machines. Each machine had two 2GHz Intel Xeon processors with Hyper-Threading enabled, 4GB of memory, two 160GB IDE disks, and a gigabit Ethernet link. The machines were arranged in a two-level tree-shaped switched network with approximately 100-200 Gbps of aggregate bandwidth available at the root. All of the machines were in the same hosting facility and therefore the round-trip time between any pair of machines was less than a millisecond.

所有程序都在一个约 1800 台机器的集群上执行。每台机器配两颗开启超线程的 2 GHz Intel Xeon 处理器、4 GB 内存、两块 160 GB IDE 磁盘和一条千兆以太网链路。机器组织成两级树形交换网络，根部可用聚合带宽约 100–200 Gbps。所有机器都在同一个机房内，因此任意两台机器之间的往返时间都不到 1 毫秒。

> Out of the 4GB of memory, approximately 1-1.5GB was reserved by other tasks running on the cluster. The programs were executed on a weekend afternoon, when the CPUs, disks, and network were mostly idle.

4 GB 内存中约有 1–1.5 GB 被集群上运行的其他任务占用。程序在周末下午执行，那时 CPU、磁盘和网络基本空闲。

### 5.2 Grep · 模式搜索

> The grep program scans through 10^10 100-byte records, searching for a relatively rare three-character pattern (the pattern occurs in 92,337 records). The input is split into approximately 64MB pieces (M = 15000), and the entire output is placed in one file (R = 1).

grep 程序扫描 `10^10` 条 100 字节记录，搜索一个相对罕见的三个字符的模式（该模式在 92,337 条记录中出现）。输入被切成约 64 MB 的片（M = 15000），全部输出放在一个文件里（R = 1）。

> Figure 2 shows the progress of the computation over time. The Y-axis shows the rate at which the input data is scanned. The rate gradually picks up as more machines are assigned to this MapReduce computation, and peaks at over 30 GB/s when 1764 workers have been assigned. As the map tasks finish, the rate starts dropping and hits zero about 80 seconds into the computation. The entire computation takes approximately 150 seconds from start to finish. This includes about a minute of startup overhead. The overhead is due to the propagation of the program to all worker machines, and delays interacting with GFS to open the set of 1000 input files and to get the information needed for the locality optimization.

图 2 展示了这次计算随时间的推进过程。纵轴是输入数据的扫描速率。随着分配给这次 MapReduce 计算的机器越来越多，速率逐渐上升，在分配了 1764 个 worker 时达到峰值，超过 30 GB/s。随着 map 任务陆续完成，速率开始下降，在计算进行到约 80 秒时降到零。整次计算从开始到结束约需 150 秒，其中包含约一分钟的启动开销。这部分开销来自把程序分发到所有 worker 机器，以及与 GFS 交互以打开 1000 个输入文件、取回局部性优化所需信息的延迟。

图 2：传输速率随时间变化（Data transfer rate over time）。原文图 2 是一条以秒为横轴（0–100 秒）、输入速率为纵轴（0–30000 MB/s）的折线图，读数要点：

| 阶段 | 读数 |
|------|------|
| 速率爬升 | 随 worker 陆续加入，输入速率从 0 逐步抬升 |
| 峰值 | 分配 1764 个 worker 时超过 30 GB/s（约 30000 MB/s） |
| 回落 | map 任务陆续完成，速率下降 |
| 归零 | 计算进行到约 80 秒时降为 0 |
| 总耗时 | 约 150 秒，其中约 1 分钟为启动开销 |

### 5.3 Sort · 排序

> The sort program sorts 10^10 100-byte records (approximately 1 terabyte of data). This program is modeled after the TeraSort benchmark [10].

sort 程序对 `10^10` 条 100 字节记录排序（约 1 TB 数据）。这个程序仿照 TeraSort 基准 [10] 编写。

> The sorting program consists of less than 50 lines of user code. A three-line Map function extracts a 10-byte sorting key from a text line and emits the key and the original text line as the intermediate key/value pair. We used a built-in Identity function as the Reduce operator. This functions passes the intermediate key/value pair unchanged as the output key/value pair. The final sorted output is written to a set of 2-way replicated GFS files (i.e., 2 terabytes are written as the output of the program).

这个排序程序的用户代码不到 50 行。一个三行的 Map 函数从文本行中抽出 10 字节的排序键，把该键与原始文本行作为中间 key/value 对发射出去。Reduce 算子用的是内置的 Identity 函数：它把中间 key/value 对原样作为输出 key/value 对传出。最终排好序的输出写到一组两份副本的 GFS 文件里（也就是说，这个程序写出了 2 TB 的输出）。

> As before, the input data is split into 64MB pieces (M = 15000). We partition the sorted output into 4000 files (R = 4000). The partitioning function uses the initial bytes of the key to segregate it into one of R pieces. Our partitioning function for this benchmark has built-in knowledge of the distribution of keys. In a general sorting program, we would add a pre-pass MapReduce operation that would collect a sample of the keys and use the distribution of the sampled keys to compute splitpoints for the final sorting pass.

和前面一样，输入数据被切成 64 MB 的片（M = 15000）。我们把排好序的输出分成 4000 个文件（R = 4000）。分区函数用 key 的起始字节把它分到 R 份中的某一份。这个基准所用的分区函数内置了 key 分布的先验知识。在通用的排序程序里，我们会额外加一次前置（pre-pass）MapReduce 操作：先采集一批 key 样本，再用样本 key 的分布算出最终排序趟的分割点。

> Figure 3 (a) shows the progress of a normal execution of the sort program. The top-left graph shows the rate at which input is read. The rate peaks at about 13 GB/s and dies off fairly quickly since all map tasks finish before 200 seconds have elapsed. Note that the input rate is less than for grep. This is because the sort map tasks spend about half their time and I/O bandwidth writing intermediate output to their local disks. The corresponding intermediate output for grep had negligible size.

图 3(a) 展示 sort 程序一次正常执行的过程。左上角的图是输入读取速率，峰值约 13 GB/s，并且下降得相当快，因为所有 map 任务在 200 秒之内就都完成了。注意输入速率低于 grep：原因是 sort 的 map 任务把约一半的时间和 I/O 带宽花在往本地磁盘写中间输出上，而 grep 对应的中间输出规模可以忽略不计。

> The middle-left graph shows the rate at which data is sent over the network from the map tasks to the reduce tasks. This shuffling starts as soon as the first map task completes. The first hump in the graph is for the first batch of approximately 1700 reduce tasks (the entire MapReduce was assigned about 1700 machines, and each machine executes at most one reduce task at a time). Roughly 300 seconds into the computation, some of these first batch of reduce tasks finish and we start shuffling data for the remaining reduce tasks. All of the shuffling is done about 600 seconds into the computation. The bottom-left graph shows the rate at which sorted data is written to the final output files by the reduce tasks. There is a delay between the end of the first shuffling period and the start of the writing period because the machines are busy sorting the intermediate data. The writes continue at a rate of about 2-4 GB/s for a while. All of the writes finish about 850 seconds into the computation. Including startup overhead, the entire computation takes 891 seconds. This is similar to the current best reported result of 1057 seconds for the TeraSort benchmark [18].

左中图是从 map 任务经网络发往 reduce 任务的数据速率。数据搬运（shuffle）在第一个 map 任务完成时就开始了。图上第一个隆起对应第一批约 1700 个 reduce 任务（整次 MapReduce 分配到了约 1700 台机器，每台机器同一时刻最多执行一个 reduce 任务）。计算进行到大约 300 秒时，这批 reduce 任务中有一部分完成，我们开始为剩余的 reduce 任务搬运数据。全部搬运在计算进行到约 600 秒时结束。左下角图是 reduce 任务把已排序数据写入最终输出文件的速率。第一段搬运结束与写入开始之间有一段延迟，因为机器正忙着对中间数据排序。写入在一段时间内维持约 2–4 GB/s 的速率，全部写完大约在计算进行到 850 秒时。加上启动开销，整次计算耗时 891 秒。这与 TeraSort 基准当时公布的最好成绩 1057 秒 [18] 大体相当。

> A few things to note: the input rate is higher than the shuffle rate and the output rate because of our locality optimization - most data is read from a local disk and bypasses our relatively bandwidth constrained network. The shuffle rate is higher than the output rate because the output phase writes two copies of the sorted data (we make two replicas of the output for reliability and availability reasons). We write two replicas because that is the mechanism for reliability and availability provided by our underlying file system. Network bandwidth requirements for writing data would be reduced if the underlying file system used erasure coding [14] rather than replication.

有几点值得注意：输入速率高于 shuffle 速率和输出速率，这是局部性优化的结果——大部分数据从本地磁盘读取，绕开了我们带宽相对受限的网络。shuffle 速率高于输出速率，是因为输出阶段要写两份排序数据（出于可靠性和可用性考虑，我们对输出做了两份副本）。之所以写两份副本，是因为这是我们底层文件系统提供的可靠性与可用性机制。如果底层文件系统改用纠删码 [14] 而不是副本，写数据的网络带宽需求会更低。

图 3：sort 程序不同执行方式下的传输速率（Data transfer rates over time for different executions of the sort program）。原文图 3 是三块并排的三联折线图，横轴为秒（0–1000 以上），纵轴从上到下依次为 Input / Shuffle / Output（MB/s，0–20000）。三块的读数要点：

| 子图 | 配置 | 读数要点 | 总耗时 |
|------|------|----------|--------|
| (a) Normal execution | 正常执行 | 输入峰值约 13 GB/s，200 秒前 map 全部结束；shuffle 第一个隆起对应约 1700 个 reduce 任务，300 秒起部分完成，600 秒搬运完毕；输出写入维持 2–4 GB/s，850 秒写完 | 891 秒 |
| (b) No backup tasks | 关闭备份任务 | 流程与 (a) 类似，但尾部很长，几乎无写入活动；960 秒时除 5 个 reduce 任务外全部完成，这最后几个掉队者又拖了 300 秒 | 1283 秒（+44%） |
| (c) 200 tasks killed | 杀掉 200 个 worker | 被杀掉的 200 个进程（共 1746 个）导致已完成的部分 map 工作消失，输入速率出现负值，需要重做；重做很快完成 | 933 秒（+5%） |

### 5.4 Effect of Backup Tasks · 备份任务的效果

> In Figure 3 (b), we show an execution of the sort program with backup tasks disabled. The execution flow is similar to that shown in Figure 3 (a), except that there is a very long tail where hardly any write activity occurs. After 960 seconds, all except 5 of the reduce tasks are completed. However these last few stragglers don't finish until 300 seconds later. The entire computation takes 1283 seconds, an increase of 44% in elapsed time.

图 3(b) 展示的是关闭备份任务后 sort 程序的一次执行。执行流程与图 3(a) 类似，只是尾部拖得很长，期间几乎没有任何写入活动。960 秒之后，除 5 个之外的所有 reduce 任务都已完成；但最后这几个掉队者又过了 300 秒才结束。整次计算耗时 1283 秒，墙上时间增加了 44%。

### 5.5 Machine Failures · 机器故障

> In Figure 3 (c), we show an execution of the sort program where we intentionally killed 200 out of 1746 worker processes several minutes into the computation. The underlying cluster scheduler immediately restarted new worker processes on these machines (since only the processes were killed, the machines were still functioning properly).

图 3(c) 展示的是这样一次 sort 执行：在计算进行几分钟后，我们有意杀掉了 1746 个 worker 进程中的 200 个。底层集群调度器立即在这些机器上重启了新的 worker 进程（因为只是进程被杀，机器本身仍然正常工作）。

> The worker deaths show up as a negative input rate since some previously completed map work disappears (since the corresponding map workers were killed) and needs to be redone. The re-execution of this map work happens relatively quickly. The entire computation finishes in 933 seconds including startup overhead (just an increase of 5% over the normal execution time).

worker 的死亡表现为输入速率出现负值，因为一部分此前已完成的 map 工作随之消失（对应的 map worker 被杀掉了），需要重做。这部分 map 工作的重新执行完成得相对快。整次计算含启动开销在 933 秒内结束（相比正常执行只增加 5%）。

---

## 6 Experience · 实践经验

> We wrote the first version of the MapReduce library in February of 2003, and made significant enhancements to it in August of 2003, including the locality optimization, dynamic load balancing of task execution across worker machines, etc. Since that time, we have been pleasantly surprised at how broadly applicable the MapReduce library has been for the kinds of problems we work on. It has been used across a wide range of domains within Google, including:

我们在 2003 年 2 月写出 MapReduce 库的第一版，并在 2003 年 8 月对它做了重要增强，包括局部性优化、跨 worker 机器的任务执行动态负载均衡等。从那时起，我们惊喜地发现 MapReduce 库对我们所处理的那类问题适用面如此之广。它被用在 Google 内部众多不同领域，包括：

> • large-scale machine learning problems,
> • clustering problems for the Google News and Froogle products,
> • extraction of data used to produce reports of popular queries (e.g. Google Zeitgeist),
> • extraction of properties of web pages for new experiments and products (e.g. extraction of geographical locations from a large corpus of web pages for localized search), and
> • large-scale graph computations.

- 大规模机器学习问题；
- Google News 和 Froogle 产品的聚类问题；
- 用于生成热门查询报告（例如 Google Zeitgeist）的数据抽取；
- 为新实验和新产品抽取网页属性（例如从大规模网页语料中抽取地理位置，用于本地化搜索）；
- 大规模图计算。

> Figure 4 shows the significant growth in the number of separate MapReduce programs checked into our primary source code management system over time, from 0 in early 2003 to almost 900 separate instances as of late September 2004. MapReduce has been so successful because it makes it possible to write a simple program and run it efficiently on a thousand machines in the course of half an hour, greatly speeding up the development and prototyping cycle. Furthermore, it allows programmers who have no experience with distributed and/or parallel systems to exploit large amounts of resources easily.

图 4 展示了签入我们主源码管理系统的独立 MapReduce 程序数量随时间的显著增长：从 2003 年初的 0 个，增长到 2004 年 9 月下旬的近 900 个独立实例。MapReduce 之所以如此成功，是因为它让一个人可以写一个简单程序，并在半小时内高效地跑在一千台机器上，极大加快了开发与原型迭代周期。此外，它让没有分布式或并行系统经验的程序员也能轻松用上大量资源。

图 4：MapReduce 实例数随时间变化（MapReduce instances over time）。原文图 4 是一条以 2003/03 至 2004/09 为横轴、以源码树中实例数（0–1000）为纵轴的折线图，读数要点：起点为 0，终点接近 900，整段区间单调上升。

> At the end of each job, the MapReduce library logs statistics about the computational resources used by the job. In Table 1, we show some statistics for a subset of MapReduce jobs run at Google in August 2004.

每个作业结束时，MapReduce 库会记录该作业所用计算资源的统计信息。表 1 给出了 2004 年 8 月在 Google 运行的 MapReduce 作业中一部分作业的统计值。

表 1：2004 年 8 月运行的 MapReduce 作业（MapReduce jobs run in August 2004）

| 指标 | 数值 |
|------|------|
| 作业数 | 29,423 |
| 作业平均完成时间 | 634 秒 |
| 消耗机器天数 | 79,186 机器日 |
| 读取的输入数据 | 3,288 TB |
| 产生的中间数据 | 758 TB |
| 写出的输出数据 | 193 TB |
| 每作业平均 worker 机器数 | 157 |
| 每作业平均 worker 死亡数 | 1.2 |
| 每作业平均 map 任务数 | 3,351 |
| 每作业平均 reduce 任务数 | 55 |
| 不同的 map 实现数 | 395 |
| 不同的 reduce 实现数 | 269 |
| 不同的 map/reduce 组合数 | 426 |

### 6.1 Large-Scale Indexing · 大规模索引

> One of our most significant uses of MapReduce to date has been a complete rewrite of the production indexing system that produces the data structures used for the Google web search service. The indexing system takes as input a large set of documents that have been retrieved by our crawling system, stored as a set of GFS files. The raw contents for these documents are more than 20 terabytes of data. The indexing process runs as a sequence of five to ten MapReduce operations. Using MapReduce (instead of the ad-hoc distributed passes in the prior version of the indexing system) has provided several benefits:

迄今为止 MapReduce 最有份量的一次应用，是完全重写了生产索引系统——这套系统产出的数据结构被 Google 网页搜索服务使用。索引系统以爬取系统抓回的一大批文档为输入，这些文档以一组 GFS 文件的形式存放。这些文档的原始内容超过 20 TB。整个索引过程由五到十次 MapReduce 操作串联而成。改用 MapReduce（取代旧版索引系统中临时拼凑的分布式多趟处理）带来了若干好处：

> • The indexing code is simpler, smaller, and easier to understand, because the code that deals with fault tolerance, distribution and parallelization is hidden within the MapReduce library. For example, the size of one phase of the computation dropped from approximately 3800 lines of C++ code to approximately 700 lines when expressed using MapReduce.

- 索引代码更简单、更小、更容易理解，因为处理容错、分发和并行化的代码被藏进了 MapReduce 库里。例如，计算中某一阶段的规模从约 3800 行 C++ 代码下降到用 MapReduce 表达后的约 700 行。

> • The performance of the MapReduce library is good enough that we can keep conceptually unrelated computations separate, instead of mixing them together to avoid extra passes over the data. This makes it easy to change the indexing process. For example, one change that took a few months to make in our old indexing system took only a few days to implement in the new system.

- MapReduce 库的性能足够好，使我们能把概念上互不相关的计算分开，而不必为了少跑几趟数据就把它们搅在一起。这让索引过程容易改动。例如，有一个改动在旧索引系统里要花几个月才能完成，在新系统里只花几天就实现了。

> • The indexing process has become much easier to operate, because most of the problems caused by machine failures, slow machines, and networking hiccups are dealt with automatically by the MapReduce library without operator intervention. Furthermore, it is easy to improve the performance of the indexing process by adding new machines to the indexing cluster.

- 索引过程变得容易运维得多，因为机器故障、慢机器和网络抖动引起的大部分问题都由 MapReduce 库自动处理，不需要运维介入。此外，往索引集群里加机器就能轻松提升索引过程的性能。

---

## 7 Related Work · 相关工作

> Many systems have provided restricted programming models and used the restrictions to parallelize the computation automatically. For example, an associative function can be computed over all prefixes of an N element array in log N time on N processors using parallel prefix computations [6, 9, 13]. MapReduce can be considered a simplification and distillation of some of these models based on our experience with large real-world computations. More significantly, we provide a fault-tolerant implementation that scales to thousands of processors. In contrast, most of the parallel processing systems have only been implemented on smaller scales and leave the details of handling machine failures to the programmer.

许多系统都提供受限的编程模型，并利用这些限制自动并行化计算。例如，利用并行前缀计算 [6, 9, 13]，一个结合性函数可以在 N 个处理器上用 `log N` 时间在整个 N 元素数组的所有前缀上算出。MapReduce 可以看作是其中一些模型的简化和提纯，依据是我们对真实大规模计算的经验。更关键的是，我们提供的实现具备容错能力，可扩展到数千个处理器。相比之下，多数并行处理系统只在较小规模上实现，把处理机器故障的细节留给了程序员。

> Bulk Synchronous Programming [17] and some MPI primitives [11] provide higher-level abstractions that make it easier for programmers to write parallel programs. A key difference between these systems and MapReduce is that MapReduce exploits a restricted programming model to parallelize the user program automatically and to provide transparent fault-tolerance.

整体同步并行编程（Bulk Synchronous Programming）[17] 和某些 MPI 原语 [11] 提供了更高级的抽象，让程序员更容易写并行程序。这些系统与 MapReduce 的一个关键差别是：MapReduce 利用受限的编程模型来自动并行化用户程序，并提供透明的容错。

> Our locality optimization draws its inspiration from techniques such as active disks [12, 15], where computation is pushed into processing elements that are close to local disks, to reduce the amount of data sent across I/O subsystems or the network. We run on commodity processors to which a small number of disks are directly connected instead of running directly on disk controller processors, but the general approach is similar.

我们的局部性优化灵感来自主动磁盘（active disks）[12, 15] 之类的技术：把计算下推到靠近本地磁盘的处理单元上，以减少经 I/O 子系统或网络传输的数据量。我们是在直连少量磁盘的通用处理器上运行，而不是直接在磁盘控制器处理器上运行，但总体思路是相似的。

> Our backup task mechanism is similar to the eager scheduling mechanism employed in the Charlotte System [3]. One of the shortcomings of simple eager scheduling is that if a given task causes repeated failures, the entire computation fails to complete. We fix some instances of this problem with our mechanism for skipping bad records.

我们的备份任务机制与 Charlotte 系统 [3] 采用的急切调度（eager scheduling）机制相似。简单急切调度的缺点之一是：如果某个任务反复导致失败，整次计算就无法完成。我们通过跳过坏记录的机制，修复了这一问题的一部分实例。

> The MapReduce implementation relies on an in-house cluster management system that is responsible for distributing and running user tasks on a large collection of shared machines. Though not the focus of this paper, the cluster management system is similar in spirit to other systems such as Condor [16].

MapReduce 实现依赖一套自研的集群管理系统，它负责在一大批共享机器上分发并运行用户任务。虽然这不是本文的重点，但这套集群管理系统在思路上与 Condor [16] 等系统相似。

> The sorting facility that is a part of the MapReduce library is similar in operation to NOW-Sort [1]. Source machines (map workers) partition the data to be sorted and send it to one of R reduce workers. Each reduce worker sorts its data locally (in memory if possible). Of course NOW-Sort does not have the user-definable Map and Reduce functions that make our library widely applicable.

MapReduce 库中自带的排序设施，在工作方式上与 NOW-Sort [1] 相似。源机器（map worker）对待排序数据做分区，并把数据送到 R 个 reduce worker 中的某一个；每个 reduce worker 在本地排序自己的数据（尽可能在内存中完成）。当然，NOW-Sort 没有那对让我们的库具备广泛适用性的、由用户定义的 Map 和 Reduce 函数。

> River [2] provides a programming model where processes communicate with each other by sending data over distributed queues. Like MapReduce, the River system tries to provide good average case performance even in the presence of non-uniformities introduced by heterogeneous hardware or system perturbations. River achieves this by careful scheduling of disk and network transfers to achieve balanced completion times. MapReduce has a different approach. By restricting the programming model, the MapReduce framework is able to partition the problem into a large number of fine-grained tasks. These tasks are dynamically scheduled on available workers so that faster workers process more tasks. The restricted programming model also allows us to schedule redundant executions of tasks near the end of the job which greatly reduces completion time in the presence of non-uniformities (such as slow or stuck workers).

River [2] 提供了一套编程模型，其中各进程通过分布式队列发送数据来相互通信。与 MapReduce 一样，River 系统也试图在异构硬件或系统扰动引入的不均匀性存在时，仍能提供良好的平均情形性能。River 的做法是精心调度磁盘和网络传输，以取得均衡的完成时间。MapReduce 走的是另一条路：通过限制编程模型，MapReduce 框架能把问题切成大量细粒度任务；这些任务被动态调度到可用 worker 上，使更快的 worker 处理更多任务。受限的编程模型还让我们能在作业接近尾声时调度任务的冗余执行，这在存在不均匀性（比如慢 worker 或卡住的 worker）时能大幅缩短完成时间。

> BAD-FS [5] has a very different programming model from MapReduce, and unlike MapReduce, is targeted to the execution of jobs across a wide-area network. However, there are two fundamental similarities. (1) Both systems use redundant execution to recover from data loss caused by failures. (2) Both use locality-aware scheduling to reduce the amount of data sent across congested network links.

BAD-FS [5] 的编程模型与 MapReduce 差别很大，而且与 MapReduce 不同，它面向的是跨广域网执行作业。但两者有两个根本相似之处：（1）都用冗余执行来从故障导致的数据丢失中恢复；（2）都用感知局部性的调度来减少经拥塞网络链路传输的数据量。

> TACC [7] is a system designed to simplify construction of highly-available networked services. Like MapReduce, it relies on re-execution as a mechanism for implementing fault-tolerance.

TACC [7] 是一套旨在简化高可用网络服务构建的系统。与 MapReduce 一样，它依靠重新执行作为实现容错的机制。

---

## 8 Conclusions · 结论

> The MapReduce programming model has been successfully used at Google for many different purposes. We attribute this success to several reasons. First, the model is easy to use, even for programmers without experience with parallel and distributed systems, since it hides the details of parallelization, fault-tolerance, locality optimization, and load balancing. Second, a large variety of problems are easily expressible as MapReduce computations. For example, MapReduce is used for the generation of data for Google's production web search service, for sorting, for data mining, for machine learning, and many other systems. Third, we have developed an implementation of MapReduce that scales to large clusters of machines comprising thousands of machines. The implementation makes efficient use of these machine resources and therefore is suitable for use on many of the large computational problems encountered at Google.

MapReduce 编程模型已在 Google 成功用于许多不同目的。我们把这一成功归因于几点。第一，模型易用，即便对没有并行和分布式系统经验的程序员也是如此，因为它把并行化、容错、局部性优化和负载均衡的细节都藏了起来。第二，各种各样的问题都能轻松表达成 MapReduce 计算。例如 MapReduce 被用于为 Google 生产环境的网页搜索服务生成数据，也用于排序、数据挖掘、机器学习以及许多其他系统。第三，我们开发的 MapReduce 实现可扩展到由数千台机器组成的大集群。该实现能高效利用这些机器资源，因此适用于 Google 遇到的许多大规模计算问题。

> We have learned several things from this work. First, restricting the programming model makes it easy to parallelize and distribute computations and to make such computations fault-tolerant. Second, network bandwidth is a scarce resource. A number of optimizations in our system are therefore targeted at reducing the amount of data sent across the network: the locality optimization allows us to read data from local disks, and writing a single copy of the intermediate data to local disk saves network bandwidth. Third, redundant execution can be used to reduce the impact of slow machines, and to handle machine failures and data loss.

我们从这项工作中得到几点认识。第一，限制编程模型能让计算易于并行化、易于分发，也易于做到容错。第二，网络带宽是稀缺资源。因此我们系统里的一系列优化都指向减少经网络传输的数据量：局部性优化让我们能从本地磁盘读数据，把中间数据的单份副本写到本地磁盘则节省了网络带宽。第三，冗余执行可以用来减轻慢机器的影响，并处理机器故障和数据丢失。

---

## Acknowledgements · 致谢

> Josh Levenberg has been instrumental in revising and extending the user-level MapReduce API with a number of new features based on his experience with using MapReduce and other people's suggestions for enhancements. MapReduce reads its input from and writes its output to the Google File System [8]. We would like to thank Mohit Aron, Howard Gobioff, Markus Gutschke, David Kramer, Shun-Tak Leung, and Josh Redstone for their work in developing GFS. We would also like to thank Percy Liang and Olcan Sercinoglu for their work in developing the cluster management system used by MapReduce. Mike Burrows, Wilson Hsieh, Josh Levenberg, Sharon Perl, Rob Pike, and Debby Wallach provided helpful comments on earlier drafts of this paper. The anonymous OSDI reviewers, and our shepherd, Eric Brewer, provided many useful suggestions of areas where the paper could be improved. Finally, we thank all the users of MapReduce within Google's engineering organization for providing helpful feedback, suggestions, and bug reports.

Josh Levenberg 在修订和扩展用户层 MapReduce API 方面起了关键作用，他依据自己使用 MapReduce 的经验以及其他人提出的增强建议，加入了许多新特性。MapReduce 的输入来自 Google File System，输出也写入其中 [8]。我们要感谢 Mohit Aron、Howard Gobioff、Markus Gutschke、David Kramer、Shun-Tak Leung 和 Josh Redstone 在开发 GFS 上的工作。我们也要感谢 Percy Liang 和 Olcan Sercinoglu 在开发 MapReduce 所用集群管理系统上的工作。Mike Burrows、Wilson Hsieh、Josh Levenberg、Sharon Perl、Rob Pike 和 Debby Wallach 对本文早期草稿给出了有益的意见。匿名 OSDI 审稿人以及我们的 shepherd Eric Brewer 提供了许多关于论文可改进之处的好建议。最后，我们感谢 Google 工程组织内所有 MapReduce 使用者提供的反馈、建议和 bug 报告。

---

## References · 参考文献

> [1] Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, David E. Culler, Joseph M. Hellerstein, and David A. Patterson. High-performance sorting on networks of workstations. In Proceedings of the 1997 ACM SIGMOD International Conference on Management of Data, Tucson, Arizona, May 1997.

[1] Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, David E. Culler, Joseph M. Hellerstein, David A. Patterson. High-performance sorting on networks of workstations. 1997 年 ACM SIGMOD 数据管理国际会议论文集，亚利桑那州图森，1997 年 5 月。

> [2] Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, and Kathy Yelick. Cluster I/O with River: Making the fast case common. In Proceedings of the Sixth Workshop on Input/Output in Parallel and Distributed Systems (IOPADS '99), pages 10-22, Atlanta, Georgia, May 1999.

[2] Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, Kathy Yelick. Cluster I/O with River: Making the fast case common. 第 6 届并行与分布式系统输入／输出研讨会（IOPADS '99）论文集，第 10–22 页，佐治亚州亚特兰大，1999 年 5 月。

> [3] Arash Baratloo, Mehmet Karaul, Zvi Kedem, and Peter Wyckoff. Charlotte: Metacomputing on the web. In Proceedings of the 9th International Conference on Parallel and Distributed Computing Systems, 1996.

[3] Arash Baratloo, Mehmet Karaul, Zvi Kedem, Peter Wyckoff. Charlotte: Metacomputing on the web. 第 9 届并行与分布式计算系统国际会议论文集，1996 年。

> [4] Luiz A. Barroso, Jeffrey Dean, and Urs Hölzle. Web search for a planet: The Google cluster architecture. IEEE Micro, 23(2):22-28, April 2003.

[4] Luiz A. Barroso, Jeffrey Dean, Urs Hölzle. Web search for a planet: The Google cluster architecture. IEEE Micro, 23(2):22–28, 2003 年 4 月。

> [5] John Bent, Douglas Thain, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, and Miron Livny. Explicit control in a batch-aware distributed file system. In Proceedings of the 1st USENIX Symposium on Networked Systems Design and Implementation NSDI, March 2004.

[5] John Bent, Douglas Thain, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, Miron Livny. Explicit control in a batch-aware distributed file system. 第 1 届 USENIX 网络系统设计与实现研讨会（NSDI）论文集，2004 年 3 月。

> [6] Guy E. Blelloch. Scans as primitive parallel operations. IEEE Transactions on Computers, C-38(11), November 1989.

[6] Guy E. Blelloch. Scans as primitive parallel operations. IEEE Transactions on Computers, C-38(11), 1989 年 11 月。

> [7] Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, and Paul Gauthier. Cluster-based scalable network services. In Proceedings of the 16th ACM Symposium on Operating System Principles, pages 78-91, Saint-Malo, France, 1997.

[7] Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, Paul Gauthier. Cluster-based scalable network services. 第 16 届 ACM 操作系统原理研讨会论文集，第 78–91 页，法国圣马洛，1997 年。

> [8] Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. The Google file system. In 19th Symposium on Operating Systems Principles, pages 29-43, Lake George, New York, 2003.

[8] Sanjay Ghemawat, Howard Gobioff, Shun-Tak Leung. The Google file system. 第 19 届操作系统原理研讨会，第 29–43 页，纽约州乔治湖，2003 年。

> [9] S. Gorlatch. Systematic efficient parallelization of scan and other list homomorphisms. In L. Bouge, P. Fraigniaud, A. Mignotte, and Y. Robert, editors, Euro-Par'96. Parallel Processing, Lecture Notes in Computer Science 1124, pages 401-408. Springer-Verlag, 1996.

[9] S. Gorlatch. Systematic efficient parallelization of scan and other list homomorphisms. 载 L. Bouge, P. Fraigniaud, A. Mignotte, Y. Robert 编, Euro-Par'96. Parallel Processing, Lecture Notes in Computer Science 1124, 第 401–408 页. Springer-Verlag, 1996 年。

> [10] Jim Gray. Sort benchmark home page. http://research.microsoft.com/barc/SortBenchmark/.

[10] Jim Gray. Sort benchmark home page. http://research.microsoft.com/barc/SortBenchmark/。

> [11] William Gropp, Ewing Lusk, and Anthony Skjellum. Using MPI: Portable Parallel Programming with the Message-Passing Interface. MIT Press, Cambridge, MA, 1999.

[11] William Gropp, Ewing Lusk, Anthony Skjellum. Using MPI: Portable Parallel Programming with the Message-Passing Interface. MIT Press, 马萨诸塞州剑桥，1999 年。

> [12] L. Huston, R. Sukthankar, R. Wickremesinghe, M. Satyanarayanan, G. R. Ganger, E. Riedel, and A. Ailamaki. Diamond: A storage architecture for early discard in interactive search. In Proceedings of the 2004 USENIX File and Storage Technologies FAST Conference, April 2004.

[12] L. Huston, R. Sukthankar, R. Wickremesinghe, M. Satyanarayanan, G. R. Ganger, E. Riedel, A. Ailamaki. Diamond: A storage architecture for early discard in interactive search. 2004 年 USENIX 文件与存储技术会议（FAST）论文集，2004 年 4 月。

> [13] Richard E. Ladner and Michael J. Fischer. Parallel prefix computation. Journal of the ACM, 27(4):831-838, 1980.

[13] Richard E. Ladner, Michael J. Fischer. Parallel prefix computation. Journal of the ACM, 27(4):831–838, 1980。

> [14] Michael O. Rabin. Efficient dispersal of information for security, load balancing and fault tolerance. Journal of the ACM, 36(2):335-348, 1989.

[14] Michael O. Rabin. Efficient dispersal of information for security, load balancing and fault tolerance. Journal of the ACM, 36(2):335–348, 1989。

> [15] Erik Riedel, Christos Faloutsos, Garth A. Gibson, and David Nagle. Active disks for large-scale data processing. IEEE Computer, pages 68-74, June 2001.

[15] Erik Riedel, Christos Faloutsos, Garth A. Gibson, David Nagle. Active disks for large-scale data processing. IEEE Computer, 第 68–74 页, 2001 年 6 月。

> [16] Douglas Thain, Todd Tannenbaum, and Miron Livny. Distributed computing in practice: The Condor experience. Concurrency and Computation: Practice and Experience, 2004.

[16] Douglas Thain, Todd Tannenbaum, Miron Livny. Distributed computing in practice: The Condor experience. Concurrency and Computation: Practice and Experience, 2004 年。

> [17] L. G. Valiant. A bridging model for parallel computation. Communications of the ACM, 33(8):103-111, 1997.

[17] L. G. Valiant. A bridging model for parallel computation. Communications of the ACM, 33(8):103–111, 1997。（原文此处标为 1997 年，该文实际发表于 1990 年 8 月，此处按原文保留。）

> [18] Jim Wyllie. Spsort: How to sort a terabyte quickly. http://alme1.almaden.ibm.com/cs/spsort.pdf.

[18] Jim Wyllie. Spsort: How to sort a terabyte quickly. http://alme1.almaden.ibm.com/cs/spsort.pdf。

---

## Appendix A · 附录 A：词频统计

> This section contains a program that counts the number of occurrences of each unique word in a set of input files specified on the command line.

本附录给出一个程序，用来统计命令行指定的一组输入文件中每个唯一词的出现次数。

```cpp
#include "mapreduce/mapreduce.h"
// User's map function
class WordCounter : public Mapper {
 public:
  virtual void Map(const MapInput& input) {
    const string& text = input.value();
    const int n = text.size();
    for (int i = 0; i < n; ) {
      // Skip past leading whitespace
      while ((i < n) && isspace(text[i]))
        i++;

      // Find word end
      int start = i;
      while ((i < n) && !isspace(text[i]))
        i++;
      if (start < i)
        Emit(text.substr(start,i-start),"1");
    }
  }
};
REGISTER_MAPPER(WordCounter);

// User's reduce function
class Adder : public Reducer {
  virtual void Reduce(ReduceInput* input) {
    // Iterate over all entries with the
    // same key and add the values
    int64 value = 0;
    while (!input->done()) {
      value += StringToInt(input->value());
      input->NextValue();
    }

    // Emit sum for input->key()
    Emit(IntToString(value));
  }
};
REGISTER_REDUCER(Adder);

int main(int argc, char** argv) {
  ParseCommandLineFlags(argc, argv);

  MapReduceSpecification spec;

  // Store list of input files into "spec"
  for (int i = 1; i < argc; i++) {
    MapReduceInput* input = spec.add_input();
    input->set_format("text");
    input->set_filepattern(argv[i]);
    input->set_mapper_class("WordCounter");
  }

  // Specify the output files:
  // /gfs/test/freq-00000-of-00100
  // /gfs/test/freq-00001-of-00100
  // ...
  MapReduceOutput* out = spec.output();
  out->set_filebase("/gfs/test/freq");
  out->set_num_tasks(100);
  out->set_format("text");
  out->set_reducer_class("Adder");

  // Optional: do partial sums within map
  // tasks to save network bandwidth
  out->set_combiner_class("Adder");

  // Tuning parameters: use at most 2000
  // machines and 100 MB of memory per task
  spec.set_machines(2000);
  spec.set_map_megabytes(100);
  spec.set_reduce_megabytes(100);

  // Now run it
  MapReduceResult result;
  if (!MapReduce(spec, &result)) abort();

  // Done: 'result' structure contains info
  // about counters, time taken, number of
  // machines used, etc.

  return 0;
}
```

程序里的 `out->set_combiner_class("Adder")` 就是 4.3 节 Combiner 函数的用法：在 map 任务本地先做部分求和，减少经网络传输的数据量。

