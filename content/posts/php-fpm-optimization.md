---
title: "PHP-FPM性能优化参考"
date: 2016-03-29T16:55:16+08:00
draft: false
tags: ['php', 'php-fpm']
categories: ['tech']
description: "PHP-FPM性能优化参考"
---

php-fpm.conf有两个至关重要的参数：一个是"max_children"，另一个是"request_terminate_timeout"。

<!--more-->

我的两个设置的值一个是"40"，一个是"900"，但是这个值不是通用的，而是需要自己计算的。

计算的方式如下：

如果你的服务器性能足够好，且宽带资源足够充足，PHP脚本没有系循环或BUG的话你可以直接将"request_terminate_timeout"设置成0s。0s的含义是让PHP-CGI一直执行下去而没有时间限制。

而"max_children"这个值又是怎么计算出来的呢？这个值原则上是越大越好，php-cgi的进程多了就会处理的很快，排队的请求就会很少。设置"max_children"也需要根据服务器的性能进行设定，一般来说一台服务器正常情况下每一个php-cgi所耗费的内存在20M左右。
