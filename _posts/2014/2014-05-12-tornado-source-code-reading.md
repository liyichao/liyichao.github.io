---
layout: post
title: "tornado 3.2"
category: 
tags: []
---
{% include JB/setup %}

netty整个程序完全由类组成，一个程序就是许多类组合起来，类的继承层次较多，并且因为有Interface，所以不像多重继承那么难以理解。一个类就是一个概念，只要理解这个概念，就理解了这个类的层次。因为是概念，以后回忆起来也相对容易，所以不会说看了这个源代码，前面看的就忘记了。理解tornado也可以按这个思路走。

netty和tornado有一个不同就是，netty一个文件只有一个类（这个类可能有内嵌类），但是tornado可能有多个，像ioloop.py就有`IOLoop`、`PollIOLoop`、`_Timeout`、`PeriodicCallback`这4个类。当然，把相关类集中到一个文件是很好的，不然很难看出类之间的关联。

相比于C项目nginx，netty和tornado更关注的是实现的简洁和概念的清晰性，而对内存和效率，则没有nginx那么注重。

####ioloop.py
基于select的事件循环的设计思路：调用poll等待IO事件发生，poll返回时处理每个IO事件，然后继续等待。可以加入时间事件，这样就调用带超时的poll，等待直到时间事件到期，另外也可以支持callback，再select前先把callback和时间事件给处理完毕，再调用poll，select模块的具体实现可以是epoll或者是kqueue，需要支持poll，register，modify等操作。

不得不感叹，python这个高级语言就是高级，其`select`模块封装了`select`、`epoll`、`kqueue`，面向对象的API，用起来就是比较方便。

IOLoop用的是水平触发。

`connection_ready(sock, fd, events)`：

*  都有socket了，还要fd干嘛？fd和events是ioloop调用callback时默认传的参数。而sock，通过`callback = functools.partial(connection_ready, sock)`实现跟callback的绑定。
*  accept是在循环里的调用的，意味着一次监听套接字可读，就把所有连接都接收进来，这样减少了客户的等待时间。

对于多种状态是用一位还是用一个值表示？如果可能同时处于多种状态，位表示比较合适。

`instance()`返回的是全局单例，表示主线程的IOLoop。

`IOLoop()`返回的对象可能是`EpollIOLoop`、`KQueueIOLoop`、`SelectIOLoop`，因为IOLoop继承了Configurable，而Configurable重定义了`__new__()`。

可以这样实现每个线程一个IOLoop：

        ioloop = IOLoop()
        ioloop.make_current()

`add_handler(self, fd, handler, events)`：看注释可知，当events发生时，调用`handler(fd, events)`，这意味着，handler必须处理读事件，写事件，错误时间，可以想象，其代码类似：
        
        if (events & ioloop.READ) {
            ...
        }
        if (events & ioloop.WRITE) {
            ...
        }
        if (events & ioloop.ERROR) {
            ...
        }
        
fd的写事件函数和读事件函数一般来说是固定不变的，因此不会有单独改变写事件的需求，需求只是对监听事件的变更，如取消对写事件的关注，所以IOLoop提供了`update_handler(self, fd, events)`。事件循环的接口设计，和redis几乎完全一样。

`add_future()`：`future.add_done_callback(lambda future: self.add_callback(callback, future))`，为什么一定要在IOLoop里执行callback？

`PollIOLoop.close()`里：
    
    with self._callback_lock:
        self._closing = True
这个lock略奇怪，不是用在add_callback里的么？

`PollIOLoop.stop()`是可以在IOLoop以外的线程调用的，callback一般是在IOLoop线程调用，而stop可能在callback里调用，所以stop可能在IOLoop所在的线程调用，一般也不会去关闭eventloop吧，只有用户要结束程序的时候才会调用stop，所以理论上来说应该在信号处理函数里调用stop，这样的话select就可能被中断了，如果select会被重启的话，还是需要wake()的。

用heapq实现超时。

`start()`是事件循环实现的地方：

