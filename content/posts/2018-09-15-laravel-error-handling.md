---
title: "Laravel 错误和异常处理用法"
date: 2018-09-15T05:41:51+00:00
url: /2018/09/laravel-error-handling/
categories:
 - tech

---
Laravel 自带错误和异常处理，App\Exceptions\Handler 负责上报异常和如何返回内容，以及未登录的处理。

<!--more-->

### 忽略异常

在 `$dontReport` 中可以定义忽略的异常类名：

```php
protected $dontReport = [
    \Illuminate\Auth\AuthenticationException::class,
    \Illuminate\Auth\Access\AuthorizationException::class,
    \Symfony\Component\HttpKernel\Exception\HttpException::class,
    \Illuminate\Database\Eloquent\ModelNotFoundException::class,
    \Illuminate\Validation\ValidationException::class,
];
```

这些异常就不会经过 report 方法。

### 几个重要方法

主要介绍这三个方法：report、render 和 unauthenticated 的用法。

#### report 方法

report 方法可以用来记录日志，可以根据不同的异常类型定制不同的日志级别和日志内容：

```php
if ($exception instanceof ABCException) {
    Log::emergency('ABC异常', $context);
} else if ($exception instanceof HeheException) {
    Log::info('Hehe异常', $context);
}
```

report 方法没有返回值，也不应该在这里中断程序。

#### render 方法

render 方法可以根据不同的异常类型返回不同的数据：

```php
if (get_class($exception) == 'Exception' || $exception instanceof NotAllowedException) {
    return response()->json(['message' => $exception->getMessage()], 400);
} elseif ($exception instanceof ValidationException) {
    return response()->json(['message' => '校验失败', 'errors' => $exception->validator->errors()], 400);
}
```

#### unauthenticated 方法

在访问需要登录态的页面时，用户未登录就会进入这个方法进行处理：

```php
protected function unauthenticated($request, AuthenticationException $exception)
{
    if ($request->expectsJson()) {
        return response()->json(['error' => 'Unauthenticated.'], 401);
    }

    // 如果是后台页面未认证，跳转到后台登陆页面
    $guard = $exception->guards();
    if (in_array('admin', $guard)) {
        return redirect()->guest('/admin/login');
    }

    return redirect()->guest('login');
}
```

如果是返回 json，则统一返回格式。默认情况下返回前台的登录页，如果是访问后台页面未登录，则跳转到后台登录页。
