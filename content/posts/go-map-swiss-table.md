---
title: "Go map 使用 Swiss Table 重新实现，性能最高提升近50%"
date: 2024-11-15
draft: false
tags: ["go", "golang", "performance", "algorithm"]
categories: ["tech"]
description: "Go 1.24 将引入 Swiss Table 作为 map 的底层实现，相比原实现查询、插入、删除操作性能提升 20%~50%，迭代性能提升 10%，内存使用减少 0%~25%。"
---

在 2024年11月5日的 Go compiler and runtime meeting notes 中，我们注意到一个重要信息：来自字节的一位工程师在两年多前提出的「使用 Swiss table 重新实现 Go map」的建议即将落地，该 issue 已经被纳入 **Go 1.24 里程碑**。

## 什么是 Swiss Table

Swiss Table 是 Google 工程师于 2017 年开发的一种高效哈希表实现，旨在优化内存使用和提升性能。目前已被应用于多种编程语言：

- C++ Abseil 库的 `flat_hash_map`
- Rust 标准库 HashMap 的默认实现

Google 工程师 Matt Kulukundis 在 2017 年 CppCon 大会上详细介绍了他们在 Swiss Table 上的工作：

<https://www.youtube.com/watch?v=ncHmEUmJZf4>

## 字节跳动的推动力

据 issue 描述，Go map 的 CPU 消耗约占服务总体开销的 **4%**。其中，map 的插入（mapassign）和访问（mapaccess）操作的 CPU 消耗几乎是 1:1。

以字节跳动这样大厂的体量，减少 1% 也意味着真金白银的大幅节省。字节工程师初版实现的基准测试结果显示，与原实现相比：

- 查询、插入、删除操作：**提升 20%~50%**
- 迭代性能：**提升 10%**
- 内存使用：**减少 0%~25%**

## 当前 Go map 的实现

Go map 采用**链式哈希**（拉链法）实现：

- 每个 bucket 可存储 **8 个键值对**
- 超过容量时创建 **overflow bucket** 存储多余数据
- 通过 **load factor = 6.5** 判定是否扩容
- 采用**增量扩容**，"蚂蚁搬家"式搬移元素

主要问题：
1. 扩容后**不再缩容**，给内存带来压力
2. 性能和延迟敏感场景下仍不够快

## Swiss Table 的工作原理

Swiss Table 使用的不是拉链法，而是**开放寻址**。其核心创新是将桶分为多个 **Group**，每个 Group 有 **16 个 Slot**。

### Group Probing

Group Probing 允许通过 **SIMD 指令**并行探测 16 个 Slot，快速定位目标位置。这种方法就像"布隆过滤器"一样，能快速排除不可能的匹配项，减少不必要的内存访问。

16 个条目的分组大小是基于 **SSE2 寄存器长度（128bit, 16bytes）** 和现代 CPU 的**缓存行大小（64字节）** 优化的，保证了一个 Group 的控制字节能被单次 SIMD 指令处理。

### 元数据设计

Group 的 metadata 是一个 16 字节数组，每个字节对应一个 Slot 的控制字节：

- **最高位为 1，其余全零**：空闲状态（Empty）
- **最高位为 0，后 7 位为哈希指纹**：已使用状态
- **最高位为 1，其他位为 1111110**：已删除状态

## Go tip 版本的实现

Go 团队在 Swiss Table 之上做了**局部改进**，引入了多 table 设计以支持**渐进式扩容**：

1. **多 Table 设计**：一个 map 实际上是多个 Swiss Table，每个 table 拥有自己的 load factor，可以独立扩容
2. **Extendible Hashing**：通过动态调整哈希位数实现渐进式扩容，每次只处理一部分数据
3. **Small Map 优化**：少量元素（<=8）时直接使用单个 Group，避免性能回退

## Benchmark 性能对比

基于 Go 1.23.0 和 gotip (devel go1.24) 的基准测试结果：

| 测试项 | 原实现 | Swiss Table | 提升 |
|--------|--------|-------------|------|
| MapAssignReuse/Int64/256 | 8.737µs | 4.716µs | **46%** |
| MapAssignPreAllocate/Int64/256 | 10.412µs | 6.055µs | **42%** |
| MapAccessHit/Int64/8192 | 25.99ns | 14.93ns | **43%** |
| MapIter/Int/256 | 4.328µs | 3.748µs | **13%** |

几何平均提升：**15.11%**

## 小结

经过两年多的实验与评估，Go 团队决定将 Swiss Table 作为 Go map 的底层实现，预计在 **Go 1.24** 中正式落地。新的实现不仅承继了原有的语义约束，还通过引入多表和渐进式扩容的设计，进一步优化了扩容过程的性能。

对于 Go 开发者来说，这是一个值得关注的重要改进——特别是在处理大规模数据或对性能敏感的场景中。

## 参考资料

- [runtime: use SwissTable - golang/go#54766](https://github.com/golang/go/issues/54766)
- [Swiss Tables Design Notes - Abseil](https://abseil.io/about/design/swisstables)
- [Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step](https://www.youtube.com/watch?v=ncHmEUmJZf4)
- [Go 1.24 Milestone](https://github.com/golang/go/milestone/322)
