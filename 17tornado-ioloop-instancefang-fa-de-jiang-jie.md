Tornado 的源码写得有点难懂，需要你理解好 socket、epoll 这样的东西才能充分理解。需要深入到 Tornado 的源码，ioloop.py 这个文件很关键。

接下来，我们继续读 ioloop.py 这个文件。

[IOLoop](http://www.nowamagic.net/academy/tag/IOLoop) 是基于 epoll 实现的底层网络I/O的核心调度模块，用于处理 socket 相关的连接、响应、异步读写等网络事件。每个 Tornado 进程都会初始化一个全局唯一的 IOLoop 实例，在 IOLoop 中通过静态方法 instance\(\) 进行封装，获取 IOLoop 实例直接调用此方法即可。

    @staticmethod
    def instance():
    	"""Returns a global `IOLoop` instance.

    	Most applications have a single, global `IOLoop` running on the
    	main thread.  Use this method to get this instance from
    	another thread.  To get the current thread's `IOLoop`, use `current()`.
    	"""
    	if not hasattr(IOLoop, "_instance"):
    		with IOLoop._instance_lock:
    			if not hasattr(IOLoop, "_instance"):
    				# New instance after double check
    				IOLoop._instance = IOLoop()
    	return IOLoop._instance


Tornado 服务器启动时会创建监听 socket，并将 socket 的 file descriptor 注册到 IOLoop 实例中，IOLoop 添加对 socket 的IOLoop.READ 事件监听并传入回调处理函数。当某个 socket 通过 accept 接受连接请求后调用注册的回调函数进行读写。接下来主要分析IOLoop 对 epoll 的封装和 I/O 调度具体实现。

epoll是Linux内核中实现的一种可扩展的I/O事件通知机制，是对POISX系统中 select 和 poll 的替代，具有更高的性能和扩展性，FreeBSD中类似的实现是kqueue。Tornado中基于Python C扩展实现的的epoll模块\(或kqueue\)对epoll\(kqueue\)的使用进行了封装，使得IOLoop对象可以通过相应的事件处理机制对I/O进行调度。具体可以参考前面小节的 [预备知识：我读过的对epoll最好的讲解](http://www.nowamagic.net/academy/detail/13321005) 。

IOLoop模块对网络事件类型的封装与epoll一致，分为READ / WRITE / ERROR三类，具体在源码里呈现为：

```
# Our events map exactly to the epoll events
NONE = 0
READ = _EPOLLIN
WRITE = _EPOLLOUT
ERROR = _EPOLLERR | _EPOLLHUP

```

回到前面章节的 [开始用Tornado：从Hello World开始](http://www.nowamagic.net/academy/detail/1332505) 里面的示例，

```
http_server = tornado.httpserver.HTTPServer(application)
http_server.listen(options.port)
tornado.ioloop.IOLoop.instance().start()

```

前两句是启动服务器，启动服务器之后，还需要启动 IOLoop 的实例，这样可以启动事件循环机制，配合非阻塞的 HTTP Server 工作。更多关于 IOLoop的与Http服务器的细节，在 [Tornado对Web请求与响应的处理机制](http://www.nowamagic.net/academy/detail/1332514) 这里有介绍到。

这就是 IOLoop 的 [instance\(\)](http://www.nowamagic.net/academy/tag/instance) 方法的一些细节，接下来我们再看看 start\(\) 的细节。

