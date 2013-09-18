---
layout: post
title: 使用github管理我的Vim
category: Vim
tagline: "Supporting tagline"
tags : [vim]
---
{% include JB/setup %}


自从把Vim的配置和插件全部管理在github上后，幸福指数直线飙升，再也不用担心丢掉vim的配置了。当然，你们不用告诉我，Emacs有xxx更好的，再好我也不会叛变的，嘿嘿。先说一下，以前的vim插件官网被墙了，现在有[http://vim-scripts.org/](http://vim-scripts.org/)可用，基本同步了所有插件。这里，必须得再感谢一下gmarik开发的[vundle](https://github.com/gmarik/vundle)，vundle是一个vim插件管理工具，这货太方便了，太实用了，推荐每一个Vimer使用。

管理Vim，主要就是管理Vim配置文件和所有安装的插件。我的所有插件现在都是来自于vim-scripts.org和github，这样一来我就不用再自己download插件，解压到.vim目录了，而是直接用vundle工具从网上自动安装我需要的插件。

####我的插件管理
插件实在太多，想用vundle来自动管理插件，就得先手工安装vundle(别告诉我，这你都懒得装)，然后在.vimrc文件里配置好vundle工具以及想要的插件名字，我使用到的所有插件如下：

	" ----------------------------------------------------------------------------
	"                           vundle, 插件管理
	" ----------------------------------------------------------------------------
	set nocompatible
	filetype off
	set rtp+=~/.vim/bundle/vundle/
	call vundle#rc()
	Bundle 'gmarik/vundle'


	" github 库
	"
	" Go 语言插件
	Bundle 'skoo87/go.vim'

	" vim里面支持shell终端
	Bundle 'skoo87/vimproc'
	Bundle 'skoo87/vimshell'

	" c/c++项目工程插件
	Bundle 'skoo87/p'

	" 书签插件
	Bundle 'skoo87/bookmarking.vim'

	" 主要提供 xolox#shell#execute() 后台执行外部命令的接口
	Bundle 'vim-scripts/shell.vim--Odding'

	" vim-scripts 库
	"
	Bundle 'taglist.vim'
	Bundle 'OmniCppComplete'
	Bundle 'The-NERD-tree'
	Bundle 'SuperTab'
	Bundle 'a.vim'
	Bundle 'c.vim'
	Bundle 'genutils'
	Bundle 'grep.vim'
	Bundle 'SudoEdit.vim'

	" NOTE: 自己修改了plugin/lookupfile.vim中的快捷键为F4, 默认F5.
	Bundle 'lookupfile'

	Bundle 'unite.vim'
	Bundle 'desertEx'
	Bundle 'CSApprox'

	filetype plugin indent on

我实在懒得解释这个配置的意思，其实不需要过多的解释，vundle的文档介绍的足够的详细。我的配置做的事情主要就是将我需要的插件名字全部写到配置文件中，并不需要我去一个一个的download下来。

到这里，配置已经搞好了，这个时候只需要打开Vim，执行命令`:BundleInstall`就可以自动安装好所有插件了，是不是很赞啊？我实在觉得特别赞啊。vundle还可以搜索插件，卸载插件等等。

####我的.vimrc配置文件管理

经过vundle来自动管理插件后，可以看出.vimrc文件显得更加的重要了，其实我们只需要把这个文件拷贝到另外一台机器，然后打开vim执行`:BundleInstall`就完成了整个Vim的安装配置，彻底把我们解放出来、不再折腾了。

这里，我也将.vimrc文件push到github保存。我在github上建了一个项目叫`vimrc`，这个项目里有一个文件也叫`vimrc`, 这个文件就是我的.vimrc。然后我将这个vimrc clone到本地，再在home目录下创建一个`.vimrc`的`软连接`指向从github clone下来的配置文件。

	/Users/marckywu/.vimrc -> /Users/marckywu/local/vimrc/vimrc

从此过来，我只要修改了.vimrc配置文件，就把修改push到github上。我真的是再也不用为vim配置操心了。


最后，再次感谢vim-scripts和vundle两个项目，真的是利天下。

不过，用Vim和Emacs真的效率会高很多吗？扯淡。反正我是因为在Linux/Mac上找不到一个轻量的IDE才用Vim的，都是被逼的。