---
title: 在centos7上安装V2ray 比shadowsock更科学的上网方式
tags:
  - 科学上网
  - V2ray
categories:
  - 科学上网
date: '2019-06-13 20:40'
abbrlink: 57533
---

近期ss的存活周期越来越短了，具体什么原因，咱也不知道，咱也不敢问，咱只能选择另一种更加科学的上网方式了，这里还是推荐一下  [Vultr家的VPS](https://www.vultr.com/?ref=7290537) ，换IP方便

<!--more-->

# 服务端安装

> Bash <(curl -L -s https://install.direct/go.sh)

执行完上面的脚本，就会输出如下安装的提示信息了，其中`PORT`和`UUID` 使我们后面需要用到的配置信息

![v2ray安装](http://note-1253518569.cossh.myqcloud.com/20190613145405.png)

然后通过`service v2ray start / stop /status`，就可以进行v2ray的启停了

接下来执行以下下面命令，可以看出配置文件为 `/etc/v2ray/config.json`， 里面也包含了 `UUID`和`PORT`等信息

> ps aux | grep v2ray

![](http://note-1253518569.cossh.myqcloud.com/20190620190741.png)



# 客户端配置

|         项目          |                              值                              |
| :-------------------: | :----------------------------------------------------------: |
|   主机/服务器/地址    |                           服务器ip                           |
|       端口 port       |                          图中的PORT                          |
|        用户ID         |                          图中的UUID                          |
|    额外ID AlterId     |                              64                              |
|   加密方式 security   |                             auto                             |
|       用户等级        |                              1                               |
| 网络/传输协议 network |                             Tcp                              |
|       加密方式        |                             none                             |
|          Mux          |                             开启                             |
|  远程路由/DNS (可选)  |                           1.1.1.1                            |
|         路由          | BifrsotV:绕过局域网和中国大陆地址与网站;V2RayN:参数设置-绕过中国大陆地址和ip(这一步的目的是直连国内网站，降低延迟) |



# 客户端

## 安卓

- [birfrostV](https://play.google.com/store/apps/details?id=com.github.dawndiy.bifrostv)   推荐这个，v2rayNG对于很多国内的网址，还是会走代理
- [v2rayNG](https://play.google.com/store/apps/details?id=com.v2ray.ang)

## Windows

以下两个都需要下载

- [v2ray-core](https://github.com/v2ray/v2ray-core/releases) ，点击链接，到github上选择自己的平台版本即可
- [v2rayN](https://github.com/2dust/v2rayN/releases/) 点击链接，下载里面的[v2rayN.zip](https://github.com/2dust/v2rayN/releases/download/2.29/v2rayN.zip)即可
- [v2rayW](https://github.com/Cenmrev/V2RayW/releases) 下载压缩包即可，这个程序会主动帮助下载 v2ray-core 所以更方便点，界面也比 v2rayN 看着舒服

# Mac

- [v2ray-core](https://github.com/v2ray/v2ray-core/releases) ，点击链接，到github上选择Mac的压缩包，下载解压就可以了

