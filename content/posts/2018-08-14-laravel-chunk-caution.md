---
title: "Laravel chunk 使用注意事项"
date: 2018-08-14T15:39:12+00:00
url: /2018/08/laravel-chunk-caution/
categories:
 - tech

---
使用 Laravel 的 chunk 可以用来优化大结果集的查询，提供分块处理数据的方法，但是如下的例子就会有问题：

```php
User::where('approved', 0)->chunk(100, function ($users) {
    foreach ($users as $user) {
        $user->update(['approved' => 1]);
    }
});
```

<!--more-->

原因在于第一次查询：

```sql
select * from users where approved = 0 limit 100 offset 0;
```

update 这一批数据的 approved 为 1 之后，再看第二次查询：

```sql
select * from users where approved = 0 limit 100 offset 100;
```

**这个时候因为有 where approved = 0 条件并且偏移量从 100 开始，这样其实就漏掉了 100 条 approved 为 0 的数据。**

所以，我们要避免使用 chunk 的时候，更改和过滤条件的字段的值。
