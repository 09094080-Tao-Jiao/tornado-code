前面小节谈到了Tornado的RequestHandler和Application类，这两块内容还很多，分开来再补充一下，这篇先谈谈Application类。

总的来说，Application对象提供如下几个[接口](http://www.nowamagic.net/academy/tag/%E6%8E%A5%E5%8F%A3)：

* \_\_init\_\_ 接受路由-处理器列表，制定路由转发规则
* listen 建立服务器并监听端口，是对httpserver的封装调用
* add\_handlers 添加路由转发规则（包括主机名匹配）
* add\_transform 添加输出过滤器。例如gzip，chunk
* \_\_call\_\_ 服务器连接的网关处理接口，一般是被服务器调用

最简单的应该算是 add\_transform 了。将一个类添加到列表里就结束了。它会在输出时被调用，比较简单，略过不提。

然后是 listen。它接受端口，地址，其它参数。也很简单，用自身和参数构造 http 服务器，并让服务器监听在端口-地址上。其中涉及到底层 socket 的使用和 ioloop 事件绑定，放在以后再说。总之，可以认为产生了如下效果：在特定端口-地址上创建并监听 socket，并注册了该 socket 的可读事件到自身的\_\_call\_\_方法（亦即，每逢一个新连接到来时，\_\_call\_\_就会被调用）

接下来看 \_\_call\_\_ 方法。这是 python 的一个语法特性，这个函数使得 Application 可以直接被当成函数来使用。

这里有一个问题，为什么不直接定义一个函数例如 call 并在 listen 方法里把 self.call 传给服务器做回调函数，而是使用 self 呢？它们不都是函数吗？有什么区别呢？

区别还是有的。首先，如果使用self.call方法，那么它就是一个纯粹的函数，那么 application 的内部成员就不能用了（比如路由规则表）。而使用 self（也不是self.\_\_call\_\_\)传递给服务器做回调，当这个对象被当作函数调用时，\_\_call\_\_会被自动调用，此时对象上下文就被保留下来了。python 里好像经常这么搞。。。

好，来看看\_\_call\_\_的参数：request，HttpRequest对象，在 httputil 里被定义，在这里被用到的是它的 host 和 path 成员，用在路由匹配。忽略错误情况，这个方法的执行流程如下：\_get\_host\_handler\(request\) 得到该 host 对应的路径路由列表，默认情况下，任何 host 都会被匹配（原因详见\_\_init\_\_\)，返回的列表直接就是传递给构造 application 时的那个 tuple 列表，当然，对象变了，蛋内容是一样的。然后，对于路径路由列表中的每一个对象，用 request.path 来匹配，如果匹配上了，就生成 RequestHandler 对象，并根据正则表达式解析路径里的参数，可以用数字做键值也可以用字符串做键值。具体见 python 的 re.match.groups\(\)。然后跳出列表，执行\_execute\(\) 方法，这个方法在 RequestHandler 里被定义，下次再说，简言之，它的作用是 根据 http 方法转到对应方法，路径正则表达式里解析到的参数也被原样保留传进去。

一直有个疑问，路由规则是什么时候建立的呢？为此，我们先看 add\_handlers 方法，它接受两个参数，主机名正则，路径路由列表。同时，我们还要注意到，self.handlers 也是一个列表，是主机名路由列表，它的每个元素是一个 tuple，包含主机名和对应的路径路由列表。如图：

![](http://www.nowamagic.net/librarys/images/201312/2013_12_05_02.png)

所以，add\_handlers 的流程就很简单了：将路径路由列表和主机名合成一个 tuple 添加到 self.handlers 里，这样该主机名就会在\_get\_host\_handler 里被检索，就可以根据主机名找到对应的路径路由规则了。这里需要注意的一个问题是：由于.\*的特殊性（它会匹配任意字符），因此总是需要保证它被放置在列表的最后，所以就有了这段代码：

```
if self.handlers and self.handlers[-1][0].pattern == '.*$':
	self.handlers.insert(-1, (re.compile(host_pattern), handlers))

```

之前还说到，默认情况下，所有的主机名都会被匹配，那是因为在\_\_init\_\_方法里，它调用了 add\_handlers\(".\*",handlers\)。由于.\*匹配所有主机名，所以构造 application 对象时传入的路径路由规则列表就是最终默认路由列表了。

最后看一下\_\_init\_\_方法的流程。它一般接受一个参数，handlers，亦即最终匹配主机的路径路由列表。先设定 transform 列表，再设定静态文件的路由，然后添加主机\(.\*\)的路由列表。

好，回顾一下。[Application](http://www.nowamagic.net/academy/tag/Application) 的\_\_init\_\_方法设定了.\*主机的路径路由规则，listen 方法里开启了服务器并把自身作为回调。\_\_call\_\_方法在服务器 accept 到一个新连接时被调用，主要是根据路由规则转发请求到不同的处理器类，并在处理器里被分派到对应的具体方法中，到此完成请求的处理。

下一节是RequestHandler。

