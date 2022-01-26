---
title: "关于Golang GC的一些分析"
date: "2021-01-02"
categories: 
  - "golang"
tags: 
  - "gc"
  - "golang"
---

> 本文介绍了Golang中GC的执行过程，然后用一个例子，结合golang提供的GC日志和pprof工具，介绍GC调优的基本方法
> 
> 本文的讨论基于golang 1.15.4版本

<!--more-->

## 1 Golang采用的GC方案

Golang采用了**并发标记-清除**的GC方法，非分代，没有内存整理和移动，这与JVM有很大的不同。对于整个GC过程，源码`runtime/mgc.go`中给出了较为详细的描述，这里再做一个简要的梳理。整个GC分为以下4个阶段：

1.**sweep termination**

这个阶段发生在真正GC之前，用来清除上次GC后未被回收的内存，通常这个阶段不会发生，只有在执行了force gc的时候才会触发这个阶段，比如手动调用`runtime.GC()`函数。

2.**mark**

mark阶段分为如下几步：

1. 将变量`gcphase`从`_GCoff`设置为 `_GCmark`、开启写屏障功能、启动GC Assists。这个阶段会**STW(Stop The World)**，用户代码会暂停，直到所有P启动写屏障功能为止(如果对P的概念不理解，可以查看另一篇文章了解golang调度的原理)。
2. 写屏障启动之后会结束STW状态，用户代码可以正常执行，调度器启动GC Goroutine执行真正的标记任务，GC Assists开始接管GC过程中的内存分配，在这个过程中：
    
    - 写屏障会将GC过程中修改过的对象标记为灰色，GC过程中新分配的对象则直接标记为黑色，写屏障的作用是为了**维持三色不变性，保证三色标记GC算法正常工作**（三色标记算法过程后文介绍）
    
    - GC Assist用来**调节GC过程中的内存分配速度**，直观理解，就是GC过程中，用户Goroutine分配的内存越多，其分配速度越慢，甚至会被调度器执行抢占式调度。
3. GC Goroutine 会扫描**所有栈**，按照三色标记算法给扫描到的变量标记颜色, 扫描栈的时候，会暂停对应的goroutine, 扫描完成后再恢复gorouine的执行。
    - 三色标记算法的原理如下：
        
        - 初始所有变量为白色
        
        - 首先将全局变量、被栈引用的堆内对象标记为灰色
        - 随机选择一个灰色的对象，将其标记为黑色
            - 接着扫描该对象的所有可达对象，将可达对象也标记为灰色
        - 重复上述过程，直到没有灰色的对象为止
        - 这时候白色对象就是可以被回收的。可以看出在整个算法过程中，灰色表示待扫描的对象，黑色代表扫描完毕不可以回收的对象，白色代表可回收的对象，这个不变的特性就**称为三色不变性**。

3.**mark termination**

在GC Goroutine扫描完所有的栈之后进入mark termination阶段，这个阶段同样会**STW(Stop The World)**, 设置`gcphase`变量为`_GCmarktermination`, 然后执行一些资源清理的工作：

- 停止GC Goroutine
- 禁用GC Assists

4.**sweep**  
标记结束之后进入sweep阶段，设置`gcphace`为 _GCoff, 禁用写屏障, 在这之后会Start The World， 启动执行用户代码，这时候意味着并发标记已经结束了，剩下的就是内存回收的工作了。

- 从这个阶段开始，新的内存分配请求会**复用**被标记为可回收的内存，这意味着内存的回收是惰性的，不是立刻回收。
- 基于cpu使用情况，调度器可能启动额外的goroutine来执行内存回收

当新分配的内存达到GOGC设定的值后，再次触发GC, 重复上述步骤。

- GOGC用于调整GC频率，GOGC的默认值是100, 意味着下一次GC的触发时机是： 新分配的内存 = 当前GC结束后的剩余内存 × 100%， 例如当前GC结束后剩余4MB内存，当堆内存达到8MB时候，会再次触发GC

![](images/Screenshot_2021-01-02-GP_2019_-_An_Insight_Into_Go_Garbage_Collection-pdf.png)

GC过程示意图

