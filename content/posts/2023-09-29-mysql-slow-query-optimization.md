---
title: "另辟蹊径的MySQL慢查询优化"
date: 2023-09-29T10:00:00+08:00
draft: false
categories: ["tech"]
tags: ["mysql", "database", "performance-optimization", "index-optimization"]
description: "总结项目中几个真实慢查询案例的优化思路，有些方案稍显「偏门」，但亲测有效。"
---

做业务开发的同学，大概都有过被 MySQL 慢查询折磨的经历——明明索引建了、数据量也不大，接口却总是超时。常规的加索引、加缓存套路固然有效，但有时候换个思路反而能四两拨千斤。本文整理了项目中几个真实慢查询案例的优化思路，部分方案稍显「偏门」，但亲测有效。

<!--more-->

## 1. 索引优化：别让索引形同虚设

索引是加速查询最直接的手段，但「建了索引却没生效」是高频踩坑点。常见原因有：

- **索引列参与计算或函数运算**，如 `WHERE YEAR(created_at) = 2025`
- **使用负向查询**，如 `WHERE status != 1` 或 `WHERE id NOT IN (1,2,3)`
- **隐式类型转换**，后文会单独展开

举一个容易忽略的例子：联合索引 `privileges_target_type_target_id_index` 为 `(target_type, target_id)`，但查询条件写成：

```sql
WHERE target_id = 568  -- target_id 是 varchar，传入整数 568
```

此时数据库对 `target_id` 做了隐式类型转换，索引失效，查询从索引扫描退化为全表扫描。解决方案是在代码层将整数转为字符串，严格匹配类型。

## 2. 分页性能优化：告别深度 OFFSET

这是一个常见的导致慢查询的场景：

```sql
SELECT * FROM some_table LIMIT N OFFSET M
```

当 M 很大时，比如 M = 200000，数据库需要先扫描并跳过前 20 万条数据，再返回 N 条。如果索引使用不当，加上其他排序条件，性能影响更为显著。

**优化方案一：游标分页（推荐）**

用主键或唯一列作为游标，翻页时带上上次的锚点：

```sql
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

通过缩小查询范围（如加日期条件）降低扫描量，避免 SELECT *，只取必要字段。

## 3. 排序性能优化：混合排序方向的索引抢救

来看一个典型案例：

```sql
SELECT * FROM `point_ranks`
WHERE `period` = 'week' AND `time` = '202228'
ORDER BY `point` DESC, `created_at` ASC, `staff_id` DESC
LIMIT 1000;
```

ORDER BY 中 point 和 created_at 的排序方向不一致（DESC vs ASC），导致联合索引只能部分生效，排序效率大打折扣。

**MySQL 8 优化方案：**

创建不同排序方向的联合索引：

```sql
CREATE INDEX idx_order ON point_ranks(time, period, point DESC, created_at ASC, staff_id DESC);
```

**MySQL 5.6 兼容方案（老版本适用）：**

新增一个虚拟字段 `created_at_reverse`，存储 `created_at` 时间戳的负数：

```sql
-- 新增字段 created_at_reverse = -created_at
-- 新建索引
CREATE INDEX idx_order ON point_ranks(time, period, point, created_at_reverse, staff_id);
```

查询改写为：

```sql
ORDER BY point DESC, created_at_reverse ASC, staff_id DESC
```

所有字段排序方向一致，索引完整利用，查询性能大幅提升。

## 4. COUNT 性能优化：别让总数拖累了列表

高频列表接口中，COUNT 查询往往是隐藏的性能杀手——每次返回列表数据时还要单独计算总数，而总数变更频繁，缓存容易失效。

**优化思路：**

- **接受近似值**：对于展示性需求（如列表页顶部的那句「共 XXX 条」），可以用 Redis 缓存一个近似值，不必每次实时 COUNT。
- **分页时跳过 COUNT**：第一页显示总页数（可缓存），后续页码用前端计算，不在每次请求时实时 COUNT。
- **覆盖索引**：如果必须 COUNT，确保只扫描索引而非回表。

## 5. 拆分 INNER JOIN：把连表拆成两条 SQL

来看一个真实案例：

```sql
SELECT staffs.*, survey_staff.survey_id as pivot_survey_id,
       survey_staff.staff_id as pivot_staff_id,
       survey_staff.created_at as pivot_created_at,
       survey_staff.updated_at as pivot_updated_at
