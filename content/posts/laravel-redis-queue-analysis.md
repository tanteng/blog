---
title: "Laravel Redis 队列原理分析"
date: 2017-12-23T11:26:19+08:00
draft: false
tags: ['laravel', 'redis']
categories: ['tech']
description: "Laravel Redis 队列原理分析"
---

使用 Supervisor 开启多个进程处理队列任务时，是否会导致重复消费同一个任务？本文从源码角度分析 Laravel Redis 队列的工作原理。

<!--more-->

### 多进程会重复消费吗？

**结论：不会。**

即使开启 8 个进程同时处理队列，也不会出现重复消费同一个任务的情况。

### Laravel 队列实现原理

#### 入队列

使用 Redis 的 list 类型，rpush 命令添加任务：

```php
public function pushRaw($payload, $queue = null, array $options = [])
{
    $this->getConnection()->rpush($this->getQueue($queue), $payload);
}
```

#### 出队列

使用 Lua 脚本保证原子性：

```lua
local job = redis.call('lpop', KEYS[1])
local reserved = false
if (job ~= false) then
    reserved = cjson.decode(job)
    reserved['attempts'] = reserved['attempts'] + 1
    reserved = cjson.encode(reserved)
    -- 放入 reserved 有序集合，设置过期时间
    redis.call('zadd', KEYS[2], ARGV[1], reserved)
end
return {job, reserved}
```

关键点：
1. lpop 从队列头部取出任务
2. 同时将任务放入 reserved 有序集合（用于超时检测）
3. 任务执行完成后从 reserved 中删除

### 为什么不会重复？

Lua 脚本保证了 lpop 是原子操作，只有一个进程能取出同一个任务。

### 可能导致重复的情况

如果任务执行时间超过 60 秒（默认超时时间），Laravel 会认为任务失败并重新入队，导致重复执行。