整个GC的过程可以见上图。这里再罗嗦一下，Golang采用无内存复制/移动的GC算法，很大的原因是为了兼容谷歌内部的大量C和C++代码，如果GC导致了堆对象的地址变化，那么对于调用C和C++代码而言非常困难。而由于协程的使用，启动写屏障、调度器启动GC Goroutine等操作实际是不涉及内核线程的上下文切换的，因此STW速度很快，这也是并发标记清除算法名称的由来，因此我们看到了Golang这种跟JVM完全不同的内存回收方案。

## 2 实践GC调优

程序代码的执行必然伴随着内存的分配、修改和释放，GC将程序员从手动内存管理的繁琐工作中解放了出来，很大程度上提高了开发效率。就像任何事情一样，GC也不是免费的，GC的代价就在于消耗了CPU来执行了业务无关代码，从而造成用户代码执行的延迟。从这一点看GC调优与优化内存使用是密切相关的，造成GC过度延迟的根本原因就**在于不恰当的内存分配和使用**。

GC调优的目的很简单：节约堆内存，降低GC造成的CPU消耗，避免因为GC而导致用户代码执行延迟，尤其是在某些延迟敏感的特定场景。衡量GC损耗的关键指标有三个：GC过程中的cpu使用率，STW时间和STW频率，这三个指标通常是互相关联的，总体上来说，GC的调优就是需要**尽量减少内存分配和释放，尽量复用已经分配的内存**。

下面通过一个例子来分析GC调优的一些方法，这个例子中首先创建了长度为1000的256位随机字符串数组，然后在每一个web请求中随机生成一个字符串，然后再到字符串数组中查找，这个过程是对查找用户缓存的一个简单模拟，具体代码功能可以参考注释：

```go
package main

import (
	"io"
	"math/rand"
	"net/http"

	// 用于启动pprof的web接口
	_ "net/http/pprof"

	"strings"
)

// ids 模拟含有1000个用户id的缓存
var ids = make([]string, 1000)

func main() {
	// 初始化缓存
	for i := 0; i != 1000; i++ {
		s := getRandString(256)
		ids = append(ids, s)
	}

	http.HandleFunc("/slowgc", slowGC)
	http.ListenAndServe(":8080", nil)
}

// slowGC 模拟由于内存分配导致的GC
func slowGC(w http.ResponseWriter, r *http.Request) {
	currentUser := getRandString(256)
	for _, id := range ids {
		// 这里调用了strings.ToLower函数，这个函数会导致字符串的复制
		if strings.Contains(strings.ToLower(currentUser), strings.ToLower(id)) {
		// if strings.Contains(id, currentUser) {
			io.WriteString(w, "hit")
		}
	}
	io.WriteString(w, "not found")
}

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

// getRandString 生成指定位数的字符串
func getRandString(length int) string {
	rnd := make([]byte, length)
	for i := range rnd {
		rnd[i] = letters[rand.Intn(len(letters))]
	}
	s := string(rnd)
	return s
}
```

### 2.1 分析GC问题的瓶颈

要观察GC信息主要有以下三种方式

- 使用**GODEBUG=gctrace=1**参数启动程序观察GC日志
- **go tool pprof**工具可以用来详细分析内存分配，找出程序哪里分配了不合理的内存造成了GC压力
- **go tool trace**工具可以列出整个代码执行过程中的详细信息。

下面我们通过上述的例子，打开`GODEBUG=gctrace=1`来分析一下GC信息。首先我们执行如下命令启动程序：

```shell
go build main.go
GODEBUG=gctrace=1 ./main
```

接着使用`ab`工具对接口进行压力测试：

```shell
# -c 表示同时发送100个请求
# -n 表示请求总量一共10000个
ab -c 100 -n 10000 http://localhost:8080/slowgc
```

`ab`工具可以得到类似的结果：

```shell
ab -c 100 -n 10000 http://localhost:8080/slowgc
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /slowgc
Document Length:        3009 bytes

Concurrency Level:      100
Time taken for tests:   23.753 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      31060000 bytes
HTML transferred:       30090000 bytes
Requests per second:    421.00 [#/sec] (mean)
Time per request:       237.529 [ms] (mean)
Time per request:       2.375 [ms] (mean, across all concurrent requests)
Transfer rate:          1276.98 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.3      0       5
Processing:     6  237 136.6    210    1217
Waiting:        6  230 136.2    204    1217
Total:          6  237 136.6    210    1217

Percentage of the requests served within a certain time (ms)
  50%    210
  66%    263
  75%    301
  80%    331
  90%    412
  95%    489
  98%    598
  99%    704
 100%   1217 (longest request)
```

