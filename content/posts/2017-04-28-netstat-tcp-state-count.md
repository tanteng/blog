---
title: netstat 命令统计 TCP 各状态数量
author: tanteng
type: post
date: 2017-04-28T03:37:31+00:00
url: /2017/04/netstat-tcp-state-count/
categories:
 - Linux

---
统计 tcp 各种状态的个数：

```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

结果：

```ini
[nobody@test14439 ~]$ netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
TIME_WAIT 418
CLOSE_WAIT 109
ESTABLISHED 65
SYN_RECV 1
```

<!--more-->

### TCP状态说明

状态：描述
 
CLOSED：无连接是活动的或正在进行
 
LISTEN：服务器在等待进入呼叫
 
SYN_RECV：一个连接请求已经到达，等待确认
 
SYN_SENT：应用已经开始，打开一个连接
 
ESTABLISHED：正常数据传输状态
 
FIN_WAIT1：应用说它已经完成
 
FIN_WAIT2：另一边已同意释放
 
ITMED_WAIT：等待所有分组死掉
 
CLOSING：两边同时尝试关闭
 
TIME_WAIT：另一边已初始化一个释放
 
LAST_ACK：等待所有分组死掉

### 解决大量 TIME_WAIT 状态

如发现系统存在大量TIME_WAIT状态的连接，通过调整内核参数解决，

```ini
vim /etc/sysctl.conf
```

编辑文件，加入以下内容：

```ini
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
```


然后执行 `/sbin/sysctl -p` 让参数生效。

**net.ipv4.tcp_syncookies = 1** 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
 
**net.ipv4.tcp\_tw\_reuse = 1** 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
 
**net.ipv4.tcp\_tw\_recycle = 1** 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
 
**net.ipv4.tcp\_fin\_timeout** 修改系統默认的 TIMEOUT 时间

### **各个状态的出现场景**

#### FIN_WAIT1

主动方调用close函数关闭连接后立刻进入FIN\_WAIT1状态，此时只要收到对端确认ACK后马上会进入FIN\_WAIT2状态。

出现场景：主动方等待ACK过程中网络断掉了，导致长时间收不到ACK，主动方就会停留在FIN_WAIT1状态上（超时时间：一般默认60s超时）。此时我们可以使用netstat -anpt 命令看到这种状态。这个状态在实际的工作中很少见。

#### FIN_WAIT2

主动端在等待对端FIN到来过程中，会一你直保持这个状态（超时时间：一般默认是60s）。由于网络中断，或者对端很忙还没来得及发送FIN、或者对端有bug忘记关闭连接等都会导致主动端长时间处于FIN\_WAIT2状态。如果主动方发现大量FIN\_WAIT2状态时，应该引起相关人员的注意，这可能是网络不稳、对端程序bug的表现。这个状态比较常见。

#### TIME_WAIT

主动方收到对端的FIN后进入TIME\_WAIT状态。然后发送最后一个确认ACK到对端。之后等待2个最大的报文存活周期，正常的关闭流程客户端TCP连接都会经过这个状态，最终进入CLOSED状态。所以我们使用netstat -anpt命令发现客户端有很多的TIME\_WAIT，一般这是正常的现象。这个状态最常见。

#### CLOSING

双发几乎同时都调用了close接口主动关闭连接，此时都进入了FIN\_WAIT1状态。如果在FIN\_WAIT1状态期望收到对方的ACK但却收到了对方的FIN，这时候双方都进入CLOSING状态。然后都给对方一个ACK确认，收到了ACK后就会进入CLOSED状态了。

#### CLOSE_WAIT

这个状态表明TCP连接等待被关闭。只可能在被动方出现。如果被动方存在大量的CLOSE_WAIT状态需要因为我们的特别注意了。我们要仔细研究确认为什么被动方迟迟不愿关闭连接（或许是我们程序中的bug开启了连接，用完后却忘记关闭）

目前开发过程中遇到如下这个场景导致被动方有很多的CLOSE_WAIT状态：

A是一个应用程序，B是一个tomcat服务器

A开了一个连接Conn，发送请求给B

A接受相应数据后没有调用Conn.close关闭连接，在A端垃圾回收这些Conn对象前，这些连接一直保持着

B端的连接超时后会主动发起关闭连接请求给A，此时A进入了CLOSE\_WAIT状态，B进入了FIN\_WAIT2状态，由于A迟迟不发送FIN给B，B端触发timeout直接进入了CLOSED状态。

这样一个场景B端由于有超时设置一个为60s，不会存在大量的FIN_WAIT2状态
 
但是A端就会残留大量的CLOSE\_WAIT状态（CLOSE\_WAIT状态也有超时，但是太大，默认为43200s，详情见tcp\_timeout\_close\_wait系统配置）。还好A端的java虚拟机的最大对内存配置较小，由于CLOSE\_WAIT状态连接同样占用了内存资源，数量很多后就会触发垃圾回收，此时A端的CLOSE_WAIT的连接Conn对象就会被销毁了（同时内存和句柄、端口等资源也被释放了）

#### LAST_ACK

当被动端调用close接口关闭连接后便会进入这个状态，同时发送一个FIN给对端。在接受对端的ACK确认后便会进入CLOSED状态，这个状态一般不易出现，除非网络中断，一般对端会很快给与响应的。这个状态只可能在被动端出现。

### 状态总结

主动端可能出现的状态：FIN\_WAIT1、FIN\_WAIT2、CLOSING、TIME_WAIT
 
被动端可能出现的状态：CLOSE\_WAIT LAST\_ACK

NOTE：

（1）主动端出现大量的FIN\_WAIT1时需要注意网络是否畅通、出现大量的FIN\_WAIT2需要仔细检查程序为何迟迟收不到对端的FIN（可能是主动方或者被动方的bug）、出现大量的TIME\_WAIT需要注意系统的并发量/socket句柄资源/内存使用/端口号资源等。

（2）被动端出现大量的 CLOSE_WAIT 需要仔细检查为何自己迟迟不愿调用close关闭连接（可能是bug，socket打开用完没有关闭）

### Questions

1、 为什么建立连接协议是三次握手，而关闭连接却是四次握手呢？

这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意现在可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

2、 为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？

这是因为：虽然双方都同意关闭连接了，而且握手的4个报文也都协调和发送完毕，按理可以直接回到CLOSED状态（就好比从SYN\_SEND状态到ESTABLISH状态那样）；但是因为我们必须要假想网络是不可靠的，你无法保证你最后发送的ACK报文会一定被对方收到，因此对方处于LAST_ACK状态下的SOCKET可能会因为超时未收到ACK报文，而重发FIN报文，所以这个TIME_WAIT状态的作用就是用来重发可能丢失的ACK报文，并保证于此。

##### 参考文章

- <http://www.cnblogs.com/derekchen/archive/2011/02/26/1965839.html>

- <http://cpjsjxy.iteye.com/blog/2090386>
