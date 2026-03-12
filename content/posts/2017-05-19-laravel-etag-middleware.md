---
title: 在 Laravel 中设置 Etag 缓存
author: tanteng
type: post
date: 2017-05-19T04:51:35+00:00
url: /2017/05/laravel-etag-middleware/
categories:
 - Develop
 - Laravel
 - PHP

---
本文介绍浏览器缓存 Etag 的概念，和客户端服务器如何生成和比较 Etag 的过程，以及使用 Laravel 中间件的示例。

<!--more-->

### 什么是"ETag"？

Etag 是一种标识，一般附带在响应头部中，值是页面内容的哈希值，用来判断资源（页面，json，xml）有没有修改，如果没有修改，就返回 304 状态码，有修改则生成新的 Etag 值。

浏览器根据状态码判断是否缓存过期。

服务器生成 Etag，和客户端保存 Etag ，并在请求中附带 Etag 的过程如下：

 1. 客户端请求一个页面（A）。
 2. 服务器返回页面A，并在给A加上一个ETag。
 3. 客户端展现该页面，并将页面连同ETag一起缓存。
 4. 客户再次请求页面A，并将上次请求时服务器返回的ETag一起传递给服务器。
 5. 服务器检查该ETag，并判断出该页面自上次客户端请求之后还未被修改，直接返回响应304（未修改——Not Modified）和一个空的响应体。

### 用法示例

在典型用法中，当一个URL被请求，Web服务器会返回资源和其相应的ETag值，它会被放置在HTTP的"ETag"字段中：

```
ETag: "686897696a7c876b7e"
```

然后，客户端可以决定是否缓存这个资源和它的ETag。以后，如果客户端想再次请求相同的URL，将会发送一个包含已保存的ETag和"If-None-Match"字段的请求。

```
If-None-Match: "686897696a7c876b7e"
```

客户端请求之后，服务器可能会比较客户端的ETag和当前版本资源的ETag。如果ETag值匹配，这就意味着资源没有改变，服务器便会发送回一个极短的响应，包含HTTP "304 未修改"的状态。304状态告诉客户端，它的缓存版本是最新的，并应该使用它。

然而，如果ETag的值不匹配，这就意味着资源很可能发生了变化，那么，一个完整的响应就会被返回，包括资源的内容，就好像ETag没有被使用。这种情况下，客户端可以用新返回的资源和新的ETag替代先前的缓存版本。

ETag值可用于网页监视系统。大多数站点没有为网页设置ETag头信息，这一事实阻碍了高效的网页监测。当Web监视器不知道网站内容是否发生变化的时候，它不得不被检索和分析所有的内容，这不仅占用发布者的计算资源，还有订阅者的。

### Laravel Etag 中间件

这里有一个 Laravel 的中间件，可以给 Laravel 设置 Etag 缓存：

```php
<?php namespace App\Http\Middleware;
use Closure;
class ETagMiddleware {
 /**
 * Implement Etag support
 *
 * @param \Illuminate\Http\Request $request
 * @param \Closure $next
 * @return mixed
 */
 public function handle($request, Closure $next)
 {
 // Get response
 $response = $next($request);
 // If this was a GET request...
 if ($request->isMethod('get')) {
 // Generate Etag
 $etag = md5($response->getContent());
 $requestEtag = str_replace('"', '', $request->getETags());
 // Check to see if Etag has changed
 if($requestEtag && $requestEtag[0] == $etag) {
 $response->setNotModified();
 }
 // Set Etag
 $response->setEtag($etag);
 }
 // Send response
 return $response;
 }
}
```

中间件将页面内容进行 md5 得到哈希值，然后跟客户端带过来的 Etag 进行对比。

### Etag 的弊端

不过ETag/If-None-Match这点功能实在是个鸡肋，首先，Server端的资源不大可能Roll Back，更重要的是，有可能造成Client Performance下降。对于只有一个Server的网站，没什么问题，但是现在稍微上点规模的网站都需要Scale Out，也就是说需要前端一个Load Balancer，后面接多台Server来处理请求，俗称Cluster，既然是Cluster，**那么每个请求到底返回什么结果应该和分配到哪个 Server无关**，不过这个ETag可能就坏事了。假如用户的第一次请求分配给Server A，返回"ETag: "abcdefg1234:0001""，但是第二次请求分配给了Server B，Server B上这个资源和Server A上的一模一样，但是计算出这个资源的ETag是"abcdefg1234:0002"，这下麻烦了，虽然内容一样，但是ETag不匹配，还是浪费了带宽把资源发送了一遍，冤枉啊！而事实上，不同Server上的ETag很有可能不同，对于Apache，ETag的计算考虑了inode，对于 IIS,ETag考虑了metabase的修改版本，要保证不同server上的这些信息一致，有点小难。不过不是有Last-Modified/If- Not-Modified吗？Server端看到If-Modified-Since，对照一下时间对得上，不管If-None-Match，可以直接发回304(Not Modified)呀，很不幸，RFC2616对这种情况做了规定，如果既有If-None-Match又有If-Modified-Since，除非两者不冲突，不然不会返回304。
