---
title: PHP 调试函数 debug_backtrace
author: tanteng
type: post
date: 2017-09-11T07:37:49+00:00
url: /2017/09/php-debug-backtrace/
categories:
 - Develop
 - PHP

---
有时候我们想知道这个函数或方法的调用堆栈，也就是它是如何一级一级是被调用到的，可以用 PHP 的 debug_backtrace 函数打印。

<!--more-->

就像这样：

```php
public function update(Request $request, $id)
{
    dd(debug_backtrace());
    
    $getGameID = function ($request) {
        if (!$request->game_id) {
            return 1000 + intval($request->id);
        }
        return $request->game_id;
    };

    $previews = $this->getGamePreviews($request->game_preview);

    $request->merge([
        'game_preview' => json_encode($previews),
        'game_id' => $getGameID($request)
    ]);
    
    EgretGame::where('id', $id)->update($request->except(['_token', '_method']));
    return redirect()->route('egretgame.index')->with('success', '编辑成功！');
}
```

你可以控制需要回溯的堆栈层级数量，其中 debug_backtrace 第一个参数默认是一个常量 DEBUG_BACKTRACE_PROVIDE_OBJECT，表示显示这个对象的信息，第二个参数用于控制回溯的堆栈数量，默认是全部。

PHP 官方文档：<http://php.net/manual/zh/function.debug-backtrace.php>
