---
title: GORM之ErrRecordNotFound采坑记录
tags:
  - GO
  - Gorm
categories:
  - Go
date: '2019-12-22 16:08'
abbrlink: 43551
---

在我印象中有个错误的认知：如果GORM没有找到record，则会返回`ErrRecordNotFound` 的错误，知道上次业务中出现了bug，我才发现这个印象中的认知是错误的，且没有官方文档的支持。那么，`ErrRecordNotFound`  到底在什么时候返回呢，这篇文章将会根据源码来进行分析一下

<!-- more -->

# demo

首先我们先来看一个示例，然后，猜测一下打印的结果

```go
package main

import (
  "fmt"

  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/mysql"
)

type User struct {
  ID    int64  `gorm:"column:id;primary_key" json:"id"`
  Name  string `gorm:"column:name" json:"name"`
  Email string `gorm:"column:email" json:"email"`
}

func (user *User) TableName() string {
  return "ranger_user"
}

func main() {
  db := open()
  user := &User{}
  users := make([]*User, 0, 0)
  err := db.Model(user).Where("id = ?", 1).First(user).Error
  fmt.Println(err, user)

  err = db.Model(user).Where("id = ?", 1).Find(&users).Error
  fmt.Println(err, user)
}

func open() *gorm.DB {
  db, err := gorm.Open("mysql",
     "user:pass@(ip:port)/db?charset=utf8&parseTime=True&loc=Local")
  if err != nil {
     panic(err)
  }
  return db
}
```

结果：

~~~shell
record not found &{0  }
<nil> &{0  }
~~~

综上，可以发现，`First()` 函数找不到record的时候，会返回`ErrRecordNotFound` ， 而`Find()` 则是返回nil，好了，这篇文章就到此结束了

当然，上面一句是开玩笑的，我可不是标题党，没点干货，怎好意思在这扯淡，下面我们开始追进源码

# 结构

这里是后面可能会用到的一些数据结构，放在前面有个印象，便于理解

## DB

~~~go
// DB contains information for current db connection
type DB struct {
	sync.RWMutex
	Value        interface{}  // 这里存放的model结构体，可通过结构体的方法，找到需要查询的表
	Error        error        // 出错信息
	RowsAffected int64        // sql返回的 rows affected

	// single db
	db                SQLCommon // db信息，这里是最小数据库连接所需用到的函数的interface
	blockGlobalUpdate bool
	logMode           logModeValue
	logger            logger
	search            *search   // where order group 等条件存放的地方
	values            sync.Map  // 数据结果集

	// global db
	parent        *DB    // parent db， gorm的大部分函数，都会clone一个自身，然后把被clone的对象放在这里
	callbacks     *Callback // 回调函数
	dialect       Dialect 
	singularTable bool

	// function to be used to override the creating of a new timestamp
	nowFuncOverride func() time.Time
}
~~~



## Scope

scope结构体记录了当前对数据库的所有操作

~~~go
type Scope struct {
	Search          *search
	Value           interface{}  // 用户通过First,Find 传入的结果容器变量
	SQL             string
	SQLVars         []interface{}
	db              *DB
	instanceID      string
	primaryKeyField *Field
	skipLeft        bool
	fields          *[]*Field
	selectAttrs     *[]string
}
~~~



## SQLCommon

~~~go
// SQLCommon is the minimal database connection functionality gorm requires.  Implemented by *sql.DB.
type SQLCommon interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
	Prepare(query string) (*sql.Stmt, error)
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
}
~~~



## search

sql条件的搜索语句都记录在这里了，最后拼接出来sql

~~~go
type search struct {
	db               *DB
	whereConditions  []map[string]interface{}
	orConditions     []map[string]interface{}
	notConditions    []map[string]interface{}
	havingConditions []map[string]interface{}
	joinConditions   []map[string]interface{}
	initAttrs        []interface{}
	assignAttrs      []interface{}
	selects          map[string]interface{}
	omits            []string
	orders           []interface{}
	preload          []searchPreload
	offset           interface{}
	limit            interface{}
	group            string
	tableName        string
	raw              bool
	Unscoped         bool
	ignoreOrderQuery bool
}
~~~



## Callback

记录了各个查询的回调函数，相应查询完成后，会调用对应的回调函数

