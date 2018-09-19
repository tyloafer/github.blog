---
title: Linux下mail发送邮件8015错误
tags:
  - mail
  - Linux
categories:
  - Linux
date: '2018-01-25 20:05'
abbrlink: 58415
---

# 起因

在上上篇文章[Linux下使用mail发送邮件](https://tyloafer.github.io/2017/12/03/mail/)中介绍了如何在linux下通过mail发送邮件，然而在后来我使用的过程中，却发现报错了

> Error initializing NSS: Unknown error -8015.

<!--more-->

# 分析

出现这个原因，主要是因为我监控系统需要用到，但监系统的执行用户和属组肯定不是root，但是我安装及配置的时候是使用root，同时，我在使用root用户发送邮件的时候却并没有这个错误。此上，基本确定应该是文件权限的问题了。

查看配置的时候，发现使用smtps发送的时候需要配置证书，也即下面一段

> set nss-config-dir=/etc/mail/.certs    # SSL证书保存位置，稍后个人制作

# 解决办法

赋予其他用户 nss-config-dir下面文件的写的权限即可，如果感觉不安全，给予其他用户 你使用的 证书 写的权限即可

> chmod +r /etc/mail/.certs -R

