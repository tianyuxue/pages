---
title: "Kratos源码分析 - http部分"
date: "2021-01-19"
categories: 
  - "golang"
tags: 
  - "http"
  - "kratos"
---

> 本文的讨论基于Kratos v1.0.x版本

Kratos是bilibili开源的一套Go微服务框架，包含大量微服务相关框架及工具，本文主要从源码角度分析一下Kratos中与缓存相关的代码，在分析的过程中，我会从Kratos提供的不同的功能点来结合自己的一些理解进行阐述。

<!--more-->

## 1 核心对象的分析

Kratos http模块叫做blademaster，其主要设计参考了借鉴Gin的代码。整个http 模块可以拆分如下3个部分：

![](images/bm-arch-2-2.png)

图 1 blademaster模块架构

- Router对象借鉴了Gin的设计，使用**Radix Tree**存储了全部的路由信息
- Context封装了一个http请求相关的所有上下文信息，Context的存在统一了Handler的接口
- Handlers对象是**职责链模式**的实现，用于处理http请求。对于每一个请求，有一串handler依次处理，一个handler处理成功后可以交给下一个，也可以根据情况直接将http响应返回给客户端。这三个组件的交互见下图：

![](images/bm-arch-2-3.png)

图 2 blademaster组件交互图

Kratos使用`struct Engine`封装了上面3个对象，Engine对象的定义如下：

type Engine struct {
	RouterGroup  // 用于注册路由信息

	lock sync.RWMutex
	conf \*ServerConfig // 配置信息

	address string  // 服务地址

	trees     methodTrees                       // 基数树存储路由信息
	server    atomic.Value                      // store \*http.Server
	metastore map\[string\]map\[string\]interface{} // metastore is the path as key and the metadata of this path as value, it export via /metadata

	pcLock        sync.RWMutex
	methodConfigs map\[string\]\*MethodConfig

	injections \[\]injection

	// If enabled, the url.RawPath will be used to find parameters.
	UseRawPath bool

	// If true, the path value will be unescaped.
	// If UseRawPath is false (by default), the UnescapePathValues effectively is true,
	// as url.Path gonna be used, which is already unescaped.
	UnescapePathValues bool

	// If enabled, the router checks if another method is allowed for the
	// current route, if the current request can not be routed.
	// If this is the case, the request is answered with 'Method Not Allowed'
	// and HTTP status code 405.
	// If no other Method is allowed, the request is delegated to the NotFound
	// handler.
	HandleMethodNotAllowed bool

	allNoRoute  \[\]HandlerFunc
	allNoMethod \[\]HandlerFunc
	noRoute     \[\]HandlerFunc
	noMethod    \[\]HandlerFunc

	pool sync.Pool // Context池，复用Context对象，减少GC压力
}

每一个`Context`对象中保存了它所需要的Handler，`Context`对象的定义如下:

type Context struct {
	context.Context  // 提供golang标准context的接口

	Request \*http.Request  // http请求的内存对象
	Writer  http.ResponseWriter // http响应的内存对象

	// flow control
	index    int8  //当前执行到了第几个handler
	handlers \[\]HandlerFunc //所有需要执行的handler

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map\[string\]interface{}
	// This mutex protect Keys map
	keysMutex sync.RWMutex

	Error error

	method string
	engine \*Engine

	RoutePath string

	Params Params
}

这样通过这Context + Handler，通过职责链模式提供了很高的代码扩展性，尤其契合Http请求处理这种场景。下面具体探讨下Http服务器需要重点关注的一些内容：

- IO模型
- 路由信息的处理

### 1.1 IO模型

blademaster直接使用了golang官方的`http.Server`包。在Engine对象的启动代码`Run()`方法可以看到：

func (engine \*Engine) Run(addr ...string) (err error) {
	address := resolveAddress(addr)
	server := &http.Server{
		Addr:    address,
		Handler: engine,
	}
	engine.server.Store(server)
	if err = server.ListenAndServe(); err != nil {
		err = errors.Wrapf(err, "addrs: %v", addr)
	}
	return
}

代码中启动了`http.Server`，在`Engine.ServeHttp()`方法（代码如下）中可以看到，Engine接管了所有的http请求，每当请求到来时候，就从Context Pool中获取一个Context对象，处理完毕后归还到Context Pool中。

// ServeHTTP conforms to the http.Handler interface.
func (engine \*Engine) ServeHTTP(w http.ResponseWriter, req \*http.Request) {
	c := engine.pool.Get().(\*Context)
	c.Request = req
	c.Writer = w
	c.reset()

	engine.handleContext(c)
	engine.pool.Put(c)
}

