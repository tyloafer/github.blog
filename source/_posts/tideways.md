---
title: PHP性能分析工具xhgui+tideways实践
tags:
  - PHP
categories:
  - PHP
date: '2018-09-19 10:04'
abbrlink: 7748
---
自从线上接口报内存溢出的问题后，就一直想搭建一个性能分析的平台，但后来一直没有时间，知道后来出现了接口调用时间过长，才将这个任务提上议程。

我同事先前所在的部门使用了`xhprof`  + `xhgui` 的处理方式，但是研究后发现 `xhprof` 只支持到php5.6，无奈放弃了，同时，虽然 `tideways` 自己也提供了UI，但是炫酷的都是要收费的，综合考虑后，选用了 `tideways` + `xhgui`  的解决方案

<!--more-->

# 安装Tideways

## 安装PHP及扩展

在`https://webtatic.com/` 有每个版本的php的源，这里就简单过一下

```
yum install epel-release
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
yum install php72w* --skip-broken
```

## 安装Tideways

我这里存放源码的目录是 `/data/local/` 下，目录是不会产生任何影响的，phpize编译完扩展后会自动拷贝到php的modules对应的目录的， 拷贝命令请去除注释

~~~
git clone https://github.com/tideways/php-xhprof-extension.git   // 克隆下git文件
cd php-xhprof-extension 
phpize  // 生成configure文件
./configure
make && make install  // 安装

// 开启php扩展
echo '
; enable tideways_xhprof
extension=tideways_xhprof.so
tideways.auto_prepend_library=0' > /etc/php.d/tideways_xhprof.ini

// 重启php-fpm
service php-fpm restart
~~~

# 安装xhgui

这个项目放到一个web可访问的路径，这个最终是通过浏览器展示的

~~~
git clone https://github.com/laynefyc/xhgui-branch.git // 汉化版
// git clone https://github.com/perftools/xhgui.git  // 原版
mv xhgui-branch xhgui
cd xhgui
chmod 777 cache // 把cache目录给与代码可读写的权限
php install.php  // 安装xhgui
~~~

# 安装MongoDB

## 安装MongoDB

创建 `/etc/yum.repos.d/mongo.repo` 文件， 内容如下

