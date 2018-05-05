---
title: PHP获取环境变量
tags: 
    - Nginx
    - Linux
    - PHP
    - php-fpm
categories:
    - PHP
date: 2018-02-05 17:35
---

# 起因

使用PHP的exec等函数与：Linux进行交互是很常见的方式，但是有时候发现，在终端里面通过命令行模式运行的代码可行，放到网站上去访问就出问题了，这里主要是因为在通过Nginx调起PHP-FPM的时候，会存在一些参数的配置问题下面就简单介绍一下这两种方式。

<!--more-->

# 解决-PHP-FPM模式 

1. 通过Nginx传递

   如在nginx的配置里设置： 
   `fastcgi_param  ENV_XXX  123456;` 
   每次页面请求nginx都会将此变量传递给php，php可以通过getenv函数或$_SERVER全局变量获得。 

2. 通过PHP-FPM配置传递

   ~~~
   ; Clear environment in FPM workers
   ; Prevents arbitrary environment variables from reaching FPM worker processes
   ; by clearing the environment in workers before env vars specified in this
   ; pool configuration are added.
   ; Setting to "no" will make all environment variables available to PHP code
   ; via getenv(), _ENV and _SERVER.
   ; Default Value: yes 
   ; clear_env = no

   ; Pass environment variables like LD_LIBRARY_PATH. All $VARIABLEs are taken from
   ; the current environment.
   ; Default Value: clean env 
   ;env[HOSTNAME] = $HOSTNAME
   env[PATH] = /usr/local/bin:/usr/bin:/bin:/usr/local/sbin
   ;env[TMP] = /tmp
   ;env[TMPDIR] = /tmp
   ;env[TEMP] = /tmp
   ~~~

   上面是php-fpm.conf（包括php-fpm.d/www.conf）里面关于环境变量的配置，在里面有一个`clear_env`的参数配置，这个默认是*yes*，而他的含义就是会把Linux上设置的环境变量给清空，这样的设置也是基于安全角度来考虑，我们此时把这个值设置为*no*即可，即

   ~~~
   clear_env = no
   ~~~

   在下面的配置中有一个`env[PATH]`的参数配置，这里也可以满足我们设置环境变量的需求

   ​

   # 解决-命令行模式 

   命令行模式限制较少，可以通过getenv函数或$_SERVER全局变量获取对当前执行用户有效的系统环境变量，同样要注意sudo的限制