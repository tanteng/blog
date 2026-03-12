---
title: "查询数据库重复记录sql"
date: 2016-01-25T09:59:07+08:00
draft: false
tags: ['mysql', 'sql']
categories: ['tech']
description: "查询数据库重复记录sql语句"
---

每条记录都有个hash字段，hash是把这条记录几个不同的字段组成唯一的值进行hash算法存的一个值，有了这个hash，可以判断记录是否重复，sql语句如下：

```sql
SELECT COUNT(*) as total ,`hash` FROM `js_receipt_detail` GROUP BY `hash` HAVING total > 1;
```

<!--more-->

对hash进行group并筛选count个数大于1的。