* `signal.set_wakeup_fd`真是神函数，把信号转化为套接字可读事件。
* 这个循环处理三种事件：IO事件，时间事件，callback。
* 先执行callback，再执行时间事件，如果在时间事件或callback中又添加了新的callback，那么select不等待，否则等待直到最近一次时间事件到期。
* 在select前先判断IOLoop是否已经被停止了，这样使得IOLoop能正常停止。
* `set_blocking_signal_threshold`这个blocking不是指select的等待时间，在select时计时器是关闭的，在select返回后才打开，所以这个blocking是指事件的处理时间。
* 一次select后调用`self._events.update(event_pairs)`，在下一次select前也没有清除`self._events`，这正确吗？这理解不对，因为`fd, events = self._events.popitem()`，处理的时候已经pop出来了，所以事实上在下一次select前已经清除了`self._events`。

####iostream.py
这个文件比较难以理解的就是`_pending_callbacks`、`_maybe_add_error_listener`、`_maybe_run_close_callback`、`with stack_context.NullContext():`。

iostream其实是一个传输层。和stp、redis等协议有点像。它提供`read_until_regex`、`read_until(delimiter)`、`read_bytes(num_bytes)`、`read_until_close()`，所有都接收一个callback参数，因为操作都是异步的。用这些函数，可以实现各种传输层协议。怪不得netty把transport专门列为一个包，其实传输层挺重要的。

各种read_函数，区别是read完成的标志不同。如果read完成了，callback会在下一个IOLoop循环中调用，如果没有完成，那么会继续监听套接字的读事件。回调函数分为callback和streaming_callback。callback是读完完整的数据才调用，如正则表达式匹配成功，遇到分界符，读取了指定的字节，套接字关闭。而streaming_callback是每次调用read从套接字读到了数据，就调用。各read函数的实现比较一致，都是先处理buffer里的数据，如果没有数据不完整，就从套接字一直读到EAGAIN，数据读到buffer后再处理，如果依然不完整，那么就注册套接字读事件了，在读事件里，先从套接字读，然后再调用`_read_from_buffer()`处理。`_read_from_buffer()`应该叫`_process_buffer()`。

各read_*函数对应的数据结构：

* `read_until_regex()` -> `_read_regex`
* `read_until(delimiter)` -> `_read_delimiter`
* `read_bytes(bytes)` -> `_read_bytes`


`read_until_regex`：由于TCP为字节流，一次read的内容可能不能匹配regex，多次read的内容合起来才匹配regex，所以在匹配的时候，需要对整个buffer的内容匹配，而不能只对当前read读到的内容匹配，所以在`_read_from_buffer()`里有`_double_prefix(self._read_buffer)`，就是把最前两项合并成一个字符串，然后用regex去匹配，不行，就再把前两项合并，一直到全都合成一项或者匹配成功为止。

    def _maybe_add_error_listener(self):
        if self._state is None and self._pending_callbacks == 0:
            if self.closed():
                self._maybe_run_close_callback()
            else:
                self._add_io_state(ioloop.IOLoop.READ)
第一个if怎么理解？


`_read_to_buffer()`：`_read_buffer`是collections.deque类型的，用来存从套接字读到的数据，`_read_buffer_size`记录缓存的数据的字节数。这个和twemproxy里mbuf的设计是类似的，用list实现缓冲区。不过mbuf是固定大小的chunk连成list，而`_read_buffer`容器里每个chunk的大小是由`os.read()`决定的，因此没有chunk   cache，而mbuf的实现中，在释放mbuf时，是把每个chunk放到一个freelist，而不是free掉。

`_try_inline_read()`：一直读套接字直到它不可读，这样，万一一个套接字比较快，其他套接字就容易饥饿。而redis，一次读事件只读一次，限定最大读字节数，使得对每个套接字公平。不过，在`_read_to_buffer()`里有对缓冲区大小的限制，一旦超过，这个套接字就被关闭了。

`_run_callback(self, callback, *args)`里

    with stack_context.NullContext():
            self._pending_callbacks += 1
            self.io_loop.add_callback(wrapper)
with那行有什么用？

`_read_buffer_size`是已经读到buffer里的字节数，而`_read_bytes`是希望读的字节数，如果buffer没有那么多，则等待。`_read_bytes`可以看做是redis协议里的bulk length，先传个数据长度len，接着传数据。

