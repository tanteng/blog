---
title: Golang 统计字符串字数
date: 2018-05-02T02:52:22+00:00
url: /2018/05/golang-string-length/
categories:
 - Develop
 - Go
tags:
 - Golang
 - Unicode

---

比如新浪微博发微博的输入框有一个已输入字数的统计，它的规则推测是：汉字和中文标点算 1 个字数，英文和其他符号算 0.5 个字数。不足 1 个字算 1 个。大家可以去微博体验一下计算方式。

<!--more-->

golang 可以使用正则和 unicode 包的方法判断。

以下函数 GetStrLength 返回输入的字符串的字数，每个汉字和中文标点算 1 个字数，英文和其他字符算半个字数，不足 1 个字算 1 个。

```golang
// GetStrLength 返回输入的字符串的字数，汉字和中文标点算 1 个字数，英文和其他字符 2 个算 1 个字数，不足 1 个算 1个
func GetStrLength(str string) float64 {
 var total float64

 reg := regexp.MustCompile("/·|，|。|《|》|‘|’|”|“|；|：|【|】|？|（|）|、/")

 for _, r := range str {
 if unicode.Is(unicode.Scripts["Han"], r) || reg.Match([]byte(string(r))) {
 total = total + 1
 } else {
 total = total + 0.5
 }
 }

 return math.Ceil(total)
}
```
