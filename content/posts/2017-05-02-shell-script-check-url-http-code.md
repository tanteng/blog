---
title: Shell 脚本请求 URL 判断状态码
author: tanteng
type: post
date: 2017-05-02T03:05:16+00:00
url: /2017/05/shell-script-check-url-http-code/
categories:
 - 服务器

---
Shell 脚本请求 URL判断状态码是否异常并执行相关操作。

<!--more-->

示例如下：

```ini
#!/bin/bash

fileurl='./check_url.txt'
DATE_N=`date "+%Y-%m-%d %H:%M:%S"`
for chkurl in $(cat ${fileurl}) # ${}忽略空格
do
 # -o 输出内容到/dev/null; -s 静默方式 ；-w 定义显示输出格式；"%{http_code}" 在最后被检索的 HTTP(S) 页中被找到的数字的代码
 HTTP_CODE=`curl -o /dev/null -s --head -w "%{http_code}" "${chkurl}"`
 if [ ${HTTP_CODE} -ne 200 ]
 then
 if [ ${HTTP_CODE} == 404 ]
 then
 echo -e "[${DATE_N}]error-${HTTP_CODE}:" $chkurl >> check-result.txt
 sh /usr/local/bin/restart_php_mysql.sh >> check-result.txt
 fi
 echo -e "[${DATE_N}]error-${HTTP_CODE}:" $chkurl >> check-result.txt
 else
 echo -e [${DATE_N}]${HTTP_CODE}: $chkurl >> check-result.txt
 fi
done
```

放到 crontab 中每隔 5 分钟检测一次。