`_streaming_callback`是每接到一些数据就调用，可以看做是byte handler，而`_read_callback`则是每接到一个bulk调用一次，可以看做是message handler。

`_consume(num_bytes)`把buffer里前num_bytes的数据返回，并从buffer里踢出。返回的数据用于处理。

`_read_from_buffer`里

    if self._read_bytes is not None and self._read_buffer_size >= self._read_bytes:
            num_bytes = self._read_bytes
            callback = self._read_callback
            self._read_callback = None
            self._streaming_callback = None
            self._read_bytes = None
            self._run_callback(callback, self._consume(num_bytes))
把callback全清为None是为什么？这是因为每次读的时候都会重新设置callback，如

    def read_until_regex(self, regex, callback):
        self._set_read_callback(callback)

callback充当着协议的作用，_read_regex非None表示要一直read知道碰到与regex匹配的字符串。这不太合适吧，把callback和协议混在一起。

用deque做为read buffer的数据结构，每一次read就存为队列中的其中一项，而不是append到一个string里，当边界跨越几项时，就合并前面的。这一数据结构合适吗？

`BaseIOStream._state`，是IOLOOP.ERROR，IOLOOP.READ，IOLOOP.WRITE，这个应该属于事件模块，怎么跑到iostream来了。

    def read_bytes(self, num_bytes, callback, streaming_callback=None):
        self._set_read_callback(callback)
        assert isinstance(num_bytes, numbers.Integral)
        self._read_bytes = num_bytes
        self._streaming_callback = stack_context.wrap(streaming_callback)
        self._try_inline_read()
为什么要`stack_context.wrap`这个`streaming_callback`。

`read_until_close`无法工作吧？一旦套接字不可读了，函数就终止了。不不不，会一直读的，因为没有取消读事件。

`write`：把data切成了128KB的块，append到`_write_buffer`(collections.deque类型)。在写到套接字的时候，也是一次写128KB，直到`_write_buffer`为空，或者套接字不可写。这里也没用writev。如果一次写，没写到128KB，这个块还得分裂，没写完的重新放回`_write_buffer`去。没有必要嘛

`IOStream.connect()`的实现是非阻塞connect的实现，可以借鉴下。

`SSLIOStream`里：

    def connect(self, address, callback=None, server_hostname=None):
        # Save the user's callback and run it after the ssl handshake
        # has completed.
        self._ssl_connect_callback = stack_context.wrap(callback)
        self._server_hostname = server_hostname
        super(SSLIOStream, self).connect(address, callback=None)   
`_ssl_connect_callback`在`_do_ssl_handshake`被调用。而`_do_ssl_handshake`只有在`_ssl_accepting`为真时才调用，所以，客户端程序里，`_ssl_accepting`也为真。在`do_handshake()`成功后，才置`_ssl_accepting`为假。

####web.py
`RequestHandler`里支持的方法有7种，有PATCH没有CONNECT。

`RequestHandler`构造函数里有request，每个request都生成一个新的handler不浪费么？

`RequestHandler.ui`是干嘛用的？

为什么加了`@gen.coroutine`就成异步的了？

`RequestHandler.render()`里modules是指什么？包含UIModule和TemplateModule。

`RequestHandler.flush()`里

* `chunk = b"".join(self._write_buffer)`。这倒是方便，都不用writev了，只是join的效率怎么样？
* `self.request.write(headers + chunk, callback=callback)`，这里的write不用管具体write了多少字节，这是传输层的事情，也就是IOStream的事情。

`Application`：

         if settings.get("gzip"):
             self.transforms.append(GZipContentEncoding)
         self.transforms.append(ChunkedTransferEncoding)
注意chunked是在gzip后面的。

####tcperver.py
往IOLoop注册listen套接字，在处理函数里，accept直到EAGAIN，对于每一个accept到的connection，生成一个IOStream，然后调用handle_stream。在handle_stream里，调用各种read_*函数，如，在httpserver，调用的就是
    
    self.stream.read_until(b"\r\n\r\n", self._header_callback)

