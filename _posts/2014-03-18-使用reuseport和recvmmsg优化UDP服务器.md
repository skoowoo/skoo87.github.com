---
layout: post
title: 使用reuseport和recvmmsg优化UDP服务器
category: system
tagline: "Supporting tagline"
tags : [udp, reuseport, dns, recvmmsg]
---
{% include JB/setup %}


最近刚好完成了一个DNS服务器的开发，因此积累一点对高性能UDP服务器的开发经验。如果你也遇到UDP服务器的性能不佳，远不如你的预期，也许你也可以采用本文的手段去优化一下试试。

udp不像tcp是有连接的，因此udp不能通过建立多个连接来提高对服务器的并发访问，然后我就遇到了在多核环境下通过多线程访问一个共享的udp socket时，无论如何我都无法将所有的cpu都利用起来，最后的结局当然就是无法压测出机器的瓶颈，性能也上不去。Google为了解决他们的DNS服务器性能问题，就给linux内核打了一个patch，这个patch就是SO_REUSEPORT，经过我的实战体验，reuseport对udp服务器在多核机器上的性能提升是非常大的，值得使用。

[REUSEPORT](https://lwn.net/Articles/542629/) 的目的如其名，就是为了让多线程/多进程服务器的每个线程都listen同一个端口，并且最终每个线程拥有一个独立的socket，而不是所有线程都访问一个socket。没有reuseport这个patch的话，这么做的后果就是服务器会报出一个类似“地址/端口被占用的”错误信息。在没有reuseport的时候，客户端发给udp服务器的每个包都是被投递到唯一的一个socket上了，使用reuseport后，服务器有了多个socket，那么客户端发过来的包投递到哪个socket上呢？linux内核采用了一个四元组<客户端ip，客户端port，服务器ip，服务器port>的hash来进行包的分发，这样做至少有两个目的：一是保证同一个客户度过来的包都被递送到同一个socket上；二是在客户端量足够的时候，基本可以均衡到所有的socket上。在使用reuseport的时候需要注意：客户端太少的话，是很难压测出服务器的真实性能的，因为reuseport使用的是hash值来分发请求到socket上，所以可能出现每个socket上接收包不均衡的情况，使用较多的客户端机器来压测服务器，目的就是让每个socket尽可能的均衡。


使用reuseport后，udp服务器的并发能力大幅度的提高了，这个时候还可以继续使用[recvmmsg](http://man7.org/linux/man-pages/man2/recvmmsg.2.html)来继续降低系统调用的开销。recvmmsg是一个批量接口，它可以从socket里一次读出多个udp数据包，不像recvfrom那样一次只能读一个。如果客户端多、请求量大的话，recvmmsg的批量读就很有优势了。不过，使用recvmmsg一定要清楚，它从socket里一次读出的所有包不一定是来自同一个客户端的，大多数情况应该都是来自不同客户端的。这不像tcp，从同一个连接里读到的数据一定是同一个客户端。我们的一个同学在使用recvmmsg的时候，就犯了这个错误，误认为一次收取的数据包都是同一个客户端的，最后将所有的应答都发给了同一个客户端，其他的客户端全都超时了。高性能服务器开发中，系统调用是昂贵的，所以没事就可以用strace看看一个请求周期内有哪些系统调用，尽一切可能去优化掉他们。

开发一个UDP服务器，不是说使用了reuseport和recvmmsg后性能就高了。一个高性能的网络服务器，是需要进行方方面面的优化才行的。
