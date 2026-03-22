---
title: 深入理解 PHP 写时复制机制
date: 2018-03-26T14:04:07+00:00
url: /2018/03/php-copy-on-write-mechanism/
categories: ['tech']
tags:
 - PHP
 - 内存管理

---

一个例子：

```php
<?php
$foo = 1;
$bar = $foo;
echo $foo + $bar;
```

变量 $foo 赋值给变量 $bar，这两个变量具有相同的值，没有必要新申请内存空间，他们可以共享同一块内存。在很多场景下 PHP 的 COW 对内存进行优化。比如：变量的多次赋值、函数参数传递，并在函数体内修改实参等。

<!--more-->

### 什么是写时复制

```php
<?php
$var = "laruence";
$var_dup = $var;
$var = 1;
```

很明显在这段代码执行以后，$var_dup 的值应该还是"laruence"，这就是 PHP 的 copy on write 机制：

PHP 在修改一个变量以前，会首先查看这个变量的 refcount，如果 refcount 大于1，PHP 就会执行一个分离的例程，将原 zval 的 refcount 减1，并修改 symbol_table，使得变量分离。这个机制就是所谓的 copy on write(写时复制)。

### 写时复制应用场景

写时复制（Copy on Write，也缩写为 COW）的应用场景非常多，比如 Linux 中对进程复制中内存使用的优化，在各种编程语言中，如 C++ 的 STL 等等中均有类似的应用。COW 是常用的优化手段，可以归类于：资源延迟分配。

一个证明 PHP COW 优化内存占用的例子：

```php
<?php
$j = 1;
var_dump(memory_get_usage());

$tipi = array_fill(0, 100000, 'php-internal');
var_dump(memory_get_usage());

$tipi_copy = $tipi;
var_dump(memory_get_usage());

foreach ($tipi_copy as $i) {
    $j += count($i);
}
var_dump(memory_get_usage());
```

运行结果：

```
int(630904)
int(10479840)
int(10479944)
int(10480040)
```

内存并没有显著提高。

### 何为复制

多个相同值的变量共用同一块内存的确节省了内存空间，但变量的值是会发生变化的，如果指向同一内存的值发生了变化，就需要将变化的值"分离"出去，这个"分离"的操作就是"复制"。

在 PHP 中，Zend 引擎为了区别同一个 zval 地址是否被多个变量共享，引入了 ref_count 和 is_ref 两个变量进行标识：

- **is_ref**：标识是不是用户使用 & 的强制引用
- **ref_count**：是引用计数，用于标识此 zval 被多少个变量引用

### 实现原理

PHP 中的 COW 基于引用计数 ref_count 和 is_ref 实现，多一个变量指针，就将 ref_count 加1，反之减去1，减到0就销毁；同理，多一个强制引用 &,就将 is_ref 加1，反之减去1。

### 2025年更新

PHP 8.x 仍然延续了 COW 机制，但在某些场景下有优化改进。在现代 PHP 版本中，字符串和数组类型都采用写时复制策略，只有在真正需要修改时才会进行内存复制。

### 参考资料

- [深入理解PHP内核 - 写时复制](https://docs.kilvn.com/tipi/chapt06/06-06-copy-on-write.html)
- [PHP 官方手册 - Reference Counting Basics](https://www.php.net/manual/en/features.gc.refcounting-basics.php)
- [维基百科 - 写入时复制](https://zh.wikipedia.org/zh-hans/%E5%AF%AB%E5%85%A5%E6%99%82%E8%A4%87%E8%A3%BD)
