---
title: Munin安装及监控-报警配置
tags: 
    - munin
    - Linux
categories:
    - Linux
date: 2018-01-29 14:04
---

在完成[《Munin安装及监控》](https://tyloafer.github.io/2017/12/09/Munin监控安装及配置/)之后，另一个功能-报警就被提上日程了，苦苦查看官方文档之后，发现也仅仅是介绍了有这个报警的功能，在配置中添加

~~~
contact.email.command mail -s "Munin-notification for ${var:group} :: ${var:host}" your@email.address.here
~~~

无奈，只有去`-h`查看帮助以及查看源码了。

# 修改munin-cron脚本

通过查看munin-cron的脚本发现，在定时脚本中，调用了

~~~
/usr/share/munin/munin-limits  $@  
~~~

来做检测使用量是