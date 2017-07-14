Tornado的web框架\(tornado.web\)在web.py中实现，主要包括RequestHandler类（本质为对http请求处理的封装）和Application类（是一些列请求处理的集合，构成的一个web-application，源代码注释不翻译更容易理解：A collection of request handlers that make up a web application）。

#### RequestHandler分析

[RequestHandler](http://www.nowamagic.net/academy/tag/RequestHandler)提供了一个针对http请求处理的基类封装，方法比较多，主要有以下功能：

1. 提供了GET/HEAD/POST/DELETE/PATCH/PUT/OPTIONS等方法的功能接口，具体开发时RequestHandler的子类重写这些方法以支持不同需求的请求处理。
2. 提供对http请求的处理方法，包括对headers，页面元素，cookie的处理。
3. 提供对请求响应的一些列功能，包括redirect，write（将数据写入输出缓冲区），渲染模板（render, reander\_string）等。
4. 其他的一些辅助功能，如结束请求/响应，刷新输出缓冲区，对用户授权相关处理等。

#### Application分析

源代码中的注释写的非常好：

> A collection of request handlers that make up a web application. Instances of this class are callable and can be passed directly to HTTPServer to serve the application.

该类初始化的第一个参数接受一个\(regexp, request\_class\)形式的列表，指定了针对不同URL请求所采取的处理方法，包括对静态文件请求的处理（web.StaticFileHandler）。Application类中实现 \_\_call\_\_ 函数，这样该类就成为可调用的对象，由HTTPServer来进行调用。比如下边是httpserver.py中HTTPConection类的代码，该处request\_callback即为Application对象。

```
def _on_headers(self, data):
    # some codes...
    self.request_callback(self._request)

```

\_\_call\_\_函数会遍历Application的handlers列表，匹配到相应的URL后通过handler.\_execute进行相应处理；如果没有匹配的URL，则会调用ErrorHandler。

在[Application](http://www.nowamagic.net/academy/tag/Application)初始时有一个debug参数，当debug=True时，运行程序时当有代码、模块发生修改，程序会自动重新加载，即实现了auto-reload功能。该功能在autoreload.py文件中实现，是否需要reload的检查在每次接收到http请求时进行，基本原理是检查每一个sys.modules以及\_watched\_files所包含的模块在程序中所保存的最近修改时间和文件系统中的最近修改时间是否一致，如果不一致，则整个程序重新加载。

```
def _reload_on_update(modify_times):
    for module in sys.modules.values():
        # module test and some path handles
        _check_file(modify_times, path)
    for path in _watched_files:
        _check_file(modify_times, path)

```

Tornado的autoreload模块提供了一个对外的main接口，可以通过下边的方法实现运行test.py程序运行的auto-reload。但是测试了一下，功能有限，相比于django的autorelaod模块（具有较好的封装和较完善的功能）还是有一定的差距。最主要的原因是Tornado中的实现耦合了一些ioloop的功能，因而autoreload不是一个可独立的模块。

```
# tornado
python -m tornado.autoreload test.py [args...]

# django
from django.utils import autoreload
autoreload.main(your-main-func)

```

#### asynchronous方法

该方法通常被用为请求处理函数的decorator，以实现异步操作，被@asynchronous修饰后的请求处理为长连接，在调用self.finish之前会一直处于连接等待状态。

#### 总结

在前面小节 [Tornado HTTP服务器的基本流程](http://www.nowamagic.net/academy/detail/13321018) 中，给出了一张tornado httpserver的工作流程图，调用Application发生在HTTPConnection大方框的handle\_request椭圆中。那篇文章里使用的是一个简单的请求处理函数handle\_request，无论是handle\_request还是application，其本质是一个函数（可调用的对象），当服务器接收连接并读取http请求header之后进行调用，进行请求处理和应答。

```
http_server = httpserver.HTTPServer(handle_request)
http_server = httpserver.HTTPServer(application)

```



