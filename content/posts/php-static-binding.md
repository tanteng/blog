---
title: "PHP Static延迟静态绑定"
date: 2015-05-27T03:38:42+08:00
draft: false
tags: ['php']
categories: ['tech']
description: "PHP5.3以后引入了延迟静态绑定static，它是为了解决什么问题呢？"
---

PHP5.3以后引入了延迟静态绑定static，它是为了解决什么问题呢？php的继承模型中有一个存在已久的问题，那就是在父类中引用扩展类的最终状态比较困难。来看一个例子。

<!--more-->

```php
class A 
{ 
    public static function echoClass(){ 
        echo __CLASS__; 
    }
    
    public static function test(){ 
        self::echoClass(); 
    }
}
 
class B extends A 
{ 
    public static function echoClass() 
    { 
        echo __CLASS__; 
    } 
} 
 
B::test(); //输出A
```

在PHP5.3中加入了一个新特性：延迟静态绑定，就是把本来在定义阶段固定下来的表达式或变量，改在执行阶段才决定。

下面的例子解决了上面提出的问题：

```php
class A 
{ 
    public static function echoClass(){ 
        echo __CLASS__; 
    } 
    
    public static function test() 
    { 
        static::echoClass(); 
    } 
} 
 
class B extends A 
{ 
    public static function echoClass(){ 
        echo __CLASS__; 
    } 
} 
 
B::test(); //输出B
```

第9行定义了一个静态延迟绑定方法，直到B调用test的时候才执行原本定义的时候执行的方法。
