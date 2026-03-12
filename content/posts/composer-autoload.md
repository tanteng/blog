---
title: "Composer的自动加载机制"
date: 2015-12-24T15:41:16+08:00
draft: false
tags: ['php', 'composer']
categories: ['tech']
description: "Composer的自动加载机制介绍"
---

如项目下的composer.json文件中有autoload的定义：

```json
"autoload": {
    "classmap": [
        "database"
    ],
    "psr-4": {
        "GrahamCampbell\\BootstrapCMS\\": "app/"
    }
}
```

这样定义如何实现自动加载呢？这里classmap和psr-4的区别又是什么？

<!--more-->

了解PHP依赖包管理工具Composer的自动加载机制，链接：[查看原文](http://www.tuicool.com/articles/mARrMj6)
