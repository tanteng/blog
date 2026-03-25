---
title: "Nginx SSI 使用指南"
date: 2016-04-08T12:47:34+08:00
draft: false
tags: ['nginx']
categories: ['tech']
description: "Nginx SSI 使用指南"
---

> 这是最早期的 SSR 方式之一——服务端在返回 HTML 前完成页面组装，客户端直接拿到完整页面，而非空壳再客户端渲染。

相信很多人在浏览网页时都遇到过这样的情况：本地在开发环境运行正常的页面，部署到测试环境后却发现部分内容缺失。比如导航栏、页脚、公用组件等内容本地看不到。

如果你遇到这样的情况，先检查一下页面源码中是否有类似这样的代码：

```html
<!--#include virtual="/new/ssi/script.html"-->
```

这就对了——这就是 SSI（Server Side Include）在起作用。本地没有配置 SSI，所以包含的内容没有渲染出来。

<!--more-->

SSI（Server Side Include，服务端包含）是一种基于服务端的网页制作技术，Nginx 通过 `ngx_http_ssi_module` 模块提供支持。

它的核心原理是：在页面发送给客户端之前，服务器会解析 HTML 中的 SSI 指令，并发起子请求获取指定文件的内容，然后嵌入到主页面中。这种方式特别适合处理多个页面共用的内容，比如导航栏、页脚、版权信息等。

一个典型的应用场景就是网站导航栏——导航栏在所有页面都一样，只需要维护一份 HTML 文件，通过 SSI 引用到各个页面即可，无需每个页面都重复编写。

### SSI 的工作原理

SSI 的处理流程：

1. 客户端发起 HTTP 请求
2. Nginx 服务器接收请求
3. SSI Parser 解析 HTML 中的 SSI 指令
4. **并行**发起子请求获取被包含的文件（如 Header.html、Menu.html、Footer.html 等）
5. 将获取的内容嵌入到父页面中
6. 返回完整的 HTML 给客户端

### Nginx 配置开启 SSI

在 server 块中添加以下配置即可开启 SSI 支持：

```nginx
server { 
    listen 80; 
    server_name www.example.com; 
    root /usr/local/web/wwwroot;
    
    location / { 
        # 开启 SSI
        ssi on; 
        
        # 忽略 SSI 包含文件不存在时的错误
        ssi_silent_errors on; 
        
        # 指定处理 SSI 的文件类型
        ssi_types text/shtml; 
        
        index index.shtml; 
    } 
}
```

常用 SSI 指令说明：
- `ssi on` - 开启 SSI 支持
- `ssi_silent_errors on` - 忽略 SSI 包含文件不存在时的错误
- `ssi_types text/shtml` - 指定处理 SSI 的文件类型

### 使用示例

在 HTML 页面中使用 SSI 引用公共组件：

```html
<!-- 包含导航栏 -->
<!--#include virtual="/header.html"-->

<!-- 包含侧边栏 -->
<!--#include file="/sidebar.html"-->

<!-- 包含页脚 -->
<!--#include virtual="/footer.html"-->
```

### 使用注意事项

使用 SSI 需要注意以下几点：

1. **并发请求**：SSI 的子请求是并行获取的，但每个子请求都会占用 worker 连接数，因此不要在单个页面中使用过多 SSI 指令

2. **后端压力**：每个 SSI 指令都会向后端发起一次请求，使用 SSI 会对后端服务器产生成倍的压力，需要合理评估服务器容量

3. **响应时间**：用户最终获取页面的时间 = 父请求时间 + SSI 解析时间 + 子请求时间，如果子请求过多会影响首屏加载速度

> SSI 就像一把双刃剑，在带来便利的同时也需要谨慎使用，合理控制 SSI 的使用数量才能发挥它的最大价值。
