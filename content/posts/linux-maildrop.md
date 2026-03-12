---
title: "Linux 中 maildrop 占用空间很大的原因"
date: 2017-01-14T05:52:21+08:00
draft: false
tags: ['linux']
categories: ['tech']
description: "Linux 中 /var/spool/postfix/maildrop 占用空间很大的原因和处理方法"
---

MySQL 报错 Exception Message: SQLSTATE[08004] Too many connections，经查这次错误是硬盘空间满了导致的，于是找一些可以删除的文件腾出一些空间。

<!--more-->

### 空间占用大的原因

发现 /var/spool/postfix/maildrop 这个目录占用了 6G 多的空间，原因是：

由于 Linux 在执行 cron 时，会将 cron 执行脚本中的 output 和 warning 信息，都会以邮件的形式发送 cron 所有者，而由于客户环境中的 sendmail 和 postfix 没有正常运行，导致邮件发送不成功，全部小文件堆积在了 maildrop 目录下面，而且没有自动清理转换的机制。

### 删除方法

执行 rm -rf ./* 提示参数列表过长，后来使用如下命令删除：

```bash
ls | xargs rm -f
```

### 脚本重定向输出

在 crontab 脚本中，将输出重定向到日志或 /dev/null 2>&1，避免产生大量不必要的文件。

### 常用命令

```bash
# 查找大文件
find . -type f -size +1000000k

# 显示前十个占用空间最大的文件或目录
du -s * | sort -nr | head

# 遍历目录大小
du -sh *

# 系统各挂载硬盘空间大小
df -hl
```
