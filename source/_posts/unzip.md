---
title: Windows上传ZIP文件到Linux下，解压乱码处理
tags:
  - Debug
  - Linux
categories:
  - Debug
date: '2018-02-03 16:04'
abbrlink: 35688
---



# 起因

作为码农，从windows上上传文件一般都会避免中文命名，都知道会有编码的问题，但是这个问题是不可逃避的。

近期，我给PM做一个产品原型的管理页面的时候，这个问题便不可避免的发生了。

<!--more-->

# 处理

Linux下解压ZIP文件，主要依靠`unzip`这个命令，通过`-h`查看帮助后，发现`unzip`有个·`-O -I`这两个参数（ps: 如果没有，请升级到最新版本）

对于这两个参数的解释如下

>   -O CHARSET  specify a character encoding for DOS, Windows and OS/2 archives
>   -I CHARSET  specify a character encoding for UNIX and other archives

由上可知，我们这里使用`-O`应该就可以避免编码问题了

~~~
unzip -O gbk filename.zip
~~~

# 扩展问题-PHP下处理

在Linux终端下，处理上述文件已经没有问题了，接下来肯定要考虑使用PHP来进行处理了。

php有一个ZipArchiv 解压缩类，但是我并没有找到针对编码的处理方式，所以无奈还是通过`exec`等命令来处理。**注意：** 如果你的php脚本是通过php-fpm来调度执行而不是php-cli直接执行的话，php-fpm的配置文件*php-fpm.conf*或*php-fpm.d/www.conf*里面会有环境变量的设置，这里要注意php-fpm的环境变量里面的*LANG*的值与Linux的设置的相同，不然还是会存在乱码的问题。
