---
layout: post
title: Go Runtime hashmap实现
category: Go
tagline: "Supporting tagline"
tags : [Go, runtime, hashmap]
---
{% include JB/setup %}


前两天有小伙伴问道是否看过 Go 语言 map 的实现，当时还真没看过，于是就花了一点时间看了一遍 runtime 源码中的 hashmap 实现。map 的底层实现就是一个 hash 表，大体结构上和平时在脑海里的 hash 表差不多，但确实有很多细节(“Devils in the details”)。

<div>
<img src="/assets/images/hashmap.png" height="350" width="500">
</div>

hashmap 通过一个 bucket 数组实现，所有元素将被 hash 到数组中的 bucket 中，bucket 填满后，将通过一个 overflow 指针来扩展一个 bucket 出来形成链表，也就是解决冲突问题。这也就是一个基本的 hash 表结构，没什么新奇的东西，下面总结一些细节吧。

1. 注意一个 bucket 并不是只能存储一个 key/value 对，而是可以存储8个 key/value 对。每个 bucket 由 header 和 data 两部分组成，data 部分的内存大小为：(sizeof(key) + sizeof(value)) * 8，也就是要存储8对 key/value，这8对 key/value 在 data 内存中的存储顺序是：key0key1...key7value0value1...value7，是按照顺序先依次存储8个 key 值，然后存储对应的8个 value。 为什么不是存储为 key0value0...key7value7 呢？主要是方便访问吧。
2. 如果 key, value 的类型大小超过了128字节，将不会直接存储值，而是存储其指针。 
3. bucket 的 header 部分有一个 `uint8 tophash[8]` 数组，这个数组将用来存储8个 key 的 hash 值的高8位值。比如：tophash\[0\] 存储的值就是 hash(key0) >> (64 - 8)。保存了一个 key 的 hash 高8位部分，在查找/删除/插入一个 key 的时候，可以先判断两个 key hash 的高8位是否相等，如果不等，那就根本不用去比较 key 的内容。所以这里保存一下 hash 值的高8位可以作为第一步的粗略过滤，不少时候可以省掉比较两个 key 的内容，因为比较两个 key 是否相等的代价远比两个 uint8 的代价高。当然，这里如果存储整个 hash 值，而不仅仅是高8位的话，判断效果将更好，但内存的占用就会多很多了。
4. bucket 的8个 key/value 空间如果都填满后，就会分配新的 bucket，通过 overflow 指针串联起来。注意这个链表指针被命名为 overflow，代表的正是 bucket 溢出了，这个命名感觉很好，hash 表实现的时候我们应该努力避免 bucket overflow。
5. hashmap 是会自增长的，也就说随着插入的 kv 对越来越多，初始的 bucket 数组就可以需要增长、重新hash 所有元素，性能才会好。bucket 数组增长的时机就是插入的元素个数大于了 `bucket数组大小 * 6.5`，为什么是6.5，这个在代码注释里有说明，主要是测试出来的经验值。
6. hashmap 每次增长，都是重新分配一个新的 bucket 数组，新 bucket 数组是之前 bucket 数组的2倍大小。
7. hashmap 增长后，需要将老 bucket 数组中的元素拷贝到新的 bucket 数组，这个拷贝过程不是一口气立马完成的，而是采用了增量式的拷贝，也就是说分配了新的 bucket 数组后，并没有立刻拷贝元素，而是等接下来每次插入一个元素的时候，才拷贝一点，随着插入的动作增多，逐渐就将全部元素拷贝到了新的 bucket 数组中。
8. 在 make 一个 map 对象的时候，如果不指定大小的话，bucket 数组默认就是1了，随着插入的元素增多，就会增长成2，4，8，16等。可以看出不指定初始化大小的map，很可能要经历很多次的增长、元素拷贝。我们应该给 map 指定一个合适的大小值。


暂时就总结这么一点了。。。


----
浙江省图空调开得真冷啊，冷死我了，我要出去晒会太阳了。
