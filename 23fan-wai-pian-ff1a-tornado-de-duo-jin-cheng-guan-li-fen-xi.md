[Tornado](http://www.nowamagic.net/academy/tag/Tornado)的多进程管理我们可以参看process.py这个文件。

在编写[多进程](http://www.nowamagic.net/academy/tag/%E5%A4%9A%E8%BF%9B%E7%A8%8B)的时候我们一般都用python自带的multiprocessing，使用方法和threading基本一致，只需要继承里面的Process类以后就可以编写多进程程序了，这次我们看看tornado是如何实现他的multiprocessing，可以说实现的功能不多，但是更加简单高效。

我们只看fork\_process里面的代码：

```
global _task_id
    assert _task_id is None
    if num_processes is None or num_processes 
<
= 0:
        num_processes = cpu_count()
    if ioloop.IOLoop.initialized():
        raise RuntimeError("Cannot run in multiple processes: IOLoop instance "
                           "has already been initialized. You cannot call "
                           "IOLoop.instance() before calling start_processes()")
    logging.info("Starting %d processes", num_processes)
    children = {}

```

这一段很简单，就是在没有传入进程数的时候使用默认的cpu个数作为将要生成的进程个数。

```
def start_child(i):
	pid = os.fork()
	if pid == 0:
		# child process
		_reseed_random()
		global _task_id
		_task_id = i
		return i
	else:
		children[pid] = i
		return None

```

这是一个内函数，作用就是生成子进程。fork是个很有意思的方法，他会同时返回两种状态，为什么呢？其实fork相当于在原有的一条路（父进程）旁边又修了一条路（子进程）。如果这条路修成功了，那么在原有的路上（父进程）你就看到旁边来了另外一条路（子进程），所以也就是返回新生成的那条路的名字（子进程的pid），但是在另外一条路上（子进程），你看到的是自己本身修建成功了，也就返回自己的状态码（返回结果是0）。

所以if pid==0表示这时候cpu已经切换到子进程了，相当于我们在新生成的这条路上面做事（返回任务id）；else表示又跑到原来的路上做事了，在这里我们记录下新生成的子进程，这时候children\[pid\]=i里面的pid就是新生成的子进程的pid，而 i 就是刚才在子进程里面我们返回的任务id（其实就是用来代码子进程的id号）。

```
for i in range(num_processes):
	id = start_child(i)
	if id is not None:
		return id

```

if id is not None表示如果我们在刚刚生成的那个子进程的上下文里面，那么就什么都不干，直接返回子进程的任务id就好了，啥都别想了，也别再折腾。如果还在父进程的上下文的话那么就继续生成子进程。

```
num_restarts = 0
    while children:
        try:
            pid, status = os.wait()
        except OSError, e:
            if e.errno == errno.EINTR:
                continue
            raise
        if pid not in children:
            continue
        id = children.pop(pid)
        if os.WIFSIGNALED(status):
            logging.warning("child %d (pid %d) killed by signal %d, restarting",
                            id, pid, os.WTERMSIG(status))
        elif os.WEXITSTATUS(status) != 0:
            logging.warning("child %d (pid %d) exited with status %d, restarting",
                            id, pid, os.WEXITSTATUS(status))
        else:
            logging.info("child %d (pid %d) exited normally", id, pid)
            continue
        num_restarts += 1
        if num_restarts 
>
 max_restarts:
            raise RuntimeError("Too many child restarts, giving up")
        new_id = start_child(id)
        if new_id is not None:
            return new_id

```

剩下的这段代码都是在父进程里面做的事情（因为之前在子进程的上下文的时候已经返回了，当然子进程并没有结束）。

pid, status = os.wait\(\)的意思是等待任意子进程退出或者结束，这时候我们就把它从我们的children表里面去除掉，然后通过status判断子进程退出的原因。

如果子进程是因为接收到kill信号或者抛出exception了，那么我们就重新启动一个子进程，用的当然还是刚刚退出的那个子进程的任务号。如果子进程是自己把事情做完了才退出的，那么就算了，等待别的子进程退出吧。

我们看到在重新启动子进程的时候又使用了

```
if new_id is not None:
    return new_id

```

主要就是退出子进程的空间，只在父进程上面做剩下的事情，不然刚才父进程的那些代码在子进程里面也会同样的运行，就会形成无限循环了，我没试过，不如你试试？

