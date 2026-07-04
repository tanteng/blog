---
title: "PHP 异步的革命：True Async 要来了"
date: 2026-07-05T10:00:00+08:00
draft: false
tags: ["php", "async", "coroutine"]
categories: ["tech"]
description: "PHP True Async 项目为 PHP 带来了真正的 async/await 能力，让现有 PHP 代码无需适配即可在协程中非阻塞运行。这可能是 PHP 自 FrankenPHP 以来最大的变革。"
---

PHP 异步有个原罪。

几十年来，PHP 开发者习惯了「请求-响应」的同步模型。想做异步？要么用 callback 堆成回调地狱，要么引入 Swoole、Amp、ReactPHP 这套完全不同的运行时 —— 然后发现所有主流库都不支持，你得自己写适配层。

<!--more-->

这种割裂感，让 PHP 异步始终停留在小众玩家的手里。

**True Async** 想要改变这一切。

## True Async 是什么

True Async 是一个 PHP 扩展，目前处于 RFC 草案阶段，计划于 2026 年 11 月随 PHP 8.6 发布。它的核心理念用一句话就能说清：**让普通 PHP 代码直接跑在协程里，自动变成非阻塞**。

这意味着什么？

你熟悉的 `file_get_contents()`、Laravel 的数据库查询、`sleep()`、甚至 cURL —— 这些现有的同步 API，不需要任何修改，直接在协程里就能实现并发调用。

对比一下就知道了。

### 传统异步框架的写法

用 Amp 或 ReactPHP，你需要这样写：

```php
use React\Http\HttpClient;
use React\EventLoop\Loop;

$loop = Loop::getEventLoop();
$client = $loop->makeResolvesClient();

$client->get('https://api.example.com/users')
    ->then(function ($response) {
        echo $response->getBody();
    });

$loop->run();
```

注意这里的问题：**所有 HTTP 库都得是「支持 ReactPHP 的」异步版本**。你现有的 `guzzle/guzzle` 库、WordPress 的 `wp_remote_get()`，通通跑不了。

### True Async 的写法

```php
// 普通的 PHP 代码，无需任何修改
$promise = async\spawn(function() {
    $users = file_get_contents('https://api.example.com/users');
    $posts = file_get_contents('https://api.example.com/posts');
    return ['users' => $users, 'posts' => $posts];
});

$result = async\await($promise);
```

**这才是真正的游戏改变者**。你不需要学习一套新的 API生态，不需要把现有代码重写一遍。协程？自动帮你调度。

## API 设计一览

True Async 的 API 简洁但功能完整：

### 核心原语

**`spawn`**：派生一个异步任务

```php
$task = async\spawn(function() {
    // 这里是协程体
    $result = file_get_contents('https://example.com/api');
    return json_decode($result, true);
});
```

**`spawn_with`**：派生时传递共享上下文

```php
$ctx = new Context(['request_id' => 'req-123']);
$task = async\spawn_with($ctx, function($ctx) {
    // 通过 $ctx 访问共享数据
    return process($ctx->request_id);
});
```

### 等待机制

```php
// 等待单个任务
$result = async\await($task);

// 等待全部任务（类似 Promise.all）
$results = async\await_all($task1, $task2, $task3);

// 任意一个失败立即终止（类似 Promise.race）
$result = async\await_any($task1, $task2, $task3);
```

这种设计借鉴了 JavaScript 的 `async/await` 语义，但底层实现更优雅 —— 它基于 Fiber（纤程），由 PHP 核心提供支持。

## 最大的壁垒：兼容性

你可能还记得之前关于 WordPress 能否跑在协程里的讨论。WordPress 满地都是全局变量 `$_GET`、`$_POST`、`$wpdb` —— 传统的协程模型里，这些状态会在所有协程间泄漏。

True Async 的解决方案是 **Scope Context**：协程内的超全局变量（`$_GET`、`$_SESSION` 等）是独立的，不会泄漏到其他协程。每个协程都有自己的沙箱。

这也意味着 Laravel、Symfony 这类框架，理论上可以**零修改或极少修改**就跑在 True Async 上。现有的单例、静态变量会被隔离到各自的作用域。

## 时间线与展望

2026 年 11 月会是 PHP 历史上关键的一个月。届时将同时迎来：

- **PHP 8.6** 发布
- **Swoole 团队的 Type PHP** 改动
- **True Async** 正式提交 RFC 并发布稳定版扩展

作者 Nuno Maduro 认为，这是自 FrankenPHP 以来 PHP 生态中发生的最大变革。如果 RFC 通过，async/await 可能成为 PHP 核心的一部分。

## 如何尝鲜

目前 True Async 扩展已经可以在 PHP 8.6 上本地测试。项目官网 [true-sync.github.io](https://true-sync.github.io) 提供了 IDE 辅助工具，可以在 PhpStorm 中获得代码补全。

对于 PHP 开发者来说，这是一次值得持续关注的技术演进。它解决的不是「能不能写异步」的问题，而是「能不能用熟悉的代码写异步」的问题。

---

*参考视频：[True Async/Await Is Coming to PHP?](https://www.youtube.com/watch?v=_Q_Ezm6bbF0)*
