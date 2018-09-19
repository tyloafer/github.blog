---
title: php配置xdebug实现内网本地调试远程服务器
tags:
  - PHP
  - Xdebug
  - Sublime
categories:
  - PHP
date: '2018-08-10 20:30'
abbrlink: 15205
---

对于一般的项目来说，`print_r() echo file_put_contents() exit` 这些函数调试就足够了，但是如果你调试的环境还有其他人，那就略胃疼了

服务器环境：centos7 php7.0

本地环境：windows php7.2（压缩包）

本地环境是公司内网，所以中间会借助xshell做一次tcp的转发

<!--more-->

# 服务器配置

## 安装xdebug

~~~
yum install php70w* --skip-broken
yum install php70w-pecl-xdebug
~~~

## 配置xdebug

修改xdebug的配置文件 `/etc/php.d/xdebug.ini`

~~~
; Enable xdebug extension module
zend_extension=/usr/lib64/php/modules/xdebug.so
xdebug.idekey = "PHPSTORM"
; 因为DBGp调试代理是可以做调试分发的，所以，定义一个IDEKEY，目的是让调试器知道，是不是发给自己的请求

xdebug.remote_enable = 1
; 设置为1的时候，会先参数链接到remote_host和remote_port指定的调试器端口，如果连接不上，就继续执行，类似设置>了0

xdebug.remote_mode = "req"
xdebug.remote_handler = "dbgp"
; 也就是调试走哪种协议，有老的PHP3协议，也有GDB协议，DBGP是当前的默认协议，也是当前主流支持的协议

xdebug.remote_connect_back = 0
; 如果为1，xdebug会通过$_SERVER[‘REMOTE_ADDR’]变量，向发起HTTP请求的客户端发起链接，和remote_host功能类似>，但是优先级比remote_host高，所以，设置了这个选项就会忽略remote_host，由于我的运行时服务器不能主动链接IDE所>在PC，所以不能开启自动回连模式(内网访问外网的网页，也不能开启)

xdebug.remote_host = "127.0.0.1"
; 默认是主机，如果服务器可以直接你的本地电脑，可以直接填写本机ip，我这里使用xshell做tcp转发

xdebug.remote_port = 9909
; 默认端口是9000，由于担心服务器端口冲突，我修改为9909，建议你没有端口冲突时不修改

xdebug.remote_autostart = 0
; 这个为1，会忽略COOKIE或者POST/GET中带的XDEBUG_SESSION参数，不管有没有，都会启动调试，所以，还是设置为0比较好，默认0
~~~

重启php-fpm

# Xshell TCP转发

首先连上你需要配置的服务器或者找到目标服务器的会话设置，然后右键找到`属性` ，然后找到` 连接 -> ssh -> 隧道 -> 添加`，按如图所示添加，上面配置目标服务器，下面配置本机

![配置属性](http://github-1253518569.cossh.myqcloud.com/xshell-1.png)

![配置属性](http://github-1253518569.cossh.myqcloud.com/xshell-2.png)

![tcp转发配置](http://github-1253518569.cossh.myqcloud.com/xshell-3.png)

配置完成后分别查看一下 目标服务器的9909 是否被监听

~~~
netstat -ano | grep 9909
~~~

# Sublime配置

## Xdebug配置

首先通过 ` package controller`安装 `xdebug client`

然后通过菜单栏的 `tool -> Xdebug-> Settings-default`， 可以找到xdebug的默认配置（简单摘取）

~~~
{
    "path_mapping": {

    },

    // Determine which URL to launch in the default web browser
    // when starting/stopping a session.
    "url": "",
    "ide_key": "sublime.xdebug",
    "host": "",

    // Which port number Sublime Text should listen
    // to connect with debugger engine.
    "port": 9000,

	.
	.
	.
	.
    "python_path" : "",

    // Show detailed log information about communication
    // between debugger engine and Sublime Text.
    // Log can be found at Packages/User/Xdebug.log
    "debug": false
}
~~~

参考系统的配置及解释，用户可以自行定义 `tool -> Xdebug-> Settings-User` ，但是用户一般不是只有一个项目，所以个人还是建议针对于每个 project 进行设置，可以通过菜单栏 `Project -> Edit Project`， 当然，你需要先创建一个project，这都是小事了。

附上个人project配置

~~~
{
	"folders":
	[
		{
			"path": "."
		}
	],
	"settings": {
		"xdebug": {
			"path_mapping": {
			    "/data/www/html/httpserver" : "D:/workplace/cloudserver"
			},
			"url": "http://121.xx.xx.xxx/httpserver/",
			"ide_key": "PHPSTORM",
			"host": "0.0.0.0",
			"port": 9050,
			"close_on_stop": true,
			"debug_layout" : {
			    "cols": [0.0, 0.5, 1.0],
			    "rows": [0.0, 0.7, 1.0],
			    "cells": [[0, 0, 2, 1], [0, 1, 1, 2], [1, 1, 2, 2]]
			},
			"python_path" : "C:\\Program Files\\Python35\\python.exe",
			"debug": true
		}
	}
}
~~~



## SFTP配置

sftp配置就不多废话了，参考另一篇博客[Sublime倒腾系列：配置sftp实现文件上传下载](https://tyloafer.github.io/posts/45096/)

# PHPStrom配置

## SFTP配置

`Tools->Deployment->Configuration`，在弹出的对话框里一次填入用户名，密码，

![phpstrom sft配置](http://github-1253518569.cossh.myqcloud.com/phpstrome-1.png)

然后右边旁边一个标签页 `Mappings`

![phpstrom-mapping](http://github-1253518569.cossh.myqcloud.com/phpstrome-29.png)

配置完成后，通过 `Tools—>Deployment—>Browse Remote Host` 看一下右侧是否会列出远程文件，显示绿色表名是对应上了

## Server配置

通过 `File -> Settings -> Languages & Frameworks -> PHP -> Servers`   新建一个Server,输入服务器的IP和端口，最关键的是，将本地工程文件夹和服务器上的文件夹对应起来，方便调试的时候找到源码：

![phpstrome server配置](http://github-1253518569.cossh.myqcloud.com/phpstrome-3.png)

## 配置Debug

通过`Run -> Edit Configurations -> PHP Remote Debug`，创建一个新的配置，Servers就选择刚才配置的，Ide Key设置为php.ini里面xdebug.idekey设置的， 这里就是 **PHPSTROME**

![phpstrom remote debug](http://github-1253518569.cossh.myqcloud.com/phpstrom-4.png)

## 调试（全部配置结束后测试）

![调试](http://github-1253518569.cossh.myqcloud.com/phpstrom-5.png)

# Postman配置

Postman 地址： [谷歌商店](https://chrome.google.com/webstore/detail/postman/agkicnjkbankpckaomnhfjglafjcegkk?hl=zh-CN)

Postman Interceptor 地址： [谷歌商店](https://chrome.google.com/webstore/detail/postman-interceptor/aicmkgpgakddgnaphhhpliifpcfhicfo?hl=zh-CN)

安装完成后，打开**Postman**， 在右上角开启 **Postman Interceptor** 即可设置cookie 进行请求了

![postman](http://github-1253518569.cossh.myqcloud.com/postman-1.png)

这时候，只要在模拟请求的时候设置一下cookie，在 **phpstrom** 或 **sublime** 设置一下断点就ok了

~~~
Cookie:XDEBUG_SESSION=PHPSTORM
~~~

![postman cookie设置](http://github-1253518569.cossh.myqcloud.com/postman-2.png)