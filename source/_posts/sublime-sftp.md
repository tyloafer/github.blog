---
title: Sublime倒腾系列：配置sftp实现文件上传下载
tags: 
    - Sublime
    - Sftp
    - Package
categories:
    - Sublime
date: 2018-04-21 13:40
---

# 起因

作为一枚PHP程序猿，用sublime做开发总感觉自己很不入流，所以后来接触了一段时间的PHPStorm，不得不说，PHPStorm真的很强大，git、xdebug、sftp等能想到的基本都集成了，但是，对于PHPStorm，一直有两点不满意，一是自动补全，二是 界面！！！所以，也就从此开启了个人的Sublime倒腾之旅。而我对Sublime而别钟爱的地方也就是他的插件，每次玩的时候都感觉跟发现了新大陆一样，让我时刻保持着新鲜感。

下面就简单介绍一下，Sublime的插件-SFTP，标榜PHPStorm的SFTP。

<!--more-->

# 安装sftp

## 安装package controller

官网虽然介绍的比较详细了，但是还是多废话一遍吧。

首先打开sublime，然后通过快捷键 *ctrl+`* 或者菜单的 *View > Show Consoled*打开Sublime的控制台，然后输入下列代码，等待执行完成即可。

~~~
import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaee' + 'ebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
~~~

## 安装sftp

通过快捷键*ctrl_shift+p* ，然后在弹出的输入框中输入 *pci* , 找到 *Package Controller: Install Package* ,确定回车，然后输入*sftp*, 等待安装完成即可

## 新建 sftp配置文件

安装完成*sftp*，下一步就是配置了，将我们的目录指向远程服务器的目录，从而实现通过简单的快捷键或者自动上传更新远程文件以与本地保持同步

1. 在打开的文件夹的根目录新建文件`sftp-config.json`

2. 通过快捷键*Ctrl+shift+p* ,调出插件管理panel

3. 输入*sftp*, 并找到 *SFTP: Brower Server*, 回车确定就发现sftp-config.json以被修改成如下

   ~~~
   {
       // The tab key will cycle through the settings when first created
       // Visit http://wbond.net/sublime_packages/sftp/settings for help
       
       // sftp, ftp or ftps
       "type": "sftp",

       "sync_down_on_open": true,
       "sync_same_age": true,
       
       "host": "example.com",
       "user": "username",
       //"password": "password",
       //"port": "22",
       
       "remote_path": "/example/path/",
       //"file_permissions": "664",
       //"dir_permissions": "775",
       
       //"extra_list_connections": 0,

       "connect_timeout": 30,
       //"keepalive": 120,
       //"ftp_passive_mode": true,
       //"ftp_obey_passive_host": false,
       //"ssh_key_file": "~/.ssh/id_rsa",
       //"sftp_flags": ["-F", "/path/to/ssh_config"],
       
       //"preserve_modification_times": false,
       //"remote_time_offset_in_hours": 0,
       //"remote_encoding": "utf-8",
       //"remote_locale": "C",
       //"allow_config_upload": false,
   }
   ~~~

4. 对配置文件进行修改，分别填写上自己的host user password remote_path 即可

## 扩展

通过上面的几步操作，已经可以实现将本地与远程文件实现同步了，但是，像我的服务器，关闭了用户名密码登录，仅允许密钥登录，这又该如何操作呢？

### 生成ppk密钥

经过我的尝试，我发现，如果通过密钥登录的话，Sublime不识别id_rsa这样格式的密钥，而仅仅识别.ppk结尾的密钥，这样我们就需要将我们的id_rsa文件转成.ppk格式

我这里是使用puttygen来进行密钥格式转换的

1. 下载好puttygen
2. 选择puttygen菜单上的Conversions->Import Key 选择 密钥
3. 点击面板上的 Save private key ，选择保存位置即可转换完成

### 配置sftp的密钥模式

密钥登录其实主要是上面配置文件中的 *ssh_key_file* 这个选项来决定的，这里要使用绝对地址，指向你的 .ppk 结尾的密钥，简单的上一下我的配置吧，各位读者应该就一目了然了

~~~
{
    // The tab key will cycle through the settings when first created
    // Visit http://wbond.net/sublime_packages/sftp/settings for help
    
    // sftp, ftp or ftps
    "type": "sftp",

    "save_before_upload": true,
    "upload_on_save": false,     // 当为true的时候，保存文件的时候会自动将你的文件上传到服务器，个人开发比较适合
    "sync_down_on_open": false,   // 当为true的时候，会自动同步远程的文件夹
    "sync_skip_deletes": false,
    "sync_same_age": true,
    "confirm_downloads": false,
    "confirm_sync": true,
    "confirm_overwrite_newer": false,
    
    "host": "xxx.xxx.xxx.xxx",
    "user": "root",
    //"password": "password",
    //"port": "22",
    
    "remote_path": "/data/www/html/httpserver/",    // 这里是服务器的文件夹路径
    "ignore_regexes": [
        "\\.sublime-(project|workspace)", "sftp-config(-alt\\d?)?\\.json",
        "sftp-settings\\.json", "/venv/", "\\.svn/", "\\.hg/", "\\.git/",
        "\\.bzr", "_darcs", "CVS", "\\.DS_Store", "Thumbs\\.db", "desktop\\.ini"
    ],
    //"file_permissions": "664",
    //"dir_permissions": "775",
    //"extra_list_connections": 0,

    "connect_timeout": 30,
    //"keepalive": 120,
    "ftp_passive_mode": true,
    "ftp_obey_passive_host": false,
    "ssh_key_file": "D:\\workplace\\cloudserver\\httpserver.ppk",   // windows下路径是\\， linux和Mac Os 是 /
    // "sftp_flags": ["-F", "/etc/ssh/ssh_config"],
    
    //"preserve_modification_times": false,
    //"remote_time_offset_in_hours": 0,
    //"remote_encoding": "utf-8",
    //"remote_locale": "C",
    // "allow_config_upload": false,
}
~~~



