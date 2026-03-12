---
title: "PHP 7 安装和开启 OPcache"
date: 2016-06-02T16:20:20+08:00
draft: false
tags: ['php', 'opcache', '性能优化']
categories: ['tech']
description: "PHP 7 安装和开启 OPcache 性能优化"
---

鸟哥在博客中说，提高PHP 7性能的几个tips，第一条就是开启OPcache：

> 记得启用Zend OPcache, 因为PHP7即使不启用Opcache速度也比PHP-5.6启用了Opcache快

<!--more-->

### 什么是 OPcache？

OPcache 通过将 PHP 脚本预编译的字节码存储在共享内存中来减少每次请求的解析和编译时间，从而提升 PHP 性能。

简单来说：第一次访问 PHP 页面时，PHP 会解析并编译脚本；启用 OPcache 后，编译好的字节码会被缓存，后续请求直接执行缓存的字节码，无需再次解析和编译。

### 安装 OPcache

#### 检查是否已安装

```bash
php -m | grep -i opcache
```

#### 安装 OPcache（CentOS/RHEL）

```bash
sudo dnf install -y php-opcache
# 或
sudo yum install -y php-opcache
```

#### 重启 PHP-FPM

```bash
sudo systemctl restart php-fpm
```

### 推荐配置

生产环境的 OPcache 配置：

```ini
[opcache]
; 启用 OPcache
opcache.enable=1

; 内存分配（MB），根据应用大小调整
opcache.memory_consumption=256

; 内部字符串缓冲区大小（MB）
opcache.interned_strings_buffer=16

; 最大缓存文件数，应大于应用中的PHP文件数
opcache.max_accelerated_files=20000

; 检查文件变更的频率（秒），生产环境设为0
opcache.revalidate_freq=0

; 生产环境应关闭时间戳验证
opcache.validate_timestamps=0

; 启用大页内存，提升性能
opcache.huge_code_pages=1

; 启用 JIT 编译（PHP 8.0+）
opcache.jit=1255
opcache.jit_buffer_size=128M
```

### 验证 OPcache 是否启用

```bash
php -i | grep -A 20 "opcache"
# 或
php -r "var_dump(opcache_get_status());"
```

### 部署后重置缓存

由于生产环境禁用了时间戳验证，部署新代码后需要重置 OPcache：

```bash
# 重启 PHP-FPM
sudo systemctl reload php-fpm

# 或通过 PHP 函数重置
php -r "opcache_reset();"
```

### 性能提升

正确配置 OPcache 后，可以：
- 减少 50-70% 的页面加载时间
- 显著降低 CPU 使用率
- 减少服务器负载