~~~go
type Callback struct {
	logger     logger
	creates    []*func(scope *Scope)
	updates    []*func(scope *Scope)
	deletes    []*func(scope *Scope)
	queries    []*func(scope *Scope)
	rowQueries []*func(scope *Scope)
	processors []*CallbackProcessor
}
~~~



## CallbackProcessor

callback的详情信息，可根据这些信息，对callback进行排序，然后再放入到 `Callback` 结构体 `creates ` 等属性中

~~~go
// CallbackProcessor contains callback informations
type CallbackProcessor struct {
	logger    logger
	name      string              // current callback's name
	before    string              // register current callback before a callback
	after     string              // register current callback after a callback
	replace   bool                // replace callbacks with same name
	remove    bool                // delete callbacks with same name
	kind      string              // callback type: create, update, delete, query, row_query
	processor *func(scope *Scope) // callback handler
	parent    *Callback
}
~~~



# 新建连接

查询第一步，先建立好连接

~~~go
func Open(dialect string, args ...interface{}) (db *DB, err error) {
	if len(args) == 0 {
		err = errors.New("invalid database source")
		return nil, err
	}
	var source string
	var dbSQL SQLCommon
	var ownDbSQL bool

	switch value := args[0].(type) {
	case string:
		// 根据 dialect 判断数据库驱动类型
		var driver = dialect
		if len(args) == 1 {
			source = value
		} else if len(args) >= 2 {
			driver = value
			source = args[1].(string)
		}
		// 调用底层的database.sql 来建立一个sql.DB
		dbSQL, err = sql.Open(driver, source)
		ownDbSQL = true
	case SQLCommon:
		// 如果原先就是 SQLCommon interface，直接拿过来使用即可
		dbSQL = value
		ownDbSQL = false
	default:
		return nil, fmt.Errorf("invalid database source: %v is not a valid type", value)
	}

	db = &DB{
		db:        dbSQL,
		logger:    defaultLogger,
		callbacks: DefaultCallback,
		dialect:   newDialect(dialect, dbSQL),
	}
	db.parent = db
	if err != nil {
		return
	}
	// Send a ping to make sure the database connection is alive.
	// 发个ping，确保连接有效
	if d, ok := dbSQL.(*sql.DB); ok {
		if err = d.Ping(); err != nil && ownDbSQL {
			d.Close()
		}
	}
	return
}
~~~

可以看出，gorm.Open 也是调用了go提供的 sql.Open 来建立一个链接，最后ping一下，确保这个链接有效

# 查询

## 查询函数

链接建立完了，后面就开始进行查询了，逐个分析查询中的各个函数

~~~go
func (s *DB) Model(value interface{}) *DB {
  // 首先clone自身，确保使用了函数后，不会影响原先的变量
	c := s.clone()
  // 把传过来的结构体，设给Value
	c.Value = value
	return c
}
~~~

~~~go
// Where return a new relation, filter records with given conditions, accepts `map`, `struct` or `string` as conditions, refer http://jinzhu.github.io/gorm/crud.html#query
func (s *DB) Where(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Where(query, args...).db
}

// search.Where
func (s *search) Where(query interface{}, values ...interface{}) *search {
  // 把搜索条件追加到search的 whereConditions 属性里面
	s.whereConditions = append(s.whereConditions, map[string]interface{}{"query": query, "args": values})
	return s
}
~~~

### First

~~~go
// First find first record that match given conditions, order by primary key
func (s *DB) First(out interface{}, where ...interface{}) *DB {
	newScope := s.NewScope(out)
	newScope.Search.Limit(1)

	return newScope.Set("gorm:order_by_primary_key", "ASC").
		inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}

// NewScope create a scope for current operation
// 创建一个新的scope结构体，并记录当前的所有操作
func (s *DB) NewScope(value interface{}) *Scope {
	dbClone := s.clone()
	dbClone.Value = value
	scope := &Scope{db: dbClone, Value: value}
	if s.search != nil {
		scope.Search = s.search.clone()
	} else {
		scope.Search = &search{}
	}
	return scope
}

// 记录追加当前操作的条件语句
func (scope *Scope) inlineCondition(values ...interface{}) *Scope {
	if len(values) > 0 {
		scope.Search.Where(values[0], values[1:]...)
	}
	return scope
}
~~~

### Find

Find方法与First的逻辑很像，First增加了一个Limit(1)， 而Find没有

