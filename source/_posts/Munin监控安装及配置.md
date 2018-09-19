---
title: Munin安装及监控
tags:
  - munin
  - Linux
categories:
  - Linux
date: '2017-12-09 14:04'
abbrlink: 62852
---



# 安装munin
1. 通过yum安装munin munin-node httpd

        yum -y install munin munin-node httpd

   <!-- more -->

2. 修改配置文件/etc/munin/munin.conf 

    ~~~
    dbdir /data/www/html/munin/databases // 数据库存放地址 
    htmldir /data/www/html/munin/html //页面存放地址
    logdir /data/www/html/munin/log // 日志存放地址 
    rundir /var/run/munin // 运行时pid存放地址 

    # Where to look for the HTML templates 
    # 
    tmpldir /etc/munin/templates 

    # a simple host tree 
    [localhost] 
        address 127.0.0.1 
        use_node_name yes
    ~~~

3. 创建存放地址并修改权限
   ​      
        mkdir /data/www/html/munin/databases /data/www/html/munin/html /data/www/html/munin/log
        chown munin:munin /data/www/html/munin -R


3. 启动munin-node
   ​      
       service munin-node start
   查看配置中htmldir的路径下是否生成了HTML等静态文件，如没有，请执行下面命令   
       su munin --shell=/bin/bash
       munin-cron

4. 访问静态页面的存放地址即可查看,此处没有单独配置域名，所以直接访问http://域名/munin/html/即可


# 利用cgi动态绘制图形

    yum -y install  spawn-fcgi # 安装绘图的cgi
    spawn-fcgi -s /var/run/munin/fastcgi-graph.sock
        -u munin -g munin /var/www/cgi-bin/munin-cgi-graph # 启动进程，这里的sock的路径可自定义，后期需要在nginx中进行配置

综上将fcgi安装启动完成，下面将fcgi整合到nginx中进行动态的绘制图片

~~~
location ^~ /munin-cgi/munin-cgi-graph/ {
    root /data/www/html
    fastcgi_split_path_info ^(/munin-cgi/munin-cgi-graph)(.*);
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_pass unix:/var/run/munin/fastcgi-graph.sock;
    include fastcgi_params;
}
~~~

# 监控nginx
监控nginx其实是利用了nginx的*http_stub_status_module*m模块来获取nginx的请求和状态。
~~~
my $URL = exists $ENV{'url'} ? $ENV{'url'} : "http://localhost/nginx_status";                   
my $port = exists $ENV{'port'} ? $ENV{'port'} : "80";                                           
                                                                                                
if ( exists $ARGV[0] and $ARGV[0] eq "autoconf" )                                               
{
    if ($ret){
        print "no ($ret)\n";
        exit 0;
    }
    my $ua = LWP::UserAgent->new(timeout => 30,
            agent => sprintf("munin/%s (libwww-perl/%s)",    	$Munin::Common::Defaults::MUNIN_VERSION, $LWP::VERSION));     
    my $response = $ua->request(HTTP::Request->new('GET',$URL));
    unless ($response->is_success and $response->content =~ /server/im)
    {
        print "no (no nginx status on $URL)\n";
        exit 0;
    }
    else
    {
        print "yes\n";
        exit 0;
    }
}
~~~
这是nginx_request模块的源码，通过上面可以看出，其实这个程序是去请求里面的url，从而获得nginx的一些状态
接下来我们看一下官方nginx的配置
~~~
server {
      listen 80;
      server_name localhost;
      location /nginx_status {
              stub_status on;
              access_log   off;
              allow 127.0.0.1;
              deny all;
      }
}
~~~
综上，整体思路可以理清：

```
graph LR
munin发起请求-->nginx获取请求
nginx获取请求-->nginx解析请求,启动stu_status模块
```
所以，只要将官方的nginx配置加入到nginx.conf中即可，但是，在此我遇到了一个问题，通过curl访问这*个https://localhost/nginx_status* 返回的是

    curl: (7) Failed connect to localhost:80; Connection refused

但是访问127.0.0.1却是可以的，我的配置如下

**nginx.conf**
~~~
server {
      listen 80;
      server_name 127.0.0.1;
      location /nginx_status {
              stub_status on;
              access_log   off;
              allow 127.0.0.1;
              deny all;
      }
}
~~~
**nginx_request**
~~~
my $URL = exists $ENV{'url'} ? $ENV{'url'} : "http://127.0.0.1/nginx_status";                   
my $port = exists $ENV{'port'} ? $ENV{'port'} : "80";                                           
                                                                                                
if ( exists $ARGV[0] and $ARGV[0] eq "autoconf" )                                               
{
    if ($ret){
        print "no ($ret)\n";
        exit 0;
    }
    my $ua = LWP::UserAgent->new(timeout => 30,
            agent => sprintf("munin/%s (libwww-perl/%s)", $Munin::Common::Defaults::MUNIN_VERSION, $LWP::VERSION));     
    my $response = $ua->request(HTTP::Request->new('GET',$URL));
    unless ($response->is_success and $response->content =~ /server/im)
    {
        print "no (no nginx status on $URL)\n";
        exit 0;
    }
    else
    {
        print "yes\n";
        exit 0;
    }
}
~~~

检查模块是否正常
~~~
munin-node-configure |grep nginx
nginx_request | yes |  
nginx_status | yes |  
~~~

