---
title: Redis哨兵
tags:
  - Redis
  - Linux
categories:
  - Redis
date: '2018-05-25 18:53'
abbrlink: 51207
---

Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。

<!--more-->

在介绍哨兵之前，我们先看一下一些中小型的Redis的一个简单的主从架构

![http://github-1253518569.cossh.myqcloud.com/redis-master-slave.png](http://github-1253518569.cossh.myqcloud.com/redis-master-slave.png)

我们的程序，在进程写入操作的时候，可能会直连Master，但是，如果这时候Master宕了，就会导致数据写入不进去，而等我们反应过来并处理好，说不定十几分钟甚至更长时间就过去了。但是，如果我们引入哨兵，采用下面的架构

![http://github-1253518569.cossh.myqcloud.com/redis-sentinal2.png](http://github-1253518569.cossh.myqcloud.com/redis-sentinal2.png)

我们设置多台哨兵，我们的程序去连接哨兵，加入Master宕掉了，哨兵们去帮我们选一台Slave当新的Master，这样也就能实现快速的故障转移及处理了，毕竟线上是耽误不起每分每秒的。

# 简介

Redis-Sentinel是用于管理Redis集群,该系统执行以下三个任务:

1. 监控(Monitoring):Sentinel会不断地检查你的主服务器和从服务器是否运作正常
2. 提醒(Notification):当被监控的某个Redis服务器出现问题时,Sentinel可以通过API向管理员或者其他应用程序发送通知
3. 自动故障迁移(Automatic failover):当一个主服务器不能正常工作时,Sentinel 会开始一次自动故障迁移操作,它会将失效主

服务器的其中一个从服务器升级为新的主服务器,并让失效主服务器的其他从服务器改为复制新的主服务器;当客户端试图连接失效的主服务器时,集群也会向客户端返回新主服务器的地址,使得集群可以使用新主服务器代替失效服务器

# 搭建

Redis2.8和3.0已经附带了稳定版本的哨兵，通过yum安装后，在`redis-server`的同一目录下会有一个`redis-sentinel`的可执行脚本，这个就是Redis的哨兵了，后面用`sentinel`来代替哨兵这个名词，因为这是人家的正式的名字。

# 配置

在`/etc/`目录下，也就是redis.conf的同级目录下，会有一个`redis-sentinel.conf` , 这个就是 sentinel 的配置文件

我们通过下面的命令查看一下他的主要内容

~~~
cat sentinel-26379.conf | grep -v "#" | grep -v "^$"
~~~

输出如下（# 为我的注释）：

~~~
# 哨兵监听的端口
port 26379
# 工作目录
dir /tmp
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
# 后面的参数分别是master的名字，这个在redis的conf文件里面有设置的
# redis的ip
# redis监听的端口
# quorum 是指，有几个哨兵认为master down了，才会认为这个master真正的down了，从而执行后面的灾备方案，一# 般这里设置的是总共 sentinel-num/2 + 1
sentinel monitor mymaster 127.0.0.1 6379 2
# 这个是指 sentinel 多少毫秒连接不上 master，就会认为它 down了
sentinel down-after-milliseconds mymaster 30000

# 表示如果master重新选出来后，其它slave节点能同时并行从新master同步缓存的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保定的设置为1，只同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢
sentinel parallel-syncs mymaster 1

# 表示多长时间后, master仍没活过来，则启动failover，从剩下的slave中选一个升级为master
sentinel failover-timeout mymaster 180000

# sentinel的日志
logfile /var/log/redis/sentinel.log
~~~

注：

我们一般以守护进程的方式开启哨兵，但是配置文件里面没有写，我们可以在配置文件的开始加上

~~~
daemonize yes
~~~

# 实战

## 主从配置

了解了上面的配置后，我们可以先撘一个主从服务器

> 思路如下：
>
> 1. 开启三台服务器，分别监听 6379 6380 6381 端口
> 2. 将6379 设置为master， 6380和6381 分别slaveof 6379

6379.conf

~~~~
bind 127.0.0.1
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile "/var/run/redis_6379-6379.pid"
loglevel notice
logfile "/var/log/redis/redis-6379.log"
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename "dump-6379.rdb"
dir "/var/lib/redis"
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly yes
appendfilename "appendonly-6379.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
~~~~

6380的配置

~~~
bind 127.0.0.1
protected-mode yes
port 6381
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile "/var/run/redis_6381-6381.pid"
loglevel notice
logfile "/var/log/redis/redis-6381.log"
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename "dump-6381.rdb"
dir "/var/lib/redis"
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly yes
appendfilename "appendonly-6381.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
slaveof 127.0.0.1 6379
~~~

6381的配置与6380基本相同，只是修改一下端口和log，rdb，aof的名字而已

然后我们开启了三台redis，并且设置好了主从关系

> redis-server /path/to/6379.conf
>
> redis-server  /path/to/6380.conf
>
> redis-server  /path/to/6381.conf



# 哨兵配置

> 思路如下：
>
> 1. 我们一样开启三台 sentinel 分别监听 26379 26380 26381
> 2. 将这三台 sentinel 全部监控我们的master 6379端口的Redis

26379.conf

~~~
port 26379
dir /tmp
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
logfile /var/log/redis/sentinel-26379.log
~~~

其余两个配置，除端口和logfile外，均一样，可参考着写

然后我们开启这三台哨兵

> redis-sentinel /path/to/26379.conf
>
> redis-sentinel  /path/to/26380.conf
>
> redis-sentinel  /path/to/26381.conf

# 分析日志

我们查看任一个哨兵的日志，启动后可以发现，日志里面输入了一下内容

~~~
2739:X 23 May 09:00:15.188 # Sentinel ID is 41b05827fdb8f7f8e88e74aa7070190c3c8f84f6
2739:X 23 May 09:00:15.188 # +monitor master mymaster 127.0.0.1 6379 quorum 2
2739:X 23 May 09:00:22.026 * +sentinel sentinel 50c453e7f9fa3acbab3f685621dabf26c02b53e3 127.0.0.1 26380 @ mymaster 127.0.0.1 6379
2739:X 23 May 09:00:24.191 * +sentinel sentinel 3d8d577704e9c0f6ac4cde7febab25180b7d7cc5 127.0.0.1 26381 @ mymaster 127.0.0.1 6379
~~~

也就是说sentinel 跟我我们配置的 找到了master ，并且找到了另外的两台 Sentinel ，这时候，我们去看 我们原先配置的 sentinel 的配置文件，应该也发生了变化

~~~
# Generated by CONFIG REWRITE
sentinel known-slave mymaster 127.0.0.1 6379
sentinel known-slave mymaster 127.0.0.1 6380
sentinel known-sentinel mymaster 127.0.0.1 26381 3d8d577704e9c0f6ac4cde7febab25180b7d7cc5
sentinel known-sentinel mymaster 127.0.0.1 26380 50c453e7f9fa3acbab3f685621dabf26c02b53e3
~~~

这时候，Sentinel 集群监控已经开启并且开始监控了。

那么，Sentinel 是如何工作的呢，又是怎么投票选举新的master的呢，其实还是要继续依赖他锁打印的日志

我们这时候，主动下线一台，然后继续分析一下 Sentinel 的日志

> redis-cli -p 6379 shutdown



这时候，日志又会继续打印内容

~~~
2739:X 23 May 09:05:02.978 # +sdown master mymaster 127.0.0.1 6379
2739:X 23 May 09:05:03.108 # +new-epoch 1
2739:X 23 May 09:05:03.116 # +vote-for-leader 3d8d577704e9c0f6ac4cde7febab25180b7d7cc5 1
2739:X 23 May 09:05:04.077 # +odown master mymaster 127.0.0.1 6379 #quorum 3/2
2739:X 23 May 09:05:04.078 # Next failover delay: I will not start a failover before Wed May 23 09:11:03 2018
2739:X 23 May 09:05:04.208 # +config-update-from sentinel 3d8d577704e9c0f6ac4cde7febab25180b7d7cc5 127.0.0.1 26381 @ mymaster 127.0.0.1 6379
2739:X 23 May 09:05:04.208 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6381
~~~

流程可参考：

1. Sentinel 发现了master宕掉了
2. 然后给另一台 slave 投票了
3. 另外一个又发现 master 宕了，超过我们的设定了
4. 开始协商：某个时间点之后 master还没有起，我们就另立新主吧
5. 哎哟歪，时间到了，我们把其他的配置文件都更新一下吧，不然重启了我们的决议又失效了咋办
6. 最后，我们欢迎新主降临



我们这时候把6379再开启，又会发生什么情况了

~~~
2739:X 23 May 09:05:04.209 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6381
2739:X 23 May 09:05:04.209 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6381
~~~

其实，在上面的宕机并另立master的时候 6379 的配置已经被 Sentinel 修改了，这时候，就只有乖乖的当 slave了