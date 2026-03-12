---
title: "Nginx性能调优之buffer参数设置"
date: 2016-03-27T10:07:51+08:00
draft: false
tags: ['nginx']
categories: ['tech']
description: "Nginx性能调优之buffer参数设置"
---

打开Nginx的error.log日志文件，发现很多warn的警告错误，提示：

①an upstream response is buffered to a temporary file

<!--more-->

以及这样的警告：

②a client request body is buffered to a temporary file

这个需要设置增加client_body_buffer_size的大小。缓冲区设置小了，Nginx会把内容写到硬盘，这样会影响性能。于是在nginx.conf中增加如下fastcgi buffers参数设置：

```nginx
location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_connect_timeout 60;
    fastcgi_send_timeout 180;
    fastcgi_read_timeout 180;
    fastcgi_buffer_size 128k;
    fastcgi_buffers 256 16k;
    client_body_buffer_size 1024k;
    include fastcgi_params;
}
```

### Nginx 的 buffer 机制

对于来自 FastCGI Server 的 Response，Nginx 将其缓冲到内存中，然后依次发送到客户端浏览器。缓冲区的大小由 fastcgi_buffers 和 fastcgi_buffer_size 两个参数控制。

### Buffer Size 优化

buffer的大小是最需要调优的参数。如果buffer size太小就会导致nginx使用临时文件存储response，这会引起磁盘读写IO，流量越大问题越明显。
