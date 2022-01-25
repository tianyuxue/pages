---
title: "Kratos 源码分析 - MySQL部分"
date: "2021-01-07"
categories: 
  - "golang"
  - "kratos"
  - "mysql"
tags: 
  - "kratos"
  - "mysql"
---

> 本文的讨论基于Kratos v1.0.x版本

Kratos是bilibili开源的一套Go微服务框架，包含大量微服务相关框架及工具，本文主要从源码角度分析一下Kratos中与MySQL相关的代码，在分析的过程中，我会从Kratos提供的不同的功能点来结合自己的一些理解进行阐述（这篇文章没有复杂的原理，只有从工程角度的一些最佳实践)。

<!--more-->

## 1 核心对象的封装

Kratos的数据层代码主要是对Golang SDK的二次封装。在GolangSDK中，database/sql包提供了对SQL的支持，具体而言，这个包提供了如下的抽象：

- `DB` 抽象表示数据库本身，也可以将其理解为数据库连接池
- `Conn` 抽象表示单独一个数据库连接
- `Tx` 抽象表示一次数据库事务操作
- `Stmt` 抽象表示一个Prepared Statement
- `Row/Rows` 抽象表示一次数据库交互后的结果

`database/sql`包提供的这些抽象接口是直观的，易用的，足够应付简单的应用场景。Kratos对上面这几个对象再次进行了封装，提供了实际开发中经常用到的如下功能：

- MySQL读写分离
- 链路追踪
- 统计信息
- 慢查询日志记录
- 融断保护

下面我从代码的角度分析下Kratos是如何实现上述功能的。

## 2 功能点的实现

### 2.1 MySQL读写分离

Kratos从DAO层的代码层面实现了对MySQL读写分离的支持，并未使用数据库中间件。首先看下Kratos需要用户提供的MySQL集群配置信息：

```
pkg/database/sql/mysql.go
type Config struct {
    DSN          string          // write data source name.
    ReadDSN      []string        // read data source name
    Active       int             // pool
    Idle         int             // pool
    IdleTimeout  time.Duration   // connect max life time.
    QueryTimeout time.Duration   // query sql timeout
    ExecTimeout  time.Duration   // execute sql timeout
    TranTimeout  time.Duration   // transaction sql timeout
    Breaker      *breaker.Config // breaker
}
```

可以看出，`DSN`和`ReadDSN`分别存储了MySQL主节点地址和从节点地址，从节点地址有多个。Kratos会根据上述配置，生成如下的连接池对象：

```
type DB struct {
write  *conn   //主节点连接池，conn定义见下文
read   []*conn //从节点连接池
idx    int64   // 用于选取从节点的id
master *DB      //MySQL的主节点
}
```

DB对象使用`master`字段冗余存储主节点的信息，用来支持一些特定业务场景下，必须查询主库的操作。`write/read`字段分别表示数据库具体连接池，从`conn`类型的定义可以看到，其内部封装了Golang SDK中的 `sql.DB`对象：

```
// conn database connection
type conn struct {
    *sql.DB  // 对Golang SDK连接池的封装，用来支持MySQL读写分离
    breaker breaker.Breaker
    conf    *Config
    addr    string
}
```

这样通过把Golang SDK中的`sql.DB`嵌入到`conn`对象中，再把多个`conn`对象封装到`DB`对象中，就使Kratos的`DB`对象支持了对MySQL集群的多个实例的读写。下面是Kratos中查询从库的方法：

```
// Query executes a query that returns rows, typically a SELECT. The args are
// for any placeholder parameters in the query.
func (db *DB) Query(c context.Context, query string, args ...interface{}) (rows *Rows, err error) {
        // 获取要读取哪一个从库
	idx := db.readIndex()
	for i := range db.read {
		// 依次尝试每一个从库, 只要一个成功了就返回
		if rows, err = db.read[(idx+i)%len(db.read)].query(c, query, args...); !ecode.EqualError(ecode.ServiceUnavailable, err) {
			return
		}
	}
	// 从库失败了的话就查询主库
	return db.write.query(c, query, args...)
}

...

// 简单的使用轮询策略选择要读取的从库
func (db *DB) readIndex() int {
	if len(db.read) == 0 {
		return 0
	}
	v := atomic.AddInt64(&db.idx, 1)
	return int(v) % len(db.read)
}
```

