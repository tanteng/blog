---
title: "Laravel 消息队列注意事项"
date: 2017-12-11T10:08:13+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "使用 Laravel 消息队列要注意的问题"
---

使用 Laravel 的消息队列处理异步任务，Redis 作为队列数据库，Supervisor 监控脚本异常中断并自动重启，这是 Laravel 处理队列任务的标准流程，但是实际中可能还会出现各种各样的问题，为了保证系统可靠性，还要注意几个问题。

<!--more-->

### 一、执行失败重试次数设置

一定要设置任务执行失败重试次数，避免无限失败重试，超过重试次数 Laravel 会默认写到失败任务表中。

```bash
php artisan queue:work redis --tries=3
```

需要先执行以下命令创建数据表：

```bash
php artisan queue:failed-table
php artisan migrate
```

### 二、程序异常的处理

有时候程序执行过程会发生异常，比如依赖其他接口，请求 HTTP 接口超时等等，如果不捕捉异常，那么当前这个队列就会中断不能继续运行下去。

需要使用 try-catch 捕获异常，确保任务不会中断。

### 三、修改代码记得重启 Supervisor

修改了处理队列的程序，记得要重启 Supervisor，否则脚本不会生效。

### Laravel 往 Redis 写队列的数据结构

队列用 list 类型存储，包含失败重试次数、队列标识、处理队列的类、以及队列的数据等等。
