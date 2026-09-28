---
title: "Bigtable 论文中英对照全文翻译（A Distributed Storage System for Structured Data）"
date: 2014-06-22T10:00:00+08:00
url: /2014/06/bigtable-paper-cn-en/
draft: false
tags: ["bigtable", "google", "distributed", "paper", "database"]
categories: ["tech"]
description: "Bigtable 论文 A Distributed Storage System for Structured Data 的完整中英对照翻译，共 11 节，含稀疏多维有序映射数据模型、列族与时间戳、tablet 三级定位层级、Chubby 与 tablet 分配、memtable 与 SSTable 的合并整理、布隆过滤器与提交日志实现，以及单机与扩展到 500 台 tablet server 的实测数据和 Google Analytics、Google Earth、Personalized Search 三个真实应用。"
---

这是 Bigtable 论文的完整中英对照翻译。原文 14 页，正文 11 节，参考文献 38 条。

体例：每段先列英文原文（引用块），紧接中文译文。图与表格按原文内容重绘；专业术语保留英文并附中文，原文的引用编号 [n] 对应文末参考文献。

<!--more-->

## 论文信息

| 项目 | 内容 |
|------|------|
| 标题 | Bigtable: A Distributed Storage System for Structured Data |
| 作者 | Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes, Robert E. Gruber |
| 机构 | Google, Inc. |
| 发表 | OSDI 2006（第 7 届 USENIX 操作系统设计与实现研讨会），2006 年 11 月，西雅图 |
| 篇幅 | 14 页，正文 11 节，参考文献 38 条 |
| 核心机制 | 稀疏多维有序映射、列族与时间戳、tablet 三级定位层级、Chubby 锁服务、memtable 与 SSTable、合并整理（compaction）、布隆过滤器、单提交日志 |
| 实测规模 | 截至 2006 年 8 月共有 388 个非测试集群、约 24,500 台 tablet server；14 个繁忙集群合计每秒超过 120 万次请求 |

---

## Abstract · 摘要

{{% bilingual %}}
Bigtable is a distributed storage system for managing structured data that is designed to scale to a very large size: petabytes of data across thousands of commodity servers. Many projects at Google store data in Bigtable, including web indexing, Google Earth, and Google Finance. These applications place very different demands on Bigtable, both in terms of data size (from URLs to web pages to satellite imagery) and latency requirements (from backend bulk processing to real-time data serving). Despite these varied demands, Bigtable has successfully provided a flexible, high-performance solution for all of these Google products. In this paper we describe the simple data model provided by Bigtable, which gives clients dynamic control over data layout and format, and we describe the design and implementation of Bigtable.
<!--col-->
Bigtable 是一个用于管理结构化数据的分布式存储系统，设计目标是扩展到非常大的规模：跨数千台廉价服务器的 PB 级数据。Google 内部的许多项目都把数据存在 Bigtable 里，包括网页索引、Google Earth 和 Google Finance。这些应用对 Bigtable 提出的要求差别极大，无论从数据规模上（从 URL 到网页到卫星影像）还是从延迟要求上（从后端批量处理到实时的数据服务）都是如此。尽管需求如此多样，Bigtable 仍然为所有这些 Google 产品提供了一个灵活、高性能的解决方案。本文描述 Bigtable 提供的简单数据模型——它让客户端能动态控制数据的布局与格式——以及 Bigtable 的设计与实现。
{{% /bilingual %}}

---

## 1 Introduction · 引言

{{% bilingual %}}
Over the last two and a half years we have designed, implemented, and deployed a distributed storage system for managing structured data at Google called Bigtable. Bigtable is designed to reliably scale to petabytes of data and thousands of machines. Bigtable has achieved several goals: wide applicability, scalability, high performance, and high availability. Bigtable is used by more than sixty Google products and projects, including Google Analytics, Google Finance, Orkut, Personalized Search, Writely, and Google Earth. These products use Bigtable for a variety of demanding workloads, which range from throughput-oriented batch-processing jobs to latency-sensitive serving of data to end users. The Bigtable clusters used by these products span a wide range of configurations, from a handful to thousands of servers, and store up to several hundred terabytes of data.
<!--col-->
在过去两年半里，我们设计、实现并部署了一套用于管理 Google 结构化数据的分布式存储系统，叫做 Bigtable。Bigtable 的设计目标是可靠地扩展到 PB 级数据和数千台机器。它达成了若干目标：广泛的适用性、可扩展性、高性能和高可用性。Bigtable 被六十多个 Google 产品和项目使用，包括 Google Analytics、Google Finance、Orkut、Personalized Search、Writely 和 Google Earth。这些产品把 Bigtable 用于各种严苛的负载，涵盖从面向吞吐量的批处理作业到对延迟敏感的终端用户数据服务。这些产品所用的 Bigtable 集群配置跨度很大，从几台到数千台服务器，存储的数据量最高达数百 TB。
{{% /bilingual %}}

{{% bilingual %}}
In many ways, Bigtable resembles a database: it shares many implementation strategies with databases. Parallel databases [14] and main-memory databases [13] have achieved scalability and high performance, but Bigtable provides a different interface than such systems. Bigtable does not support a full relational data model; instead, it provides clients with a simple data model that supports dynamic control over data layout and format, and allows clients to reason about the locality properties of the data represented in the underlying storage. Data is indexed using row and column names that can be arbitrary strings. Bigtable also treats data as uninterpreted strings, although clients often serialize various forms of structured and semi-structured data into these strings. Clients can control the locality of their data through careful choices in their schemas. Finally, Bigtable schema parameters let clients dynamically control whether to serve data out of memory or from disk.
<!--col-->
从很多方面看，Bigtable 像是一个数据库：它与数据库共享许多实现策略。并行数据库 [14] 和主存数据库 [13] 已经做到了可扩展与高性能，但 Bigtable 提供的接口与这类系统不同。Bigtable 不支持完整的关系数据模型；它提供的是一套简单的数据模型，支持客户端动态控制数据的布局与格式，并让客户端能对底层存储中数据的局部性特性做出推理。数据用行名和列名做索引，两者都可以是任意字符串。Bigtable 也把数据当作未解释的字符串处理，尽管客户端常常把各种形式的结构化、半结构化数据序列化成这些字符串。客户端可以通过精心选择 schema 来控制自己数据的局部性。最后，Bigtable 的 schema 参数让客户端能动态控制数据是从内存服务还是从磁盘服务。
{{% /bilingual %}}

{{% bilingual %}}
Section 2 describes the data model in more detail, and Section 3 provides an overview of the client API. Section 4 briefly describes the underlying Google infrastructure on which Bigtable depends. Section 5 describes the fundamentals of the Bigtable implementation, and Section 6 describes some of the refinements that we made to improve Bigtable's performance. Section 7 provides measurements of Bigtable's performance. We describe several examples of how Bigtable is used at Google in Section 8, and discuss some lessons we learned in designing and supporting Bigtable in Section 9. Finally, Section 10 describes related work, and Section 11 presents our conclusions.
<!--col-->
第 2 节更详细地描述数据模型，第 3 节概述客户端 API。第 4 节简要介绍 Bigtable 依赖的底层 Google 基础设施。第 5 节描述 Bigtable 实现的基础部分，第 6 节描述我们为提升 Bigtable 性能所做的一些改进。第 7 节给出 Bigtable 的性能测量。第 8 节给出 Bigtable 在 Google 内部使用的几个例子，第 9 节讨论我们在设计和支持 Bigtable 过程中得到的一些教训。最后，第 10 节介绍相关工作，第 11 节给出结论。
{{% /bilingual %}}

---

## 2 Data Model · 数据模型

{{% bilingual %}}
A Bigtable is a sparse, distributed, persistent multi-dimensional sorted map. The map is indexed by a row key, column key, and a timestamp; each value in the map is an uninterpreted array of bytes.
<!--col-->
一个 Bigtable 是稀疏的、分布式的、持久的多维有序映射（map）。这个映射由行键（row key）、列键（column key）和时间戳索引；映射中的每个值都是一个未被解释的字节数组。
{{% /bilingual %}}

```
(row:string, column:string, time:int64) → string
```

{{% bilingual %}}
We settled on this data model after examining a variety of potential uses of a Bigtable-like system. As one concrete example that drove some of our design decisions, suppose we want to keep a copy of a large collection of web pages and related information that could be used by many different projects; let us call this particular table the Webtable. In Webtable, we would use URLs as row keys, various aspects of web pages as column names, and store the contents of the web pages in the contents: column under the timestamps when they were fetched, as illustrated in Figure 1.
<!--col-->
我们是在考察了 Bigtable 类系统的各种潜在用途之后才确定这个数据模型的。作为一个推动了部分设计决策的具体例子，假设我们想保存一大批网页及相关信息的副本，供许多不同项目使用；我们把这一个表称为 Webtable。在 Webtable 中，我们用 URL 作为行键，用网页的各个方面作为列名，并把网页内容存在 `contents:` 列下，以抓取时间为时间戳，如图 1 所示。
{{% /bilingual %}}

图 1：存放网页的示例表的一个切片（A slice of an example table that stores Web pages）。行名是反转后的 URL。`contents` 列族存放页面内容，`anchor` 列族存放引用该页面的锚文本。CNN 主页同时被 Sports Illustrated 和 MY-look 两个主页引用，因此该行含有名为 `anchor:cnnsi.com` 和 `anchor:my.look.ca` 的列。每个 anchor 单元只有一个版本；`contents` 列有三个版本，时间戳分别是 t3、t5、t6。

| 行键 | `contents:` t3 | `contents:` t5 | `contents:` t6 | `anchor:cnnsi.com` t9 | `anchor:my.look.ca` t8 |
|------|----------------|----------------|----------------|-----------------------|------------------------|
| com.cnn.www | "<html>…" | "<html>…" | "<html>…" | "CNN.com" | "CNN" |

### Rows · 行

{{% bilingual %}}
The row keys in a table are arbitrary strings (currently up to 64KB in size, although 10-100 bytes is a typical size for most of our users). Every read or write of data under a single row key is atomic (regardless of the number of different columns being read or written in the row), a design decision that makes it easier for clients to reason about the system's behavior in the presence of concurrent updates to the same row.
<!--col-->
表中的行键是任意字符串（目前上限 64 KB，不过对大多数用户来说 10–100 字节是典型大小）。单个行键下数据的每次读或写都是原子的（无论这一行里被读写的不同列有多少个）。这个设计决定让客户端更容易推理系统在同一行存在并发更新时的行为。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable maintains data in lexicographic order by row key. The row range for a table is dynamically partitioned. Each row range is called a tablet, which is the unit of distribution and load balancing. As a result, reads of short row ranges are efficient and typically require communication with only a small number of machines. Clients can exploit this property by selecting their row keys so that they get good locality for their data accesses. For example, in Webtable, pages in the same domain are grouped together into contiguous rows by reversing the hostname components of the URLs. For example, we store data for maps.google.com/index.html under the key com.google.maps/index.html. Storing pages from the same domain near each other makes some host and domain analyses more efficient.
<!--col-->
Bigtable 按行键的字典序维护数据。一个表的行区间是动态划分的。每个行区间称为一个 tablet，它是分发和负载均衡的单位。因此，读取较短的行区间效率很高，通常只需要与少数几台机器通信。客户端可以利用这个特性——通过选择行键来让自己的数据访问获得良好的局部性。例如在 Webtable 中，我们把 URL 的主机名各段反转，使同一域名下的页面聚成连续的行。比如 maps.google.com/index.html 的数据我们存在键 `com.google.maps/index.html` 下。把同一域名的页面放在彼此附近，能让某些主机与域名分析更高效。
{{% /bilingual %}}

### Column Families · 列族

{{% bilingual %}}
Column keys are grouped into sets called column families, which form the basic unit of access control. All data stored in a column family is usually of the same type (we compress data in the same column family together). A column family must be created before data can be stored under any column key in that family; after a family has been created, any column key within the family can be used. It is our intent that the number of distinct column families in a table be small (in the hundreds at most), and that families rarely change during operation. In contrast, a table may have an unbounded number of columns.
<!--col-->
列键被归入若干集合，这些集合称为列族（column family），它们是访问控制的基本单位。同一列族中存放的所有数据通常属于同一类型（我们会对同一列族中的数据一起做压缩）。必须先创建列族，才能把数据存到该族下的任何列键上；列族创建之后，族内任何列键都可以使用。我们的意图是让一个表中的列族数量尽量少（最多几百个），并且列族在运行期间很少变动。相比之下，一个表可以有无限多的列。
{{% /bilingual %}}

{{% bilingual %}}
A column key is named using the following syntax: family:qualifier. Column family names must be printable, but qualifiers may be arbitrary strings. An example column family for the Webtable is language, which stores the language in which a web page was written. We use only one column key in the language family, and it stores each web page's language ID. Another useful column family for this table is anchor; each column key in this family represents a single anchor, as shown in Figure 1. The qualifier is the name of the referring site; the cell contents is the link text.
<!--col-->
列键的命名语法是 `family:qualifier`。列族名必须是可打印字符，但限定符（qualifier）可以是任意字符串。Webtable 的一个示例列族是 `language`，存放网页所用的语言。`language` 族里我们只用一个列键，它存放每个网页的语言 ID。这个表的另一个有用列族是 `anchor`；该族中每个列键代表一个锚，如图 1 所示。限定符是引用方的站点名，单元内容则是链接文本。
{{% /bilingual %}}

{{% bilingual %}}
Access control and both disk and memory accounting are performed at the column-family level. In our Webtable example, these controls allow us to manage several different types of applications: some that add new base data, some that read the base data and create derived column families, and some that are only allowed to view existing data (and possibly not even to view all of the existing families for privacy reasons).
<!--col-->
访问控制以及磁盘与内存的计量都在列族这一级进行。在我们的 Webtable 例子里，这些控制让我们能管理几类不同的应用：有的只添加新的基础数据，有的读取基础数据并创建派生的列族，还有的只被允许查看已有数据（出于隐私原因，甚至可能连已有的列族都不能全部查看）。
{{% /bilingual %}}

### Timestamps · 时间戳

{{% bilingual %}}
Each cell in a Bigtable can contain multiple versions of the same data; these versions are indexed by timestamp. Bigtable timestamps are 64-bit integers. They can be assigned by Bigtable, in which case they represent "real time" in microseconds, or be explicitly assigned by client applications. Applications that need to avoid collisions must generate unique timestamps themselves. Different versions of a cell are stored in decreasing timestamp order, so that the most recent versions can be read first.
<!--col-->
Bigtable 中的每个单元可以包含同一数据的多个版本；这些版本由时间戳索引。Bigtable 的时间戳是 64 位整数。它们可以由 Bigtable 赋值——此时表示以微秒计的「真实时间」——也可以由客户端应用显式赋值。需要避免冲突的应用必须自己生成唯一时间戳。同一个单元的不同版本按时间戳递减的顺序存放，这样最新的版本能被最先读到。
{{% /bilingual %}}

{{% bilingual %}}
To make the management of versioned data less onerous, we support two per-column-family settings that tell Bigtable to garbage-collect cell versions automatically. The client can specify either that only the last n versions of a cell be kept, or that only new-enough versions be kept (e.g., only keep values that were written in the last seven days).
<!--col-->
为了让带版本数据的管理不那么繁重，我们支持两个按列族设置的选项，让 Bigtable 自动回收单元版本。客户端可以指定只保留某个单元的最后 n 个版本，或者只保留足够新的版本（例如只保留最近七天内写入的值）。
{{% /bilingual %}}

