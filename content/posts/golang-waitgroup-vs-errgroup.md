---
title: "Go 并发进阶：WaitGroup vs ErrGroup 详解"
date: 2024-08-23
categories:
  - 技术
tags:
  - Go
  - 并发
  - goroutine
---

> Go 语言提供了丰富的并发原语，本文详细介绍 sync.WaitGroup 和 golang.org/x/sync/errgroup 的区别和使用场景。

## 简介

Go 的并发模型以 goroutine 和 channel 为核心，但在实际项目中，我们经常需要协调多个 goroutine 的执行。这就涉及到两组常用的工具：

- **sync.WaitGroup**：Go 标准库，简单同步
- **errgroup**：Go 扩展库，功能更强大

## sync.WaitGroup

WaitGroup 是 Go 标准库的一部分，用于等待一组 goroutine 完成。

### 基本用法

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        // 执行任务
        fmt.Printf("任务 %d 完成\n", i)
    }(i)
}

wg.Wait()
fmt.Println("所有任务完成")
```

### 特点

- ✅ 轻量级，使用简单
- ✅ 无需引入额外依赖
- ❌ 无法获取任务执行结果
- ❌ 无法处理错误
- ❌ 没有内置的取消机制

## errgroup

errgroup 是 `golang.org/x/sync` 扩展包提供的，相比 WaitGroup 增加了错误处理和 Context 取消功能。

### 基本用法

```go
import "golang.org/x/sync/errgroup"

func main() {
    g := &errgroup.Group{}

    g.Go(func() error {
        // 任务1
        return nil
    })

    g.Go(func() error {
        // 任务2，可能返回错误
        return errors.New("任务失败")
    })

    // 等待所有任务完成，并返回第一个错误
    if err := g.Wait(); err != nil {
        fmt.Println("有任务失败:", err)
    }
}
```

### 配合 Context 使用

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    // 支持 context 取消
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // 执行任务
        return nil
    }
})

g.Go(func() error {
    // 另一个任务
    return doSomething()
})

if err := g.Wait(); err != nil {
    fmt.Println("错误:", err)
}
```

## 核心区别对比

| 特性 | WaitGroup | errgroup |
|------|-----------|----------|
| 等待完成 | ✅ | ✅ |
| 返回错误 | ❌ | ✅ |
| Context 取消 | ❌ | ✅ |
| 并发数限制 | ❌ | ✅ (需额外处理) |
| 标准库 | ✅ | ❌ (需引入 golang.org/x/sync) |

## 使用场景

### 适合用 WaitGroup 的场景

1. **简单的同步场景**
   - 并发执行多个独立任务
   - 不关心任务执行结果
   - 不需要错误处理

```go
var wg sync.WaitGroup
for _, file := range files {
    wg.Add(1)
    go func(f string) {
        defer wg.Done()
        processFile(f)
    }(file)
}
wg.Wait()
```

2. **基准测试**
   - 并发压测
   - 简单计数

### 适合用 errgroup 的场景

1. **需要错误处理的场景**
   - 任何一个任务失败都需要知道
   - 需要根据错误做相应处理

```go
g.Go(func() error {
    return fetchFromAPI()
})
g.Go(func() error {
    return saveToDatabase()
})
if err := g.Wait(); err != nil {
    log.Fatal(err)
}
```

2. **需要 Context 取消的场景**
   - 主任务失败时取消其他任务
   - 超时控制
   - 优雅关闭

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    return fetchData(ctx)
})
g.Go(func() error {
    return processData(ctx)
})

// 任何一个返回错误，其他都会通过 context 取消
g.Wait()
```

3. **需要知道具体错误信息的场景**
   - 多个任务可能有不同错误
   - 需要知道哪个任务失败了

## 进阶用法

### 限制并发数

```go
g := &errgroup.Group{}

// 使用 channel 模拟并发限制
sem := make(chan struct{}, 3)

for i := 0; i < 10; i++ {
    g.Go(func() error {
        sem <- struct{}{}        // 获取信号量
        defer func() { <-sem }() // 释放
        
        // 最多同时3个任务
        return doTask()
    })
}
```

### 收集所有错误

```go
var mu sync.Mutex
var errors []error

g := &errgroup.Group{}
for i := 0; i < 5; i++ {
    g.Go(func() error {
        if err := doTask(); err != nil {
            mu.Lock()
            errors = append(errors, err)
            mu.Unlock()
        }
        return nil // 不阻断其他任务
    })
}

g.Wait()
for _, err := range errors {
    fmt.Println("错误:", err)
}
```

## 总结

- **WaitGroup**：简单场景的首选，不需要关注结果，只需要"把任务跑完"
- **errgroup**：复杂场景的首选，需要错误处理、Context 取消、优雅关闭

> 根据实际需求选择合适的工具，不要过度设计。如果只是简单的等待，用 WaitGroup 就够了。

---

## 参考资料

- [errgroup 官方文档](https://pkg.go.dev/golang.org/x/sync/errgroup)
- [Go 官方博客：Context and Structs](https://go.dev/blog/context)
- [When to use ErrGroup rather than WaitGroup](https://medium.com/@pengcheng1222/golang-when-do-you-need-to-use-errgroup-rather-than-waitgroup-ad5e0779a2c3)
