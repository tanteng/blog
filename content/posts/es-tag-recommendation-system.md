---
title: "基于 Elasticsearch 标签的推荐系统设计"
date: 2024-12-14
tags: ["elasticsearch", "recommendation", "tag-matching", "user-profile"]
categories: ["tech"]
draft: false
---

## 背景

在内容平台中，如何让用户快速找到感兴趣的知识是一个核心问题。基于标签的推荐是一种简单而有效的方案——通过分析用户的兴趣标签，与知识内容的标签进行匹配，实现个性化的内容推荐。Elasticsearch 凭借强大的倒排索引和灵活的查询能力，成为构建标签推荐系统的理想选择。本文将介绍一种基于 Elasticsearch 标签的推荐系统设计方案，涵盖标签匹配、用户画像构建、以及推荐算法的实现。

<!--more-->

### 1.1 字段类型选择

```json
{
  "tags": {
    "type": "keyword"  // 不分词，整体存储
  },
  "tag_ids": {
    "type": "integer"
  }
}
```

**为什么选择 keyword 类型？**

- 标签是精确值，不需要分词
- `term` 查询比 `match` 更高效
- 避免 "MySQL" 被分成 "mysql" 导致匹配失败

### 1.2 查询方式

**单个标签精确匹配：**
```json
{
  "term": {
    "tags": "MySQL"
  }
}
```

**多个标签（任选其一）：**
```json
{
  "terms": {
    "tags": ["MySQL", "Python", "Redis"]
  }
}
```

**多个标签（全部匹配）：**
```json
{
  "bool": {
    "must": [
      { "term": { "tags": "MySQL" }},
      { "term": { "tags": "优化" }}
    ]
  }
}
```

**带权重的推荐排序：**
```json
{
  "function_score": {
    "query": { "match_all": {} },
    "functions": [
      { "filter": { "term": { "tags": "Python" }}, "weight": 0.8 },
      { "filter": { "term": { "tags": "MySQL" }}, "weight": 0.6 }
    ],
    "score_mode": "sum"
  }
}
```

## 二、数据同步方案

### 2.1 MySQL 到 ES 的数据同步

MySQL 中，文章和标签是多对多关系：
- `articles` 表：文章内容
- `tags` 表：标签定义
- `article_tags` 表：关联表

**同步策略：**

1. **定时任务同步（简单场景）**
```python
# 每分钟同步一次
articles = db.query("""
    SELECT a.*, GROUP_CONCAT(t.name) as tag_names
    FROM articles a
    LEFT JOIN article_tags at ON a.id = at.article_id
    LEFT JOIN tags t ON at.tag_id = t.id
    GROUP BY a.id
""")

for a in articles:
    es.index(index="articles", id=a.id, body={
        "title": a.title,
        "tags": a.tag_names.split(','),
        "created_at": a.created_at
    })
```

2. **Binlog 同步（生产环境）**
- 使用 Canal / Debezium 监听 MySQL binlog
- 实时推送变更到 ES
- 适合高并发写入场景

### 2.2 推荐：冗余存储

```json
{
  "article_id": 456,
  "title": "MySQL 主从复制",
  "tags": ["MySQL", "数据库", "架构"],  // 冗余存名称，查询方便
  "tag_ids": [1, 3, 5]                  // 业务关联用
}
```

## 三、用户画像构建

### 3.1 标签来源

用户标签有两种获取方式：

**1. 显式标签（用户主动提供）**

注册或资料页面让用户选择感兴趣的领域：
```json
{
  "user_id": 123,
  "tags": ["Python", "MySQL", "架构"],
  "source": "self-selected"
}
```

**2. 隐式标签（自动分析行为）**

根据用户行为自动计算标签权重：

| 行为 | 权重 |
|------|------|
| 浏览文章 | +0.1 |
| 点赞/收藏 | +0.3 |
| 分享 | +0.5 |
| 评论 | +0.3 |
| 搜索关键词 | +0.4 |

**计算逻辑：**
```python
# 每天定时计算
user_behaviors = redis.zrange(f"user:views:{user_id}", 0, -1)

tag_weights = {}
for tag in user_behaviors:
    tag_weights[tag] = tag_weights.get(tag, 0) + 0.1

# 只保留权重 > 0.3 的标签
final_tags = [k for k, v in tag_weights.items() if v > 0.3]
```

**ES 用户画像文档：**
```json
{
  "user_id": 123,
  "tags": ["Python", "MySQL"],
  "tag_weights": {"Python": 0.8, "MySQL": 0.5},
  "last_updated": "2026-03-05"
}
```

### 3.2 冷启动问题

新用户没有行为数据时：
- **默认标签**：让用户选择 3-5 个兴趣领域
- **热门标签**：推荐热门知识对应的标签
- **社交推荐**：参考好友关注的领域

### 3.3 技术架构

```
用户行为 → Kafka → Flink 实时计算 → ES 用户画像
              ↓
         定时任务（每天聚合）→ ES
```

## 四、推荐系统实现

### 4.1 推荐流程

```
用户画像（tags）→ 标签匹配 → 知识（tags）→ 推荐
```

### 4.2 查询示例

根据用户标签推荐文章：
```json
{
  "function_score": {
    "query": {
      "bool": {
        "should": [
          { "term": { "tags": user.tags[0] }},
          { "term": { "tags": user.tags[1] }},
          { "term": { "tags": user.tags[2] }}
        ]
      }
    },
    "functions": [
      { 
        "filter": { "term": { "tags": "Python" }}, 
        "weight": 0.8 
      },
      { 
        "filter": { "term": { "tags": "MySQL" }}, 
        "weight": 0.6 
      }
    ],
    "score_mode": "sum",
    "boost_mode": "multiply"
  }
}
```

### 4.3 进阶优化

1. **混合推荐**：结合协同过滤 + 内容标签
2. **时间衰减**：近期行为权重更高
3. **多样性**：避免推荐内容过于单一

## 五、总结

基于 Elasticsearch 标签的推荐系统，关键点在于：

1. **标签字段设计**：使用 keyword 类型，避免分词问题
2. **数据同步**：根据业务规模选择定时任务或 Binlog 同步
3. **用户画像**：显式 + 隐式标签结合，解决冷启动
4. **推荐匹配**：使用 function_score 实现加权推荐

这个方案实现简单、性能优秀，适合大多数知识推荐场景。

---

*有问题欢迎评论区交流~*
