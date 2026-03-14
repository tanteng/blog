---
title: "Redis 事务是否具有原子性？深度解析"
date: 2018-06-24T02:43:46+00:00
url: /2018/06/redis-transaction-atomicity/
categories:
 - tech

---
ACID 中关于原子性的定义：事务中的所有操作，要么全部完成，要么全部不完成。

那么 Redis 的事务到底符不符合原子性的特征呢？

<!--more-->

### Redis 事务特性

官方文档对事务的描述：事务可以一次执行多个命令，带有以下保证：

- 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序执行
- 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行

### 事务执行失败的情况

使用事务可能会遇到以下两种错误：

1. **EXEC 之前出错**：命令产生语法错误，参数错误等，Redis 会检查出来并返回错误，自动放弃事务
2. **EXEC 之后出错**：命令处理了错误类型的键，比如将列表命令用在字符串键上面

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

对于 EXEC 执行之前的错误，Redis 会检查出来并返回错误自动放弃事务，但是对于在 EXEC 调用后执行失败的情况，该条语句会执行失败，但事务中的其他命令仍会执行。

因此严格来说， Redis 事务确实不具备原子性的特征。

### Redis 为什么不支持回滚？

- Redis 命令只会因为错误的语法而失败，或是命令用在了错误类型的键上面
- 因为不需要对回滚进行支持，Redis 的内部可以保持简单且快速
- 回滚并不能解决编程错误带来的问题

### 2025年更新

Redis 事务机制在 Redis 7.x 中引入了 Redis Functions，进一步增强了脚本能力。对于需要更强原子性保证的场景，建议使用 Lua 脚本或 Redis Functions。

### 参考资料

- [Redis 事务文档](https://redis.io/docs/latest/interact/transactions/)
- [Redis 事务实现](http://redisbook.readthedocs.io/en/latest/feature/transaction.html)
- [ACID - Wikipedia](https://zh.wikipedia.org/wiki/ACID)
