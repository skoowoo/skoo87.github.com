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

<br>
####编译源码
1. 从github上克隆出Heka源码库

		git clone https://github.com/mozilla-services/heka
	

2. 查看Heka已经release的版本，其实就是打的tag

		git tag
		
	我看到的最新release版本是v0.5.1，因此我们选择这个最新版本的代码：
		
		git checkout v0.5.1
				
3. 现在可以编译当前最新版本v0.5.1代码了(windows平台忽略，暂时只关心Linux平台)

		sh build.sh
		
	build.sh脚本是Heka的编译工具，整个工程是通过cmake来管理的。第一次build过程可能比较慢，因为还会下载一些其他的依赖库和工具，不过不需要人为干预，坐等build完成即可。build完成后，当前源码目录下多出一个build目录：
	<div>
	<img src="/assets/images/heka-build.png" height="280" width="450">
	</div>
	
	build目录中的heka目录是我们需要关注的。这里放置了所有编译结果，包括heka可执行的二进制文件等。
	<div>
	<img src="/assets/images/heka-build-bin.png" height="200" width="500">
	</div>
	**hekad**文件就是我们最关心的二进制执行文件。只需要这个二进制加上配置文件就可以运行整个Heka软件。

<br>	
####配置Heka实例

为了能够直观的感受Heka，我们配置一个简单的实例，让它监控本机上的nginx access日志目录，实时的读取增量日志，并做条数统计，然后将结果打印到屏幕。

Nginx Access日志的目录路径是：

	/Users/marckywu/projects/logserver2/tmp/logs
	
Nginx Access日志文件名(不做rotation)是：

	access.log

Nginx Access日志format是：

	'$remote_addr - [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"'
	
基于这三个必要的信息，Heka配置文件如下：

	[hekad]
	base_dir = "/tmp/hekad/cache"
	share_dir = "/Users/marckywu/github/heka/build/heka"
	
	
	[LogstreamerInput]
	log_directory = "/Users/marckywu/projects/logserver2/tmp/logs"
	file_match = 'access\.log'
	decoder = "FxaNginxAccessDecoder"
	
	
	[FxaNginxAccessDecoder]
	type = "SandboxDecoder"
	script_type = "lua"
	filename = "/Users/marckywu/github/heka/sandbox/lua/decoders/nginx_access.lua"
	module_directory = "modules"
	    [FxaNginxAccessDecoder.config]
	    log_format = '$remote_addr - [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"'
	    type = "nginx"
	
	
	[CounterFilter]
	message_matcher = "Type == 'nginx'"
	
	[print]
	type = "LogOutput"
	message_matcher = "Type == 'heka.counter-output'"

将配置文件保存为hekad.toml，注意heka的配置文件是toml语法。我们启动hekad进程：

	bin/hekad -config hekad.toml
	
这个时候，Heka已经开始监控nginx access日志了，只要有日志数据，就会读取并处理。我们用ab发送100个请求给nginx，产生100条日志看看Heka的打印效果：

	2014/04/30 13:54:36 <
		Timestamp: 2014-04-30 13:54:36.136374705 +0800 CST
		Type: heka.counter-output
		Hostname: WudeMacBook-Pro.local
		Pid: 56972
		UUID: 0ed1f5c9-5691-45f9-9c4d-a044b70f17ad
		Logger:
		Payload: Got 100 messages. 20.00 msg/sec
		EnvVersion:
		Severity: 7
		Fields: []
	>

上面打印到屏幕中的**Payload: Got 100 messages. 20.00 msg/sec**，就是counter插件的统计计算结果，counter插件是Heka自带的一个filter插件，这里打印到屏幕也是用的Heka自带的LogOut插件。