~~~go
// Find find records that match given conditions
func (s *DB) Find(out interface{}, where ...interface{}) *DB {
	return s.NewScope(out).inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}
~~~

执行到这里，我们发现，我们设置了sql的查询条件，但是貌似这个sql还没有执行呢，就剩下一个 `callCallbacks` 没有执行了，难道在这个函数里面，拼接sql并执行吗，后面就继续探索一下

### callCallbacks

~~~go
func (scope *Scope) callCallbacks(funcs []*func(s *Scope)) *Scope {
	defer func() {
		if err := recover(); err != nil {
			if db, ok := scope.db.db.(sqlTx); ok {
				db.Rollback()
			}
			panic(err)
		}
	}()
  // 好像就是根据传过来的函数，一个个的执行了
	for _, f := range funcs {
		(*f)(scope)
		if scope.skipLeft {
			break
		}
	}
	return scope
}
~~~

就这样，那么 `callCallbacks` 到底执行了啥？

想要知道`callCallbacks` 到底执行了什么，我们就要看看，调用`callCallbacks` 时，传了什么过来

~~~go
s.NewScope(out).inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
~~~

`s.parent.callbacks.queries` 这个好像一直没有看到有赋值的地方，那就只有 `init()` 来解释了，那么就看一下 gorm初始化都干了什么



## gorm 初始化

callback_query.go

~~~go
func init() {
	DefaultCallback.Query().Register("gorm:query", queryCallback)
	DefaultCallback.Query().Register("gorm:preload", preloadCallback)
	DefaultCallback.Query().Register("gorm:after_query", afterQueryCallback)
}
~~~

callback_delete.go

~~~go
func init() {
	DefaultCallback.Delete().Register("gorm:begin_transaction", beginTransactionCallback)
	DefaultCallback.Delete().Register("gorm:before_delete", beforeDeleteCallback)
	DefaultCallback.Delete().Register("gorm:delete", deleteCallback)
	DefaultCallback.Delete().Register("gorm:after_delete", afterDeleteCallback)
	DefaultCallback.Delete().Register("gorm:commit_or_rollback_transaction", commitOrRollbackTransactionCallback)
}
~~~

等等，create、update均有对应的init函数，这里就不继续扩展了

## 注册回调

~~~go
// Query could be used to register callbacks for querying objects with query methods like `Find`, `First`, `Related`, `Association`...
// Refer `Create` for usage
// 创建一个CallbackProcessor 的结构体，在这个结构体上注册信息
func (c *Callback) Query() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "query", parent: c}
}
~~~

~~~go
// Register a new callback, refer `Callbacks.Create`
func (cp *CallbackProcessor) Register(callbackName string, callback func(scope *Scope)) {
	if cp.kind == "row_query" {
		if cp.before == "" && cp.after == "" && callbackName != "gorm:row_query" {
			cp.logger.Print(fmt.Sprintf("Registering RowQuery callback %v without specify order with Before(), After(), applying Before('gorm:row_query') by default for compatibility...\n", callbackName))
			cp.before = "gorm:row_query"
		}
	}
	//继续赋值CallbackProcessor的属性，并把CallbackProcessor结构体追加到processors slice中
	cp.name = callbackName
	cp.processor = &callback
	cp.parent.processors = append(cp.parent.processors, cp)
  // 通过record，区分是 create update，并追加到不同的属性上
	cp.parent.reorder()
}
~~~



~~~go
// reorder all registered processors, and reset CRUD callbacks
func (c *Callback) reorder() {
	var creates, updates, deletes, queries, rowQueries []*CallbackProcessor

	for _, processor := range c.processors {
		if processor.name != "" {
			switch processor.kind {
			case "create":
				creates = append(creates, processor)
			case "update":
				updates = append(updates, processor)
			case "delete":
				deletes = append(deletes, processor)
			case "query":
				queries = append(queries, processor)
			case "row_query":
				rowQueries = append(rowQueries, processor)
			}
		}
	}
	// 根据CallbackProcessor 的before after 等信息，最这个slice进行排序，以便顺序调用
	c.creates = sortProcessors(creates)
	c.updates = sortProcessors(updates)
	c.deletes = sortProcessors(deletes)
	c.queries = sortProcessors(queries)
	c.rowQueries = sortProcessors(rowQueries)
}
~~~

