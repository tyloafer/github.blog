---
title: Elastic Stack倒腾（1）- 使用Logstash从MySQL导入数据到Elastic
tags:
  - ELK
  - 大数据
categories:
  - ELK
date: '2018-05-11 18:04'
abbrlink: 41170
---

# 序言

Elastic Stack是是Elastic公司的一套开源项目，这套技术栈也被广泛的称之为ELK。包括elasicsearch，logstash，kibana。其主要功能就是对数据进行收集，格式化，索引，分析和可视化。具有搭建简单，配置容易等优点。

起初，只有elasticsearch是elastic公司的，不过在接下来的一段时间内，elastic公司先后收购了logstash和kibana，统一三者的发布版本号，完善了三者间的配合，将三者打造成了数据收集分析和展示的利器。

ElasticStack从2014年开始逐渐趋于完善稳定，不过现在（2016年05月16日）仍处在快速迭代中。

**对于分析日志而言，一般分为三个步骤；日志收集，日志整理存储，数据展示。对应ElasticStack中的：L(ogstash)， E(lasticsearch)，K(ibana)。**

<!--more-->

# 安装

本人的所有安装均在CentOs7下进行安装的，如系统不同，请做适当修改。

# 安装JAVA

ELK的运行需要基于Java，其次运行的时候，他会去在环境变量寻找Java，所以还需要设置环境变量。

1. 安装Java

   ~~~
   yum install java-1.7.0-openjdk 
   ~~~

2. 查找文件安装位置

   如果我们不熟悉yum安装后的文件位置，可以通过`rpm -ql`来显示安装后的文件所在位置

   ~~~
   rpm -ql java-1.8.0-openjdk 
   ~~~

3. 设置环境变量

   编辑`~/.bashrc`或`/etc/bashrc`， 在里面添加 `JAVA_HOME`和 `JRE_HOME`

   ~~~php
   #set java environment 
   JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.75.x86_64 
   JRE_HOME=$JAVA_HOME/jre 
   CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib 
   PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin 
   export JAVA_HOME JRE_HOME CLASS_PATH PATH 
   ~~~

## 安装Elastic

官方提供了两种安装方式，一种是压缩包的方式，这种方式比较的简单，解压后既可以使用

### 压缩包安装

1. 从官网下载好 tar.gz 文件 （tar.gz 和 zip文件一样 看个人习惯）

   ~~~
   wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.4.tar.gz
   ~~~

2. 解压压缩文件并进入文件夹

   ~~~
   tar -zxf elasticsearch-6.2.4.tar.gz
   cd elasticsearch-6.2.4
   ~~~

3. 运行

   ~~~
   bin/elasticsearch
   ~~~

   通过上面的命令运行的时候，你会发现elasticsearch是在前台运行的，终端关闭，elasticsearch也就关闭了，我们可以加个参数 `-d` 实现以守护进程的方式运行即可

   ~~~
   bin/elasticsearch -d
   ~~~

4. 可能会遇到的问题

   1. **can not run elasticsearch as root**

      新建一个 用户，并切换到这个用户，然后再启动elastic

   2. **max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]**

      切换到 root 用户修改

      ```
      vim /etc/security/limits.conf
      ```

      在最后面追加下面内容

      ```
      *** hard nofile 65536
      *** soft nofile 65536
      ```

   3. **Error occurred during initialization of VM，Could not reserve enough space for 2097152KB object heap**

      这里主要是内存不足，将elastic的运行内存调低即可

      切换到 root 用户修改

      ```
      vim /etc/security/limits.conf
      ```

      将原文件中的下面内容

      ~~~
       -Xms2g
       -Xms2g
      ~~~

      修改为

      ~~~
       -Xms1g
       -Xms1g
      ~~~


### YUM安装

1. 下载并安装签名密钥

   ~~~
   rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
   ~~~

2. 添加repo源

   编辑`elk.repo`

   ~~~
   vim /etc/yum.repos.d/elk.repo
   ~~~

   并添加一下内容

   ~~~
   [elasticsearch-6.x]
     name=Elasticsearch repository for 6.x packages
   baseurl=https://artifacts.elastic.co/packages/6.x/yum
   gpgcheck=1
   gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
   enabled=1
   autorefresh=1
   type=rpm-md
   ~~~

3. 使用Yum安装

   ~~~
   yum install elasticsearch
   ~~~

4.  测试是否安装成功

   运行后，通过curl访问对应的端口

   ~~~
   curl -XGET http://localhost:9200
   ~~~

   如果有内容返回，恭喜你。

   ​

## 安装Logstash

### 压缩包安装

1. 从官网下载好 tar.gz 文件 （tar.gz 和 zip文件一样 看个人习惯）

   ~~~
   wget https://artifacts.elastic.co/downloads/logstash/logstash-6.2.4.tar.gz
   ~~~

2. 解压压缩文件并进入文件夹

   ~~~
   tar -zxf logstash-6.2.4.tar.gz
   cd logstash-6.2.4
   ~~~

3. 运行

   ~~~
   bin/logstash -f logstash.conf
   ~~~



### YUM安装

1. 下载并安装签名密钥

   这里其实跟elastic一样，如果上面执行过，可跳过

   ~~~
   rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
   ~~~

2. 添加repo源

   编辑`elk.repo`

   ~~~
   vim /etc/yum.repos.d/elk.repo
   ~~~

   并添加一下内容

   ~~~
   [logstash-6.x]
   name=Elastic repository for 6.x packages
   baseurl=https://artifacts.elastic.co/packages/6.x/yum
   gpgcheck=1
   gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
   enabled=1
   autorefresh=1
   type=rpm-md
   ~~~

