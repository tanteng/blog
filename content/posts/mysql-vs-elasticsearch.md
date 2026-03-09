---
title: "MySQL vs Elasticsearch：使用场景深度解析"
date: 2023-08-23
draft: false
tags: ["Tech", "Database", "Elasticsearch", "MySQL"]
categories: ["Tech"]
---

在做系统设计时，数据库选型是一个关键决策。MySQL 和 Elasticsearch 是两种不同定位的存储方案，今天结合实际项目经验，系统性地对比一下它们的适用场景。

## MySQL 和 Elasticsearch 的定位差异

| 特性 | MySQL | Elasticsearch |
|------|-------|---------------|
| **定位** | 关系型数据库 | 分布式搜索和分析引擎 |
| **数据结构** | 行存储，B+树索引 | 倒排索引，文档型 |
| **擅长** | 事务、关联查询、结构化数据 | 全文搜索、聚合分析、海量数据 |
| **数据模型** | Table（表） | Index（索引），Document（文档） |

<!--more-->

> MySQL 是**正排索引**（文档 → 词），ES 是**倒排索引**（词 → 文档）。这是两者最核心的差异。

## MySQL 适用场景

MySQL 适合以下场景：

1. **结构化数据**：数据有固定 schema，需要严格的关系型约束
2. **事务需求**：需要 ACID 事务保证数据一致性
3. **关联查询**：多表 JOIN 查询是常态
4. **数据量适中**：百万级以下，查询模式固定
5. **简单条件查询**：精确匹配、范围查询，不需要全文搜索

典型的 MySQL 业务场景：
- 用户账户系统（需要事务）
- 电商订单系统（关联查询多）
- 内容管理系统（结构化内容）
- **推荐模块**（关联查询为主）

## 必须使用 Elasticsearch 的场景

### 1. 全文搜索

**场景**：课程内容搜索，用户输入"财务报表分析"，想搜到包含"财务报表"、"财务分析"、"会计报表"等内容。

**MySQL 方案**：
```sql
SELECT * FROM courses 
WHERE title LIKE '%财务报表%' OR content LIKE '%财务报表%';
-- 索引失效，全表扫描千万级数据需要 3-10 秒
```

**ES 方案**：
```json
GET /courses/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "财务报表分析" }},
        { "match": { "content": "财务报表分析" }}
      ]
    }
  }
}
```

**原理**：ES 使用倒排索引 + IK 分词器 + BM25 算法，实现近义词匹配和相关性排序。

---

### 2. 海量数据

**场景**：电商平台商品库 5000 万条，需要按关键词搜索并分页。

**MySQL 方案**：
```sql
SELECT * FROM products 
WHERE name LIKE '%手机%' 
ORDER BY sales DESC 
LIMIT 20 OFFSET 1000;
-- OFFSET 越大越慢，可能要 30+ 秒
```

**ES 方案**：
```json
GET /products/_search
{
  "from": 1000,
  "size": 20,
  "query": { "match": { "name": "手机" }},
  "sort": [{ "sales": "desc" }]
}
```

**原理**：ES 分布式架构下，每个分片独立排序，汇总后全局排序。使用 `search_after` 游标方式可解决深度分页，延迟基本恒定在 50-200ms。

---

### 3. 复杂聚合（Faceted Search）

**场景**：搜索"笔记本电脑"，需要同时返回各品牌、数量、价格区间分布。

**MySQL 方案**：需要 4-5 次查询才能完成。

**ES 方案**：一次查询同时返回搜索结果和聚合统计：
```json
GET /products/_search
{
  "size": 20,
  "query": { "match": { "name": "笔记本" }},
  "aggs": {
    "brand_agg": { "terms": { "field": "brand", "size": 10 }},
    "price_agg": { 
      "range": { 
        "field": "price",
        "ranges": [
          { "to": 3000 },
          { "from": 3000, "to": 6000 },
          { "from": 6000 }
        ]
      }
    }
  }
}
```

---

### 4. 相关性排序（BM25）

**场景**：搜索"Python 学习"，标题含"Python"的应该比正文中提到的排在前面。

**ES 方案**：
```json
GET /articles/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": { "query": "Python学习", "boost": 3 }}},
        { "match": { "content": "Python学习" }}
      ],
      "should": [
        { "term": { "is_premium": { "boost": 2 }}},
        { "range": { "views": { "boost": 1.5, "gte": 1000 }}}
      ]
    }
  }
}
```