这样就能看出blademaster并未实现自己的线程模型，完全使用了官方`http`包中**一个连接一个goroutine的线程模型**，所有处理连接的goroutine会调用阻塞式的读写方法，一旦数据未准备好，就会进入阻塞状态，数据到达后由netpoller唤醒阻塞的goroutine进行处理，因此，Kratos 的http线程模型可以视为**单Reactor多线程模式**。

### 1.2 路由信息的存储

在Http服务器处理业务请求的时候，查询最多的就是路由信息了，一个大型应用可能会注册很多路由信息，在高并发下被大量查询，因此其查询效率非常重要。众所周知，哈希表的O(1)查询效率当然是最快的，将所有路由信息存储到hash表里能解决问题么？如果路由信息都是常量，这样是最简单的方式。但是路由信息中还存在很多参数，尤其在restful风格的接口设计中，因此blademaster或者说是Gin采用了**基数树**这个数据结构来存储。

在blademaster的代码中，每一个Http方法，比如GET、POST方法，其路由信息存储在一个基数树中，所有Http方法的路由信息构成了一个森林，存储在`Engine`结构体的`methodTrees`对象中。树中每一个节点的定义如下：

type node struct {
	path      string // 存储的路径
	indices   string
	children  \[\]\*node // 孩子节点
	handlers  \[\]HandlerFunc // 该路径的处理函数
	priority  uint32
	nType     nodeType  // 节点类型
	maxParams uint8
	wildChild bool // 是否是含有url参数的节点
}

通过基数树这种数据结构，解决了带参数URL的查询效率问题。具体的基数树的代码过于细节，本文主要讨论blademaster的实现，这里就不再深入，后续我计划专门写一篇文章来理解基数树在Gin中的使用。

## 2 部分常用业务功能的实现

### 2.1 CORS

CORS用来支持浏览器跨域请求，Kratos的http模块提供了支持CORS的方法，其核心功能很简单：

- 预先配置允许跨域访问的地址
- 当CORS请求到达之后，验证与配置的地址是否匹配
- 验证成功后返回特定的Http Header给客户端，否则返回错误信息

Kratos通过传入**地址数组**创建出CORS middleware，创建代码如下：

// CORS returns the location middleware with default configuration.
// 传入允许访问的跨域地址
func CORS(allowOriginHosts \[\]string) HandlerFunc {
	config := &CORSConfig{
		AllowMethods:     \[\]string{"GET", "POST"},
		AllowHeaders:     \[\]string{"Origin", "Content-Length", "Content-Type"},
		AllowCredentials: true,
		MaxAge:           time.Duration(0),
		AllowOriginFunc: func(origin string) bool {
			for \_, host := range allowOriginHosts {
				if strings.HasSuffix(strings.ToLower(origin), host) {
					return true
				}
			}
			return false
		},
	}
	return newCORS(config)
}

// newCORS returns the location middleware with user-defined custom configuration.
// 返回cors middleware
func newCORS(config \*CORSConfig) HandlerFunc {
	if err := config.Validate(); err != nil {
		panic(err.Error())
	}
	cors := &cors{
		allowOriginFunc:  config.AllowOriginFunc,
		allowAllOrigins:  config.AllowAllOrigins,
		allowCredentials: config.AllowCredentials,
		allowOrigins:     normalize(config.AllowOrigins),
		normalHeaders:    generateNormalHeaders(config),
		preflightHeaders: generatePreflightHeaders(config),
	}

	return func(c \*Context) {
		cors.applyCORS(c)
	}
}

`cors.applyCORS()` 方法中实现CORS的验证逻辑，代码如下，具体细节可以参考注释。

func (cors \*cors) applyCORS(c \*Context) {
	origin := c.Request.Header.Get("Origin")
	// 通过http header中的orgin判断是否是跨域请求
	if len(origin) == 0 {
		// request is not a CORS request
		return
	}
	// 通过当前请求域名与允许的域名匹配
	if !cors.validateOrigin(origin) {
		log.V(5).Info("The request's Origin header \`%s\` does not match any of allowed origins.", origin)
		// 如果不匹配，就返回403状态码
		c.AbortWithStatus(http.StatusForbidden)
		return
	}

	// 对于OPTIONS请求进行特殊处理
	if c.Request.Method == "OPTIONS" {
		cors.handlePreflight(c)
		defer c.AbortWithStatus(200)
	} else {
		// http响应的header中添加CORS相关的header
		cors.handleNormal(c)
	}

	if !cors.allowAllOrigins {
		header := c.Writer.Header()
		header.Set("Access-Control-Allow-Origin", origin)
	}
}

