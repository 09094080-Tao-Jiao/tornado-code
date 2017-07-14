#### IOLoop的初始化

初始化过程中选择 epoll 的实现方式，Linux 平台为 epoll，BSD 平台为 kqueue，其他平台如果安装有C模块扩展的 epoll 则使用 tornado对 epoll 的封装，否则退化为 select。

```
def __init__(self, impl=None):
    self._impl = impl or _poll()
    #省略部分代码
    self._waker = Waker()
    self.add_handler(self._waker.fileno(),
                     lambda fd, events: self._waker.consume(),
                     self.READ)

def add_handler(self, fd, handler, events):
    """Registers the given handler to receive the given events for fd."""
    self._handlers[fd] = stack_context.wrap(handler)
    self._impl.register(fd, events | self.ERROR)

```

在 IOLoop 初始化的过程中创建了一个 Waker 对象，将 Waker 对象 fd 的读端注册到事件循环中并设定相应的回调函数（这样做的好处是当事件循环阻塞而没有响应描述符出现，需要在最大 timeout 时间之前返回，就可以向这个管道发送一个字符）。

Waker 的使用：一种是在其他线程向 IOLoop 添加 callback 时使用，唤醒 IOLoop 同时会将控制权转移给 IOLoop 线程并完成特定请求。唤醒的方法向管道中写入一个字符'x'。另外，在 IOLoop的stop 函数中会调用self.\_waker.wake\(\)，通过向管道写入'x'停止事件循环。

add\_handler 函数使用了stack\_context 提供的 wrap 方法。wrap 返回了一个可以直接调用的对象并且保存了传入之前的堆栈信息，在执行时可以恢复，这样就保证了函数的异步调用时具有正确的运行环境。

#### IOLoop的start方法

IOLoop 的核心调度集中在 [start\(\)](http://www.nowamagic.net/academy/tag/start) 方法中，IOLoop 实例对象调用 start 后开始 epoll 事件循环机制，该方法会一直运行直到 IOLoop 对象调用 stop 函数、当前所有事件循环完成。start 方法中主要分三个部分：一个部分是对超时的相关处理；一部分是 epoll 事件通知阻塞、接收；一部分是对 epoll 返回I/O事件的处理。

* 为防止 IO event starvation，将回调函数延迟到下一轮事件循环中执行。
* 超时的处理 heapq 维护一个最小堆，记录每个回调函数的超时时间（deadline）。每次取出 deadline 最早的回调函数，如果callback标志位为 True 并且已经超时，通过 \_run\_callback 调用函数；如果没有超时需要重新设定 poll\_timeout 的值。
* 通过 self.\_impl.poll\(poll\_timeout\) 进行事件阻塞，当有事件通知或超时时 poll 返回特定的 event\_pairs。
* epoll 返回通知事件后将新事件加入待处理队列，将就绪事件逐个弹出，通过stack\_context.wrap\(handler\)保存的可执行对象调用事件处理。

```
while True:
    poll_timeout = 3600.0

    with self._callback_lock:
        callbacks = self._callbacks
        self._callbacks = []
    for callback in callbacks:
        self._run_callback(callback)

    # 超时处理
    if self._timeouts:
        now = time.time()
        while self._timeouts:
            if self._timeouts[0].callback is None:
                # the timeout was cancelled
                heapq.heappop(self._timeouts)
            elif self._timeouts[0].deadline 
<
= now:
                timeout = heapq.heappop(self._timeouts)
                self._run_callback(timeout.callback)
            else:
                seconds = self._timeouts[0].deadline - now
                poll_timeout = min(seconds, poll_timeout)
                break

    if self._callbacks:
        # If any callbacks or timeouts called add_callback,
        # we don't want to wait in poll() before we run them.
        poll_timeout = 0.0

    if not self._running:
        break

    if self._blocking_signal_threshold is not None:
        # clear alarm so it doesn't fire while poll is waiting for events.
        signal.setitimer(signal.ITIMER_REAL, 0, 0)

    # epoll阻塞，当有事件通知或超时返回event_pairs
    try:
        event_pairs = self._impl.poll(poll_timeout)
    except Exception, e:
        # 异常处理，省略

    # 对epoll返回event_pairs事件的处理
    self._events.update(event_pairs)
    while self._events:
        fd, events = self._events.popitem()
        try:
            self._handlers[fd](fd, events)
        except Exception e:
            # 异常处理，省略

```

#### 3.0后的一些改动

Tornado3.0以后 [IOLoop](http://www.nowamagic.net/academy/tag/IOLoop) 模块的一些改动。

IOLoop 成为 util.Configurable 的子类，IOLoop 中绝大多数成员方法都作为抽象接口，具体实现由派生类 PollIOLoop 完成。IOLoop 实现了 Configurable 中的 configurable\_base 和 configurable\_default 这两个抽象接口，用于初始化过程中获取类类型和类的实现方法（即 IOLoop 中 poller 的实现方式）。

在 Tornado3.0+ 中针对不同平台，单独出 poller 相应的实现，EPollIOLoop、KQueueIOLoop、SelectIOLoop 均继承于 PollIOLoop。下边的代码是 configurable\_default 方法根据平台选择相应的 epoll 实现。初始化 IOLoop 的过程中会自动根据平台选择合适的 poller 的实现方法。

```
@classmethod
def configurable_default(cls):
	if hasattr(select, "epoll"):
		from tornado.platform.epoll import EPollIOLoop
		return EPollIOLoop
	if hasattr(select, "kqueue"):
		# Python 2.6+ on BSD or Mac
		from tornado.platform.kqueue import KQueueIOLoop
		return KQueueIOLoop
	from tornado.platform.select import SelectIOLoop
	return SelectIOLoop

```



