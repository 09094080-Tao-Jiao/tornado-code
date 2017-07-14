前文已经说过，HTTPServer是派生自[TCPServer](http://www.nowamagic.net/academy/tag/TCPServer)，从协议层次上讲，这再自然不过。

从TCPServer的实现上看，它是一个通用的server框架，基本是按照BSD socket的思想设计的。create-bind-listen三段式一个都不少。

从helloworld.py往下追，可以看到：

1. helloworld.py中的main函数创建了HTTPServer.
2. HTTPServer继承自TCPServer，在HTTPServer的构造函数中直接调用了TCPServer的构造函数。

接下来我们就去看看TCPServer这个类的实现，它的代码放在tornado/tcpserver.py中。tcpserver.py只有两百多行，不算多。所有代码都是在实现TCPServer这个类。

#### TCPServer

在TCPServer类的注释中，首先强调了它是一个non-blocking, single-threaded TCP Server。

怎么理解呢？ 

non-blocking，就是说，这个服务器没有使用阻塞式API。

什么是阻塞式设计？举个例子，在BSD Socket里，recv函数默认是阻塞式的。使用recv读取客户端数据时，如果对方并未发送数据，则这个API就会一直阻塞那里不返回。这样服务器的设计不得不使用多线程或者多进程方式，避免因为一个API的阻塞导致服务器没法做其它事。阻塞式API是很常见的，我们可以简单认为，阻塞式设计就是“不管有没有数据，服务器都派API去读，读不到，API就不会回来交差”。

而[非阻塞](http://www.nowamagic.net/academy/tag/%E9%9D%9E%E9%98%BB%E5%A1%9E)，对recv来说，区别在于没有数据可读时，它不会在那死等，它直接就返回了。你可能会认为这办法比阻塞式还要矬，因为服务器无法预知有没有数据可读，不得不反复派recv函数去读。这不是浪费大量的CPU资源么？

当然不会这么傻。tornado这里说的非阻塞要高级得多，基本上是另一种思路：服务器并不主动读取数据，它和操作系统合作，实现了一种“监视器”，TCP连接就是它的监视对象。当某个连接上有数据到来时，操作系统会按事先的约定通知服务器：某某号连接上有数据到来，你去处理一下。服务器这时候才派API去取数据。服务器不用创建大量线程来阻塞式的处理每个连接，也不用不停派API去检查连接上有没有数据，它只需要坐那里等操作系统的通知，这保证了recv API出手就不会落空。

tornado另一个被强调的特征是single-threaded，这是因为我们的“监视器”非常高效，可以在一个线程里监视成千上万个连接的状态，基本上不需要再动用线程来分流。实测表明，它比阻塞式多线程或者多进程设计更加高效——当然，这依赖于操作系统的大力配合，现在主流操作系统都提供了非常高端大气上档次的“监视器”机制，比如epoll、kqueue。

作者提到这个类一般不直接被实例化，而是由它派生出子类，再用子类实例化。

为了强化这个设计思想，作者定义了一个未直接实现的接口，叫handle\_stream\(\)。

    def handle_stream(self, stream, address):
        """Override to handle a new `.IOStream` from an incoming connection."""
        raise NotImplementedError()


这倒是个不错的技巧，强制让子类覆盖本方法，不然就报错给你看！

TCPServer是支持SSL的。由于Python的强大，支持SSL一点都不费事。要启动一个支持SSL的TCPServer，只需要告诉它你的certifile和keyfile就行。

```
TCPServer(ssl_options={"certfile": os.path.join(data_dir, "mydomain.crt"),
	"keyfile": os.path.join(data_dir, "mydomain.key"),})

```

关于这两个文件的来龙去脉，可以去Google“数字证书原理”这篇文章。

#### TCPServer的三种形式

TCPServer的初始化有三种形式。

1. 单进程形式

```
server = TCPServer()
server.listen(8888)
IOLoop.instance().start()

```

我们在helloworld.py中看到的就是这种用法，不再赘述。

2. 多进程形式。

```
server = TCPServer()
server.bind(8888)
server.start(0)  # Forks multiple sub-processes
IOLoop.instance().start()

```

区别主要在server.start\(0\)这里。后面分析listen\(\)与start\(\)两个成员函数时，就会看到它们是怎么跟进程结合的。

注意：这种模式启动时，不能把IOLoop对象传递给TCPServer的构造函数，这样会导致TCPServer直接按单进程启动。

3. 高级多进程形式。

```
sockets = bind_sockets(8888)
tornado.process.fork_processes(0)
server = TCPServer()
server.add_sockets(sockets)
IOLoop.instance().start()

```

高级意味着复杂。从上面代码看，虽然只多了一两行，实际里面的流程有比较大的差别。

这种方式的主要优点就是 tornado.process.fork\_processes\(0\)这句，它为进程的创建提供了更多的灵活性。当然现在说了也是糊涂，后面钻进这些代码后，我们再来验证这里的说法。

以上内容都是TCPServer类的doc string中提到的。后面小节开始看code。

