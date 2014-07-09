---
layout: post
title: 深入讨论channel timeout
category: go
tagline: "Supporting tagline"
tags : [Go, channel, timeout]
---
{% include JB/setup %}

Go 语言的 channel 本身是不支持 timeout 的，所以一般实现 channel 的读写超时都采用 select，如下：

	select {
	case <-c:
	case <-time.After(time.Second):
	}
	
这两天在写码的过程中突然对这样实现 channel 超时产生了怀疑，这种方式真的好吗？于是我写了这样一个测试程序：

	package main

	import (
	    "os"
	    "time"
	)
	
	func main() {
	    c := make(chan int, 100)
	
	    go func() {
	        for i := 0; i < 10; i++ {
	            c <- 1
	            time.Sleep(time.Second)
	        }
	
	        os.Exit(0)
	    }()
	
	    for {
	        select {
	        case n := <-c:
	            println(n)
	        case <-timeAfter(time.Second * 2):
	        }
	    }
	}
	
	func timeAfter(d time.Duration) chan int {
	    q := make(chan int, 1)
	
	    time.AfterFunc(d, func() {
	        q <- 1
	        println("run") 		// 重点在这里
	    })
	
	    return q
	}

这个程序很简单，你会发现运行结果将会输出 10 次 “run”，也就是每一遍执行 select 注册的 timer 最终都执行了，虽然这里读 channel 都没有超时。原因其实很简单，每次执行 select 语句，都会将 case 条件语句给执行一遍，于是 timeAfter 的执行结果就是会创建一个定时器，并注册到 runtime 中，select 语句执行完成后，这个定时器本身并没有撤销，还继续保留在 runtime 的小顶堆中，所以这些 timer 一超时就会执行挂载的函数。

当然，用 `time.After()` 函数来做 channel 的读写超时，在应用层根本感受不到底层的定时器还保留着、继续执行；问题是，如果这里的 select 语句在循环中执行得非常快，也就是 channel 中的消息来得非常频繁，会出现的问题就是 runtime 中会有大量的定时器存在，timeout 的时间设置得越长，底层维护的定时器就会越多。原因就是每次 select 都会注册一个新的 timer，并且 timer 只有在它超时后才会被删除。

想想，自己的 channel 每秒钟将传输成千上万的消息，将会有多少 timer 对象存在底层 runtime 中。大量的临时对象会不会影响内存？大量的 timer 会不会影响其他定时器的准确度？

最后，我觉得正确的 channel timeout 也许应该这么做：

	to := time.NewTimer(time.Second)
	for {
	    to.Reset(time.Second)
	    select {
	    case <-c:
	    case <-to.C:
	    }
	}
	
这样做就是为了维护一个全局单一的定时器，每次操作前调整一下定时器的超时时间，从而避免每次循环都生成新的定时器对象。

简单测试了一下两种 channel 超时实现方式，在全力收发数据的情况的内存对象和 gc 情况。
<br>
<img src="/assets/images/chan-to-objects.png" height="350" width="500">
<br>
* 蓝线是采用 `time.After()`，并设置4s 超时的堆内存对象分配的数量
* 绿线是采用 `time.After()`，并设置2s 超时的堆内存对象分配的数量
* 黄线是采用全局 timer，并设置4s 超时的堆内存对象分配的数量

这个现象其实是预料之中的，重点可以注意设置的超时时间越长，`time.After()` 的表现将越糟糕。

<br>
<img src="/assets/images/chan-to-gc.png" height="350" width="500">
<br>

这三条线和上图的三条线描述的对象是一样的，图中的 gc 时间是平均每次 gc 的时间。

针对这个 channel timeout，我没有去测试是否会影响其他定时器的准确性，但我认为这是必然的，随着定时器的增多。


最后，我始终觉得 channel 本身应该支持超时机制，而不是利用 select 来实现。

----

探索任何一个现象背后的真正原因，才是最有趣的事情。

