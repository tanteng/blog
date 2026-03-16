---
title: "Uber Go 开发规范"
date: 2019-09-03T08:00:00+08:00
draft: false
tags: ['go']
categories: ['tech']
description: "Uber Go 开发规范解读"
---

Uber 官方发布的 Go 语言编码规范，是 Go 开发者的重要参考指南。

<!--more-->

## 指南原则

### 接口指针

**几乎不需要使用指向接口的指针**。应该直接传递接口值——底层数据仍然是指针。

接口包含两个字段：
1. 指向类型信息的指针
2. 数据指针

### 验证接口实现

在编译时验证接口实现是否正确：

```go
type Handler struct {
  // ...
}

var _ http.Handler = (*Handler)(nil)

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  // ...
}
```

这样如果 Handler 不再实现 http.Handler 接口，编译会报错。

### 零值 Mutex 有效

`sync.Mutex` 和 `sync.RWMutex` 的零值是有效的，**不需要使用指针**：

```go
// 错误
mu := new(sync.Mutex)
mu.Lock()

// 正确
var mu sync.Mutex
mu.Lock()
```

**不要在结构体中嵌入 mutex**，即使是未导出的结构体：

```go
// 错误：Mutex 方法会暴露为 SMap 的导出 API
type SMap struct {
  sync.Mutex
  data map[string]string
}

// 正确：Mutex 是实现细节
type SMap struct {
  mu   sync.Mutex
  data map[string]string
}
```

### 边界处复制 Slice 和 Map

Slice 和 Map 包含指向底层数据的指针，需要注意复制场景：

**接收时：**
```go
// 错误：用户可能修改原始 slice
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips
}

// 正确：复制一份
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}
```

**返回时：**
```go
// 错误：返回内部 map 的引用
func (s *Stats) Snapshot() map[string]int {
  return s.counters
}

// 正确：返回副本
func (s *Stats) Snapshot() map[string]int {
  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}
```

### 使用 Defer 清理资源

使用 defer 清理资源（如文件和锁），代码更易读：

```go
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}
p.count++
return p.count
```

Defer 性能开销极小，可读性提升远超那点开销。

### Channel 大小为 1 或无缓冲

Channel 通常应该是带缓冲的大小为 1，或者无缓冲：

```go
// 错误
c := make(chan int, 64)

// 正确
c := make(chan int, 1)  // 或
c := make(chan int)     // 无缓冲
```

### 枚举从 1 开始

使用 iota 定义枚举时，**从非零值开始**：

```go
// 错误
type Operation int

const (
  Add Operation = iota  // 0
  Subtract             // 1
  Multiply             // 2
)

// 正确
const (
  Add Operation = iota + 1  // 1
  Subtract                  // 2
  Multiply                  // 3
)
```

### 使用 time 包处理时间

时间处理很复杂，要使用 time 包：

- 一天不一定是 24 小时
- 一周不一定是 7 天
- 一年不一定是 365 天

```go
// 错误
func isActive(now, start, stop int) bool {
  return start <= now && now < stop
}

// 正确
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

使用 `time.Duration` 表示时间段：
```go
// 错误
func poll(delay int) {
  time.Sleep(time.Duration(delay) * time.Millisecond)
}

// 正确
func poll(delay time.Duration) {
  time.Sleep(delay)
}
```

## 错误处理

### 错误类型

根据需求选择错误声明方式：
- 需要调用者匹配错误：使用顶层错误变量或自定义类型
- 静态错误信息：使用 errors.New
- 动态错误信息：使用 fmt.Errorf

### 错误包装

使用 %w 包装错误：
```go
if err != nil {
  return fmt.Errorf("failed to fetch: %w", err)
}
```

### 不要 Panic

**不要使用 panic**。在 main 函数中使用 log.Fatalf 或自定义错误处理。

## 并发

### 不要丢弃 Goroutine

启动 goroutine 时要确保它有退出机制：

```go
// 错误
go func() {
  // 没有退出机制
}()
```

### 等待 Goroutine 退出

使用 sync.WaitGroup 或 channel 等待 goroutine 退出。

### init() 中不使用 Goroutine

init() 函数中不要启动 goroutine。

## 性能

### 优先使用 strconv

```go
// 优先
strconv.Itoa(n)

// 避免
fmt.Sprintf("%d", n)
```

### 避免重复字符串转换

```go
// 错误：每次转换都分配内存
for _, v := range s {
  w.Write([]byte(v))
}

// 正确：预分配 buffer
data := []byte(s)
for _, v := range s {
  w.Write(data)
}
```

### 指定容器容量

创建 slice 或 map 时，如果知道大小，指定容量可以避免重新分配：

```go
make([]int, 0, 100)  // 预分配 100
make(map[string]int, 100)
```

## 代码风格

### 避免过长行

每行代码不超过 100 个字符。

### 保持一致

整个代码库保持一致的编码风格。

### 包命名

- 使用简洁的包名
- 使用小写字母
- 不使用下划线

### 函数分组和顺序

1. 常量
2. 变量
3. 类型
4. 函数

同类型按导出顺序排列。

### 减少嵌套

减少代码嵌套层级，提前返回：

```go
// 减少 else
if valid {
  // 处理
  return
}
// 继续
```

### 减少变量作用域

尽量在需要使用时再声明变量。

### 使用原始字符串

```go
// 避免转义
query := `SELECT * FROM users
WHERE id = 1`
```

### 结构体初始化

使用字段名初始化：

```go
user := User{
  Name: "Tom",
  Age:  20,
}
```

零值字段可以省略。

### 初始化 Map

```go
// 预分配容量
m := make(map[string]int, 100)
```

## 总结

Uber 的 Go 规范涵盖了代码编写的方方面面，遵循这些规范可以让代码更易读、易维护。建议团队统一采用这套规范。
