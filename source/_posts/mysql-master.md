---
title: MySQL主从配置
tags:
  - MySQL
  - DataBase
categories:
  - DataBase
date: '2018-07-14 14:04'
abbrlink: 42030
---
**MySQL主从复制原理：**master服务器将数据的改变记录二进制日志，当master上的数据发生改变时，则将其改变写入二进制日志中，salve服务器会在一定时间间隔内对master二进制日志进行探测其是否发生改变，如果发生改变，则开始一个I/OThread请求master二进制事件，同时主节点为每个I/O线程启动一个dump线程，用于向其发送二进制事件，并保存至从节点本地的中继日志中，从节点将启动SQL线程从中继日志中读取二进制日志，在本地重放，使得其数据和主节点的保持一致，最后I/OThread和SQLThread将进入睡眠状态，等待下一次被唤醒。

接下来将会记录一下具体实现

<!--more-->

# Master设置

1. 开启mysql的bin-log，主备是基于二进制日志的

   ~~~
   # master
   log-bin=/var/lib/mysql/mysql-bin
   server-id=1
   ~~~
   配置里面的*server-id* 是可以随意设置的，但是要保证在集群中的唯一性，

2. 重启mysql，让设置生效

3. 创建一个备份账号

   ~~~
   GRANT REPLICATION SLAVE ON *.* to 'username'@'ip' identified by 'password';
   ~~~

4. 导出master的数据，并且记录导出时的bin-log偏移

   ~~~
   mysqldump -uroot -p --master-data=2  --all-databases > all_databases.sql
   ~~~

   ~~~
   ➜  /data/www/html  ✗ more httpserver.sql
   --
   -- Position to start replication or point-in-time recovery from
   --

   -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=750663;
   ~~~

   命令中的 *--master-data=2* 是为了将 *CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=750663;*  进行注释，这里记录的 log_file和 log_pos 将会在备份服务器的设置上使用

# Slave设置

1. 首先设置server-id，在 `my.cnf` 中加入

   ~~~
   # replicate
   log-bin=/var/lib/mysql/mysql-bin   // 可以不开启
   server-id=2    // 这个值不可与master相同
   replicate-do-db = db1,db2///   // 选择需要同步的数据库，不写则全部同步
   ~~~

2. 重启mysql

3. 导入数据

   ~~~
   mysql -uroot -p < all_databases.sql
   ~~~

4. 设置master信息

   ~~~
   mysql root@localhost:(none)>  change master to master_host='ip',master_user='user',master_password='password', master_log_file='mysql-bin.000001',master_log_pos=750663; 
   ~~~
   这天命令中的 *master_log_file='mysql-bin.000001',master_log_pos=750663;*  便是上面sql文件中注释的内容

5. 开启同步

   ~~~
   mysql root@localhost:(none)>  start slave  
   ~~~

6. 查看slave状态

   ~~~
   mysql root@localhost:(none)> show slave status\G

      *************************** 1. row ***************************

                 Slave_IO_State: Waiting for master to send event
                 Master_Host: ip  //主服务器地址
                 Master_User: user   //授权帐户名，尽量避免使用root
                 Master_Port: 3306    //数据库端口，部分版本没有此行
                 Connect_Retry: 60
                 Master_Log_File: mysql-bin.000004
                 Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
                 Relay_Log_File: ddte-relay-bin.000003
                 Relay_Log_Pos: 251
                 Relay_Master_Log_File: mysql-bin.000004
                 Slave_IO_Running: Yes    //此状态必须YES
                 Slave_SQL_Running: Yes     //此状态必须YES
   ~~~