从`readIndex()`方法中可以看到，Kratos在查询从库时候使用了简单轮寻的方法，每次查询前使用cas方式将内部变量idx自增，然后通过取mod的方式确定要查询哪一个从库。`Query()`方法中循环尝试读取每一个从库，**只有读取成功了一次后才返回**，这样避免了个别从库故障导致的读取失败。

除了读操作之外，对于写入的操作，Kratos同样对Golang SDK中sql.DB的如下方法进行了封装：

```
// 启动MySQL事务
func (db *DB) Begin(c context.Context) (tx *Tx, err error) {
	return db.write.begin(c)
}

// 执行无返回结果的SQL语句
func (db *DB) Exec(c context.Context, query string, args ...interface{}) (res sql.Result, err error) {
	return db.write.exec(c, query, args...)
}

// 创建Prepared Statement，如果出错会返回错误信息
func (db *DB) Prepare(query string) (*Stmt, error) {
	return db.write.prepare(query)
}

// 创建Prepared Statement，如果出错会后台重试
func (db *DB) Prepared(query string) (stmt *Stmt) {
	return db.write.prepared(query)
}

```
通过上述几个方法的封装，Kratos直接使用主库的数据库连接池进行插入删除等操作。这样Kratos在DAO的代码层面实现对Golang原生`database/sql`包的封装。

再看一下prepare statement的创建, Kratos的提供了`Prepared()`方法，如果创建prepare statement错误，那么启动一个goroutine不断重试，直到创建成功了，就用cas方法把prepare statement存储在Kratos封装的`Stmt`对象中， 个人感觉这样不够优雅，不知道b站内部是怎么用这个方法的：

```
// Stmt prepared stmt.
type Stmt struct {
	db    *conn
	tx    bool
	query string
	stmt  atomic.Value  // 存储创建好的prepare statement
	t     trace.Trace
}

...

func (db *conn) prepare(query string) (*Stmt, error) {
	defer slowLog(fmt.Sprintf("Prepare query(%s)", query), time.Now())
	stmt, err := db.Prepare(query)
	if err != nil {
		err = errors.Wrapf(err, "prepare %s", query)
		return nil, err
	}
	st := &Stmt{query: query, db: db}
	st.stmt.Store(stmt)
	return st, nil
}

func (db *conn) prepared(query string) (stmt *Stmt) {
	defer slowLog(fmt.Sprintf("Prepared query(%s)", query), time.Now())
	stmt = &Stmt{query: query, db: db}
	s, err := db.Prepare(query)
	if err == nil {
		stmt.stmt.Store(s)
		return
	}
	// 如果执行创建prepare出错了，这里后台执行不断重试，直到成功为止
	go func() {
		for {
			s, err := db.Prepare(query)
			if err != nil {
				time.Sleep(time.Second)
				continue
			}
			stmt.stmt.Store(s)
			return
		}
	}()
	return
}
```

除了MySQL读写分离，Kratos对其他方法的封装主要是为了提供慢查询日志，trace日志，监控信息和熔断保护这四个功能。

### 2.2 慢查询日志

Kratos在每个与数据库交互方法中记录了慢查询日志，结合关键字`defer`，其实现方法非常简单：

```
...
func (db *conn) Query(c context.Context) (tx *Tx, err error) {
	// 获取当前时间，通过defer来确定结束时间
        now := time.Now()
	defer slowLog("Begin", now)

...

// 时间超过阈值，就打印一条警告记录
func slowLog(statement string, now time.Time) {
	du := time.Since(now)
	if du > _slowLogDuration {
		log.Warn("%s slow log statement: %s time: %v", _family, statement, du)
	}
}
```