{{% bilingual %}}
In our Webtable example, we set the timestamps of the crawled pages stored in the contents: column to the times at which these page versions were actually crawled. The garbage-collection mechanism described above lets us keep only the most recent three versions of every page.
<!--col-->
在 Webtable 例子里，我们把 `contents:` 列中存放的抓取页面时间戳设为这些页面版本实际被爬取的时间。上面描述的垃圾回收机制让我们只保留每个页面最近的三个版本。
{{% /bilingual %}}

---

## 3 API · 接口

{{% bilingual %}}
The Bigtable API provides functions for creating and deleting tables and column families. It also provides functions for changing cluster, table, and column family metadata, such as access control rights.
<!--col-->
Bigtable API 提供了创建和删除表、列族的函数，也提供了修改集群、表和列族元数据（比如访问控制权限）的函数。
{{% /bilingual %}}

{{% bilingual %}}
Client applications can write or delete values in Bigtable, look up values from individual rows, or iterate over a subset of the data in a table. Figure 2 shows C++ code that uses a RowMutation abstraction to perform a series of updates. (Irrelevant details were elided to keep the example short.) The call to Apply performs an atomic mutation to the Webtable: it adds one anchor to www.cnn.com and deletes a different anchor.
<!--col-->
客户端应用可以写入或删除 Bigtable 中的值、从单行中查找值，或者遍历表中数据的某个子集。图 2 是一段 C++ 代码，它用 RowMutation 抽象执行一系列更新。（为了让例子简短，无关细节被省略。）对 `Apply` 的调用对 Webtable 做了一次原子修改：给 www.cnn.com 增加一个锚，并删除另一个锚。
{{% /bilingual %}}

```cpp
// Open the table
Table *T = OpenOrDie("/bigtable/web/webtable");

// Write a new anchor and delete an old anchor
RowMutation r1(T, "com.cnn.www");
r1.Set("anchor:www.c-span.org", "CNN");
r1.Delete("anchor:www.abc.com");
Operation op;
Apply(&op, &r1);
```

图 2：向 Bigtable 写入（Writing to Bigtable）。

{{% bilingual %}}
Figure 3 shows C++ code that uses a Scanner abstraction to iterate over all anchors in a particular row. Clients can iterate over multiple column families, and there are several mechanisms for limiting the rows, columns, and timestamps produced by a scan. For example, we could restrict the scan above to only produce anchors whose columns match the regular expression anchor:*.cnn.com, or to only produce anchors whose timestamps fall within ten days of the current time.
<!--col-->
图 3 是一段 C++ 代码，它用 Scanner 抽象遍历某一行的全部锚。客户端可以遍历多个列族，也有多种机制限制扫描产出的行、列和时间戳。例如，我们可以把上面的扫描限制为只产出列名匹配正则表达式 `anchor:*.cnn.com` 的锚，或者只产出时间戳落在当前时间十天以内的锚。
{{% /bilingual %}}

```cpp
Scanner scanner(T);
ScanStream *stream;
stream = scanner.FetchColumnFamily("anchor");
stream->SetReturnAllVersions();
scanner.Lookup("com.cnn.www");
for (; !stream->Done(); stream->Next()) {
  printf("%s %s %lld %s\n",
         scanner.RowName(),
         stream->ColumnName(),
         stream->MicroTimestamp(),
         stream->Value());
}
```

图 3：从 Bigtable 读取（Reading from Bigtable）。

{{% bilingual %}}
Bigtable supports several other features that allow the user to manipulate data in more complex ways. First, Bigtable supports single-row transactions, which can be used to perform atomic read-modify-write sequences on data stored under a single row key. Bigtable does not currently support general transactions across row keys, although it provides an interface for batching writes across row keys at the clients. Second, Bigtable allows cells to be used as integer counters. Finally, Bigtable supports the execution of client-supplied scripts in the address spaces of the servers. The scripts are written in a language developed at Google for processing data called Sawzall [28]. At the moment, our Sawzall-based API does not allow client scripts to write back into Bigtable, but it does allow various forms of data transformation, filtering based on arbitrary expressions, and summarization via a variety of operators.
<!--col-->
Bigtable 还支持其他几项特性，让用户能以更复杂的方式操作数据。第一，Bigtable 支持单行事务，可以用来对单个行键下存放的数据执行原子的「读—改—写」序列。Bigtable 目前不支持跨行键的通用事务，不过它提供了一个在客户端对跨行键写入做批处理的接口。第二，Bigtable 允许把单元当作整数计数器使用。最后，Bigtable 支持在服务器的地址空间里执行客户端提供的脚本。这些脚本用一种 Google 内部开发的、用于处理数据的语言 Sawzall [28] 编写。目前，我们这套基于 Sawzall 的 API 不允许客户端脚本写回 Bigtable，但它允许各种形式的数据转换、基于任意表达式的过滤，以及用多种算子做汇总。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable can be used with MapReduce [12], a framework for running large-scale parallel computations developed at Google. We have written a set of wrappers that allow a Bigtable to be used both as an input source and as an output target for MapReduce jobs.
<!--col-->
Bigtable 可以与 MapReduce [12] 配合使用——后者是 Google 开发的、用于运行大规模并行计算的框架。我们写了一组封装，让 Bigtable 既能作为 MapReduce 作业的输入源，也能作为其输出目标。
{{% /bilingual %}}

---

## 4 Building Blocks · 基础构件

{{% bilingual %}}
Bigtable is built on several other pieces of Google infrastructure. Bigtable uses the distributed Google File System (GFS) [17] to store log and data files. A Bigtable cluster typically operates in a shared pool of machines that run a wide variety of other distributed applications, and Bigtable processes often share the same machines with processes from other applications.
<!--col-->
Bigtable 建立在 Google 的其他几项基础设施之上。Bigtable 用分布式文件系统 Google File System（GFS）[17] 存放日志文件和数据文件。一个 Bigtable 集群通常运行在一个共享的机器池中，池里还跑着各种各样的其他分布式应用，Bigtable 进程常常与其他应用的进程共用同一批机器。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable depends on a cluster management system for scheduling jobs, managing resources on shared machines, dealing with machine failures, and monitoring machine status.
<!--col-->
Bigtable 依赖一套集群管理系统来调度作业、管理共享机器上的资源、处理机器故障和监控机器状态。
{{% /bilingual %}}

{{% bilingual %}}
The Google SSTable file format is used internally to store Bigtable data. An SSTable provides a persistent, ordered immutable map from keys to values, where both keys and values are arbitrary byte strings. Operations are provided to look up the value associated with a specified key, and to iterate over all key/value pairs in a specified key range. Internally, each SSTable contains a sequence of blocks (typically each block is 64KB in size, but this is configurable). A block index (stored at the end of the SSTable) is used to locate blocks; the index is loaded into memory when the SSTable is opened. A lookup can be performed with a single disk seek: we first find the appropriate block by performing a binary search in the in-memory index, and then reading the appropriate block from disk. Optionally, an SSTable can be completely mapped into memory, which allows us to perform lookups and scans without touching disk.
<!--col-->
Bigtable 内部用 Google 的 SSTable 文件格式存放数据。一个 SSTable 提供的是从键到值的持久化、有序、不可变映射，其中键和值都是任意字节串。它提供两类操作：查找指定键对应的值，以及遍历指定键区间内的所有 key/value 对。在内部，每个 SSTable 包含一串块（每块通常 64 KB，可配置）。块索引（存放在 SSTable 末尾）用于定位块；SSTable 打开时索引被加载进内存。一次查找只需一次磁盘寻道：先在内存索引中做二分查找找到合适的块，再从磁盘读出该块。可选地，一个 SSTable 可以被完整映射进内存，这样查找和扫描就完全不碰磁盘。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable relies on a highly-available and persistent distributed lock service called Chubby [8]. A Chubby service consists of five active replicas, one of which is elected to be the master and actively serve requests. The service is live when a majority of the replicas are running and can communicate with each other. Chubby uses the Paxos algorithm [9, 23] to keep its replicas consistent in the face of failure. Chubby provides a namespace that consists of directories and small files. Each directory or file can be used as a lock, and reads and writes to a file are atomic. The Chubby client library provides consistent caching of Chubby files. Each Chubby client maintains a session with a Chubby service. A client's session expires if it is unable to renew its session lease within the lease expiration time. When a client's session expires, it loses any locks and open handles. Chubby clients can also register callbacks on Chubby files and directories for notification of changes or session expiration.
<!--col-->
Bigtable 依赖一个高可用、持久的分布式锁服务 Chubby [8]。一个 Chubby 服务由五个活跃副本组成，其中一个被选为 master 并实际处理请求。当多数副本在运行且能相互通信时，该服务是可用的。Chubby 使用 Paxos 算法 [9, 23] 在出现故障时保持各副本一致。Chubby 提供一个由目录和小文件构成的命名空间。每个目录或文件都可以当作锁使用，对一个文件的读写是原子的。Chubby 客户端库提供对 Chubby 文件的一致性缓存。每个 Chubby 客户端都与 Chubby 服务维持一个会话。如果客户端无法在租约到期前续租，它的会话就会过期；会话过期时，客户端会丢失所有锁和打开的文件句柄。Chubby 客户端还可以在 Chubby 文件和目录上注册回调，以便得到变更或会话过期的通知。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable uses Chubby for a variety of tasks: to ensure that there is at most one active master at any time; to store the bootstrap location of Bigtable data (see Section 5.1); to discover tablet servers and finalize tablet server deaths (see Section 5.2); to store Bigtable schema information (the column family information for each table); and to store access control lists. If Chubby becomes unavailable for an extended period of time, Bigtable becomes unavailable. We recently measured this effect in 14 Bigtable clusters spanning 11 Chubby instances. The average percentage of Bigtable server hours during which some data stored in Bigtable was not available due to Chubby unavailability (caused by either Chubby outages or network issues) was 0.0047%. The percentage for the single cluster that was most affected by Chubby unavailability was 0.0326%.
<!--col-->
Bigtable 把 Chubby 用于多种任务：确保任意时刻最多只有一个活跃 master；存放 Bigtable 数据的引导位置（见 5.1 节）；发现 tablet server 并最终确认 tablet server 的死亡（见 5.2 节）；存放 Bigtable 的 schema 信息（每个表的列族信息）；以及存放访问控制列表。如果 Chubby 长时间不可用，Bigtable 也会不可用。我们最近在跨 11 个 Chubby 实例的 14 个 Bigtable 集群上测量了这一影响。由于 Chubby 不可用（由 Chubby 故障或网络问题引起）导致 Bigtable 中某些数据不可用的时长占 Bigtable 服务器总时长的平均比例为 0.0047%；受 Chubby 不可用影响最严重的那个集群，这一比例是 0.0326%。
{{% /bilingual %}}

---

## 5 Implementation · 实现

{{% bilingual %}}
The Bigtable implementation has three major components: a library that is linked into every client, one master server, and many tablet servers. Tablet servers can be dynamically added (or removed) from a cluster to accomodate changes in workloads.
<!--col-->
Bigtable 实现有三个主要组成部分：一个链接进每个客户端的库、一台 master 服务器，以及许多 tablet 服务器。tablet 服务器可以在集群中动态增删，以适应工作负载的变化。
{{% /bilingual %}}

{{% bilingual %}}
The master is responsible for assigning tablets to tablet servers, detecting the addition and expiration of tablet servers, balancing tablet-server load, and garbage collection of files in GFS. In addition, it handles schema changes such as table and column family creations.
<!--col-->
master 负责把 tablet 分配给 tablet 服务器、检测 tablet 服务器的加入与过期、均衡 tablet 服务器的负载，以及回收 GFS 中的文件。此外，它还处理 schema 变更，比如创建表和列族。
{{% /bilingual %}}

{{% bilingual %}}
Each tablet server manages a set of tablets (typically we have somewhere between ten to a thousand tablets per tablet server). The tablet server handles read and write requests to the tablets that it has loaded, and also splits tablets that have grown too large.
<!--col-->
每个 tablet 服务器管理一组 tablet（通常每个 tablet 服务器上有十个到一千个 tablet）。tablet 服务器处理对它已加载 tablet 的读写请求，并对长得过大的 tablet 做分裂。
{{% /bilingual %}}

{{% bilingual %}}
As with many single-master distributed storage systems [17, 21], client data does not move through the master: clients communicate directly with tablet servers for reads and writes. Because Bigtable clients do not rely on the master for tablet location information, most clients never communicate with the master. As a result, the master is lightly loaded in practice.
<!--col-->
和许多单 master 的分布式存储系统 [17, 21] 一样，客户端数据不经过 master：客户端直接与 tablet 服务器通信来读写。由于 Bigtable 客户端不依赖 master 获取 tablet 位置信息，大多数客户端从不与 master 通信。因此实践中 master 的负载很轻。
{{% /bilingual %}}

{{% bilingual %}}
A Bigtable cluster stores a number of tables. Each table consists of a set of tablets, and each tablet contains all data associated with a row range. Initially, each table consists of just one tablet. As a table grows, it is automatically split into multiple tablets, each approximately 100-200 MB in size by default.
<!--col-->
一个 Bigtable 集群存放若干个表。每个表由一组 tablet 组成，每个 tablet 包含一个行区间关联的全部数据。起初每个表只有一个 tablet。随着表增长，它会被自动分裂成多个 tablet，默认每个约 100–200 MB。
{{% /bilingual %}}

### 5.1 Tablet Location · Tablet 定位

{{% bilingual %}}
We use a three-level hierarchy analogous to that of a B+tree [10] to store tablet location information (Figure 4).
<!--col-->
我们用类似 B+ 树 [10] 的三级层级结构来存放 tablet 位置信息（图 4）。
{{% /bilingual %}}

```
  ┌──────────────────────────────────┐
  │           Chubby file            │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │           Root tablet            │
  │      (1st METADATA tablet)       │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │   Other METADATA tablets         │
  │   (contain user tablet locations)│
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │   UserTable1 ... UserTableN      │
  └──────────────────────────────────┘
```

图 4：tablet 位置层级（Tablet location hierarchy）。

{{% bilingual %}}
The first level is a file stored in Chubby that contains the location of the root tablet. The root tablet contains the location of all tablets in a special METADATA table. Each METADATA tablet contains the location of a set of user tablets. The root tablet is just the first tablet in the METADATA table, but is treated specially--it is never split--to ensure that the tablet location hierarchy has no more than three levels.
<!--col-->
第一级是存放在 Chubby 里的一个文件，内容是 root tablet 的位置。root tablet 包含一个特殊 METADATA 表中所有 tablet 的位置。每个 METADATA tablet 包含一组用户 tablet 的位置。root tablet 就是 METADATA 表中的第一个 tablet，但它被特殊对待——它永不分裂——以确保 tablet 位置层级不超过三级。
{{% /bilingual %}}

