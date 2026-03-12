---
title: "使用 Supervisor 管理 Laravel 队列进程"
date: 2017-01-07T12:48:53+08:00
draft: false
tags: ['laravel', 'supervisor']
categories: ['tech']
description: "Supervisor 是一个 Python 写的进程管理工具，用于管理 Laravel 队列进程"
---

Supervisor 是一个 Python 写的进程管理工具，有时一个进程需要在后台运行，并且意外挂掉后能够自动重启，就需要这么一管理进程的工具。在 Laravel 开发中，也经常使用到队列监听，可以配合 Supervisor 来管理 Laravel 队列进程。

<!--more-->

### 安装 Supervisor

1. 使用 pip 工具进行安装：

```bash
sudo pip install supervisor
```

2. Ubuntu 系统使用 apt-get

```bash
sudo apt-get install supervisor
```

### 配置 Supervisor

运行这个命令可以生成一个默认的配置文件：

```bash
echo_supervisord_conf > /etc/supervisord.conf
```

生成成功后，打开编辑这个文件，把最后的 include 块的注释打开，并修改如下：

```ini
[include]
files = /etc/supervisor/*.conf
```

新增的 Supervisor 配置文件放在 /etc/supervisor 目录下，并且以 conf 结尾。

### 使用 Supervisor 管理 Laravel 队列进程

首先在 /etc/supervisor 目录下新增一个 Supervisor 的配置文件：

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /usr/share/nginx/html/tanteng.me/artisan queue:work --tries=3
autostart=true
autorestart=true
user=vagrant
numprocs=8
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-queue.log
```

启动队列进程监听：

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

### supervisorctl 命令

```bash
supervisorctl status    # 查看程序状态
supervisorctl stop usercenter    # 关闭程序
supervisorctl start usercenter   # 启动程序
supervisorctl restart usercenter # 重启程序
supervisorctl reread  # 读取有更新（增加）的配置文件
supervisorctl update  # 重启配置文件修改过的程序
```
