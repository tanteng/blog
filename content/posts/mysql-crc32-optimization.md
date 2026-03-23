---
title: "MySQL 使用 CRC32 做索引优化查询"
date: 2016-02-01T01:29:07+08:00
draft: false
tags: ['mysql', 'index-optimization']
categories: ['tech']
description: "MySQL 字符串字段通过 CRC32 转换建索引提高查询效率"
---

给字符串类型的字段建立索引效率不高，但如果需要经常查询这个字段，可以通过 CRC32 转换来提高查询效率。

假设有一个字符串字段 `sys_trans_id`，需要频繁查询。可以新增一个整型字段 `sys_trans_id_crc32` 来存储 CRC32 的值，并在这个字段上建立索引。

<!--more-->

CRC32 是 32 位整型，在 MySQL 中给整型字段建立索引效率很高。

### 查询示例

```sql
SELECT *
FROM js_checking_third_detail
WHERE biz_date = '20160104'
  AND diff_type IN('2', '3')
  AND sys_trans_id_crc32 = CRC32('11451875264169885')
  AND sys_trans_id = '11451875264169885'
  AND trans_type = '1'
  AND pay_type = '1'
  AND account_id = '1'
ORDER BY diff_type DESC, biz_date DESC, id DESC
LIMIT 50
```

关键点：同时使用 `crc32` 字段和原文字段。CRC32 字段走索引快速筛选，原始字段确保精确匹配。

### CRC32 冲突概率

1. **完全随机输入**：冲突概率较高，特别是数据量达到 1 亿后
2. **连续段数据输入**：冲突概率很低，因为冲突的原值理论上相隔很远

### 适用场景

- 字符串字段查询频繁但长度较大
- 对精确性要求不是绝对严格（允许极小概率的冲突）
- 需要大幅缩小查询范围提升性能