{{% bilingual %}}
The METADATA table stores the location of a tablet under a row key that is an encoding of the tablet's table identifier and its end row. Each METADATA row stores approximately 1KB of data in memory. With a modest limit of 128 MB METADATA tablets, our three-level location scheme is sufficient to address 2^34 tablets (or 2^61 bytes in 128 MB tablets).
<!--col-->
METADATA 表以行为键存放 tablet 的位置，该行键是 tablet 所属表的标识符与其结束行的编码。每个 METADATA 行在内存中约占 1 KB 数据。把 METADATA tablet 的容量限制在适度的 128 MB，我们这套三级定位方案就足以寻址 `2^34` 个 tablet（或按每个 tablet 128 MB 计共 `2^61` 字节）。
{{% /bilingual %}}

{{% bilingual %}}
The client library caches tablet locations. If the client does not know the location of a tablet, or if it discovers that cached location information is incorrect, then it recursively moves up the tablet location hierarchy. If the client's cache is empty, the location algorithm requires three network round-trips, including one read from Chubby. If the client's cache is stale, the location algorithm could take up to six round-trips, because stale cache entries are only discovered upon misses (assuming that METADATA tablets do not move very frequently). Although tablet locations are stored in memory, so no GFS accesses are required, we further reduce this cost in the common case by having the client library prefetch tablet locations: it reads the metadata for more than one tablet whenever it reads the METADATA table.
<!--col-->
客户端库会缓存 tablet 位置。如果客户端不知道某个 tablet 的位置，或者发现缓存的位置信息有误，它就沿 tablet 位置层级逐级向上查找。如果客户端缓存为空，定位算法需要三次网络往返，其中包括一次对 Chubby 的读取。如果客户端缓存已陈旧，定位算法最多可能需要六次往返，因为陈旧的缓存项只有在未命中时才会被发现（这里假设 METADATA tablet 不频繁移动）。尽管 tablet 位置存在内存中、不需要访问 GFS，我们还是在常见情形下进一步降低了这一开销：让客户端库预取 tablet 位置——每当它读取 METADATA 表时，会一次读入多个 tablet 的元数据。
{{% /bilingual %}}

{{% bilingual %}}
We also store secondary information in the METADATA table, including a log of all events pertaining to each tablet (such as when a server begins serving it). This information is helpful for debugging and performance analysis.
<!--col-->
我们还在 METADATA 表中存放次级信息，包括与每个 tablet 相关的所有事件日志（比如某个服务器何时开始服务它）。这些信息对调试和性能分析很有帮助。
{{% /bilingual %}}

### 5.2 Tablet Assignment · Tablet 分配

