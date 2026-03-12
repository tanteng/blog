---
title: "Laravel 分割 routes.php 路由文件的一种方式"
date: 2016-08-02T09:20:26+08:00
draft: false
tags: ['laravel']
categories: ['tech']
description: "Laravel 分割 routes.php 路由文件的一种方式"
---

Laravel 的路由功能很强大，路由规则默认都定义在 routes.php 文件中，但是随着项目越来越大，我们需要的定义的规则越来越多，如果几百上千个路由都定义在一个文件中，如何去维护？如果不同的人都在同一个文件定义路由，这就造成了冲突，因此我们有必要将 routes.php 文件分割成多个文件，可以按照功能模块来划分。

<!--more-->

在 app/Providers/RouteServiceProvider.php 的 map 方法中可以如下定义：

```php
public function map(Router $router)
{
    $router->group(['namespace' => $this->namespace], function ($router) {
        foreach (glob(app_path('Http//Routes') . '/*.php') as $file) {
            $this->app->make('App\\Http\\Routes\\' . basename($file, '.php'))->map($router);
        }
    });
}
```

这样就可以根据功能模块分开管理路由文件了。
