---
title: "Laravel Redis 多进程队列重复问题"
date: 2017-12-23T11:26:19+08:00
draft: false
tags: ['laravel', 'redis']
categories: ['tech']
description: "Laravel Redis 多个进程同时取队列问题"
---

开启多个进程处理队列会重复读取 Redis 中队列吗？是否因此导致重复执行任务？

使用 Supervisor 监听 Laravel 队列任务，numprocs = 8 代表开启 8 个进程来执行队列命令。

<!--more-->

### Laravel 多进程读取队列是否会重复？

通过测试发现：从 Laravel 的处理方式和打印的日志结果看，即使多个进程读取同一个队列，也不会读取到同样的数据。

### Laravel 队列处理原理

Laravel 使用 Redis 的 list 作为队列的数据结构，入队列用 rpush 命令：

```php
public function pushRaw($payload, $queue = null, array $options = [])
{
    $this->getConnection()->rpush($this->getQueue($queue), $payload);
}
```

取队列用 lua 脚本：

```lua
local job = redis.call('lpop', KEYS[1])
-- 取队列的同时放入 reserved 有序集合
```

**结论：Laravel 的队列机制确保了多进程不会重复消费同一个任务。**

### 可能导致重复的原因

如果一个任务执行时间大于 60 秒，Laravel 会认为执行失败并重新加入队列，导致重复执行。
