---
title: PHP如何读取大文件
tags:
    - PHP
    - 优化
    - 大文件
categories:
    - PHP
date: 2018-01-24 14:04
---

# 起因

这是偶然间看到的一篇文章，感觉收获颇丰，故转载。转载自[芦雨强的网络日志](https://www.luyuqiang.com/how-php-read-a-large-file)



# 干货分割线

作为一个PHP开发者，我们经常需要关注内存管理。PHP引擎在我们运行脚本之后做了很好的清理工作，短周期执行的web服务器模型意味着即使是烂代码也不会长时间影响。

<!--more-->

我们很少需要走出舒适的边界--比如我们尝试在一个小的VPS上为创建一个大项目运行Composer，或者当我们在小服务器上读取一个大文件。

这是后续将在本教程中呈现的问题。

教程代码可以在[github](https://github.com/sitepoint-editors/sitepoint-performant-reading-of-big-files-in-php)找到

# 衡量成功

确定我们完善代码的唯一方式是把烂代码和修正过的代码进行比较。换句话说，我们不知道它是否是解决办法，除非我们知道它帮了多少。

有两个我们需要关心的指标。第一个是CPU的使用。我们想要过程快或者慢？第二个是内存的使用。脚本运行使用了多少内存？这些通常是成反比的-意味着我们可以在看CPU的使用时候，不看内存的使用，反之亦然。

在一个异步程序模型中（比如多进程或者多线程的PHP应用），CPU和内存使用都需要谨慎考虑的。在传统PHP架构中，当它们中的哪个达到服务器极限的时候通常就会有问题。

在PHP中测量CPU使用不切实际。如果你关注，可以考虑在Ubuntu或者MacOs中使用top命令。Windows可以考虑安装一个linux子系统，你就可以在Ubuntu上使用top。

这个教程的目的是测量内存使用。我们将看到「传统」脚本中内存的使用情况，之后将会优化并且测量，最后我希望你可以做一个学习后的选择。

```
//php.net文档中格式化字节的方法
memory_get_peak_usage();

function formatBytes($bytes, $precision = 2) {
    $units = array('b', 'kb', 'mb', 'gb', 'tb');

    $bytes = max($bytes, 0);
    $pow = floor(($bytes ? log($bytes) : 0) / log(1024));
    $pow = min($pow, count($units) - 1);

    $bytes /= (1 << (10 * $pow));

    return round($bytes, $precision) . ' ' . $units[$pow];
}
```

我们将会在脚本的最后使用这个函数，因此可以在第一时间看到哪个脚本使用了更多的内存。

# 选项

我们可以采取很多高效读取文件方法。但是有两种常用的场景。我们可以先读取后处理数据，然后输出处理后的数据或者执行其他操作。我们可能也想要转换一个数据流而不用获取数据。

对于第一种情况，我们读取一个文件，然后每一万行创建一个独立的队列进程。我们需要至少把一万行放到在内存中，然后把他们发送到队列管理器。

对于第二种情况，我们压缩一个特别大的API响应。我们不在乎它说什么，但我们需要确保它是以压缩形式备份的。

两种情况下，我们都需要读取大文件，只不过一个关注数据一个不关注。让我们探索这些选项吧。。。

# 一行一行读文件

有很多处理文件的函数。让我们使用一个简单明了的文件读取：

```
// from memory.php

function formatBytes($bytes, $precision = 2) {
    $units = array('b', 'kb', 'mb', 'gb', 'tb');

    $bytes = max($bytes, 0);
    $pow = floor(($bytes ? log($bytes) : 0) / log(1024));
    $pow = min($pow, count($units) - 1);

    $bytes /= (1 << (10 * $pow));

    return round($bytes, $precision) . ' ' . $units[$pow];
}

print formatBytes(memory_get_peak_usage());
```

```
// from reading-files-line-by-line-1.php

function readTheFile($path) {
    $lines = [];
    $handle = fopen($path, 'r');

    while(!feof($handle)) {
        $lines[] = trim(fgets($handle));
    }

    fclose($handle);
    return $lines;
}

readTheFile('shakespeare.txt');

require 'memory.php';
```

我们正在读取一个包含莎士比亚全集的文本文件。文本文件大约5.5MB，消耗了12.8MB的内存。现在，让我们使用生成器来读取每一行：

```
// from reading-files-line-by-line-2.php

function readTheFile($path) {
    $handle = fopen($path, 'r');

    while(!feof($handle)) {
        yield trim(fgets($handle));
    }

    fclose($handle);
}

readTheFile('shakespeare.txt');

require 'memory.php';
```

这个文本文件同样大小，但是消耗了393KB的内存。这也说明不了什么，除非我们使用读取的数据做一些事。假设我们把文档以每两个空行分成小片段。就像：

```
// from reading-files-line-by-line-3.php

$iterator = readTheFile('shakespeare.txt');

$buffer = '';

foreach ($iterator as $iteration) {
    preg_match('/\n{3}/', $buffer, $matches);

    if (count($matches)) {
        print '.';
        $buffer = '';
    } else {
        $buffer .= $iteration . PHP_EOL;
    }
}

require 'memory.php';
```

猜一下我们现在用了多少内存？尽管我们把文档分割成了1216个片段，我们却只用了458KB的内存，意外吗？鉴于生成器的性质，我们内存消耗最大的是需要在循环中存储最大文本块的内存。在这种情况下，最大的块是101,985个字符。

我已经写了[使用生成器的性能提升](https://www.sitepoint.com/memory-performance-boosts-with-generators-and-nikiciter/)和[Nikita Popov的生成器库](https://github.com/nikic/iter)，所以你想要了解更多就去看吧。

生成器也有其他用法，但对读取大文件有很明显的性能提升。如果我们需要去处理数据，生成器也是最好的方式。

# 文件间的管道输送

在某些情况下，我们不需要处理数据，而是把一个文件的数据传递到另一个文件。这通常被叫做管道输送（大概因为我们只看到了两头，没看到管道内。。。当然它不是透明的）。我们可以通过使用流方法获取它们。写了个从一个文件传递到另一个的脚本，方便我们可以测量内存使用：

```
// from piping-files-1.php

file_put_contents(
    'piping-files-1.txt', file_get_contents('shakespeare.txt')
);

require 'memory.php';
```

不出意外地，这个脚本使用比文件的拷贝更多的内存。这是因为它不得不读取、把文本内容放到内存中，然后写入到一个新文件。对于小文件还好。但是当我们处理一个大文件，就不妙了。。。

让我们使用流的方式从一个文件传递到另一个（或者叫管道输送）

```
// from piping-files-2.php

$handle1 = fopen('shakespeare.txt', 'r');
$handle2 = fopen('piping-files-2.txt', 'w');

stream_copy_to_stream($handle1, $handle2);

fclose($handle1);
fclose($handle2);

require 'memory.php';
```

这段代码很奇怪。我们打开两个文件的句柄，第一个使用读模式，第二使用写模式。然后我们从第一个复制到第二个。然后关闭两个文件的句柄。是不是惊喜到你了，内存只使用了393KB。

这看起来是不是很熟悉。不就是我们使用生成器的代码一行一行读取然后存储吗？这是因为第二个变量使用fgets指定每行读取多少字节（默认-1或者直到一个新行）

stream_copy_to_stream的第三个参数是完全相同的参数（具有完全相同的默认值）。stream_copy_to_stream正在读取一个流，一次一行，并将其写入另一个流。 它跳过了生成器产生值的部分，因为我们不需要使用该值。

管道输送这些文本对我们来说没用，所以让我们仔细思考一下其他可能的例子。假设我们想要从CDN输出一个图像，重定向应用的路由。代码如下：

```
// from piping-files-3.php

file_put_contents(
    'piping-files-3.jpeg', file_get_contents(
        'https://github.com/assertchris/uploads/raw/master/rick.jpg'
    )
);

// ...or write this straight to stdout, if we don't need the memory info

require 'memory.php';
```

我们可以使用以上代码解决一个应用的路由问题。但我们想从CDN获取而不是把文件存储在本地文件系统中。我们可能使用更优雅的（像[Guzzle](http://docs.guzzlephp.org/en/stable/)）替代file_get_contents，但是效果一样。

图片的内存使用大约581KB。现在，我们试着使用流替代？

```
// from piping-files-4.php

$handle1 = fopen(
    'https://github.com/assertchris/uploads/raw/master/rick.jpg', 'r'
);

$handle2 = fopen(
    'piping-files-4.jpeg', 'w'
);

// ...or write this straight to stdout, if we don't need the memory info

stream_copy_to_stream($handle1, $handle2);

fclose($handle1);
fclose($handle2);

require 'memory.php';
```

内存使用会略少（400KB），但是结果却一样。如果我们需要更多的内存信息，我们可以打印到standard output。事实上，PHP为实现这个提供了简单的方法。

```
$handle1 = fopen(
    'https://github.com/assertchris/uploads/raw/master/rick.jpg', 'r'
);

$handle2 = fopen(
    'php://stdout', 'w'
);

stream_copy_to_stream($handle1, $handle2);

fclose($handle1);
fclose($handle2);

// require 'memory.php';
```

# 其他流

有一些其他流我们可以管道传递、读、或者写：

- php://stdin (只读)
- php://stderr (只写, 像 php://stdout)
- php://input (只读) 获取原请求体
- php://output (只写) 可以写到缓冲区
- php://memory 和 php://temp (读写)存储临时数据的地方。php://temp不同的是以文件存储，php://memory存储在内存

# 过滤器

还有一个使用流的技巧叫过滤器。它们是中间步骤，提供管理流而不暴露给我们的功能。设想一下我们想要压缩莎士比亚.txt。可能会使用Zip扩展：

```
// from filters-1.php

$zip = new ZipArchive();
$filename = 'filters-1.zip';

$zip->open($filename, ZipArchive::CREATE);
$zip->addFromString('shakespeare.txt', file_get_contents('shakespeare.txt'));
$zip->close();

require 'memory.php';
```

整洁的代码，但是却消耗了10.75MB。我们使用过滤器改进：

```
// from filters-2.php

$handle1 = fopen(
    'php://filter/zlib.deflate/resource=shakespeare.txt', 'r'
);

$handle2 = fopen(
    'filters-2.deflated', 'w'
);

stream_copy_to_stream($handle1, $handle2);

fclose($handle1);
fclose($handle2);

require 'memory.php';
```

可以看到使用php://filter/zlib.defalte的过滤器来压缩资源。我们可以把一个压缩后的数据管道传递到另一个文件。内存消耗896KB。

我知道这不是同一个格式，或者使用zip压缩更好。但是你不得不怀疑：如果你选择不同的格式可以节省掉12倍的内存，何乐而不为呢？

可以通过另一个zlib的解压缩过滤器解压文件：

```
// from filters-2.php

file_get_contents(
    'php://filter/zlib.inflate/resource=filters-2.deflated'
);
```

流已经在[理解PHP中的流](https://www.sitepoint.com/%EF%BB%BFunderstanding-streams-in-php/) 和 [PHP流与效率](https://www.sitepoint.com/using-php-streams-effectively/)中大量提及。如果你想要了解更多，点开看看。

# 自定义流

fopen和file_get_contents有他们自己的默认设置，但是可以完全的自定义。为了方便理解，自己创建一个新的流：

```
// from creating-contexts-1.php

$data = join('&', [
    'twitter=assertchris',
]);

$headers = join('\r\n', [
    'Content-type: application/x-www-form-urlencoded',
    'Content-length: ' . strlen($data),
]);

$options = [
    'http' => [
        'method' => 'POST',
        'header'=> $headers,
        'content' => $data,
    ],
];

$context = stream_content_create($options);

$handle = fopen('https://example.com/register', 'r', false, $context);
$response = stream_get_contents($handle);

fclose($handle);
```

在这个例子中，我们尝试向API发出POST请求。API端是安全的，但是仍需要使用http上下文属性（用于http和http）。我们设置一些头并且打开API文件句柄。考虑到安全，我们以只读方式打开。

可以自定义很多东西，所以如果你想了解更多，最好查看[文档](https://php.net/function.stream-context-create)。

# 自定义协议的过滤器

在本文结束之前，来谈谈自定义协议。 如果你看[文档](https://php.net/manual/en/class.streamwrapper.php)，你可以找到一个示例类来实现：

```
Protocol {
    public resource $context;
    public construct ( void )
    public destruct ( void )
    public bool dir_closedir ( void )
    public bool dir_opendir ( string $path , int $options )
    public string dir_readdir ( void )
    public bool dir_rewinddir ( void )
    public bool mkdir ( string $path , int $mode , int $options )
    public bool rename ( string $path_from , string $path_to )
    public bool rmdir ( string $path , int $options )
    public resource stream_cast ( int $cast_as )
    public void stream_close ( void )
    public bool stream_eof ( void )
    public bool stream_flush ( void )
    public bool stream_lock ( int $operation )
    public bool stream_metadata ( string $path , int $option , mixed $value )
    public bool stream_open ( string $path , string $mode , int $options ,
        string &$opened_path )
    public string stream_read ( int $count )
    public bool stream_seek ( int $offset , int $whence = SEEK_SET )
    public bool stream_set_option ( int $option , int $arg1 , int $arg2 )
    public array stream_stat ( void )
    public int stream_tell ( void )
    public bool stream_truncate ( int $new_size )
    public int stream_write ( string $data )
    public bool unlink ( string $path )
    public array url_stat ( string $path , int $flags )
}
```

我们不打算实现在教程中，因为我认为这是值得的自己完成过程。需要做很多工作，但是一旦这个工作完成，可以很容易地注册的流包装：

```
if (in_array('highlight-names', stream_get_wrappers())) {
    stream_wrapper_unregister('highlight-names');
}

stream_wrapper_register('highlight-names', 'HighlightNamesProtocol');

$highlighted = file_get_contents('highlight-names://story.txt');
```

类似地，可以自己创建一个自定义流过滤器。文档有一个过滤器类的例子：

```
Filter {
    public $filtername;
    public $params
    public int filter ( resource $in , resource $out , int &$consumed ,
        bool $closing )
    public void onClose ( void )
    public bool onCreate ( void )
}
```

很容易注册：

```
$handle = fopen('story.txt', 'w+');
stream_filter_append($handle, 'highlight-names', STREAM_FILTER_READ);
```

高亮名字过滤器需要去匹配新的过滤器类的过滤器名属性。也可以在php：//filter/highligh-names/resource=story.txt字符串中使用自定义过滤器。定义过滤器比定义协议要容易得多。 其中一个原因是协议需要处理目录操作，而过滤器只需处理每个数据块。

如果你有强烈的进取心，鼓励你编写协议的过滤器。如果你可以将过滤器应用于stream_copy_to_stream操作，那么即使处理大容量的大文件，你的应用程序内存也不会超阈值。 试着编写一个调整图像大小的过滤器或加密应用程序的过滤器。

# 总结

尽管这不是我们经常处理的问题，在读取大文件时也很容易陷入困境。在异步应用中，当我们不注意内存使用时，很容易就把整个服务搞挂。

这个教程希望给你讲解一些新想法（或者唤醒你的记忆），以便你能在读、写大文件时想得更多。当开始熟练掌握流和生成器后，停止使用像file_get_contents函数：一些莫名其妙问题就在程序中消失了。这就是意义所在！

# 声明

本文转载自[芦雨强的网络日志](https://www.luyuqiang.com/how-php-read-a-large-file)