{{% bilingual %}}
Each tablet is assigned to one tablet server at a time. The master keeps track of the set of live tablet servers, and the current assignment of tablets to tablet servers, including which tablets are unassigned. When a tablet is unassigned, and a tablet server with sufficient room for the tablet is available, the master assigns the tablet by sending a tablet load request to the tablet server.
<!--col-->
每个 tablet 在任一时刻只分配给一个 tablet 服务器。master 跟踪活跃 tablet 服务器的集合，以及 tablet 到 tablet 服务器的当前分配情况，包括哪些 tablet 尚未分配。当一个 tablet 处于未分配状态、且存在有足够空间容纳它的 tablet 服务器时，master 就通过向该 tablet 服务器发送一个 tablet 加载请求来完成分配。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable uses Chubby to keep track of tablet servers. When a tablet server starts, it creates, and acquires an exclusive lock on, a uniquely-named file in a specific Chubby directory. The master monitors this directory (the servers directory) to discover tablet servers. A tablet server stops serving its tablets if it loses its exclusive lock: e.g., due to a network partition that caused the server to lose its Chubby session. (Chubby provides an efficient mechanism that allows a tablet server to check whether it still holds its lock without incurring network traffic.) A tablet server will attempt to reacquire an exclusive lock on its file as long as the file still exists. If the file no longer exists, then the tablet server will never be able to serve again, so it kills itself. Whenever a tablet server terminates (e.g., because the cluster management system is removing the tablet server's machine from the cluster), it attempts to release its lock so that the master will reassign its tablets more quickly.
<!--col-->
Bigtable 用 Chubby 跟踪 tablet 服务器。tablet 服务器启动时，会在某个特定 Chubby 目录下创建一个唯一命名的文件，并获取其排他锁。master 监视这个目录（servers 目录）来发现 tablet 服务器。如果 tablet 服务器丢失了排他锁，它就会停止服务自己的 tablet——比如因为网络分区导致该服务器丢失了 Chubby 会话。（Chubby 提供了一种高效机制，让 tablet 服务器能够在不产生网络流量的情况下检查自己是否仍持有锁。）只要文件还存在，tablet 服务器就会不断尝试重新获取该文件的排他锁。如果文件已不存在，那么该 tablet 服务器永远无法再服务，于是它会杀掉自己。每当 tablet 服务器终止时（比如因为集群管理系统要把这台服务器从集群中移除），它会尝试释放自己的锁，以便 master 更快地重新分配它的 tablet。
{{% /bilingual %}}

{{% bilingual %}}
The master is responsible for detecting when a tablet server is no longer serving its tablets, and for reassigning those tablets as soon as possible. To detect when a tablet server is no longer serving its tablets, the master periodically asks each tablet server for the status of its lock. If a tablet server reports that it has lost its lock, or if the master was unable to reach a server during its last several attempts, the master attempts to acquire an exclusive lock on the server's file. If the master is able to acquire the lock, then Chubby is live and the tablet server is either dead or having trouble reaching Chubby, so the master ensures that the tablet server can never serve again by deleting its server file. Once a server's file has been deleted, the master can move all the tablets that were previously assigned to that server into the set of unassigned tablets. To ensure that a Bigtable cluster is not vulnerable to networking issues between the master and Chubby, the master kills itself if its Chubby session expires. However, as described above, master failures do not change the assignment of tablets to tablet servers.
<!--col-->
master 负责检测某个 tablet 服务器何时不再服务它的 tablet，并尽快重新分配这些 tablet。为了检测 tablet 服务器是否已停止服务，master 周期性地询问每个 tablet 服务器其锁的状态。如果某个 tablet 服务器报告自己已丢失锁，或者 master 在最近几次尝试中都无法联系上某台服务器，master 就会尝试获取该服务器文件的排他锁。如果 master 能拿到锁，说明 Chubby 是活的，而该 tablet 服务器要么已死、要么难以访问 Chubby，于是 master 通过删除它的服务器文件来确保它永远不会再服务。服务器文件一旦被删除，master 就可以把原先分配给该服务器的所有 tablet 移入未分配集合。为确保 Bigtable 集群不会因 master 与 Chubby 之间的网络问题而受影响，master 在自己的 Chubby 会话过期时会杀掉自己。不过如上所述，master 故障不会改变 tablet 到 tablet 服务器的分配关系。
{{% /bilingual %}}

{{% bilingual %}}
When a master is started by the cluster management system, it needs to discover the current tablet assignments before it can change them. The master executes the following steps at startup. (1) The master grabs a unique master lock in Chubby, which prevents concurrent master instantiations. (2) The master scans the servers directory in Chubby to find the live servers. (3) The master communicates with every live tablet server to discover what tablets are already assigned to each server. (4) The master scans the METADATA table to learn the set of tablets. Whenever this scan encounters a tablet that is not already assigned, the master adds the tablet to the set of unassigned tablets, which makes the tablet eligible for tablet assignment.
<!--col-->
当 master 被集群管理系统启动时，它需要先发现当前的 tablet 分配情况，然后才能改动它们。master 在启动时执行下列步骤。（1）master 在 Chubby 中抢到一个唯一的 master 锁，这防止同时出现多个 master 实例。（2）master 扫描 Chubby 中的 servers 目录，找出活跃服务器。（3）master 与每一个活跃的 tablet 服务器通信，查明各服务器上已经分配了哪些 tablet。（4）master 扫描 METADATA 表，了解 tablet 的集合。每当这次扫描遇到一个尚未分配的 tablet，master 就把它加入未分配集合，使该 tablet 具备被分配的资格。
{{% /bilingual %}}

{{% bilingual %}}
One complication is that the scan of the METADATA table cannot happen until the METADATA tablets have been assigned. Therefore, before starting this scan (step 4), the master adds the root tablet to the set of unassigned tablets if an assignment for the root tablet was not discovered during step 3. This addition ensures that the root tablet will be assigned. Because the root tablet contains the names of all METADATA tablets, the master knows about all of them after it has scanned the root tablet.
<!--col-->
有一个麻烦之处：METADATA 表的扫描必须等到 METADATA tablet 已被分配之后才能进行。因此在开始这次扫描（第 4 步）之前，如果在第 3 步中没有发现 root tablet 的分配信息，master 就把 root tablet 加入未分配集合。这一添加确保 root tablet 一定会被分配。由于 root tablet 包含所有 METADATA tablet 的名字，master 在扫描完 root tablet 之后就知道了全部 METADATA tablet。
{{% /bilingual %}}

{{% bilingual %}}
The set of existing tablets only changes when a table is created or deleted, two existing tablets are merged to form one larger tablet, or an existing tablet is split into two smaller tablets. The master is able to keep track of these changes because it initiates all but the last. Tablet splits are treated specially since they are initiated by a tablet server. The tablet server commits the split by recording information for the new tablet in the METADATA table. When the split has committed, it notifies the master. In case the split notification is lost (either because the tablet server or the master died), the master detects the new tablet when it asks a tablet server to load the tablet that has now split. The tablet server will notify the master of the split, because the tablet entry it finds in the METADATA table will specify only a portion of the tablet that the master asked it to load.
<!--col-->
已有 tablet 的集合只在下列情况下改变：创建或删除一个表；两个已有 tablet 合并成一个更大的 tablet；或者一个已有 tablet 分裂成两个更小的 tablet。master 能跟踪这些变化，因为除最后一种之外，其余都由它发起。tablet 分裂被特殊对待，因为它由 tablet 服务器发起。tablet 服务器通过在 METADATA 表中记录新 tablet 的信息来提交这次分裂。分裂提交后，它通知 master。万一分裂通知丢失（无论是 tablet 服务器还是 master 挂掉），master 在要求某个 tablet 服务器加载那个已经分裂的 tablet 时会发现这个新 tablet。该 tablet 服务器会把这次分裂通知 master，因为它在 METADATA 表中找到的 tablet 条目只会指明 master 要求它加载的那个 tablet 的一部分。
{{% /bilingual %}}

### 5.3 Tablet Serving · Tablet 服务

{{% bilingual %}}
The persistent state of a tablet is stored in GFS, as illustrated in Figure 5. Updates are committed to a commit log that stores redo records. Of these updates, the recently committed ones are stored in memory in a sorted buffer called a memtable; the older updates are stored in a sequence of SSTables. To recover a tablet, a tablet server reads its metadata from the METADATA table. This metadata contains the list of SSTables that comprise a tablet and a set of a redo points, which are pointers into any commit logs that may contain data for the tablet. The server reads the indices of the SSTables into memory and reconstructs the memtable by applying all of the updates that have committed since the redo points.
<!--col-->
tablet 的持久状态存在 GFS 中，如图 5 所示。更新被提交到一个存放重做记录（redo record）的提交日志（commit log）。在这些更新中，最近提交的那些存放在内存里一个叫做 memtable 的有序缓冲区中；更早的更新则存放在一串 SSTable 中。要恢复一个 tablet，tablet 服务器从 METADATA 表读取它的元数据。这些元数据包含构成该 tablet 的 SSTable 列表，以及一组重做点（redo point）——它们是指向可能含有该 tablet 数据的各个提交日志的指针。服务器把 SSTable 的索引读进内存，并通过施加自重做点以来已提交的全部更新来重建 memtable。
{{% /bilingual %}}

```
            Read Op                            Write Op
               │                                  │
               ▼                                  ▼
     ┌───────────────────┐              ┌───────────────────┐
     │     memtable      │◀─── insert ──│    tablet log     │
     │   (in memory)     │              │       (GFS)       │
     └─────────┬─────────┘              └───────────────────┘
               │ merge
               ▼
     ┌───────────────────┐
     │   SSTable files   │
     │       (GFS)       │
     └───────────────────┘
```

图 5：tablet 的表示（Tablet Representation）。读操作作用于 memtable 与 SSTable 文件的合并视图；写操作先追加到 tablet log，再写入 memtable。

{{% bilingual %}}
When a write operation arrives at a tablet server, the server checks that it is well-formed, and that the sender is authorized to perform the mutation. Authorization is performed by reading the list of permitted writers from a Chubby file (which is almost always a hit in the Chubby client cache). A valid mutation is written to the commit log. Group commit is used to improve the throughput of lots of small mutations [13, 16]. After the write has been committed, its contents are inserted into the memtable.
<!--col-->
当写操作到达 tablet 服务器时，服务器检查它格式是否合法，以及发送方是否有权执行该修改。授权检查的方式是从一个 Chubby 文件中读取允许的写入者列表（这一读取几乎总是命中 Chubby 客户端缓存）。合法的修改会被写入提交日志。为了让大量小修改的吞吐更高，我们使用了组提交（group commit）[13, 16]。写入提交之后，其内容被插入 memtable。
{{% /bilingual %}}

{{% bilingual %}}
When a read operation arrives at a tablet server, it is similarly checked for well-formedness and proper authorization. A valid read operation is executed on a merged view of the sequence of SSTables and the memtable. Since the SSTables and the memtable are lexicographically sorted data structures, the merged view can be formed efficiently.
<!--col-->
当读操作到达 tablet 服务器时，同样要检查格式合法性与授权。合法的读操作在一串 SSTable 与 memtable 的合并视图上执行。由于 SSTable 和 memtable 都是按字典序排序的数据结构，合并视图可以高效地构造出来。
{{% /bilingual %}}

{{% bilingual %}}
Incoming read and write operations can continue while tablets are split and merged.
<!--col-->
在 tablet 分裂和合并期间，到来的读写操作可以继续执行。
{{% /bilingual %}}

### 5.4 Compactions · 合并整理

{{% bilingual %}}
As write operations execute, the size of the memtable increases. When the memtable size reaches a threshold, the memtable is frozen, a new memtable is created, and the frozen memtable is converted to an SSTable and written to GFS. This minor compaction process has two goals: it shrinks the memory usage of the tablet server, and it reduces the amount of data that has to be read from the commit log during recovery if this server dies. Incoming read and write operations can continue while compactions occur.
<!--col-->
随着写操作不断执行，memtable 的大小会增长。当 memtable 大小达到阈值时，它被冻结，一个新的 memtable 被创建，而冻结的 memtable 被转换成 SSTable 并写入 GFS。这个「小合并」（minor compaction）过程有两个目标：缩小 tablet 服务器的内存占用；以及在该服务器挂掉时，减少恢复期间必须从提交日志读取的数据量。合并整理发生期间，到来的读写操作可以继续执行。
{{% /bilingual %}}

{{% bilingual %}}
Every minor compaction creates a new SSTable. If this behavior continued unchecked, read operations might need to merge updates from an arbitrary number of SSTables. Instead, we bound the number of such files by periodically executing a merging compaction in the background. A merging compaction reads the contents of a few SSTables and the memtable, and writes out a new SSTable. The input SSTables and memtable can be discarded as soon as the compaction has finished.
<!--col-->
每次小合并都会产生一个新的 SSTable。如果任由这种行为持续，读操作可能不得不合并任意多个 SSTable 中的更新。为此，我们通过在后台周期性地执行「合并整理」（merging compaction）来限定这类文件的数量。一次合并整理读取若干个 SSTable 以及 memtable 的内容，写出一个新的 SSTable。整理一旦完成，输入的 SSTable 和 memtable 就可以被丢弃。
{{% /bilingual %}}

{{% bilingual %}}
A merging compaction that rewrites all SSTables into exactly one SSTable is called a major compaction. SSTables produced by non-major compactions can contain special deletion entries that suppress deleted data in older SSTables that are still live. A major compaction, on the other hand, produces an SSTable that contains no deletion information or deleted data. Bigtable cycles through all of its tablets and regularly applies major compactions to them. These major compactions allow Bigtable to reclaim resources used by deleted data, and also allow it to ensure that deleted data disappears from the system in a timely fashion, which is important for services that store sensitive data.
<!--col-->
把所有 SSTable 重写成恰好一个 SSTable 的合并整理称为「大合并」（major compaction）。非大合并产生的 SSTable 可以包含特殊的删除条目，用于抑制仍然存活的更早 SSTable 中的已删除数据。而大合并产生的 SSTable 不含任何删除信息或已删除数据。Bigtable 会轮遍它所有的 tablet，定期对它们执行大合并。这些大合并让 Bigtable 能回收已删除数据占用的资源，也让它能确保已删除数据及时从系统中消失——这对存放敏感数据的服务很重要。
{{% /bilingual %}}

---

## 6 Refinements · 改进

{{% bilingual %}}
The implementation described in the previous section required a number of refinements to achieve the high performance, availability, and reliability required by our users. This section describes portions of the implementation in more detail in order to highlight these refinements.
<!--col-->
上一节描述的实现需要若干改进，才能达到用户要求的高性能、高可用和高可靠。本节更详细地描述实现中的部分内容，以突出这些改进。
{{% /bilingual %}}

### Locality groups · 局部性组

{{% bilingual %}}
Clients can group multiple column families together into a locality group. A separate SSTable is generated for each locality group in each tablet. Segregating column families that are not typically accessed together into separate locality groups enables more efficient reads. For example, page metadata in Webtable (such as language and checksums) can be in one locality group, and the contents of the page can be in a different group: an application that wants to read the metadata does not need to read through all of the page contents.
<!--col-->
客户端可以把多个列族组合成一个局部性组（locality group）。每个 tablet 中，每个局部性组都会生成独立的 SSTable。把通常不会一起访问的列族隔离到不同的局部性组，能让读更高效。例如 Webtable 中的页面元数据（如语言、校验和）可以放在一个局部性组里，而页面内容放在另一个组里：只想读元数据的应用不必读遍全部页面内容。
{{% /bilingual %}}

{{% bilingual %}}
In addition, some useful tuning parameters can be specified on a per-locality group basis. For example, a locality group can be declared to be in-memory. SSTables for in-memory locality groups are loaded lazily into the memory of the tablet server. Once loaded, column families that belong to such locality groups can be read without accessing the disk. This feature is useful for small pieces of data that are accessed frequently: we use it internally for the location column family in the METADATA table.
<!--col-->
此外，还有一些有用的调优参数可以按局部性组指定。例如，可以把某个局部性组声明为驻留内存（in-memory）。驻留内存的局部性组，其 SSTable 会被惰性加载进 tablet 服务器的内存。一旦加载完成，属于这些局部性组的列族就可以在不访问磁盘的情况下被读取。这个特性对频繁访问的小块数据很有用：我们在内部就把它用于 METADATA 表中的 location 列族。
{{% /bilingual %}}

### Compression · 压缩

{{% bilingual %}}
Clients can control whether or not the SSTables for a locality group are compressed, and if so, which compression format is used. The user-specified compression format is applied to each SSTable block (whose size is controllable via a locality group specific tuning parameter). Although we lose some space by compressing each block separately, we benefit in that small portions of an SSTable can be read without decompressing the entire file. Many clients use a two-pass custom compression scheme. The first pass uses Bentley and McIlroy's scheme [6], which compresses long common strings across a large window. The second pass uses a fast compression algorithm that looks for repetitions in a small 16 KB window of the data. Both compression passes are very fast--they encode at 100-200 MB/s, and decode at 400-1000 MB/s on modern machines.
<!--col-->
客户端可以控制某个局部性组的 SSTable 是否压缩，以及用哪种压缩格式。用户指定的压缩格式作用在每个 SSTable 块上（块大小可以通过局部性组专属的调优参数控制）。虽然逐块压缩会损失一些空间，但好处是读取 SSTable 的一小部分时不必解压整个文件。许多客户端使用一套两趟的自定义压缩方案。第一趟用 Bentley 和 McIlroy 的方案 [6]，在大窗口内压缩长公共字符串。第二趟用一种快速压缩算法，在数据的一个 16 KB 小窗口内寻找重复。两趟压缩都非常快——在现代机器上编码速度为 100–200 MB/s，解码速度为 400–1000 MB/s。
{{% /bilingual %}}

{{% bilingual %}}
Even though we emphasized speed instead of space reduction when choosing our compression algorithms, this two-pass compression scheme does surprisingly well. For example, in Webtable, we use this compression scheme to store Web page contents. In one experiment, we stored a large number of documents in a compressed locality group. For the purposes of the experiment, we limited ourselves to one version of each document instead of storing all versions available to us. The scheme achieved a 10-to-1 reduction in space. This is much better than typical Gzip reductions of 3-to-1 or 4-to-1 on HTML pages because of the way Webtable rows are laid out: all pages from a single host are stored close to each other. This allows the Bentley-McIlroy algorithm to identify large amounts of shared boilerplate in pages from the same host. Many applications, not just Webtable, choose their row names so that similar data ends up clustered, and therefore achieve very good compression ratios. Compression ratios get even better when we store multiple versions of the same value in Bigtable.
<!--col-->
尽管我们在选择压缩算法时强调速度而非空间缩减，这套两趟压缩方案的效果仍好得出人意料。例如在 Webtable 中，我们用这套方案存放网页内容。在一次实验里，我们把大量文档存进一个压缩的局部性组。出于实验目的，我们只保留每篇文档的一个版本，而不存放所有可用版本。这套方案实现了 10:1 的空间缩减。这远好于 Gzip 在 HTML 页面上典型的 3:1 或 4:1，原因在于 Webtable 行的布局方式：同一主机的所有页面都存放在彼此附近，这让 Bentley-McIlroy 算法能识别出同一主机各页面中大量共有的样板内容。许多应用（不只是 Webtable）都会通过选择行名让相似数据聚在一起，从而获得非常好的压缩率。当我们在 Bigtable 中存放同一个值的多个版本时，压缩率还会更高。
{{% /bilingual %}}

### Caching for read performance · 为读性能做缓存

{{% bilingual %}}
To improve read performance, tablet servers use two levels of caching. The Scan Cache is a higher-level cache that caches the key-value pairs returned by the SSTable interface to the tablet server code. The Block Cache is a lower-level cache that caches SSTables blocks that were read from GFS. The Scan Cache is most useful for applications that tend to read the same data repeatedly. The Block Cache is useful for applications that tend to read data that is close to the data they recently read (e.g., sequential reads, or random reads of different columns in the same locality group within a hot row).
<!--col-->
为了提升读性能，tablet 服务器使用两级缓存。Scan Cache 是较上层的缓存，缓存 SSTable 接口返回给 tablet 服务器代码的 key/value 对。Block Cache 是较下层的缓存，缓存从 GFS 读出的 SSTable 块。Scan Cache 对倾向于反复读取同一批数据的应用最有用。Block Cache 对倾向于读取与最近读过的数据相邻的数据的应用有用（例如顺序读，或者对某个热行内同一局部组中不同列的随机读）。
{{% /bilingual %}}

### Bloom filters · 布隆过滤器

{{% bilingual %}}
As described in Section 5.3, a read operation has to read from all SSTables that make up the state of a tablet. If these SSTables are not in memory, we may end up doing many disk accesses. We reduce the number of accesses by allowing clients to specify that Bloom filters [7] should be created for SSTables in a particular locality group. A Bloom filter allows us to ask whether an SSTable might contain any data for a specified row/column pair. For certain applications, a small amount of tablet server memory used for storing Bloom filters drastically reduces the number of disk seeks required for read operations. Our use of Bloom filters also implies that most lookups for non-existent rows or columns do not need to touch disk.
<!--col-->
如 5.3 节所述，一次读操作必须从构成某个 tablet 状态的所有 SSTable 中读取。如果这些 SSTable 不在内存里，我们可能就要做很多次磁盘访问。我们通过允许客户端指定为某个特定局部性组的 SSTable 创建布隆过滤器 [7]，来减少访问次数。布隆过滤器让我们能询问某个 SSTable 是否可能含有指定「行/列」对的任何数据。对某些应用而言，用少量 tablet 服务器内存存放布隆过滤器，就能大幅减少读操作所需的磁盘寻道次数。使用布隆过滤器还意味着，对不存在的行或列的大多数查找都不必碰磁盘。
{{% /bilingual %}}

### Commit-log implementation · 提交日志的实现

{{% bilingual %}}
If we kept the commit log for each tablet in a separate log file, a very large number of files would be written concurrently in GFS. Depending on the underlying file system implementation on each GFS server, these writes could cause a large number of disk seeks to write to the different physical log files. In addition, having separate log files per tablet also reduces the effectiveness of the group commit optimization, since groups would tend to be smaller. To fix these issues, we append mutations to a single commit log per tablet server, co-mingling mutations for different tablets in the same physical log file [18, 20].
<!--col-->
如果我们为每个 tablet 各维护一个独立的提交日志文件，那么 GFS 中将有极大量的文件被并发写入。取决于各 GFS 服务器上底层文件系统的实现，这些写入可能导致大量的磁盘寻道，因为要写到不同的物理日志文件。此外，每个 tablet 一个日志文件也会降低组提交优化的效果，因为组往往会更小。为解决这些问题，我们把修改追加到每个 tablet 服务器唯一的提交日志中，把不同 tablet 的修改混放在同一个物理日志文件里 [18, 20]。
{{% /bilingual %}}

{{% bilingual %}}
Using one log provides significant performance benefits during normal operation, but it complicates recovery. When a tablet server dies, the tablets that it served will be moved to a large number of other tablet servers: each server typically loads a small number of the original server's tablets. To recover the state for a tablet, the new tablet server needs to reapply the mutations for that tablet from the commit log written by the original tablet server. However, the mutations for these tablets were co-mingled in the same physical log file. One approach would be for each new tablet server to read this full commit log file and apply just the entries needed for the tablets it needs to recover. However, under such a scheme, if 100 machines were each assigned a single tablet from a failed tablet server, then the log file would be read 100 times (once by each server).
<!--col-->
使用单个日志在正常运行期间带来显著的性能收益，但它让恢复变得复杂。当一台 tablet 服务器挂掉时，它服务的 tablet 会被迁移到一大批其他 tablet 服务器上：每台服务器通常只加载原服务器的一小部分 tablet。要恢复某个 tablet 的状态，新的 tablet 服务器需要从原 tablet 服务器写的提交日志中重新施加属于该 tablet 的修改。然而这些 tablet 的修改是混放在同一个物理日志文件里的。一种做法是让每台新的 tablet 服务器读取整个提交日志文件，只施加它需要恢复的那些 tablet 的条目。但在这种方案下，如果有 100 台机器各自从失败的 tablet 服务器那里分到一个 tablet，那么这个日志文件会被读 100 次（每台服务器各读一次）。
{{% /bilingual %}}

{{% bilingual %}}
We avoid duplicating log reads by first sorting the commit log entries in order of the keys ⟨table, row name, log sequence number⟩. In the sorted output, all mutations for a particular tablet are contiguous and can therefore be read efficiently with one disk seek followed by a sequential read. To parallelize the sorting, we partition the log file into 64 MB segments, and sort each segment in parallel on different tablet servers. This sorting process is coordinated by the master and is initiated when a tablet server indicates that it needs to recover mutations from some commit log file.
<!--col-->
我们避免重复读取日志的方式是：先按 ⟨表, 行名, 日志序号⟩ 的键序对提交日志条目排序。在排序后的输出中，属于某个特定 tablet 的所有修改都是连续的，因此可以用一次磁盘寻道加一次顺序读取高效地读完。为并行化排序，我们把日志文件切成 64 MB 的段，在不同 tablet 服务器上并行排序各段。这个排序过程由 master 协调，并在某个 tablet 服务器表示自己需要从某个提交日志文件恢复修改时启动。
{{% /bilingual %}}

{{% bilingual %}}
Writing commit logs to GFS sometimes causes performance hiccups for a variety of reasons (e.g., a GFS server machine involved in the write crashes, or the network paths traversed to reach the particular set of three GFS servers is suffering network congestion, or is heavily loaded). To protect mutations from GFS latency spikes, each tablet server actually has two log writing threads, each writing to its own log file; only one of these two threads is actively in use at a time. If writes to the active log file are performing poorly, the log file writing is switched to the other thread, and mutations that are in the commit log queue are written by the newly active log writing thread. Log entries contain sequence numbers to allow the recovery process to elide duplicated entries resulting from this log switching process.
<!--col-->
把提交日志写入 GFS 有时会因各种原因造成性能抖动（例如参与写入的某台 GFS 服务器机器崩溃，或者通往特定那三台 GFS 服务器的网络路径正在拥塞、负载很重）。为保护修改不受 GFS 延迟尖峰的影响，每台 tablet 服务器实际上有两个日志写入线程，各自写自己的日志文件；同一时刻只有一个线程在使用。如果写入当前活跃日志文件的性能很差，日志写入就切换到另一个线程，而位于提交日志队列中的修改由新的活跃写入线程写出。日志条目含有序号，以便恢复过程能剔除这种日志切换造成的重复条目。
{{% /bilingual %}}

### Speeding up tablet recovery · 加速 tablet 恢复

{{% bilingual %}}
If the master moves a tablet from one tablet server to another, the source tablet server first does a minor compaction on that tablet. This compaction reduces recovery time by reducing the amount of uncompacted state in the tablet server's commit log. After finishing this compaction, the tablet server stops serving the tablet. Before it actually unloads the tablet, the tablet server does another (usually very fast) minor compaction to eliminate any remaining uncompacted state in the tablet server's log that arrived while the first minor compaction was being performed. After this second minor compaction is complete, the tablet can be loaded on another tablet server without requiring any recovery of log entries.
<!--col-->
如果 master 要把一个 tablet 从一台 tablet 服务器迁到另一台，源 tablet 服务器先对该 tablet 做一次小合并。这次合并通过减少该 tablet 服务器提交日志中未合并状态的数量来缩短恢复时间。完成这次合并后，tablet 服务器停止服务该 tablet。在真正卸载该 tablet 之前，tablet 服务器还会再做一次（通常非常快的）小合并，以清除第一次小合并进行期间到达该服务器日志的剩余未合并状态。第二次小合并完成后，这个 tablet 就可以被加载到另一台 tablet 服务器上，而不需要恢复任何日志条目。
{{% /bilingual %}}

### Exploiting immutability · 利用不可变性

{{% bilingual %}}
Besides the SSTable caches, various other parts of the Bigtable system have been simplified by the fact that all of the SSTables that we generate are immutable. For example, we do not need any synchronization of accesses to the file system when reading from SSTables. As a result, concurrency control over rows can be implemented very efficiently. The only mutable data structure that is accessed by both reads and writes is the memtable. To reduce contention during reads of the memtable, we make each memtable row copy-on-write and allow reads and writes to proceed in parallel.
<!--col-->
除了 SSTable 缓存之外，Bigtable 系统的其他多个部分也因为「我们生成的所有 SSTable 都是不可变的」这一事实而得到简化。例如，从 SSTable 读取时，我们不需要对文件系统访问做任何同步。因此，针对行的并发控制可以非常高效地实现。唯一会被读写同时访问的可变数据结构是 memtable。为减少读取 memtable 时的争用，我们让 memtable 的每一行都采用写时复制（copy-on-write），并允许读写并行进行。
{{% /bilingual %}}

{{% bilingual %}}
Since SSTables are immutable, the problem of permanently removing deleted data is transformed to garbage collecting obsolete SSTables. Each tablet's SSTables are registered in the METADATA table. The master removes obsolete SSTables as a mark-and-sweep garbage collection [25] over the set of SSTables, where the METADATA table contains the set of roots.
<!--col-->
由于 SSTable 不可变，「永久删除已删除数据」的问题就转化成了「回收废弃 SSTable」的垃圾回收问题。每个 tablet 的 SSTable 都登记在 METADATA 表中。master 在一组 SSTable 上以标记—清除（mark-and-sweep）垃圾回收 [25] 的方式移除废弃 SSTable，其中 METADATA 表充当根集合。
{{% /bilingual %}}

{{% bilingual %}}
Finally, the immutability of SSTables enables us to split tablets quickly. Instead of generating a new set of SSTables for each child tablet, we let the child tablets share the SSTables of the parent tablet.
<!--col-->
最后，SSTable 的不可变性让我们能快速分裂 tablet。我们不为每个子 tablet 生成一组新的 SSTable，而是让子 tablet 共享父 tablet 的 SSTable。
{{% /bilingual %}}

---

## 7 Performance Evaluation · 性能评估

{{% bilingual %}}
We set up a Bigtable cluster with N tablet servers to measure the performance and scalability of Bigtable as N is varied. The tablet servers were configured to use 1 GB of memory and to write to a GFS cell consisting of 1786 machines with two 400 GB IDE hard drives each. N client machines generated the Bigtable load used for these tests. (We used the same number of clients as tablet servers to ensure that clients were never a bottleneck.) Each machine had two dual-core Opteron 2 GHz chips, enough physical memory to hold the working set of all running processes, and a single gigabit Ethernet link. The machines were arranged in a two-level tree-shaped switched network with approximately 100-200 Gbps of aggregate bandwidth available at the root. All of the machines were in the same hosting facility and therefore the round-trip time between any pair of machines was less than a millisecond.
<!--col-->
我们搭建了一个含 N 台 tablet 服务器的 Bigtable 集群，以测量 Bigtable 在 N 变化时的性能与可扩展性。tablet 服务器配置为使用 1 GB 内存，写入一个由 1786 台机器组成的 GFS cell，每台机器带两块 400 GB IDE 硬盘。N 台客户端机器产生这些测试所用的 Bigtable 负载。（我们使用的客户端数量与 tablet 服务器数量相同，以确保客户端永远不会成为瓶颈。）每台机器配两颗双核 Opteron 2 GHz 芯片、足以容纳所有运行进程工作集的物理内存，以及一条千兆以太网链路。机器组织成两级树形交换网络，根部可用聚合带宽约 100–200 Gbps。所有机器都在同一个机房内，因此任意两台机器之间的往返时间都不到 1 毫秒。
{{% /bilingual %}}

{{% bilingual %}}
The tablet servers and master, test clients, and GFS servers all ran on the same set of machines. Every machine ran a GFS server. Some of the machines also ran either a tablet server, or a client process, or processes from other jobs that were using the pool at the same time as these experiments.
<!--col-->
tablet 服务器与 master、测试客户端以及 GFS 服务器都跑在同一批机器上。每台机器都运行一个 GFS 服务器。部分机器还同时运行着 tablet 服务器、客户端进程，或者在这些实验进行时同样使用这个机器池的其他作业进程。
{{% /bilingual %}}

{{% bilingual %}}
R is the distinct number of Bigtable row keys involved in the test. R was chosen so that each benchmark read or wrote approximately 1 GB of data per tablet server.
<!--col-->
R 是测试涉及的 Bigtable 行键的不同数量。R 的取值使得每个基准测试对每台 tablet 服务器大致读写 1 GB 数据。
{{% /bilingual %}}

{{% bilingual %}}
The sequential write benchmark used row keys with names 0 to R −1. This space of row keys was partitioned into 10N equal-sized ranges. These ranges were assigned to the N clients by a central scheduler that assigned the next available range to a client as soon as the client finished processing the previous range assigned to it. This dynamic assignment helped mitigate the effects of performance variations caused by other processes running on the client machines. We wrote a single string under each row key. Each string was generated randomly and was therefore uncompressible. In addition, strings under different row key were distinct, so no cross-row compression was possible. The random write benchmark was similar except that the row key was hashed modulo R immediately before writing so that the write load was spread roughly uniformly across the entire row space for the entire duration of the benchmark.
<!--col-->
顺序写基准使用命名从 0 到 R−1 的行键。这个行键空间被划分成 10N 个等大小的区间。这些区间由中心调度器分配给 N 个客户端：一旦某个客户端处理完分配给它的上一个区间，调度器就把下一个可用区间分配给它。这种动态分配有助于缓解客户端机器上其他进程造成的性能波动。我们在每个行键下写一个字符串。每个字符串都是随机生成的，因此不可压缩。此外，不同行键下的字符串互不相同，所以不可能跨行压缩。随机写基准与此类似，只是在写入之前先把行键对 R 取哈希，从而在整个基准测试期间把写入负载大致均匀地摊到整个行键空间上。
{{% /bilingual %}}

{{% bilingual %}}
The sequential read benchmark generated row keys in exactly the same way as the sequential write benchmark, but instead of writing under the row key, it read the string stored under the row key (which was written by an earlier invocation of the sequential write benchmark). Similarly, the random read benchmark shadowed the operation of the random write benchmark.
<!--col-->
顺序读基准生成行键的方式与顺序写基准完全相同，但它不在行键下写入，而是读取该行键下存放的字符串（由早先一次顺序写基准写入）。类似地，随机读基准对照随机写基准的操作进行。
{{% /bilingual %}}

{{% bilingual %}}
The scan benchmark is similar to the sequential read benchmark, but uses support provided by the Bigtable API for scanning over all values in a row range. Using a scan reduces the number of RPCs executed by the benchmark since a single RPC fetches a large sequence of values from a tablet server.
<!--col-->
扫描基准与顺序读基准类似，但使用的是 Bigtable API 提供的、对一个行区间内所有值做扫描的能力。使用扫描减少了基准测试执行的 RPC 次数，因为单次 RPC 就能从 tablet 服务器取回一大串值。
{{% /bilingual %}}

{{% bilingual %}}
The random reads (mem) benchmark is similar to the random read benchmark, but the locality group that contains the benchmark data is marked as in-memory, and therefore the reads are satisfied from the tablet server's memory instead of requiring a GFS read. For just this benchmark, we reduced the amount of data per tablet server from 1 GB to 100 MB so that it would fit comfortably in the memory available to the tablet server.
<!--col-->
随机读（内存）基准与随机读基准类似，但包含基准数据的局部性组被标记为驻留内存，因此读由 tablet 服务器的内存满足，而不需要读 GFS。仅对这个基准，我们把每台 tablet 服务器的数据量从 1 GB 降到 100 MB，以便它舒适地放进 tablet 服务器可用的内存。
{{% /bilingual %}}

{{% bilingual %}}
Figure 6 shows two views on the performance of our benchmarks when reading and writing 1000-byte values to Bigtable. The table shows the number of operations per second per tablet server; the graph shows the aggregate number of operations per second.
<!--col-->
图 6 从两个视角展示了这些基准在向 Bigtable 读写 1000 字节值时的性能。表格给出每台 tablet 服务器每秒的操作数，图形给出每秒的聚合操作数。
{{% /bilingual %}}

表：每台 tablet 服务器每秒读写 1000 字节值的数量（原文图 6 中的表格部分；原文另有以「tablet 服务器数量」为横轴、以「每秒读写的值数量」为纵轴的聚合折线图）

| 实验 | 1 台 | 50 台 | 250 台 | 500 台 |
|------|------|-------|--------|--------|
| 随机读 | 1212 | 593 | 479 | 241 |
| 随机读（内存） | 10811 | 8511 | 8000 | 6250 |
| 随机写 | 8850 | 3745 | 3425 | 2000 |
| 顺序读 | 4425 | 2463 | 2625 | 2469 |
| 顺序写 | 8547 | 3623 | 2451 | 1905 |
| 扫描 | 15385 | 10526 | 9524 | 7843 |

{{% bilingual %}}
Let us first consider performance with just one tablet server. Random reads are slower than all other operations by an order of magnitude or more. Each random read involves the transfer of a 64 KB SSTable block over the network from GFS to a tablet server, out of which only a single 1000-byte value is used. The tablet server executes approximately 1200 reads per second, which translates into approximately 75 MB/s of data read from GFS. This bandwidth is enough to saturate the tablet server CPUs because of overheads in our networking stack, SSTable parsing, and Bigtable code, and is also almost enough to saturate the network links used in our system. Most Bigtable applications with this type of an access pattern reduce the block size to a smaller value, typically 8KB. Random reads from memory are much faster since each 1000-byte read is satisfied from the tablet server's local memory without fetching a large 64 KB block from GFS.
<!--col-->
先看只有一台 tablet 服务器时的性能。随机读比所有其他操作都慢一个数量级或更多。每次随机读都要从 GFS 经网络把 64 KB 的 SSTable 块传到 tablet 服务器，而其中只用到一个 1000 字节的值。该 tablet 服务器每秒执行约 1200 次读，折算下来相当于从 GFS 读取约 75 MB/s 的数据。由于我们的网络协议栈、SSTable 解析和 Bigtable 代码本身的开销，这个带宽已足以把 tablet 服务器的 CPU 打满，也几乎足以把系统中使用的网络链路打满。大多数具有这类访问模式的 Bigtable 应用会把块大小调小，通常调到 8 KB。从内存做随机读则快得多，因为每次 1000 字节的读都由 tablet 服务器的本地内存满足，不必从 GFS 取回 64 KB 的大块。
{{% /bilingual %}}

{{% bilingual %}}
Random and sequential writes perform better than random reads since each tablet server appends all incoming writes to a single commit log and uses group commit to stream these writes efficiently to GFS. There is no significant difference between the performance of random writes and sequential writes; in both cases, all writes to the tablet server are recorded in the same commit log.
<!--col-->
随机写和顺序写的表现好于随机读，因为每台 tablet 服务器把所有到来的写都追加到同一个提交日志，并用组提交把这些写高效地流式写入 GFS。随机写与顺序写的性能没有显著差异；两种情况下，对该 tablet 服务器的所有写都记录在同一个提交日志里。
{{% /bilingual %}}

{{% bilingual %}}
Sequential reads perform better than random reads since every 64 KB SSTable block that is fetched from GFS is stored into our block cache, where it is used to serve the next 64 read requests.
<!--col-->
顺序读的表现好于随机读，因为从 GFS 取回的每个 64 KB SSTable 块都会存入我们的块缓存，并被用来服务接下来的 64 次读请求。
{{% /bilingual %}}

{{% bilingual %}}
Scans are even faster since the tablet server can return a large number of values in response to a single client RPC, and therefore RPC overhead is amortized over a large number of values.
<!--col-->
扫描还要更快，因为 tablet 服务器可以针对单次客户端 RPC 返回大量值，于是 RPC 开销被摊到大量值上。
{{% /bilingual %}}

{{% bilingual %}}
Aggregate throughput increases dramatically, by over a factor of a hundred, as we increase the number of tablet servers in the system from 1 to 500. For example, the performance of random reads from memory increases by almost a factor of 300 as the number of tablet server increases by a factor of 500. This behavior occurs because the bottleneck on performance for this benchmark is the individual tablet server CPU.
<!--col-->
当我们把系统中的 tablet 服务器数量从 1 增加到 500 时，聚合吞吐量大幅提升，超过一百倍。例如，当 tablet 服务器数量增加 500 倍时，从内存做随机读的性能几乎提升了 300 倍。出现这种表现是因为该基准的性能瓶颈在于单台 tablet 服务器的 CPU。
{{% /bilingual %}}

{{% bilingual %}}
However, performance does not increase linearly. For most benchmarks, there is a significant drop in per-server throughput when going from 1 to 50 tablet servers. This drop is caused by imbalance in load in multiple server configurations, often due to other processes contending for CPU and network. Our load balancing algorithm attempts to deal with this imbalance, but cannot do a perfect job for two main reasons: rebalancing is throttled to reduce the number of tablet movements (a tablet is unavailable for a short time, typically less than one second, when it is moved), and the load generated by our benchmarks shifts around as the benchmark progresses.
<!--col-->
然而，性能并非线性增长。对大多数基准而言，从 1 台 tablet 服务器增加到 50 台时，单服务器吞吐量有明显下降。这个下降由多服务器配置下的负载不均衡引起，通常是因为其他进程在争抢 CPU 和网络。我们的负载均衡算法会尝试应对这种不均衡，但做不到完美，主要有两个原因：为了减少 tablet 迁移次数，再均衡被限流（tablet 被迁移时会短暂不可用，通常不到一秒）；以及基准产生的负载会随着基准的推进而漂移。
{{% /bilingual %}}

{{% bilingual %}}
The random read benchmark shows the worst scaling (an increase in aggregate throughput by only a factor of 100 for a 500-fold increase in number of servers). This behavior occurs because (as explained above) we transfer one large 64KB block over the network for every 1000-byte read. This transfer saturates various shared 1 Gigabit links in our network and as a result, the per-server throughput drops significantly as we increase the number of machines.
<!--col-->
随机读基准的可扩展性最差（服务器数量增加 500 倍，聚合吞吐量只提升 100 倍）。出现这种表现的原因是（如上所述）每做一次 1000 字节的读，我们都要经网络传输一个 64 KB 的大块。这种传输会把网络中若干共享的千兆链路打满，结果就是随着机器数量增加，单服务器吞吐量显著下降。
{{% /bilingual %}}

---

## 8 Real Applications · 真实应用

{{% bilingual %}}
As of August 2006, there are 388 non-test Bigtable clusters running in various Google machine clusters, with a combined total of about 24,500 tablet servers. Table 1 shows a rough distribution of tablet servers per cluster.
<!--col-->
截至 2006 年 8 月，在 Google 的各个机器集群中共运行着 388 个非测试 Bigtable 集群，合计约 24,500 台 tablet 服务器。表 1 给出每个集群 tablet 服务器数量的大致分布。
{{% /bilingual %}}

表 1：Bigtable 集群中 tablet 服务器数量的分布（Distribution of number of tablet servers in Bigtable clusters）

| tablet 服务器数量 | 集群数量 |
|-------------------|----------|
| 0 .. 19 | 259 |
| 20 .. 49 | 47 |
| 50 .. 99 | 20 |
| 100 .. 499 | 50 |
| > 500 | 12 |

{{% bilingual %}}
Many of these clusters are used for development purposes and therefore are idle for significant periods. One group of 14 busy clusters with 8069 total tablet servers saw an aggregate volume of more than 1.2 million requests per second, with incoming RPC traffic of about 741 MB/s and outgoing RPC traffic of about 16 GB/s.
<!--col-->
这些集群中有许多用于开发目的，因此有相当长的时间处于空闲状态。有一组 14 个繁忙集群合计 8069 台 tablet 服务器，其聚合请求量超过每秒 120 万次，入向 RPC 流量约 741 MB/s，出向 RPC 流量约 16 GB/s。
{{% /bilingual %}}

{{% bilingual %}}
Table 2 provides some data about a few of the tables currently in use. Some tables store data that is served to users, whereas others store data for batch processing; the tables range widely in total size, average cell size, percentage of data served from memory, and complexity of the table schema. In the rest of this section, we briefly describe how three product teams use Bigtable.
<!--col-->
表 2 给出当前在使用中的几个表的一些数据。有些表存放面向用户服务的数据，另一些存放用于批处理的数据；这些表在总大小、平均单元大小、从内存服务的数据比例以及表 schema 的复杂度上都相差很大。本节余下部分简要介绍三个产品团队如何使用 Bigtable。
{{% /bilingual %}}

表 2：若干生产环境表（Characteristics of a few tables in production use）。表大小（压缩前测量）与单元数均为近似值；关闭压缩的表不给出压缩率。

| 项目名 | 表大小（TB） | 压缩率 | 单元数（十亿） | 列族数 | 局部性组数 | 内存中占比 | 延迟敏感 |
|--------|--------------|--------|----------------|--------|------------|------------|----------|
| Crawl | 800 | 11% | 1000 | 16 | 8 | 0% | 否 |
| Crawl | 50 | 33% | 200 | 2 | 2 | 0% | 否 |
| Google Analytics | 20 | 29% | 10 | 1 | 1 | 0% | 是 |
| Google Analytics | 200 | 14% | 80 | 1 | 1 | 0% | 是 |
| Google Base | 2 | 31% | 10 | 29 | 3 | 15% | 是 |
| Google Earth | 0.5 | 64% | 8 | 7 | 2 | 33% | 是 |
| Google Earth | 70 | - | 9 | 8 | 3 | 0% | 否 |
| Orkut | 9 | - | 0.9 | 8 | 5 | 1% | 是 |
| Personalized Search | 4 | 47% | 6 | 93 | 11 | 5% | 是 |

### 8.1 Google Analytics

{{% bilingual %}}
Google Analytics (analytics.google.com) is a service that helps webmasters analyze traffic patterns at their web sites. It provides aggregate statistics, such as the number of unique visitors per day and the page views per URL per day, as well as site-tracking reports, such as the percentage of users that made a purchase, given that they earlier viewed a specific page.
<!--col-->
Google Analytics（analytics.google.com）是一项帮助网站站长分析其站点流量模式的服务。它提供聚合统计，比如每天独立访客数和每个 URL 每天的页面浏览量，也提供网站跟踪报告，比如在先前浏览过某个特定页面的用户中完成购买的比例。
{{% /bilingual %}}

{{% bilingual %}}
To enable the service, webmasters embed a small JavaScript program in their web pages. This program is invoked whenever a page is visited. It records various information about the request in Google Analytics, such as a user identifier and information about the page being fetched. Google Analytics summarizes this data and makes it available to webmasters.
<!--col-->
要启用这项服务，站长需要在自己的网页中嵌入一小段 JavaScript 程序。每当页面被访问时，这段程序就会执行。它把关于该请求的各种信息记录到 Google Analytics 中，比如用户标识符和正在被抓取的页面的信息。Google Analytics 对这些数据做汇总，并提供给站长使用。
{{% /bilingual %}}

{{% bilingual %}}
We briefly describe two of the tables used by Google Analytics. The raw click table (˜200 TB) maintains a row for each end-user session. The row name is a tuple containing the website's name and the time at which the session was created. This schema ensures that sessions that visit the same web site are contiguous, and that they are sorted chronologically. This table compresses to 14% of its original size.
<!--col-->
我们简要介绍 Google Analytics 使用的两个表。原始点击表（raw click table，约 200 TB）为每个终端用户会话维护一行。行名是一个元组，包含网站名和会话创建时间。这个 schema 确保访问同一网站的会话是连续的，并且按时间先后排序。这个表压缩后为原始大小的 14%。
{{% /bilingual %}}

{{% bilingual %}}
The summary table (˜20 TB) contains various predefined summaries for each website. This table is generated from the raw click table by periodically scheduled MapReduce jobs. Each MapReduce job extracts recent session data from the raw click table. The overall system's throughput is limited by the throughput of GFS. This table compresses to 29% of its original size.
<!--col-->
汇总表（summary table，约 20 TB）包含为每个网站预先定义的各种汇总。这个表由周期性调度的 MapReduce 作业从原始点击表生成。每个 MapReduce 作业从原始点击表中抽取近期的会话数据。整个系统的吞吐受 GFS 吞吐的限制。这个表压缩后为原始大小的 29%。
{{% /bilingual %}}

### 8.2 Google Earth

{{% bilingual %}}
Google operates a collection of services that provide users with access to high-resolution satellite imagery of the world's surface, both through the web-based Google Maps interface (maps.google.com) and through the Google Earth (earth.google.com) custom client software. These products allow users to navigate across the world's surface: they can pan, view, and annotate satellite imagery at many different levels of resolution. This system uses one table to preprocess data, and a different set of tables for serving client data.
<!--col-->
Google 运营着一组服务，让用户能访问世界地表的高分辨率卫星影像，其入口既有基于 Web 的 Google Maps 界面（maps.google.com），也有 Google Earth（earth.google.com）的自定义客户端软件。这些产品让用户能在地表上漫游：他们可以在许多不同的分辨率层级上平移、查看和标注卫星影像。这个系统用一个表做数据预处理，并用另一组表来向客户端提供数据服务。
{{% /bilingual %}}

{{% bilingual %}}
The preprocessing pipeline uses one table to store raw imagery. During preprocessing, the imagery is cleaned and consolidated into final serving data. This table contains approximately 70 terabytes of data and therefore is served from disk. The images are efficiently compressed already, so Bigtable compression is disabled.
<!--col-->
预处理流水线用一个表存放原始影像。在预处理过程中，影像被清洗并合并成最终的服务数据。这个表包含约 70 TB 数据，因此从磁盘提供服务。这些图像本身已经被高效压缩过，所以 Bigtable 的压缩被关闭。
{{% /bilingual %}}

{{% bilingual %}}
Each row in the imagery table corresponds to a single geographic segment. Rows are named to ensure that adjacent geographic segments are stored near each other. The table contains a column family to keep track of the sources of data for each segment. This column family has a large number of columns: essentially one for each raw data image. Since each segment is only built from a few images, this column family is very sparse.
<!--col-->
影像表中每一行对应一个地理分片。行的命名确保相邻的地理分片存放在彼此附近。表中有一个列族用于跟踪每个分片的数据来源。这个列族有非常多的列：基本上每张原始数据影像一列。由于每个分片只由少数几张影像构成，这个列族非常稀疏。
{{% /bilingual %}}

{{% bilingual %}}
The preprocessing pipeline relies heavily on MapReduce over Bigtable to transform data. The overall system processes over 1 MB/sec of data per tablet server during some of these MapReduce jobs.
<!--col-->
预处理流水线大量依赖在 Bigtable 之上跑 MapReduce 来做数据变换。在其中一些 MapReduce 作业期间，整个系统每台 tablet 服务器处理的数据超过 1 MB/s。
{{% /bilingual %}}

{{% bilingual %}}
The serving system uses one table to index data stored in GFS. This table is relatively small (˜500 GB), but it must serve tens of thousands of queries per second per datacenter with low latency. As a result, this table is hosted across hundreds of tablet servers and contains in-memory column families.
<!--col-->
服务系统用一个表为存放在 GFS 中的数据建立索引。这个表相对较小（约 500 GB），但它必须为每个数据中心以低延迟服务每秒数万次查询。因此，这个表被托管在数百台 tablet 服务器上，并包含驻留内存的列族。
{{% /bilingual %}}

### 8.3 Personalized Search

{{% bilingual %}}
Personalized Search (www.google.com/psearch) is an opt-in service that records user queries and clicks across a variety of Google properties such as web search, images, and news. Users can browse their search histories to revisit their old queries and clicks, and they can ask for personalized search results based on their historical Google usage patterns. Personalized Search stores each user's data in Bigtable. Each user has a unique userid and is assigned a row named by that userid. All user actions are stored in a table. A separate column family is reserved for each type of action (for example, there is a column family that stores all web queries). Each data element uses as its Bigtable timestamp the time at which the corresponding user action occurred. Personalized Search generates user profiles using a MapReduce over Bigtable. These user profiles are used to personalize live search results.
<!--col-->
Personalized Search（www.google.com/psearch）是一项需用户主动选择加入的服务，它记录用户在 Google 各类产品（如网页搜索、图片、新闻）上的查询和点击。用户可以浏览自己的搜索历史，重温过去的查询与点击，也可以基于自己历史上的 Google 使用模式要求个性化的搜索结果。Personalized Search 把每个用户的数据存放在 Bigtable 中。每个用户有一个唯一的 userid，并被分配一行以该 userid 命名。所有用户动作都存放在一个表里。每种动作类型各有一个专属列族（例如有一个列族存放所有网页查询）。每个数据元素以其对应的用户动作发生时间作为 Bigtable 时间戳。Personalized Search 通过在 Bigtable 上跑 MapReduce 生成用户画像。这些用户画像被用于个性化实时搜索结果。
{{% /bilingual %}}

{{% bilingual %}}
The Personalized Search data is replicated across several Bigtable clusters to increase availability and to reduce latency due to distance from clients. The Personalized Search team originally built a client-side replication mechanism on top of Bigtable that ensured eventual consistency of all replicas. The current system now uses a replication subsystem that is built into the servers.
<!--col-->
Personalized Search 的数据被复制到若干 Bigtable 集群，以提升可用性并降低因与客户端距离而产生的延迟。Personalized Search 团队最初在 Bigtable 之上自建了一套客户端复制机制，确保所有副本最终一致。当前的系统则使用内建在服务器里的复制子系统。
{{% /bilingual %}}

{{% bilingual %}}
The design of the Personalized Search storage system allows other groups to add new per-user information in their own columns, and the system is now used by many other Google properties that need to store per-user configuration options and settings. Sharing a table amongst many groups resulted in an unusually large number of column families. To help support sharing, we added a simple quota mechanism to Bigtable to limit the storage consumption by any particular client in shared tables; this mechanism provides some isolation between the various product groups using this system for per-user information storage.
<!--col-->
Personalized Search 存储系统的设计允许其他团队在自己的列中添加新的按用户区分的信息，如今许多需要存放按用户区分的配置项与设置的 Google 产品都在使用这套系统。多个团队共享一个表，导致列族数量异常地多。为支持共享，我们给 Bigtable 加了一个简单的配额机制，用来限制任何特定客户端在共享表中的存储消耗；这个机制为使用该系统存放按用户信息的各个产品团队之间提供了一定的隔离。
{{% /bilingual %}}

---

## 9 Lessons · 教训

{{% bilingual %}}
In the process of designing, implementing, maintaining, and supporting Bigtable, we gained useful experience and learned several interesting lessons.
<!--col-->
在设计、实现、维护和支持 Bigtable 的过程中，我们积累了有用的经验，也得到了一些有意思的教训。
{{% /bilingual %}}

{{% bilingual %}}
One lesson we learned is that large distributed systems are vulnerable to many types of failures, not just the standard network partitions and fail-stop failures assumed in many distributed protocols. For example, we have seen problems due to all of the following causes: memory and network corruption, large clock skew, hung machines, extended and asymmetric network partitions, bugs in other systems that we are using (Chubby for example), overflow of GFS quotas, and planned and unplanned hardware maintenance. As we have gained more experience with these problems, we have addressed them by changing various protocols. For example, we added checksumming to our RPC mechanism. We also handled some problems by removing assumptions made by one part of the system about another part. For example, we stopped assuming a given Chubby operation could return only one of a fixed set of errors.
<!--col-->
我们得到的一条教训是：大型分布式系统容易遭受许多类型的故障，而不只是许多分布式协议所假设的标准网络分区和故障停止（fail-stop）故障。例如，我们见过由下列全部原因引起的问题：内存与网络损坏、严重的时钟偏移、机器挂死、长时间且不对称的网络分区、我们所使用其他系统（比如 Chubby）中的 bug、GFS 配额溢出，以及计划内和计划外的硬件维护。随着在这些问题上积累更多经验，我们通过修改各种协议来解决它们。例如，我们给 RPC 机制加上了校验和。我们还通过移除系统中某一部分对另一部分所做的假设来解决一些问题。例如，我们不再假设某个 Chubby 操作只可能返回一组固定错误中的某一个。
{{% /bilingual %}}

{{% bilingual %}}
Another lesson we learned is that it is important to delay adding new features until it is clear how the new features will be used. For example, we initially planned to support general-purpose transactions in our API. Because we did not have an immediate use for them, however, we did not implement them. Now that we have many real applications running on Bigtable, we have been able to examine their actual needs, and have discovered that most applications require only single-row transactions. Where people have requested distributed transactions, the most important use is for maintaining secondary indices, and we plan to add a specialized mechanism to satisfy this need. The new mechanism will be less general than distributed transactions, but will be more efficient (especially for updates that span hundreds of rows or more) and will also interact better with our scheme for optimistic cross-data-center replication.
<!--col-->
另一条教训是：在新特性将被如何使用还不清楚之前，先不要急着加它。例如，我们最初计划在 API 中支持通用事务。但由于当时没有立即的用处，我们就没有实现。如今 Bigtable 上跑着许多真实应用，我们得以考察它们的实际需求，结果发现大多数应用只需要单行事务。在人们要求分布式事务的场景里，最重要的用途是维护二级索引，我们计划加一个专门的机制来满足这一需求。这个新机制会比分布式事务通用性更弱，但更高效（尤其是对跨数百行以上的更新），并且能更好地与我们那套乐观的跨数据中心复制方案配合。
{{% /bilingual %}}

{{% bilingual %}}
A practical lesson that we learned from supporting Bigtable is the importance of proper system-level monitoring (i.e., monitoring both Bigtable itself, as well as the client processes using Bigtable). For example, we extended our RPC system so that for a sample of the RPCs, it keeps a detailed trace of the important actions done on behalf of that RPC. This feature has allowed us to detect and fix many problems such as lock contention on tablet data structures, slow writes to GFS while committing Bigtable mutations, and stuck accesses to the METADATA table when METADATA tablets are unavailable. Another example of useful monitoring is that every Bigtable cluster is registered in Chubby. This allows us to track down all clusters, discover how big they are, see which versions of our software they are running, how much traffic they are receiving, and whether or not there are any problems such as unexpectedly large latencies.
<!--col-->
从支持 Bigtable 的过程中，我们得到一条实践教训：恰当的系统级监控非常重要（也就是既监控 Bigtable 本身，也监控使用 Bigtable 的客户端进程）。例如，我们扩展了 RPC 系统，对抽样的 RPC 记录其为该 RPC 所做重要动作的详细轨迹。这个特性让我们发现并修复了许多问题，比如 tablet 数据结构上的锁争用、提交 Bigtable 修改时写入 GFS 缓慢，以及 METADATA tablet 不可用时对 METADATA 表的访问卡住。另一个有用的监控例子是：每个 Bigtable 集群都在 Chubby 中注册。这让我们能找出所有集群、发现它们有多大、看它们跑的是哪个软件版本、接收多少流量，以及是否存在诸如延迟异常偏大之类的问题。
{{% /bilingual %}}

{{% bilingual %}}
The most important lesson we learned is the value of simple designs. Given both the size of our system (about 100,000 lines of non-test code), as well as the fact that code evolves over time in unexpected ways, we have found that code and design clarity are of immense help in code maintenance and debugging. One example of this is our tablet-server membership protocol. Our first protocol was simple: the master periodically issued leases to tablet servers, and tablet servers killed themselves if their lease expired. Unfortunately, this protocol reduced availability significantly in the presence of network problems, and was also sensitive to master recovery time. We redesigned the protocol several times until we had a protocol that performed well. However, the resulting protocol was too complex and depended on the behavior of Chubby features that were seldom exercised by other applications. We discovered that we were spending an inordinate amount of time debugging obscure corner cases, not only in Bigtable code, but also in Chubby code. Eventually, we scrapped this protocol and moved to a newer simpler protocol that depends solely on widely-used Chubby features.
<!--col-->
我们得到的最重要的一条教训是：简单设计有价值。考虑到我们系统的规模（约 10 万行非测试代码），以及代码会以出人意料的方式随时间演化这一事实，我们发现代码与设计上的清晰性对代码维护和调试帮助极大。一个例子是我们的 tablet 服务器成员协议。我们的第一个协议很简单：master 周期性地向 tablet 服务器发放租约，租约到期时 tablet 服务器就杀掉自己。不幸的是，这个协议在网络出现问题时会严重降低可用性，而且对 master 的恢复时间很敏感。我们把它重新设计了好几次，直到得到一个表现良好的协议。然而，最终的协议过于复杂，且依赖 Chubby 中其他应用很少用到的特性行为。我们发现自己在调试晦涩的边角情况上花了过多时间，这既包括 Bigtable 的代码，也包括 Chubby 的代码。最终我们废弃了这个协议，改用一个新的、更简单的协议，它只依赖被广泛使用的 Chubby 特性。
{{% /bilingual %}}

---

## 10 Related Work · 相关工作

{{% bilingual %}}
The Boxwood project [24] has components that overlap in some ways with Chubby, GFS, and Bigtable, since it provides for distributed agreement, locking, distributed chunk storage, and distributed B-tree storage. In each case where there is overlap, it appears that the Boxwood's component is targeted at a somewhat lower level than the corresponding Google service. The Boxwood project's goal is to provide infrastructure for building higher-level services such as file systems or databases, while the goal of Bigtable is to directly support client applications that wish to store data.
<!--col-->
Boxwood 项目 [24] 的某些组件与 Chubby、GFS 和 Bigtable 有重叠，因为它提供了分布式共识、锁、分布式块存储和分布式 B 树存储。在每一处重叠的地方，Boxwood 的组件看起来都比对应的 Google 服务面向更低的层次。Boxwood 项目的目标是为构建文件系统或数据库这类更高层服务提供基础设施，而 Bigtable 的目标是直接支持希望存放数据的客户端应用。
{{% /bilingual %}}

{{% bilingual %}}
Many recent projects have tackled the problem of providing distributed storage or higher-level services over wide area networks, often at "Internet scale." This includes work on distributed hash tables that began with projects such as CAN [29], Chord [32], Tapestry [37], and Pastry [30]. These systems address concerns that do not arise for Bigtable, such as highly variable bandwidth, untrusted participants, or frequent reconfiguration; decentralized control and Byzantine fault tolerance are not Bigtable goals.
<!--col-->
近期许多项目处理的是在广域网上（常常是「互联网规模」）提供分布式存储或更高层服务的问题。这包括分布式哈希表方面的工作，始于 CAN [29]、Chord [32]、Tapestry [37]、Pastry [30] 等项目。这些系统应对的是 Bigtable 不会遇到的问题，比如带宽剧烈变化、参与者不可信、频繁重新配置；去中心化控制和拜占庭容错并不是 Bigtable 的目标。
{{% /bilingual %}}

{{% bilingual %}}
In terms of the distributed data storage model that one might provide to application developers, we believe the key-value pair model provided by distributed B-trees or distributed hash tables is too limiting. Key-value pairs are a useful building block, but they should not be the only building block one provides to developers. The model we chose is richer than simple key-value pairs, and supports sparse semi-structured data. Nonetheless, it is still simple enough that it lends itself to a very efficient flat-file representation, and it is transparent enough (via locality groups) to allow our users to tune important behaviors of the system.
<!--col-->
至于可能提供给应用开发者的分布式数据存储模型，我们认为分布式 B 树或分布式哈希表所提供的 key-value 对模型限制过强。key-value 对是有用的构件，但它不应该是提供给开发者的唯一构件。我们选择的数据模型比简单的 key-value 对更丰富，支持稀疏的半结构化数据。尽管如此，它仍然足够简单，适合用一种非常高效的平面文件表示；也足够透明（通过局部性组），让我们的用户能够调优系统的重要行为。
{{% /bilingual %}}

{{% bilingual %}}
Several database vendors have developed parallel databases that can store large volumes of data. Oracle's Real Application Cluster database [27] uses shared disks to store data (Bigtable uses GFS) and a distributed lock manager (Bigtable uses Chubby). IBM's DB2 Parallel Edition [4] is based on a shared-nothing [33] architecture similar to Bigtable. Each DB2 server is responsible for a subset of the rows in a table which it stores in a local relational database. Both products provide a complete relational model with transactions.
<!--col-->
若干数据库厂商开发了能存放海量数据的并行数据库。Oracle 的 Real Application Cluster 数据库 [27] 用共享磁盘存放数据（Bigtable 用 GFS），并使用分布式锁管理器（Bigtable 用 Chubby）。IBM 的 DB2 Parallel Edition [4] 基于与 Bigtable 类似的 shared-nothing [33] 架构。每台 DB2 服务器负责表中一个行子集，并把它存放在本地关系数据库中。这两个产品都提供含事务的完整关系模型。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable locality groups realize similar compression and disk read performance benefits observed for other systems that organize data on disk using column-based rather than row-based storage, including C-Store [1, 34] and commercial products such as Sybase IQ [15, 36], SenSage [31], KDB+ [22], and the ColumnBM storage layer in MonetDB/X100 [38]. Another system that does vertical and horizontal data partioning into flat files and achieves good data compression ratios is AT&T's Daytona database [19]. Locality groups do not support CPU-cache-level optimizations, such as those described by Ailamaki [2].
<!--col-->
Bigtable 的局部性组实现了与其他那些在磁盘上按列而非按行组织数据的系统所观察到的类似压缩与磁盘读性能收益，这类系统包括 C-Store [1, 34] 以及 Sybase IQ [15, 36]、SenSage [31]、KDB+ [22] 和 MonetDB/X100 [38] 中的 ColumnBM 存储层等商业产品。另一个把数据做垂直和水平划分成平面文件、并取得良好压缩率的系统是 AT&T 的 Daytona 数据库 [19]。局部性组不支持 Ailamaki [2] 所描述的那种 CPU 缓存级优化。
{{% /bilingual %}}

{{% bilingual %}}
The manner in which Bigtable uses memtables and SSTables to store updates to tablets is analogous to the way that the Log-Structured Merge Tree [26] stores updates to index data. In both systems, sorted data is buffered in memory before being written to disk, and reads must merge data from memory and disk.
<!--col-->
Bigtable 使用 memtable 和 SSTable 来存放 tablet 更新的方式，与 Log-Structured Merge Tree [26] 存放索引数据更新的方式类似。两个系统中，有序数据都先在内存中缓冲，然后才写到磁盘，而且读必须合并来自内存和磁盘的数据。
{{% /bilingual %}}

{{% bilingual %}}
C-Store and Bigtable share many characteristics: both systems use a shared-nothing architecture and have two different data structures, one for recent writes, and one for storing long-lived data, with a mechanism for moving data from one form to the other. The systems differ significantly in their API: C-Store behaves like a relational database, whereas Bigtable provides a lower level read and write interface and is designed to support many thousands of such operations per second per server. C-Store is also a "read-optimized relational DBMS", whereas Bigtable provides good performance on both read-intensive and write-intensive applications.
<!--col-->
C-Store 与 Bigtable 有许多共同特征：两者都采用 shared-nothing 架构，都有两种不同的数据结构——一种用于近期写入，一种用于长期存活的数据——并配有把数据从一种形式搬到另一种的机制。两者在 API 上差别很大：C-Store 表现得像关系数据库，而 Bigtable 提供的是更低层的读写接口，设计目标是每台服务器每秒支持数千次这类操作。C-Store 还是一个「面向读优化的关系型 DBMS」，而 Bigtable 在读密集和写密集的应用上都提供良好性能。
{{% /bilingual %}}

{{% bilingual %}}
Bigtable's load balancer has to solve some of the same kinds of load and memory balancing problems faced by shared-nothing databases (e.g., [11, 35]). Our problem is somewhat simpler: (1) we do not consider the possibility of multiple copies of the same data, possibly in alternate forms due to views or indices; (2) we let the user tell us what data belongs in memory and what data should stay on disk, rather than trying to determine this dynamically; (3) we have no complex queries to execute or optimize.
<!--col-->
Bigtable 的负载均衡器必须解决 shared-nothing 数据库（例如 [11, 35]）所面临的某些同类负载与内存均衡问题。我们的问题要简单一些：（1）我们不考虑同一份数据存在多个副本的可能性，也不考虑因视图或索引而产生替代形式的副本；（2）我们让用户告诉我们哪些数据该放在内存、哪些该留在磁盘，而不是试图动态判定；（3）我们没有复杂查询需要执行或优化。
{{% /bilingual %}}

---

## 11 Conclusions · 结论

{{% bilingual %}}
We have described Bigtable, a distributed system for storing structured data at Google. Bigtable clusters have been in production use since April 2005, and we spent roughly seven person-years on design and implementation before that date. As of August 2006, more than sixty projects are using Bigtable. Our users like the performance and high availability provided by the Bigtable implementation, and that they can scale the capacity of their clusters by simply adding more machines to the system as their resource demands change over time.
<!--col-->
我们描述了 Bigtable——一套在 Google 用于存放结构化数据的分布式系统。Bigtable 集群自 2005 年 4 月起投入生产使用，在那之前我们在大致七个人年的设计实现上花掉了时间。截至 2006 年 8 月，已有六十多个项目在使用 Bigtable。我们的用户喜欢 Bigtable 实现所提供的性能与高可用，也喜欢他们可以随着资源需求的变化，通过简单地往系统里加机器来扩展集群容量。
{{% /bilingual %}}

{{% bilingual %}}
Given the unusual interface to Bigtable, an interesting question is how difficult it has been for our users to adapt to using it. New users are sometimes uncertain of how to best use the Bigtable interface, particularly if they are accustomed to using relational databases that support general-purpose transactions. Nevertheless, the fact that many Google products successfully use Bigtable demonstrates that our design works well in practice.
<!--col-->
考虑到 Bigtable 的接口并不寻常，一个有意思的问题是：我们的用户适应它有多难？新用户有时不确定怎样最好地使用 Bigtable 接口，尤其是当他们习惯了使用支持通用事务的关系数据库。尽管如此，许多 Google 产品成功使用 Bigtable 这一事实，说明我们的设计在实践中行之有效。
{{% /bilingual %}}

{{% bilingual %}}
We are in the process of implementing several additional Bigtable features, such as support for secondary indices and infrastructure for building cross-data-center replicated Bigtables with multiple master replicas. We have also begun deploying Bigtable as a service to product groups, so that individual groups do not need to maintain their own clusters. As our service clusters scale, we will need to deal with more resource-sharing issues within Bigtable itself [3, 5].
<!--col-->
我们正在实现若干额外的 Bigtable 特性，比如对二级索引的支持，以及用于构建带多个 master 副本的跨数据中心复制 Bigtable 的基础设施。我们也已开始把 Bigtable 作为一项服务提供给各产品团队，这样各个团队就不必自己维护集群。随着服务型集群规模扩大，我们将需要处理更多 Bigtable 内部的资源共享问题 [3, 5]。
{{% /bilingual %}}

{{% bilingual %}}
Finally, we have found that there are significant advantages to building our own storage solution at Google. We have gotten a substantial amount of flexibility from designing our own data model for Bigtable. In addition, our control over Bigtable's implementation, and the other Google infrastructure upon which Bigtable depends, means that we can remove bottlenecks and inefficiencies as they arise.
<!--col-->
最后我们发现，在 Google 自建存储方案有显著优势。为自己的 Bigtable 设计数据模型，让我们获得了相当大的灵活性。此外，我们对 Bigtable 实现以及 Bigtable 所依赖的其他 Google 基础设施的掌控，意味着我们可以在瓶颈和低效出现时就消除它们。
{{% /bilingual %}}

---

## Acknowledgements · 致谢

{{% bilingual %}}
We thank the anonymous reviewers, David Nagle, and our shepherd Brad Calder, for their feedback on this paper. The Bigtable system has benefited greatly from the feedback of our many users within Google. In addition, we thank the following people for their contributions to Bigtable: Dan Aguayo, Sameer Ajmani, Zhifeng Chen, Bill Coughran, Mike Epstein, Healfdene Goguen, Robert Griesemer, Jeremy Hylton, Josh Hyman, Alex Khesin, Joanna Kulik, Alberto Lerner, Sherry Listgarten, Mike Maloney, Eduardo Pinheiro, Kathy Polizzi, Frank Yellin, and Arthur Zwiegincew.
<!--col-->
我们感谢匿名审稿人、David Nagle 以及我们的 shepherd Brad Calder 对本文的反馈。Bigtable 系统从 Google 内部众多用户的反馈中获益良多。此外，我们感谢下列各位对 Bigtable 的贡献：Dan Aguayo、Sameer Ajmani、Zhifeng Chen、Bill Coughran、Mike Epstein、Healfdene Goguen、Robert Griesemer、Jeremy Hylton、Josh Hyman、Alex Khesin、Joanna Kulik、Alberto Lerner、Sherry Listgarten、Mike Maloney、Eduardo Pinheiro、Kathy Polizzi、Frank Yellin 和 Arthur Zwiegincew。
{{% /bilingual %}}

---

## References · 参考文献

{{% bilingual %}}
[1] Abadi, D. J., Madden, S. R., and Ferreira, M. C. Integrating compression and execution in column-oriented database systems. Proc. of SIGMOD (2006).
<!--col-->
[1] Abadi, D. J., Madden, S. R., Ferreira, M. C. Integrating compression and execution in column-oriented database systems. SIGMOD 论文集，2006 年。
{{% /bilingual %}}

{{% bilingual %}}
[2] Ailamaki, A., DeWitt, D. J., Hill, M. D., and Skounakis, M. Weaving relations for cache performance. In The VLDB Journal (2001), pp. 169-180.
<!--col-->
[2] Ailamaki, A., DeWitt, D. J., Hill, M. D., Skounakis, M. Weaving relations for cache performance. The VLDB Journal，2001 年，第 169–180 页。
{{% /bilingual %}}

{{% bilingual %}}
[3] Banga, G., Druschel, P., and Mogul, J. C. Resource containers: A new facility for resource management in server systems. In Proc. of the 3rd OSDI (Feb. 1999), pp. 45-58.
<!--col-->
[3] Banga, G., Druschel, P., Mogul, J. C. Resource containers: A new facility for resource management in server systems. 第 3 届 OSDI 论文集，1999 年 2 月，第 45–58 页。
{{% /bilingual %}}

{{% bilingual %}}
[4] Baru, C. K., Fecteau, G., Goyal, A., Hsiao, H., Jhingran, A., Padmanabhan, S., Copeland, G. P., and Wilson, W. G. DB2 parallel edition. IBM Systems Journal 34, 2 (1995), 292-322.
<!--col-->
[4] Baru, C. K., Fecteau, G., Goyal, A., Hsiao, H., Jhingran, A., Padmanabhan, S., Copeland, G. P., Wilson, W. G. DB2 parallel edition. IBM Systems Journal 34, 2 (1995), 292–322。
{{% /bilingual %}}

{{% bilingual %}}
[5] Bavier, A., Bowman, M., Chun, B., Culler, D., Karlin, S., Peterson, L., Roscoe, T., Spalink, T., and Wawrzoniak, M. Operating system support for planetary-scale network services. In Proc. of the 1st NSDI (Mar. 2004), pp. 253-266.
<!--col-->
[5] Bavier, A., Bowman, M., Chun, B., Culler, D., Karlin, S., Peterson, L., Roscoe, T., Spalink, T., Wawrzoniak, M. Operating system support for planetary-scale network services. 第 1 届 NSDI 论文集，2004 年 3 月，第 253–266 页。
{{% /bilingual %}}

{{% bilingual %}}
[6] Bentley, J. L., and McIlroy, M. D. Data compression using long common strings. In Data Compression Conference (1999), pp. 287-295.
<!--col-->
[6] Bentley, J. L., McIlroy, M. D. Data compression using long common strings. 数据压缩会议（DCC），1999 年，第 287–295 页。
{{% /bilingual %}}

{{% bilingual %}}
[7] Bloom, B. H. Space/time trade-offs in hash coding with allowable errors. CACM 13, 7 (1970), 422-426.
<!--col-->
[7] Bloom, B. H. Space/time trade-offs in hash coding with allowable errors. CACM 13, 7 (1970), 422–426。
{{% /bilingual %}}

{{% bilingual %}}
[8] Burrows, M. The Chubby lock service for loosely-coupled distributed systems. In Proc. of the 7th OSDI (Nov. 2006).
<!--col-->
[8] Burrows, M. The Chubby lock service for loosely-coupled distributed systems. 第 7 届 OSDI 论文集，2006 年 11 月。
{{% /bilingual %}}

{{% bilingual %}}
[9] Chandra, T., Griesemer, R., and Redstone, J. Paxos made live -- An engineering perspective. In Proc. of PODC (2007).
<!--col-->
[9] Chandra, T., Griesemer, R., Redstone, J. Paxos made live -- An engineering perspective. PODC 论文集，2007 年。
{{% /bilingual %}}

{{% bilingual %}}
[10] Comer, D. Ubiquitous B-tree. Computing Surveys 11, 2 (June 1979), 121-137.
<!--col-->
[10] Comer, D. Ubiquitous B-tree. Computing Surveys 11, 2 (1979 年 6 月), 121–137。
{{% /bilingual %}}

{{% bilingual %}}
[11] Copeland, G. P., Alexander, W., Boughter, E. E., and Keller, T. W. Data placement in Bubba. In Proc. of SIGMOD (1988), pp. 99-108.
<!--col-->
[11] Copeland, G. P., Alexander, W., Boughter, E. E., Keller, T. W. Data placement in Bubba. SIGMOD 论文集，1988 年，第 99–108 页。
{{% /bilingual %}}

{{% bilingual %}}
[12] Dean, J., and Ghemawat, S. MapReduce: Simplified data processing on large clusters. In Proc. of the 6th OSDI (Dec. 2004), pp. 137-150.
<!--col-->
[12] Dean, J., Ghemawat, S. MapReduce: Simplified data processing on large clusters. 第 6 届 OSDI 论文集，2004 年 12 月，第 137–150 页。
{{% /bilingual %}}

{{% bilingual %}}
[13] DeWitt, D., Katz, R., Olken, F., Shapiro, L., Stonebraker, M., and Wood, D. Implementation techniques for main memory database systems. In Proc. of SIGMOD (June 1984), pp. 1-8.
<!--col-->
[13] DeWitt, D., Katz, R., Olken, F., Shapiro, L., Stonebraker, M., Wood, D. Implementation techniques for main memory database systems. SIGMOD 论文集，1984 年 6 月，第 1–8 页。
{{% /bilingual %}}

{{% bilingual %}}
[14] DeWitt, D. J., and Gray, J. Parallel database systems: The future of high performance database systems. CACM 35, 6 (June 1992), 85-98.
<!--col-->
[14] DeWitt, D. J., Gray, J. Parallel database systems: The future of high performance database systems. CACM 35, 6 (1992 年 6 月), 85–98。
{{% /bilingual %}}

{{% bilingual %}}
[15] French, C. D. One size fits all database architectures do not work for DSS. In Proc. of SIGMOD (May 1995), pp. 449-450.
<!--col-->
[15] French, C. D. One size fits all database architectures do not work for DSS. SIGMOD 论文集，1995 年 5 月，第 449–450 页。
{{% /bilingual %}}

{{% bilingual %}}
[16] Gawlick, D., and Kinkade, D. Varieties of concurrency control in IMS/VS fast path. Database Engineering Bulletin 8, 2 (1985), 3-10.
<!--col-->
[16] Gawlick, D., Kinkade, D. Varieties of concurrency control in IMS/VS fast path. Database Engineering Bulletin 8, 2 (1985), 3–10。
{{% /bilingual %}}

{{% bilingual %}}
[17] Ghemawat, S., Gobioff, H., and Leung, S.-T. The Google file system. In Proc. of the 19th ACM SOSP (Dec. 2003), pp. 29-43.
<!--col-->
[17] Ghemawat, S., Gobioff, H., Leung, S.-T. The Google file system. 第 19 届 ACM SOSP 论文集，2003 年 12 月，第 29–43 页。
{{% /bilingual %}}

{{% bilingual %}}
[18] Gray, J. Notes on database operating systems. In Operating Systems -- An Advanced Course, vol. 60 of Lecture Notes in Computer Science. Springer-Verlag, 1978.
<!--col-->
[18] Gray, J. Notes on database operating systems. 载 Operating Systems -- An Advanced Course, Lecture Notes in Computer Science 第 60 卷. Springer-Verlag, 1978 年。
{{% /bilingual %}}

{{% bilingual %}}
[19] Greer, R. Daytona and the fourth-generation language Cymbal. In Proc. of SIGMOD (1999), pp. 525-526.
<!--col-->
[19] Greer, R. Daytona and the fourth-generation language Cymbal. SIGMOD 论文集，1999 年，第 525–526 页。
{{% /bilingual %}}

{{% bilingual %}}
[20] Hagmann, R. Reimplementing the Cedar file system using logging and group commit. In Proc. of the 11th SOSP (Dec. 1987), pp. 155-162.
<!--col-->
[20] Hagmann, R. Reimplementing the Cedar file system using logging and group commit. 第 11 届 SOSP 论文集，1987 年 12 月，第 155–162 页。
{{% /bilingual %}}

{{% bilingual %}}
[21] Hartman, J. H., and Ousterhout, J. K. The Zebra striped network file system. In Proc. of the 14th SOSP (Asheville, NC, 1993), pp. 29-43.
<!--col-->
[21] Hartman, J. H., Ousterhout, J. K. The Zebra striped network file system. 第 14 届 SOSP 论文集，北卡罗来纳州阿什维尔，1993 年，第 29–43 页。
{{% /bilingual %}}

{{% bilingual %}}
[22] KX.com. kx.com/products/database.php. Product page.
<!--col-->
[22] KX.com. kx.com/products/database.php. 产品页。
{{% /bilingual %}}

{{% bilingual %}}
[23] Lamport, L. The part-time parliament. ACM TOCS 16, 2 (1998), 133-169.
<!--col-->
[23] Lamport, L. The part-time parliament. ACM TOCS 16, 2 (1998), 133–169。
{{% /bilingual %}}

{{% bilingual %}}
[24] MacCormick, J., Murphy, N., Najork, M., Thekkath, C. A., and Zhou, L. Boxwood: Abstractions as the foundation for storage infrastructure. In Proc. of the 6th OSDI (Dec. 2004), pp. 105-120.
<!--col-->
[24] MacCormick, J., Murphy, N., Najork, M., Thekkath, C. A., Zhou, L. Boxwood: Abstractions as the foundation for storage infrastructure. 第 6 届 OSDI 论文集，2004 年 12 月，第 105–120 页。
{{% /bilingual %}}

{{% bilingual %}}
[25] McCarthy, J. Recursive functions of symbolic expressions and their computation by machine. CACM 3, 4 (Apr. 1960), 184-195.
<!--col-->
[25] McCarthy, J. Recursive functions of symbolic expressions and their computation by machine. CACM 3, 4 (1960 年 4 月), 184–195。
{{% /bilingual %}}

{{% bilingual %}}
[26] O'Neil, P., Cheng, E., Gawlick, D., and O'Neil, E. The log-structured merge-tree (LSM-tree). Acta Inf. 33, 4 (1996), 351-385.
<!--col-->
[26] O'Neil, P., Cheng, E., Gawlick, D., O'Neil, E. The log-structured merge-tree (LSM-tree). Acta Inf. 33, 4 (1996), 351–385。
{{% /bilingual %}}

{{% bilingual %}}
[27] Oracle.com. www.oracle.com/technology/products/database/clustering/index.html. Product page.
<!--col-->
[27] Oracle.com. www.oracle.com/technology/products/database/clustering/index.html. 产品页。
{{% /bilingual %}}

{{% bilingual %}}
[28] Pike, R., Dorward, S., Griesemer, R., and Quinlan, S. Interpreting the data: Parallel analysis with Sawzall. Scientific Programming Journal 13, 4 (2005), 227-298.
<!--col-->
[28] Pike, R., Dorward, S., Griesemer, R., Quinlan, S. Interpreting the data: Parallel analysis with Sawzall. Scientific Programming Journal 13, 4 (2005), 227–298。
{{% /bilingual %}}

{{% bilingual %}}
[29] Ratnasamy, S., Francis, P., Handley, M., Karp, R., and Shenker, S. A scalable content-addressable network. In Proc. of SIGCOMM (Aug. 2001), pp. 161-172.
<!--col-->
[29] Ratnasamy, S., Francis, P., Handley, M., Karp, R., Shenker, S. A scalable content-addressable network. SIGCOMM 论文集，2001 年 8 月，第 161–172 页。
{{% /bilingual %}}

{{% bilingual %}}
[30] Rowstron, A., and Druschel, P. Pastry: Scalable, distributed object location and routing for large-scale peer-to-peer systems. In Proc. of Middleware 2001 (Nov. 2001), pp. 329-350.
<!--col-->
[30] Rowstron, A., Druschel, P. Pastry: Scalable, distributed object location and routing for large-scale peer-to-peer systems. Middleware 2001 论文集，2001 年 11 月，第 329–350 页。
{{% /bilingual %}}

{{% bilingual %}}
[31] SenSage.com. sensage.com/products-sensage.htm. Product page.
<!--col-->
[31] SenSage.com. sensage.com/products-sensage.htm. 产品页。
{{% /bilingual %}}

{{% bilingual %}}
[32] Stoica, I., Morris, R., Karger, D., Kaashoek, M. F., and Balakrishnan, H. Chord: A scalable peer-to-peer lookup service for Internet applications. In Proc. of SIGCOMM (Aug. 2001), pp. 149-160.
<!--col-->
[32] Stoica, I., Morris, R., Karger, D., Kaashoek, M. F., Balakrishnan, H. Chord: A scalable peer-to-peer lookup service for Internet applications. SIGCOMM 论文集，2001 年 8 月，第 149–160 页。
{{% /bilingual %}}

{{% bilingual %}}
[33] Stonebraker, M. The case for shared nothing. Database Engineering Bulletin 9, 1 (Mar. 1986), 4-9.
<!--col-->
[33] Stonebraker, M. The case for shared nothing. Database Engineering Bulletin 9, 1 (1986 年 3 月), 4–9。
{{% /bilingual %}}

{{% bilingual %}}
[34] Stonebraker, M., Abadi, D. J., Batkin, A., Chen, X., Cherniack, M., Ferreira, M., Lau, E., Lin, A., Madden, S., O'Neil, E., O'Neil, P., Rasin, A., Tran, N., and Zdonik, S. C-Store: A column-oriented DBMS. In Proc. of VLDB (Aug. 2005), pp. 553-564.
<!--col-->
[34] Stonebraker, M., Abadi, D. J., Batkin, A., Chen, X., Cherniack, M., Ferreira, M., Lau, E., Lin, A., Madden, S., O'Neil, E., O'Neil, P., Rasin, A., Tran, N., Zdonik, S. C-Store: A column-oriented DBMS. VLDB 论文集，2005 年 8 月，第 553–564 页。
{{% /bilingual %}}

{{% bilingual %}}
[35] Stonebraker, M., Aoki, P. M., Devine, R., Litwin, W., and Olson, M. A. Mariposa: A new architecture for distributed data. In Proc. of the Tenth ICDE (1994), IEEE Computer Society, pp. 54-65.
<!--col-->
[35] Stonebraker, M., Aoki, P. M., Devine, R., Litwin, W., Olson, M. A. Mariposa: A new architecture for distributed data. 第 10 届 ICDE 论文集，1994 年，IEEE Computer Society，第 54–65 页。
{{% /bilingual %}}

{{% bilingual %}}
[36] Sybase.com. www.sybase.com/products/databaseservers/sybaseiq. Product page.
<!--col-->
[36] Sybase.com. www.sybase.com/products/databaseservers/sybaseiq. 产品页。
{{% /bilingual %}}

{{% bilingual %}}
[37] Zhao, B. Y., Kubiatowicz, J., and Joseph, A. D. Tapestry: An infrastructure for fault-tolerant wide-area location and routing. Tech. Rep. UCB/CSD-01-1141, CS Division, UC Berkeley, Apr. 2001.
<!--col-->
[37] Zhao, B. Y., Kubiatowicz, J., Joseph, A. D. Tapestry: An infrastructure for fault-tolerant wide-area location and routing. 技术报告 UCB/CSD-01-1141，加州大学伯克利分校计算机科学系，2001 年 4 月。
{{% /bilingual %}}

{{% bilingual %}}
[38] Zukowski, M., Boncz, P. A., Nes, N., and Heman, S. MonetDB/X100 -- A DBMS in the CPU cache. IEEE Data Eng. Bull. 28, 2 (2005), 17-22.
<!--col-->
[38] Zukowski, M., Boncz, P. A., Nes, N., Heman, S. MonetDB/X100 -- A DBMS in the CPU cache. IEEE Data Eng. Bull. 28, 2 (2005), 17–22。
{{% /bilingual %}}
