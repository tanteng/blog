---
title: "Laravel 队列任务重复执行问题"
date: 2017-12-24T07:49:30+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "Laravel 队列任务重复执行问题分析"
---

在使用 Laravel Redis 队列时，发现一个任务被多次执行，这是为什么呢？

原因：Laravel 中如果一个队列任务执行时间大于 60 秒，就会被认为执行失败并重新加入队列，这样就会导致重复执行。

<!--more-->

### 问题分析

Laravel 队列有一个默认的过期时间设置：

```php
// vendor/laravel/framework/src/Illuminate/Queue/RedisQueue.php
protected $expire = 60;
```

Laravel 认为一个队列 60 秒应该执行完成。

### 处理流程

取队列时有几步操作：
1. 把因执行失败的队列从 delayed 集合重新 rpush 到当前队列
2. 把因执行超时的队列从 reserved 集合重新 rpush 到当前队列
3. 从队列中取任务开始执行

### Lua 脚本

```lua
-- 从 reserved 集合找出超时的任务
local val = redis.call('zrangebyscore', KEYS[1], '-inf', ARGV[1])
-- 重新 rpush 到队列
redis.call('rpush', KEYS[2], unpack(val, i, math.min(i+99, #val)))
```

### 解决方案

1. 将任务拆分，减少单个任务执行时间
2. 增加 expire 时间
3. 使用 Laravel 5.4+ 版本（已修复此问题）
