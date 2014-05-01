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

####编译源码
1、从github上克隆出Heka源码库

	git clone https://github.com/mozilla-services/heka
	

2、查看Heka已经release的版本，其实就是打的tag

	git tag
		
我看到的最新release版本是v0.5.1，因此我们选择这个最新版本的代码：
		
	git checkout v0.5.1
				
3、现在可以编译当前最新版本v0.5.1代码了(windows平台忽略，暂时只关心Linux平台)

	sh build.sh
		
build.sh脚本是Heka的编译工具，整个工程是通过cmake来管理的。第一次build过程可能比较慢，因为还会下载一些其他的依赖库和工具，不过不需要人为干预，坐等build完成即可。build完成后，当前源码目录下多出一个build目录：
<div>
<img src="/assets/images/heka-build.png" height="280" width="450">
</div>
	
