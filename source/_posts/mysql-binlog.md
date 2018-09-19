---
title: 使用binlog还原MySQL中的误操作
tags:
  - MySQL
  - DataBase
categories:
  - DataBase
date: '2018-07-14 14:04'
abbrlink: 39337
---

在操作数据库的时候，总会有些时候那么的手贱，导致数据丢失，如果是测试环境还好，但是要是正式环境，那就删库跑路吧。其实，如果你打开的bin-log，这些操作还是可以被还原的。

<!--more-->

# 开启bin-log

bin-log是MySQL的主从复制中开启的一个选项，在 `/etc/my.cnf` 中有两个配置如下

~~~
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin

# These are commonly set, remove the # and set as required.
# basedir = .....
#datadir = .....
# port = .....
# server_id = .....
# socket = .....
~~~

其中 我们需要关注的 `log_bin`  `server_id` 这两个，配置到你需要的地方即可，`server_id` 可以随意，主从不要冲突即可，个人配置如下

~~~
log_bin=/data/logs/mysql/mysql-bin.log   # binlog日志文件
server_id=1
~~~

重启一下MySQL，然后进入数据库，查看一下bin-log的状态

~~~
mysql> show variables like 'log_bin%';
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |
| log_bin_basename                | /data/mysql/mysql-bin       |
| log_bin_index                   | /data/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                         |
| log_bin_use_v1_row_events       | OFF                         |
+---------------------------------+-----------------------------+
~~~

其中 `log_bin` 的值为`On` 即表明已经开启了 `log_bin` 

# 开始手贱吧

## 制作点假数据

我们开始创建点假数据，用于我们的测试

~~~
mysql> create database test;
mysql> use test;
mysql> create table binlog_test (id int unsigned not null);
mysql> insert into binlog_test values (1), (2), (3), (4), (5);
~~~

先来5条数据，然后我们备份一下这个表，备份之前，先介绍一下`--master-data`这个参数

~~~
  --master-data[=#]   This causes the binary log position and filename to be
                      appended to the output. If equal to 1, will print it as a
                      CHANGE MASTER command; if equal to 2, that command will
                      be prefixed with a comment symbol. 
                      这个参数会在导出的文件中加入此时bin-log的偏移文件和偏移量，
                      如果这个值是1的，会打印出这条命令，如果等于2的话，这条命令
                      将被注释掉
~~~

我们在备份的时候，需要加上这条参数，并将其设置为2，以方便我们查找我们丢失的那部分数据

~~~
mysqldump -uroot -p --master-data=2 test > test.sql
~~~

备份完成了，我们再加点假数据，最后看一下这些数据会不会还在

~~~
mysql> insert into binlog_test values (6), (7), (8), (9), (10);
~~~

## 手贱吧

~~~
mysql> delete from binlog_test;
mysql> drop table binlog_test;
~~~

这样的话，binlog_test表的数据应该全部都没有了。

# 开始正文了-恢复数据

1. 先查看一下 备份的偏移

   ~~~
   λ  /data/mysql  more test.sql 
   -- MySQL dump 10.13  Distrib 5.7.18, for Linux (x86_64)
   --
   -- Host: localhost    Database: test
   -- ------------------------------------------------------
   -- Server version	5.7.18-log

   /*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
   /*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
   /*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
   /*!40101 SET NAMES utf8 */;
   /*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
   /*!40103 SET TIME_ZONE='+00:00' */;
   /*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
   /*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
   /*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
   /*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

   --
   -- Position to start replication or point-in-time recovery from
   --

   -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=627;
   ~~~

   从备份的sql文件可以看到，我们备份的偏移文件是 *mysql-bin.000002* ，偏移量是 *627*

2. 查看手贱命令的偏移

   我们这时候应该就是查找一下 我们执行的delete操作的偏移

   ~~~
   λ  /data/mysql  mysqlbinlog --base64-output=decode-rows mysql-bin.000002 | grep Delete -8
   # at 974
   #180717 16:37:59 server id 1  end_log_pos 1046 CRC32 0x19b80686 	Query	thread_id=6	exec_time=0	error_code=0
   SET TIMESTAMP=1531816679/*!*/;
   BEGIN
   /*!*/;
   # at 1046
   #180717 16:37:59 server id 1  end_log_pos 1100 CRC32 0x2daa51e1 	Table_map: `test`.`binlog_test` mapped to number 514
   # at 1100
   #180717 16:37:59 server id 1  end_log_pos 1185 CRC32 0x05ebafa5 	Delete_rows: table id 514 flags: STMT_END_F
   # at 1185
   #180717 16:37:59 server id 1  end_log_pos 1216 CRC32 0x56c7fe9e 	Xid = 80
   COMMIT/*!*/;
   # at 1216
   #180717 16:44:14 server id 1  end_log_pos 1281 CRC32 0xa285c84c 	Anonymous_GTID	last_committed=4	sequence_number=5
   SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
   # at 1281
   #180717 16:44:14 server id 1  end_log_pos 1405 CRC32 0x4474e0a7 	Query	thread_id=10	exec_time=0	error_code=0
   ~~~

   我们可以看到，在第10行的时候进行了删除操作，在第8行进行了表的map映射查找，那我们这时候，将结束的偏移就可以定位到 *mysql-bin.000002* 的 *1100* position

3. 生成备份到手贱这段时间的数据

   由于，我们仅仅对test库进行处理，所以我们可以在使用`mysqbinlog` 处理的时候指定`test` 库即可，主要用到了以下几个参数，可参考

   ~~~
   λ  /data/mysql  mysqlbinlog --help | grep position -8
    -j, --start-position=# 
                         Start reading the binlog at position N. Applies to the
                         first binlog passed on the command line.
    --stop-position=#   Stop reading the binlog at position N. Applies to the
                         last binlog passed on the command line.
    -d, --database=name List entries for just this database (local log only).
   ~~~

   ~~~
   mysqlbinlog --start-position=627 --stop-position=1100 --database=test  mysql-bin.000002 > binlog_test_drop.sql
   ~~~

   这样，我们需要的原始备份文件，和中间缺失的时间段的备份文件就都生成好了，下一步就可以恢复了

4. 恢复

   ~~~
   mysql -uroot -p test < test.sql
    mysql -uroot -p test < binlog_test_drop.sql
   ~~~

5. 查看一下吧

   ~~~
   mysql> select * from binlog_test;
   +----+
   | id |
   +----+
   |  1 |
   |  2 |
   |  3 |
   |  4 |
   |  5 |
   |  6 |
   |  7 |
   |  8 |
   |  9 |
   | 10 |
   +----+
   ~~~

   至此，基本大功告成了，当然bin-log不仅仅是用来恢复数据的，上文也讲过了，MySQL的主从就是要依靠他来完成的，下一篇文章[《MySQL主从配置》](http://tyloafer.github.io/posts/42030/)将会简单的介绍一下