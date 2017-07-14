TCPServer.bind\_sockets\(\)会返回一个socket对象的列表，列表中的socket都是用来监听客户端连接的。

列表由TCPServer.add\_sockets\(\)处理。在这个函数里我们就会看到IOLoop相关的东西。

```
def add_sockets(self, sockets):
    if self.io_loop is None:
        self.io_loop = IOLoop.current()

    for sock in sockets:
        self._sockets[sock.fileno()] = sock
        add_accept_handler(sock, self._handle_connection, io_loop=self.io_loop)

```

首先，io\_loop是[TCPServer](http://www.nowamagic.net/academy/tag/TCPServer)的一个成员变量，这说明每个TCPServer都绑定了一个io\_loop。注意，跟传统的做法不同，ioloop不是跟socket一一对应，而是跟TCPServer一一对应。也就是说，一个Server上即使有多个listening socket，他们也是由同一个ioloop在处理。

前面提到过，[HTTPServer](http://www.nowamagic.net/academy/tag/HTTPServer)的初始化可以带一个ioloop参数，最终它会被赋值给TCPServer的成员。如果没有带ioloop参数（如helloworld.py所展示的），TCPServer则会自己倒腾一个，即IOLoop.current\(\)。

add\_accept\_handler\(\)定义在netutil.py中（bind\_sockets在这里）。它的代码并复杂，如下：

```
def add_accept_handler(sock, callback, io_loop=None):
    if io_loop is None:
        io_loop = IOLoop.current()
    def accept_handler(fd, events):
        while True:
            try:
                connection, address = sock.accept()
            except socket.error as e:
                # EWOULDBLOCK and EAGAIN indicate we have accepted every
                # connection that is available.
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                # ECONNABORTED indicates that there was a connection
                # but it was closed while still in the accept queue.
                # (observed on FreeBSD).
                if e.args[0] == errno.ECONNABORTED:
                    continue
                raise
            callback(connection, address)
    io_loop.add_handler(sock.fileno(), accept_handler, IOLoop.READ)

```

文档中说，IOLoop.current\(\)返回当前线程的IOLoop对象。可能有点不好理解。

实际上IOLoop的实例就相当于一个线程，有start, run, stop这些函数。IOLoop实例都包含一个叫\_current的成员，指向创建它的线程。每个线程在创建IOLoop时，都被绑定到创建它的线程上。在IOLoop的类定义中，有这么一行：

```
_current = threading.local()

```

创建好IOLoop后，下面又定义了一个accept\_handler。这是一个函数内定义的函数。

accept\_handler相当于连接监视器的，所以我们把它绑定到listening socket上。

```
io_loop.add_handler(sock.fileno(), accept_handler, IOLoop.READ)

```

它正式对socket句柄调用 accept。当接收到一个connection后，调用callback\(\)。不难想到，callback就是客户端连上来后对应的响应函数。

回到add\_sockets函数里看：

```
add_accept_handler(sock, self._handle_connection,
    io_loop=self.io_loop)

```

callback就是我们传进去的TCPServer.\_handle\_connection\(\)函数。TCPServer.\_handle\_connection\(\)本身不复杂。前面大段都是在处理SSL事务，把这些东西都滤掉的话，实际上代码很少。

```
def _handle_connection(self, connection, address):
    try:
        stream = IOStream(connection, io_loop=self.io_loop, max_buffer_size=self.max_buffer_size)
        self.handle_stream(stream, address)
    except Exception:
        app_log.error("Error in connection callback", exc_info=True)

```

思路是很清晰的，客户端连接在这里被转化成一个IOStream。然后由handle\_stream函数处理。

这个handle\_stream就是我们前面提到过的未直接实现的接口，它是由HTTPServer类实现的。

```
def handle_stream(self, stream, address):
    HTTPConnection(stream, address, self.request_callback,
		self.no_keep_alive, self.xheaders, self.protocol)

```

最后，处理流程又回到了HTTPServer类中。可以预见，在HTTConnection这个类中，stream将和我们注册的RequestHandler协作，一边读客户端请求，一边调用相应的handler处理。

总结一下：

listening socket创建起来以后，我们给它绑定一个响应函数叫accept\_handler。当用户连接上来时，会触发listening socket上的事件，然后accept\_handler被调用。accept\_handler在listening socket上获得一个针对该用户的新socket。这个socket专门用来跟用户做数据沟通。TCPServer把这个socket封闭成一个IOStream，最后交到HTTPServer的handle\_stream里。

经过一翻周围，用户连接后的HTTP通讯终于被我们导入到了HTTPConnection。在HTTPConnection里我们将看到熟悉的HTTP通信协议的处理。

HTTPConnection类也定义在httpserver.py中，它的构造函数如下：

```
def __init__(self, stream, address, request_callback, no_keep_alive=False,
             xheaders=False, protocol=None):
    self.stream = stream
    self.address = address
    # Save the socket's address family now so we know how to
    # interpret self.address even after the stream is closed
    # and its socket attribute replaced with None.
    self.address_family = stream.socket.family
    self.request_callback = request_callback
    self.no_keep_alive = no_keep_alive
    self.xheaders = xheaders
    self.protocol = protocol
    self._clear_request_state()
    # Save stack context here, outside of any request.  This keeps
    # contexts from one request from leaking into the next.
    self._header_callback = stack_context.wrap(self._on_headers)
    self.stream.set_close_callback(self._on_connection_close)
    self.stream.read_until(b"\r\n\r\n", self._header_callback)

```

常规的参数保存动作，还有一些初始化、清理动作，最后一句开始办正事：

```
self.stream.read_until(b”\r\n\r\n”, self._header_callback)

```

从socket中读数据，直到读到”\r\n\r\n”为止。这是HTTP头部结束标志，读到的数据就会由self.\_header\_callback处理。

经过stack\_context.wrap\(\)的传递，HTTP头会交给HTTPConnection.\_on\_headers\(\)。

HTTPConnection.\_on\_headers\(\)完成了HTTP头的分析，具体过程这里就不必详述了（如果有写HTTPServer需求，倒是可以借鉴一下）。

经过一轮校验与分析，HTTP头（注意，只是HTTP头部哦）被组装成一个HTTPRequest对象。（HTTP Body如果数据量不是太大，就直接放进了一个buffer里，就叫self.\_on\_request\_body。）

```
self._request = HTTPRequest(
    connection=self, method=method, uri=uri, version=version,
    headers=headers, remote_ip=remote_ip, protocol=self.protocol)

```

HTTPRequest对象被交给了（其实HTTP Body最后也是交给他的……）

```
self.request_callback(self._request)

```

这个request\_callback是什么来头呢？它是在HTTPConnection构造时传进来的参数。我们回到HTTPServer.handle\_stream\(\)

```
def handle_stream(self, stream, address):
    HTTPConnection(stream, address, self.request_callback,
                   self.no_keep_alive, self.xheaders, self.protocol)

```

它是一个HTTPServer类的成员，继续往回追来历：

```
def __init__(self, request_callback, no_keep_alive=False, io_loop=None,
             xheaders=False, ssl_options=None, protocol=None, **kwargs):
    self.request_callback = request_callback
    self.no_keep_alive = no_keep_alive
    self.xheaders = xheaders
    self.protocol = protocol
    TCPServer.__init__(self, io_loop=io_loop, ssl_options=ssl_options,
                       **kwargs)

```

Bingo！这就是HTTPServer初始化时传进来的那个RequestHandler。

在helloworld.py里，我们看到的是：

```
application = tornado.web.Application([(r"/", MainHandler), ])
http_server = tornado.httpserver.HTTPServer(application)

```

在另一个例子里，我们看到的是：

```
def handle_request(request):
   message = "You requested %s\n" % request.uri
   request.write("HTTP/1.1 200 OK\r\nContent-Length: %d\r\n\r\n%s" % (
                 len(message), message))
   request.finish()
http_server = tornado.httpserver.HTTPServer(handle_request)

```

可见这个request\_handler通吃很多种类型的参数，可以是一个Application类的对象，也可是一个简单的函数。

如果是handler是简单函数，如上面的handle\_request，这个很好理解，由一个函数处理HTTPRequest对象嘛。

如果是一个Application对象，就有点奇怪了。我们能把一个对象作另一个对象的参数来呼叫吗？

Python中有一个有趣的语法，只要定义类型的时候，实现\_\_call\_\_函数，这个类型就成为可调用的。换句话说，我们可以把这个类的对象当作函数来使用，相当于重载了括号运算符。

