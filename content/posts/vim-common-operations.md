---
title: "vim 常用操作"
date: 2016-09-21T14:06:02+08:00
draft: false
tags: ['vim', 'linux']
categories: ['tech']
description: "vim 常用操作方法"
---

vim 在 Linux 下使用很多，但是习惯了在 Windows 下的文本操作，在 vim 中进行文本操作会觉得很不方便，但是 vim 是一个很强大的工具，下面是一些常用的 vim 文本操作方法。

<!--more-->

### vim 撤销和恢复操作

在不可编辑模式下，使用 u 即可撤销上一次操作，使用 Ctrl+r 恢复上一次操作。

### vim 区块选择和复制粘贴

vim 进入某个文件，按 v，进入 VISUAL 模式，使用 h,j,k,l 或者方向键移动光标即可选中内容，按 y 完成复制，在需要粘贴的地方按 p 完成粘贴。

### vim 移动到开头和末尾

gg:命令将光标移动到文档开头
G:命令将光标移动到文档末尾

0 移到一行开头
$ 移到一行末尾

### vim 翻页

Ctrl+f 往前滚动一整屏
Ctrl+b 往后滚动一整屏
Ctrl+d 往前滚动半屏
Ctrl+u 往后滚动半屏

### vim 删除一行或多行

dd 删除一行
ndd 删除以当前行开始的n行
