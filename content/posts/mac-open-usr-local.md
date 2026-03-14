---
title: "Mac 上打开 /usr/local 路径的文件夹"
date: 2017-11-29T14:56:23+08:00
draft: false
tags: ['mac']
categories: ['tech']
description: "Mac 上打开 /usr/local 路径的文件夹"
---

在 Mac 的 Sublime 或者 Visual Studio Code 中选择打开文件夹，无论如何也无法选择 /usr/local 路径，在 Finder 中还可以使用"前往文件夹"输入路径，不过有解决办法。

<!--more-->

在终端使用命令行的方式启动软件打开指定文件夹，如：

```bash
# 打开 golang 标准库的 src 文件夹
open -a "Visual Studio Code" /usr/local/go/src/
```

或者：

```bash
open -a Sublime\ Text /usr/local/go/src/
```

这样就可以成功运行软件并打开指定的文件夹了。
