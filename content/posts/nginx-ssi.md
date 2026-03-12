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

它的工作原因是：在页面内容发送到客户端之前，使用SSI指令将文本、图片或代码信息包含到网页中。对于在多个文件中重复出现内容，使用SSI是一种简便的方法，将内容存入一个包含文件中即可，不必将其输入所有文件。

一个典型的应用场景就是导航栏，当页面都是HTML静态页的时候，动态导航栏页面内容可以使用SSI包含。

### SSI的工作机制

SSI的运作机制：
1. 客户端发起HTTP请求
2. NGINX服务器接收请求
3. SSI Parser解析HTML中的SSI指令
4. 并行发起子请求获取Include的文件（Header.html、Menu.html、Footer.html等）
5. 将获取的内容嵌入到父页面中
6. 返回完整的HTML给客户端

### 在Nginx配置中开启SSI

只需在server块中添加以下配置：

```nginx
server { 
    listen 80; 
    server_name www.example.com; 
    location / { 
        ssi on; 
        ssi_silent_errors on; 
        ssi_types text/shtml; 
        index index.shtml; 
        root /usr/local/web/wwwroot; 
    } 
}
```

常用SSI指令说明：
- `ssi on` - 开启SSI支持
- `ssi_silent_errors on` - 忽略SSI包含文件不存在时的错误
- `ssi_types text/shtml` - 指定处理 SSI 的文件类型

### 使用示例

```html
<!--#include virtual="/header.html"-->
<!--#include virtual="/menu.html"-->
<!--#include file="/footer.html"-->
```

### 注意事项

使用SSI需要注意：

1. **性能影响**：SSI的子请求是并行获取的，但每个子请求都会占用worker连接数
2. **对后端压力**：使用SSI会对后端服务器产生成倍的压力，需要合理评估
3. **响应时间**：用户获取请求的时间 = 父请求时间 + SSI解析时间 + 子请求时间

SSI是一把双刃剑，在带来便利的同时也需要谨慎使用。
