---
title: 使用redis-dump对redis的数据进行导入导出
tags:
  - Redis
  - Linux
categories:
  - Redis
date: '2018-02-03 16:40'
abbrlink: 55761
---

# 起因

Redis虽然可以进行aof和rdb的备份，但是总是用起来没有MySQL对数据的管理感觉顺手，便产生了使用第三方来进行数据管理的想法以及实现。

<!--more-->

# 安装Ruby

Redis-dump的安装依赖Ruby较高版本，yum源里面的Ruby版本较低，所以这里使用`rvm`进行安装

1. 安装`RVM`

   ~~~
   gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
   \curl -sSL https://get.rvm.io | bash -s stable
   ~~~

2. 使用`RVM` 安装`Ruby`

   ~~~
   rvm install ruby
   ~~~

# 安装redis-dump

直接使用`Ruby`的`gem`包管理直接安装即可

~~~
gem install redis-dump
~~~

# 使用redis-dump

1. 导出

   ~~~
   redis-dump -u :password@xxx.xxx.xxx.xxx:6379 > redis.json
   ~~~

2. 导入

   ~~~
   redis-load -u :password@localhost < redis.json
   ~~~

3. 更多

   更多可参考官方使用手册

   http://delanotes.com/redis-dump/