**原理**：BM25 算法综合考虑词频（TF）、逆文档频率（IDF）、字段长度三个因子，实现精准的相关性排序。

---

### 5. 近实时（NRT）

**场景**：资讯平台发布文章后，要求 3 秒内能被搜索到。

**ES 方案**：
```json
POST /articles/_doc/1
{
  "title": "新闻标题",
  "content": "新闻内容"
}

# 默认 1 秒刷新，文档立即可搜索
GET /articles/_search?q=新闻
```

**原理**：ES 的 Near Real-Time 特性，Write → Memory Buffer → Refresh（每秒一次生成新 Segment）→ 可搜索。

---

## 核心技术概念解析

### 倒排索引（Inverted Index）

一种**词 → 文档**的映射结构，与 MySQL 的 B+树（文档 → 词）相反。

```
正排索引（MySQL）：
Doc1 → [title: "Python教程", content: "Python入门学习"]

倒排索引（ES）：
"Python" → [Doc1]
"教程"   → [Doc1, Doc2]
"入门"   → [Doc1]
```

### IK 分词器

中文没有空格，需要分词器把句子切成"词"。

```
原始："Python入门教程"

ik_max_word：["Python", "入门", "教程", "Python入门", "入门教程"]
ik_smart：   ["Python", "入门教程"]
```

### BM25 算法

排序算法，决定搜到一个词时哪个文档该排前面。

核心因子：
- **IDF**：词越稀有，权重越高（"Python"比"的"重要）
- **TF**：词在文档中出现越多越相关
- **字段长度**：短字段匹配比长字段权重高

### 工作流程

```
用户输入："Python教程"

     IK分词
       ↓
"Python" + "教程"

     倒排索引
       ↓
查 "Python" → [Doc1, Doc2]
查 "教程"   → [Doc1, Doc3]

     BM25排序
       ↓
Doc1: TF=3, 标题命中 → Score=15.2
Doc2: TF=1, 正文命中 → Score=8.1

     返回结果
       ↓
[Doc1, Doc2...]
```

## 实战：商品推荐模块选型分析

最近在设计一个**电商首页推荐模块**：
- 根据用户最近浏览的商品分类（Top5）
- 通过分类标签推荐相关商品

**为什么选择 MySQL 而不是 ES？**

1. **查询模式固定**：用户 ID → 最近浏览分类 → 推荐相关商品，是关联查询，不是搜索
2. **不需要全文搜索**：用户不输入搜索词，是系统推荐
3. **数据量不大**：几十万级商品数据，MySQL + 索引完全够用
4. **不需要模糊匹配**：精确的分类关联关系
5. **已有 MySQL 基础设施**：无需引入新组件

这套逻辑用 MySQL 实现简单高效：

```sql
-- 获取用户最近浏览的商品分类 Top5
SELECT c.id, c.name, COUNT(*) AS view_count
FROM user_browse_history ubh
JOIN products p ON ubh.product_id = p.id
JOIN categories c ON p.category_id = c.id
WHERE ubh.user_id = ? AND ubh.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY c.id, c.name
ORDER BY view_count DESC
LIMIT 5;

-- 根据分类推荐商品
SELECT p.id, p.name, p.price, p.image_url
FROM products p
WHERE p.category_id IN (...) AND p.status = 1
ORDER BY p.sales DESC
LIMIT 20;
```

## 总结

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 固定关联查询 | MySQL | 查询模式固定，精确匹配 |
| 全文搜索 | ES | 倒排索引 + IK 分词 |
| 海量数据搜索 | ES | 分布式 + 深度分页优化 |
| 复杂聚合分析 | ES | 一次查询多聚合 |
| 相关性排序 | ES | BM25 算法 |
| 近实时搜索 | ES | 1 秒 refresh |
| 事务/关联查询 | MySQL | ACID + JOIN |

**核心原则**：根据业务场景的查询模式来选型，而不是根据数据量。查询模式简单、固定，就用 MySQL；需要全文搜索、模糊匹配、复杂排序，才考虑 ES。

> 不要为了"高性能"而盲目上 ES，引入 ES 意味着额外的基础设施维护成本和数据同步复杂性。

---

*参考资料：*
- *[Elasticsearch vs MySQL - InfluxData](https://www.influxdata.com/comparison/elasticsearch-vs-mysql)*
- *[Elasticsearch vs MySQL - Knowi](https://www.knowi.com/blog/elasticsearch-vs-mysql-what-to-choose/)*

