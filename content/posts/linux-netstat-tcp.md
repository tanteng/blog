---
title: "Linux netstat 统计 TCP 连接状态"
date: 2017-04-28T03:37:31+08:00
draft: false
tags: ['linux', 'netstat']
categories: ['tech']
description: "使用 netstat 命令统计 TCP 连接状态"
---

在 Linux 服务器运维中，了解 TCP 连接状态对于排查网络问题非常重要。本文介绍如何使用 netstat 统计各状态的连接数量。

### 统计 TCP 各状态数量

```bash
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

输出结果示例：

<!--more-->

```
TIME_WAIT 418
CLOSE_WAIT 109
ESTABLISHED 65
SYN_RECV 1
```

### TCP 状态说明

| 状态 | 描述 |
|------|------|
| CLOSED | 无连接是活动的或正在进行 |
| LISTEN | 服务器在等待进入呼叫 |
| SYN_RECV | 一个连接请求已经到达，等待确认 |
| SYN_SENT | 应用已经开始，打开一个连接 |
| ESTABLISHED | 正常数据传输状态 |
| FIN_WAIT1 | 应用说它已经完成 |
| FIN_WAIT2 | 另一边已同意释放 |
| TIME_WAIT | 另一边已初始化一个释放 |
| CLOSING | 两边同时尝试关闭 |
| LAST_ACK | 等待所有分组死掉 |

### 解决大量 TIME_WAIT 状态

如发现系统存在大量 TIME_WAIT 状态的连接，可以通过调整内核参数解决：

```bash
vim /etc/sysctl.conf
```

添加以下内容：

```ini
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
```

执行使参数生效：

```bash
/sbin/sysctl -p
```

参数说明：
- `tcp_syncookies = 1` - 开启 SYN Cookies，防范少量 SYN 攻击
- `tcp_tw_reuse = 1` - 允许将 TIME-WAIT sockets 重新用于新的 TCP 连接
- `tcp_tw_recycle = 1` - 开启 TIME-WAIT sockets 的快速回收
- `tcp_fin_timeout` - 修改 FIN_WAIT 超时时间

### 各状态出现场景

**FIN_WAIT1**：主动方调用 close 函数后进入，等待 ACK 过程中网络断开会导致长时间停留

**FIN_WAIT2**：主动方等待对端 FIN，正常关闭流程中常见

**TIME_WAIT**：主动方收到 FIN 后进入，正常关闭流程都会经过此状态，2MSL 后进入 CLOSED

**CLOSE_WAIT**：被动方等待被关闭，程序中忘记调用 close() 会导致大量此状态

**LAST_ACK**：被动方调用 close 后进入，等待最后确认
