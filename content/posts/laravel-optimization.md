---
title: "优化 Laravel 网站打开速度"
date: 2016-06-02T11:59:07+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "Laravel是一个功能强大的框架，组件很多，代码也很庞大，它的易用方便是牺牲了性能的，即便如此它仍然是一个优秀的框架，但在正式环境下要做好优化提升网站的打开速度。"
---

Laravel是一个功能强大的框架，组件很多，代码也很庞大，它的易用方便是牺牲了性能的，即便如此它仍然是一个优秀的框架，但在正式环境下要做好优化提升网站的打开速度。

<!--more-->

### 1.关闭debug

打开.env文件，把debug设置为false.

### 2.缓存路由和配置

```bash
php artisan route:cache
php artisan config:cache
```

### 3.Laravel优化命令

```bash
php artisan optimize
```

### 4.composer优化

```bash
sudo composer dump-autoload optimize
```

### 5.使用Laravel缓存

### 6.使用CDN

### 7.使用PHP 7并开启OPcache

### 8.nginx开启gzip压缩
