---
title: "Kratos源码分析 - 缓存部分"
date: "2021-01-11"
cover:
    image: "images/mountain.jpg"
    hidden: false
categories: 
  - "golang"
  - "kratos"
tags: 
  - "cache"
  - "golang"
  - "kratos"
---

> 本文的讨论基于Kratos v1.0.x版本

Kratos是bilibili开源的一套Go微服务框架，包含大量微服务相关框架及工具，本文主要从源码角度分析一下Kratos中与缓存相关的代码，在分析的过程中，我会从Kratos提供的不同的功能点来结合自己的一些理解进行阐述（这篇文章没有复杂的原理，只有从工程角度的一些最佳实践)。

<!--more-->

## 1 Kratos缓存部分提供的功能

Kratos提供了缓存相关的常用功能，见下表：

- 使用空缓存避免缓存穿透
- 缓存失效后，回源请求支持分批获取数据库数据，减少数据库访问延时
- 异步添加缓存
- 监控缓存回源比
- 缓存失效后，通过`singleflight`模式限制回源访问数据库的并发度，避免数据库压力过大

## 2 代码实现

Kratos提供了kratos tool genbts生成缓存回源代码，并且提供了一些配置，下面根据Kratos源码提供的示例代码进行分析，代码位于`tool/kartos-gen-bts/test-data/dao.bts.go`文件中的`Demos()`方法，具体含义可以参考代码中的注释：

```go
// Demos get data from cache if miss will call source method, then add to cache.
func (d *dao) Demos(c context.Context, keys []int64) (res map[int64]*Demo, err error) {
	if len(keys) == 0 {
		return
	}
	addCache := true
    // 首先尝试从缓存中加载数据
	if res, err = d.CacheDemos(c, keys); err != nil {
		addCache = false
		res = nil
		err = nil
	}
    // miss 用来存储缓存未命中的数据
	var miss []int64
	for _, key := range keys {
		if (res == nil) || (res[key] == nil) {
			miss = append(miss, key)
		}
	}
    // 统计缓存命中数量
	cache.MetricHits.Add(float64(len(keys)-len(miss)), "bts:Demos")
 
	for k, v := range res {
		if v.ID == -1 {
			delete(res, k)
		}
	}
	missLen := len(miss)
	if missLen == 0 {
		return
	}
	missData := make(map[int64]*Demo, missLen)
    // 统计缓存未命中的数量
        cache.MetricMisses.Add(float64(missLen), "bts:Demos")
	var mutex sync.Mutex

    // 通过errgroup包，分批从数据库中执行回源，请求数据量大的的情况下可以降低延迟
	group := errgroup.WithCancel(c)
	if missLen > 20 {
		group.GOMAXPROCS(20)
	}
	var run = func(ms []int64) {
		group.Go(func(ctx context.Context) (err error) {
			data, err := d.RawDemos(ctx, ms)
			mutex.Lock()
			for k, v := range data {
				missData[k] = v
			}
			mutex.Unlock()
			return
		})
	}
	var (
		i int
		n = missLen / 2
	)
	for i = 0; i < n; i++ {
		run(miss[i*2 : (i+1)*2])
	}
	if len(miss[i*2:]) > 0 {
		run(miss[i*2:])
	}
	err = group.Wait()
	if res == nil {
		res = make(map[int64]*Demo, len(keys))
	}
	for k, v := range missData {
		res[k] = v
	}
	if err != nil {
		return
	}
    // 数据库中不存在的数据也在缓存中存储一份默认值
	for _, key := range miss {
		if res[key] == nil {
			missData[key] = &Demo{ID: -1}
		}
	}
	if !addCache {
		return
	}
    // 通过异步的方法添加缓存，减少本次请求的延迟
	d.cache.Do(c, func(c context.Context) {
		d.AddCacheDemos(c, missData)
	})
	return
}
```

### 2.1 缓存穿透

