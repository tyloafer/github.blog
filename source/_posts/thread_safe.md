---
title: 深入理解PHP系列之线程安全
tags:
  - PHP
categories:
  - PHP
date: '2019-01-04 20:04'
abbrlink: 56229
---

进程: 进程拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程由操作系统调度。

线程: 线程拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程亦由操作系统调度(标准线程是的)。

根据前面三篇文章《[浅析堆栈和内存溢出](https://tyloafer.github.io/posts/30726/)》 《[深入PHP系列之PHP数组底层的实现](https://tyloafer.github.io/posts/17570/)》《[深入PHP系列-变量分离与引用](https://tyloafer.github.io/posts/47934/)》可以得知，PHP的变量是由zval结构体构成的，数组则是hash表和链表构成的，这些，都是程序员进行分配销毁的内存，也即堆内存。由此可得，在不同的线程中，PHP其实是共享变量的，如果线程1修改了变量a， 则线程2使用变量a的时候，就是线程1修改后的结果，所谓线程安全也就是，如何保障，各个线程之间可以安全的使用公共的资源，不受影响且不影响其他线程。因此，PHP实现了一个线程安全资源管理器（Thread Safe Resource Manager, TSRM），用于解决这个问题，实现线程之间安全的操作公共资源。

<!--more-->



# 核心思想

![thread_1](https://github-1253518569.cos.ap-shanghai.myqcloud.com/thread_1.png)

如图，如果三个线程的内存地址不同，那么在 Thread1 里面操作的是Thread1对应的内存地址，就不会影响 Thread2 和 Thread3 对应的内存地址的值。

TSRM的核心思想就是为不同的线程分配独立的内存空间，如果一个资源会被多线程使用，那么首先需要预先向TSRM注册资源，然后TSRM为这个资源分配一个唯一的编号，并把这种资源的大小、初始化函数等保存到一个`tsrm_resource_type`结构中，各线程只能通过TSRM分配的那个编号访问这个资源；然后当线程拿着这个编号获取资源时TSRM如果发现是第一次请求，则会根据注册时的资源大小分配一块内存，然后调用初始化函数进行初始化，并把这块资源保存下来供这个线程后续使用。

# 基本实现

![线程安全结构](https://github-1253518569.cos.ap-shanghai.myqcloud.com/thread_safe.png)



TSRM的整体结构主要如上图所示，现从左向右开始梳理

## tsrm_tls_table

左侧第一个结构是 `tsrm_tls_table`, tsrm_tls_table 是一个数组，数组每个索引对应的值是一个地址，也就是为线程申请内存地址。那么如何确定线程在 tsrm_tls_table  的哪个索引下面呢，在PHP的数组的实现中，通过hash散列取模来确定索引，这里也是一样，只是简化了，直接通过线程id % tsrm_tls_table_size 来确定索引。

我们以上图为例， tsrm_tls_table_size 为 2， 如果新的一个线程， 线程id（后面用 thread_id来替代）， 3 % 2 = 1， 所以 我们在 tsrm_tls_table[1] 下面去寻找这个线程内存地址。进而，这里就会引发一个问题， 5 % 2 = 1，那是不是就冲突了?

## tsrm_tls_entry

~~~
struct _tsrm_tls_entry {
    void **storage; //资源数组
    int count; //拥有的资源数:storage数组大小
    THREAD_T thread_id; //所属线程id
    tsrm_tls_entry *next;
};
~~~

按照上面的情况来考虑，肯定会存在冲突的情况，除非这个数组特别大，但是那样内存耗费就得不偿失了。

所以，`tsrm_tls_entry` 的结构体 有一个这样的 属性 `tsrm_tls_entry *next;`，这样就可以把各个冲突的 `tsrm_tls_entry` 串成链表来解决冲突了，这也是hash冲突时，常用的解决方法。

### tsrm_resource_type

~~~
typedef struct {
    size_t size; //资源的大小
    ts_allocate_ctor ctor; //初始化函数
    ts_allocate_dtor dtor;
    int done;
} tsrm_resource_type;
~~~

`tsrm_resource_type` 这个结构体好像在上图中并没有体现出来具体是干什么的，接下来，先看一下，TSRM的初始化过程，应该就能知道这个结构体的作用了

**初始化**

~~~
TSRM_API int tsrm_startup(int expected_threads, int expected_resources, int debug_level, char *debug_filename)
{
    pthread_key_create( &tls_key, 0 );

    //分配tsrm_tls_table
    tsrm_tls_table_size = expected_threads;
    tsrm_tls_table = (tsrm_tls_entry **) calloc(tsrm_tls_table_size, sizeof(tsrm_tls_entry *));
    ...
    //初始化资源的递增id，注册资源时就是用的这个值
    id_count=0;

    //分配资源类型数组：resource_types_table
    resource_types_table_size = expected_resources;
    resource_types_table = (tsrm_resource_type *) calloc(resource_types_table_size, sizeof(tsrm_resource_type));
    ...
    //创建锁
    tsmm_mutex = tsrm_mutex_alloc();
}
~~~

**注册资源**

~~~
#ifdef ZTS
ZEND_API int executor_globals_id;
#endif

int zend_startup(zend_utility_functions *utility_functions, char **extensions)
{
    ...
#ifdef ZTS
    ts_allocate_id(&executor_globals_id, sizeof(zend_executor_globals), (ts_allocate_ctor) executor_globals_ctor, (ts_allocate_dtor) executor_globals_dtor);
    
    executor_globals = ts_resource(executor_globals_id);
    ...
#endif
}
~~~

~~~
TSRM_API ts_rsrc_id ts_allocate_id(ts_rsrc_id *rsrc_id, size_t size, ts_allocate_ctor ctor, ts_allocate_dtor dtor)
{
    //加锁，保证各线程串行调用此函数
    tsrm_mutex_lock(tsmm_mutex);

    //分配id，即id_count当前值，然后把id_count加1
    *rsrc_id = TSRM_SHUFFLE_RSRC_ID(id_count++);

    //检查resource_types_table数组当前大小是否已满
    if (resource_types_table_size < id_count) {
        //需要对resource_types_table扩容
        resource_types_table = (tsrm_resource_type *) realloc(resource_types_table, sizeof(tsrm_resource_type)*id_count);
        ...
        //把数组大小修改新的大小
        resource_types_table_size = id_count;
    }

    //将新注册的资源插入resource_types_table数组，下标就是分配的资源id
    resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].size = size;
    resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].ctor = ctor;
    resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].dtor = dtor;
    resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].done = 0;
    ...
}
~~~

