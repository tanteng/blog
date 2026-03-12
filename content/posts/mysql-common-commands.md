---
title: "常用Linux操作数据库命令及MySQL语句"
date: 2016-06-15T11:29:06+08:00
draft: false
tags: ['mysql', 'linux']
categories: ['tech']
description: "常用Linux操作数据库命令及MySQL语句"
---

以下是在Linux下经常会用到的MySQL的一些命令，导出，导入，建库建表，备份，以及MySQL修改字段，添加字段等语法。

<!--more-->

### 数据库表导入

```bash
mysql -uroot -p tanteng.me < mobile_promote.sql
```

### 数据库表导出

#### 导出整个数据库

```bash
mysqldump -u root -p tanteng.me > www.sql
```

### MySQL增加修改字段

#### 增加字段

```sql
ALTER TABLE js_business_bank_info ADD bbi_ramdom char(100) DEFAULT '' COMMENT '生成的随机银行备注' AFTER bbi_source;
```

#### 修改字段

修改字段类型：

```sql
ALTER TABLE supplier MODIFY supplier_name char(100) NOT NULL;
```

修改字段名称：

```sql
ALTER TABLE new_mobile_promote CHANGE `promote_link` `link` varchar(255);
```
