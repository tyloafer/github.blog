---
title: file_get_contents使用SSL连接报错
tags:
    - PHP
    - Debug
categories:
    - PHP
date: 2018-01-18 14:04
---

# 分析

在使用file_get_contents的时候，有时候获取受信任的https的内容是正常的，但是遇到一些不受信任的https连接，就会报错，主要原因还是不受信任的https大部分是自制证书或者已过期等等，检测证书的时候没通过。

既然知道了原因，那不让他验证ssl不就可以了。

<!--more-->

# 处理

~~~
string file_get_contents ( string $filename [, bool $use_include_path = false [, resource $context [, int $offset = -1 [, int $maxlen ]]]] )
~~~

上述是*file_get_contents()* 函数的文档，第三参数可以设置一些头信息等，就如同使用curl一样，我们在头部添加一下信息就可以不验证ssl了

~~~
"ssl"=>array(
  "verify_peer"=>false,
  "verify_peer_name"=>false,
),
~~~

完整使用用例如下：

~~~
<?php
$opts = [
    "ssl" => [
        "verify_peer"=>false,
        "verify_peer_name"=>false,
    ],
];
$context = stream_context_create($opts);
$response = file_get_contents("https://tyloafer.github.io/2018/12/03/mail/", false, $context);
~~~

# 扩展

官方实例中有如下用法：

~~~
<?php
// Create a stream
$opts = array(
  'http'=>array(
    'method'=>"GET",
    'header'=>"Accept-language: en\r\n" .
              "Cookie: foo=bar\r\n"
  )
);

$context = stream_context_create($opts);

// Open the file using the HTTP headers set above
$file = file_get_contents('http://www.example.com/', false, $context);

~~~

也就是说 其实我们可以在第三个参数中这是header、cookie、params等信息，这样就可以跟curl一样模拟post和get请求了。用例就不详述了，各位猿们自行探索吧。