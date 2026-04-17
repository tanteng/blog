---
title: "Laravel Redis::throttle 限流组件解析"
date: 2020-07-02T10:00:00+08:00
draft: false
tags:
 - laravel
 - redis
 - rate-limiting
categories: ['tech']
description: "详解 Laravel Redis::throttle 的实现原理，分析其与标准令牌桶算法的区别。"
---

在调用第三方 API 时，经常会遇到 QPS 限制。如果请求频率超过限制，API 会返回错误。此时我们需要限流 + 重试机制来保护下游服务。

本文详细介绍 Laravel 中 `Redis::throttle` 的实现原理。**需要注意的是：Laravel 的实现实际上是固定窗口计数器，而非标准的令牌桶算法。**

<!--more-->

## 限流算法概述

常见的限流算法有三种：

| 算法 | 优点 | 缺点 |
|------|------|------|
| 固定窗口 | 实现简单高效 | 窗口边界存在突刺 |
| 令牌桶 | 平滑，允许突发 | 实现稍复杂 |
| 滑动窗口 | 精确，无突刺 | 实现最复杂 |

## 标准令牌桶 vs Laravel 实现

| 特性 | 标准令牌桶 (Token Bucket) | Laravel `Redis::throttle` |
|------|--------------------------|--------------------------|
| **令牌生成** | 以固定速率**持续**往桶中添加令牌 | ❌ **没有令牌生成过程** |
| **令牌积累** | 空闲时令牌会积累，最多到桶容量上限 | ❌ **不会积累**，窗口重置后直接从 0 开始 |
| **突发处理** | 桶中有积累的令牌时，可以瞬间消费多个 | ❌ 每个窗口固定 N 次，不存在"积累"概念 |
| **时间模型** | 连续的、平滑的 | 离散的、按窗口切割 |

## Redis 数据结构

限流状态存储在 Redis Hash 中：

![Redis Hash 结构](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/04/04d5db83ee7a79f94c02bb8c16eabf12.png)

| Field | 说明 |
|-------|------|
| `start` | 窗口起始时间（Unix 秒） |
| `end` | 窗口结束时间（start + every） |
| `count` | 当前窗口内已消耗的请求次数 |

TTL 设置为 `every * 2`，作为自动过期的兜底。

## 限流判断流程

{{< mermaid >}}
flowchart TD
    A["读取 Hash key<br/>start / end / count"] --> B{key 存在?}

    B -->|否| C["场景1：初始化窗口<br/>count = 1"]
    C --> G["✅ 允许通过"]

    B -->|是| D{当前时间 <= end?}

    D -->|否| E["场景3：窗口过期<br/>重置窗口，count = 1"]
    E --> G

    D -->|是| F["场景2：count + 1"]
    F --> H{count <= allow?}

    H -->|否| I["❌ 限流拒绝"]
    H -->|是| G
{{< /mermaid >}}

## 底层 Lua 脚本

`Redis::throttle` 核心逻辑（来自 Laravel 源码 `DurationLimiter.php`）：

```lua
local function reset()
    redis.call('HMSET', KEYS[1], 'start', ARGV[2], 'end', ARGV[2] + ARGV[3], 'count', 1)
    return redis.call('EXPIRE', KEYS[1], ARGV[3] * 2)
end

-- 1. key 不存在 → 初始化窗口，计数=1，允许通过
if redis.call('EXISTS', KEYS[1]) == 0 then
    return {reset(), ARGV[2] + ARGV[3], ARGV[4] - 1}
end

-- 2. 当前时间在窗口内 → 计数+1，判断是否超限
if ARGV[1] >= redis.call('HGET', KEYS[1], 'start') and ARGV[1] <= redis.call('HGET', KEYS[1], 'end') then
    return {
        tonumber(redis.call('HINCRBY', KEYS[1], 'count', 1)) <= tonumber(ARGV[4]),
        redis.call('HGET', KEYS[1], 'end'),
        ARGV[4] - redis.call('HGET', KEYS[1], 'count')
    }
end

-- 3. 窗口已过期 → 重置窗口
return {reset(), ARGV[2] + ARGV[3], ARGV[4] - 1}
```

