---
layout: post
title: TCP self-connection
category: system
tagline: "Supporting tagline"
tags : [tcp, go]
---
{% include JB/setup %}

Go 语言的 net 库里有下面这样的一段代码，这段代码是用来发起一个 tcp 连接的，仔细阅读这段代码可以发现代码里处理了一种很不常见的特殊情形，也就是 tcp self-connection。代码中的注释解释得很详细了。

	func dialTCP(net string, laddr, raddr *TCPAddr, deadline time.Time) (*TCPConn, error) {
		fd, err := internetSocket(net, laddr, raddr, deadline, syscall.SOCK_STREAM, 0, "dial", sockaddrToTCP)
	
		// TCP has a rarely used mechanism called a 'simultaneous connection' in
		// which Dial("tcp", addr1, addr2) run on the machine at addr1 can
		// connect to a simultaneous Dial("tcp", addr2, addr1) run on the machine
		// at addr2, without either machine executing Listen.  If laddr == nil,
		// it means we want the kernel to pick an appropriate originating local
		// address.  Some Linux kernels cycle blindly through a fixed range of
		// local ports, regardless of destination port.  If a kernel happens to
		// pick local port 50001 as the source for a Dial("tcp", "", "localhost:50001"),
		// then the Dial will succeed, having simultaneously connected to itself.
		// This can only happen when we are letting the kernel pick a port (laddr == nil)
		// and when there is no listener for the destination address.
		// It's hard to argue this is anything other than a kernel bug.  If we
		// see this happen, rather than expose the buggy effect to users, we
		// close the fd and try again.  If it happens twice more, we relent and
		// use the result.  See also:
		//	http://golang.org/issue/2690
		//	http://stackoverflow.com/questions/4949858/
		//
		// The opposite can also happen: if we ask the kernel to pick an appropriate
		// originating local address, sometimes it picks one that is already in use.
		// So if the error is EADDRNOTAVAIL, we have to try again too, just for
		// a different reason.
		//
		// The kernel socket code is no doubt enjoying watching us squirm.
		for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
			if err == nil {
				fd.Close()
			}
			fd, err = internetSocket(net, laddr, raddr, deadline, syscall.SOCK_STREAM, 0, "dial", sockaddrToTCP)
		}
	
		if err != nil {
			return nil, &OpError{Op: "dial", Net: net, Addr: raddr, Err: err}
		}
		return newTCPConn(fd), nil
	}
	
	func selfConnect(fd *netFD, err error) bool {
		// If the connect failed, we clearly didn't connect to ourselves.
		if err != nil {
			return false
		}
	
		// The socket constructor can return an fd with raddr nil under certain
		// unknown conditions. The errors in the calls there to Getpeername
		// are discarded, but we can't catch the problem there because those
		// calls are sometimes legally erroneous with a "socket not connected".
		// Since this code (selfConnect) is already trying to work around
		// a problem, we make sure if this happens we recognize trouble and
		// ask the DialTCP routine to try again.
		// TODO: try to understand what's really going on.
		if fd.laddr == nil || fd.raddr == nil {
			return true
		}
		l := fd.laddr.(*TCPAddr)
		r := fd.raddr.(*TCPAddr)
		return l.Port == r.Port && l.IP.Equal(r.IP)
	}
	
	
tcp 这个不常显的自连接现象，可以用如下的一段脚本程序来复现：

	while true
	do
		telnet 127.0.0.1 40000
	done
	
本地机器并没有启动一个监听在40000端口的服务器程序，但是执行这段脚本一段时间，就会发现 telnet 程序连接上了，通过 netstat 看到的现象还是连接到自己，不是别的服务。

注意：只有当这里选择的端口位于这个范围的时候会出现这个现象：

	> cat /proc/sys/net/ipv4/ip_local_port_range
	32768	61000
	
为什么会出现自连接的情况，可以自行 Google，已经有很多文章基于 tcp 状态机做了详细解释。我个人认为：你可以把这个情况看做是 Linux 协议栈的一个 Bug，也可以看作是协议栈的一个特性，这都无关紧要；重要的是要清楚——“我们在写网络程序的时候，连接断开后，不断做建连重试就有小概率的情况会发生自连接”。

所以，在写网络程序的时候，我们应该主动的去处理自连接的情况，就像上面 Go 语言的处理方式就可以；另外，我们在选择服务器端口的时候，也可以稍加考虑，避免选择 /proc/sys/net/ipv4/ip_local_port_range 这个文件中描述的端口范围。


你的代码处理了这种情形了吗？