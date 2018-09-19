---
title: PHP基于共享内存实现single信号量控制脚本启停
tags:
  - PHP
categories:
  - PHP
date: '2018-05-18 20:30'
abbrlink: 41236
---

我们在使用nginx的时候，有`-s reload`实现配置的重载，别的应用也有`kill -USR1` 实现配置的重载，但是在php的脚本在执行的时候，就只有简单粗暴的`kill` 来杀死，再启动。所以，是否可以实现 类似于 nginx的`-s`一样，给与一个信号量，来进行平滑重启或杀死？

<!--more-->

# 思路

我们在写死循环的时候一般是`while(true)`, 那么只要在`while(true)`紧接着给他一个变量，然后在根据这个变量的值来判断是否需要结束不久可以实现程序的自动结束了吗？

这样的话，php程序内自定义变量肯定是不行了，我们可以借助一个文件或redis或mysql等来实现，例如，a脚本在执行开始的时候，在redis里面设置一个key a，然后每次while的时候，读取redis里面a的值，如果改变了，便执行相关的操作

这当然是可行的，但是在测试的时候，你就会发现，cpu会占用很高，所以这个就被kill了

# 共享内存

既然php的所有变量都是放在内存里面的，那一个php脚本a在执行的时候，设置一个变量$flag, 然后其他脚本可以访问这个变量并修改变量的值，这样的话，资源的消耗就是几乎为0了，这也就引进了一个概念：**共享内存**

共享内存的使用主要是为了能够在同一台机器不同的进程中共享一些数据，比如在多个 php-fpm 进程中共享当前进程的使用情况。这种通信也称为进程间通信（Inter-Process Communication），简称 IPC。