这是一个**教科书式的固定窗口计数器**：
- 维护一个固定时间窗口 `[start, start + every]`
- 窗口内每次请求 `count + 1`
- `count > allow` 就拒绝
- 窗口过期就重置

## 固定窗口的边界问题

假设配置为 `allow=20, every=1`（每秒最多 20 次请求），窗口边界处可能产生突发流量。

假设请求恰好这样分布：

```
时间轴：
|-------- 窗口1 (0s~1s) --------|-------- 窗口2 (1s~2s) --------|

时刻 0.5s ~ 0.9s：收到 20 个请求（窗口 1 配额耗尽）
时刻 1.0s：窗口 2 开始，配额立即变为 20
时刻 1.0s ~ 1.2s：又收到 20 个请求

结果：0.7 秒内处理了 40 个请求，平均 QPS ≈ 57
```

**原因**：窗口 1 结束时（1s 时刻），窗口 2 的 20 次配额立即可用。如果请求在窗口切换时集中到达，两个窗口的配额会被集中在短时间内消耗掉。

**解决方式**：留 20% 余量 + 指数退避重试。

## Laravel Redis::throttle 实现

Laravel 提供了 `Redis::throttle` 门面来实现限流：

```php
Redis::throttle($throttleKey)
    ->allow($allow)   // 每个时间窗口允许的最大请求数
    ->every($every)   // 时间窗口（秒）
    ->block(0)        // 不阻塞，立即返回
    ->then(
        function () {
            // 限流通过，执行实际业务逻辑
        },
        function () {
            // 限流拒绝
        }
    );
```

### 核心参数

| 参数 | 说明 |
|------|------|
| `throttleKey` | 限流标识 key，不同 API 用不同 key |
| `allow` | 每个时间窗口允许的最大请求数 |
| `every` | 时间窗口（秒） |
| `block(0)` | 非阻塞模式，立即返回，不等待 |

### 配置示例

## 封装 Trait 复用

封装为 `ApiThrottleTrait`，核心方法 `callWithThrottle($throttleKey, $callback, $options)` 统一处理限流 + 指数退避重试。

使用时仅需 `use` trait，用 `$this->callWithThrottle()` 包裹原有 API 调用即可：

```php
class Asr
{
    use ApiThrottleTrait;

    public function createRecTask($audioUrl)
    {
        return $this->callWithThrottle('asr_create_rec_task', function () use ($audioUrl) {
            return $this->client->createRecTask($audioUrl);
        }, [
            'allow' => 20,
            'every' => 1,
        ]);
    }
}
```

限流被触发时自动指数退避重试，支持配置：
- `maxRetries`：最大重试次数
- `baseDelay` / `maxDelay`：退避延迟范围
- `retryOn`：异常白名单（支持类名或消息关键词匹配）

重试耗尽后抛出 `ThrottleExceededException`，调用方（通常是队列任务）可据此将任务放入延迟队列重试。

## 注意事项

1. **throttleKey 必须全局唯一**：不同 API 用不同 key，否则会互相影响限流计数
2. **Redis 连接**：`Redis::throttle` 使用 Redis Facade 的 default 连接
3. **同步阻塞**：`callWithThrottle` 内部使用 `usleep` 做退避等待，是同步阻塞的，适合队列任务场景
4. **callback 中不要捕获异常**：异常应该抛出来让 Trait 处理重试逻辑
5. **限流是进程级共享的**：同一个 throttleKey 的限流计数在所有进程间共享（基于 Redis）

## 总结

Laravel 的 `Redis::throttle` 虽然名字和注释中提到"bucket"，但底层实现实际上是**固定窗口计数器**。它没有令牌的"持续生成"和"积累"机制，只是在固定时间窗口内做简单计数。

不过对于大多数场景（如 ASR 限流），这个差异可以忽略。配合封装好的 Trait，可以方便地在多个服务中复用限流 + 重试逻辑。

## 参考资料

- [Laravel 文档 - Redis](https://learnku.com/docs/laravel/9.x/redis/12242)
- [Redis 官方文档 - HASH](https://redis.io/docs/data-types/hashes/)