~~~
[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc
~~~

使用yum安装

~~~
yum install -y mongodb-org
~~~

启动

~~~
service mongod start
~~~

## 添加索引

tideways会将结果集存放在MongoDB中，当然是可以选择的，权衡一下，我们这里使用MongoDB

~~~
mongo
> use xhprof
> db.results.ensureIndex( { 'meta.SERVER.REQUEST_TIME' : -1 } )
> db.results.ensureIndex( { 'profile.main().wt' : -1 } )
> db.results.ensureIndex( { 'profile.main().mu' : -1 } )
> db.results.ensureIndex( { 'profile.main().cpu' : -1 } )
> db.results.ensureIndex( { 'meta.url' : 1 } )
> db.results.ensureIndex( { 'meta.simple_url' : 1 } )
~~~

# 整合Tideways和Xhgui到项目

## 整合代码

汉化版xhgui的作者是建议如下使用

~~~
server {
  listen 80;
  server_name site.localhost;
  root /Users/markstory/Sites/awesome-thing/app/webroot/;  // 汉化xhgui作者的目录，仅拷贝
  fastcgi_param PHP_VALUE "auto_prepend_file=/Users/markstory/Sites/xhgui/external/header.php"; // 汉化xhgui作者的目录，仅拷贝
}
~~~

这里是你的需要做性能监控的web服务的nginx的配置

但是，我在这样使用的时候，php-fpm在我调用的接口的时候就会自动挂掉，然后看了一下文件 `xhgui/external/header.php` ， 在这个里面，作者注册了`register_shutdown_function` 函数，我使用的框架是Yii2，其次，里面还会做判断各种扩展及存储方案，这个对于我已经确定了方案的用户来说，这种判断是没用的，相对还要消耗性能，所以这里根据header.php的逻辑，直接写入代码

在beforeAction中加入 

~~~
    private function startXhprof()
    {
        $dir = '/data/www/html/xhgui';
        // Use the callbacks defined in the configuration file
        // to determine whether or not XHgui should enable profiling.
        //
        // Only load the config class so we don't pollute the host application's
        // autoloaders.
        require_once $dir . '/src/Xhgui/Config.php';
        \Xhgui_Config::load($dir . '/config/config.default.php');
        if (file_exists($dir . '/config/config.php')) {
            \Xhgui_Config::load($dir . '/config/config.php');
        }

        $filterPath = \Xhgui_Config::read('profiler.filter_path');
        if(is_array($filterPath)&&in_array($_SERVER['DOCUMENT_ROOT'],$filterPath)){
            return;
        }

        if (!isset($_SERVER['REQUEST_TIME_FLOAT'])) {
            $_SERVER['REQUEST_TIME_FLOAT'] = microtime(true);
        }

        tideways_xhprof_enable(TIDEWAYS_XHPROF_FLAGS_MEMORY | TIDEWAYS_XHPROF_FLAGS_MEMORY_MU | TIDEWAYS_XHPROF_FLAGS_MEMORY_PMU | TIDEWAYS_XHPROF_FLAGS_CPU);
    }
    
    public function beforeAction($action)
    {
      $this->startXhprof();
      return parent::beforeAction($action);
    }
~~~

在 `afterAction` 中获取分析数据并存入MongoDB

~~~
    private function stopXhprof()
    {
        $data['profile'] = tideways_xhprof_disable();
        
        // ignore_user_abort(true) allows your PHP script to continue executing, even if the user has terminated their request.
        // Further Reading: http://blog.preinheimer.com/index.php?/archives/248-When-does-a-user-abort.html
        // flush() asks PHP to send any data remaining in the output buffers. This is normally done when the script completes, but
        // since we're delaying that a bit by dealing with the xhprof stuff, we'll do it now to avoid making the user wait.
        ignore_user_abort(true);
        flush();
        
        if (!defined('XHGUI_ROOT_DIR')) {
            require '/data/www/html/xhgui/src/bootstrap.php';
        }

        $uri = array_key_exists('REQUEST_URI', $_SERVER)
            ? $_SERVER['REQUEST_URI']
            : null;
        if (empty($uri) && isset($_SERVER['argv'])) {
            $cmd = basename($_SERVER['argv'][0]);
            $uri = $cmd . ' ' . implode(' ', array_slice($_SERVER['argv'], 1));
        }

        $time = array_key_exists('REQUEST_TIME', $_SERVER)
            ? $_SERVER['REQUEST_TIME']
            : time();
        $requestTimeFloat = explode('.', $_SERVER['REQUEST_TIME_FLOAT']);
        if (!isset($requestTimeFloat[1])) {
            $requestTimeFloat[1] = 0;
        }

        $requestTs = new \MongoDate($time);
        $requestTsMicro = new \MongoDate($requestTimeFloat[0], $requestTimeFloat[1]);

        $data['meta'] = array(
            'url' => $uri,
            'SERVER' => $_SERVER,
            'get' => $_GET,
            'env' => $_ENV,
            'simple_url' => \Xhgui_Util::simpleUrl($uri),
            'request_ts' => $requestTs,
            'request_ts_micro' => $requestTsMicro,
            'request_date' => date('Y-m-d', $time),
        );

        try {
            $config = \Xhgui_Config::all();
            $config += array('db.options' => array());
            $saver = \Xhgui_Saver::factory($config);
            $saver->save($data);
        } catch (\Exception $e) {
            error_log('xhgui - ' . $e->getMessage());
        }
    }
    
    public function afterAction($action, $result)
    {
        $this->stopXhprof();
        return parent::afterAction($action, $result);
    }
~~~

## 整理nginx

我们需要添加一个server，指向`xhgui/webroot/` 目录，用于展示

~~~
server {
    listen       80;
    server_name  xx.xxx.com;
    root  /data/www/html/xhgui/webroot;   //指向自身的目录

    location / {
        index  index.php;
        if (!-e $request_filename) {
            rewrite . /index.php last;
        }
    }
	# 下面这段可根据自身配置的php解析进行相应的修改
    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
~~~

到此为止，访问配置的域名就可以看到性能分析平台了

# 异常

有可能你会遇到如下的问题

![mongo异常](http://github-1253518569.cossh.myqcloud.com/mongo-exception.png)

修改 `/path/to/xhgui/src/Xhgui/Profiles.php` 的**aggregate** 的传参，约172行

~~~
137         $results = $this->_collection->aggregate(array(
138             array('$match' => $match),
139             array(
140                 '$project' => array(
141                     'date' => $col,
142                     'profile.main()' => 1
143                 )
144             ),
145             array(
146                 '$group' => array(
147                     '_id' => '$date',
148                     'row_count' => array('$sum' => 1),
149                     'wall_times' => array('$push' => '$profile.main().wt'),
150                     'cpu_times' => array('$push' => '$profile.main().cpu'),
151                     'mu_times' => array('$push' => '$profile.main().mu'),
152                     'pmu_times' => array('$push' => '$profile.main().pmu'),
153                 )
154             ),
155             array(
156                 '$project' => array(
157                     'date' => '$date',
158                     'row_count' => '$row_count',
159                     'raw_index' => array(
160                         '$multiply' => array(
161                             '$row_count',
162                             $percentile / 100
163                         )
164                     ),
165                     'wall_times' => '$wall_times',
166                     'cpu_times' => '$cpu_times',
167                     'mu_times' => '$mu_times',
168                     'pmu_times' => '$pmu_times',
169                 )
170             ),
171             array('$sort' => array('_id' => 1)),
172         )
173			,array('cursor' => array('batchSize' => 0)));
~~~

