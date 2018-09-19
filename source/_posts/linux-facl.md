---
title: Linux权限控制-ACL
tags:
  - Linux
categories:
  - Linux
date: '2018-07-21 19:40'
abbrlink: 45399
---

在Linux进行权限管理接触最多的应该就是 `chmod` `chown` ，但是用的时间长了就会发现很多弊端，很明显的就是root用户用的越来越多， 777权限的文件越来越多，部分原因是开发者过懒导致的，但是简单的`chmod` `chown` 会误杀很多用户也是一部分原因。那么接下来就介绍一下更为聪明和牛逼的权限控制----ACL

<!--more-->

**访问控制表**（Access Control List，ACL），又称**存取控制串列**，是使用以[访问控制矩阵](https://zh.wikipedia.org/wiki/%E5%AD%98%E5%8F%96%E6%8E%A7%E5%88%B6%E7%9F%A9%E9%99%A3)为基础的访问控制方法，每一个对象对应一个串列主体
。访问控制表描述每一个对象各自的访问控制，并记录可对此对象进行访问的所有主体对对象的权限。

- Linux文件系统ACL功能

  主要的目的是在提供传统的 owner,group,others 的 read,write,execute 权限之外的细部权限设置。ACL 可以针对单一使用者，单一文件或目录来进行 r,w,x 的权限规范，对于需要特殊权限的使用状况非常有帮助。

  简而言之，ACL功能是为了加强Linux系统文件权限管理。

# 开启 ACL 权限

对于CentOS6，系统安装时的分区支持acl,额外的文件系统挂载时需添加acl选项。

> mount -o remount, acl [mount point]

对于CentOS7，默认支持acl功能。

# 查看和设置文件 ACL 权限

设置 ACL 权限用`setfacl -m [u|g]:[用户名|组名]:权限 文件名`命令。
查看 ACL 权限用`getfacl 文件名`

| 选项   | 说明                           | 使用                                       |
| ---- | ---------------------------- | ---------------------------------------- |
| m    | 设置 ACL 权限                    | setfacl -m [u\|g]:[用户名 \| 组名]: 权限 文件名    |
| x    | 删除指定 ACL 权限                  | setfacl -x [u\|g]:[用户名 \| 组名] 文件名        |
| b    | 删除全部 ACL 权限                  | setfacl -b 文件名                           |
| d    | 设定默认 ACL 权限 (子文件继承目录 ACL 权限) | setfacl -m d:[u\|g]:[用户名 \| 组名]: 权限 文件名  |
| k    | 删除默认 ACL 权限 (子文件继承目录 ACL 权限) | setfacl -m                               |
| R    | 递归设置 ACL 权限 (容易给文件 x 权限)     | setfacl -m [u\|g]:[用户名 \| 组名]: 权限 -R 目录名 |

eg:

~~~
# 1. 创建权限为drwxrwx---, 用户和用户组为root的dir目录
[root@localhost ~]# mkdir ~ahao/dir 
[root@localhost ~]# chmod 770 ~ahao/dir
[root@localhost ~]# ll ~ahao
总用量 0
drwxrwx---. 2 root root 6 11月  4 22:32 dir

# 2. 操作1: ahao用户尝试进入dir目录失败, 权限不足
[ahao@localhost ~]$ cd dir
-bash: cd: dir: 权限不够

# 3. root用户设置ACL权限, 给ahao用户赋予rx权限
[root@localhost ~]# setfacl -m u:ahao:rx ~ahao/dir
[ahao@localhost ~]$ ll
总用量 0
drwxrwx---+ 2 root root 6 11月  4 22:32 dir

# 4. 操作2: ahao用户尝试进入dir目录成功, dir的+权限位代表ACL权限
[ahao@localhost ~]$ cd dir
[ahao@localhost dir]$ # 成功进入dir目录 

# 5. 操作3: 查看ACL权限
[ahao@localhost dir]$ getfacl ~ahao/dir/ 
getfacl: Removing leading '/' from absolute path names
# file: home/ahao/dir/
# owner: root
# group: root
user::rwx
user:ahao:r-x # ACL权限
group::rwx
mask::rwx
other::---
~~~

# mask 掩码

上面的例子在使用`getfacl dir`之后, 可以看到有一项是`mask`。
这个和默认权限`umask`差不多, 也是一个权限掩码, 表示所能赋予的权限最大值。
这里的`mask`和`ACL权限`进行`&与`运算, 得到的才是真正的`ACL权限`。
用人话讲, 就是

> 你考一百分是因为实力只有一百分
> 我考一百分是因为总分只有一百分

`mask`限制了权限的最高值。

~~~
# 1. 修改ACL权限mask为r-x
[root@localhost ~]# setfacl -m m:rx ~ahao/tmp/av 
[root@localhost ~]# getfacl ~ahao/tmp/av
getfacl: Removing leading '/' from absolute path names
# file: home/ahao/tmp/av
# owner: root
# group: root
user::rwx
group::r-x
mask::r-x # 修改ACL权限mask为r-x
other::---

# 2. 为用户ahao添加ACL权限rwx
[root@localhost ~]# setfacl -m u:ahao:rwx ~ahao/tmp/av/ 
[root@localhost ~]# getfacl ~ahao/tmp/av
getfacl: Removing leading '/' from absolute path names
# file: home/ahao/tmp/av
# owner: root
# group: root
user::rwx
user:ahao:rwx
group::r-x
mask::rwx # 注意, 这里的mask掩码会改变, 因为赋予的ACL权限大于mask
other::---

# 3. 修改ACL权限mask为r-x
[root@localhost ~]# setfacl -m m:rx ~ahao/tmp/av
[root@localhost ~]# getfacl ~ahao/tmp/av
getfacl: Removing leading '/' from absolute path names
# file: home/ahao/tmp/av
# owner: root
# group: root
user::rwx
user:ahao:rwx			#effective:r-x # 这里会提示真实的ACL权限为r-x
group::r-x
mask::r-x # 这里mask不会再改变
other::---
~~~

1. `mask` 会限制 `ACL` 权限的最大值。
2. 赋予`ACL` 权限大于 `mask` 的时候, 会将 `mask` **撑大**。