### 2.2 CSRF

CSRF指的是跨站请求伪造攻击，通常造成影响较为严重。对CSRF的防护手段主要是，当表单展示的时候，在表单中增加验证的token，当表单提交后也带这这个token来确保这个表单提交不是伪造的。

Kratos对CSRF的防护做的很基础，只是检查了Http Header中的Referer字段是否合法，代码如下：

// CSRF returns the csrf middleware to prevent invalid cross site request.
// Only referer is checked currently.
func CSRF(allowHosts \[\]string, allowPattern \[\]string) HandlerFunc {
	validations := \[\]func(\*url.URL) bool{}

	addHostSuffix := func(suffix string) {
		validations = append(validations, matchHostSuffix(suffix))
	}
	addPattern := func(pattern string) {
		validations = append(validations, matchPattern(regexp.MustCompile(pattern)))
	}

	for \_, r := range allowHosts {
		addHostSuffix(r)
	}
	for \_, p := range allowPattern {
		addPattern(p)
	}

	return func(c \*Context) {
		// 获取Header中的Refer字段
		referer := c.Request.Header.Get("Referer")
		if referer == "" {
			log.V(5).Info("The request's Referer or Origin header is empty.")
			c.AbortWithStatus(403)
			return
		}
		illegal := true

		// 校验是否符合预定义的地址
		if uri, err := url.Parse(referer); err == nil && uri.Host != "" {
			for \_, validate := range validations {
				if validate(uri) {
					illegal = false
					break
				}
			}
		}
		if illegal {
			log.V(5).Info("The request's Referer header \`%s\` does not match any of allowed referers.", referer)
			// 不符合返回403
			c.AbortWithStatus(403)
			return
		}
	}
}

对于需要严格防护CSRF攻击的场景，还需要使用其他第三方的库，或者自己实现解决方案。

### 2.3 限流

kratos 借鉴了 Sentinel 项目的自适应限流系统，通过综合分析服务的 cpu 使用率、请求成功的 qps 和请求成功的 rt 来做自适应限流保护。这篇文章只讨论http模块，具体的限流算法我会另外选择文章来讨论。Kratos也将限流功能以中间件方式实现：

// Limit return a bm handler func.
func (b \*RateLimiter) Limit() HandlerFunc {
	return func(c \*Context) {
		uri := fmt.Sprintf("%s://%s%s", c.Request.URL.Scheme, c.Request.Host, c.Request.URL.Path)
		limiter := b.group.Get(uri)
		// 执行限流
		done, err := limiter.Allow(c)
		if err != nil { // 流量被限制后返回错误
			\_metricServerBBR.Inc(uri, c.Request.Method)
			c.JSON(nil, err)
			c.Abort()
			return
		}
		// 成功处理后更新统计信息，作为后续限流依据
		defer func() {
			done(limit.DoneInfo{Op: limit.Success})
			b.printStats(uri, limiter)
		}()
		c.Next()
	}
}

可以看出，中间件的抽象有助于代码的扩展，可以根据项目情况灵活的选用不同的限流方式。

### 2.4 日志

blademaster中的日志功能与其他框架类似，提供了http请求相关信息的记录，Log中间件的具体代码如下：

// Logger is logger  middleware
func Logger() HandlerFunc {
	const noUser = "no\_user"
	return func(c \*Context) {
		// 获取执行剩余handler前的信息
		now := time.Now()
		req := c.Request
		path := req.URL.Path
		params := req.Form
		var quota float64
		if deadline, ok := c.Context.Deadline(); ok {
			quota = time.Until(deadline).Seconds()
		}
		// 执行剩余handler
		c.Next()

		// 获取handler执行完毕后的信息
		err := c.Error
		cerr := ecode.Cause(err)
		// 请求处理的时间
		dt := time.Since(now)
		caller := metadata.String(c, metadata.Caller)
		if caller == "" {
			caller = noUser
		}

		if len(c.RoutePath) > 0 {
			\_metricServerReqCodeTotal.Inc(c.RoutePath\[1:\], caller, req.Method, strconv.FormatInt(int64(cerr.Code()), 10))
			\_metricServerReqDur.Observe(int64(dt/time.Millisecond), c.RoutePath\[1:\], caller, req.Method)
		}

		lf := log.Infov
		errmsg := ""
		isSlow := dt >= (time.Millisecond \* 500)
		// 通过是否有错误和处理时间是否过长来决定是不是使用warn级别的日志输出
		if err != nil {
			errmsg = err.Error()
			lf = log.Errorv
			if cerr.Code() > 0 {
				lf = log.Warnv
			}
		} else {
			if isSlow {
				lf = log.Warnv
			}
		}
		// 记录日志
		lf(c,
			log.KVString("method", req.Method),
			log.KVString("ip", c.RemoteIP()),
			log.KVString("user", caller),
			log.KVString("path", path),
			log.KVString("params", params.Encode()),
			log.KVInt("ret", cerr.Code()),
			log.KVString("msg", cerr.Message()),
			log.KVString("stack", fmt.Sprintf("%+v", err)),
			log.KVString("err", errmsg),
			log.KVFloat64("timeout\_quota", quota),
			log.KVFloat64("ts", dt.Seconds()),
			log.KVString("source", "http-access-log"),
		)
	}
}

