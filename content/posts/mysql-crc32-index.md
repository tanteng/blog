---
title: "MySQL 字符串字段转换 crc32 建索引提高查询效率"
date: 2016-02-01T01:29:07+08:00
draft: false
tags: ['mysql', 'crc32']
categories: ['tech']
description: "给字符串类型的字段建立索引效率不高，但可以通过crc32转换来提高查询效率"
---

给字符串类型的字段建立索引效率不高，但是必须要经常查这个字段怎么建索引？比如这个字段名称是 sys_trans_id 字符串类型，那么可以建一个字段 sys_trans_id_src32 来存储 crc32 的值，并给这个字段建立索引。

crc32 是整型，在MySQL中，给整型字段建立索引效率比较高，crc32虽然不能确保唯一性，但是无碍，相同的机率也是极小，关键是可以大大减少查询的范围，给sys_trans_id_src32 这个字段建立索引，查询的时候带上 crc32 字段就可以利用到索引。

<!--more-->

SQL 如下，比如要查询 sys_trans_id，同时带上 sys_trans_id_src32 走索引：

```sql
SELECT
 *
FROM
 `js_checking_third_detail`
WHERE
 (`biz_date` = '20160104')
AND (`diff_type` IN('2', '3'))
AND (
 `sys_trans_id_src32` = '509417929'
)
AND (
 `sys_trans_id` = '11451875264169885'
)
AND (`trans_type` = '1')
AND (`pay_type` = '1')
AND (`account_id` = '1')
ORDER BY
 diff_type DESC,
 biz_date DESC,
 id DESC
LIMIT 0,
 50
```

这样就通过 crc32 字段利用到了索引。

### crc32 的冲突概率

1）CRC32 在完全随机的输入情况下，冲突概率还是比较高的，特别是到了 1 亿的数据量后，冲突概率会更高

2）CRC32 在输入某个连续段的数据情况下，冲突概率反而很低，这是因为两个冲突的原值理论上应该是相隔很远，只输入某段数据的情况下，这个段里面冲突的原值会很少
