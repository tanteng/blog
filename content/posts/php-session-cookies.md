---
title: "深入理解 PHP 中 session 和 cookies 的联系"
date: 2012-09-26T01:46:43+08:00
draft: false
tags: ['php']
categories: ['tech']
description: "深入理解 PHP 中 session 和 cookies 的联系"
---

1. session概念

2. http协议与状态保持

3. 理解cookie

4. php中session的生成机制

5. php中session的过期回收机制

6. php中session的客户端存储机制

<!--more-->

# 1. session概念

在web服务器蓬勃发展的时代，session在web开发语境下的语义是指一类用来在客户端与服务器之间保持状态的解决方案。

# 2. http协议与状态保持

http协议本身是无状态的，客户端只需要简单的向服务器请求下载某些文件，无论是客户端还是服务器都没有必要纪录彼此过去的行为，每一次请求之间都是独立的。

然而人们很快发现如果能够提供一些按需生成的动态信息会使web变得更加有用，就像给有线电视加上点播功能一样。这种需求一方面迫使HTML逐步添加了表单、脚本、DOM等客户端行为，另一方面在服务器端则出现了CGI规范以响应客户端的动态请求，作为传输载体的HTTP协议也添加了文件上载、cookie这些特性。其中cookie的作用就是为了解决HTTP协议无状态的缺陷所作出的努力。至于后来出现的session机制则是又一种在客户端与服务器之间保持状态的解决方案。

session机制可能需要借助于cookie机制来达到保存标识的目的。所以有必要了解下cookie。

# 3. 理解cookie

cookie分发是通过扩展HTTP协议来实现的，服务器通过在HTTP的响应头中加上一行特殊的指示以提示浏览器按照指示生成相应的cookie。然而纯粹的客户端脚本如JavaScript或者VBScript也可以生成cookie。

而cookie 的使用是由浏览器按照一定的原则在后台自动发送给服务器的。浏览器检查所有存储的cookie，如果某个cookie所声明的作用范围大于等于将要请求的资源所在的位置，则把该cookie附在请求资源的HTTP请求头上发送给服务器。

cookie的内容主要包括：名字，值，过期时间，路径和域。

其中域可以指定某一个域比如.google.com，相当于总店招牌，比如宝洁公司，也可以指定一个域下的具体某台机器比如www.google.com或者froogle.google.com，可以用飘柔来做比。路径就是跟在域名后面的URL路径，比如/或者/foo等等，可以用某飘柔专柜做比。

路径与域合在一起就构成了cookie的作用范围。

如果不设置过期时间，则表示这个cookie的生命期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了。这种生命期为浏览器会话期的 cookie被称为会话cookie。会话cookie一般不存储在硬盘上而是保存在内存里，当然这种行为并不是规范规定的。如果设置了过期时间，浏览器就会把cookie保存到硬盘上，关闭后再次打开浏览器，这些cookie仍然有效直到超过设定的过期时间

存储在硬盘上的cookie 不可以在不同的浏览器间共享，可以在同一浏览器的不同进程间共享，比如两个IE窗口。

这是因为每中浏览器存储cookie的位置不一样，比如

Chrome下的cookie放在：

C:\Users\sharexie\AppData\Local\Google\Chrome\User Data\Default\Cache

Firefox下的cookie放在：

C:\Users\sharexie\AppData\Roaming\Mozilla\Firefox\Profiles\tq2hit6m.default\cookies.sqlite （倒数第二个文件名是随机的文件名字）

Ie下的cookie放在：

C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Cookies

# 4. php中session的生成机制

我们先来分析一下PHP中是怎么生成一个session的。设计出session的目的是保持每一个用户的各种状态来弥补HTTP协议的不足(无状态)。session是保存在服务器的，既然它用于保持每一个用户的状态那它利用什么来区别用户的呢？这个时候就得借助cookie了。当我们在代码中调用session_start();时，PHP会同时往SESSION的存放目录(默认为/tmp/)和客户端的cookie目录各生成一个文件。session文件名称像这样：

格式为sess_{SESSIONID} ，这时session文件中没有任何内容，当我们在session_start();添加了这两行代码：

```php
$_SESSION['name'] = 'sharexie';
$_SESSION['webUlr'] = 'www.qq.com';
```

这时文件就有内容了：

```
name|s:8:"sharexie";webUlr|s:10:"www.qq.com";
```

# 5. php中session的过期回收机制

我们明白了session的生成及工作原理，发现在session目录下会有许多session文件。当然这些文件一定不是永远存在的，PHP一定提供了一种过期回收机制。在php.ini中session.gc_maxlifetime为session设置了生存时间(默认为1440s)。如果session文件的最后更新时间到现在超过了生存时间，这个session文件就被认为是过期的了。在下一次session回收的时候就会被删除。

# 6. php中session的客户端存储机制

由于cookie可以被人为的禁止，必须有其他机制以便在cookie被禁止时仍然能够把session id传递回服务器。解决办法有：

1、URL重写，就是把session id直接附加在URL路径的后面

2、作为查询字符串附加在URL后面

3、表单隐藏字段
