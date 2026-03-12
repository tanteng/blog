---
title: "MySQL数据库性能优化之缓存参数设置"
date: 2016-03-28T04:02:15+08:00
draft: false
tags: ['mysql']
categories: ['tech']
description: "MySQL数据库性能优化之缓存参数设置"
---

网站运行在阿里云上，1G内存，PHP7+PHP-FPM+Nginx+MariaDB+Redis都安装在一台服务器上，而网站访问量一天也有500IP，不多，但也造成了一点压力，刚放上去几天数据库经常会挂掉，于是查阅数据库方面的性能优化，需要设置一些参数。

<!--more-->

无论是对于哪一种数据库来说，缓存技术都是提高数据库性能的关键技术，物理磁盘的访问速度永远都会与内存的访问速度永远都不是一个数量级的。通过缓存技术无论是在读还是写方面都可以大大提高数据库整体性能。

### innodb_buffer_pool_size参数

Innodb存储引擎的缓存机制和MyISAM的最大区别就在于Innodb不仅仅缓存索引，同时还会缓存实际的数据。所以，完全相同的数据库，使用Innodb存储引擎可以使用更多的内存来缓存数据库相关的信息，当然前提是要有足够的物理内存。

innodb_buffer_pool_size参数用来设置Innodb最主要的Buffer的大小，也就是缓存用户表及索引数据的最主要缓存空间，对Innodb整体性能影响也最大。

### max_allowed_packet参数

如果一条sql语句很长，或者包含BLOB数据，会造成内存溢出。max_allowed_packet 参数的作用是，用来控制其通信缓冲区的最大长度。
