---
title: "MySQL查看数据库大小"
date: 2016-06-03T11:24:42+08:00
draft: false
tags: ['mysql']
categories: ['tech']
description: "MySQL查看数据库大小的方法"
---

1、进入information_schema 数据库（存放了其他的数据库的信息）

```sql
use information_schema;
```

2、查询所有数据的大小：

```sql
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables;
```

<!--more-->

3、查看指定数据库的大小：

```sql
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables where table_schema='home';
```

4、查看指定数据库的某个表的大小

```sql
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables where table_schema='home' and table_name='members';
```
