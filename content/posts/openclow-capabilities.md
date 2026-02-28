---
title: 'MacBook Pro 上安装 OpenClaw AI 助手能做什么'
date: 2026-03-01T01:30:00+08:00
draft: false
tags: ['OpenClaw', 'AI助手', '效率工具', '技术']
categories: ['技术']
---

在 MacBook Pro 上安装了 OpenClaw 后，AI 助手可以直接在本地运行，不依赖云端服务，数据更安全。作为一个自托管的 AI 网关，它连接了各种聊天渠道和 AI Agent，让日常工作和生活更加高效。

## 什么是 OpenClaw？

OpenClaw 是一个开源的**自托管 AI 网关**，通过它可以把 WhatsApp、Telegram、Discord、iMessage 等各种聊天工具和 AI Agent 连接起来。一个 Gateway 进程同时服务多个渠道，数据留在本地，安全可控。

官方文档：[https://docs.openclaw.ai](https://docs.openclaw.ai)

## 能做什么？

### 📁 文件和代码管理

这是最基础也是最强大的能力。AI 可以直接读写文件、搜索代码内容、帮助编写和修改代码。

- 读写任意文本文件
- 搜索代码内容
- 运行终端命令：Git、Node.js、Python、Shell、Brew 等
- 维护项目、整理代码库
- 自动化执行脚本

### 🌐 浏览器自动化

通过浏览器控制能力，可以实现复杂的网页操作：

- 自动填表、点击按钮
- 抓取网页内容（支持 Markdown 格式提取）
- 截图和页面快照
- UI 自动化测试
- 无头浏览器操作

### 📱 多渠道消息通讯

OpenClaw 支持众多消息渠道，一个网关同时接入多个平台：

- **WhatsApp**：通过 WhatsApp Web (Baileys)
- **Telegram**：Bot 支持 (grammY)
- **Discord**：Bot 支持
- **iMessage**：通过本地 imsg CLI（macOS 专享）
- **Signal**
- **Slack**
- **Microsoft Teams**
- **IRC**
- **Line**
- 以及更多...

支持的功能：
- 发送文本、图片、音频、文档
- 创建投票和内联按钮
- 消息引用和回复
- 群组消息管理
- 主题/线程支持（Telegram）

### 🔍 信息查询

- 搜索网页（需配置 Brave API）
- 获取任意网页内容并提取为 Markdown
- 查天气（wttr.in，无需 API）
- 定时任务和轮询

### 🖥️ 设备控制（配对移动节点）

如果配对了 iOS 或 Android 设备，还可以：

- 截屏、摄像头拍照
- 获取设备位置
- 发送系统通知
- 语音交互（Talk Mode）
- 语音唤醒

### 🧠 记忆能力

这是我觉得最实用的特性之一：

- **短期记忆**：记住对话上下文
- **长期记忆**：写入 MEMORY.md 文件，下次还能想起来
- 支持语义搜索记忆内容
- 每日记录和知识沉淀

### ⚙️ 定时任务与自动化

- **Heartbeat（心跳）**：定期主动检查任务（如查邮件、看天气），有需要才提醒你
- **Cron Jobs**：定时执行任务
- **Webhooks**：接收外部事件触发
- **Hooks**：自定义自动化脚本

### 🤖 多 Agent 路由

- 支持多个独立的 Agent
- 按工作区或发送者隔离会话
- 可以为不同渠道配置不同的 Agent

### 🎨 控制界面

- **Web Control UI**：浏览器访问 http://127.0.0.1:18789/
- **macOS 菜单栏应用**
- **TUI 终端界面**
- CLI 命令行工具

### 🔐 安全特性

- 本地运行，数据不外泄
- 支持 OAuth 认证（Anthropic、OpenAI）
- 敏感信息加密存储
- 沙箱模式隔离执行

## 适用场景

1. **个人助理**：回答问题、查资料、整理信息
2. **代码助手**：运行终端命令、查看 Git 状态、写代码
3. **博客维护**：帮我写文章、发布到 Hugo
4. **信息汇总**：定期检查邮件、天气、日程
5. **自动化任务**：定时提醒、数据同步、监控系统
6. **远程控制**：通过手机发消息控制 Mac

## 技术栈

- 系统：macOS / Linux / Windows (WSL2)
- 运行时：Node.js 22+
- API：支持 Anthropic、OpenAI、MiniMax、Moonshot、GLM 等

---

这就是运行在 MacBook Pro 上的本地 AI 助手 —— 数据留在本地，用起来更安心 🚀

了解更多：[OpenClaw 官网](https://docs.openclaw.ai)
