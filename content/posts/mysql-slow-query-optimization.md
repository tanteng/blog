---
title: '10 个另辟蹊径的 MySQL 慢查询优化'
slug: 'mysql-slow-query-optimization'
date: 2023-09-29T10:00:00+08:00
draft: false
tags: ['mysql', 'performance-optimization', 'database']
categories: ['tech']
description: '索引、分页、JOIN、排序……常规优化手段大家都会，但实际项目中有些慢查询靠这些套路根本解不了。本文整理了 10 个真实慢查询案例的优化思路，部分方案稍显偏门，但亲测有效。'
---

做业务开发的同学，大概都有过被 MySQL 慢查询折磨的经历——明明索引建了、数据量也不大，接口却总是超时。常规的加索引、加缓存套路固然有效，但有时候换个思路反而能四两拨千斤。本文整理了项目中几个真实慢查询案例的优化思路，部分方案稍显「偏门」，但亲测有效。

<!--more-->

## 1. 索引优化：别让索引形同虚设

索引是加速查询最直接的手段，但「建了索引却没生效」是高频踩坑点。常见原因有：

- 索引列参与计算或函数运算，如 `WHERE YEAR(created_at) = 2025`
- 使用了 `LIKE '%keyword'` 前缀通配符
- 隐式类型转换导致索引失效（后文详述）
- 联合索引的列顺序与查询条件不匹配

**优化思路：** 拿到慢查询后，先用 `EXPLAIN` 看执行计划，确认是否走了索引、走了哪颗索引、是否需要回表。索引失效不一定是 SQL 写的问题，有时调整列顺序或增加覆盖索引就能解决。

## 2. 分页性能优化：OFFSET 是隐藏的性能杀手

分页是最常见的慢查询场景之一：

```sql
SELECT * FROM some_table LIMIT 20 OFFSET 200000;
```

当 OFFSET 很大时，数据库会从第 0 行开始扫描，跳过前 20 万行再返回 20 条数据——这 20 万行扫描是实打实的 IO 开销，LIMIT 再小也没用。

**优化方案一：游标分页（Keyset Pagination）**

不用 OFFSET，基于上一页最后一条记录的 ID 做条件：

```sql
-- 第一页
SELECT * FROM some_table WHERE id > 0 ORDER BY id ASC LIMIT 20;

-- 下一页，传入上一页最大 ID
SELECT * FROM some_table WHERE id > 200000 ORDER BY id ASC LIMIT 20;
```

无论翻到第几页，查询复杂度始终是 O(1)。缺点是不支持随机跳页。

**优化方案二：避免回表**

如果只需要索引列的数据，直接 select 索引列，不触发回表：

```sql
-- 只取索引列，无需回表
SELECT id, created_at FROM some_table ORDER BY id DESC LIMIT 20;
```

**优化方案三：调整查询条件减少扫描范围**

结合业务逻辑在 WHERE 条件中加入日期或状态过滤，把数据总量压下去，比单纯优化分页本身更有效。

## 3. 排序性能优化：ORDER BY 字段排序规则不一致

来看一个实际案例：

```sql
SELECT * FROM `point_ranks`
WHERE `period` = 'week' AND `time` = '202228'
ORDER BY `point` DESC, `created_at` ASC, `staff_id` DESC
LIMIT 1000;
```

分析：ORDER BY 中 `point` DESC（倒序）、`created_at` ASC（正序）、`staff_id` DESC（倒序）混用，导致索引只能匹配部分列，排序只能在 filesort 里做，数据量大了就很慢。

**MySQL 8 优化方案：** 创建不同排序方向的联合索引：

```sql
CREATE INDEX idx_order ON point_ranks(time, period, point, created_at, staff_id);
-- 搭配 DESC 排序：point DESC, created_at ASC, staff_id DESC
```

**MySQL 5.6 优化方案（兼容老版本）：** 新增一个虚拟字段 `created_at_reverse`，存储 `created_at` 时间戳的负数：

```sql
-- 新增字段 created_at_reverse = -created_at
-- 新建索引
CREATE INDEX idx_order ON point_ranks(time, period, point, created_at_reverse, staff_id);
```

这样 ORDER BY 所有字段方向一致（正序），索引完整覆盖排序阶段，查询直接从索引返回数据。

## 4. COUNT 性能优化：别用 COUNT(*) 数大表

`SELECT COUNT(*)` 在数据量大的表上非常慢，因为 MySQL 需要逐行扫描来统计。

**优化思路：**

- 用 `WHERE` 条件限定范围，结合缓存的计数器字段
- 如果只需要近似值，用 `SHOW TABLE STATUS LIKE 'table_name'` 的估算值
- 业务层面维护一个 `counter_cache` 表，专门存各维度的统计数

## 5. 拆分 INNER JOIN：化繁为简

来看一个连表慢查询：

