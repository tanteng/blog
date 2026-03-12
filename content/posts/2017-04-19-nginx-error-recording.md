---
title: Nginx 报错和解决方法记录
author: tanteng
type: post
date: 2017-04-19T07:49:24+00:00
url: /2017/04/nginx-error-recording/
categories:
 - Develop
 - PHP

---
记录一下遇到的各种 Nginx 的报错和解决办法。

<!--more-->

### 13: Permission denied

Nginx错误：

2017/04/19 14:46:46 [crit] 4172#0: *671 open() "/data/vhosts/xunlei.com/test/" failed (13: Permission denied), client: 192.168.35.54, server: www.test.com, request: "GET / HTTP/1.1", host: "www.test.com"

经查权限问题导致，网站目录是 root 用户组，而 nginx 是运行的 nobody 用户进程，修改网站目录为 nobody 用户组。

### 111: Connection refused

Nginx错误：

2017/04/19 14:48:02 [error] 4172#0: *672 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.35.54, server: www.test.com, request: "GET / HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "www.test.com"

**出现的原因：**

1.php-fpm 服务没有开启

2.权限问题，查看目录用户组

### *1111335 upstream prematurely closed connection

Nginx错误：

2017/04/28 17:34:48 [error] 11058#0: *1111335 upstream prematurely closed connection while reading response header from upstream, client: 192.168.35.54, server: www.test.com, request: "GET /test/TestSADD HTTP/1.1", upstream: "http://127.0.0.1:8080/test/TestSADD", host: "www.test.com"

出现错误的背景是使用 golang 的一个项目在 logs 目录下 debug.log 写日志，报 502 错误。

在业务代码打印错误日志：open logs/debug.log: no such file or directory

这个就归类于代码问题导致 nginx 挂掉，解决办法：修改业务代码。

### 110: Connection timed out

18244#0: *7617 upstream timed out (110: Connection timed out) while reading response header from upstream, client: 192.168.35.54, server: eagleye.vip.xx.com, request: "POST /rules HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "eagleye.vip.xx.com", referrer: "http://eagleye.vip.xx.com/rules/create?type=2"

错误背景：循环分批写入 uids （用户id）到 Redis 集合，文件过大超时。
