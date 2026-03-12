---
title: "Nginx 开启 SSI"
date: 2016-04-08T12:47:34+08:00
draft: false
tags: ['nginx']
categories: ['tech']
description: "Nginx 开启 SSI"
---

在一个HTML页面中发现一段这样的代码：

`<!--#include virtual="/new/ssi/script.html"-->`

在本地环境打开网页总觉得缺少什么，和测试服务器网页对比，发现确实少了很多内容，原来include virtual是包含另一个页面的意思。

<!--more-->

### 什么是SSI？

SSI：Server Side Include，是一种基于服务端的网页制作技术，大多数web服务器如Nginx均支持SSI命令。

它的工作原因是：在页面内容发送到客户端之前，使用SSI指令将文本、图片或代码信息包含到网页中。对于在多个文件中重复出现内容，使用SSI是一种简便的方法。

一个典型的应用场景就是登陆导航栏，当页面都是html静态页的时候，动态导航栏页面内容可以使用SSI包含。

### 在Nginx配置中开启SSI

只需在server块中添加一行代码就可以开启SSI：

```nginx
server { 
    listen 80; 
    server_name www.tanteng.me; 
    location / { 
        ssi on; 
        ssi_silent_errors on; 
        ssi_types text/shtml; 
        index index.shtml; 
        root /usr/local/web/wwwroot; 
    } 
}
```
