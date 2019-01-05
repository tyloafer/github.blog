---
title: Nginx倒腾笔记（七）：Nginx限制访问频率
tags:
  - Nginx
categories:
  - Nginx
date: '2018-10-13 15:31'
abbrlink: 34870
---

面对而已的DDOS攻击是一种很让人头疼的问题，其中CC攻击是DDOS的一种，也是一种常见的网站攻击方法，通过有限的IP不断的去请求对方服务器，造成对方服务器资源耗尽直至宕机。

而通过Nginx的**HttpLimitReqModul**和**HttpLimitZoneModule** 来限制同一IP在同一时间段内的访问次数来降低CC攻击带来的危害

<!--more-->

下面是一种简单的频率限制在nginx里面的配置

```
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

    ...

    server {

        ...

        location /search/ {
            limit_req zone=one burst=5;
        }
```

# limit_req_zone

limit_req_zone的使用规则是 

**limit_req_zone** *key* zone=*name*:*size* rate=*rate*

`limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;`

这一段是定义了一个名为*one* 的zone空间，空间大小为10m， 以$binary_remote_addr 为key,限制平均每秒的请求为1个，1M能存储16000个状态，rate的值必须为整数

# limit_req

limit_req的使用规则是

**limit_req** zone=*name* [burst=*number*] \[nodelay\];

`limit_req zone=one burst=5;`

zone: zone是指使用我们上面定义的zone，这个很好理解

burst: burst其实是一个桶的概念，以我们的设置为例，我们设置的是1request/s，如果一次性来了3个request，nginx会在第一秒处理一个请求，另外两个请求放到这个**burst**桶里，然后下一秒先处理桶里的请求，后面来的请求继续放进桶里，如果桶满了，这时候的请求就会返回503了

nodelay: 如果不希望在请求受限的情况下延迟过多的请求，可以使用这个参数，同时配合burst，他会在第一秒的时候，处理限制的请求数和burst里面的请求数