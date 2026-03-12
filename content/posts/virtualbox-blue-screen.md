---
title: "VirtualBox 启动蓝屏问题"
date: 2017-03-24T11:18:09+08:00
draft: false
tags: ['virtualbox']
categories: ['tech']
description: "VirtualBox 启动蓝屏问题解决方法"
---

用 vagrant + VirtualBox 虚拟机，最近几次没有关闭虚拟机重启电脑，导致虚拟机无法启动，每次到 Booting VM 这个步骤就会蓝屏，重新安装 vagrant，VirtualBox 软件都无法正常运行。

<!--more-->

解决方法：

在"启用或关闭Windows功能"中把 Hyper-V 特性关闭并重启电脑，再重新安装虚拟机可以正常启动了。

设备：Win 10
