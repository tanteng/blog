---
title: "Golang 值传递与引用传递深度解析"
type: post
date: 2017-11-19T12:22:01+08:00
draft: false
tags: ['golang', 'go', 'performance-optimization']
categories: ['tech']
description: "深入理解 Golang 的参数传递机制：都是值传递，包括切片、映射、通道等引用类型"
---

在 Golang 中，函数之间传递变量时总是以值的方式传递的。无论是 int、string、bool、array 这样的内置类型，还是 slice、map、channel 这样的引用类型，在函数间传递变量时，都是以值的方式传递。

<!--more-->

## 值传递的基本概念

Go 语言中，**所有参数传递都是值传递**。这意味着当你把变量作为参数传递给函数时，实际上传递的是该变量的一个副本。

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
    fmt.Println(num)    // 仍然是 10
    fmt.Println(num2)   // 20
}

func increase(num int, add int) int {
    return num + add
}
```

这里 num 传入 increase 函数，是拷贝值的副本，函数内的修改不会影响原变量。

## 引用类型也是值传递

### 引用类型的"陷阱"

很多人认为切片、映射、通道等引用类型在函数间传递时是引用传递，但实际上它们也是值传递：

```go
package main

import (
    "fmt"
)

func main() {
    slice1 := []string{"zhang", "san"}
    modify(slice1)
    fmt.Println(slice1)  // 仍然是 [zhang san]
}

func modify(data []string) {
    data = nil  // 这只是修改了副本
}
```

运行结果：`[zhang san]`

这个例子证明了作为引用类型的切片，参数传递不是传的引用，而是传的值。如果是传的引用，那么函数对它的修改应该会受到影响。

## 什么是标头（Header）？

Go 语言里的引用类型包括：**切片、映射、通道、接口和函数类型**。

当声明上述类型的变量时，创建的变量被称作**标头（Header）值**。每个引用类型创建的标头值包含一个指向底层数据结构的指针。

因为标头值是为复制而设计的，所以永远不需要共享一个引用类型的值。标头值里包含一个指针，因此通过复制来传递一个引用类型的值的副本，本质上就是在共享底层数据结构。

**总结：引用类型在函数传递的时候，是值传递，只不过这里的"值"指的是标头值的拷贝。**

## 何时使用指针？

如果你希望函数能够修改传入变量的值，应该使用指针：

```go
package main

import (
    "fmt"
)

func main() {
    num := 10
    modify(&num)  // 传递指针
    fmt.Println(num)  // 输出 20
}

func modify(p *int) {
    *p = 20  // 通过指针修改原值
}
```

## 最佳实践

除非你有特定的需求需要让函数修改原值，否则**默认使用值传递**。这会让代码更清晰、可预测性更强。

- 使用指针的场景：
  - 需要在函数内修改传入变量的值
  - 传递大型结构体（避免复制开销）
  - 需要共享可变状态

- 使用值传递的场景：
  - 不需要修改原值
  - 传递小型数据（复制成本更低）
  - 确保数据的不可变性
