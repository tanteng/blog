---
title: '龙虾帮龙虾：用一只 AI 修另一只 AI'
date: 2026-03-15T10:00:00+08:00
draft: false
tags: ['openclaw', 'workbuddy', 'troubleshooting', 'cron', 'performance']
categories: ['tech']
description: '用一只 AI 远程给另一只 AI 看病——排查 cron 故障、诊断配置问题、优化服务器性能，全程不需要手动敲命令。'
featured_image: 'https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/26bd9117d173eca0578ac48c82b00b9d.png'
---

在腾讯云轻量服务器上部署 OpenClaw 后，cron 全面报错、配置混乱、内存虚高。让 OpenClaw 自己查了几轮没什么进展，于是换 WorkBuddy 上的 Claude Opus 接手——全程没手动敲一行命令。

<!--more-->

两位主角都是龙虾（🦞）：

- **OpenClaw**（🦞 患者）：部署在腾讯云 Lighthouse 上的自动化工具，底层 MiniMax-2.5 模型，负责跑 cron 定时任务推送到 Telegram/Discord。
- **WorkBuddy**（🦞 医生）：腾讯出品的 AI 编程助手，底层 Claude Opus 模型。

WorkBuddy 可以通过 SSH 免密登录直接连到远程服务器，查进程、读日志、改配置、重启服务。它不只是能排查故障，还能做全面的服务器诊断和性能优化——龙虾帮龙虾，能做的事情比想象中多。

