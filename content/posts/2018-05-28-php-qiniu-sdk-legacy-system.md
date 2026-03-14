---
title: 在旧有 PHP 系统上集成七牛云 PHP-SDK
date: 2018-05-28T11:32:34+00:00
url: /2018/05/php-qiniu-sdk-legacy-system/
categories:
 - 七牛云
 - PHP
tags:
 - PHP
 - 七牛云
 - SDK

---

最近在一个古老的 PHP 系统上使用七牛云的上传图片功能，需要集成七牛云提供的 PHP-SDK，这个系统暂时不支持 composer，还好七牛云这个 SDK 不依赖其他的包，于是事情就变得简单了，只需要提供一个 spl_autoload_register 方法注册自动加载机制就可以使用了。

<!--more-->

```php
<?php

use Qiniu\Auth;
use Qiniu\Storage\UploadManager;

//本框架不支持 composer，为了用七牛云，又不敢在全局用 sql_autoload_register，目前就在当前活动引入七牛云的 SDK
spl_autoload_register('classLoader');

function classLoader($class)
{
 $path = str_replace('\\', DIRECTORY_SEPARATOR, $class);
 $file = SDK_PATH . 'vendor/' . $path . '.php';
 if (file_exists($file)) {
 require_once $file;
 }
}

require_once SDK_PATH . 'vendor/Qiniu/functions.php';


class xxxx extends baseIndex{

//

}
```

新建一个 vendor 文件夹存放第三方包，把七牛云 PHP-SDK 下载并复制到该目录下。

因为是某个需求使用，暂时不考虑全局使用，那么就在需要用的类文件里使用 spl_autoload_register 注册自动加载机制即可，如上面代码所示。

这样就可以开始使用七牛云的 PHP-SDK了。本文仅为在旧系统使用开源包提供一种思路和方式。
