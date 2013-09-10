---
layout: post
title: Go RPC Benchmark
category: Go
tagline: "Supporting tagline"
tags : [Go, rpc]
---
{% include JB/setup %}


这篇完成得太折腾了，为了更好的展示benchmark的一系列结果数据，我必须得找个软件将数据进行图表化。以前在windows上基本都用execel画曲线图、柱状图等，但在linux/mac上却找不到顺手的工具了。我也使用过gnuplot，这货不知道是太专业，还是太古老的原因，始终用得不顺手、不开心。于是，我就决定自己先用Go和chart.js库折腾了一个**[goplot](https://github.com/skoo87/goplot)**工具来绘制图表，然后再才开始写这篇博客。

有人可能会说我又在折腾轮子了，确实是折腾了一个轮子。话说，这又怎么样呢？作为一个程序员，最大的优势就是自己用得不开心的工具，可以自己动手完善、甚至写一个新的。我认为一个geek程序员首先就是要学会不断的装备自己的工具库。不扯废话了，回归正题。

####测试维度
这次benchmark主要以下两个维度进行测试

1. 单个连接上，不同并发数的QPS
2. 固定100个并发，不同连接数的QPS

测试过web server的人可能对这里的并发有一点疑惑。web服务器的并发数一般是指连接数，这里的并发数不是指的连接数，而是同时在向服务器发送请求的goroutine数(理解为线程数)。因为rpc的网络连接是支持并发请求的，也就是一个连接上是可以同时有很多的请求在跑，这不同于http协议一问一答的模式，而是和SPDY协议非常类似。

`维度1` 关注的是一个连接上能跑的请求的极限究竟是多少，这个可以用来粗略的衡量与后端的连接数。毕竟后端系统(如:DB)并不希望存在大量的连接，这和web服务器有一点不同。这个维度更多的也是在衡量rpc的客户端究竟能够发送得多快。

`维度2` 关注的基本就是rpc服务器的性能了，也就是看一个goroutine一个连接后，服务器还能应付多大的并发。

####测试环境

* Client和Server分别部署在两台机器上，当然两台机器都在同一个局域网内，可以忽略网络。
* Client和Server的机器都是8核cpu超线程到16核，内存和网卡都远远足够。
* Client每个请求只发送一个100bytes的字符串给服务器。
* GOMAXPROCS设置为16

####测试结果

**单连接上，不同并发数的QPS**

<div align="center">
<img src="/assets/images/rpc-benchmark-one-conn.png" height="350" width="500">
</div>

可以看到随着这个连接上的并发数增大，qps最后稳定在5.5W左右。这个性能有点偏低了。这一块的性能影响主要来自锁的开销、序列化/反序列化、以及goroutine的调度。一个rpc客户端（也就是和服务器一个连接）有一个input goroutine来负责读取服务器的响应，然后分发给每个写入请求的goroutine，如果这个goroutine长时间不能被调度到，那么就会导致发送者的阻塞等待，Go目前的调度明显是会有这个问题的，留到调度器再聊。

**100并发，不同连接数的QPS**

<div align="center">
<img src="/assets/images/rpc-benchmark-multi-conn.png" height="350" width="500">
</div>

多个rpc客户端可以到随着连接数的增多，qps最后可以达到20W左右，这个时候瓶颈主要出现在cpu上了。很久以前，我用同一批机器测试avro的时候100个线程100个连接，qps在14+W。这个维度来看Go的rpc性能非常不错。

性能应该是一个需要我们持续关注和优化的问题。