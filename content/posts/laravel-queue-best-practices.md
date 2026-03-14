---
title: "Laravel 队列实践指南"
date: 2017-12-11T10:08:13+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "Laravel 队列实践指南"
---

使用 Laravel 消息队列处理异步任务，Redis 作为队列驱动，Supervisor 监控进程异常中断并自动重启，这是 Laravel 处理队列任务的标准配置。但在生产环境中，为了保证系统可靠性，还需要注意以下几点。

<!--more-->

### 1. 设置失败重试次数

一定要设置任务执行失败重试次数，避免无限失败重试。

```bash
php artisan queue:work redis --tries=3
```

创建失败任务表：

```bash
php artisan queue:failed-table
php artisan migrate
```

### 2. 捕获异常

程序执行过程中可能发生各种异常，如依赖接口超时等。如果不捕获异常，当前队列会中断无法继续执行。

```php
try {
    $response = $client->request('POST', $url, $options);
} catch (RequestException $e) {
    Log::warning('RequestException: ' . $e->getMessage());
} catch (Exception $e) {
    Log::emergency('Exception: ' . $e->getMessage());
}
```

### 3. 修改代码后重启 Supervisor

修改队列处理逻辑后，必须重启 Supervisor 才能生效：

```bash
sudo supervisorctl restart laravel-worker:*
```

### 4. 生产环境推荐配置

Laravel 11+ 推荐使用以下配置：

```bash
php artisan queue:work --max-jobs=500 --max-time=3600
```

- `--max-jobs=500`：每处理 500 个任务后重启 worker，释放内存
- `--max-time=3600`：每运行 1 小时后重启，防止内存泄漏

### 5. 监控队列状态

使用 Laravel Horizon 可以直观监控队列的 jobs 数量、失败任务、重试情况等。
