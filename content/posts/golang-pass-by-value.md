---
title: "理解 Golang 的参数传递都是值传递"
date: 2017-11-19T12:22:01+08:00
draft: false
tags: ['go']
categories: ['tech']
description: "理解 Golang 的参数传递都是值传递"
---

在 Golang 中函数之间传递变量时总是以值的方式传递的，无论是 int, string, bool, array 这样的内置类型，还是 slice, channel, map 这样的引用类型，在函数间传递变量时，都是以值的方式传递。

<!--more-->

### 内置类型参数传递

内置类型传递的时候是值的副本：

```go
package main

import (
    "fmt"
)

func main() {
    num := 10
    num2 := increase(num, 10)
    fmt.Println(num2)
}

func increase(num int, add int) int {
    return num + add
}
```

这里 num 传入 increase 函数，是拷贝值的副本。

### 引用类型的参数传递

引用类型的参数传递也是值的拷贝：

```go
package main

import (
    "fmt"
)

func main() {
    slice1 := []string{"zhang", "san"}
    modify(slice1)
    fmt.Println(slice1)
}

func modify(data []string) {
    data = nil
}
```

运行结果：`[zhang san]`

这个例子证明了作为引用类型的切片，参数传递不是传的引用，而是传的值。如果是传的引用，那么函数对它的修改会受到影响。

### 什么是标头？

Go 语言里的引用类型有：切片、映射、通道、接口和函数类型。**当声明上述类型的变量时，创建的变量被称作标头（header）值。**每个引用类型创建的标头值是包含一个指向底层数据结构的指针。

因为标头值是为复制而设计的，所以永远不需要共享一个引用类型的值。标头值里包含一个指针，因此通过复制来传递一个引用类型的值的副本，本质上就是在共享底层数据结构。

**总而言之，引用类型在函数传递的时候，是值传递，只不过这里的"值"指的是标头值的拷贝。**
