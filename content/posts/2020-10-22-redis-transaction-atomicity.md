---
title: 深入理解 Redis 事务与原子性
date: 2020-10-22T02:00:00+00:00
url: /2020/10/redis-transaction-atomicity/
categories:
 - Redis
tags:
 - Redis
 - 事务
 - 原子性

---

Redis 事务是一组命令的集合，通过 MULTI 和 EXEC 命令来执行。事务中的所有命令都会序列化，按顺序串行执行，不会被其他命令插入。本文深入解析 Redis 事务的特性和原子性问题。

<!--more-->

### Redis 事务基本命令

Redis 事务通过以下命令实现：

- **MULTI**：开启一个事务，之后的命令会被放入队列中
- **EXEC**：执行队列中所有的命令
- **DISCARD**：中断当前事务，清空事务队列并放弃执行
- **WATCH**：监视一个或多个 key，如果在事务执行之前被其他命令修改，则事务会被取消

### 事务的特性

Redis 事务具有以下特性：

1. **隔离性**：事务中的所有命令会按顺序串行执行，不会被其他命令插入
2. **原子性**：这是最常被讨论的特性，但实际上 Redis 事务并不具备严格的原子性
3. **持久性**：取决于 Redis 的持久化配置（RDB 或 AOF）

### Redis 事务的原子性问题

这是最容易被误解的地方。Redis 事务的原子性需要分两种情况讨论：

**1. EXEC 执行之前出错**

如果命令存在语法错误或参数错误，Redis 会检查出来并返回错误，自动放弃整个事务。这种情况下，事务中的命令都不会执行。

**2. EXEC 执行之后出错**

如果命令处理了错误类型的键（如对字符串执行列表命令），该条命令会执行失败，但事务中的其他命令仍会继续执行。

示例：

```
MULTI
SET a 3 abc
LPOP a
EXEC

*2
+OK
-ERR Operation against a key holding the wrong kind of value
```

从结果可以看到，第一条命令执行成功，第二条命令执行失败。这说明 Redis 事务**不具备严格的原子性**。

### Redis 为什么不支持回滚？

- Redis 命令只会因为语法错误或错误类型的键而失败
- 不支持回滚可以让 Redis 保持简单且高性能
- 回滚并不能解决编程错误带来的问题

### WATCH 乐观锁

Redis 使用 WATCH 命令实现乐观锁。在执行事务之前监视指定的 key，如果事务执行前这些 key 被其他命令修改，事务会自动失败。

```
WATCH user:1001
$balance = GET user:1001
MULTI
SET user:1001 $balance + 100
EXEC
```

如果其他客户端在此期间修改了 user:1001，事务会执行失败，需要重试。

### Lua 脚本

对于需要更强原子性保证的场景，推荐使用 Lua 脚本。Redis 会原子地执行整个 Lua 脚本，类似数据库的存储过程。

```
EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 mykey "hello"
```

### Redis 7.x Functions

Redis 7.x 引入了 Redis Functions，提供了更强大的脚本能力。Functions 是存储在 Redis 中的可重用脚本，提供了更好的版本控制和命名空间管理。

### 最佳实践

1. 对于简单的事务场景，使用 Redis 事务（MULTI/EXEC）
2. 对于需要强原子性的复杂业务逻辑，使用 Lua 脚本
3. 使用 WATCH 命令实现乐观锁，解决并发问题
4. Redis 事务不适合需要严格回滚的场景，考虑使用数据库事务

### 参考资料

- [Redis 官方文档 - Transactions](https://redis.io/docs/latest/interact/transactions/)
- [Redis 官方文档 - BLPOP](https://redis.io/docs/latest/commands/blpop/)
- [小林coding - Redis面试题](https://www.xiaolincoding.com/interview/redis.html)
