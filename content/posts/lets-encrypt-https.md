---
title: "Let's Encrypt 开启多域名 HTTPS"
date: 2016-09-12T11:26:47+08:00
draft: false
tags: ['https', 'nginx']
categories: ['tech']
description: "Let's Encrypt 开启多域名 HTTPS"
---

上周去腾讯参加了 IMWEB 前端大会，听了关于 HTTPS 的讲座，回来把自己的主页和博客升级了一下，开启了 HTTPS。

<!--more-->

### 1.克隆 acme-tiny 项目

```bash
sudo git clone https://github.com/diafygi/acme-tiny.git 
cd acme-tiny
```

### 2.创建私钥

```bash
openssl genrsa 4096 > account.key
```

### 3.创建 CSR 文件

```bash
openssl genrsa 4096 > www.key 
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```

### 4. nginx 配置文件修改

### 5. 获取签名证书

### 6. 安装证书

### 多域名开启 HTTPS

以上仅是给主域名开启了 HTTPS，给子域名开启只需要重复以上的步骤。

### 开启 HTTPS 需要注意的问题

成功开启 HTTPS 后，网站的所有图片，css，js，都必须替换成 https 协议，如果使用的是 CDN，需要到CDN后台开启 https 域名。
