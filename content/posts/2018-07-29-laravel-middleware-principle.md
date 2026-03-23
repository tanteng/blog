---
title: Laravel 中间件原理
date: 2018-07-29T16:25:08+00:00
url: /2018/07/laravel-middleware-principle/
categories: ['tech']
tags:
 - laravel
 - principle

---

Laravel 的中间件机制提供了一种管道的方式，每个 HTTP 请求经过一个又一个中间件进行过滤，Laravel 内置了很多中间件，比如 CSRF 机制，身份认证，Cookie 加密，设置 Cookie 等等。

<!--more-->

本文就来探究 Laravel 中间件的实现原理，看 Laravel 如何把 PHP 的 array_reduce 函数和闭包用到了极致。

需要先了解 Laravel 中间件的用法，如何定义一个中间件，还有前置中间件，后置中间件的概念。

### 开始

为了弄懂 Laravel 中间件原理，可以构造一个路由，并使用 debug_backtrace 函数来打印方法调用过程。

```php
Route::get('test',function(){
 dump(debug_backtrace());
});
```

可见许多地方都跟 Pipeline 组件有关，并且重复执行一个闭包方法。

这里 pipes 数组就是需要用到的中间件。

### 中间件核心类 Pipeline

在 Laravel 框架 index.php 入口文件里，$kernel->handle() 方法就调用了 Pipeline 的方法，可以说它是贯穿始终的，这是把请求发到中间件进行处理的方法：

```php
protected function sendRequestThroughRouter($request)
{
 $this->app->instance('request', $request);

 Facade::clearResolvedInstance('request');

 $this->bootstrap();

 return (new Pipeline($this->app))
 ->send($request)
 ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
 ->then($this->dispatchToRouter());
}
```

其中 send 方法是设置 passable 属性也就是 $request，through 是设置 pipes 属性，也就是需要用到的中间件，是一个数组，重点是这里的 then 方法，参数也是一个闭包函数。

### array_reduce 妙用

这里就需要讲解一下 array_reduce 的用法了，可以说是妙用，这是理解 Laravel 中间件的重点。

先看一个官方例子明白它的基本用法：

```php
function sum($carry, $item)
{
 $carry += $item;
 return $carry;
}

$a = array(1, 2, 3, 4, 5);

var_dump(array_reduce($a, "sum")); // int(15)
```

array_reduce 会迭代每个元素，回调函数第一个参数是上次执行的结果，然后返回最终的一个值。

那么第二个参数的回调函数返回的是一个闭包呢？每次迭代返回的是一个闭包，也就是说 array_reduce 函数最终返回的也是一个闭包，除非执行这个闭包，否则里面的逻辑不会执行。

### 实现的核心

carry 方法返回一个闭包，作为 array_reduce 的回调函数。每次迭代就是把闭包函数丢到一个栈里面，后进先出。

最后，then 方法里 return $pipeline($this->passable) 才是调用 array_reduce 返回的最终的闭包，开始真正执行这些中间件了。

### 前置和后置中间件

执行 $next($request) 的时候就会把所有中间件都执行完，然后别忘了前面说的第一个闭包是 $this->dispatchToRouter() 提供的，它会进入到控制器逻辑，然后再是执行每个中间件中 $response = $next($request) 接下来的逻辑。

这也是前置中间件和后置中间件的原理。

### Laravel Pipeline 的现代应用

Laravel 的 Pipeline 模式不仅用于 HTTP 请求处理，还广泛应用于数据处理链式调用。在 Laravel 10+ 中，Pipeline 仍然是核心组件，被用于输入净化、格式化等多种场景。

### 参考资料

- [Laravel Pipeline 官方文档](https://laravel.com/docs/pipeline)
- [Laravel Pipeline for Input Sanitization - Medium](https://medium.com/@zulfikarditya/laravel-pipeline-for-input-sanitization-safe-reusable-and-fast-484009cc3323)
