---
title: 在Vultr（centos7）上安装shadowsock及Google BBR实现全速翻墙
tags:
  - 科学上网
  - ShadowSock
categories:
  - 科学上网
date: '2018-06-23 13:40'
abbrlink: 561
---

对于程序猿来说，百度就是一个坑的存在，找一个问题，前面几页都是抄袭、雷同的问题，还有若干的百度经验，但是，对于近期的墙是越来越厚了，各种ss账号都失效了，无奈开始自己动手搭梯子吧。通过网上各种对比后，最后选了了[ Vultr](https://www.vultr.com/?ref=7290537) , 安装Google BBR后基本可以满速翻墙，而且，最强大的是，可以更换IP，现在最便宜的套餐 2.5$每月，500G的流量，也是足够了

> [Vultr官网注册地址](https://www.vultr.com/?ref=7540935)

<!--more-->

下面开始介绍安装配置过程，当然不仅局限于 Vultr的服务器，CentOS7的都可以参考。

# 安装pip

~~~
yum install python python-pip
pip install --upgrade pip
~~~

# 安装配置shadowsock

## 安装shadowsock

~~~shell
pip install shadowsocks
~~~
这样就安装完成了shadowsock

## 配置shadowsock

首先要创建配置文件，然后shadowsock用这个配置文件启动即可

个人习惯将配置文件放在 `/etc` 目录下

所以，这里首先创建文件 `/etc/shadowsocks.json` ，并添加一下内容

~~~
{
      "server":"0.0.0.0",
      "server_port":2082,
      "local_port":1080,
      "password":"yourpass",
      "timeout":600,
      "method":"rc4-md5"
 }
~~~

如果想要创建多用户的， 可以使用下面的配置文件格式

~~~
#### 多用户版本
{
    "server":"0.0.0.0",
    "local_port":1080,
    "port_password":{
         "8989":"password0",
         "9001":"password1",
         "9002":"password2",
         "9003":"password3",
         "9004":"password4"
    },
    "timeout":300,
    "method":"aes-256-cfb"
}
~~~

说明：

* `method`为加密方法，可选`aes-128-cfb, aes-192-cfb, aes-256-cfb, bf-cfb, cast5-cfb, des-cfb, rc4-md5, chacha20, salsa20, rc4, table`
* `server_port`  为服务监听端口， 这个需要在 vultr或者服务器管理界面开启
* `port_password` 为多用户的时候，配置的 port: pass 的json串

## 配置自启动

新建启动脚本文件`/etc/systemd/system/shadowsocks.service`，内容如下：

~~~
[Unit]
Description=Shadowsocks

[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json

[Install]
WantedBy=multi-user.target
~~~

执行以下命令启动 shadowsocks 服务：

~~~
systemctl enable shadowsocks  # 启用服务
systemctl start shadowsocks   # 启动服务
systemctl stop shadowsocks   # 停止服务
~~~

注： 如果使用多用户的模式，有可能无法使用 system 来启动，可以使用下面的命令来启动

~~~
/usr/bin/ssserver -c /etc/shadowsocks.json -d start # 启动
/usr/bin/ssserver -c /etc/shadowsocks.json -d stop  # 停止
echo "/usr/bin/ssserver -c /etc/shadowsocks.json -d start" >> /etc/rc.local # 添加到开机自启动
~~~

# 安装BBR加速

> **注意: **需要内核4.9及以上版本，可使用`uname -r`查看。

最简单的方法就是使用Google BBR一键安装脚本。

1. 获取脚本并执行(root)

   ~~~
   wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
   sh ./bbr.sh
   ~~~

2. 查看内核版本是否 > 4.9 

   ~~~
   uname -r
   ~~~

3. 执行 `sysctl net.ipv4.tcp_available_congestion_control` ， 查看是否返回

   ~~~
   net.ipv4.tcp_available_congestion_control = reno cubic bbr
   ~~~

4. 执行`sysctl net.core.default_qdisc` ， 查看是否返回

   ~~~
   net.core.default_qdisc = fq
   ~~~

5. 执行 `lsmod | grep bbr`， 返回值有tcp_bbr 即已启动

安装完成后，脚本会提示需要重启 VPS，输入 y 并回车后重启。



# 附

近期的服务器老是被尝试暴力破解，然后ip被封，提供一下脚本进行处理

~~~
#! /bin/bash
cat /var/log/secure|awk '/Failed/{print $(NF-3)}'|sort|uniq -c|awk '{print $2"="$1;}' > /home/sxdgy/lastb
for i in `cat  /home/sxdgy/lastb`
do
  IP=`echo $i |awk -F= '{print $1}'`
  NUM=`echo $i|awk -F= '{print $2}'`
  if [ ${#NUM} -gt 1 ]; then
    grep $IP /etc/hosts.deny > /dev/null
    if [ $? -gt 0 ];then
      echo "sshd:$IP:deny" >> /etc/hosts.deny
    fi
  fi
done
~~~

