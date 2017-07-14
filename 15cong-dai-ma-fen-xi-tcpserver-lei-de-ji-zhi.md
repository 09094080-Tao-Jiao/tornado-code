接上面一小节，开始看 TCPServer 的 code。

[TCPServer](http://www.nowamagic.net/academy/tag/TCPServer)的\_\_init\_\_函数很简单，仅保存了参数而已。

唯一要注意的是，它可以接受一个io\_loop为参数。实际上io\_loop对TCPServer来说并不是可有可无，它是必须的。不过TCPServer提供了多种渠道来与一个io\_loop绑定，初始化参数只是其中一种绑定方式而已。

#### listen

接下来我们看一下listen函数，在helloworld.py中，httpserver实例创建之后，它被第一个调用。

TCPServer类的listen函数是开始接受指定端口上的连接。注意，这个listen与BSD Socket中的listen并不等价，它做的事比BSD socket\(\)+bind\(\)+listen\(\)还要多。

注意在函数注释中提到的一句话：你可以在一个server的实例中多次调用listen，以实现一个server侦听多个端口。

怎么理解？在BSD [Socket](http://www.nowamagic.net/academy/tag/Socket)架构里，我们不可能在一个socket上同时侦听多个端口。反推之，不难想到，TCPServer的listen函数内部一定是执行了全套的BSD Socket三段式（create socket-&gt;bind-&gt;listen），使得每调用一次listen实际上是创建了一个新的socket。

代码很好的符合了我们的猜想：

```
def listen(self, port, address=""):
	sockets = bind_sockets(port, address=address)
	self.add_sockets(sockets)

```

两步走，先创建了一个socket，然后把它加到自己的侦听队列里。

#### bind\_socket

bind\_socket函数并不是TCPServer的成员，它定义在netutil.py中，原型：

```
def bind_sockets(port, address=None, family=socket.AF_UNSPEC, backlog=128, flags=None):

```

它也有大段的注释。

bind\_socket完成的工作包括：创建socket，绑定socket到指定的地址和端口，开启侦听。

解释一下参数：

* port不用说，端口号嘛。
* address可以是IP地址，如“192.168.1.100”，也可以是hostname，比如“localhost”。如果是hostname，则可以监听该hostname对应的所有IP。如果address是空字符串（“”）或者None，则会监听主机上的所有接口。
* family是指网络层协议类型。可以选AF\_INET和AF\_INET6，默认情况下则两者都会被启用。这个参数就是在BSD Socket创建时的那个sockaddr\_in.sin\_family参数哈。
* backlog就是指侦听队列的长度，即BSD listen\(n\)中的那个n。
* flags参数是一些位标志，它是用来传递给socket.getaddrinfo\(\)函数的。比如socket.AI\_PASSIVE等。

另外要注意，在IPV6和IPV4混用的情况下，这个函数的返回值可以是一个socket列表，因为这时候一个address参数可能对应一个IPv4地址和一个IPv6地址，它们的socket是不通用的，会各自独立创建。

现在来一行一行看下bind\_socket的代码：

```
sockets = []
if address == "":
	address = None
if not socket.has_ipv6 and family == socket.AF_UNSPEC:
	# Python can be compiled with --disable-ipv6, which causes
	# operations on AF_INET6 sockets to fail, but does not
	# automatically exclude those results from getaddrinfo
	# results.
	# http://bugs.python.org/issue16208
	family = socket.AF_INET
if flags is None:
	flags = socket.AI_PASSIVE

```

这一段平淡无奇，基本上都是前面讲到的参数赋值。

接下来就是一个大的循环：

```
for res in set(socket.getaddrinfo(address, port, family, socket.SOCK_STREAM,0, flags)):

```

闹半天，前面解释的参数全都被socket.getaddrinfo\(\)这个函数吃下去了。

socket.getaddrinfo\(\)是python标准库中的函数，它的作用是将所接收的参数重组为一个结构res，res的类型将可以直接作为socket.socket\(\)的参数。跟BSD Socket中的getaddrinfo差不多嘛。

之所以用了一个循环，正如前面讲到的，因为IPv6和IPv4混用的情况下，getaddrinfo会返回多个地址的信息。参见python文档中的说明和示例：

> The function returns a list of 5-tuples with the following structure: \(family, type, proto, canonname, sockaddr\)

```
>
>
>
 socket.getaddrinfo("www.python.org", 80, proto=socket.SOL_TCP)
[(2, 1, 6, '', ('82.94.164.162', 80)),
 (10, 1, 6, '', ('2001:888:2000:d::a2', 80, 0, 0))]

```

接下来的代码在循环体中，是针对单个地址的。循环体内一开始就如我们猜想，直接拿getaddrinfo的返回值来创建socket。

```
af, socktype, proto, canonname, sockaddr = res
try:
	sock = socket.socket(af, socktype, proto)
except socket.error as e:
	if e.args[0] == errno.EAFNOSUPPORT:
		continue
raise

```

先从tuple中拆出5个参数，然后拣需要的来创建socket。

```
set_close_exec(sock.fileno())

```

这行是设置进程退出时对sock的操作。lose\_on\_exec 是一个进程所有文件描述符（文件句柄）的位图标志，每个比特位代表一个打开的文件描述符，用于确定在调用系统调用execve\(\)时需要关闭的文件句柄（参见include/fcntl.h）。当一个程序使用fork\(\)函数创建了一个子进程时，通常会在该子进程中调用execve\(\)函数加载执行另一个新程序。此时子进程将完全被新程序替换掉，并在子进程中开始执行新程序。若一个文件描述符在close\_on\_exec中的对应比特位被设置，那么在执行execve\(\)时该描述符将被关闭，否则该描述符将始终处于打开状态。

当打开一个文件时，默认情况下文件句柄在子进程中也处于打开状态。因此sys\_open\(\)中要复位对应比特位。

```
if os.name != 'nt':
	sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

```

对非NT的内核，需要额外设置一个SO\_REUSEADDR参数。有些系统的设计里，服务器进程结束后端口也会被内核保持一段时间，若我们迅速的重启服务器，可能会遇到“端口已经被占用”的情况。这个标志就是通知内核不要保持了，进程一关，立马放手，便于后来者重用。

```
if af == socket.AF_INET6:
	 # On linux, ipv6 sockets accept ipv4 too by default,
	 # but this makes it impossible to bind to both
	 # 0.0.0.0 in ipv4 and :: in ipv6.  On other systems,
	 # separate sockets *must* be used to listen for both ipv4
	 # and ipv6.  For consistency, always disable ipv4 on our
	 # ipv6 sockets and use a separate ipv4 socket when needed.
	 #
	 # Python 2.x on windows doesn't have IPPROTO_IPV6.
	 if hasattr(socket, "IPPROTO_IPV6"):
		 sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)

```

这段代码的说明已经很清楚了。

```
sock.setblocking(0)
sock.bind(sockaddr)
sock.listen(backlog)
sockets.append(sock)

```

前面经常提BSD Socket的这几个家伙，现在它们终于出现了。“非阻塞”性质也是在这里决定的。

每创建一个socket都将它加入到前面定义的列表里，最后函数结束时，将列表返回。其实这个函数蛮简单的。为什么它不是TCPServer的成员函数？

