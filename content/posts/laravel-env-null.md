---
title: "Laravel 使用 env 读取环境变量为 null 的问题"
date: 2016-12-04T15:35:18+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "Laravel 使用 env 读取环境变量为 null 的问题"
---

不知道大家有没有遇到过，在 Laravel 中使用 env 函数读取环境变量，有时有用，有时返回 null，究竟怎么回事？

在 Laravel 项目中，如果执行了 php artisan config:cache 命令把配置文件缓存起来后，在 Tinker 中使用 env 函数读取环境变量的值为 null，只有执行 php artisan config:clear 清除配置缓存后就可以读取了。

<!--more-->

### 原因何在？

在 Laravel 中，如果执行 php artisan config:cache 命令，Laravel 将会把 app/config 目录下的所有配置文件"编译"整合成一个缓存配置文件到 bootstrap/cache/config.php。一旦有了这个缓存配置文件，在其他地方使用 env 函数是读取不到环境变量的，所以返回 null。

因此，在配置文件即 app/config 目录下的其他地方，读取配置不要使用 env 函数去读环境变量，这样一旦执行 php artisan config:cache 之后，env 函数就不起作用了。
