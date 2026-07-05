---
title: '解决部署时新旧构建产物切换导致的静态资源 404'
date: 2026-07-05T08:00:00+08:00
draft: false
tags: ['deployment', 'nextjs', 'devops', 'static-assets']
categories: ['tech']
description: '深入解析部署时新旧构建产物切换导致的静态资源 404 问题，以及如何通过保留旧 static chunks 的方式从根本上解决这类问题。'
---

## 问题现象

在构建部署过程中，访问页面时出现 404 错误，刷新后消失。这不是代码问题，而是**部署时新旧构建产物切换导致的资源 404** —— 典型的 Next.js 部署原子性问题。

<!--more-->

## 问题根因

部署脚本通常这样写：

```bash
mv .next .next-old       # 旧的 .next 备份
mv $BUILD_DIR/.next .next  # 新的 .next 替换过来
pm2 reload photo-blog
```

从旧 chunk 消失到新进程接管之间，有一个短暂空窗：

1. **老 HTML 仍在被访问**（浏览器缓存或 CDN 边缘缓存）→ 引用的是老 chunk hash（`60eada1c6de28afa.js`）
2. `mv .next .next-old` 那一刻，新版本的 `.next/static/chunks/` 里**没有这个 hash 的 chunk**（新 build 会重新生成一套 hash）
3. 静态资源 404

更糟的是 CDN 边缘还缓存着老 HTML，用户拿到的 HTML 里指向的 chunk 在源站已经不存在。

## 根本解法：保留多个构建的 static chunks

Next.js 官方推荐的做法是——**新旧 chunk 目录合并共存一段时间**，让老 HTML 也能读到自己那套 chunk。

核心思路：在 `mv .next` 前，用 `rsync -a --ignore-existing` 把旧 `.next/static/` 合并进新构建：

```bash
if [ -d .next/static ] && [ -d "$BUILD_DIR/.next/static" ]; then
  rsync -a --ignore-existing .next/static/ "$BUILD_DIR/.next/static/"
fi
```

- `--ignore-existing` = 只搬旧构建独有的文件（不覆盖新构建同名 chunk）
- 新构建同名文件保持权威
- 旧 chunk 作为兼容兜底存在

附加优化：`.next-old` 改成时间戳快照（`.next-old.20260705-220300`），保留最近 3 份：

```bash
# 备份旧构建（带时间戳，保留 3 份）
find .next-old* -maxdepth 0 -type d 2>/dev/null | sort -r | tail -n +4 | xargs rm -rf
mv .next .next-old.$(date +%Y%m%d-%H%M%S)
```

这样：
- 磁盘可控（旧的自动清理）
- 万一发现问题，任何一次历史构建都能人工回滚
- 更长的历史窗口保护慢刷新的用户（比如后台标签页几小时后回来）

## 为什么这个方案能根治 chunk 404

| 场景 | 修复前 | 修复后 |
|---|---|---|
| 用户 A：老 HTML 在浏览器缓存里，页面加载新 chunk | ❌ 404（新构建里没这个 hash） | ✅ 200（旧 chunk 还在） |
| CDN 边缘缓存老 HTML，新用户拿到后加载 chunk | ❌ 404 | ✅ 200 |
| 新用户拿到新 HTML，加载新 chunk | ✅ 200 | ✅ 200 |
| 部署失败自动回滚 | 老 static 已被 mv 到 .next-old，回滚后自然能用 | 依然正常 |

## 首次部署提醒

首次上线时 `.next` 里存的还是"旧脚本产生的旧 static"，rsync 合并会先跑一次——**首次生效需要等第二次部署以后才完整**。第一次部署完，如果还有短暂 404 是正常的，之后就不会再有了。

## 完整 deploy.sh 示例

```bash
#!/bin/bash
set -e

BUILD_DIR=${1:-.}
ENV=${2:-production}

# 1. 清理旧备份（保留最近 3 份）
find .next-old* -maxdepth 0 -type d 2>/dev/null | sort -r | tail -n +4 | xargs rm -rf 2>/dev/null || true

# 2. 备份当前构建
if [ -d .next ]; then
  mv .next .next-old.$(date +%Y%m%d-%H%M%S)
fi

# 3. 移动新构建
mv $BUILD_DIR/.next .next

# 4. 核心：合并旧 static chunks（保留老 HTML 需要的 chunk）
if [ -d .next/static ]; then
  # 找到最新的一份旧备份
  LATEST_OLD=$(ls -dt .next-old.*/ 2>/dev/null | head -1)
  if [ -n "$LATEST_OLD" ] && [ -d "$LATEST_OLD/static" ]; then
    rsync -a --ignore-existing "$LATEST_OLD/static/" .next/static/
  fi
fi

# 5. 重载服务
pm2 reload photo-blog
```

## 总结

这个方案的本质是：**在部署这个非原子操作中，通过文件系统的简单合并实现最终一致性**。不需要复杂的版本追踪，不需要双写，只需要一个 `rsync --ignore-existing`，就能让新老 HTML 都能找到自己依赖的 chunk。
