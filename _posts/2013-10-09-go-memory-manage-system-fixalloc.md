---
layout: post
title: Go语言内存分配器-FixAlloc
category: Go
tagline: "Supporting tagline"
tags : [Go, runtime]
---
{% include JB/setup %}


昨天写了一篇[Go语言内存分配器设计](http://www.bigendian123.com/go/2013/10/08/go-memory-manage-system-design/)，记录了一下内存分配器的大体结构。在介绍内存分配器的核心实现前，本文先介绍一下内存分配器中一个工具组件——FixAlloc。FixAlloc称不上是核心组件，只是辅助实现整个内存分配器核心的一个基础工具罢了，由此可以看出FixAlloc还是一个比较重要的组件。引入FixAlloc的目的只是用来分配`MCache`和`MSpan`两个特定的对象，所以内存分配器中有`spanalloc`和`cachealloc`两个组件（见《Go语言内存分配器设计》的图）。MCache和MSpan两个结构在malloc.h中有定义。

定义在malloc.h文件中的FixAlloc结构如下，比较关键的三个字段是alloc、list和chunk，其他的字段主要都是用来统计一些状态数据的，比如分配了多少内存之类。

	struct FixAlloc
	{
		uintptr size;
		void *(*alloc)(uintptr);
		void (*first)(void *arg, byte *p);	// called first time p is returned
		void *arg;
		MLink *list;
		byte *chunk;
		uint32 nchunk;
		uintptr inuse;	// in-use bytes now
		uintptr sys;	// bytes obtained from system
	};


<div align="center">
<img src="/assets/images/fixalloc.jpg" height="350" width="500">
</div>

FixAlloc的内存结构图，一看就很简单，简单到没有出现本文的必要了。

`list`指针上挂的一个链表，这个链表的每个节点是一个固定大小的内存块，cachealloc中的list存储的内存块大小为`sizeof(MCache)`，而spanalloc中的list存储的内存块大小为`sizeof(MSpan)`。`chunk`指针始终挂载的是一个128k大的内存块。

FixAlloc提供了三个API，分别是runtime·FixAlloc_Init、runtime·FixAlloc_Alloc和runtime·FixAlloc_Free。

分配一个mcache和mspan的伪代码：

	MCache *mcache;
	mcache = (MCache *) runtime·FixAlloc_Alloc(cachealloc);
	
	MSpan *mspan;
	mspan = (MSpan *) runtime·FixAlloc_Alloc(spanalloc);
	
	
这段伪代码展示的是分配一个MCache和MSpan对象，内存分配器并不是直接使用malloc类函数向系统申请，而是走了FixAlloc。使用FixAlloc分配MCache和MSpan对象的时候，首先是查找FixAlloc的list链表，如果list不为空，就直接拿一个内存块返回使用; 如果list为空，就把焦点转移到chunk上去，如果128k的chunk内存中有足够的空间，就切割一块内存出来返回使用，如果chunk内存没有剩余内存的话，就从操作系统再申请128k内存替代老的chunk。FixAlloc的固定对象分配逻辑就这么简单，相反释放逻辑更简单了，释放的对象就是直接放到list中，并不会返回给操作系统。当然mcache的个数基本是稳定的，也就是底层线程个数，但span对象就不一定那么稳定了，所以FixAlloc的内存可能增长的因素就是span的对象太多。

FixAlloc的实现位于mfixalloc.c文件中，代码目前还不到100行，实在是太简单了。本来是计划本文一起介绍完FixAlloc和MSpan两个基础组件，今天身体不舒服，小感冒了，没精力再写MSpan了。

注：本文基于Go1.1.2版本。


<br>
***
*下班躺在床上，用小米盒子一边看着无聊的电视剧，一边抱着电脑写技术文章，貌似也可以让自己处于一种非常轻松的状态，前提可能是自己对需要写的东西了如指掌，不再需要去翻代码吧。*
	
