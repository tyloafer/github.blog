---
title: Linux下使用mail发送邮件
tags:
    - mail
    - Linux
categories:
    - Linux
date: 2018-12-03 14:04
---
近期在做监控的时候，需要通过命令行来发送邮件。普通邮件通过25端口发送，简单配置一下即可，但是我们的邮件服务器并不支持普通的smtp邮件发送，仅仅支持smtps发送邮件，这就需要证书验证了，我们在这里通过自制证书并忽略验证来通过smtps发送邮件

<!--more -->

# mailx

## 安装mailx

~~~
yum -y install mailx
~~~

## 配置mailx（smtps - 465端口）

编辑/etc/mail.rc，并添加上以下内容

~~~
set from=username@hostname.com         # 这里是发件人的邮箱地址
set smtp=smtps://smtp_hostname:465	   # 这里设置发件服务器的ssl地址,例：smtps://smtp.163.com:465
set nss-config-dir=/etc/mail/.certs    # SSL证书保存位置，稍后个人制作
set ssl-verify=ignore                  # 表示不对ssl的证书进行验证
set smtp-auth-user=username@hostname.com  # 邮箱验证用户名，一般同邮箱地址
set smtp-auth-password=password        # 邮箱验证密码
set smtp-auth=login                    # 认证方式
~~~

## 配置mailx（smtp - 25端口）

smtp的发送方式相对于smtps的发送方式的配置要简单需要，因为不需要ssl验证，所以也就不需要自制证书，配置完下面部分后即可使用

~~~
set from=username@hostname.com         # 这里是发件人的邮箱地址
set smtp=smtp_hostname             	   # 这里设置发件服务器的ssl地址,例：smtp.163.com
set smtp-auth-user=username@hostname.com  # 邮箱验证用户名，一般同邮箱地址
set smtp-auth-password=password        # 邮箱验证密码
set smtp-auth=login                    # 认证方式
~~~

# 证书制作

首先创建一个目录保存证书，然后创建证书和密钥的数据库

```
$ mkdir ~/.certs
$ certutil -N -d ~/.certs
```

然后从邮箱服务器获取证书，并导入到本地数据库（将hostname修改成对应的邮箱主机）

```
$ echo -n | openssl s_client -connect smtp_hostname:465 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > .certs/hostname.crt
$ certutil -A -n "Google Internet Authority" -t "C,," -d .certs -i .certs/hostname.crt
```

# 发送邮件

执行一下命令即可发送邮件

~~~
$ mail -s "test" keven518@163.com 
this is test email
crtl+d
~~~

crtl+d结束输入 或者执行 `man mailx`查看帮助文档

~~~
$ Error in certificate: Peer's certificate issuer is not recognized.
~~~

上述错误直接无视即可