---
title: "PHP 知识整理：基础、WEB安全与网络"
date: 2018-04-10T03:11:14+00:00
url: /2018/04/php-knowledge-summary/
categories:
 - tech

---
本篇文章是 PHP 知识系统整理系列之——PHP基础、WEB安全、网络，包括 PSR 规范，PHP7特性和性能提升，HTTP、HTTPS、TCP、WebSocket协议，WEB安全和计算机网络的内容。

<!--more-->

## PHP 基础部分

### PSR标准规范

PSR 是 PHP Standard Recommendations 的简写，由 PHP FIG 组织制定的 PHP 规范，是 PHP 开发的实践标准。

PHP FIG，FIG 是 Framework Interoperability Group（框架可互用性小组）的缩写，由几位开源框架的开发者成立于 2009 年，代表了大部分的 PHP 社区。

目前已表决通过了6套标准：

| PSR | 说明 |
|-----|------|
| PSR 1 | 基础编码规范 |
| PSR 2 | 编程风格规范 |
| PSR 3 | 日志接口规范 |
| PSR 4 | 自动加载规范 |
| PSR 6 | 缓存接口规范 |
| PSR 7 | HTTP消息接口规范 |

注：PSR 0 自动加载已废弃，PSR 4 取代

### PHP 7 新特性

- 标量和返回值类型声明
- null 合并运算符、组合比较符、匿名类等新语法
- AST（抽象语法树）
- 性能突破
- Zend引擎优化
- 函数调用机制

### PHP-FPM调优

PHP_FPM 是 PHP 的 FastCGI 的进程管理器。

Fast-CGI 是 CGI 增强版，WEB 服务器启动时同时启动 PHP 进程管理器，1个 master 进程和 N 个子进程（worker 进程），可以动态增减子进程。

```
pm = dynamic
pm.max_children = 8
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 4
pm.max_requests = 1000
```

### PHP Session 回收参数设置

```ini
session.gc_probability = 1
session.gc_divisor = 1000
session.gc_maxlifetime = 1440
```

## WEB安全

### 密码哈希

过去对密码进行 md5，sha1 等方式加 salt 加密后保存到数据库，如今这种方式已经不再安全。

现在更安全的方式是使用 password_hash 生成密码，用 password_verify 验证密码。

### XSS攻击

- 非持久型攻击：仅对当前页面造成影响
- 持久型攻击：存储在数据中，可能重复造成影响

防范：不相信用户的任何输入，对输入做过滤、转义处理。

### CSRF攻击

跨站请求伪造，挟持用户在当前已登录的 Web 应用程序上执行非本意的操作。

防范：服务器端每次请求生成一个 token，放在表单隐藏域，每次提交请求进行 token 验证。

### SQL注入和防范

- 过滤、转义参数
- 使用 PDO 操作数据库

## 计算机网络

### HTTP 2.0 特点

- **二进制分帧层**：所有性能增强的核心
- **多路复用**：同一个连接可同时发起多重请求-响应消息
- **标头压缩**
- **服务器推送**

### TCP三次握手和四次挥手

**三次握手**是为了防止已失效的连接请求报文段突然又传送到了服务端。

**四次挥手**：A 要先进入 TIME-WAIT 状态，等待 2MSL 时间后才进入 CLOSED 状态，为了保证服务器能收到客户端的确认应答。

### HTTPS连接过程

1. 浏览器发送加密规则给网站
2. 网站返回证书（包含公钥）
3. 浏览器生成随机密码并用公钥加密
4. 服务器解密并建立安全通道

### WebSocket连接过程

WebSocket 是基于 HTTP 协议的，完成握手后变成持久化连接。

1. 客户端发送 GET 请求，带有 Upgrade 头
2. 服务器响应 101 Switching Protocols
3. 连接建立，后续数据通过 WebSocket 帧传输
