---
title: Laravel 中为何不使用 blpop 取队列？
date: 2018-07-31T17:18:21+00:00
url: /2018/07/laravel-queue-blpop/
categories:
 - Laravel
 - Redis
tags:
 - Laravel
 - Redis
 - 队列

---

Redis 的 list 数据结构常用来做消息队列，通常使用的命令有 lpop/rpop ，还有带阻塞版的 blpop/brpop 等。blpop 的优点是避免一直轮询占用资源，而且支持多个列表作为参数并按照顺序弹出数据，如 blpop high low 30，可以更方便实现队列的优先级。

<!--more-->

Laravel 5.3 消息队列是用的 lpop 取消息，为什么不用阻塞版的 blpop 呢？

### 安全队列和不安全队列

什么是不安全的队列？比如客户端 lpop 从 list 中取出来的 job 还没处理完进程挂掉了或者遇到了异常，由于此时服务器上已经没有副本了，这个 job 就丢失了。这种队列就是不安全的。

Laravel 正是为了保证消息队列的可靠，进程挂掉了或者处理失败还可以重试等，做了比较完善的机制，如取队列的同时把队列放入另一个集合中"暂存"起来。使用 lpop 取出队列，同时 zadd 到另一个集合，使用 redis lua 脚本来保证原子性。

### 为什么不用 blpop？

blpop 是阻塞版的 lpop，如果队列没有数据过来，那么在超时时间内就会一直阻塞，直到 rpush 数据到队列。

这个问题作者在 github 上有回答。Redis lua 脚本实际上就是事务，为了保证原子性不可能一直阻塞，它跟 MULTI/EXEC 包裹起来的 blpop 一样不进行任何阻塞操作，因此这里用 blpop 没有意义。

Redis 官方文档也有说明：

> **在MULTI/EXEC事务中的BLPOP**
> 
> BLPOP 可以用于流水线(pipline)，但把它用在 MULTI / EXEC 块当中没有意义。因为这要求整个服务器被阻塞以保证块执行时的原子性，该行为阻止了其他客户端执行 LPUSH 或 RPUSH 命令。
> 
> 因此，一个被包裹在 MULTI / EXEC 块内的 BLPOP 命令，行为表现得就像 LPOP 一样，对空列表返回 nil ，对非空列表弹出列表元素，不进行任何阻塞操作。

结论就是：通过 lua 脚本操作 blpop 和 zadd 没有意义，因为没用到阻塞的特性，或者无法保证原子性。

此外，Redis 操作 list 还有一个命令 RPOPLPUSH/BRPOPLPUSH 可以用来维护安全队列。

### 参考资料

- [Laravel GitHub Issue #22939](https://github.com/laravel/framework/issues/22939)
- [Redis BLPOP 官方文档](https://redis.io/docs/latest/commands/blpop/)
- [JavaGuide - Redis常见面试题](https://javaguide.cn/database/redis/redis-questions-01.html)