缓存穿透指的是，在读数据的时候，没有命中缓存，请求“穿透”了缓存，**直接访问后端数据库**的情况。缓存穿透的情况下，缓存没有发挥作用，业务系统虽然去缓存查询数据，但缓存中没有数据，因此导致了业务系统需要再次去存储系统查询数据。通常不存在的数据访问应该不会太多，但是如果针对不存在数据进行大量恶意访问，就会导致这些请求全部被数据库处理，情况严重的话会造成整个业务系统不可用。这种情况的解决方案是，**对于不存在的请求，也在缓存中存放一份表示不存在的数据**，Kratos的代码中也是这样实现的。

Krato tool genbts 工具提供了配置项-`nullcache`来避免缓存穿透，配置了之后，生成的代码中会在回源之后增加判断，从上述代码的74-78行可以看到，从数据库读取数据之后，如果请求的缓存仍然为空，那么就缓存一个不存在的值，避免缓存穿透。

### 2.2 回源数据分批请求数据库

如果缓存未命中，就需要从数据库中回源访问，Kratos的回源访问支持多线程并发访问，见上述代码**38-62**行。上述代码首先使用了github.com/go-kratos/kratos/pkg/sync/errgroup包， 定义了回源函数，然后将总的数据库**请求分片**，分批执行。

这里errgroup包的作用是什么呢？errgroup实际上是一个并发工具，可以并发执行子任务，等待子任务返回，并提供每一个子任务的错误堆栈，执行出错返回的功能。其内部使用了channel存储要执行的任务，其核心对象如下：

```go
// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
type Group struct {
	err     error  // 最终的错误信息
	wg      sync.WaitGroup  // 用于确保所有子任务结束执行
	errOnce sync.Once

	workerOnce sync.Once
	ch         chan func(ctx context.Context) error  // 存储待处理的任务
	chs        []func(ctx context.Context) error

	ctx    context.Context
	cancel func()  // 出错后的取消函数
}

...

// Go方法增加子任务
func (g *Group) Go(f func(ctx context.Context) error) {
	g.wg.Add(1)
	if g.ch != nil {
		select {
		case g.ch <- f:
		default:
			g.chs = append(g.chs, f)
		}
		return
	}
	go g.do(f)
}

// Wait方法等待所有子任务结束
func (g *Group) Wait() error {
	if g.ch != nil {
		for _, f := range g.chs {
			g.ch <- f
		}
	}
	g.wg.Wait()
	if g.ch != nil {
		close(g.ch) // let all receiver exit
	}
	if g.cancel != nil {
		g.cancel()
	}
	return g.err
}
```

### 2.3 异步添加缓存

如果缓存未命中，从数据库回源之后，还需要将数据加入到缓存中。但是当前请求已经获取了客户端需要的数据，因此这时候可以采用异步添加缓存的策略，当前请求直接返回给客户端。从2.1 部分开始前的代码83-85行可以看到，Kratos使用了`github.com/go-kratos/kratos/pkg/sync/pipeline/fanout`包来异步增加缓存。

fanout包可以理解为一个用groutine实现的线程池，有goroutine数量、任务buffer大小两个核心参数，其内部使用了**buffer类型的channel来存储任务**，其处理任务的核心代码为：

```go
// New new a fanout struct.
func New(name string, opts ...Option) *Fanout {
	if name == "" {
		name = "anonymous"
	}
	o := &options{
		worker: 1,
		buffer: 1024,
	}
	for _, op := range opts {
		op(o)
	}
	c := &Fanout{
		ch:      make(chan item, o.buffer),
		name:    name,
		options: o,
	}
	c.ctx, c.cancel = context.WithCancel(context.Background())
	c.waiter.Add(o.worker)
    // 在这里启动worker
	for i := 0; i < o.worker; i++ {
		go c.proc()
	}
	return c
}
...
func (c *Fanout) proc() {
	defer c.waiter.Done()
	for {
		select {
        // 循环处理channel中待处理的任务
		case t := <-c.ch:
			wrapFunc(t.f)(t.ctx)
			_metricChanSize.Set(float64(len(c.ch)), c.name)
			_metricCount.Inc(c.name)
		case <-c.ctx.Done():
			return
		}
	}
}
```

### 2.4 监控缓存命中率

缓存监控的重点指标是缓存命中率，Kratos在也是在请求缓存之后，根据请求结果统计了如下数据：

- **请求缓存之后**记录缓存命中数
- **请求缓存之后**记录缓存未命中数

