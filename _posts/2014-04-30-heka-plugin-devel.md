---
layout: post
title: Heka插件开发
category: system
tagline: "Supporting tagline"
tags: [Heka, Go]
---
{% include JB/setup %}


Heka是一个实时数据收集、处理和分析的工具，具备高可扩展的插件开发能力。本文是自己调研Heka插件开发的一个总结，方便快速入门插件开发。

2014 gophercon上关于Heka的演讲Slide：[https://cdn.rawgit.com/gophercon/2014-talks/master/rob_miller_heka/index.html#/](https://cdn.rawgit.com/gophercon/2014-talks/master/rob_miller_heka/index.html#/)，从这个Slide中借用一张Heka内部架构图：

<div align="center">
<img src="/assets/images/heka-overview-diagram.png" height="300" width="600">
</div>

内部架构图清晰的反应出了数据从进入Heka到流出Heka的整个过程需要经历一些什么样的组件。图中的箭头符号反应出了，一个数据进入Heka后可以选择什么样的路径，路径并不是唯一的，一切都可以根据需求来设置。

内部架构图中展示的所有组件，我们可以通过开发插件定制的部分分别是：Inputs、Decoders、Filters和Outputs。

