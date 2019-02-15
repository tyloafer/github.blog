---
title: SeLinux引发的问题小记
tags:
  - Linux
categories:
  - Linux
date: '2019-01-07 13:40'
abbrlink: 15998
---

以前自己在部署服务器的时候，忘了遇到了什么坑，导致后来习惯性把新服务器上的 **SeLinux** 服务直接关闭，今天处理了一台不是我部署的服务器，又踩了两个坑，这里做一下记录，后面有时间还是要好好了解一下这块。

# FastCGI sent in stderr: "Primary script unknown" while reading response header from upstream

>  这个问题一般是 **SCRIPT_FILENAME**   这个变量没有设置好，但是打印出来其实是正确的，也就是证明nginx的配置时没有问题的，可以考虑是 **SeLinux** 的原因

处理： chcon -R -t httpd_sys_content_t /path/to/webroot

# chmod 777 后依旧 failed to open stream: Permission denied

处理： 

~~~
setenforce Permissive
~~~