可以看出上面代码的简单逻辑，每秒钟处理的请求数量只有421个，按道理来说简单的字符串查找不应该这么慢，那么问题出在哪里呢，接着看下刚才过程中的gctrace信息：

```
...
gc 2619 @25.802s 9%: 0.060+0.61+0.019 ms clock, 0.24+0.73/0.49/0+0.079 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 2620 @25.810s 9%: 0.085+0.68+0.022 ms clock, 0.34+0.80/0.57/0+0.088 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
gc 2621 @25.818s 9%: 0.057+0.42+0.019 ms clock, 0.23+0.63/0.30/0+0.076 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
gc 2622 @25.826s 9%: 0.76+1.7+0.019 ms clock, 3.0+1.9/0.40/0+0.078 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
```

观察压测结束后的gc日志，可以看出10000个请求一共触发了2622次GC，其中GC Groutine一共使用了将近百分之十的CPU，在这样一个简单的业务逻辑下，GC触发的频率明显过高，并且占用的CPU时间也过高（关于GC trace信息的阅读可以参考文末附录），这是不合理的，GC的行为不符合预期的时候，我们首先应该从**内存是否合理**分配来入手分析，接下来我们可以使用go tool pprof工具来查看一下上述代码的内存使用情况：

```shell
go tool pprof http://localhost:8080/debug/pprof/allocs
Fetching profile over HTTP from http://localhost:8080/debug/pprof/allocs
Saved profile in /root/pprof/pprof.go-gc-tutorial.alloc_objects.alloc_space.inuse_objects.inuse_space.001.pb.gz
File: go-gc-tutorial
Type: alloc_space
Time: Jan 2, 2021 at 9:39pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top 6 -cum
Showing nodes accounting for 0, 0% of 7.26GB total
Dropped 58 nodes (cum <= 0.04GB)
Showing top 6 nodes out of 8
      flat  flat%   sum%        cum   cum%
         0     0%     0%     7.25GB 99.91%  net/http.(*conn).serve
         0     0%     0%     7.16GB 98.68%  main.slowGC
         0     0%     0%     7.16GB 98.68%  net/http.(*ServeMux).ServeHTTP
         0     0%     0%     7.16GB 98.68%  net/http.HandlerFunc.ServeHTTP
         0     0%     0%     7.16GB 98.68%  net/http.serverHandler.ServeHTTP
         0     0%     0%     7.16GB 98.61%  strings.(*Builder).Grow (inline)
(pprof) 
(pprof) list slowGC
Total: 7.26GB
ROUTINE ======================== main.slowGC in /data/codes/go-gc/main1.go
         0     7.16GB (flat, cum) 98.68% of Total
         .          .     25:	http.ListenAndServe(":8080", nil)
         .          .     26:}
         .          .     27:
         .          .     28:// slowGC 模拟由于内存分配导致的GC
         .          .     29:func slowGC(w http.ResponseWriter, r *http.Request) {
         .     4.50MB     30:	currentUser := getRandString(256)
         .          .     31:	for _, id := range ids {
         .          .     32:		// 这里调用了strings.ToLower函数，这个函数会导致字符串的复制
         .     7.16GB     33:		if strings.Contains(strings.ToLower(currentUser), strings.ToLower(id)) {
         .          .     34:		// if strings.Contains(id, currentUser) {
         .   512.02kB     35:			io.WriteString(w, "hit")
         .          .     36:		}
         .          .     37:	}
         .          .     38:	io.WriteString(w, "not found")
         .          .     39:}
         .          .     40:
```

首先使用命令

```
go tool pprof http://localhost:8080/debug/pprof/allocs
```

进入pprof工具的交互模式，然后输人`top 5 -cum`查看分配内存最多的5个函数，从输出结果我们可以看出，`slowGC`函数分配了7GB左右的内存，这有一些奇怪，然后我们可以执行`list slowGC`命令，可以看出在第33行调用的操作字符串的方法分配内存最多，至此原因大概明白了，golang中string类型是不可变的，每次调用`strings.ToLower()`方法时候都会申请内存分配一个新的字符串，最后就造成的每次循环中都会申请一块新内存，从而造成GC压力（调用ToLower()方法只是为了举例说明GC问题，并无实际的意义）。我们可以针对这个问题做一下简单的修改，将第33行的ToLower()函数移动到循环之外：

