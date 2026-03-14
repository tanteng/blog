---
title: "使用 Docker 搭建 Laravel 本地环境"
date: 2017-10-14T11:28:48+08:00
draft: false
tags: ['laravel', 'docker']
categories: ['tech']
description: "使用 Docker 搭建 Laravel 本地环境"
---

Laravel 官方提供 Homestead 和 Valet 作为本地开发环境，Homestead 是一个官方预封装的 Vagrant Box，也就是一个虚拟机，但是跟 Docker 比，它占用体积太大，启动速度慢，同时响应速度很慢，现在有了 Docker 这种更好的方式，可以轻松方便的搭建整套 PHP 开发环境。

<!--more-->

### 安装 Docker

首先安装 Docker。

### 克隆 Laradock

Laradock 官方文档：http://laradock.io/

Laradock github：https://github.com/laradock/laradock

Laradock 是一个包含全功能用于 Docker 的 PHP 运行环境，使用 docker-compose 方式部署。（特别说明：它不仅用于 Laravel 环境搭建，而且支持所有其他 PHP 框架，它就是一整套 PHP 的环境。）

### 部署 PHP 环境

1.克隆 laradock

```bash
git clone https://github.com/Laradock/laradock.git
```

2.创建环境变量文件

```bash
cp env-example .env
```

3.直接用 docker-compose 运行需要启用的服务，如：

```bash
docker-compose up -d nginx mysql redis beanstalkd
```

这样就启动了所需的 PHP 运行环境，php-fpm 默认会运行，所以不需要指定。

### Laravel 配置文件

Laravel 配置文件需要注意的问题是，在 .env 文件中，mysql 和 redis 的地址需填写成这样：

```ini
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=tanteng.me
DB_USERNAME=root
DB_PASSWORD=root

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379
```

### Nginx 配置

在本地通过域名方式访问站点，要将 host 中域名绑定到本地，同时还需要增加 nginx 配置。在 laradock 项目的 nginx 文件夹下的 sites 目录下添加配置文件即可。

### 执行 Composer

执行 composer 等操作，需要进入到 workspace 容器中进行，使用命令：

```bash
docker-compose exec workspace bash
```

进入到 workspace 容器，就可以进行 compose 命令等操作了。