根据上面的初始化和注册资源流程可以看出， `tsrm_tls_entry`  结构是 `resource_types_table` 的基本组成单位，`resource_types_table` 就是线程的资源聚集地， 在注册资源的时候，tsrm会给资源分配一个id，然后线程再去使用这个资源的时候，首先根据tread_id找到 分配的线程内存地址，然后再根据 资源id，找到资源在线程内存地址里面的地址

# 流程优化

梳理一下，线程里面查找一个资源可以分为三步

1. 获取当前线程的thread_id
2. 根据thread_id获取tsrm_tls_entry，这个过程需要对tsrm_tls_table加锁，遍历链表
3. 根据资源id，获取tsrm_tls_entry里面的对应的资源

线程对资源的操作是频繁的，每次都要进行上面三步操作是很费时的，而且第二步还需要加锁，这个将严重的影响性能。

TSRM通过线程私有数据（Thread-Specific Data, TSD）优化了这个问题，TSD是由POSIX数据库维护的，使用同一名称为不同线程保存数据的一种存储形式，即各线程根据同名的key可以获取到不同的变量地址。

~~~
// 创建名为key的TSD
int pthread_key_create(pthread_key_t* keyp, void (*destructor)(void*) );
// 销毁名为key的TSD
int pthread_key_delete(pthread_key_t* key);
// 根据key设置TSD
int pthread_setspecific(pthread_key_t key, const void* value);
// 根据key获取
int pthread_setspecific(pthread_key_t key, const void* value);
~~~

如此，便可以将上面三步优化为下面两步

1. 根据资源id获取资源的地址
2. 根据资源地址，获取资源内容