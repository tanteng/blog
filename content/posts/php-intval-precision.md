---
title: "PHP intval 转换浮点数精度丢失问题"
date: 2017-10-15T08:15:42+08:00
draft: false
tags: ['php']
categories: ['tech']
description: "PHP intval 转换浮点数精度丢失问题"
---

在 PHP 和其他一些语言都会存在这个问题，转换浮点数为整数的时候会出现精度丢失，如下：

```php
$n = "19.99";
print intval($n * 100); // prints 1998
```

<!--more-->

### 解决办法

1.使用 round 函数替代 floatval

```bash
php -r "echo round(19.99 * 100);"
// 输出 1999
```

2.先转换成字符串再取整

```php
print intval(strval($n * 100)); // prints 1999
```

3.使用 bc 系列函数

```php
echo bcmul('19.99', '100'); // 输出 1999
```

### 原因

这是因为浮点数的二进制表示精度问题导致的。
