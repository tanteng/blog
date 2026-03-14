---
title: "Redis 生产环境使用 SCAN 替代 KEYS"
date: 2018-09-05T14:17:03+00:00
url: /2018/09/redis-scan-command/
categories:
 - tech

---
如果要在 Redis 生产环境服务器查看有哪些数据库键，当数据量特别大的时候千万不要用 `keys *`，这样会卡死，可以使用 `scan` 命令迭代。
