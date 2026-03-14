---
title: "MySQL 字段为 NULL 的查询问题"
date: 2018-10-30T03:29:00+08:00
draft: false
tags: ['mysql']
categories: ['tech']
description: "MySQL 字段为 NULL 的查询问题"
---

MySQL 中 NULL 的处理与普通值不同，需要特别注意。

<!--more-->

### NULL 查询问题

假设某数据表 code 字段可以为 NULL：

```sql
-- 这个查不出 code IS NULL 的数据
... AND code NOT IN ()

// 正确写法
... AND (code IS NULL OR code NOT IN())
```

SQL NULL 是特殊的，不能使用 `= NULL` 或 `!= NULL`，必须使用 `IS NULL` 或 `IS NOT NULL`。

> NULL = NULL 永远为 false

### 不建议使用 NULL 的理由

1. NULL 和 "" 不同，查询 `code != 'xx'` 的记录不包含 NULL，必须加 `AND code IS NULL`，容易遗漏
2. MySQL 难以优化引用可空列查询，会使索引、索引统计和值更加复杂
3. 可空列需要更多存储空间，MySQL 内部需要特殊处理
4. 可空列被索引后，每条记录需要额外一个字节，可能导致 MyISAM 中固定大小索引变成可变大小索引

—— 出自《高性能 MySQL》第二版

### 建议

除 timestamps 这种字段外，都用 `NOT NULL DEFAULT ''` 表示。

关于 NULL 和唯一索引：唯一性索引是允许多个 NULL 值存在的。
