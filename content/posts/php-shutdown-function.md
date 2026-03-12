---
title: "PHP捕捉异常中断"
date: 2016-09-28T09:31:50+08:00
draft: false
tags: ['php']
categories: ['tech']
description: "PHP捕捉异常中断的处理方法"
---

当 PHP 程序出现异常情况，如出现致命错误，超时，或者不可知的逻辑错误导致程序中断，这个时候可以用 register_shutdown_function 进行异常处理。

<!--more-->

比如判断一个脚本是否执行完成，可以设置一个属性为 false，在执行完成时设为 true，最后通过 register_shutdown_function 函数指定的方法进行判断，并做进一步异常处理：

```php
class IndexController extends Controller
{
    protected $complete = false;

    public function __construct()
    {
        register_shutdown_function([$this, 'shutdown']);
    }

    public function shutdown()
    {
        if ($this->complete === false) {
            //此处应该输出日志并进行异常处理操作
        }
    }
}
```

### register_shutdown_function 执行机制

1. 当页面被用户强制停止时
2. 当程序代码运行超时时
3. 当PHP代码执行完成时
