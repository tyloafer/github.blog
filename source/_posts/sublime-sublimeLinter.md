---
title: Sublime倒腾系列：SublimeLinter-PHP语法检验
tags: 
    - Sublime
    - SublimeLinter
    - Package
categories:
    - Sublime
date: 2018-5-5 14:40
---

# 安装

通过`package controller`搜索安装`SublimeLinter`和`SublimeLinter-php`即可。

如果不清楚`package controller`的使用的话，可参考[《Sublime倒腾系列：配置sftp实现文件上传下载》](https://tyloafer.github.io/2018/04/21/sublime-sftp/) 或 参考官方文档 [https://packagecontrol.io/installation](https://packagecontrol.io/installation) 

# 配置

`Preference` -> `Package Setting` -> `SublimeLinter` ->`setting` 即可打开 `SublimeLinter` 的用户自定义界面，左侧是系统预设的配置，可以根据左侧的预设 在右边的用户设置中进行一下设置：

我的设置如下：

~~~
// SublimeLinter Settings - User
{

    "linters": {
        "php": {
            "executable": "D:\\soft\\php72\\php.exe",     // 这里填写php执行脚本的绝对路径 mac和linux使用 / ，windows下使用\\
        }
    },
    "paths": {
        "linux": [],
        "osx": [],
        "windows": ["D:\\soft\\php72\\php.exe"]  // // 这里填写环境变量的绝对地址  用到的添加上去即可
    },
}
~~~

当然，你还可以根据自己的喜好进行配置，例如 style，显示的样式， panel显示等，这个就不多介绍了，注释写的很清晰了。