这个方法在handler开始处理前后记录时间，用户等想关元信息，根据处理结果是否出错以及处理时间是否过长来决定日志的输出级别。

### 2.5 Trace

blademaster中Trace信息的记录方式同日志信息一样，其实现代码如下：

// Trace is trace middleware
func Trace() HandlerFunc {
	return func(c \*Context) {
		// 从http request header 中获取trace信息
		t, err := trace.Extract(trace.HTTPFormat, c.Request.Header)
		if err != nil {
			var opts \[\]trace.Option
			if ok, \_ := strconv.ParseBool(trace.KratosTraceDebug); ok {
				opts = append(opts, trace.EnableDebug())
			}
			// 如果trace 信息不存在，就根据当前url创建一个trace单元
			t = trace.New(c.Request.URL.Path, opts...)
		}
		// 设定trace信息 包含标题和tag
		t.SetTitle(c.Request.URL.Path)
		t.SetTag(trace.String(trace.TagComponent, \_defaultComponentName))
		t.SetTag(trace.String(trace.TagHTTPMethod, c.Request.Method))
		t.SetTag(trace.String(trace.TagHTTPURL, c.Request.URL.String()))
		t.SetTag(trace.String(trace.TagSpanKind, "server"))
		// business tag
		t.SetTag(trace.String("caller", metadata.String(c.Context, metadata.Caller)))

		// 把trace信息加入到http header中，让用户可以查看，便于debug，同时也让下游代码可以继续记录trace信息
		c.Writer.Header().Set(trace.KratosTraceID, t.TraceID())
		// 根据trace信息，新建一个官方包里的context
		c.Context = trace.NewContext(c.Context, t)
		// 调用剩余的handler
		c.Next()
		// 标记trace信息已经完成
		t.Finish(&c.Error)
	}
}

要记录trace信息，blademaster首先尝试从当前http request header中获取已经存在trace信息，如果trace信息不存在，说明这是链路调用的开始，就依据当前请求信息新建一个Trace单元。同样的，在handler执行前后，记录相关信息到trace单元中，然后将trace信息存放在**http response header**中，方便下游系统处理。trace信息中比较关键的要素有：

- trace id
- url地址
- 业务模块相关的信息
- 处理结果信息

### 2.6 Recovery

上面已经讨论过，blademaster使用的一个请求一个goroutine的IO模型，如果goroutine出现panic怎么办？这时候u需要把错误信息传递给调用方，但是panic会导致整个进程挂掉，这显然是不合理的，blademaster的recovery中间件就是用来**将panic信息转化为http 500的错误信息**，即保证进程运行，也把错误信息返回给客户端，类似于Spring中的Global Exception Handler。其实现代码如下：

// Recovery returns a middleware that recovers from any panics and writes a 500 if there was one.
func Recovery() HandlerFunc {
	return func(c \*Context) {
		// 通过defer方法从panic中恢复
		defer func() {
			var rawReq \[\]byte
			if err := recover(); err != nil {
				const size = 64 << 10
				buf := make(\[\]byte, size)
				// 获取出错的栈帧信息
				buf = buf\[:runtime.Stack(buf, false)\]
				// 获取出错的http请求的详细信息
				if c.Request != nil {
					rawReq, \_ = httputil.DumpRequest(c.Request, false)
				}
				pl := fmt.Sprintf("http call panic: %s\\n%v\\n%s\\n", string(rawReq), err, buf)
				fmt.Fprintf(os.Stderr, pl)
				// 记录错误日志
				log.Error(pl)
				// 返回500错误信息
				c.AbortWithStatus(500)
			}
		}()
		c.Next()
	}
}

从上述代码中可以看到，Recovery中间件`recovery()`函数从panic中恢复，并返回给客户端错误信息，这也意味着Recovery中间件应该放在所有中间件的**最前面**。

## 3 总结

本文讨论kratos blademaster模块的一些实现细节，可以看出，得益与职责链模式的使用，中间件的扩展非常灵活。
