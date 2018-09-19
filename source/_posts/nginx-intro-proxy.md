---
title: Nginx倒腾笔记（二）：状态码
tags:
  - Nginx
categories:
  - Nginx
date: '2018-06-08 19:35'
abbrlink: 48459
---

这里记录了一下Nginx使用过程中常见的状态码，仅仅是做了一下解释，如果遇到下面的问题，可以到根据所代表的意义，去相应的应用程序查看 error log，以此来找出对应的解决方案

<!--more-->

| 类别              | 状态码  | 状态意义                            |
| --------------- | ---- | ------------------------------- |
| **消息类(1字头)**    |      |                                 |
| **成功类（2字头）**    | 200  | 成功处理请求                          |
| **重定向累（3字头）**   | 301  | 永久重定向                           |
|                 | 302  | 临时重定向                           |
|                 | 304  | 未修改，走缓存                         |
| **请求错误类（4字头）**  | 400  | 服务器不理解请求的语法。                    |
|                 | 401  | 请求要求身份验证。 对于需要登录的网页，服务器可能返回此响应。 |
|                 | 403  | 服务器拒绝请求                         |
|                 | 404  | 没有找到资源                          |
|                 | 499  | 客户端主动断开连接                       |
| **服务器错误类（5字头）** | 500  | 服务器遇到错误，无法完成请求                  |
|                 | 502  | 网关出错                            |
|                 | 504  | 服务端处理请求超时                       |