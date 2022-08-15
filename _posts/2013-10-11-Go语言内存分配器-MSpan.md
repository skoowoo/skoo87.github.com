---
layout: post
title: Go语言内存分配器-MSpan
category: Go
tagline: "Supporting tagline"
tags : [Go, runtime]
---
{% include JB/setup %}


MSpan和FixAlloc一样，都是内存分配器的基础工具组件，但和FixAlloc没太大的交集，各自发挥功效而已。span(MSpan简称span)是用来管理一组组page对象，先解释一下page，page就是一个4k大小的内存块而已。span就是将这一个个连续的page给管理起来，注意是连续的page，不是东一个西一个的乱摆设的page。为了直观形象的感受一下span，还是得画个图吧，图形是最好的交流语言。

<div align="center">
<img src="/assets/images/span.png" height="350" width="500">
</div>

MSpan结构定义在malloc.h头文件中，代码如下：

	struct MSpan
	{
		MSpan	*next;		// in a span linked list
		MSpan	*prev;		// in a span linked list
		PageID	start;		// starting page number
		uintptr	npages;		// number of pages in span
		MLink	*freelist;	// list of free objects
		uint32	ref;		// number of allocated objects in this span
		int32	sizeclass;	// size class
		uintptr	elemsize;	// computed from sizeclass or from npages
		uint32	state;		// MSpanInUse etc
		int64   unusedsince;	// First time spotted by GC in MSpanFree state
		uintptr npreleased;	// number of pages released to the OS
		byte	*limit;		// end of data in span
		MTypes	types;		// types of allocated objects in this span
	};

span结构比较重要的字段，都出现在上面的结构图中了，当然并不是说其他的字段就不重要了。span的结构中有pre/next两个指针，一看就能猜到是用来构造双向链表的。没错，在实际的使用中，span确实是出现在双向循环链表中。span可能会用在分配小对象（小于等于32k）的过程中，也可能会用于分配大对象(大于32k)，在分配不同类型对象的时候，span管理的元数据也大不相同。

`npages`表示是此span存储的page的个数（比如：上图中就画了3个page），`start`可以看作一个page指针，指向第一个page，有了第一个page当然就可以算出后面的任何一个page的起始地址了，因为span管理的始终是连续的一组page。这里需要注意start的类型是PageID，由此可以看出这个start保存的并不是第一个page的起始地址，而是第一个page的id值。这个id值是如何算出来的呢？其实给每个page算一个id，是非常简单的事情，只要将这个page的的地址除以4096取整(伪代码：`page_addr>>20`)即可，当然前提是已经保证好了每个page按4k对齐。是不是觉得很精妙，这样一来每个page都有一个整数id了，并且任何一个内存地址都可以通过移位算出这个地址属于哪个page，这个很重要。

start是span最重要的一个字段，它维护好了所有的page。`sizeclass`如果是0的话，就代表这个span是用来分配大对象的，其他值当然都是分配小对象了。在分配小对象的时候，start字段维护的所有page，最后将会被切分成一个一个的连续内存块，内存块大小当然就是小对象的大小，这些切分出来的内存块将被链接成为一个链表挂在freelist字段上。分配大对象的时候，freelist就没什么用了。

span干的活，也就这么一点，反正就是管理一组连续的page。内存分配器中的每个page都会属于一个span，page永远不会独立存在。span相关的API有：

	// 初始化一个span结构，将分配的page放入到这个span中。
	void	runtime·MSpan_Init(MSpan *span, PageID start, uintptr npages);

	// 下面这些都是操作span构成的双向链表了。
	void	runtime·MSpanList_Init(MSpan *list);
	bool	runtime·MSpanList_IsEmpty(MSpan *list);
	void	runtime·MSpanList_Insert(MSpan *list, MSpan *span);
	void	runtime·MSpanList_Remove(MSpan *span);
	
	
span就先写到这里，接下来写mcache, mcentral, mheap等核心组件的时候还会涉及到如何使用span的。

注：本文基于Go1.1.2版本代码

<br>
***

*写到这里，我感觉自己并没有将span写清楚了，写源码分析类技术文章真心比读源码有挑战，我试图在用最小的代码展示来将原理述说清楚。发现自己的功底还是不够啊。*
