---
title: "PHP PDO prepare 预处理详解：如何安全地防止 SQL 注入"
date: 2012-09-30T04:03:39+08:00
draft: false
tags: ['php', 'pdo', 'sql安全', '索引优化']
categories: ['tech']
description: "详解 PHP PDO prepare 预处理语句的正确使用方式，以及如何真正防止 SQL 注入"
---

PDO（PHP Data Objects）是 PHP 数据库操作的统一接口，提供了预处理语句（Prepared Statements）功能，是防止 SQL 注入的最佳实践。

<!--more-->

## 什么是预处理语句？

预处理语句是一种编译过的 SQL 语句模板，可以使用不同的变量参数重复执行。

**预处理语句的两个主要优点：**

1. **性能提升**：查询只需要被解析一次，但可以使用相同或不同参数执行多次。对于复杂查询，可以避免重复分析、编译、优化。
2. **安全防护**：传给预处理语句的参数不需要使用引号，底层驱动会自动处理，可以有效防止 SQL 注入。

## 基本用法

### 命名参数形式

```php
$sql = 'SELECT name, colour, calories
FROM fruit
WHERE calories < :calories AND colour = :colour';

$sth = $dbh->prepare($sql, array(PDO::ATTR_CURSOR => PDO::CURSOR_FWDONLY));

// 执行一次
$sth->execute(array(':calories' => 150, ':colour' => 'red'));
$red = $sth->fetchAll();

// 再次执行，不同参数
$sth->execute(array(':calories' => 175, ':colour' => 'yellow'));
$yellow = $sth->fetchAll();
```

### 问号参数形式

```php
$sth = $dbh->prepare('SELECT name, colour, calories
FROM fruit
WHERE calories < ? AND colour = ?');

$sth->execute(array(150, 'red'));
$red = $sth->fetchAll();

$sth->execute(array(175, 'yellow'));
$yellow = $sth->fetchAll();
```

### 使用 bindParam 绑定参数

```php
$stmt = $dbh->prepare('SELECT * FROM users WHERE email = :email');
$email = 'test@example.com';
$stmt->bindParam(':email', $email);
$stmt->execute();
```

## PDO 防止 SQL 注入的原理

PDO 预处理语句的核心原理是**参数绑定**：

1. SQL 语句结构先发送给数据库服务器，进行语法分析和编译
2. 参数数据通过**独立的二进制协议**传输，不与 SQL 语句混合
3. 数据库将参数作为**纯数据**处理，不会被解释为 SQL 代码

```
传统拼接：SELECT * FROM users WHERE id = '1 OR 1=1'  ← 语句已被污染

预处理绑定：SELECT * FROM users WHERE id = ? 
           ↓ 参数单独传输
           [ '1 OR 1=1' ]  ← 仅作为数据
```

## 重要注意事项：并非 100% 安全！

PDO 预处理并不能 100% 防止 SQL 注入，存在以下需要注意的边缘情况：

### 1. 字符集攻击（Charset Attack）

当使用 `SET NAMES gbk` 等字符集设置时，可能存在注入风险：

```php
// 危险示例
$pdo->query('SET NAMES gbk');
$var = "\xbf\x27 OR 1=1 /*";
$query = 'SELECT * FROM test WHERE name = ? LIMIT 1';
$stmt = $pdo->prepare($query);
$stmt->execute(array($var)); // 可能被注入！
```

**防御方法：**

```php
// 方法1：使用正确的字符集连接
$pdo = new PDO('mysql:host=localhost;dbname=test;charset=utf8mb4', 'user', 'pass');

// 方法2：禁用模拟预处理
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
```

### 2. 模拟预处理（Emulate Prepares）

PDO 默认使用模拟预处理，可能存在安全风险：

```php
// 禁用模拟预处理，强制使用真实预处理
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
```

### 3. 动态构建的查询部分

如果查询的其他部分由未转义的输入构建，仍存在注入风险：

```php
// 危险：列名动态拼接
$order = $_GET['order']; // 可能被注入！
$stmt = $pdo->prepare("SELECT * FROM users ORDER BY {$order}");
```

## 最佳实践

### 1. 正确初始化 PDO 连接

```php
$options = [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES => false,  // 禁用模拟预处理
];

$pdo = new PDO(
    'mysql:host=localhost;dbname=test;charset=utf8mb4',
    'username',
    'password',
    $options
);
```

### 2. 始终使用参数绑定

```php
// ✅ 正确
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([$userId]);

// ❌ 错误
$stmt = $pdo->query("SELECT * FROM users WHERE id = $userId");
```

### 3. 验证输入

预处理语句不是万能的，仍需验证输入格式和类型：

```php
$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
if ($id === false) {
    throw new InvalidArgumentException('Invalid ID');
}
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([$id]);
```

### 4. 限制权限

数据库用户应该只拥有应用所需的最小权限，不要使用 root 用户运行 PHP 应用。

## 总结

| 要点 | 说明 |
|------|------|
| 预处理语句 | 性能提升 + 防止注入 |
| 参数绑定 | 核心安全机制 |
| 字符集 | 使用 utf8mb4，正确设置 |
| ATTR_EMULATE_PREPARES | 建议设为 false |
| 输入验证 | 仍然必要 |

> 预处理语句是防止 SQL 注入的重要手段，但安全是一个体系工程，需要将所有环节都做对才能真正安全。