####httpserver.py
`Transfer-Encoding: chunked`只有响应头才有，请求头只能用`Content-Length`。

`HTTPRequest`有`write()`方法真是让人惊奇。这是为了写RequestHandler方便。更名为`reply()`好一些。

`HTTPConnection._finish_request()`：

     self.stream.set_nodelay(False)
开启了Nagle算法。在`finish()`里关闭了Nagle算法，相当于是一个flush操作。
     
####escape.py

    def parse_qs_bytes(qs, keep_blank_values=False, strict_parsing=False):
        result = _parse_qs(qs, keep_blank_values, strict_parsing,
                           encoding='latin1', errors='strict')
        encoded = {}
        for k, v in result.items():
            encoded[k] = [i.encode('latin1') for i in v]
        return encoded
'latin1'？万一query string里有中文怎么办？可是`qs`不是unicode类型的么，为什么还需要encoding参数？qs总是Unicode类型的，因为是url百分号编码后的，用parse_qs后，百分号解码后，可能会出现：
    
    {'key': ['\xd0\xb7\xd0\xbd\xd0\xb0\xd1\x87\xd0\xb5\xd0\xbd\xd0\xb8\xd0\xb5']}   

`key`对应的值是一个utf-8编码的bytes，parse_qs会用encoding指定的编码去解码它。所以，用'latin1'的确会有乱码问题。但是，请注意这个函数下面的部分，它又用'latin1'编码回去了，所以得到的字典，key是unicode类型，值是bytes类型。

####process.py
`fork_processes(num_processes, max_restarts=100)`并没有像nginx那样有`ngx_master_process_cycle`，`ngx_worker_process_cycle`，而是

    for i in range(num_processes):
        id = start_child(i)
        if id is not None:
            return id
    num_restarts = 0
    while children:
    try:
            pid, status = os.wait()
    ...       
    sys.exit(0)
子进程返回id，继续运行，而父进程wait所有子进程，然后退出，这样，就和工作进程的程序流，就和单进程是一样的。listen socket要提前生成，然后由子进程继承，否则子进程是无法监听同一端口的。

tornado没有解决惊群问题。同一端口的listen套接字，注册到了所有子进程的IOLoop，这样，有连接到来时，所有IOLoop都会被惊醒。

####gen.py
`coroutine`里func是同步调用的吗？是。coroutine是用来return值的。供别的函数yield from coroutine的。coroutine是一种generator，所以它必须yield语句。

####stack_context.py

####util.py
`import_object`有什么用，直接用import不就行了？它支持字符串，即可以在程序运行时动态导入，而import是写死的。

为何不直接定义`exec_in`而要放在exec("def exec_in...")里面？

`Configurable`的作用就是它的构造函数是子类的工厂函数，并且生成的子类可以用`configure`在运行时换掉。`IOLoop`也是`Configurable`的子类，为的就是支持`epool`，`kqueue`，`select`。

####template.py
`_CodeWriter.include`函数里`__enter__(_)`，_是什么参数？

`Template.generate`最后调用`execute()`，它的输入怎么得到？这是一个函数闭包。

####httpclient.py
`AsyncHTTPClient.fetch`里，`callback = stack_context.wrap(callback)`，说明callback总是被stack_context给wrap的，有什么作用？

####platform/posix.py
`set_close_exec`居然也设为一个函数，就两条语句`fcntl(F_GETFD)+fcntl(F_SETFD)`，像这种函数，也挺难说它属于哪个模块，可能就归为utilities。`_set_nonblocking`也是用两个系统调用，nginx好像只用了一个`ioctl`就实现了。

`Waker.__init__()`里`os.fdopen(r,"rb",0)`用0指定无缓冲是必须的，否则`wake`里就必须调用flush。可能调用多次`wake`，在`consume`里一次性把之前`wake`往套接字写的内容都消耗掉了。

####platform/interface.py
这文件命名为`interface.py`真是搞笑，哪有文件命名为interface的。由`Waker`可以看出，类有时是很方便的，把fd和方法定义为一个类，概念上清晰，用起来也方便。