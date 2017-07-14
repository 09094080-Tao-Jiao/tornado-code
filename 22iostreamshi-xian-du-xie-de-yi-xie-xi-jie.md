接下来还是阅读 [IOStream](http://www.nowamagic.net/academy/tag/IOStream) 这一块。看到这名字，我就想到了 C++ 的 iostream，也许 facebook 有意为之？对于 IOStream，整体的认识就是，它负责IO读写，顺便回调。

先认识一个工具函数：\_merge\_prefix。它的作用是将双端队列（deque）的首项调整为指定大小，如果明白双端队列的popleft和appendleft方法，这个函数还是很容易看懂的，我略过对它的分析。接下来是非阻塞读写基类BaseIOStream。首先是\_\_init\_\_方法，记录了ioloop（毕竟要实现异步非阻塞，ioloop是必须的），然后初始化了两个缓冲区双端队列和缓冲区大小：

```
self.max_buffer_size = max_buffer_size
self.read_chunk_size = read_chunk_size
self.error = None
self._read_buffer = collections.deque()
self._write_buffer = collections.deque()

```

以及将其它标志置为默认none。很明显，io的最底层就是缓冲区了，先来看关于读缓冲区的两个方法: \_read\_to\_buffer 和 \_read\_from\_buffer 外加一个\_consume方法。先看第一个函数：

```
def _read_to_buffer(self):
	"""Reads from the socket and appends the result to the read buffer.

	Returns the number of bytes read.  Returns 0 if there is nothing
	to read (i.e. the read returns EWOULDBLOCK or equivalent).  On
	error closes the socket and raises an exception.
	"""
	try:
		chunk = self.read_from_fd()
	except (socket.error, IOError, OSError) as e:
		# ssl.SSLError is a subclass of socket.error
		if e.args[0] == errno.ECONNRESET:
			# Treat ECONNRESET as a connection close rather than
			# an error to minimize log spam  (the exception will
			# be available on self.error for apps that care).
			self.close(exc_info=True)
			return
		self.close(exc_info=True)
		raise
	if chunk is None:
		return 0
	self._read_buffer.append(chunk)
	self._read_buffer_size += len(chunk)
	if self._read_buffer_size 
>
= self.max_buffer_size:
		gen_log.error("Reached maximum read buffer size")
		self.close()
		raise IOError("Reached maximum read buffer size")
	return len(chunk)

```

首先是调用read\_from\_fd函数\(由子类覆盖重写，简单的认为就是fd .read\(\)）得到chunk。一般fd可读时操作系统缓冲区里都会有一定长度的chunk，所以一般总是能得到某个chunk（但不一定是符合预期的chunk，比如我希望将所有的内容读完直到结束，但系统缓冲区里不一定就放的下。。。）。得到chunk后，把它放到自己的缓冲区里（这样操作系统的缓冲区就可以复用为新内容服务）并增加buffesize，检查是否超过了缓冲区最大允许容量，最后返回chunk的大小。

接下来是 \_consume，它用作从自身缓冲区中取出指定长度的内容。代码就不贴了，流程很简单，先\_merge\_prefix使缓冲区首项符合指定大小，再popleft弹出首项并调整buffersize即可。然后是read\_from\_buffer，这个函数比较重要了，因为iostream需要支持很多种读的方式，例如rea\_until,read\_bytes,read\_regex等，这些模式和对应的callback都是在这个函数里被实现和调用的：

```
def _read_from_buffer(self):
	"""Attempts to complete the currently-pending read from the buffer.

	Returns True if the read was completed.
	"""
	if self._streaming_callback is not None and self._read_buffer_size:
		bytes_to_consume = self._read_buffer_size
		if self._read_bytes is not None:
			bytes_to_consume = min(self._read_bytes, bytes_to_consume)
			self._read_bytes -= bytes_to_consume
		self._run_callback(self._streaming_callback,
						   self._consume(bytes_to_consume))
	if self._read_bytes is not None and self._read_buffer_size 
>
= self._read_bytes:
		num_bytes = self._read_bytes
		callback = self._read_callback
		self._read_callback = None
		self._streaming_callback = None
		self._read_bytes = None
		self._run_callback(callback, self._consume(num_bytes))
		return True
	elif self._read_delimiter is not None:
		# Multi-byte delimiters (e.g. '\r\n') may straddle two
		# chunks in the read buffer, so we can't easily find them
		# without collapsing the buffer.  However, since protocols
		# using delimited reads (as opposed to reads of a known
		# length) tend to be "line" oriented, the delimiter is likely
		# to be in the first few chunks.  Merge the buffer gradually
		# since large merges are relatively expensive and get undone in
		# consume().
		if self._read_buffer:
			while True:
				loc = self._read_buffer[0].find(self._read_delimiter)
				if loc != -1:
					callback = self._read_callback
					delimiter_len = len(self._read_delimiter)
					self._read_callback = None
					self._streaming_callback = None
					self._read_delimiter = None
					self._run_callback(callback,
									   self._consume(loc + delimiter_len))
					return True
				if len(self._read_buffer) == 1:
					break
				_double_prefix(self._read_buffer)
	elif self._read_regex is not None:
		if self._read_buffer:
			while True:
				m = self._read_regex.search(self._read_buffer[0])
				if m is not None:
					callback = self._read_callback
					self._read_callback = None
					self._streaming_callback = None
					self._read_regex = None
					self._run_callback(callback, self._consume(m.end()))
					return True
				if len(self._read_buffer) == 1:
					break
				_double_prefix(self._read_buffer)
	return False

```

首先是检查\_streaming\_callback回调（字符流回调，一般是读操作没有彻底读够而处于streaming状态，一般默认是None，如果调用read\_bytes和read\_until\_close并指定了streaming\_callback参数就会造成这个回调）和buffersize。如果read\_bytes没有被设定则说明调用的是read\_until\_close，则直接把buffer里所有内容读出并调用回调，否则的话根据read\_bytes和buffersize决定要读取的大小，其它步骤同read\_until\_close。这只是开胃菜，然后开始判断各个标志来决定回调。不同的标志在各自读函数里分别设置，在此就不一一赘述了。简言之就是根据不同的标志采取不同的条件判断，如果判断成功就回调。

和ioloop异步有关的是两个函数：\_add\_io\_state 和 \_handle\_events。第一个函数就比较简单了，主要是更新自身的状态并告诉ioloop监听新事件。另一个函数主要是负责事件的分发，将发生的事件和和read/write进行比对并调用响应的回调，同时检查自身状态机的状态（是否在读，是否在写，是否已经关闭等）向ioloop注册新的回调\(这个函数有点像memcached里的超级状态机drive\_machine，不过很明显tornado比它简单多了（tornado的状态机挺弱的，我觉得）。相应的handle\_read和handle\_write被调用，handle\_read就是调用了read\_to\_buffer把内容复制进读缓冲区并调用read\_from\_buffer在条件被满足时执行回调，handle\_write就调用write\_to\_fd 把写缓冲区中的内容移除。

看了半天，终于可以看到iostream对外提供的接口了。以read\_bytes为例，它首先设置了回调及读取内容的大小，接着调用\_try\_inline\_read做结。而的\_try\_inline\_read代码如下：

```
def _try_inline_read(self):
	"""Attempt to complete the current read operation from buffered data.

	If the read can be completed without blocking, schedules the
	read callback on the next IOLoop iteration; otherwise starts
	listening for reads on the socket.
	"""
	# See if we've already got the data from a previous read
	if self._read_from_buffer():
		return
	self._check_closed()
	try:
		# See comments in _handle_read about incrementing _pending_callbacks
		self._pending_callbacks += 1
		while not self.closed():
			if self._read_to_buffer() == 0:
				break
	finally:
		self._pending_callbacks -= 1
	if self._read_from_buffer():
		return
	self._maybe_add_error_listener()

```

首先尝试从自身缓冲区读取，如果失败则反复调用直到close或者缓冲区里没有东西可读（由于设置了fd为非阻塞模式，read不会被阻塞而是返回0），在此尝试从自身缓冲区读取，还是没达到要求的话就调用\_maybe\_add\_error\_listener，其实就是开始监听read事件。中间还有个 \_pending\_callbacks 信号量，作用稍候再说。综上述，当上层调用iostream的read\_\*方法时，它首先设置回调，然后调用\_try\_inline\_read进行非阻塞式读取，能一次性读到满足条件最好，不行就监听read。当read事件发生时再调用handle\_read会做好善后处理（在上一段）。整个过程不会被阻塞，就是回调里跳来跳去的可能会花点时间（和网络延迟比起来简直不值一提）。同理，write函数也是这样，把内容放进自身缓冲区，当write事件到来时再输出，省去了网络延迟。

IOStream相对就比较简单了，主要是实现了[socket](http://www.nowamagic.net/academy/tag/socket)的读写，叫它SocketStream也许更合适？ 最后是关于\_pending\_callbacks 信号量。首先它总是成对出现，有增就有减，并且总是先增后减。另外，凡是在它减一后总是会执行 \_maybe\_run\_close\_callback 或者 \_maybe\_add\_error\_listener。是的，没错，\_pending\_callbacks只对这两个函数起作用。根据注释的说法，信号量的作用是为了防止读缓冲区中部出现的‘’空字符串导致被误认为close，为了防止所有的回调都不会因为空字符串而被close所中断，就使用信号量告诉系统现在暂时不要close。而且由于信号量增减后总是会调用两个函数之一，因此close回调总是会被调用而不会因为信号量而没有得到正确执行）。

以上，就是IO层的代码执行。整个思路大概明晰了许多吧。

