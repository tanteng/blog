---
title: "MySQL 大表加字段思路"
date: 2017-01-19T12:58:04+08:00
draft: false
tags: ['mysql']
categories: ['tech']
description: "MySQL 大表加字段思路和方法"
---

给 MySQL 一张表加字段执行如下 sql 就可以了：

```sql
ALTER TABLE tbl_tpl ADD title VARCHAR(255) DEFAULT '' COMMENT '标题' AFTER id;
```

但是线上的一张表如果数据量很大，执行加字段操作就会锁表，这个过程可能需要很长时间甚至导致服务崩溃。

<!--more-->

### 大表加字段思路

① 创建临时新表，复制旧表结构

```sql
CREATE TABLE new_table LIKE old_table;
```

② 给新表加上新增的字段

③ 把旧表的数据复制过来

```sql
INSERT INTO new_table(field1, field2...) SELECT field1, field2... FROM old_table
```

④ 删除旧表，重命名新表的名字为旧表的名字

### 注意

执行第三步时，可能需要时间，这时有新数据进来。如果表有记录写入时间的字段，可以找到执行这一步操作之后的数据，重复导入到新表，直到数据差异很小。不过还是会可能损失极少量的数据。

如果表的数据特别大，同时又要保证数据完整，最好停机操作。

### 其他方法

1. 在从库进行加字段操作，然后主从切换
2. 使用第三方在线改字段的工具

一般情况下，十几万的数据量，可以直接进行加字段操作。
