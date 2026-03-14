---
title: "Golang 使用接口实现类似泛型的排序功能"
author: tanteng
type: post
date: 2017-08-11T10:14:10+00:00
url: /2017/08/golang-generic-interface/
categories:
 - Develop
 - Go

---
Golang 不支持 Java 那样的泛型，但没有泛型也可以实现类似功能，而且代码更简单直接。

<!--more-->

## 冒泡排序问题

首先看一个冒泡排序的例子，针对整型数组切片排序：

```go
package main

import "fmt"

func bubbleSort(array []int) {
    for i := 0; i < len(array); i++ {
        for j := 0; j < len(array)-i-1; j++ {
            if array[j] > array[j+1] {
                array[j], array[j+1] = array[j+1], array[j]
            }
        }
    }
}

func main() {
    a1 := []int{3, 2, 6, 10, 7, 4, 6, 5}
    bubbleSort(a1)
    fmt.Println(a1)
}
```

输出：`[2 3 4 5 6 6 7 10]`

## 如何支持多种类型？

如果希望 bubbleSort 同时支持 float 类型或按字符串长度排序，可以定义一个接口：

```go
type Sortable interface {
    Len() int
    Less(int, int) bool
    Swap(int, int)
}
```

任何实现了这个接口的类型，都可以用 bubbleSort 排序。

## 完整示例

```go
package main

import "fmt"

type Sortable interface {
    Len() int
    Less(int, int) bool
    Swap(int, int)
}

func bubbleSort(array Sortable) {
    for i := 0; i < array.Len(); i++ {
        for j := 0; j < array.Len()-i-1; j++ {
            if array.Less(j+1, j) {
                array.Swap(j, j+1)
            }
        }
    }
}

// 整型切片
type IntArr []int

func (array IntArr) Len() int {
    return len(array)
}

func (array IntArr) Less(i int, j int) bool {
    return array[i] < array[j]
}

func (array IntArr) Swap(i int, j int) {
    array[i], array[j] = array[j], array[i]
}

// 字符串切片（按长度排序）
type StringArr []string

func (array StringArr) Len() int {
    return len(array)
}

func (array StringArr) Less(i int, j int) bool {
    return len(array[i]) < len(array[j])
}

func (array StringArr) Swap(i int, j int) {
    array[i], array[j] = array[j], array[i]
}

func main() {
    // 整型排序
    intArray := IntArr{3, 4, 2, 6, 10, 1}
    bubbleSort(intArray)
    fmt.Println(intArray)  // [1 2 3 4 6 10]

    // 字符串按长度排序
    stringArray := StringArr{"hello", "i", "am", "go", "lang"}
    bubbleSort(stringArray)
    fmt.Println(stringArray)  // [i am go lang hello]
}
```

## 总结

这种通过接口实现"泛型"的方法：
- 不是真正的泛型
- 但提供了一种多类型一致性处理的方案
- 代码清晰、易懂

Go 1.18+ 已支持泛型（Generics），可以更优雅地实现类似功能。
