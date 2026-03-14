---
title: "HTTP 204 状态码"
date: 2018-10-19T06:15:51+08:00
draft: false
tags: ['http']
categories: ['tech']
description: "HTTP 204 No Content 状态码说明"
---

HTTP 协议中 204 No Content 成功状态响应码表示目前请求成功，但客户端不需要更新其现有页面。204 响应默认是可以被缓存的，在响应中需要包含头信息 ETag。

<!--more-->

### 使用场景

在 PUT 请求中进行资源更新时，如果不需要改变当前展示给用户的页面，那么返回 204 No Content。

- 如果新创建了资源，返回 201 Created
- 如果页面需要更新以反映更新后的资源，返回 200

### 特点

- 响应不能包含消息体
- 客户端不需要刷新页面或跳转
- 可以被缓存