FROM staffs
INNER JOIN survey_staff ON staffs.id = survey_staff.staff_id
WHERE survey_staff.survey_id IN ('?')
ORDER BY pivot_created_at ASC
LIMIT 5;
```

分析问题：联合索引不合理，INNER JOIN 扩大了扫描范围，ORDER BY 也在大结果集上执行。

**优化方案：拆成两条 SQL**

```sql
-- ① 先从关联表查出符合条件的 ID（利用索引）
SELECT staff_id FROM survey_staff
WHERE survey_id = ?
ORDER BY created_at ASC
LIMIT 5;
```

需要新建索引 `(survey_id, created_at)`。

```sql
-- ② 再批量查询实体数据
SELECT * FROM staffs WHERE id IN (?, ?, ?, ?, ?);
```

这种「先查索引再查实体」的模式，将大范围 JOIN 拆解为两次小范围查询，大幅减少扫描量。

## 6. 避免 N+1 查询：关联数据批量加载

在循环里分别查询关联数据，是最容易被忽视的性能杀手：

```php
foreach ($users as $user) {
    $user->posts = Post::where('user_id', $user->id)->get();
}
```

随着关联数据越来越多，一个列表可能产生几百条查询，接口响应极慢。

**优化方案：** 在 ORM 层使用 `with()` / `preload()` 预加载关联数据，把 N+1 合并成 1~2 条查询：

```php
$users = User::with('posts')->get();
```

## 7. 字段类型不一致：隐式类型转换的坑

来看这条看起来很正常的 SQL：

```sql
SELECT id, visible_type, visible_id, created_at, options
FROM privileges
WHERE target_id = 568
  AND target_id IS NOT NULL
  AND target_type = 'document'
  AND visible_type = 'staff'
ORDER BY id DESC
LIMIT 20 OFFSET 0;
```

privileges 表的 target_id 是 varchar 类型，联合索引 `(target_type, target_id)` 本应生效，但由于传入的是整数 568，MySQL 做了隐式类型转换，索引直接失效。

**优化方案：** 实体的 target 是 MorphiMany 关联关系，要确保不同实体的 target_id 类型与索引列一致，代码层做类型转换：

```php
// 将整数转为字符串
$targetId = (string) $targetId;
```

## 8. 使用虚拟字段：以空间换时间

当索引优化已经做到极致，慢查询依然存在时（比如数据量过亿、并发上万），可以考虑：

- **预计算/反规范化**：把复杂计算结果存进字段，查询时直接读，比如将多表关联的结果同步到一个冗余字段。
- **影子字段**：如前文提到的 `created_at_reverse`，通过冗余字段解决排序方向不一致的问题。

核心思路是「读写换位」，用写入时的一点额外成本，换取查询时的极致性能。

## 9. 大表拆表：从源头降低单表数据量

当数据持续增长，单表数据过千万级别时，即使索引再优化也很难根本解决性能问题：

- **按时间拆表**：按月/季度分表，热数据放新表，历史数据归档。
- **按业务维度拆库**：不同业务线走不同库，单库数据量可控。
- **使用自动分表中间件**：如 TDSQL 等，逻辑分表，物理上也做了隔离，业务无需关心底层路由逻辑。

## 10. 同步 ES 查询：把复杂查询移出 MySQL

对于需要模糊搜索、多条件组合、排序维度多的查询场景，可以将数据同步到 Elasticsearch，在 ES 侧完成检索，MySQL 只负责结构化存储和事务写入。

这适合搜索量大但写一致性要求不高的场景，如商品搜索、日志分析、内容检索等。

---

慢查询优化的本质是「找到查询的瓶颈点，然后针对性地消除它」。索引、回表、扫描量、关联方式、类型转换——每一个细节都可能是性能的关键。希望这几个实战案例，能给你提供一些新的思路。