```
package main

import (
	"io"
	"math/rand"
	"net/http"

	// 用于启动pprof的web接口
	_ "net/http/pprof"

	"strings"
)

// ids 模拟含有1000个用户id的缓存
var ids = make([]string, 1000)

func main() {
	// 初始化缓存
	for i := 0; i != 1000; i++ {
		s := getRandString(256)
		s = strings.ToLower(s)
		ids = append(ids, s)
	}

	http.HandleFunc("/slowgc", slowGC)
	http.ListenAndServe(":8080", nil)
}

// slowGC 模拟由于内存分配导致的GC
func slowGC(w http.ResponseWriter, r *http.Request) {
	currentUser := getRandString(256)
        // 将ToLower()的调用移动到循环之外
	currentUser = strings.ToLower(currentUser)
	for _, id := range ids {
		if strings.Contains(id, currentUser) {
			io.WriteString(w, "hit")
		}
	}
	io.WriteString(w, "not found")
}

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

// getRandString 生成指定位数的字符串
func getRandString(length int) string {
	rnd := make([]byte, length)
	for i := range rnd {
		rnd[i] = letters[rand.Intn(len(letters))]
	}
	s := string(rnd)
	return s
}
```

再次执行上述压测命令，可以看到压测结果为：

```
ab -c 100 -n 10000 http://localhost:8080/slowgc
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /slowgc
Document Length:        9 bytes

Concurrency Level:      100
Time taken for tests:   0.880 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1250000 bytes
HTML transferred:       90000 bytes
Requests per second:    11369.89 [#/sec] (mean)
Time per request:       8.795 [ms] (mean)
Time per request:       0.088 [ms] (mean, across all concurrent requests)
Transfer rate:          1387.93 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    4   0.8      3      10
Processing:     1    5   1.8      5      19
Waiting:        1    4   1.6      4      17
Total:          4    9   1.8      8      23
WARNING: The median and mean for the initial connection time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      8
  66%      9
  75%      9
  80%     10
  90%     11
  95%     12
  98%     13
  99%     16
 100%     23 (longest request)
```

从ab命令执行结果来看，处理速度有了极大的改进，每秒处理的请求数量达到了11369，而查看gctrace日志，也可以看到整个过程中只触发了10几次GC：

```
...
gc 10 @5.149s 0%: 0.097+0.84+0.007 ms clock, 0.38+1.4/0.001/0+0.028 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 11 @5.208s 0%: 0.037+0.79+0.004 ms clock, 0.14+0.65/0.56/0.25+0.016 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 12 @5.277s 0%: 0.039+0.71+0.003 ms clock, 0.15+0.71/0.55/0+0.014 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 13 @5.346s 0%: 0.023+1.0+0.013 ms clock, 0.094+0.78/0.66/0+0.055 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
```

小结一下，这部分使用了gctrace日志和pprof工具发现了代码中内存分配的缺陷，通过修改代码解决了问题。

### 2.2 调节GOGC参数

上述内存造成的问题也可通过调整GOGC参数来给予临时解决，为什么说是临时解决呢？因为通常修改GOGC参数，虽然减少了GC的触发频率，也可能会带来更长单次GC时间。要从根本上解决GC问题，还是要从代码，或者业务逻辑入手进行优化。下面我们再看下GOGC参数如何影响GC操作。我们用下面的命令设置GOGC的值为500， 意味着堆内存增长5倍，下一次GC才会触发，重新执行2.1中有问题的代码：

```
GOGC=500 GODEBUG=gctrace=1 ./main
```

同样执行ab命令进行压测后，得到如下结果：

