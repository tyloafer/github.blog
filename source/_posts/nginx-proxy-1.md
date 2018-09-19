---
title: Nginx倒腾笔记（先导）：$1等变量的使用
tags:
  - Nginx
  - PHP
categories:
  - Nginx
date: '2018-05-18 19:33'
abbrlink: 59969
---

这篇文章的命名也是比较晦涩，这也充分显示了我的语文功底是多么的薄弱。简单解释一下吧，我们在使用nginx里面的正则做匹配及rewrite或proxy_pass的时候，会经常使用\$1,\$2这样的变量，这边文章就是做这个解释的。

其实我是准备写一个Nginx的系列的，而这个是我近期踩到的一个坑，又想记录下来，所以就变成了先导篇。

<!--more-->

#起因

先贴一下配置的例子吧：

~~~
location ~ ^/lixy/(.*(?<!\.php)$) {
    if ($request_uri ~ (create-group|set-avatar|change-avatar|set-group-avatar|resumable_upload|uploadFile\.php|webUpload\.php|resumable_upload_origin)) {
        access_log /data/logs/access.log file;
    }
    proxy_pass https://tyloafer.github.io/$1;
} 
~~~

这里的场景是，我在记录*request_body*的时候，如果是文件上传的话，就会把二进制流给记录下来，这个没必要骗的，所以，我便做了两个 *log_format* , 一个记录request_body, 另一个不记录 ，默认使用记录的

~~~
access_log /data/logs/access.log file;
~~~

这句话，也就是指定*access.log* 的 *log_format*, 然后再讲我的请求代理到另一台 服务器上

例：xxxxx.com/lixy/2018/05/18/nginx-proxy-1 => https://tyloafer.github.io/2018/05/18/nginx-proxy-1 , 我在请求的时候， location会匹配到lixy/后面的字符串并复制给$1这个变量，我在proxy_pass的时候，使用这个变量也就没什么问题了，后来发现，如果请求的接口地址包含上面的   (create-group|set-avatar|change-avatar|set-group-avatar|resumable_upload|uploadFile\.php|webUpload\.php|resumable_upload_origin) 这些字符的时候，还算正常，但是没有就会出问题。

# 调试

最简单的就是调试，打印参数了，这里我在nginx里面直接返回了$1,

```
location ~ ^/lixy/(.*(?<!\.php)$) {
    if ($request_uri ~ (create-group|set-avatar|change-avatar|set-group-avatar|resumable_upload|uploadFi    le\.php|webUpload\.php|resumable_upload_origin)) {
        access_log /data/logs/access.log file;
    }
    root /home/;   
    add_header  Content-Type 'text/html; charset=utf-8';
    return 200 $1;
} 
```

当请求xxxxx.com/lixy/2018/05/18/nginx-proxy-1 返回空白，

当请求xxxxx.com/lixy/2018/05/18/create-group 返回 create-group，

由此可见，在经过if的正则匹配的时候，同样会对$1进行赋值，这也就是出错的原因了，变量被重新赋值了。

# 解决



```
location ~ ^/lixy/(.*(?<!\.php)$) {
    set $path $1;
    if ($request_uri ~ (create-group|set-avatar|change-avatar|set-group-avatar|resumable_upload|uploadFi    le\.php|webUpload\.php|resumable_upload_origin)) {
        access_log /data/logs/access.log file;
    }
    root /home/;   
    proxy_pass https://tyloafer.github.io/$path;
} 
```

我们先给 location 匹配到的路由地址，赋值给$path，然后后面使用\$path就可以，\$1你们抢你们玩吧，其他的\$2 \$3 \$4 等参数，在使用的时候也需要注意。