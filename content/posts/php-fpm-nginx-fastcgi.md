---
title: "PHP-FPM、Nginx、FastCGI 之间的关系"
date: 2017-11-09T08:44:27+08:00
draft: false
tags: ['php', 'nginx']
categories: ['tech']
description: "PHP-FPM、Nginx、FastCGI 之间的关系"
---

本文介绍 PHP-FPM、Nginx、FastCGI 三者之间的关系，以及 Nginx 反向代理和负载均衡的配置。

<!--more-->

### PHP-FPM、Nginx、FastCGI 之间的关系

FastCGI 是一个协议，它是应用程序和 WEB 服务器连接的桥梁。Nginx 并不能直接与 PHP-FPM 通信，而是将请求通过 FastCGI 交给 PHP-FPM 处理。

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

这里 fastcgi_pass 就是把所有 php 请求转发给 php-fpm 进行处理。

### Nginx 反向代理

Nginx 反向代理最重要的指令是 proxy_pass：

```nginx
location /seckill_query/ {
    proxy_pass http://ris.filemail.gdrive:8090/;
    proxy_set_header Host ris.filemail.gdrive;
}
```

通过 location 匹配 url 路径，将其转发到另外一个服务器处理。

### Nginx 负载均衡

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

负载均衡模块用于从"upstream"指令定义的后端主机列表中选取一台主机。负载均衡的算法有多种，如轮询、ip_hash 等。

### 反向代理和负载均衡的区别

负载均衡它更多的是强调的是一种算法或策略，将请求分布到不同的机器上，因此实际上也起到了反向代理的作用。

### proxy_pass 和 fastcgi_pass 的区别

一个是反向代理模块，一个是转发给 FastCGI 后端处理。
