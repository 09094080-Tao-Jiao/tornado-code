IOLoop 是 tornado 的核心。程序中主函数通常调用 tornado.ioloop.IOLoop.instance\(\).start\(\) 来启动IOLoop，但是看了一下 IOLoop 的实现，start 方法是这样的：

    def start(self):
    	"""Starts the I/O loop.

    	The loop will run until one of the callbacks calls `stop()`, which
    	will make the loop stop after the current event iteration completes.
    	"""
    	raise NotImplementedError()


也就是说 [IOLoop](http://www.nowamagic.net/academy/tag/IOLoop) 是个抽象的基类，具体工作是由它的子类负责的。由于是 Linux 平台，所以应该用 Epoll，对应的类是 PollIOLoop。PollIOLoop 的 start 方法开始了事件循环。

问题来了，tornado.ioloop.IOLoop.instance\(\) 是怎么返回 PollIOLoop 实例的呢？刚开始有点想不明白，后来看了一下 IOLoop 的代码就豁然开朗了。

IOLoop 继承自 [Configurable](http://www.nowamagic.net/academy/tag/Configurable)，后者位于 tornado/util.py。

> A configurable interface is an \(abstract\) class whose constructor acts as a factory function for one of its implementation subclasses. The implementation subclass as well as optional keyword arguments to its initializer can be set globally at runtime with configure.

Configurable 类实现了一个工厂方法，也就是设计模式中的“工厂模式”，看一下\_\_new\_\_函数的实现：

```
def __new__(cls, **kwargs):
	base = cls.configurable_base()
	args = {}
	if cls is base:
		impl = cls.configured_class()
		if base.__impl_kwargs:
			args.update(base.__impl_kwargs)
	else:
		impl = cls
	args.update(kwargs)
	instance = super(Configurable, cls).__new__(impl)
	# initialize vs __init__ chosen for compatiblity with AsyncHTTPClient
	# singleton magic.  If we get rid of that we can switch to __init__
	# here too.
	instance.initialize(**args)
	return instance

```

当创建一个Configurable类的实例的时候，其实创建的是configurable\_class\(\)返回的类的实例。

```
@classmethod
def configured_class(cls):
	"""Returns the currently configured class."""
	base = cls.configurable_base()
	if cls.__impl_class is None:
		base.__impl_class = cls.configurable_default()
	return base.__impl_class

```

最后，就是返回的configurable\_default\(\)。此函数在IOLoop中的实现如下：

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

EPollIOLoop 是 PollIOLoop 的子类。至此，这个流程就理清楚了。

