---
title: "Next.js 照片博客性能优化：回源协议与 HTTP/3 升级"
date: 2026-03-29T00:00:00+08:00
draft: false
tags: ['next.js', 'performance', 'nginx', 'edgeone', 'tencent-cloud']
categories: ['tech']
description: "在完成 Vercel 到腾讯云的迁移后，围绕回源协议做了一系列优化：从双重 SSL 到 HTTP 明文，再到 HTTPS + HTTP/2 回源，最终实现全链路 HTTP/2 多路复用。同时启用 HTTP/3 (QUIC) 提升用户侧体验"
---

[上一篇文章](/posts/vercel-to-tencent-cloud-migration/)记录了将照片站点从 Vercel 迁移到腾讯云 Lighthouse 的过程。迁移完成后，站点功能正常、性能也有明显提升。但在对 Nginx 日志和响应时间做进一步分析后，发现回源架构和协议层面还有优化空间。

<!--more-->

## 回源协议的来回折腾

这部分经历了三次架构变更，值得简单记录一下决策过程。

### 第一阶段：双重 SSL（迁移初始状态）

```
用户 ──HTTPS──→ EdgeOne(SSL终止) ──HTTPS──→ Nginx(:443, Certbot证书) ──→ Next.js
```

EdgeOne 到 Nginx 之间走 HTTPS，SSL 被终止了两次。虽然都在腾讯云体系内，但 EdgeOne 边缘节点遍布全国，回源到广州的 Lighthouse 并非 VPC 内网通信，第二次 TLS 握手在安全上意义不大，主要是额外开销。实测这层多余的加解密给 TTFB 增加了 85-98ms。

### 第二阶段：去掉 Nginx SSL，HTTP 明文回源

```
用户 ──HTTPS──→ EdgeOne(SSL终止) ──HTTP──→ Nginx(:80) ──→ Next.js
```

去掉 Nginx 的 443 端口和 Certbot 证书，回源改走 HTTP。TTFB 降低了 26%-73%，运维也简化了（不用管证书续期）。看似完美，但埋了一个隐患。

### 第三阶段：恢复 HTTPS，启用 HTTP/2 回源

在后续[引入 COS 数据万象做图片优化](/posts/cos-ci-image-optimization/)后，图片请求不再经过源站，服务器 CPU 压力消失。但测试中发现部署清缓存后 JS 静态文件加载异常缓慢——20+ 个 JS chunk 全部 MISS 回源，部分请求等了 8 秒。

查 Nginx 日志发现了典型的 HTTP/1.1 排队模式：

```
# 同一秒内 12 个 JS 文件回源
前 6 个：rt=0.002s ~ 0.007s    ← 立即处理
后 6 个：rt=1.55s ~ 1.77s      ← 排队等待
```

**HTTP/1.1 每个连接只能串行处理请求**，EdgeOne 到源站大约建了 6 个并发连接，前 6 个请求秒回，后 6 个要等前一批完成。在 EdgeOne 内部可能还有多层节点逐级回源，排队延迟层层叠加，浏览器端就变成了好几秒。

解决方案是恢复 HTTPS 443 端口，让 EdgeOne 走 **HTTP/2 回源**——HTTP/2 的多路复用可以在单个连接上并行处理所有请求，消除排队。

```
用户 ──HTTP/2 or HTTP/3──→ EdgeOne(SSL终止) ──HTTPS + H2──→ Nginx(:443) ──→ Next.js
```

虽然加回了一层 TLS，但内网 TLS 握手只增加几毫秒，而 HTTP/2 多路复用在并发回源时能节省秒级延迟。**权衡之下，H2 多路复用的收益远大于 TLS 的微小开销。**

### 为什么不用 h2c？

在第二阶段（HTTP 明文回源）时，理论上可以用 **h2c**（HTTP/2 cleartext）在不加密的情况下获得 HTTP/2 的多路复用，两全其美。Nginx 配了 `http2 on` 也确实支持 h2c。

但实测发现 EdgeOne 回源仍然走 HTTP/1.1。原因是 h2c 有两种握手方式：

| 方式 | 过程 | Nginx 支持 |
|------|------|-----------|
| **Upgrade** | 先发 HTTP/1.1，服务端返回 101 后切换 | ❌ |
| **Prior Knowledge** | 直接发 HTTP/2 帧 | ✅ |

EdgeOne 使用 Upgrade 方式，Nginx 只支持 Prior Knowledge，两边对不上。所以 h2c 这条路走不通，最终选择了 HTTPS + H2 回源。

## HTTP/2 和 HTTP/3 (QUIC) 升级

回源协议折腾完毕，来看真正重要的部分——用户侧的协议升级。

### HTTP/2：多路复用的威力

在 EdgeOne 开启 HTTP/2 后，浏览器到 CDN 之间的连接方式发生了质变：

**HTTP/1.1 时代**：浏览器对同一域名最多 6 个并发 TCP 连接，20+ 个 JS/CSS 文件只能排队。