![龙虾帮龙虾配图](https://notes-1303209934.cos.ap-guangzhou.myqcloud.com/2026/03/26bd9117d173eca0578ac48c82b00b9d.png)
*配图由 Gemini 生成*

## 一、排查故障

### 问题一：Gateway 连接失败

```bash
[root@VM-0-4-opencloudos ~]# openclaw cron list

🦞 OpenClaw 2026.3.13 (61d171a)

gateway connect failed: Error: gateway closed (1000): 
◇
Error: gateway closed (1000 normal closure): no close reason
Gateway target: ws://127.0.0.1:18789
Source: local loopback
```

OpenClaw 的架构是 CLI + Gateway 分离的：CLI 通过 WebSocket 连接本地 Gateway 守护进程来执行命令。Gateway 监听在 `ws://127.0.0.1:18789`，CLI 每次操作都要先跟它握手。

### 排查过程

**第一步：确认进程和端口**

```bash
$ ps aux | grep openclaw-gateway
root  1527560  21.6  14.1  ...  node /root/.nvm/.../openclaw-gateway

$ ss -tlnp | grep 18789
LISTEN  0  511  127.0.0.1:18789  ...
```

进程在跑，端口在听。那问题出在哪？

**第二步：查看 Gateway 日志**

```bash
$ tail -20 /tmp/openclaw/openclaw-2026-03-24.log
```

日志里出现了关键信息：

```json
{
  "cause": "handshake-timeout",
  "handshake": "failed",
  "durationMs": 3203,
  "handshakeMs": 3001
}
```

WebSocket 连接建立了，但握手在 3 秒内没完成，Gateway 主动断开。

### 初步分析

这台服务器只有 2 核 CPU，Gateway 同时加载了多个插件（Telegram bot、Discord bot、skillhub），当 Gateway 忙于处理插件事件时，无法及时响应 CLI 的 WebSocket 握手请求。这是一个**间歇性问题**——Gateway 空闲时能正常连接，忙时就超时。

但问题来了：CPU 监控显示平均利用率只有 **4.455%**，峰值也不过 49.6%，内存平均 1112MB / 3.6GB，远没到资源瓶颈。**不是服务器不够用，而是超时阈值太小。**

### 深入源码：双端超时瓶颈

深入 OpenClaw 的 dist 代码后，找到了两个超时设置：

**Gateway 端** — `gateway-cli-*.js`：

```javascript
const DEFAULT_HANDSHAKE_TIMEOUT_MS = 3e3;  // 3 秒
```

Gateway 等待握手完成的超时，超过就主动断开连接。

**CLI 端** — `auth-profiles-*.js`：

```javascript
const connectChallengeTimeoutMs = Number(rawConnectDelayMs) || 2e3;  // 2 秒
```

CLI 连上 WebSocket 后等 Gateway 发 `connect.challenge` 事件的超时，2 秒没收到就主动关闭。

完整的握手流程是：

```
CLI 连上 WebSocket
     ↓
等待 Gateway 发 connect.challenge（CLI 端 2 秒超时）
     ↓
CLI 发 connect 请求
     ↓
等待 Gateway 完成认证（Gateway 端 3 秒超时）
```

**核心矛盾**：Gateway 是单线程 Node.js 进程。即使 CPU 总体利用率不高，当事件循环正在处理 Discord/Telegram 插件的回调时（比如 cron 任务刚执行完正在投递结果），微观上事件循环无法及时处理新的 WebSocket 连接。2-3 秒的超时窗口对于一个"偶尔会忙一下"的 Node.js 进程来说太紧了。

### 修复

将两端超时从 2-3 秒改为 15 秒。需要修改 4 个文件（OpenClaw dist 中有重复的 chunk 文件）：

```bash
# Gateway 端（两份 chunk 文件）
sed -i 's/const DEFAULT_HANDSHAKE_TIMEOUT_MS = 3e3;/const DEFAULT_HANDSHAKE_TIMEOUT_MS = 15e3;/' \
  gateway-cli-CuZs0RlJ.js \
  gateway-cli-Ol-vpIk7.js

# CLI 端（两份 auth-profiles 文件）
sed -i 's/rawConnectDelayMs)) : 2e3;/rawConnectDelayMs)) : 15e3;/' \
  auth-profiles-DDVivXkv.js \
  auth-profiles-DRjqKE3G.js
```

修改前备份原文件为 `.bak`，重启 Gateway 后连续测试 5 次，**全部成功**，不再出现握手超时。

> ⚠️ 这些 patch 会在 OpenClaw 更新时被覆盖，需要重新打。

### 问题二：Cron 任务大面积 Error

解决了连接问题后，成功执行 `openclaw cron list`，看到了更大的问题：

```
  Task                      Status    Next Run
  ─────────────────────────────────────────────
  AI发展趋势汇总             error     3m
  每日Hacker News热门        error     1h
  Polymarket热门事件          ok        22h
  ...
```

多个任务 error，只有少数正常。

### 排查过程

**第一步：检查运行记录**

```bash
$ tail -5 /root/.openclaw/cron/runs/task_quote.jsonl
```

运行记录显示任务实际上**执行成功了**——AI 正常生成了内容，`status: "ok"`。那 error 从何而来？

**第二步：读取 jobs.json 配置**

```bash
$ cat /root/.openclaw/cron/jobs.json | jq '.jobs[] | select(.state.lastStatus == "error") | .state.lastError'
```

找到了错误信息：

```
Telegram API: Bad Request: chat not found (chat_id=14824013xxxxxxxxxx)
```

任务执行成功了，但投递结果时失败——它在往一个不存在的 Telegram chat 发消息。

**第三步：追踪错误的 chat_id**

正常的任务 delivery 配置：

```json
{ "channel": "telegram", "to": "7374xxxxx" }
```

出错的任务也是完全相同的配置。delivery 里 `to` 是正确的 Telegram ID，但实际投递时却用了 `14824013xxxxxxxxxx`。

这个数字从哪来的？

```bash
$ grep -r "14824013xxxxxxxxxx" /root/.openclaw/
```

在 Discord 的日志和 session 数据中找到了——这是一个 **Discord 的 #cron 频道 ID**。

### 根因

之前我在 Discord 的 #cron 频道里对 OpenClaw bot 说过"将 cron 定时任务全都发到这个频道"。OpenClaw 记住了这个上下文，给部分任务绑定了一个指向 Discord session 的 `sessionKey`。

当这些任务执行完要投递结果时，Gateway 优先使用了 session 上下文中的投递目标（Discord channel ID），但 delivery 配置里指定的是 `telegram` 通道。于是 Gateway 拿着 Discord 的 channel ID 去调 Telegram API——`chat not found`。

**这是一个 Session 路由 Bug**：跨通道的 session 上下文不应该覆盖 delivery 配置中明确指定的投递目标。

### 修复

```javascript
// 清理所有 cron 任务的 sessionKey 绑定
const data = JSON.parse(fs.readFileSync("/root/.openclaw/cron/jobs.json", "utf8"));

data.jobs.forEach(j => {
  // 删除跨通道的 sessionKey
  if (j.sessionKey) delete j.sessionKey;
  // 重置错误状态
  if (j.state && j.state.lastStatus === "error") {
    j.state.consecutiveErrors = 0;
    delete j.state.lastError;
    j.state.lastStatus = "ok";
  }
});

fs.writeFileSync("/root/.openclaw/cron/jobs.json", JSON.stringify(data, null, 2));
```

重启 Gateway 后，7 个任务全部恢复 `ok` 状态，后续投递全部正确到达 Telegram。

## 二、优化性能

排查完故障之后，我又让 WorkBuddy 对整台服务器做了一次全面体检。它通过 SSH 检查了系统负载、进程列表、配置文件、日志错误频率——然后给出了诊断报告，并逐一修复。

### 问题清单

| # | 问题 | 严重程度 |
|---|------|---------|
| 1 | Telegram bot 409 冲突，1 小时内 14 次 | 高 |
| 2 | MiniMax 备用模型 OAuth 失效（401） | 高 |
| 3 | Gateway 内存 545MB（峰值 937MB），小机器上偏高 | 中 |

### 修复过程

**1. Telegram 409 冲突**

日志里反复出现 `409 Conflict`——同一个 bot token 被多个实例同时 `getUpdates`。排查后确认服务器上只有一个 Gateway 进程，webhook 也没设置。原因可能是上次 Gateway 异常退出后，旧的 long-polling session 没有完全断开。

修复：重启 Gateway + `deleteWebhook?drop_pending_updates=true`，重启后持续观察，冲突消失。

**2. 备用模型 OAuth 失效**

主模型 `minimax-cn` 在 24 小时内被限流 28 次，正常情况下应该回退到 `minimax-portal`，但后者的 OAuth token 已过期（401）。这个需要人工重新授权，AI 帮不了——但至少它帮我发现了这个问题。

**3. 关闭不用的 Channel**

Gateway 同时加载了 Telegram、Discord、元宝（yuanbao）、LightClaw 四个通道。后两个基本没用过，但每个都要维护连接和内存。直接在配置里禁用：

```python
config["channels"]["lightclawbot"]["enabled"] = False
config["channels"]["yuanbao"]["enabled"] = False
```

### 优化效果

全部修复后重启 Gateway：

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 系统内存占用 | 1.6GB | 1.2GB |
| 可用内存 | 2.0GB | 2.4GB |
| Telegram 409 冲突 | 14 次/小时 | 0 |

在一台 2 核 / 3.6GB 的小机器上，释放 400MB 内存不是小事。

## 经验总结

### 1. Node.js 单线程的"微观延迟"不等于"CPU 不够"

服务器 CPU 平均利用率才 4%，但 Gateway 的 WebSocket 握手照样超时。原因是 Node.js 单线程模型下，事件循环被插件回调占用时，新连接的处理会被阻塞。**宏观的 CPU 利用率和微观的事件循环延迟是两回事。** 解决方案不是加 CPU，而是放宽超时阈值。

### 2. 多通道 Session 上下文可能交叉污染

在 Discord 里对 bot 说的话，影响到了 Telegram 通道的投递行为。如果你同时使用多个通道，要注意 session 上下文是否被跨通道共享。这类 Bug 很隐蔽——任务执行成功了、delivery 配置也是对的，但实际投递却走了错误的路径。

### 3. 任务执行成功 ≠ 投递成功

`openclaw cron list` 显示 error 不意味着任务没跑。排查时要区分**执行阶段**和**投递阶段**——执行生成内容、投递发送内容，两个阶段独立失败。

### 4. 让 AI 做全面体检

手动排查往往是「哪疼治哪」，容易漏掉配置错误、ID 填错、废弃进程这些不起眼但一直在消耗资源的问题。让 AI 系统性地过一遍服务器状态，能发现很多自己不会主动去查的角落。

### 5. 打了 dist 文件的 patch 要做好记录

直接改 `node_modules` 下的 dist 文件是最快的修复方式，但每次 `npm update` 都会被覆盖。建议：
- 备份原文件（`.bak`）
- 记录修改内容和文件路径
- 考虑写一个 `postinstall` 脚本自动打 patch

---

*本文基于 OpenClaw 2026.3.13 版本，部署环境为腾讯云 Lighthouse（OpenCloudOS，2 核 CPU / 3.6GB 内存）。*