```
ab -c 100 -n 10000 http://localhost:8080/slowgc
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8080

Document Path:          /slowgc
Document Length:        3009 bytes

Concurrency Level:      100
Time taken for tests:   18.846 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      31060000 bytes
HTML transferred:       30090000 bytes
Requests per second:    530.62 [#/sec] (mean)
Time per request:       188.459 [ms] (mean)
Time per request:       1.885 [ms] (mean, across all concurrent requests)
Transfer rate:          1609.48 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.3      0       4
Processing:     6  187 136.9    173    1171
Waiting:        5  184 135.5    170    1171
Total:          6  187 136.9    173    1171

Percentage of the requests served within a certain time (ms)
  50%    173
  66%    226
  75%    256
  80%    283
  90%    362
  95%    437
  98%    545
  99%    641
 100%   1171 (longest request)
```

从`ab`命令结果可以看出，每秒的请求较之前的400多次变为了500多次，性能略有提升，而gc日志也可以看出，由于修改了GC触发的频率，实际触发的GC次数为394次，比之前的2000多次也是明显减少，整个GC过程的CPU使用率也维持在1%。通过这个例子可以看出，根据实际情况调整GC触发频率对于GC效率是会有较大提升的，但是具体如何设置GOGC的值就是一个开放问题了，需要在不同的场景下不断的测试，最终得到符合当前场景的参数值。

```
...
gc 391 @21.304s 1%: 0.063+0.58+0.015 ms clock, 0.25+0.81/0.53/0+0.061 ms cpu, 20->20->1 MB, 21 MB goal, 4 P
gc 392 @21.350s 1%: 0.071+0.38+0.017 ms clock, 0.28+0.76/0.32/0+0.069 ms cpu, 20->20->1 MB, 21 MB goal, 4 P
gc 393 @21.395s 1%: 0.059+0.36+0.015 ms clock, 0.23+0.57/0.29/0+0.061 ms cpu, 20->20->0 MB, 21 MB goal, 4 P
gc 394 @21.439s 1%: 0.066+0.41+0.018 ms clock, 0.26+0.50/0.36/0+0.075 ms cpu, 20->20->0 MB, 21 MB goal, 4 P
```

### 2.3 GC调优的其他方法

除了上述的GC调优方法之外，golang编译器提供了逃逸分析的方法，具体而言就是编译器如果可以判断出一个变量创建后只存活在当前栈帧中，那么在当前栈被销毁时候，变量也可以直接销毁，这样就不再需要执行GC去做这部分工作。实际中可以使用`go build -gcflags=-m`来打开逃逸分析的提示。

除了逃逸分析之外，另一种常见的控制内存分配的方法是将常用的内存对象池化，`sync.Pool`就提供了池化对象的支持，除此之外，很多三方库也可以使用，比如[https://github.com/valyala/bytebufferpool](https://github.com/valyala/bytebufferpool)。

## 3 总结

本文首先介绍了Golang中GC的执行过程，然后用一个例子，结合golang提供的GC日志和pprof工具，介绍GC调优的基本方法。

本有还有如下的不足：未对逃逸分析和池化对象给出具体的示例。

## 4 参考

[1]Garbage Collection In Go : Part II - GC Traces， [https://www.ardanlabs.com/blog/2019/05/garbage-collection-in-go-part2-gctraces.html](https://www.ardanlabs.com/blog/2019/05/garbage-collection-in-go-part2-gctraces.html)

[2]Getting to Go: The Journey of Go's Garbage Collector, [https://blog.golang.org/ismmkeynote](https://blog.golang.org/ismmkeynote)

[3] Golang GC核心要点和度量方法, [https://w](https://wudaijun.com/2020/01/go-gc-keypoint-and-monitor/)[u](https://wudaijun.com/2020/01/go-gc-keypoint-and-monitor/)[daijun.com/2020/01/go-gc-keypoint-and-monitor/](https://wudaijun.com/2020/01/go-gc-keypoint-and-monitor/)

[4] Go memory ballast: How I learnt to stop worrying and love the heap, [https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/](https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/)

## 5 附录

### 5.1 对GODEBUG=gctrace=1信息的解读

```
gctrace的日志格式为：
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # P
其中各个参数的含义如下:
	gc #        程序启动以来的GC次数
	@#s         程序启动了多久
	#%          GC过程各个阶段的wall time，分别是sweep termination阶段/mark阶段/mark termination阶段
	#+...+#     GC消耗的CPU时间，分别是gc assist时间/后台gc时间/idle gc时间
	#->#-># MB  GC开始/结束/最后存活的内存
	# MB goal   下次触发GC的堆内存大小
	# P         一共使用了几个P
```
