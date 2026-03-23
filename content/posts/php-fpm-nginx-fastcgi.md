---
title: "PHP-FPM、Nginx、FastCGI 之间的关系与配置详解"
type: post
date: 2017-11-09T08:44:27+08:00
draft: false
tags: ['php', 'nginx', 'server-architecture']
categories: ['tech']
description: "详解 PHP-FPM、Nginx、FastCGI 三者之间的关系，以及 Nginx 反向代理和负载均衡的配置"
---

本文介绍 PHP-FPM、Nginx、FastCGI 三者之间的关系，以及 Nginx 反向代理和负载均衡的配置。

<!--more-->

## 什么是 FastCGI？

FastCGI 是一个协议，它是应用程序和 WEB 服务器连接的桥梁。传统的 CGI 方式每次请求都需要启动一个新的进程，而 FastCGI 则持久化地运行，大大提高了性能。

## PHP-FPM、Nginx、FastCGI 之间的关系

Nginx 并不能直接与 PHP-FPM 通信，而是将请求通过 FastCGI 交给 PHP-FPM 处理。

**工作流程：**

```
用户请求 → Nginx → FastCGI 协议 → PHP-FPM → PHP 解释器 → 返回结果
```

Nginx 将 PHP 文件请求通过 FastCGI 协议转发给 PHP-FPM，PHP-FPM 管理着多个 PHP 进程来处理这些请求。

### Nginx 配置示例

```nginx
location ~ \.php$ {
    try_files $uri /index.php =404;
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_buffers 16 16k;
    fastcgi_buffer_size 32k;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```

这里 `fastcgi_pass` 就是把所有 PHP 请求转发给 PHP-FPM 进行处理。

## PHP-FPM 的工作原理

PHP-FPM（FastCGI Process Manager）是 PHP 的 FastCGI 实现管理器，具有以下特性：

- **进程管理**：PHP-FPM 管理着一个 PHP 进程池
- **请求处理**：当请求到来时，从进程池中选取一个可用进程处理
- **进程类型**：
  - Master Process：主进程，负责监听端口和管理工作进程
  - Worker Processes：工作进程，负责执行 PHP 脚本

### PHP-FPM 进程模式

PHP-FPM 支持三种进程管理模式：

1. **static**：固定数量的工作进程
2. **dynamic**：动态管理工作进程
3. **ondemand**：按需创建工作进程

## Nginx 反向代理

Nginx 反向代理最重要的指令是 `proxy_pass`：

```nginx
location /seckill_query/ {
    proxy_pass http://ris.filemail.gdrive:8090/;
    proxy_set_header Host ris.filemail.gdrive;
}
```

通过 location 匹配 URL 路径，将其转发到另外一个服务器处理。

**生活中的类比**：400 客服电话就像反向代理，大家拨打的都是同一个号码，但接听的是不同的客服人员。

## Nginx 负载均衡

```nginx
upstream php-upstream {
    ip_hash;
    server 192.168.0.1;
    server 192.168.0.2;
}

location / {
    root html;
    index index.html index.htm;
    proxy_pass http://php-upstream;
}
```

负载均衡模块用于从"upstream"指令定义的后端主机列表中选取一台主机处理请求。

**生活中的类比**：商场的多部电梯就是负载均衡，把人群分散到不同的电梯。

## 反向代理和负载均衡的区别

负载均衡更多强调的是一种策略，将请求分布到不同的机器上，因此也起到了反向代理的作用。二者本质上是相辅相成的。

## proxy_pass 和 fastcgi_pass 的区别

- `proxy_pass`：反向代理模块，转发 HTTP 请求到其他服务器
- `fastcgi_pass`：转发给 FastCGI 后端处理（如 PHP-FPM）