与数据库交互的每个方法中，Kratos都加入了上述代码片段记录慢查询的警告日志。

### 2.3 监控信息

在数据库链接层面，Kratos封装了prometheus客户端代码，提供了4个主要的监控信息：

- 请求处理时长
- 错误请求数
- 数据库连接总数
- 当前数据库连接数

具体的代码如下：
```
pkg/database/sql/metrics.go

package sql

import "github.com/go-kratos/kratos/pkg/stat/metric"

const namespace = "mysql_client"

var (
	_metricReqDur = metric.NewHistogramVec(&metric.HistogramVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "duration_ms",
		Help:      "mysql client requests duration(ms).",
		Labels:    []string{"name", "addr", "command"},
		Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000, 2500},
	})
	_metricReqErr = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "requests",
		Name:      "error_total",
		Help:      "mysql client requests error count.",
		Labels:    []string{"name", "addr", "command", "error"},
	})
	_metricConnTotal = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "total",
		Help:      "mysql client connections total count.",
		Labels:    []string{"name", "addr", "state"},
	})
	_metricConnCurrent = metric.NewGaugeVec(&metric.GaugeVecOpts{
		Namespace: namespace,
		Subsystem: "connections",
		Name:      "current",
		Help:      "mysql client connections current.",
		Labels:    []string{"name", "addr", "state"},
	})
)
```

Kratos已经把上面监控数据嵌入在封装好的数据库交互的方法中，例如数据库读写发生错误后会调用`_metricReqErr.Inc()`, 读写完成后会调用`_metricReqDur.Observe()`。

### 2.4 断路器

Kratos的DB对象内置了断路器，存储在DB对象的breaker字段中，断路器的接口很简单，主要提供了如下三个方法：

```
// Breaker 定义了断路器接口
type Breaker interface {
	Allow() error
	MarkSuccess()
	MarkFailed()
}

...
//在连接发出请求前判断熔断器状态
if err = conn.breaker.Allow(); err != nil {
       return
}

//连接执行成功或失败将结果告知breaker
if(respErr != nil){
     conn.breaker.MarkFailed()
}else{
     conn.breaker.MarkSuccess()
}

Kratos在与数据库交互前会检查断路器状态，执行完SQL语句后更新断路器状态，例如在exec()方法中：

func (db *conn) exec(c context.Context, query string, args ...interface{}) (res sql.Result, err error) {
        ...
        // 执行方法前确认断路器状态
	if err = db.breaker.Allow(); err != nil {
		_metricReqErr.Inc(db.addr, db.addr, "exec", "breaker")
		return
	}
	_, c, cancel := db.conf.ExecTimeout.Shrink(c)
	res, err = db.ExecContext(c, query, args...)
	cancel()
        // 方法执行完毕后更新断路器状态
	db.onBreaker(&err)
	_metricReqDur.Observe(int64(time.Since(now)/time.Millisecond), db.addr, db.addr, "exec")
	if err != nil {
		err = errors.Wrapf(err, "exec:%s, args:%+v", query, args)
	}
	return
}
```

### 2.5 trace信息

Kratos通过context来传递trace信息，见如下代码：

```
if t, ok := trace.FromContext(c); ok {
        t = t.Fork(_family, "exec")
	t.SetTag(trace.String(trace.TagAddress, db.addr), trace.String(trace.TagComment, query))
	defer t.Finish(&amp;err)
}
```

在从context获取了trace的记录单元后，同样使用了defer关键字记录了日志的终止信息。上述代码片段在每一个数据库交互方法中都存在，这样就实现了在读写数据库层面的trace信息记录。

## 3 总结

本文从源码层面讨论了Kratos框架对读写MySQL提供的一些工程上的最佳实践。
