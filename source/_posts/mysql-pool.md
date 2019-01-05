---
title: PHP实现基于Swoole的MySQL连接池
tags:
  - Swoole
  - PHP
categories:
  - PHP
date: '2018-11-1 19:04'
abbrlink: 55187
---

对于共享资源，有一个很著名的设计模式：资源池（Resource Pool）。该模式正是为了解决资源的频繁分配﹑释放所造成的问题。数据库连接池的基本思想就是为数据库连接建立一个“池”子，在使用的时候，从“池子”中获取资源，用完后将连接放回“池子”，减少了数据库连接建立与释放造成的损耗。

<!--more-->

而对于PHP而言，每一个请求过来都是由php-fpm调起来一个cgi来处理，处理完后就会被释放，所以并没办法在代码中实现连接池的，但是由运行方式，我们可以自然而然的推想，既然php-fpm是常驻内存的，那我们将数据库连接交给php-fpm托管，也就实现了理论上的数据库连接池，而这种实现方式也就是MySQL的**pconnect**的实现方式，而且经测试，效果并没有显著的提升。那么，接下来便可以考虑基于**Swoole**的常驻内存的特性来帮忙实现了

# 思路

1. 首先基于Swoole的channel实现一个**内存队列** **quene**，（也可以通过php的共享内存实现，这里不讨论这种方式），用于存放MySQL连接的线程id，**thread_id**
2. 建立一定数量的MySQL连接存放在数组中，然后将 **thread_id**（mysqli::$thread_id）推进**内存队列** **quene**,
3. 每个请求过来的时候，从**quene**中 pop 出一个 **thread_id**， 根据**thread_id**找到对应数组中对应的数据库连接
4. 请求结束后，将**thread_id**再push进**quene**

# 代码实现

## 数据库连接管理者

> 这个类是用来管理连接的，增加和删除以及获取链接使用

~~~
class MysqlConnections {
    private $config = [
        'host' => '127.0.0.1',
        'username' => 'root',
        'password' => 'ifind@13579',
        'dbname' => 'test',
        'charset' => 'utf8',
    ];
    private $connectionsNum = 0;

    private $connections = [];
    function __construct($config = []) {
        $this->config = array_merge($this->config, $config);
    }

    /**
     * 增加一个mysql连接到连接池
     */
    public function addConnection()
    {
        $mysql = new Mysqli($this->config['host'], $this->config['username'], $this->config['password'], $this->config['dbname']);
        $this->connections[$mysql->thread_id] = $mysql;
        echo 'add ' . $mysql->thread_id . ' into pool ' . PHP_EOL;
        $this->connectionsNum ++;
        return $mysql->thread_id;
    }

    public function getConnection($id = 0)
    {
        if ($id == 0) {
            $id = $this->addConnection();
        }
        return $this->connections[$id];
    }

    public function releaseConnection($id)
    {
        $this->connectionsNum--;
        unset($this->connections[$id]);
    }

    public function __get($name)
    {
        if ($name == 'connectionsNum') {
            return $this->connectionsNum;
        } else {
            return null;
        }
    }
}
~~~

## 内存队列

> 这里是以数据库连接属性的thread_id 组成的内存队列

~~~
class Quene {
    private $length; // 队长
    private $size;

    private $chan;
    /**
     * 构造函数
     */
    public function __construct($size = 64 * 1024) 
    {
        $this->chan = new \Swoole\Channel($size);
    }

    /**
     * 入队操作
     * @param  [type] $id [description]
     * @return [type]     [description]
     */
    public function push($id)
    {
        $this->length ++;
        $this->chan->push($id);
    }
    
    /**
     * 出队操作
     * @return [type] [description]
     */
    public function pop()
    {
        $this->length --;
        $id = $this->chanpop();
        return $id;
    }

    public function __get($name)
    {
        if ($name == 'length') {
            return $this->length;
        } else {
            return null;
        }
    }
    
}
~~~

## 数据库连接池

> 结合上面的连接管理者和队列，实现对连接的分发和归还

~~~
class DbPool {
    private $max; // 最大的mysql连接数
    private $min = 0; // 建立最小的mysql连接数
    private $connections;
    private $quene;

