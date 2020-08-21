![封面](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F/v2-be7b47c096479050fb05d7c6acf9dd4d_1440w.jpg)









#### 什么是pprof

pprof是Go的性能分析工具，在程序运行过程中，可以记录程序的运行信息，可以是CPU使用情况、内存使用情况、goroutine运行情况等，当需要性能调优或者定位Bug时候，这些记录的信息是相当重要。



#### 代码实现

```go
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof" // 引入pprof,调用init方法
)

func main() {

	// 生产环境应仅在本地监听pprof
	go func() {
		ip := "127.0.0.1:9527"
		if err := http.ListenAndServe(ip, nil); err != nil {
			fmt.Println("开启pprof失败", ip, err)
		}
	}()

	// 业务代码运行中
	http.ListenAndServe("0.0.0.0:8081", nil)
}
```



#### 使用方式

**1.浏览器方式**

地址：http://127.0.0.1:9527/debug/pprof/

界面：

![image-20200718135542144](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/image-20200718135542144.png)



allocs: 内存分配分析

block:同步阻塞分析

cmdline:命令行调用分析

goroutine:goroutine分析

heap:堆内存分析

mutex:锁竞争分析

profile:30s的CPU使用情况分析

threadcreate: 创建新OS线程的堆栈跟踪

trace:当前程序执行的追溯（比如一个get请求的追溯）



**2.命令行方式（适用于服务端调试）**

```shell
# 下载cpu profile，默认从当前开始收集30s的cpu使用情况，需要等待30s
go tool pprof http://localhost:9527/debug/pprof/profile   # 30-second CPU profile
go tool pprof http://localhost:9527/debug/pprof/profile?seconds=120     # wait 120s

# 下载heap profile
go tool pprof http://localhost:9527/debug/pprof/heap      # heap profile

# 下载goroutine profile
go tool pprof http://localhost:9527/debug/pprof/goroutine # goroutine profile

# 下载block profile
go tool pprof http://localhost:9527/debug/pprof/block     # goroutine blocking profile

# 下载mutex profile
go tool pprof http://localhost:9527/debug/pprof/mutex

# 略
```



#### 什么是内存泄漏

内存泄露指的是程序运行过程中已不再使用的内存，没有被释放掉，导致这些内存无法被使用，直到程序结束这些内存才被释放的问题。

在golang中内存泄漏一般来源于

1.goroutine泄漏

2.堆内存泄漏

内存泄漏的内存使用量图一般是这样的：

![image-20190512111200988](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/1460000019222665.png)





ps: 如果没有云平台的这种内存监控工具的话就可以自己用pidstat命令crontab一下定时获取进程占用的物理内存做分析。



#### 错误的分析方式

大家第一反应肯定是根据调用路径图，火焰图等等进行追溯定位内存泄漏，这样其实**又麻烦又不精确**（代码多的时候看起来就是一大坨）



在Dave的high-performance-go-workshop中是如下说的

> https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#types_of_profiles

![image-20200718154808083](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/image-20200718154808083.png)



1. 内存分析只是对堆内存的分析，不包括栈内存（因为栈内存被认识是廉价的，不需要记录，回收起来很方便）
2. 内存分析是抽样分析，每1000次分配抽样一次这种，不精确



所以堆内存分析只能发现问题，不容易找出问题的来源。



#### 内存泄漏如何找到问题

错误案例：

```go
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof" // 引入pprof,仅使用init方法
	"time"
)

func main() {

	// 生产环境应仅在本地监听pprof
	go func() {
		ip := "127.0.0.1:9527"
		if err := http.ListenAndServe(ip, nil); err != nil {
			fmt.Println("开启pprof失败", ip, err)
		}
	}()

	// 业务代码运行
	outCh := make(chan int)
	// 死代码，永不读取
	go func() {
		if false {
			<-outCh
		}
		select {}
	}()

	// 每秒起10个goroutine，goroutine会阻塞，不释放内存
	tick := time.Tick(time.Second / 10)
	i := 0
	for range tick {
		i++
		fmt.Println(i)
		alloc1(outCh) // 不停的有goruntine因为outCh堵塞，无法释放
	}
}

// 一个外层函数
func alloc1(outCh chan<- int) {
	go alloc2(outCh)
}

// 一个内层函数
func alloc2(outCh chan<- int) {
	func() {
		defer fmt.Println("alloc-fm exit")
		// 分配内存，假用一下
		buf := make([]byte, 1024*1024*10)
		_ = len(buf)
		fmt.Println("alloc done")

		outCh <- 0 // 54行
	}()
}

```



直接上结论(打两个点+top+traces一套带走)：

1.内存泄漏大概率是goroutine泄漏，且堆内存分析不大靠谱，所以先上goroutine

2.取2个时间点的goruntine

```shell
go tool pprof http://localhost:9527/debug/pprof/goroutine
# 等一会
go tool pprof http://localhost:9527/debug/pprof/goroutine

# 生成文件：
# pprof.goroutine.001.pb.gz 和 pprof.goroutine.002.pb.gz
```

3.对比两个文件

```shell
go tool pprof -base pprof.goroutine.001.pb.gz pprof.goroutine.002.pb.gz
```

4.结合top命令发现问题（top命令能给出***所选分析内容差异最大的***10条内容-此处所选的分析内容为goroutine）

![image-20200718163206450](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/image-20200718163206450.png)

可以看到有121个goroutine处于挂起（runtime.gopark）状态，即goroutine泄漏

5.定位问题trace命令,可以查看栈调用信息，就能很快的找到问题在于main包中alloc2方法的匿名函数出现了channel send堵塞。

![image-20200718163738809](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/image-20200718163738809.png)







PS:同理，堆内存以及其他性能指标都可以用这个方法来查找差异，唯一的区别就在于打点的时候取的指标不同。

```shell
# 堆内存对比分析的话就打点堆内存
go tool pprof http://localhost:9527/debug/pprof/heap

# 其他同理
```





当然如果所在的机器上还有源代码的话，可以使用list命令更具体到究竟是哪一行代码的问题

![image-20200718164527685](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/go内存泄漏/image-20200718164527685.png)

list一下即可找到是54行 outCh <- 0 这一行发生了122个goroutine的阻塞