PHP 内置的 [shmop 扩展](http://php.net/manual/zh/book.shmop.php) (Shared Memory Operations) 提供了一系列共享内存操作的函数。当然这也是基于Linux共享内存实现的，具体的就自行google吧。

## 函数列表及简介

1. int **ftok** ( string `$pathname` , string `$proj` )  把一个执行的文件或项目转换成一个系统上的IPC key，同一个系统上，每个文件对应的IPC key是相同的，所以也就让信号量控制成为了可能
2. resource **shmop_open** ( int `$key` , string `$flags` , int `$mode` , int `$size` ) 创建一个指定大小的内存块
3. string **shmop_read** ( resource `$shmid` , int `$start` , int `$count` ) 从一个共享内存块中读取数据 如果填充的数据比共享内存的size小的话，会自动给数据填充空白符，所以记得套trim()
4. int **shmop_size** ( resource `$shmid` ) 获取一个共享内存的大小 ，参数是 **shmop_open**返回的内容
5. int **shmop_write** ( resource `$shmid` , string `$data` , int `$offset` ) 向共享内存中写入数据
6. bool **shmop_delete** ( resource `$shmid` ) 删除一个共享内存块
7. void **shmop_close** ( resource `$shmid` ) 关闭一个共享内存块，并不删除

## 注意

1. **ftok()** 函数的第一个`pathname` 可以使一个文件的全路径， 第二个参数，可以随意给，但必须是**长度为1**的字符

2. 在使用**shmop_open**的时候，会注意到 第一个参数 int `$key` 有可能会一脸迷茫，这里其实是 **fotk()**函数的返回值， 我们前面说过 **fotk()** 是将文件转换成 IPC key，也就是这里的key

3. ~~~
   "a" for access (sets SHM_RDONLY for shmat) use this flag when you need to open an existing shared memory segment for read only
   "c" for create (sets IPC_CREATE) use this flag when you need to create a new shared memory segment or if a segment with the same key exists, try to open it for read and write
   "w" for read & write access use this flag when you need to read and write to a shared memory segment, use this flag in most cases.
   "n" create a new memory segment (sets IPC_CREATE|IPC_EXCL) use this flag when you want to create a new shared memory segment but if one already exists with the same flag, fail. This is useful for security purposes, using this you can prevent race condition exploits.
   ~~~

   我在这里使用"n"的时候，会报错，使用“c”便不会，各位可自行测试

# 示例

脚本a.php 

~~~php
<?php
// 生成IPC key
$shm_key = ftok(__FILE__, 't');
// 信号量的话 0-9也足够了，所以这里分配1byte的大小即可，同时这样也不会产生空白符，省去了使用trim()
$shm_id = shmop_open($shm_key, 'c', 0644, 1);
shmop_write($shm_id, 1, 0);
while (shmop_read($shm_id, 0, shmop_size($shm_id)) == 1) {
  // to do
}
shmop_close($shm_id);
shmop_delete($shm_id);
~~~

控制启停的脚本 single.php

~~~
php single.php a.php /data/
~~~

~~~php
<?php
// 获取需要控制的脚本
$args = $_SERVER['argv'];
array_shift($args);
$file = $args[1] . '/' . $args[0];
$shm_key = ftok(__FILE__, 't');
// 信号量的话 0-9也足够了，所以这里分配1byte的大小即可，同时这样也不会产生空白符，省去了使用trim()
$shm_id = shmop_open($shm_key, 'c', 0644, 1);
shmop_write($shm_id, 0, 0);
echo "[停止] ";
while (@shmop_read($shm_id, 0, shmop_size($shm_id)) === 0) {
    echo '.';
}
shmop_close($shm_id);
shmop_delete($shm_id);
echo "成功";
~~~

# 进阶-内存锁

## 函数及列表

**ftok** 是必须要用到的

1. resource **sem_get** ( int `$key` [, int `$max_acquire` = 1 [, int `$perm` = 0666 [, int `$auto_release` = 1 ]]] ) 创建一个信号资源
2. bool **sem_acquire** ( resource `$sem_identifier` [, bool `$nowait` = FALSE ] )  根据信号资源，获得一个信号使用权限
3. bool **sem_release** ( resource `$sem_identifier` ) 释放一个获取的信号使用权限

上面三个是我使用到的，更多的可以参考PHP手册上的[Semaphore](http://php.net/manual/zh/book.sem.php)，会有更多相关的介绍，但是这里结合上面的共享内存的函数，就完全够用了。

## 注意

1. **sem_get** 的key也是 **ftok**函数产生的，所以可以与共享内存的共用一个key， 第二个参数 `$max_acquire` 设置了，这个信号资源，最多一次可以被多少个进程访问，当设置为1的时候，也就实现了锁的机制
2. **sem_acquire** 是获取一个信号资源的使用权，当这个信号资源被占用，且达到最大的请求数的时候，第二个参数`$nowait` 决定了接下来的行为，如果为设置true的话，则函数直接返回 false，而如果`$nowait`设置为false的时候，则会阻塞进程，知道获取资源的使用权限

# 进阶-为脚本加上锁

我们可以根据上面的函数，为脚本加上锁，实现一个脚本的单一执行（当然linux下的**flock**和php的**flock()**也可以实现）

脚本a.php 

```php
<?php
// 生成IPC key
$shm_key = ftok(__FILE__, 't');
// 信号量的话 0-9也足够了，所以这里分配1byte的大小即可，同时这样也不会产生空白符，省去了使用trim()
$shm_id = shmop_open($shm_key, 'c', 0644, 1);
shmop_write($shm_id, 1, 0);
$sem_id = sem_get($shm_key);
if (!sem_acquire($sem_id, true)) {
    // 有另一个脚本正在执行，结束当前执行
    echo "another is running...";
  	exit;
}
while (shmop_read($shm_id, 0, shmop_size($shm_id)) == 1) {
  // to do
}
sem_release($sem_id);
shmop_close($shm_id);
shmop_delete($shm_id);
```

控制启停的脚本 single.php

```
php single.php a.php /data/
```

```php
<?php
// 获取需要控制的脚本
$args = $_SERVER['argv'];
array_shift($args);
$file = $args[1] . '/' . $args[0];
$shm_key = ftok(__FILE__, 't');
// 信号量的话 0-9也足够了，所以这里分配1byte的大小即可，同时这样也不会产生空白符，省去了使用trim()
$shm_id = shmop_open($shm_key, 'c', 0644, 1);
shmop_write($shm_id, 0, 0);
echo "[停止] ";
$sem_id = sem_get($shm_key);
// 脚本在执行结束的时候，会释放信号资源，所以这里可以通过能否获得信号资源来判断脚本是否结束
if (!sem_acquire($sem_id, true)) {
    echo '.';
}
sem_release($sem_id);
shmop_close($shm_id);
shmop_delete($shm_id);
echo "成功";
```