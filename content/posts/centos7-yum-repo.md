---
title: "常用的CentOS 7 yum源集合"
date: 2016-03-31T09:40:55+08:00
draft: false
tags: ['centos', 'linux']
categories: ['tech']
description: "常用的CentOS 7 yum源集合"
---

记录几个常用的CentOS 7下的yum源，包括PHP7，MariaDB，Redis，Nginx等，以及阿里云源，方便虚拟机或云主机上安装这些软件。

<!--more-->

### 1.PHP7 remi源

```bash
sudo rpm --import http://rpms.famillecollet.com/RPM-GPG-KEY-remi
sudo rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

### 2.MariaDB 10.1

```bash
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck = 1
```

### 3.Redis

```bash
wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
```

### 4.Nginx 1.8

```nginx
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

### 5.阿里云

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