3. 使用Yum安装

   ~~~
   yum install logstash
   ~~~


## 安装Kiban

### 压缩包安装

1. 从官网下载好 tar.gz 文件 （tar.gz 和 zip文件一样 看个人习惯）

   ~~~
   wget https://artifacts.elastic.co/downloads/kibana/kibana-6.2.4-linux-x86_64.tar.gz
   ~~~

2. 解压压缩文件并进入文件夹

   ~~~
   tar -zxf kibana-6.2.4-linux-x86_64.tar.gz
   cd kibana-6.2.4-linux-x86_64
   ~~~

3. 修改`config/kibana.yml`里面的*elasticsearch.url* 指向你的 elastic的地址

4. 运行

   ~~~
   bin/kibana
   ~~~

5. 在浏览器查看

   打开浏览器，输入你的服务器的ip:5601， 例http://127.0.0.1:5601, 就可以看到Kibana的页面了，当然，有可能你的服务器没有开放5601端口，这里也可以通过nginx做个代理，进行访问，这个就放在下一篇文章讲解了。

### YUM安装

1. 下载并安装签名密钥

   这里其实跟elastic一样，如果上面执行过，可跳过

   ~~~
   rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
   ~~~

2. 添加repo源

   编辑`elk.repo`

   ~~~
   vim /etc/yum.repos.d/elk.repo
   ~~~

   并添加一下内容

   ~~~
   [kibana-6.x]
   name=Kibana repository for 6.x packages
   baseurl=https://artifacts.elastic.co/packages/6.x/yum
   gpgcheck=1
   gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
   enabled=1
   autorefresh=1
   type=rpm-md
   ~~~

3. 使用Yum安装

   ~~~
   yum install kibana
   ~~~




# 将数据从MySQL导入到elastic

我们这里安装了Logstash，所以我们这里可以通过Logstash连接数据库，获取数据后推送到elastic，由于我们数据库的数据是递增的，我们使用Logstash的时候，也可以以递增的方式向elastic添加数据，这样也减少各个方面的压力

Logstash连接数据库需要jdbc的支持，所以，第一步还是安装扩展

### 安装mysql-connector-java

yum直接安装既可以，安装完成后，通过`rmp -ql` 找到 **/usr/share/java/mysql-connector-java-5.1.17.jar** 的位置，这里后面会用到

### 配置Logstash的配置文件

在input里面，书写多个 对象， Logstash会从上而下执行，而不是覆盖，所以，如果你想一次添加多个表的数据到elastic，可以多写几个jdbc的对象，每个对象访问不同的库和表

[https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)

上面是elastic官方的 jdbc 插件的使用说明，下面我就上一下自己的配置，并简单的附一下说明


```
input {
    jdbc {
        jdbc_connection_string => "jdbc:mysql://localhost:3306/test"     // jdbc的数据库连接
        jdbc_user => "*****"            // 数据库用户名
        jdbc_password => "****"         // 数据库密码

        jdbc_driver_library => "/usr/share/java/mysql-connector-java-5.1.17.jar"     // java的mysql驱动，也就是上面我们安装的，地址替换成你服务器上的位置即可
        jdbc_driver_class => "com.mysql.jdbc.Driver"     // java的mysql驱动类

        codec => plain {charset => "UTF-8"}              // 编码，防乱码
        
        # 分页
        jdbc_paging_enabled => true                   // 开启分页查询，这样可以减轻压力
        jdbc_page_size => 300                         // 每次查询的数量
        use_column_value => true                      // 是否使用列的值，因为我们开启了分页，所以这里设置为true
        tracking_column => "id"                        // 跟踪的列的名称  这里是分页以及增量导入elastic的时候使用，与上面字段结合使用，logstash会将这个字段的值，存放在我们下面设置的文件里，然后每次查询的时候，会以存储的值为依据，向下查询
        last_run_metadata_path => "/data/elk/logstash-6.2.4/data/.logstash_jdbc_last_run_index"
        statement => "select * from test where id > :sql_last_value"        // 查看的sql语句 sql_last_value是内置的变量 ，也是存放在上面文件中的值
        type => "test"           // 类型，这里可以随便设置，我这里与表名相同
    }   
    jdbc {
        jdbc_connection_string => "jdbc:mysql://localhost:3306/test"
        jdbc_user => "*****"
        jdbc_password => "*****"

        jdbc_driver_library => "/usr/share/java/mysql-connector-java-5.1.17.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"

        # 分页
        jdbc_paging_enabled => true
        jdbc_page_size => 300 
        use_column_value => true
        tracking_column => "id"
        last_run_metadata_path => "/data/elk/logstash-6.2.4/data/.logstash_jdbc_last_run_test1"
        statement => "select * from test1 where id > :sql_last_value"
        type => "test1"
    }
output {
    stdout {                  // 输出到屏幕终端
        codec => rubydebug    // rubydebug格式输出
    }
    elasticsearch {             // 输出到elastic
        "hosts" => "127.0.0.1:9200"           // elastic的地址
        "index" => "%{type}"      // 类型  也就是我们上面设置的type的值，这样直接用，可以避免if 判断 但是索引就不能自定义了 有利有弊
        # "type" => "index"
        "document_type" => "index"   // elastic里面的type
        "document_id" => "%{id}"     // elastic里面的id，这里建议使用mysql的id，避免mysql数据在elastic里面重复，同时方便查找及增删改查
    }
}
```

# 后记

个人也是刚刚接触ELK，准备打算将学习过程中的东西记录下来，也算是踩坑日记了，所以，ELK的相关东西后面应该还会有很多，Fighting！