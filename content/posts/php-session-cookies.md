---
title: "深入理解 PHP 中 session 和 cookies 的联系"
date: 2012-09-26T01:46:43+08:00
draft: false
tags: ["php"]
categories: ["tech"]
description: "深入理解 PHP 中 session 和 cookies 的联系"
---

PHP 中的 session 和 cookies 是 Web 开发中非常重要的概念，本文将深入解析它们的工作原理和联系。

<!--more-->

## 1. Session 概念

在 Web 服务器蓬勃发展的时代，session 在 Web 开发语境下的语义是指**一类用来在客户端与服务器之间保持状态的解决方案**。

Session 解决了 HTTP 协议无状态的问题，让服务器能够"认识"同一个客户端的多次请求。

---

## 2. HTTP 协议与状态保持

HTTP 协议本身是**无状态**的——客户端只需要简单地请求下载某些文件，无论是客户端还是服务器都没有必要记录彼此过去的行为，每一次请求之间都是独立的。

然而人们很快发现：如果能够提供一些按需生成的动态信息，会让 Web 变得更加有用。就像给有线电视加上点播功能一样。

这种需求推动了技术的发展：
- **客户端**：HTML 逐步添加了表单、脚本、DOM 等客户端行为
- **服务器端**：出现了 CGI 规范响应客户端的动态请求
- **HTTP 协议**：添加了文件上传、Cookie 等特性

其中 **Cookie 的作用就是为了解决 HTTP 协议无状态的缺陷**。而 Session 机制则是另一种在客户端与服务器之间保持状态的解决方案。

---

## 3. 理解 Cookie

Cookie 分发是通过扩展 HTTP 协议来实现的，服务器通过在 HTTP 响应头中加上特殊指示，提示浏览器按照指示生成相应的 Cookie。纯粹的客户端脚本（如 JavaScript）也可以生成 Cookie。

### Cookie 的使用原则

浏览器会按照一定原则在后台自动发送 Cookie给服务器。浏览器检查所有存储的 Cookie，如果某个 Cookie 所声明的作用范围大于等于将要请求的资源所在的位置，则把该 Cookie 附在请求资源的 HTTP 请求头上发送给服务器。

### Cookie 的内容

Cookie 主要包括：**名字、值、过期时间、路径和域**。

- **域（Domain）**：可以指定某个域（如 `.google.com`），也可以指定域下的具体某台机器（如 `www.google.com`）
- **路径（Path）**：跟在域名后面的 URL 路径，如 `/` 或 `/foo`
- **域 + 路径**：合在一起构成 Cookie 的作用范围

### Cookie 的生命周期

- **会话 Cookie**：如果不设置过期时间，Cookie 的生命期为浏览器会话期间，关闭浏览器窗口后 Cookie 就消失了。会话 Cookie 一般保存在内存里，而不是硬盘上。

- **持久 Cookie**：如果设置了过期时间，浏览器会把 Cookie 保存到硬盘上，关闭后再次打开浏览器仍然有效，直到超过设定的过期时间。

### Cookie 的共享规则

存储在硬盘上的 Cookie **不能在不同浏览器间共享**，但可以在**同一浏览器的不同进程间共享**。

不同浏览器存储 Cookie 的位置：

| 浏览器 | Cookie 存储位置 |
|--------|---------------|
| Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cache` |
| Firefox | `%APPDATA%\Mozilla\Firefox\Profiles\随机目录\cookies.sqlite` |
| IE | `%APPDATA%\Microsoft\Windows\Cookies` |

---

## 4. PHP 中 Session 的生成机制

Session 的目的是保持每一个用户的各种状态，弥补 HTTP 协议的不足（无状态）。Session 保存在服务器端，那么它用什么来区别不同用户呢？

答案是 **借助 Cookie**。

当调用 `session_start()` 时，PHP 会同时在 Session 存放目录（默认为 `/tmp/`）和客户端的 Cookie 目录各生成一个文件。

### Session 文件格式

文件名称格式为 `sess_{SESSIONID}`，初始时 session 文件中没有任何内容。

```php
<?php
session_start();

$_SESSION['name'] = 'sharexie';
$_SESSION['webUrl'] = 'www.qq.com';
```

添加上述代码后，session 文件内容如下：

```
name|s:8:"sharexie";webUrl|s:10:"www.qq.com";
```

---

## 5. PHP 中 Session 的过期回收机制

随着使用时间增长，session 目录下会有许多 session 文件。PHP 提供了过期回收机制来清理这些文件。

在 `php.ini` 中，`session.gc_maxlifetime` 设置了 session 的生存时间（**默认为 1440 秒 = 24 分钟**）。

如果 session 文件的最后更新时间到当前时间超过了生存时间，这个 session 文件就被认为是过期的。在下一次 session 回收时，过期的 session 文件就会被删除。

---

## 6. PHP 中 Session 的客户端存储机制

由于 Cookie 可以被人为禁止，必须有其他机制在 Cookie 被禁止时仍然能够把 session id 传递回服务器。解决方案包括：

1. **URL 重写**：把 session id 直接附加在 URL 路径后面
2. **查询字符串**：把 session id 作为查询字符串附加在 URL 后面
3. **表单隐藏字段**：在表单中使用隐藏字段存储 session id

---

## 总结

- **Cookie**：保存在客户端，由服务器生成并通过 HTTP 头传递
- **Session**：保存在服务器端，借助 Cookie（或其他方式）来识别用户
- **关系**：Session 需要 Cookie 来保存 session id，但也可以在 Cookie 被禁用时通过 URL 重写等方式传递 session id
