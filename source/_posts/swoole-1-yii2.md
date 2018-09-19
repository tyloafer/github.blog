---
title: 玩玩Swoole（一）：Swoole整合Yii
tags:
  - Swoole
  - Yii
categories:
  - Swoole
date: '2018-05-18 20:04'
abbrlink: 13311
---

PHP是个单线程的脚本的语言，虽然可以通过*pcntl-fork*实现简单的多线程，但是这个肯定不是最优解，所以也就开始接触了Swoole，但是公司还没有相关的项目，所以也就玩玩而已

<!--more-->

# 入口文件




~~~
<?php
/**
 * @woole整合yii 
 * @authors Tyloafer (tyloafer@gmail.com)
 * @date    2018-05-09 08:22:21
 */

class SwooleIndex
{
	private $swoole;
	private $config;
    public function __construct()
    {
    	$option = [
    		'enable_static_handler' => true,
    		'document_root' => "/home/lixy/basic/",
    		'worker_num' => 5,
    		'log_level' => 3,
    	];
    	$this->swoole = new \Swoole\Http\Server('0.0.0.0', 9502);
    	$this->swoole->set($option);
    	$this->swoole->on('workerstart', [$this, 'workerStart']);
    	$this->swoole->on('request', [$this, 'request']);
    }

    public function __call($name, $args)
    {
    	return call_user_func_array([$this, $name], $args);
    }

    public function workerStart($server, $id)
    {
    	defined('YII_DEBUG') or define('YII_DEBUG', true);
		defined('YII_ENV') or define('YII_ENV', 'dev');

		require __DIR__ . '/../vendor/autoload.php';
		require __DIR__ . '/../vendor/yiisoft/yii2/Yii.php';

		$this->config = require __DIR__ . '/../config/web.php';
    }

    public function request($request, $response)
    {
    	$this->initParams($request);
    	ob_start();
    	(new yii\web\Application($this->config))->run();
    	$content = ob_get_contents();
    	ob_clean();
    	$response->end($content);
    }
    private function initParams($request)
    {
    	$mapping = [
    		'header' => ['suffix' => 'http', 'name' => '_SERVER', 'toupper' => true, 'replace_slash' => true],
    		'server' => ['suffix' => '', 'name' => '_SERVER', 'toupper' => true, 'replace_slash' => false],
    		'request' => ['suffix' => '', 'name' => '_REQUEST', 'toupper' => true, 'replace_slash' => false],
    		'cookie' => ['suffix' => '', 'name' => '_COOKIE', 'toupper' => false, 'replace_slash' => false],
    		'get' => ['suffix' => '', 'name' => '_GET', 'toupper' => false, 'replace_slash' => false],
    		'post' => ['suffix' => '', 'name' => '_POST', 'toupper' => false, 'replace_slash' => false],
    		'files' => ['suffix' => '', 'name' => '_FILES', 'toupper' => false, 'replace_slash' => false],
    	];

    	// 先赋值为空，防止以前的请求对后面的请求产生影响
    	foreach ($mapping as $value) {
    		$_GET = [];
    		$_POST = [];
    		$_FILES = [];
    		$_COOKIE = [];
    		$_REQUEST = [];
    		// $_SERVER = [];
    	}
    	// 针对mapping里面的循环赋值
    	foreach ($mapping as $key => $value) {
    		if (!empty($request->$key)) {
    			foreach ($request->$key as $name => $val) {
	    			if (!empty($value['suffix'])) {
	    				$name = $value['suffix'] . '_' . $name;
	    			}
	    			if ($value['toupper'] === true) {
	    				$name = strtoupper($name);
	    			}
	    			if ($value['replace_slash'] === true) {
	    				$name = str_replace('-', '_', $name);
	    			}
	    			${strtoupper($value['name'])}[$name] = $val;
	    		}
    		} else {
    			${strtoupper($value['name'])} = [];
    		}

    		$data[strtoupper($value['name'])] = ${strtoupper($value['name'])};
    	}

	 	$_GET = $data['_GET'];
		$_POST = $data['_POST'];
		$_FILES = $data['_FILES'];
		$_COOKIE = $data['_COOKIE'];
		$_REQUEST = $data['_REQUEST'];
		$_SERVER = array_merge($data['_SERVER'], $_SERVER);
    	// return true;
    }
}

$swoole_index = new SwooleIndex();
$swoole_index->start();
~~~

在上面的代码里，我准备通过可变变量来对超全局变量进行赋值的，也就是

~~~
$data[strtoupper($value['name'])] = ${strtoupper($value['name'])};
~~~

改成

~~~
${strtoupper($value['name'])} = ${strtoupper($value['name'])};
~~~

foreach里面也使用可变变量对超全局变量赋值，依然没成功，但是对非超全局变量却可以起到效果，所以暂时无解决方案。

# 配置文件web.php

~~~
$config = [
	......
    'components' => [
    	......
        // swoole task compontents
        'task' => [
            'class' => 'app\component\Task',
        ],
    ],
    ......
];
~~~

原有的便不再书写了，这里加上了task的组件。

# 任务组件Task.php

~~~
<?php
/**
 * @任务处理组件 
 * @authors Tyloafer (tyloafer@gmail.com)
 * @date    2018-05-15 08:49:25
 */
namespace app\component;

use yii;
use yii\base\Component;


class Task extends Component
{
	public function uploadFile($data)
	{
		try {
			file_put_contents('/home/lixy/upload/1.log', json_encode($data) . 'file : ' . file_get_contents($data['pic']['tmp_name']));
			move_uploaded_file($data['pic']['tmp_name'], '/home/lixy/upload/' . $data['pic']['name']);
		} catch (\Exception $e) {
			echo $e->getMessage();
		}
	}
}
~~~

# 结束

上面就简单的实现了swoole整合Yii框架，实现多线程，任务投递等。