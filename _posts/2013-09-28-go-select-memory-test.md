---
layout: post
title: select的内存分配情况
category: Go
tagline: "Supporting tagline"
tags : [Go, select]
---
{% include JB/setup %}


最近看了Go runtime中关于select的实现（[select in Go's runtime](http://www.bigendian123.com/go/2013/09/26/go-runtime-select/)），发现select语句位于for循环之内执行的时候，每一遍循环都需要在底层runtime中经历malloc对象到free对象的过程，我认为这个频繁的内存分配和释放的代价并不小，至少内存不是处于一种稳定的状态。因此，我实际的测试一把**使用select来操作channel**和**不使用select操作channel**两种情况下的内存情况。

测试过程都是运行程序3分钟，每一次循环sleep 1秒钟，每10秒钟采集一下内存使用情况的数据。为了更直观的感受，我使用[goplot](https://github.com/skoo87/goplot)工具把采集到的内存数据绘制成了图表。

**使用select**

测试代码：[https://gist.github.com/skoo87/6727151#file-test_channel_select-go](https://gist.github.com/skoo87/6727151#file-test_channel_select-go)

<img src="/assets/images/test_select.png" height="350" width="500">



**不使用select**

测试代码：[https://gist.github.com/skoo87/6727151#file-test_channel-go](https://gist.github.com/skoo87/6727151#file-test_channel-go)

<img src="/assets/images/test_no_select.png" height="350" width="500">


上面两图中，最上面的<font color="#3498DB">蓝色</font>线表示从系统分配的`总的堆内存`大小，中间的<font color="#F39C12">黄色</font>线表示`空闲的堆内存`大小，最下面的<font color="#D35400">红色</font>线表示`使用的堆内容`大小。

两种情况，总分配的堆内存都是一样的，基本没差距，两个测试程序本来就很简单，基本一致。使用的堆内存和空闲的堆内存必然具有此多彼少的相关性。这两个数据来看，两种情况有着明显的不同，使用select的时候，由于不断在分配新的内存，所以堆内存的使用一路走高，相应空闲的内存就逐渐变少了。但是不使用select，而是直接操作channel的时候，可以明显感受到内存分配情况是非常稳定的。

这个测试并不是为了证明select有多么的糟糕，而只说明select将会频繁的分配内存，这里只是简单的使用了两个select，内存的表现还是非常明显的。但我个人并不支持为了不用select，而写出一堆难看的代码，甚至是存在隐患的代码，比如无channel超时。但我们应该避免程序中大量执行select语句。

最后，用数据，将数据可视化，更能形象直观的展示现象，玩得比较开心。