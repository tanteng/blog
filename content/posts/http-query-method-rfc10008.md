---
title: "HTTP 新方法 QUERY：填补 GET 和 POST 之间的空白"
date: 2026-06-18
draft: false
tags: ["HTTP", "RFC", "Web", "API", "IETF"]
categories: []
---

2026年6月，IETF 正式发布了 [RFC 10008](https://www.rfc-editor.org/info/rfc10008/)，为 HTTP 协议引入了一个全新的方法——**QUERY**。这是一种专为查询场景设计的 HTTP 方法，试图填补 GET 和 POST 之间长期存在的空白。

<!--more-->

## 现有方案的问题

在 QUERY 出现之前，HTTP 的查询方案面临一个两难困境：

**用 GET？** 查询参数只能放在 URL 中，而 URL 有长度限制（浏览器通常限制在 2KB 左右）。复杂的查询条件（如多条件搜索、嵌套筛选）很容易超出这个限制。

```
GET /search?conditions=age>25&city=beijing&tags=golang,http,api
```

**用 POST？** 虽然能传递任意大小的请求体，但 POST 本身不是**安全（safe）** 或**幂等（idempotent）** 的方法——这意味着中间代理无法判断这个请求是否会修改服务器状态，也就无法安全地重试或缓存。

```
POST /search
Content-Type: application/json

{"conditions": [{"field": "age", "op": ">", "value": 25}], ...}
```

## QUERY 方法登场

QUERY 方法的核心设计目标是：**像 POST 一样传递请求体，又像 GET 一样安全且可缓存。**

RFC 10008 将 QUERY 定义为：

> A QUERY requests that the request target process the enclosed content in a safe and idempotent manner and then respond with the result of that processing.

翻译过来就是：QUERY 请求目标资源**以安全且幂等的方式**处理请求内容，并返回处理结果。

### 一个直观的例子

使用 POST 做查询：
```
POST /search
Content-Type: application/json

{"query": "what is RFC 10008"}
```

使用 QUERY 做同样的查询：
```
QUERY /search
Content-Type: application/json

{"query": "what is RFC 10008"}
```

看起来几乎一样，但关键区别在于：**QUERY 从语义上声明这是一个查询操作，而非状态修改操作。**

## 核心特性对比

| 特性 | GET | QUERY | POST |
|------|-----|-------|------|
| 安全（不修改资源） | ✅ | ✅ | ❌ 可能不是 |
| 幂等（可安全重试） | ✅ | ✅ | ❌ 可能不是 |
| 请求体 | ❌ | ✅ | ✅ |
| 可缓存 | ✅ | ✅ | ❌ 只能缓存后续 GET/HEAD |
| URI 标识查询本身 | ✅ | 可选（Location 响应头） | ❌ |

## 关键设计细节

### 1. Content-Type 必须存在

QUERY 请求**必须**携带 `Content-Type` 请求头，且必须与请求体内容一致。服务器如果发现缺失或不匹配，必须返回 400 错误。这确保了查询语义的明确性。

### 2. 响应可缓存

由于 QUERY 是安全且幂等的，**响应可以被缓存**。这与 POST 形成鲜明对比——POST 响应默认不可缓存（除非明确声明）。

### 3. 可为查询结果分配 URI

QUERY 响应可以包含 `Location` 响应头来标识这个查询本身，也可以包含 `Content-Location` 来标识查询结果的 URI。这意味着：

- 查询可以被**加入书签**或**分享给其他人**
- 可以通过 `GET /location/uri` 再次获取相同结果
- CDN 和代理可以正常缓存查询结果

## 适用场景

QUERY 方法特别适合以下场景：

- **复杂搜索**：多条件、多字段、嵌套逻辑的搜索请求，参数不适合放在 URL 中
- **GraphQL 查询**：GraphQL 通常用 POST 发送查询，但语义上这是只读操作，QUERY 更准确
- **API 数据筛选**：需要对大量数据进行过滤、排序、分页的查询
- **代理友好**：需要在中间层缓存查询结果的场景

## 总结

RFC 10008 引入的 QUERY 方法，本质上是给 HTTP 协议补上了一块长期缺失的"拼图"——它让查询操作在语义上自洽，同时保留了 POST 的灵活性（不限大小的请求体）和 GET 的可缓存性。

这不是一个颠覆性的新协议，而是一个**语义精确化**的改进。随着 RFC 正式发布，我们可以期待看到更多 HTTP 客户端和服务器开始支持这个新方法。

## 参考

- [RFC 10008 - The HTTP QUERY Method](https://www.rfc-editor.org/info/rfc10008/)
