---
title: Nginx倒腾笔记（六）：Nginx tcp/udp转发
tags:
  - Nginx
categories:
  - Nginx
date: '2018-07-21 19:00'
abbrlink: 27510
---

前两天给部门做了个数据库的主备用于数据分析，但是后来运维不建议直连备份服务器的3306端口，然后就很尴尬了，然后他们就从程序开始入手，脚本处理binlog，导入导出and so on，从我个人角度来说，我是不喜欢这种方式极度曲线救国的方式的，然后我就没事测试了一下Nginx到tcp转发，当然也只能做个人的小实验了。

<!--more-->

# 配置Nginx

Nginx的stream模块不仅支持TCP转发，也支持UDP，同时还可以通过socket转发，也是很强大的了。

配置文件配置如下

~~~
stream {
	server {
		listen 1008;
		# listen 1008 udp; 通过后面添加udp，可以进行udp的转发
		proxy_pass 127.0.0.1:3306;
		# proxy_pass unix:/var/lib/mysql/mysql.sock; 将数据转发给socket处理
	}
}
~~~

# 测试

我们通过`1008` 端口进行连接数据库尝试一下

~~~
➜  httpserver master ✗ mysql -uxxxx -pxxxxx -P1008 
mysql: [Warning] Using a password on the command line interface can be insecure.
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5399
Server version: 5.7.22-log MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> quit
~~~

尝试ok

# 进阶

既然nginx可以进行tcp和udp的转发，那我是否可以代理shadowsock呢？

配置如下

~~~
stream {
    server {
        listen 80;
        listen 80 udp;
        proxy_pass 127.0.0.1:xxxx;
    }
}
~~~

最终，并没有尝试成功，后期有时间找朋友一起抓下包分析一下，暂时就先倒腾到这了