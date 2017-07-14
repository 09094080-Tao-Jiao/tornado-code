IOStream对socket读写进行了封装，分别提供读、写缓冲区实现对[socket](http://www.nowamagic.net/academy/tag/socket)的异步读写。当socket被accept之后HTTPServer的\_handle\_connection会被回调并初始化IOStream对象，进一步通过IOStream提供的功能接口完成socket的读写。文章接下来将关注IOStream实现读写的细节。

#### IOStream的初始化

[IOStream](http://www.nowamagic.net/academy/tag/IOStream)初始化过程中主要完成以下操作：

1. 绑定对应的socket
2. 绑定ioloop
3. 创建读缓冲区\_read\_buffer，一个python deque容器
4. 创建写缓冲区\_write\_buffer，同样也是一个python deque容器

#### IOStream提供的主要功能接口

主要的读写接口包括以下四个：

```
class IOStream(object):
    def read_until(self, delimiter, callback): 
    def read_bytes(self, num_bytes, callback, streaming_callback=None): 
    def read_until_regex(self, regex, callback): 
    def read_until_close(self, callback, streaming_callback=None): 
    def write(self, data, callback=None):
```

* read\_until和read\_bytes是最常用的读接口，它们工作的过程都是先注册读事件结束时调用的回调函数，然后调用\_try\_inline\_read方法。\_try\_inline\_read首先尝试\_read\_from\_buffer，即从上一次的读缓冲区中取数据，如果有数据直接调用 self.\_run\_callback\(callback, self.\_consume\(data\_length\)\) 执行回调函数，\_consume消耗掉了\_read\_buffer中的数据；否则即\_read\_buffer之前没有未读数据，先通过\_read\_to\_buffer将数据从socket读入\_read\_buffer，然后再执行\_read\_from\_buffer操作。read\_until和read\_bytes的区别在于\_read\_from\_buffer过程中截取数据的方法不同，read\_until读取到delimiter终止，而read\_bytes则读取num\_bytes个字节终止。执行过程如下图所示：

![](/assets/import.png)

* read\_until\_regex相当于delimiter为某一正则表达式的read\_until。

* read\_until\_close主要用于IOStream流关闭前后的读取：如果调用read\_until\_close时stream已经关闭，那么将会\_consume掉\_read\_buffer中的所有数据；否则\_read\_until\_close标志位设为True，注册\_streaming\_callback回调函数，调用\_add\_io\_state添加io\_loop.READ状态。

* write首先将data按照数据块大小WRITE\_BUFFER\_CHUNK\_SIZE分块写入write\_buffer，然后调用handle\_write向socket发送数据。

#### 其他内部功能接口

* def \_handle\_events\(self, fd, events\): 通常为IOLoop对象add\_handler方法传入的回调函数，由IOLoop的事件机制来进行调度。
* def \_add\_io\_state\(self, state\): 为IOLoop对象的handler注册IOLoop.READ或IOLoop.WRITE状态，handler为IOStream对象的\_handle\_events方法。
* def \_consume\(self, loc\): 合并读缓冲区loc个字节，从读缓冲区删除并返回这些数据。



