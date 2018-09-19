---
title: Nginx倒腾笔记（六）：SSL_do_handshake() failed
tags:
  - Nginx
  - PHP
categories:
  - Nginx
date: '2018-09-3 19:33'
abbrlink: 34556
---

前几天在倒腾镜像站的时候，在代理ipv4的站点是ok的，但是代理ipv6的https站点的时候，发现一直返回502，也就是说明，nginx代理了，但是代理的时候，下游服务器没有给你正确的响应

<!--more-->

首先看一下原始配置

~~~
        location / {
                proxy_cache my_cache;
                proxy_set_header Accept-Encoding "none";
                proxy_pass https://m.cn.nytimes.com/;
                sub_filter 'https://m.cn.nytimes.com' 'https://www.xxx.com/';
                sub_filter 'd1f1eryiqyjs0r.cloudfront.net' 'www.xxx.com/d1f1eryiqyjs0r';
                sub_filter_once  off;
        }
        location ~ /d1f1eryiqyjs0r/(.*)$ {
                proxy_cache my_cache;
                proxy_set_header referer "https://cn.nytimes.com";
                proxy_pass https://d1f1eryiqyjs0r.cloudfront.net/$1;
        }
~~~

思路很简单，就是通过proxy_pass来代理到目标站点，获取内容后，将不能国内访问的内容的域名替换成自己的域名，后面加上不同的后缀来标识，然后再通过我们的服务器代理内容里面的超链，理论上出了繁琐外，没有什么技术难度，然而返回了502，这就有点不够意思了。

查看error.log后发现

~~~
2018/09/01 07:55:24 [error] 485#485: *6372 SSL_do_handshake() failed (SSL: error:14077410:SSL routines:SSL23_GET_SERVER_HELLO:sslv3 alert handshake failure) while SSL handshaking to upstream, client: 122.224.106.25, server: www.xxx.com, request: "GET / HTTP/1.1", upstream: "https://52.85.131.45:443/", host: "www.xxx.com"
~~~

SSL的问题，代理的时候加一下SSL处理

~~~
        location / {
                proxy_ssl_server_name on;
                proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
                proxy_cache my_cache;
                proxy_set_header Accept-Encoding "none";
                proxy_pass https://m.cn.nytimes.com/;
                sub_filter 'https://m.cn.nytimes.com' 'https://www.xxx.com/';
                sub_filter 'd1f1eryiqyjs0r.cloudfront.net' 'www.xxx.com/d1f1eryiqyjs0r';
                sub_filter_once  off;
        }
        location ~ /d1f1eryiqyjs0r/(.*)$ {
        		proxy_ssl_server_name on;
                proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
                proxy_cache my_cache;
                proxy_set_header referer "https://cn.nytimes.com";
                proxy_pass https://d1f1eryiqyjs0r.cloudfront.net/$1;
        }
~~~

综上，再次打开已经OK了。