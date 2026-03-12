---
title: "Nginx 基于 User-Agent 实现多环境测试"
date: 2016-12-17T06:19:44+08:00
draft: false
tags: ['nginx']
categories: ['tech']
description: "使用 Nginx 基于 User-Agent 实现多环境测试"
---

在团队开发中，经常会遇到多个需求同时需要测试的情况。假设只有一个测试服务器，如何让多个开发人员同时测试不同的 git 分支？

一个解决方案是：**基于 User-Agent 进行分流**。

<!--more-->

### 实现原理

每个浏览器发起请求时，请求头中都会携带 User-Agent 信息，如：

```
Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:50.0) Gecko/20100101 Firefox/50.0
```

通过浏览器插件（如 User Agent Switcher），可以自定义 UA 内容。Nginx 可以根据不同的 UA，将请求转发到不同的后端服务。

### Nginx 配置

首先，定义多个 upstream：

```nginx
upstream backend_dev1 {
    server 127.0.0.1:8901;
}

upstream backend_dev2 {
    server 127.0.0.1:8902;
}

upstream backend_dev3 {
    server 127.0.0.1:8903;
}
```

然后在 server 块中根据 User-Agent 进行判断和转发：

```nginx
server {
    listen 80;
    server_name test.example.com;
    
    location / {
        # 默认转发
        set $backend "default";
        
        # 根据 UA 设置不同的后端
        if ($http_user_agent ~* "dev1") {
            set $backend "backend_dev1";
        }
        if ($http_user_agent ~* "dev2") {
            set $backend "backend_dev2";
        }
        if ($http_user_agent ~* "dev3") {
            set $backend "backend_dev3";
        }
        
        proxy_pass http://$backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 使用步骤

1. **配置后端服务**：在服务器上启动多个端口，每个端口对应一个网站目录（不同的 git 分支）

2. **安装浏览器插件**：安装 User Agent Switcher 之类的浏览器插件

3. **自定义 UA**：在插件中设置自定义 UA，如 `dev1`、`dev2`、`dev3`

4. **测试访问**：使用不同的 UA 访问同一域名，即可看到不同的测试环境

### 注意事项

1. 每个后端服务需要单独配置 Nginx 虚拟主机，监听不同的端口
2. 建议在响应头中添加 X-Debug-Info，方便区分当前访问的是哪个环境
3. 这种方式适用于开发测试环境，生产环境不建议使用
