---
title: Munin安装及监控-报警配置
tags:
  - munin
  - Linux
categories:
  - Linux
date: '2018-01-29 14:04'
abbrlink: 25813
---

在完成[《Munin安装及监控》](https://tyloafer.github.io/2017/12/09/Munin监控安装及配置/)之后，另一个功能-报警就被提上日程了，苦苦查看官方文档之后，发现也仅仅是介绍了有这个报警的功能。

<!--more-->

# Command配置

在*munin.conf* 有个配置是 *contact.anotheruser.command*， 而本篇文章的报警也是基于这个命令实现的，先贴一个示例

~~~
contact.munin.command mail -s "Munin notification" test@163.com
~~~

可以看出，其中anotheruser是我们在监控的时候的，自己定义的一个用户，这个后面会介绍。

这里的 *mail -s* 是调用Linux终端的 mail 命令来发送邮件， 具体配置可参见[《Linux下使用mail发送邮件》](https://tyloafer.github.io/2017/12/03/mail/), 当然这里的命令也可以随便写，只要能在终端执行即可，例

~~~
contact.munin.command /usr/sbin/php /home/test.php
~~~

# 用户配置

接着上面埋下的一个疑问，就是command前面的user，我怎么知道应该写哪个user，先贴一下我的一个memory的配置，这个配置时*plugin-conf.d/* 下面的*memory* 的配置

~~~
[memory]
    user munin
    env.swap_warning 80% 
    env.swap_critical 90% 
    env.active_warning 80% 
    env.cached_warning 80% 
    env.active_critical 90% 
    env.cached_critical 90% 
    env.apps_warning 80% 
    env.apps_critical 90% 
    env.cache_warning 80% 
    env.cache_critical 90%
~~~

其实，user是我们配置的第一行指定的，这个munin在执行的时候，如果检测memory达到告警值的时候，就去找这个配置的user，然后执行相应的command

# 建议

Munin的监控插件是使用perl和python来写的，没事可以多看看插件源码，可以对监控的设置会有一个更加清晰的认知，同时也可以尝试自己用脚本语言写一下，练练手。