到这里，我们就看到了这些回调函数是怎么注册上来的，后面就是看一下，Query对应的回调函数，到底干了什么，才会导致 First 返回`ErrRecordNotFound` , 而Find 返回nil的区别

## 回调

~~~go
func queryCallback(scope *Scope) {
	if _, skip := scope.InstanceGet("gorm:skip_query_callback"); skip {
		return
	}

	//we are only preloading relations, dont touch base model
	if _, skip := scope.InstanceGet("gorm:only_preload"); skip {
		return
	}

	defer scope.trace(scope.db.nowFunc())

	var (
		isSlice, isPtr bool
		resultType     reflect.Type
		results        = scope.IndirectValue()
	)
	// 判断是否有根据primary key 排序的设置，有的话，追加order排序条件
	if orderBy, ok := scope.Get("gorm:order_by_primary_key"); ok {
		if primaryField := scope.PrimaryField(); primaryField != nil {
			scope.Search.Order(fmt.Sprintf("%v.%v %v", scope.QuotedTableName(), scope.Quote(primaryField.DBName), orderBy))
		}
	}

	if value, ok := scope.Get("gorm:query_destination"); ok {
		results = indirect(reflect.ValueOf(value))
	}
	// 判断接收数据的变量的类型，并进行处理
	if kind := results.Kind(); kind == reflect.Slice {
		isSlice = true
		resultType = results.Type().Elem()
		results.Set(reflect.MakeSlice(results.Type(), 0, 0))

		if resultType.Kind() == reflect.Ptr {
			isPtr = true
			resultType = resultType.Elem()
		}
	} else if kind != reflect.Struct {
		scope.Err(errors.New("unsupported destination, should be slice or struct"))
		return
	}
	// 拼接sql
	scope.prepareQuerySQL()

	if !scope.HasError() {
		scope.db.RowsAffected = 0
		if str, ok := scope.Get("gorm:query_option"); ok {
			scope.SQL += addExtraSpaceIfExist(fmt.Sprint(str))
		}
		// 执行拼接好的sql
		if rows, err := scope.SQLDB().Query(scope.SQL, scope.SQLVars...); scope.Err(err) == nil {
			defer rows.Close()

			columns, _ := rows.Columns()
			for rows.Next() {
        // 记录RowsAffected
				scope.db.RowsAffected++

				elem := results
				if isSlice {
					elem = reflect.New(resultType).Elem()
				}

				scope.scan(rows, columns, scope.New(elem.Addr().Interface()).Fields())
				// 数据追加到接收的结果集里面
				if isSlice {
					if isPtr {
						results.Set(reflect.Append(results, elem.Addr()))
					} else {
						results.Set(reflect.Append(results, elem))
					}
				}
			}
			// 判断是否有错，有错误，就返回错误信息
			if err := rows.Err(); err != nil {
				scope.Err(err)
			} else if scope.db.RowsAffected == 0 && !isSlice {
        // 如果RowsAffected == 0，也就是没有数据，且接收的结果集不是 slice，就报 ErrRecordNotFound 的错误
				scope.Err(ErrRecordNotFound)
			}
		}
	}
}
~~~

至此，我们可以看出，跟Find 还是 First函数没有太大的关系，而是跟传递的接收结果的变量有关，如果接收结果的变量时slice，那么就不会报`ErrRecordNotFound` 



# 进一步验证

我们修改一下demo中的main函数

~~~go
func main() {
	db := open()
	user := &User{}
	users := make([]*User, 0, 0)
	err := db.Model(user).Where("id = ?", 1).First(&users).Error
	db.Table(user.TableName()).Model(user)
	fmt.Println(err, user)

	err = db.Model(user).Where("id = ?", 1).Find(user).Error
	fmt.Println(err, user)
}
~~~

这时候，按照我们追踪源码得到的结果，应该返回

~~~
<nil> &{0  }
record not found &{0  }
~~~

打印出来结果如下

~~~
<nil> &{0  }
record not found &{0  }
~~~

甚是完美

# 总结

传入接收结果集的变量只能为Struct类型或Slice类型，当传入变量为Struc类型时，如果检索出来的数据为0条，会抛出ErrRecordNotFound错误，当传入变量为Slice类型时，任何条件下均不会抛出ErrRecordNotFound错误