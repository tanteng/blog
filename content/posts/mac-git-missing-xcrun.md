---
title: "Mac终端git命令提示错误:missing xcrun"
date: 2015-11-08T07:56:52+08:00
draft: false
tags: ['git', 'mac']
categories: ['tech']
description: "在 Mac 下 PhpStorm 和 sourcetree 配合使用，提示错误：can't start Git"
---

在 Mac 下 PhpStorm 和 sourcetree 配合使用，提示错误：can't start Git，点击"fix it", 填写的路径usr/bin/git是没错的，于是在终端下输入命令git，提示如下错误：

```
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
```

<!--more-->

出现这一问题的原因很可能是Mac系统升级之后产生的，解决办法是在终端输入：

```bash
xcode-select --install
```

会自动安装。

再次在终端输入git命令，发现git命令行可以正常使用了，再看PhpStorm测试调用git，也可以正常使用了。
