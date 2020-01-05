---
title: GORM之for(rows.Next)提前退出别忘了Close
tags:
  - GO
  - Gorm
categories:
  - Go
date: '2020-01-05 20:08'
abbrlink: 26682
---

近期一同事负责的线上模块，总是时不时的返回一下 504，检查发现，这个服务的内存使用异常的大，pprof分析后，发现有上万个goroutine，排查分析之后，是没有规范使用gorm包导致的，那么具体是什么原因呢，会不会也像 [《Go Http包解析：为什么需要response.Body.Close()》](https://segmentfault.com/a/1190000020086816) 文中一样，因为没有释放连接导致的呢？

<!-- more -->

# 问题现象

## demo

首先我们先来看一个示例，然后，猜测一下打印的结果

```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof"
	"time"

	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
)

var (
	db *gorm.DB
)

type User struct {
	ID    int64  `gorm:"column:id;primary_key" json:"id"`
	Name  string `gorm:"column:name" json:"name"`
}

func (user *User) TableName() string {
	return "ranger_user"
}

func main() {
	go func() {
		log.Println(http.ListenAndServe(":6060", nil))
	}()
	for true {
		GetUserList()
		time.Sleep(time.Second)
	}
}

func GetUserList() ([]*User, error) {
	users := make([]*User, 0)
	db := open()
	rows, err := db.Model(&User{}).Where("id > ?", 1).Rows()
	if err != nil {
		panic(err)
	}
  // 为了试验而写的特殊逻辑
	for rows.Next() {
		user := &User{}
		err = db.ScanRows(rows, user)
		return nil, err
	}
	return users, nil
}

func open() *gorm.DB {
  if db != nil {
		return db
	}
	var err error
	db, err = gorm.Open("mysql",
     "user:pass@(ip:port)/db?charset=utf8&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}
	return db
}
```

## 分析

我们先看一下上面的demo，貌似没有什么问题，我们就运行一段时间看看

![](http://note-1253518569.cossh.myqcloud.com/20200104164025.png)

![](http://note-1253518569.cossh.myqcloud.com/20200104170110.png)

有点尴尬，我就一简单的查询返回，怎么会有那么多goroutine？

继续看一下都是哪些函数产生了goroutine

![](http://note-1253518569.cossh.myqcloud.com/20200104170141.png)

`startWatcher.func1` 是个什么鬼

~~~go
func (mc *mysqlConn) startWatcher() {
	watcher := make(chan mysqlContext, 1)
	mc.watcher = watcher
	finished := make(chan struct{})
	mc.finished = finished
	go func() {
		for {
			var ctx mysqlContext
			select {
			case ctx = <-watcher:
			case <-mc.closech:
				return
			}

			select {
			case <-ctx.Done():
				mc.cancel(ctx.Err())
			case <-finished:
			case <-mc.closech:
				return
			}
		}
	}()
}
~~~

## 猜测验证

`startWatcher` 这个函数的调用者，只有 `MySQLDriver.Open` 会调用，也就是创建新的连接的时候，才会去创建一个监控者的goroutine

根据 [《Go Http包解析：为什么需要response.Body.Close()》](https://segmentfault.com/a/1190000020086816) 中的分析结果，可以大胆猜测，有可能是mysql每次去查询的时候，获取一个连接，没有空闲的连接，则创建一个新的，查询完成后释放连接到连接池，以便下一个请求使用，而由于没有调用rows.Close(), 导致拿了连接之后，没有再放回连接池复用，导致每个请求过来都创建一个新的请求，从而导致产生了大量的goroutine去运行`startWatcher.func1` 监控新创建的连接 。所以我们类似于 response.Close 一样，进行一下 rows.Close() 是不是就ok了，接下来验证一下

对上面的测试代码增加一行rows.Close()

~~~go
defer rows.Close()
	for rows.Next() {
		user := &User{}
		err = db.ScanRows(rows, user)
		return nil, err
	}
~~~

继续观察goroutine的变化

![](http://note-1253518569.cossh.myqcloud.com/20200104170442.png)

goroutine 不再上升，貌似问题就解决了

## 疑问

1. 我们一般写代码的时候，都不会调用 `rows.Close()`的，很多情况下并没有出现goroutine的暴增，这是为什么



# 结构

照例，还是先把可能用到的结构体提前放出来，混个眼熟

## rows

~~~go
// Rows is the result of a query. Its cursor starts before the first row
// of the result set. Use Next to advance from row to row.
type Rows struct {
	dc          *driverConn // owned; must call releaseConn when closed to release
	releaseConn func(error) // driverConn.releaseConn， 在query的时候，会传递过来
	rowsi       driver.Rows
	cancel      func()      // called when Rows is closed, may be nil.
	closeStmt   *driverStmt // if non-nil, statement to Close on close

	// closemu prevents Rows from closing while there
	// is an active streaming result. It is held for read during non-close operations
	// and exclusively during close.
	//
	// closemu guards lasterr and closed.
	closemu sync.RWMutex
	closed  bool
	lasterr error // non-nil only if closed is true

	// lastcols is only used in Scan, Next, and NextResultSet which are expected
	// not to be called concurrently.
	lastcols []driver.Value
}s
~~~

# 查询

建立连接、scope结构体、Model、Where 方法的逻辑就不再赘述了，上一篇文章[《GORM之ErrRecordNotFound采坑记录》](https://segmentfault.com/a/1190000021363996)已经粗略讲过了，直接进入`Rows`函数的解析

## Rows

~~~go
// Rows return `*sql.Rows` with given conditions
func (s *DB) Rows() (*sql.Rows, error) {
	return s.NewScope(s.Value).rows()
}

func (scope *Scope) rows() (*sql.Rows, error) {
	defer scope.trace(scope.db.nowFunc())

	result := &RowsQueryResult{}
  // 设置 row_query_result，供 callback 函数使用
	scope.InstanceSet("row_query_result", result)
	scope.callCallbacks(scope.db.parent.callbacks.rowQueries)

	return result.Rows, result.Error
}
~~~

感觉这里很快就进入了`callback` 的回调

根据上一篇文章的经验，`rowQueries` 所注册的回调函数，可以在 callback_row_query.go 中的 init() 函数中找到

~~~go
func init() {
	DefaultCallback.RowQuery().Register("gorm:row_query", rowQueryCallback)
}

// queryCallback used to query data from database
func rowQueryCallback(scope *Scope) {
  // 对应 上面函数里面的 scope.InstanceSet("row_query_result", result)
	if result, ok := scope.InstanceGet("row_query_result"); ok {
    // 组装出来对应的sql语句，eg： SELECT * FROM `ranger_user`  WHERE (id > ?)
		scope.prepareQuerySQL()
		if str, ok := scope.Get("gorm:query_option"); ok {
			scope.SQL += addExtraSpaceIfExist(fmt.Sprint(str))
		}

		if rowResult, ok := result.(*RowQueryResult); ok {
			rowResult.Row = scope.SQLDB().QueryRow(scope.SQL, scope.SQLVars...)
		} else if rowsResult, ok := result.(*RowsQueryResult); ok {
      // result 对应的结构体是 RowsQueryResult，所以执行到这里，继续跟进这个函数
			rowsResult.Rows, rowsResult.Error = scope.SQLDB().Query(scope.SQL, scope.SQLVars...)
		}
	}
}
~~~

上面可以看到，`rowQueryCallback` 仅仅是组装了一下sql，然后又去调用go 提供的sql包，来进行查询

## sql.Query

~~~go
// Query executes a query that returns rows, typically a SELECT.
// The args are for any placeholder parameters in the query.
// query是sql语句，args则是sql中? 所代表的值
func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
	return db.QueryContext(context.Background(), query, args...)
}

// QueryContext executes a query that returns rows, typically a SELECT.
// The args are for any placeholder parameters in the query.
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
	var rows *Rows
	var err error
  // maxBadConnRetries = 2
	for i := 0; i < maxBadConnRetries; i++ {
    // cachedOrNewConn 则是告诉query 去使用缓存的连接或者创建一个新的连接
		rows, err = db.query(ctx, query, args, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
	}
  // 如果尝试了maxBadConnRetries次后，连接还是有问题的，则创建一个新的连接去执行sql
	if err == driver.ErrBadConn {
		return db.query(ctx, query, args, alwaysNewConn)
	}
	return rows, err
}

func (db *DB) query(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
  // 根据上面定的获取连接的策略，来获取一个有效的连接
	dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
	}
  // 使用获取的连接，进行查询
	return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
}


~~~

上面的逻辑理解不难，这里有两个变量，解释一下

cachedOrNewConn: connReuseStrategy 类型，本质是uint8类型，值是1，这个标志会传递给下面的`db.conn` 函数，告诉这个函数，返回连接的策略

 	1. 如果连接池中有空闲连接，返回一个空闲的
 	2. 如果连接池中没有空的连接，且没有超过最大创建的连接数，则创建一个新的返回
 	3. 如果连接池中没有空的连接，且超过最大创建的连接数，则等待连接释放后，返回这个空闲连接

alwaysNewConn:

1. 每次都返回一个新的连接

## 获取连接

~~~go
// conn returns a newly-opened or cached *driverConn.
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
	db.mu.Lock()
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
	// Check if the context is expired.
  // 校验一下ctx是否过期了
	select {
	default:
	case <-ctx.Done():
		db.mu.Unlock()
		return nil, ctx.Err()
	}
	lifetime := db.maxLifetime

	// Prefer a free connection, if possible.
	numFree := len(db.freeConn)
	if strategy == cachedOrNewConn && numFree > 0 {
    // 如果选择连接的策略是 cachedOrNewConn，并且有空闲的连接，则尝试获取连接池中的第一个连接
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
		db.mu.Unlock()
    // 判断当前连接的空闲时间是否超过了设定的最大空闲时间
		if conn.expired(lifetime) {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		// Lock around reading lastErr to ensure the session resetter finished.
    // 判断连接的lastErr，确保连接是被重置过的
		conn.Lock()
		err := conn.lastErr
		conn.Unlock()
		if err == driver.ErrBadConn {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		return conn, nil
	}

	// Out of free connections or we were asked not to use one. If we're not
	// allowed to open any more connections, make a request and wait.
  // 走到这里说明没有获取到空闲连接，判断创建的连接数量是否超过最大允许的连接数量
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// Make the connRequest channel. It's buffered so that the
		// connectionOpener doesn't block while waiting for the req to be read.
    // 创建一个chan，用于接收释放的空闲连接
		req := make(chan connRequest, 1)
    // 创建一个key
		reqKey := db.nextRequestKeyLocked()
    // 将key 和chan绑定，便于根据key 定位所对应的chan
		db.connRequests[reqKey] = req
		db.waitCount++
		db.mu.Unlock()

		waitStart := time.Now()

		// Timeout the connection request with the context.
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
			// on it after removing.
      // 如果ctx失效了，则这个空闲连接也不需要了，删除刚刚创建的key，防止这个连接被移除后再次为这个key获取连接
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
			case ret, ok := <-req:
        // 如果获取到了空闲连接，则放回连接池里面
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
		case ret, ok := <-req:
      // 此时拿到了空闲连接，且ctx没有过期，则判断连接是否有效
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			if !ok {
				return nil, errDBClosed
			}
      // 判断连接是否过期
			if ret.err == nil && ret.conn.expired(lifetime) {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			if ret.conn == nil {
				return nil, ret.err
			}
			// Lock around reading lastErr to ensure the session resetter finished.
      // 判断连接的lastErr，确保连接是被重置过的
			ret.conn.Lock()
			err := ret.conn.lastErr
			ret.conn.Unlock()
			if err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			return ret.conn, ret.err
		}
	}
	// 上面两个都不满足，则创建一个新的连接，也就是 获取连接的策略是 alwaysNewConn 的时候
	db.numOpen++ // optimistically
	db.mu.Unlock()
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
    // 如果连接创建失败，则再尝试创建一次
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
		inUse:     true,
	}
  // 关闭连接时会用到
	db.addDepLocked(dc, dc)
	db.mu.Unlock()
	return dc, nil
}
~~~

在上面的逻辑中，可以看到，获取连接的策略跟我们上面解释 cachedOrNewConn 和 alwaysNewConn 时是一样的，但是，这里面有两个问题

1. 创建的连接数量超过最大允许的连接数量，则等待一个空闲的连接，这时候为 db.connRequests 这个map新增加了一个key，这个key对应一个chan，然后直接等待这个 chan 吐出来连接，既然是等待释放空闲连接，那么这个chan 里面插入的 连接，应该是在freeconn 函数里面，freeconn的逻辑又是怎么样的呢
2. 创建新连接失败后，会调用 db.maybeOpenNewConnections， 这个函数又不返回连接，那么它做了什么

### 释放连接

释放连接主要依靠 `putconn` 来完成的，在 `conn` 函数的下面代码中

~~~go
			case ret, ok := <-req:
        // 如果获取到了空闲连接，则放回连接池里面
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
~~~

也调用了，把获取到但不再需要的连接放回池子里，下面看一下释放连接的过程

#### putConn

~~~go
// putConn adds a connection to the db's free pool.
// err is optionally the last error that occurred on this connection.
func (db *DB) putConn(dc *driverConn, err error, resetSession bool) {
	db.mu.Lock()
  // 释放一个正在用的连接，panic
	if !dc.inUse {
		panic("sql: connection returned that was never out")
	}
	dc.inUse = false
  
  // 省略部分无关代码...

	if err == driver.ErrBadConn {
		// Don't reuse bad connections.
		// Since the conn is considered bad and is being discarded, treat it
		// as closed. Don't decrement the open count here, finalClose will
		// take care of that.
    // maybeOpenNewConnections 这个函数又见到了，它到底干了什么
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		dc.Close()
		return
	}
  
  ...
	
  if db.closed {
		// Connections do not need to be reset if they will be closed.
		// Prevents writing to resetterCh after the DB has closed.
		resetSession = false
	}
	if resetSession {
		if _, resetSession = dc.ci.(driver.SessionResetter); resetSession {
			// Lock the driverConn here so it isn't released until
			// the connection is reset.
			// The lock must be taken before the connection is put into
			// the pool to prevent it from being taken out before it is reset.
			dc.Lock()
		}
	}
  // 把连接放回连接池中，也是这个函数的核心逻辑
	added := db.putConnDBLocked(dc, nil)
	db.mu.Unlock()
  // 如果释放连接失败，则关闭连接
	if !added {
		if resetSession {
			dc.Unlock()
		}
		dc.Close()
		return
	}
	if !resetSession {
		return
	}
  // 尝试将连接放回resetterCh chan里面，如果失败，则标识连接异常
	select {
	default:
		// If the resetterCh is blocking then mark the connection
		// as bad and continue on.
		dc.lastErr = driver.ErrBadConn
		dc.Unlock()
	case db.resetterCh <- dc:
	}
}
~~~

#### putConnDBLocked

~~~go
func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
	if db.closed {
		return false
	}
  // 已经超出最大的连接数量了，不需要再放回了
	if db.maxOpen > 0 && db.numOpen > db.maxOpen {
		return false
	}
  // 如果有其他等待获取空闲连接的协程，则
	if c := len(db.connRequests); c > 0 {
		var req chan connRequest
		var reqKey uint64
    // connRequests 获取一个 chan，并把这个连接返回到这个 chan里面
		for reqKey, req = range db.connRequests {
			break
		}
		delete(db.connRequests, reqKey) // Remove from pending requests.
		if err == nil {
			dc.inUse = true
		}
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed {
    // 如果没有超出最大数量限制，则把这个连接放到 freeConn 这个slice里面
		if db.maxIdleConnsLocked() > len(db.freeConn) {
			db.freeConn = append(db.freeConn, dc)
			db.startCleanerLocked()
			return true
		}
		db.maxIdleClosed++
	}
	return false
}
~~~

梳理完释放连接的逻辑，我们可以看出连接复用的大致流程

1. 一个新的请求过来，需要获取一个新的连接
2. 首先判断是否有空闲连接，如果没有且没有超过允许创建的最大连接数，则创建一个
3. 多个请求之后，连接数量已经超过了设定的最大连接数，则等待释放空闲连接
4. 此时，第一个请求完成了，准备释放连接，去看一下有没有等待空闲连接的请求，如果有的话，则把这个连接通过chan直接传过去，否则，把这个连接放到空闲的连接池里面
5. 此时，后面等待空闲连接的请求，拿到了第一个请求传递过来的连接，继续处理请求
6. 以上，循环往复

###maybeOpenNewConnections

这个函数，在上面的分析中已经出现了两次了，先分析一下 这个函数到底做了什么

~~~go
func (db *DB) maybeOpenNewConnections() {
  // 计算需要创建的连接数，总共创建的有效连接数不能超过设置的最大连接数
	numRequests := len(db.connRequests)
	if db.maxOpen > 0 {
		numCanOpen := db.maxOpen - db.numOpen
		if numRequests > numCanOpen {
			numRequests = numCanOpen
		}
	}
	for numRequests > 0 {
		db.numOpen++ // optimistically
		numRequests--
		if db.closed {
			return
		}
    // 往 openerCh 这个chan里面插入一条数据
		db.openerCh <- struct{}{}
	}
}
~~~

在前面的分析中，如果在获取连接时，发现产生的连接数>= 最大允许的连接数，则在 db.connRequests 这个map中创建一个唯一的 key value，用于接收释放的空闲连接，但是如果在释放连接的过程中，发现这个连接失效了，这个连接就无法复用，这时候就会走到这个函数，尝试创建一个新的连接，给其他等待的请求使用



这里就会发现一个问题： 为什么 `db.openerCh <- struct{}{}` 这样一条简单的命令就能创建一个连接，接下来就需要分析 db.openerCh 的接收方了

###connectionOpener 

这个函数在db结构体创建的时候，就会开始执行了，一个常驻的goroutine

~~~go
// Runs in a separate goroutine, opens new connections when requested.
func (db *DB) connectionOpener(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-db.openerCh:
      // 这边接收到数据后，就开始创建一个新的连接
			db.openNewConnection(ctx)
		}
	}
}
~~~

### openNewConnection

~~~go
// Open one new connection
func (db *DB) openNewConnection(ctx context.Context) {
	// maybeOpenNewConnctions has already executed db.numOpen++ before it sent
	// on db.openerCh. This function must execute db.numOpen-- if the
	// connection fails or is closed before returning.
  // 调用 sql driver 库来创建一个连接
	ci, err := db.connector.Connect(ctx)
	db.mu.Lock()
	defer db.mu.Unlock()
  // 如果db已经关闭，则关闭连接并返回
	if db.closed {
		if err == nil {
			ci.Close()
		}
		db.numOpen--
		return
	}
	if err != nil {
    // 创建连接失败了，重新调用 maybeOpenNewConnections 再创建一次
		db.numOpen--
		db.putConnDBLocked(nil, err)
		db.maybeOpenNewConnections()
		return
	}
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
	}
  // 走到 putConnDBLocked，把连接交给等待的请求方或者连接池中
	if db.putConnDBLocked(dc, err) {
		db.addDepLocked(dc, dc)
	} else {
		db.numOpen--
		ci.Close()
	}
}
~~~

### Connect

这里是连接数据库的主要逻辑

~~~go
func (t dsnConnector) Connect(_ context.Context) (driver.Conn, error) {
	return t.driver.Open(t.dsn)
}

func (d MySQLDriver) Open(dsn string) (driver.Conn, error) {
	var err error

	// New mysqlConn
	mc := &mysqlConn{
		maxAllowedPacket: maxPacketSize,
		maxWriteSize:     maxPacketSize - 1,
		closech:          make(chan struct{}),
	}
  // 解析dsn
	mc.cfg, err = ParseDSN(dsn)
	if err != nil {
		return nil, err
	}
	mc.parseTime = mc.cfg.ParseTime

	// Connect to Server
  // 找到对应网络连接类型（tcp...） 的连接函数，并创建连接
	dialsLock.RLock()
	dial, ok := dials[mc.cfg.Net]
	dialsLock.RUnlock()
	if ok {
		mc.netConn, err = dial(mc.cfg.Addr)
	} else {
		nd := net.Dialer{Timeout: mc.cfg.Timeout}
		mc.netConn, err = nd.Dial(mc.cfg.Net, mc.cfg.Addr)
	}
	if err != nil {
		return nil, err
	}

	// Enable TCP Keepalives on TCP connections
  // 开启Keepalives
	if tc, ok := mc.netConn.(*net.TCPConn); ok {
		if err := tc.SetKeepAlive(true); err != nil {
			// Don't send COM_QUIT before handshake.
			mc.netConn.Close()
			mc.netConn = nil
			return nil, err
		}
	}

	// Call startWatcher for context support (From Go 1.8)
  // 这里调用startWatcher，开始对连接进行监控，及时释放连接
	if s, ok := interface{}(mc).(watcher); ok {
		s.startWatcher()
	}

	// 下面一些设置与分析无关，忽略...

	return mc, nil
}
~~~

#### startWatcher

这个函数主要是对连接进行监控

~~~go
func (mc *mysqlConn) startWatcher() {
	watcher := make(chan mysqlContext, 1)
	mc.watcher = watcher
	finished := make(chan struct{})
	mc.finished = finished
	go func() {
		for {
			var ctx mysqlContext
			select {
			case ctx = <-watcher:
			case <-mc.closech:
				return
			}

			select {
      // ctx 过期的时候，关闭连接，这时候会关闭mc.closech
			case <-ctx.Done():
				mc.cancel(ctx.Err())
			case <-finished:
      // 关闭连接
			case <-mc.closech:
				return
			}
		}
	}()
}
~~~

创建连接的逻辑

1. 首先尝试创建一个连接，如果失败，则再次调用maybeOpenNewConnections函数，再度尝试创建一个新的连接，直到创建成功或者没有请求方需要等待连接位置
2. 新连接创建时，会调用startWatcher函数，一个常驻的goroutine，来对连接进行监控，及时的关闭
3. 连接创建成功后，通过 putConnDBLocked，把连接交给等待连接的请求方或者放到连接池中



至此，基本上连接创建及复用的流程大概清晰了，至此，对于我们最开始遇到的问题也有了一个明确的解释：

- 调用 Rows() 函数进行查询的时候，需要获取一个连接
- 此时没有新的或空闲的连接，所以，需要创建一个新的连接
- 创建连接是，创建一个 startWatcher的goroutine来进行监控
- 由于 查询完成后，没有调用 rows.Close() 及时释放连接，导致此连接一直没有放回连接池或被复用，所以每次请求，都会创建一个新的连接
- 多次请求下来，就会创建很多的startWatcher的goroutine，最终产生了遇到的现象




## Rows.Close

```go
func (rs *Rows) Close() error {
	return rs.close(nil)
}

func (rs *Rows) close(err error) error {
	rs.closemu.Lock()
	defer rs.closemu.Unlock()
  // ...
  rs.closed = true

  // 相关字段的一些设置, 忽略 ....
	rs.releaseConn(err)
	return err
}

// 通过putConn 把连接释放
func (dc *driverConn) releaseConn(err error) {
	dc.db.putConn(dc, err, true)
}
```

rs.releaseConn 所对应的函数，可以在 queryDC 这个方法里面找到，这里就直接列出来了

可以看到，rows.Close() 最后就是通过 `putConn`  把当前的连接释放以便复用

## Rows.Next

Next 为scan方法准备下一条记录，以便scan方法读取，如果没有下一行的话，或者准备下一条记录的时候出错了，就会返回false

```go
func (rs *Rows) Next() bool {
	var doClose, ok bool
	withLock(rs.closemu.RLocker(), func() {
    // 准备下一条记录
		doClose, ok = rs.nextLocked()
	})
	if doClose {
    // 如果 doClose 为true，说明没有记录了，或者准备下一条记录的时候，出错了，此时关闭连接
		rs.Close()
	}
	return ok
}

func (rs *Rows) nextLocked() (doClose, ok bool) {
  // 如果 已经关闭了，就不要读取下一条了
	if rs.closed {
		return false, false
	}

	// Lock the driver connection before calling the driver interface
	// rowsi to prevent a Tx from rolling back the connection at the same time.
	rs.dc.Lock()
	defer rs.dc.Unlock()

	if rs.lastcols == nil {
		rs.lastcols = make([]driver.Value, len(rs.rowsi.Columns()))
	}
	// 获取下一条记录，并放到lastcols里面
	rs.lasterr = rs.rowsi.Next(rs.lastcols)
	if rs.lasterr != nil {
		// Close the connection if there is a driver error.
    // 读取出错，返回true，以便后面关闭连接
		if rs.lasterr != io.EOF {
			return true, false
		}
		nextResultSet, ok := rs.rowsi.(driver.RowsNextResultSet)
		if !ok {
      // 没有获取到记录了，返回true，以便后面关闭连接
			return true, false
		}
		// The driver is at the end of the current result set.
		// Test to see if there is another result set after the current one.
		// Only close Rows if there is no further result sets to read.
		if !nextResultSet.HasNextResultSet() {
			doClose = true
		}
		return doClose, false
	}
	return false, true
}
```

Next() 的逻辑：

1. 在调用Next() 的时候，准备下一条记录，以便scan读取
2. 如果在准备数据的时候出错或者没有下一条记录的时候，返回false
3. 如果Next() 在准备数据的时候，拿到了false，则调用 rows.Close() 把连接放回池子或者交给其他请求等待着，以便复用连接

所以，也就是为什么一下的demo并不会出现问题一样

~~~go
	for rows.Next() {
		user := &User{}
		err = db.ScanRows(rows, user)
		if err != nil {
			continue
		}
	}
~~~

# 总结

走到这里，开头提出的问题应该已经有了明确的答案了： rows.Next() 在获取到最后一条记录之后，会调用 rows.Close() 将连接放回连接池或交给其他等待的请求方，所以不需要手动调用 rows.Close()，

而出问题的demo中，由于rows.Next() 没有执行到最后一条记录处，也没有调用 rows.Close(), 所以在获取到连接后一直没有被放回进行复用，导致了每来一个请求创建一个新的连接，产生一个新的监控者 `startWatcher.func1`， 最终导致了内存爆炸💥