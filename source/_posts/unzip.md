---
title: Windows上传ZIP文件到Linux下，解压乱码处理
tags: 
    - Debug
    - Linux
categories:
    - Debug
date: 2018-02-03 16:04
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

php有一个ZipArchiv 解压缩类，但是我并没有找到针对编码的处理方式，所以便想通过`exec`来处理。但是，php cli处理是没有问题的，php-fpm调度php处理的时候问题又发生了。

# 扩展处理-PHP下处理

通过自我测试后，暂时找到两种解决方案

## popen

代码如下：

~~~
	$fp = popen('/usr/bin/unzip -O GBK -a ' . $filename, 'r');
	pclose($fp);
~~~

## passthru

passthru 会自动将结果输出，这并不是我们想要的，可通过*缓冲区*进行处理

代码如下：

~~~
ob_start()
passthru ('/usr/bin/unzip -O GBK -a ' . $filename);
on_clean();
~~~

