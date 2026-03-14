---
title: "使用 Docker 搭建 Laravel 本地开发环境"
author: tanteng
type: post
date: 2017-10-14T11:28:48+08:00
draft: false
tags: ['laravel', 'docker', '容器化']
categories: ['tech']
description: "使用 Docker 和 Laradock 搭建 Laravel 本地开发环境指南"
---

Laravel 官方提供 Homestead 和 Valet 作为本地开发环境，但 Docker 相比虚拟机占用体积更小、启动更快，是更好的选择。

<!--more-->

## 为什么选择 Docker？

- **占用资源小**：相比 Vagrant 虚拟机，Docker 容器更轻量
- **启动速度快**：秒级启动
- **环境一致**：开发环境与生产环境保持一致
- **易于管理**：使用 docker-compose 统一管理多容器

## Laradock 简介

Laradock 是 Docker 上最完整的 PHP 开发环境，包含：
- PHP-FPM
- Nginx
- MySQL / PostgreSQL
- Redis
- Elasticsearch
- Composer
- 等等

**官网**：http://laradock.io/  
**GitHub**：https://github.com/laradock/laradock

## 快速开始

### 1. 克隆 Laradock

```bash
git clone https://github.com/Laradock/laradock.git
```

### 2. 创建环境变量文件

```bash
cp env-example .env
```

### 3. 启动服务

```bash
docker-compose up -d nginx mysql redis
```

这会启动 Nginx、MySQL、Redis 等服务。

### 4. 配置 Laravel

在 `.env` 文件中配置数据库连接：

```ini
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=your_database
DB_USERNAME=root
DB_PASSWORD=root

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379
```

注意：Docker 容器间通过服务名通信，所以 Host 填写 `mysql`、`redis` 而不是 `127.0.0.1`。

## 常用命令

### 进入工作容器

```bash
docker-compose exec workspace bash
```

### 查看运行中的容器

```bash
docker-compose ps
```

### 停止服务

```bash
docker-compose stop
```

### 重新构建

```bash
docker-compose build
```

## Laravel Sail（官方新方案）

 Laravel 9+ 官方推出了 Laravel Sail，是官方推荐的 Docker 开发环境：

```bash
curl -s "https://laravel.build/example-app" | bash
cd example-app
./vendor/bin/sail up
```

Sail 基于 Docker Compose，提供一致的体验。

## Docker Compose 多容器架构

典型的 Laravel Docker 环境包含：

| 服务 | 作用 |
|------|------|
| nginx | Web 服务器 |
| php-fpm | PHP 解释器 |
| mysql | 数据库 |
| redis | 缓存 |
| workspace | 开发工具（Composer、Artisan） |

## 最佳实践

1. **开发环境与生产环境分离**：使用不同的 docker-compose.yml
2. **数据持久化**：挂载卷保存数据
3. **网络隔离**：使用自定义网络
4. **环境变量管理**：使用 .env 文件管理配置