    public function __construct($max, $min = 0)
    {
        if (!empty($max)) {
            $this->max = $max;
        } else {
            throw new Exception('please set max connnections', '-1');
        }
        $this->min = $min;
    }

    public function setConnection(MysqlConnections $connections)
    {
        $this->connections = $connections;
    }

    public function setQuene(Quene $quene)
    {
        $this->quene = $quene;
        $this->initQuene();
    }

    public function initQuene($min = 0, $max = 0)
    {
        for ($i = 0; $i < $this->min; $i++) {
            $this->quene->push($this->connections->addConnection());
        }
    }

    public function obtain()
    {
        if ($this->connections->connectionsNum >= $this->max) {
            // 超出链接最大限制，等待mysql链接释放
            $id = !$this->quene->pop();
            while (!$id) {
                usleep(200000);
                $id = $this->quene->pop();
            }
            return $this->connections->getConnection($id);
        } else {
            $id = $this->quene->pop();
            return $this->connections->getConnection($id);
        }
    }

    public function restitute(Mysqli $db)
    {
        echo 'restitute thread_id : ' . $db->thread_id . PHP_EOL;
        $this->quene->push($db->thread_id);
    }
}

~~~

## Swoole Http Server

> 基于swoole开启一个http server，获取相应的数据库连接，并相应

~~~

class SwooleServer
{
    private $config = [
        'Connections' => [
            'min' => 5,
            'max' => 10,
        ],
        'Db' => [
            'host' => '127.0.0.1',
            'username' => 'test',
            'password' => '123456',
            'dbname' => 'test',
            'charset' => 'utf8',
        ],
        'Swoole' => [
            'enable_static_handler' => true,
            'document_root' => "/home/lixy/basic/web/",
            'worker_num' => 5,
            'task_worker_num' => 5,
            'log_level' => 3,
        ],
        'Channel' => [
            'size' => 256 * 1024,
        ],
    ];
    private $db_pool;   // mysql链接内存队列
    public function __construct()
    {
        $this->swoole = new \Swoole\Http\Server('0.0.0.0', 9502);
        $this->swoole->set($this->config['Swoole']);
        $this->swoole->on('task', [$this, 'task']);
        $this->swoole->on('finish', [$this, 'finish']);
        $this->swoole->on('request', [$this, 'request']);
        $this->getDbPool();
    }

    public function getDbPool()
    {
        $connections = new MysqlConnections($this->config['Db']);
        $quene = new Quene();
        $this->db_pool = new DbPool($this->config['Connections']['max'], $this->config['Connections']['min']);
        $this->db_pool->setConnection($connections);
        $this->db_pool->setQuene($quene);
    }

    public function __call($name, $args)
    {
        return call_user_func_array([$this->swoole, $name], $args);
    }

    /**
     * onrequest事件
     * @param  [type] $request  请求对象
     * @param  [type] $response 相应对象
     * @return [type]
     */
    public function request($request, $response)
    {
        $db = $this->db_pool->obtain();
        echo 'get thread id : ' . $db->thread_id . PHP_EOL;
        $sql = 'select * from test';
        $result = $db->query($sql);
        $this->db_pool->restitute($db);
        $response->end('hello, response from swoole server');
    }

    public function task($serv, $task_id, $from_id, $data)
    {
        echo $task_id;
    }

    public function finish($serv, $task_id, $data)
    {
        echo $task_id;
    }
}

$swoole_index = new SwooleServer();
$swoole_index->start();
~~~

# 测试

> ab -n 10 -c 5 http://127.0.0.1:9502/

结果

~~~
add 6879 into pool 
add 6880 into pool 
add 6881 into pool 
add 6882 into pool 
add 6883 into pool 

get thread id : 6879
restitute thread_id : 6879
get thread id : 6880
get thread id : 6881
restitute thread_id : 6880
get thread id : 6882
restitute thread_id : 6881
restitute thread_id : 6882
get thread id : 6883
get thread id : 6879
restitute thread_id : 6879
get thread id : 6881
get thread id : 6880
restitute thread_id : 6881
restitute thread_id : 6880
restitute thread_id : 6883
get thread id : 6882
get thread id : 6879
restitute thread_id : 6882
restitute thread_id : 6879
get thread id : 6881
restitute thread_id : 6881
~~~

完美符合预期