```sql
SELECT `staffs`.*, `survey_staff`.`survey_id` AS `pivot_survey_id`,
       `survey_staff`.`staff_id` AS `pivot_staff_id`,
       `survey_staff`.`created_at` AS `pivot_created_at`,
       `survey_staff`.`updated_at` AS `pivot_updated_at`
FROM `staffs`
INNER JOIN `survey_staff` ON `staffs`.`id` = `survey_staff`.`staff_id`
WHERE `survey_staff`.`survey_id` IN ('?')
ORDER BY `pivot_created_at` ASC
LIMIT 5;
```

分析：联合索引不合理，INNER JOIN 扩大了扫描范围。

**优化方案：拆成两条 SQL**

```sql
-- ① 先从关联表查出符合条件的 ID
SELECT `staff_id`
FROM `survey_staff`
WHERE `survey_id` = ?
ORDER BY `created_at` ASC
LIMIT 5;

-- ② 再查主表
SELECT * FROM `staffs` WHERE `id` IN (?, ?, ?);
```

新建复合索引 `survey_id, created_at`，第一条 SQL 直接从索引返回，无需回表。第二条 SQL 用主键查，直接走主键索引。两个小查询都比原来的单条大查询快得多。

**适用场景：** 分页获取关联实体，且排序字段在关联表上。核心思路是把 JOIN 拆成「先过滤后查主表」，把数据量压到最小再关联。

## 6. 避免 N+1 查询：批量处理代替循环查询

这是 ORM 时代的高频性能杀手。在循环里逐条查询关联数据，数据量稍大就变成 N+1：

```
查询 1：SELECT * FROM orders LIMIT 100;         -- 1 次
查询 2~101：SELECT * FROM users WHERE id = ?;   -- N 次
```

一个列表页轻轻松松产生几百次数据库查询，RT（响应时间）爆炸。

**优化思路：** 用 `WHERE id IN (?, ?, ...)` 批量查，或者在 ORM 层用 `with()` / `preload()` 预加载关联数据，把 N+1 合并成 1~2 条查询。

## 7. 字段类型不一致：隐式类型转换的坑

来看这条看起来很正常的 SQL：

```sql
SELECT `id`, `visible_type`, `visible_id`, `created_at`, `options`
FROM `privileges`
WHERE `target_id` = 568
  AND `target_id` IS NOT NULL
  AND `target_type` = 'document'
  AND `visible_type` = 'staff'
ORDER BY `id` DESC
LIMIT 20 OFFSET 0;
```

分析：`privileges` 表的 `target_id` 是 `varchar` 类型，但传入的条件 `= 568` 是一个整数。MySQL 会在查询时把 `varchar` 转成整数，导致索引 `privileges_target_type_target_id_index(target_type, target_id)` 失效，变成全表扫描。

**优化方案：** 确保传入的类型与字段类型一致。代码层面在拼接查询条件时做类型转换（统一转成字符串），或者在业务入口处做类型校验，避免隐式转换破坏索引命中。

## 8. 使用虚拟字段：用空间换时间

当索引优化已经做到极致，查询仍然慢（如数据量过亿、并发很高），可以考虑在表里增加冗余字段，专门为高频查询服务：

- **预计算字段**：如「最近 7 天销量」「用户等级」等，用定时任务或触发器维护
- **统计快照字段**：替代实时 COUNT，把计数结果存下来
- **宽表设计**：将多表 join 的结果物化到一张表，减少运行时计算

核心思路是「读写换位」，用写入时的一点额外成本，换取查询时的极致性能。

## 9. 大表拆表：从源头降低单表数据量

当数据持续增长，单表数据过千万级别时，即使索引再优化也很难根本解决性能问题：

- **按时间拆表**：按月/季度分表，热数据放新表，历史数据归档
- **按业务维度拆库**：不同业务线走不同库，单库数据量可控
- **使用自动分表中间件**：如 TDSQL 等，逻辑分表，物理上也做了隔离，业务无需关心底层路由逻辑

## 10. 同步 ES 查询：把复杂查询移出 MySQL

对于需要模糊搜索、多条件组合、排序维度复杂的查询，可以考虑把数据同步到 Elasticsearch：

- MySQL 负责事务写入和数据落地
- ES 负责查询层，承担搜索、过滤、聚合等负担
- 通过 binlog 或消息队列实时同步数据

MySQL 的强项是事务和点查，ES 的强项是全文检索和复杂过滤，两者配合效果最好。

## 总结

慢查询的优化没有银弹，核心思路是：

1. **先定位**：用 EXPLAIN 找到根因，别盲猜
2. **索引优先**：确保查询能走索引、走对索引
3. **换思路**：JOIN 变批量查询、OFFSET 变游标分页、老版本加虚拟字段
4. **架构兜底**：数据量大了该拆表拆表、该上 ES 上 ES

希望这些思路对你有帮助。如果有其他优化案例，欢迎交流。