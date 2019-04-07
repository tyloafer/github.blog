---
title: Linux终端科学上网
tags:
  - 科学上网
  - ShadowSock
categories:
  - ShadowSock
date: '2019-01-17 13:40'
abbrlink: 31170
---

对于经常在服务商使用git 和 composer 的我来说，实在受不了那几k的网速，但是毕竟使用较少，而且，还可以使用国内镜像源，但是近期要使用Google API，这个就有点头疼了。所以，废话不多话，直接上步骤，没什么技术含量。

<!--more-->

# Proxychains4版本

## 安装配置shadowsocks

### 安装

~~~
pip install shadowsocks
~~~

### 配置

编辑配置文件 `/etc/sslocal.json`，配置文件可以放在任意位置，启动的时候 指定对应位置即可

~~~
{
    "server":"xxx.xxx.xxx.xxx", // ss 服务器ip
    "server_port":xxx,  // ss 端口
    "local_address": "127.0.0.1", // 本机代理
    "local_port":1080, // 本机代理端口
    "password":"*******", // ss 密码
    "timeout":300,
    "method":"rc4-md5" // ss 加密方式
}
~~~

配置时请把后面的注释去掉

### 使用

~~~
启动：sslocal -c /etc/sslocal.json -d start

停止：sslocal -c /etc/sslocal.json -d stop

重启：sslocal -c /etc/sslocal.json -d restart
~~~

## 安装配置 Proxychains4

项目地址： https://github.com/rofl0r/proxychains-ng

编译安装即可

~~~
git clone https://github.com/rofl0r/proxychains-ng.git
cd proxychains-ng
./configure
make && make install
cp ./src/proxychains.conf /etc/proxychains.conf
~~~

修改 `/etc/proxychains.conf` 将 **socks4 127.0.0.1 9095** 改成 **socks5 127.0.0.1 1080** 

这里**127.0.0.1** 与 shadowsock配置文件中 **local_address** 对应 **1080** 与 **local_port** 对应即可

至此，安装完成，只要在使用的命令前 加上 `proxychains4 ` 即可实现科学上网

~~~
proxychains4 wget https://www.google.com
~~~

# HTTP_PROXY(推荐)

~~~
http:
export http_proxy=http://proxyAddress:port
export http_proxy=socks5://127.0.0.1:1087

https:
export https_proxy=https://proxyAddress:port
export https_proxy=socks5://127.0.0.1:1087

http && https
export all_proxy=http://proxyAddress:port
export all_proxy=socks5://127.0.0.1:1087
~~~

如果http代理需要验证的话

~~~
http_proxy=http://userName:password@proxyAddress:port
~~~

其他代理方式相同