具体实现上，也是使用了封装的prometheus客户端代码提供监控入口：

```go
const _metricNamespace = "cache"

// be used in tool/kratos-gen-bts
var (
	MetricHits = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: _metricNamespace,
		Subsystem: "",
		Name:      "hits_total",
		Help:      "cache hits total.",
		Labels:    []string{"name"},
	})
	MetricMisses = metric.NewCounterVec(&metric.CounterVecOpts{
		Namespace: _metricNamespace,
		Subsystem: "",
		Name:      "misses_total",
		Help:      "cache misses total.",
		Labels:    []string{"name"},
	})
)
```

### 2.5 使用singleflight模式避免缓存失效引起的缓存雪崩

缓存数据本身是一个短时数据，超过一定的时间后，缓存可能会失效，也可能被业务系统主动驱逐。**"缓存雪崩**是指当缓存失效后引起**系统性能急剧下降**的情况。当缓存过期被清除后，业务系统需要重新生成缓存，因此需要再次访问存储系统，再次进行运算，这个处理步骤耗时几十毫秒甚至上百毫秒。而对于一个高并发的业务系统来说，几百毫秒内可能会接到几百上千个请求。**由于旧的缓存已经被清除，新的缓存还未生成，并且处理这些请求的线程都不知道另外有一个线程正在生成缓存**，因此所有的请求都会去重新生成缓存，都会去访问存储系统，从而对存储系统造成巨大的性能压力。这些压力又会拖慢整个系统，严重的会造成数据库宕机，从而形成一系列连锁反应，造成整个系统崩溃。"[1]

针对这个问题，Kratos使用了`singleflight`模式（https://github.com/golang/sync/blob/master/singleflight/singleflight.go），如果一个key失效了，同一时间只允许一个线程执行缓存更新操作。这个方法限制了缓存更新的并发度，有助于解决缓存雪崩问题。

要使用kratos tool genbts 生成singleflight模式的代码，需要增加如下`-singleflight=true`配置：

```go
// bts: -sync=true -nullcache=&Demo{ID:-1} -check_null_code=$.ID==-1 -singleflight=true
Demo(c context.Context, key int64) (*Demo, error)
```

生成代码中增加了一个全局变量`cacheSingleFlights`限制回源请求的并发度：

```go
var cacheSingleFlights = [1]*singleflight.Group{{}}
```

然后在具体的回源代码中：
```go
// Demo get data from cache if miss will call source method, then add to cache.
func (d *dao) Demo(c context.Context, key int64) (res *Demo, err error) {
	addCache := true
	res, err = d.CacheDemo(c, key)
	if err != nil {
		addCache = false
		err = nil
	}
	defer func() {
		if res.ID == -1 {
			res = nil
		}
	}()
	if res != nil {
		cache.MetricHits.Inc("bts:Demo")
		return
	}
	var rr interface{}
	sf := d.cacheSFDemo(key)
	rr, err, _ = cacheSingleFlights[0].Do(sf, func() (r interface{}, e error) {
		cache.MetricMisses.Inc("bts:Demo")
		r, e = d.RawDemo(c, key)
		return
	})
	res = rr.(*Demo)
	if err != nil {
		return
	}
	miss := res
	if miss == nil {
		miss = &Demo{ID: -1}
	}
	if !addCache {
		return
	}
	d.AddCacheDemo(c, key, miss)
	return
}
```

从18到28行的调用可以看到，Kratos使用全局变量cacheSingleFlights限制了数据库回源请求RawDemo()的并发度，如果看一下singleflight的实现，可以看到其**内部使用了map存储要执行的函数**，确保在同一时间，同一个函数只被执行一次。

singleflight模式只限制了，在缓存失效时，一个进程内只有一个请求到达数据库。在当前的微服务架构下，如果一个服务有上百个副本在运行，那么也有可能造成数据库压力增加，这种情况下可以采用分布式锁或者后台更新缓存的策略。

## 3 总结

本文从源码的角度讨论了Kratos生成的缓存回源代码的一些功能。

## 引用

[1] 高性能缓存架构 https://time.geekbang.org/column/article/8640
