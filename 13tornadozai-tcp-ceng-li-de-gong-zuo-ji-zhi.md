上一节是关于应用层的协议 HTTP，它依赖于传输层协议 [TCP](http://www.nowamagic.net/academy/tag/TCP)，例如服务器是如何绑定端口的？HTTP 服务器的 handle\_stream 是在什么时候被调用的呢？本节聚焦在 TCP 层次的实现，以便和上节的程序流程衔接起来。

首先是关于 TCP 协议。这是一个面向连接的可靠交付的协议。由于是面向连接，所以在服务器端需要分配内存来记忆客户端连接，同样客户端也需要记录服务器。由于保证可靠交付，所以引入了很多保证可靠性的机制，比如定时重传机制，SYN/ACK 机制等，相当复杂。所以，在每个系统里的 TCP 协议软件都是相当复杂的，本文不打算深入谈这些（我也谈不了多少，呵呵）。但我们还是得对 TCP 有个了解。先上一张图（UNIX 网络编程）-- 状态转换图。

![](http://www.nowamagic.net/librarys/images/201312/2013_12_06_01.png)

除外，来一段TCP服务器端编程经典三段式代码（C实现）：

```
// 创建监听socket
int sfd = socket(AF_INET, SOCK_STREAM, 0);  
// 绑定socket到地址-端口， 并在该socket上开始监听。listen的第二个参数叫backlog，和连接队列有关
bind(sfd,(struct sockaddr *)(
&
s_addr), sizeof(struct sockaddr)) 
&
&
 listen(sfd, 10); 
while(1) cfd = accept(sfd, (struct sockaddr *)(
&
cli_addr), 
&
addr_size);

```

以上，忽略所有错误处理和变量声明，顾名思义吧…… 更多详细，可以搜 Linux TCP 服务器编程。所以，对于 TCP 编程的总结就是：创建一个监听 socket，然后把它绑定到端口和地址上并开始监听，然后不停 accept。这也是 tornado 的 TCPServer 要做的工作。

TCPServer 类的定义在 tcpserver.py。它有两种用法：bind+start 或者 listen。

第一种用法可用于多线程，但在 TCP 方面两者是一样的。就以 listen 为例吧。TCPServer 的\_\_init\_\_没什么注意的，就是记住了 ioloop 这个单例，这个下节再分析（它是tornado异步性能的关键）。listen 方法接收两个参数端口和地址，代码如下：

    def listen(self, port, address=""):
    	"""Starts accepting connections on the given port.

    	This method may be called more than once to listen on multiple ports.
    	`listen` takes effect immediately; it is not necessary to call
    	`TCPServer.start` afterwards.  It is, however, necessary to start
    	the `.IOLoop`.
    	"""
    	sockets = bind_sockets(port, address=address)
    	self.add_sockets(sockets)


以上。首先 bind\_sockets 方法接收地址和端口创建 sockets 列表并绑定地址端口并监听（完成了TCP三部曲的前两部），add\_sockets 在这些 sockets 上注册 read/timeout 事件。有关高性能并发服务器编程可以参照UNIX网络编程里给的几种编程模型，tornado 可以看作是单线程事件驱动模式的服务器，TCP 三部曲中的第三部就被分隔到了事件回调里，因此肯定要在所有的文件 fd（包括sockets）上监听事件。在做完这些事情后就可以安心的调用 ioloop 单例的 start 方法开始循环监听事件了。具体细节可以参照现代高性能 web 服务器\(nginx/lightttpd等\)的事件模型，后面也会涉及一点。

简言之，基于事件驱动的服务器（tornado）要干的事就是：创建 [socket](http://www.nowamagic.net/academy/tag/socket)，绑定到端口并 listen，然后注册事件和对应的回调，在回调里accept 新请求。

bind\_sockets 方法在 netutil 里被定义，没什么难的，创建监听 socket 后为了异步，设置 socket 为非阻塞（这样由它 accept 派生的socket 也是非阻塞的），然后绑定并监听之。add\_sockets 方法接收 socket 列表，对于列表中的 socket，用 fd 作键记录下来，并调用add\_accept\_handler 方法。它也是在 netutil 里定义的，代码如下：

    def add_accept_handler(sock, callback, io_loop=None):
        """Adds an `.IOLoop` event handler to accept new connections on ``sock``.

        When a connection is accepted, ``callback(connection, address)`` will
        be run (``connection`` is a socket object, and ``address`` is the
        address of the other end of the connection).  Note that this signature
        is different from the ``callback(fd, events)`` signature used for
        `.IOLoop` handlers.
        """
        if io_loop is None:
            io_loop = IOLoop.current()

        def accept_handler(fd, events):
            while True:
                try:
                    connection, address = sock.accept()
                except socket.error as e:
                    if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                        return
                    raise
                callback(connection, address)
        io_loop.add_handler(sock.fileno(), accept_handler, IOLoop.READ)


需要注意的一个参数是 callback，现在指向的是 TCPServer 的 \_handle\_connection 方法。add\_accept\_handler 方法的流程：首先是确保ioloop对象。然后调用 add\_handler 向 loloop 对象注册在fd上的read事件和回调函数accept\_handler。该回调函数是现成定义的，属于IOLoop层次的回调，每当事件发生时就会调用。回调内容也就是accept得到新socket和客户端地址，然后调用callback向上层传递事件。从上面的分析可知，当read事件发生时，accept\_handler被调用，进而callback=\_handle\_connection被调用。

\_handle\_connection就比较简单了，跳过那些ssl的处理，简化为两句stream = IOStream\(connection, io\_loop=self.io\_loop\)和self.handle\_stream\(\)。这里IOStream代表了IO层，以后再说，反正读写是不愁了。接着是调用handle\_stream。我们可以看到，不论应用层是什么协议（或者自定义协议），当有新连接到来时走的流程是差不多的，都要经历一番上诉的回调，不同之处就在于这个handle\_stream方法。这个方法是由子类自定义覆盖的，它的HTTP实现已经在上一节看过了。

到此，和上节的代码流程接上轨了。当事件发生时是如何回调的呢？app.py里的IOLoop.instance\(\).start\(\)又是怎样的流程呢？明天继续，看tornado异步高性能的根本所在

