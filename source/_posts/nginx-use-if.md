---
title: Nginx倒腾笔记（三）：条件判断及运算符
tags:
  - Nginx
categories:
  - Nginx
date: '2018-06-08 20:00'
abbrlink: 211
---
## 简介

| key  | value                    |
| ---- | ------------------------ |
| 语法:  | `if (condition) { ... }` |
| 默认值: | —                        |
| 上下文: | `server`, `location`     |

<!--more-->

## condition 

- 变量名；如果变量值为空或者是以“`0`”开始的字符串，则条件为假；
- 使用“`=`”和“`!=`”运算符比较变量和字符串；
- 使用“`~`”（大小写敏感）和“`~*`”（大小写不敏感）运算符匹配变量和正则表达式。正则表达式可以包含匹配组，匹配结果后续可以使用变量`$1`..`$9`引用。如果正则表达式中包含字符“`}`”或者“`;`”，整个表达式应该被包含在单引号或双引号的引用中。
- 使用“`-f`”和“`!-f`”运算符检查文件是否存在；
- 使用“`-d`”和“`!-d`”运算符检查目录是否存在；
- 使用“`-e`”和“`!-e`”运算符检查文件、目录或符号链接是否存在；
- 使用“`-x`”和“`!-x`”运算符检查可执行文件；



## 例

~~~shell
if ($http_user_agent ~ MSIE) {
    return 200 'IE >_>';
}

if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
    set $id $1;
}

if ($http_refer != 'test.tyloafer.cn') {
    return 403;
}

if ($vip) {
    limit_rate 10k;
}

if (!-e $request_uri) {
    rewrite /(.*)$ /index.php?r=$request_uri;
}
~~~

## 扩展（逻辑运算实现）

有时间为了实现需求，我们可能避免不了要使用逻辑运算， 但是nginx并未提供，所以，我们这时候只有通过 设置变量的值 来变相的实现逻辑运算了

需求

~~~nginx
if (!-e $request_uri && !-d $request_uri) {
    return 404;
}
~~~

实现

~~~nginx
set $flag  0;
if (!-e $request_uri) {
    set set $flag "${flag}1";
}
if (!-d $request_uri) {
    set $flag "${flag}1";
}
if ($flag = '011') {
    return 404;
}
~~~

