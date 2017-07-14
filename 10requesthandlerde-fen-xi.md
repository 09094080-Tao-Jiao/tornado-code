前面一小节谈到了Application 类，这里再来看看 [RequestHandler](http://www.nowamagic.net/academy/tag/RequestHandler) 类。

从上一节的流程可以看出，RequestHandler 类把 \_execute 方法暴露给了 application 对象，在这个方法里完成了请求的具体分发和处理。因此，我主要看这一方法（当然还包括\_\_init\_\_\)，其它方法在开发应用时自然会用到，还是比较实用的，比如header,cookie,get/post参数的getter/setter方法，都是必须的。

首先是\_\_init\_\_。负责对象的初始化，在对象被构造时一定会被调用的。

那对象什么时候被调用呢？从上一节可以看到，在.\*主机的路径路由列表里，当路径正则匹配了当前请求的路由时，application 就会新建一个 RequestHandler 对象（实际上是子类对象），然后调用 \_execute 方法。\_\_init\_\_ 方法接受3个参数 : application, request, \*\*kwargs，分别是application单例，当前请求request 和 kwargs （暂时没被用上。不过可以覆盖initialize方法，里面就有它）。这个kwargs 是静态存储在路由列表里的，它最初是在给 application 设置路由列表时除了路径正则，处理器类之外的第三个对象，是一个字典，一般情况是空的。\_\_init\_\_ 方法也没做什么，就是建立了这个对象和 application, request 的关联就完了。构造器就应该是这样，对吧？

接下来会调用 \_execute 方法。

该方法接受三个参数 transforms\(相当于 application 的中间件吧，对流程没有影响），\*args\(用数字做索引时的正则 group\)，\*\*kwargs\(用字符串做键值时的正则 group，与\_\_init\_\_的类似但却是动态的\)。该方法先设定好 transform（因为不管错误与否都算输出，因此中间件都必须到位）。然后，检查 http 方法是否被支持，然后整合路径里的参数，检查 XSRF 攻击。然后 prepare\(\)。这总是在业务逻辑前被执行，在业务逻辑后还有个 finish\(\)。业务逻辑代码被定义在子类的 get/post/put/delete 方法里，具体视详细情况而定。

还有一个 finish 方法，它在业务逻辑代码后被执行。缓冲区里的内容要被输出，连接总得被关闭，资源总得被释放，所以这些善后事宜就交给了 finish。与缓冲区相关的还有一个函数 flush，顾名思义它会调用 transforms 对输出做预处理，然后拼接缓冲区一次性输出 self.request.write\(headers + chunk, callback=callback\)。

以上，就是 handler 的分析。handler 的读与写，实际上都是依靠 [request](http://www.nowamagic.net/academy/tag/request) 对象来完成的，而 request 到底如何呢？且看下回分解。