# 监控Redis
1. 首先下载munin redis的第三方插件

        git clone https://github.com/bpineau/redis-munin

2. 将git下载的redis-munin中的*redis_*更名，更改为redis\_*IP*\_*PORT*, eg. *redis\_127.0.0.1\_6379*

3. 加载redis插件

   ~~~
    ln -sf /path/to/redis_*IP*_*PORT* /etc/munin/plugins/
   ~~~

4. 在/etc/munin/conf.d中添加redis，创建munin的redis配置文件

       [redis_*]  
        user root     //在这里要root用户  
        env.host 127.0.0.1  
        env.port 6379

5. 测试redis插件

        munin-run redis

6. 重启munin

        service munin-node restart
   ​     
# 监控php-fpm
1. 首先开启php-fpm的状态
    > vim /etc/php-fpm.d/www.conf
    >
    > pm.status_path = /status     //把前面注释去掉 
    >
    > kill -USR2 `cat /run/php-fpm/php-fpm.pid`  // 平滑重启php-fpm，线上建议此用法

2. 修改nginx配置
    > 在上面监控nginx的server下的location下面添加一个location配置
    >
    > ~~~
    > location ~ ^/(status|ping)$ {
    > 	include fastcgi_params;
    > 	fastcgi_pass 127.0.0.1:9000;
    > 	fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;  
    > 	allow 127.0.0.1;
    > 	deny all;
    > }
    > ~~~
3. 重启nginx
4. 通过curl访问，检测配置是否正确

        curl -v http://127.0.0.1/status

5. 下载munin-phpfpm的插件

        git clone https://github.com/tjstein/php5-fpm-munin-plugins
6. 建立phpfpm相关插件的软链

        ln -s /etc/munin/thirdPlugins/php5-fpm-munin-plugins/phpfpm_* /etc/munin/plugins/

7. 编辑phpfpm的配置文件

    > vim /et/munin/conf.d/phpfpm
    > ~~~
    > [phpfpm*]
    > env.url http://127.0.01/status
    > env.ports 80
    > ~~~

8. 测试phpfpm插件

    > munin-run phpfpm_connections
    > munin-run phpfpm_memory

# 监控MySQL
munin本身自带的MySQL监控信息太少，所以我们这里选择第三方的munin MySQL插件
1. 安装第三方MySQL插件所需的依赖

        yum -y install yum install perl-Cache-Cache perl-IPC-ShareLite perl-DBD-MySQL perl-Module-Pluggable

2. 下载源码包

        git clone https://github.com/kjellm/munin-mysql

3. 编辑下载下来的源码包里面的*Makefile*

    > 修改第四行的代码 *PLUGIN_DIR:=/usr/local/share/munin/plugins*
    >
    >       PLUGIN_DIR:=/usr/share/munin/plugins
    >
    > 修改第四十五行的代码 *$(MUNIN_NODE) restart*
    >
    >       service munin-node restart
    > ​

4. 编辑*mysql.conf*
    > 注释第十一行*env.mysqlconnection DBI:mysql:mysql*并删除第九行*env.mysqlconnection DBI:mysql:mysql;host=localhost;port=3306*的注释
    >
    > 按照mysql的配置分别填写host port user password

5. 在当前路径下执行编译脚本
   ​      
        make install

6. 检测MySQL插件是否正常安装

        munin-run mysql

# 重新生成HTML文件

综上所有步骤完成后，重启*munin-node*发现页面上还是没有MySQL，redis，phpfpm的相关内容，这时需要通过**munin-cron**脚本来重新生成HTML文件

> su - munin --shell=/bin/bash
> munin-cron

# Nginx添加认证模块及禁用缓存
我们在浏览时发现，很多地方HTML会被浏览器缓存，导致很多时候需要强制刷新才能看到最新的图片，这是我们需要在nginx中禁止缓存来处理

我们利用Nginx的*ngx_http_auth_basic_module*来做用户验证以保证信息的安全

上面我们为了动态的生成图片，在nginx中做了解析，
~~~
location ^~ /munin-cgi/munin-cgi-graph/ {
    root /data/www/html
    fastcgi_split_path_info ^(/munin-cgi/munin-cgi-graph)(.*);
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_pass unix:/var/run/munin/fastcgi-graph.sock;
    include fastcgi_params;
}
~~~

在做权限认证的时候，我们首先需要生成一个用户名和密码，以供nginx使用
​        
        printf "munin:$(openssl passwd -crypt 123456)\n" >> /etc/munin/httppwd

在这段代码的上面添加权限认证及进行缓存，通知修改一下这段解析禁止缓存
~~~
#munin 下面的文件不缓存
location ~ /munin/html {
    root /data/www/html;
    expires -1;
    add_header Cache-Control no-store;
    index index.html;
    auth_basic "User Auth";
    auth_basic_user_file /etc/munin/httppwd;
    autoindex on;

}
#munin 监控动态画图
location ^~ /munin-cgi/munin-cgi-graph/ {
    root /data/www/html;
    add_header Cache-Control no-store;
    fastcgi_split_path_info ^(/munin-cgi/munin-cgi-graph)(.*);
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_pass unix:/var/run/munin/fastcgi-graph.sock;
    include fastcgi_params;
}
~~~