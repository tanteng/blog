---
title: "Next.js 照片博客性能优化：SSL 架构精简与协议升级"
date: 2026-03-29T00:00:00+08:00
draft: false
tags: ['next.js', 'performance', 'nginx', 'edgeone', 'tencent-cloud']
categories: ['tech']
description: "在完成 Vercel 到腾讯云的迁移后，进一步优化 Nginx SSL 架构和网络协议，去掉双重 SSL 终止，启用 HTTP/3 (QUIC)，TTFB 最高降低 73%"
---

[上一篇文章](/posts/vercel-to-tencent-cloud-migration/)记录了将照片站点从 Vercel 迁移到腾讯云 Lighthouse 的过程。迁移完成后，站点功能正常、性能也有明显提升。但在对 Nginx 日志和响应时间做进一步分析后，发现还有优化空间——主要集中在 SSL 架构和网络协议层面。

<!--more-->

## 发现问题

迁移完成后的架构是这样的：

```
用户 ──HTTPS──→ EdgeOne(SSL终止) ──HTTPS──→ Nginx(:443, Certbot证书) ──→ Next.js(:3000)
```

用 curl 分解各阶段耗时后，发现了问题：

| 阶段 | 耗时 |
|------|------|
| Next.js 本地响应 | **4ms** |
| 直接访问源站（绕过 CDN） | **66ms** TTFB |
| 经 EdgeOne（CDN 命中） | **152ms** TTFB |
| TLS 握手 | **85-98ms** |

Next.js 本身极快（4ms，ISR 缓存命中），瓶颈在 TLS 握手和 CDN 处理。更关键的是，SSL 被终止了**两次**：

1. **EdgeOne 终止一次**：用户到 EdgeOne 之间是 HTTPS
2. **Nginx 又终止一次**：EdgeOne 到 Nginx 之间又走 HTTPS，Nginx 用 Certbot 证书再解密一次

第二次 SSL 终止完全多余——EdgeOne 到源站走的是腾讯云内网，不需要加密。

---

## 优化一：去掉 Nginx SSL，单点终止

### 改动

**Nginx 配置**：去掉 443 端口和 SSL 证书，只保留 80 端口：

```nginx
server {
    listen 80;
    http2 on;
    server_name photos.tanteng.space;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml text/javascript image/svg+xml;
    gzip_min_length 1000;

    # Next.js 静态资源长期缓存
    location /_next/static/ {
        proxy_pass http://127.0.0.1:3000;
        expires 365d;
        add_header Cache-Control "public, immutable";
    }

    # 反向代理到 Next.js
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_cache_bypass $http_upgrade;
    }
}
```

注意 `X-Forwarded-Proto` 硬编码为 `https`——因为用户侧始终是 HTTPS（EdgeOne 管理证书），但回源走 HTTP，如果用 `$scheme` 会传 `http`，可能导致 Next.js 的重定向逻辑出问题。

**EdgeOne 配置**：回源协议从 HTTPS 改为 HTTP，回源端口 80。

### 优化后的架构

```
用户 ──HTTPS──→ EdgeOne（SSL终止）──HTTP──→ Nginx(:80) ──→ Next.js(:3000)
```

SSL 只在 EdgeOne 终止一次，证书由 EdgeOne 自动管理。Nginx 不再处理 SSL，不需要 Certbot 自动续期。

---

## 优化二：启用 HTTP/2 和 HTTP/3 (QUIC)

在 EdgeOne 控制台开启了两个协议优化：

- **HTTP/2**：多路复用，头部压缩（用户 → EdgeOne）
- **HTTP/3 (QUIC)**：基于 UDP，0-RTT 连接恢复，弱网环境提升明显

开启后，响应头中出现了 `alt-svc: h3=":443"`，支持 QUIC 的浏览器（Chrome、Edge、Firefox）在首次 HTTP/2 连接后，后续访问会自动升级到 HTTP/3。

---

## 关于 h2c 回源的探索

既然 Nginx 回源走的是 HTTP 明文，能不能让回源也用 HTTP/2？HTTP/2 有一个明文模式叫 **h2c**（HTTP/2 cleartext），理论上可以在不加密的情况下使用 HTTP/2 的多路复用和头部压缩。

Nginx 从 1.25.1 开始支持 h2c，配置 `http2 on` 即可。但实测发现回源仍然是 HTTP/1.1。

原因是 h2c 有两种握手方式：

| 方式 | 过程 | Nginx 支持 |
|------|------|-----------|
| **Upgrade** | 先发 HTTP/1.1，服务端返回 101 后切换 | ❌ |
| **Prior Knowledge** | 直接发 HTTP/2 帧 | ✅ |

EdgeOne 使用的是 Upgrade 方式，而 Nginx 有意不支持 h2c Upgrade——Nginx 团队认为这种"先试探再升级"的机制增加复杂度却带来有限收益。大部分其他 Web 服务器（Apache、Caddy、Go net/http）都支持两种方式。

不过这对实际性能影响很小：CDN 命中率高的站点回源不频繁，每次回源只拿单个 HTML 文件，HTTP/2 的多路复用优势用不上。

---

## 优化效果

### TTFB 对比

| 页面 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 首页（CDN 缓存命中） | 152ms | **113ms** | -26% |
| 首页（首次访问） | 259ms | **176ms** | -32% |
| 照片详情页 | 293ms | **113ms** | -61% |
| 标签页 | 443ms | **120ms** | -73% |

### 运维简化

| 方面 | 优化前 | 优化后 |
|------|--------|--------|
| SSL 证书 | EdgeOne + Certbot（两套） | 仅 EdgeOne（自动管理） |
| Nginx 监听端口 | 80 + 443 | 仅 80 |
| 回源加密 | HTTPS（多一次 TLS） | HTTP（无 TLS 开销） |

---

## 最终架构

```
用户 ──HTTP/2 or HTTP/3──→ EdgeOne（SSL终止 + CDN缓存）──HTTP/1.1──→ Nginx(:80) ──→ Next.js(:3000)
```

三层各司其职：

- **EdgeOne**：SSL 终止、CDN 缓存、HTTP/2 & HTTP/3、WAF 防护
- **Nginx**：反向代理、Gzip 压缩、安全头、访问日志
- **Next.js**：页面渲染、ISR 缓存

---

## 总结

这次优化的核心思路很简单：**去掉多余的 SSL 终止，让协议栈更高效**。SSL 在 EdgeOne 边缘节点处理一次就够了，回源走明文 HTTP 既快又省心。HTTP/3 (QUIC) 的启用则让用户侧的连接建立更快，尤其是移动网络和弱网环境。

有时候性能优化不需要改代码，架构层面少做一件多余的事，就是最好的优化。
