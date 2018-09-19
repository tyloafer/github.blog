---
title: 记一次被黑的经历
tags:
  - Mail
  - Linux
categories:
  - Linux
date: '2018-05-25 19:53'
abbrlink: 52665
---

某天的某时某刻，我收到了服务商的邮件，告知我出网流量过大，我次奥，我的博客站（另一个，非github）已经好久没有维护了，更何况上面也就几篇刚入门的时候写的几篇小打小闹的文章，怎么会那么火爆？

当然没有那么火爆了，登上去后发现有个程序资源基本占用100%，好吗，毫无压力的被黑了。。。

<!--more-->

# 处理

幸好这个程序并不是什么复杂的程序，也仅仅就是一个脚本，把我的服务器当成了肉鸡，攻击别人的服务器，把这个脚本kill并删除后，服务器的load和cpu也就下来了，幸好不是什么牛逼或恶意点的病毒

# 分析

通过`last` 和 `lastb` 这两个命令，发现近期有很多尝试登陆，那么攻击手段也就明了了，肉鸡的暴力破解

# 处理

# 修改密码并仅允许通过私钥登陆

其实，Linux服务器的密码登录并不是一个明智的选择，我们这里可以把用户名密码登录给关了，然后通过私钥登陆

~~~
vim /etc/ssh/sshd_config
~~~

在这个文件里面有两个配置，分别按照如下设置即可

~~~
RSAAuthentication yes      # RSA 验证
PubkeyAuthentication yes   # 公钥验证
PasswordAuthentication no  # 不允许密码登录
~~~

然后我们需要将自己的公钥添加到 公钥验证列表文件里面

~~~
cat id_rsa.pub >> ~/.ssh/authorized_keys
~~~

## 添加IP黑名单

上述的攻击主要就是找很多肉鸡来宝鸡破解密码，而这些IP大部分都是黑名单里面的，所以我们时刻更新最新的IP黑名单，那么就可以再加上一层防御

Linux上黑名单可以直接添加到 `/etc/hosts.deny` 里面，这样就可以拦截了，但是我们不可能每天去更新啊，所以，我们可以写一个脚本，放在crontab里面执行一下，这样就可以继续懒下去了，这里分享一个东北大学网络中心的脚本`fetch_neusshbl.sh`

直接附上代码：

~~~bash
#!/bin/sh
# Fetch NEU SSH Black list to /etc/hosts.deny
#

export PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

URL=http://antivirus.neu.edu.cn/ssh/lists/neu_sshbl_hosts.deny.gz
HOSTSDENY=/etc/hosts.deny
TMP_DIR=/dev/shm
FILE=hosts.deny

[ -d $TMP_DIR ] || TMP_DIR=/tmp

cd $TMP_DIR

curl --connect-timeout 60 $URL 2> /dev/null | gzip -dc > $FILE 2> /dev/null

LINES=`grep "^sshd:" $FILE | wc -l`

if [ $LINES -gt 10 ]
then
    sed -i '/^####SSH BlackList START####/,/^####SSH BlackList END####/d' $HOSTSDENY
    echo "####SSH BlackList START####" >> $HOSTSDENY
    cat $FILE >> $HOSTSDENY
    echo "####SSH BlackList END####" >> $HOSTSDENY
fi
~~~

然后将这个脚本放在`/etc/cron.daily/` 目录下面，每天执行即可

~~~
cp fetch_neusshbl.sh /etc/cron.daily/
chmod +x fetch_neusshbl.sh
~~~

## 登录邮件通知

上面的两个步骤完成后，面对这种暴力攻击，也就安全很多了，但是返回忧则生，所以我又加了最后一道防线，如果有人登录了我的服务器，就自动给我的邮箱发送一条邮件

邮件发送的服务器配置可参考我的另一篇文章[《Linux下使用mail发送邮件》](https://tyloafer.github.io/posts/47206/)

*本文还是着重从防御方面做了一点处理，可能相对来说，方法都比较的low，还请谅解，学海无涯，继续苦作舟吧！*