```
连接1: JS-A ────→ JS-G ────→ ...
连接2: JS-B ────→ JS-H ────→ ...
连接3: JS-C ────→ JS-I ────→ ...
连接4: JS-D ────→ JS-J ────→ ...
连接5: JS-E ────→ JS-K ────→ ...
连接6: JS-F ────→ JS-L ────→ ...
         ↑ 前6个立即发出，其余排队等待
```

**HTTP/2 时代**：单个连接，所有请求并行传输。

```
连接1: JS-A + JS-B + JS-C + JS-D + ... + JS-L （全部并行，多路复用）
```

有一次不小心在 EdgeOne 把 HTTP/2 关掉了（本意是关 HTTP/2 *回源*），结果 20+ 个 JS chunk 加载时间从 1 秒飙到 7-8 秒——这就是多路复用的差距。

### HTTP/3 (QUIC)：0-RTT 和弱网体验

HTTP/3 基于 QUIC 协议（UDP），相比 HTTP/2 (TCP) 有几个关键优势：

**1. 连接建立更快**

传统 HTTPS 需要 TCP 三次握手 + TLS 握手，合计 2-3 个 RTT。QUIC 将传输层和加密层合并，首次连接 1 RTT，重连 **0-RTT**。

```
HTTP/2 (TCP+TLS):
  Client ──SYN──→ Server
  Client ←──SYN-ACK── Server
  Client ──ACK+ClientHello──→ Server
  Client ←──ServerHello── Server
  Client ──Request──→ Server        (2-3 RTT 后才能发请求)

HTTP/3 (QUIC):
  Client ──Initial+Request──→ Server  (0-1 RTT，请求和握手同时发出)
```

**2. 消除队头阻塞**

HTTP/2 虽然应用层多路复用，但底层仍然是单个 TCP 连接。一旦某个 TCP 包丢失，整个连接的所有流都要等重传——这就是**队头阻塞**（Head-of-Line Blocking）。

QUIC 的多路复用在传输层实现，每个流独立，一个流丢包不影响其他流。这在移动网络（丢包率高）上提升尤为明显。

**3. 连接迁移**

手机在 Wi-Fi 和 4G 之间切换时，TCP 连接会断开重建。QUIC 用 Connection ID 标识连接（而非 IP+端口），网络切换时连接无缝迁移，不会中断正在传输的资源。

### 开启方式

在 EdgeOne 控制台开启 HTTP/2 和 HTTP/3 后，响应头中出现了：

```
alt-svc: h3=":443"; ma=2592000
```

浏览器在首次 HTTP/2 连接后，记住 `alt-svc` 指示，后续访问自动升级到 HTTP/3。在 Chrome DevTools 的 Protocol 列可以看到 `h3` 标识。

### 实际效果

在浏览器 DevTools 中可以清楚看到协议分布：

| 请求类型 | 协议 |
|----------|------|
| 首次页面访问 | HTTP/2 |
| 后续页面访问 | **HTTP/3 (h3)** |
| RSC 数据请求 | **HTTP/3 (h3)** |
| 图片（COS CDN） | HTTP/1.1 → HTTP/2（取决于 CDN 配置） |

## 最终架构

```
用户 ──H2/H3(QUIC)──→ EdgeOne（SSL终止 + CDN缓存）──H2(HTTPS)──→ Nginx(:443) ──→ Next.js(:3000)
```

三层各司其职：

- **EdgeOne**：SSL 终止、CDN 缓存、HTTP/2 & HTTP/3、WAF 防护
- **Nginx**：反向代理（HTTPS + H2）、Gzip 压缩、访问日志
- **Next.js**：页面渲染、ISR 缓存

回源走 HTTPS + HTTP/2，用户侧走 HTTP/2 和 HTTP/3 (QUIC)，全链路多路复用，无排队瓶颈。

## 源站 IP 防护

源站 443 端口暴露在公网上，需要防止绕过 CDN 直接访问。在 Nginx 中用 `default_server` 拦截所有非域名请求：

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate /etc/letsencrypt/live/photos.tanteng.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/photos.tanteng.space/privkey.pem;

    return 444;
}
```

用 IP 直接访问会被 Nginx 断开连接（444），只有带正确 `Host` 头的 EdgeOne 回源请求才能命中站点配置。

## 总结

回源协议的选择不是一个非黑即白的决策。最初去掉 SSL 是因为双重 TLS 握手浪费了 85ms+，后来恢复 HTTPS 是因为需要 HTTP/2 多路复用来解决并发回源排队。每一步都是基于实际数据做的权衡——有时候加回一层"开销"反而让整体更快。

用户侧的 HTTP/3 (QUIC) 升级则是纯粹的收益：更快的连接建立、消除队头阻塞、连接迁移。对于照片博客这种图片密集、移动端访问多的场景，效果尤其明显。
