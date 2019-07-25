---
title: 基于Nginx和dnsmasq实现路由的动态DNS解析
tags:
  - Nginx
  - DNS
categories:
  - Nginx
  - DNS
date: '2019-07-25 12:04'
abbrlink: 8255
---

近期在跟前端同学联调的时候，出现了一点小插曲，我们两个的项目是同一个域名，然而，他的本地没有搭服务端的运行环境，所以就导致了前端需要的hosts文件和请求接口需要的hosts文件不一致的情况，基于这种情况，我就考虑是否可以通过nginx来匹配不同的路由，将请求打到不同的机器上面？

<!--more-->

我们都知道 nginx可以通过 `resolver` 命令来设置域名的DNS解析服务器，所以，如果路由是前端路由，就请求 自身的DNS服务器，将域名解析到本机，如果是接口，就通过我的DNS服务器或者正常的DNS服务器解析到其他的地址

按照上面的思路，我们就需要在本机搭建一个DNS服务器，这里我选用了 `dnsmasq`，没有什么理由，就是网上资料多🤪

# dnsmasq

## 安装

通过 `brew` 安装即可

~~~
brew install dnsmasq
~~~

## 配置

### 配置dnsmasq

配置文件在 `/usr/local/etc/dnsmasq.conf`

1. 设置 `listen-address`: listen-address=127.0.0.1 或者其他需要监听的内网或公网ip地址
2. 取消注释 `strict-order`
3. 如果不想使用 `/etc/hosts` 和 `/etc/resolve.conf` 可以修改如下
   1. 配置 `resolv-file`: resolv-file=/etc/resolv.dnsmasq.conf 文件名字随意
   2. 配置 `addn-hosts`: addn-hosts=/etc/dnsmasq.hosts 文件名字随意



### 配置resolv

这里的resolv指的是 上面配置中 `resolv-file` 所对应的文件

~~~
nameserver 127.0.0.1 # 放在第一位，会首先去这个地址请求dns解析
nameserver 114.114.114.114
~~~

### 配置hosts

把 需要解析的域名 添加进去即可，例

~~~
127.0.0.1 test.tyloafer.cn
~~~

## 启停控制

- 启动： brew services start dnsmasq
- 停止： brew services stop dnsmasq
- 重启： brew services restart dnsmasq



# Nginx

~~~nginx
server {
    listen       80;
    server_name  test.tyloafer.cn;
    root html;
    add_header Content-Type 'text/html; charset=utf-8';
    location ~ /local {
        resolver 127.0.0.1 ipv6=off;
        proxy_pass http://test.tyloafer.cn/1.html$is_args$args;
    }
    location ~ /remote {
        resolver 114.114.114.114;
        proxy_pass http://test.tyloafer.cn/1.html$is_args$args;
    }
    location ~ /1.html {
        add_header Content-Type 'text/html; charset=utf-8';
        return 200 "ok";
    }
}
~~~



请求流程分析：

> http://test.tyloafer.cn/local

1. 首先 根据本机的dns解析，会解析到127.0.0.1，也就是本机
2. 然后 nginx接收到这个请求，会匹配到 `location ~ /local` 这条规则，然后 调用127.0.0.1 的DNS服务器，也就是上面搭建的，将请求转发到本机的 `/1.html` 路由下
3. nginx收到 proxy_pass的请求，匹配到 `location ~ /1.html` ，然后输出 "ok"

> http://test.tyloafer.cn/remote

1. 首先 根据本机的dns解析，会解析到127.0.0.1，也就是本机
2. 然后 nginx接收到这个请求，会匹配到 `location ~ /remote` 这条规则，然后 调用114.114.114.114 的DNS服务器，这里会进行正常的DNS解析，找到真实的服务器地址
3. 目标服务器的nginx收到 proxy_pass的请求，进行相应的处理（1.html 在我的目标服务器上没有，所以会返回404）