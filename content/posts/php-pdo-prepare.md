---
title: "PHP：PDO prepare预处理"
date: 2012-09-30T04:03:39+08:00
draft: false
tags: ['php']
categories: ['tech']
description: "PHP PDO prepare预处理语句介绍"
---

许多成熟的数据库都支持预处理语句（Prepared Statements)的概念。它们是什么东西？你可以把它们想成是一种编译过的要执行的SQL语句模板，可以使用不同的变量参数定制它。预处理语句具有两个主要的优点：

查询只需要被解析（或准备）一次，但可以使用相同或不同的参数执行多次。当查询准备好（Prepared）之后，数据库就会分析，编译并优化它要执行查询的计划。对于复杂查询来说，如果你要重复执行许多次有不同参数的但结构相同的查询，这个过程会占用大量的时间，使得你的应用变慢。通过使用一个预处理语句你就可以避免重复分析、编译、优化的环节。简单来说，预处理语句使用更少的资源，执行速度也就更快。

<!--more-->

传给预处理语句的参数不需要使用引号，底层驱动会为你处理这个。如果你的应用独占地使用预处理语句，你就可以确信没有SQL注入会发生。

正因为预处理语句是如此有用，它成了PDO唯一为不支持此特性的数据库提供的模拟实现。

示例1：Prepare an SQL statement with named parameters

```php
$sql = 'SELECT name, colour, calories
 FROM fruit
 WHERE calories < :calories AND colour = :colour';
$sth = $dbh->prepare($sql, array(PDO::ATTR_CURSOR => PDO::CURSOR_FWDONLY));
$sth->execute(array(':calories' => 150, ':colour' => 'red'));
$red = $sth->fetchAll();
$sth->execute(array(':calories' => 175, ':colour' => 'yellow'));
$yellow = $sth->fetchAll();
```

示例2：Prepare an SQL statement with question mark parameters

```php
$sth = $dbh->prepare('SELECT name, colour, calories
 FROM fruit
 WHERE calories < ? AND colour = ?');
$sth->execute(array(150, 'red'));
$red = $sth->fetchAll();
$sth->execute(array(175, 'yellow'));
$yellow = $sth->fetchAll();
```
