ioloop中`add_handler`只能增加一个handler，无法为读事件和写事件分别建立事件。并且当fd已有handler时无法更新handler，只能更新事件，这又有什么用呢？

用heapq实现超时。

######iostream.py
`_run_callback(self, callback, *args)`里

    with stack_context.NullContext():
            self._pending_callbacks += 1
            self.io_loop.add_callback(wrapper)
with那行有什么用？

`_read_buffer_size`是已经读到buffer里的字节数，而`_read_bytes`是希望读的字节数，如果buffer没有那么多，则等待。`_read_bytes`可以看做是redis协议里的bulk length，先传个数据长度len，接着传数据。

`_streaming_callback`是每接到一些数据就调用，可以看做是byte handler，而`_read_callback`则是每接到一个bulk调用一次，可以看做是message handler。

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
        self._read_regex = re.compile(regex)
        self._try_inline_read()
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

#######process.py
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

tornado没有解决惊群问题。同一端口的套接字，注册到了所有子进程的IOLoop，这样，有连接到来时，所有IOLoop都会被惊醒。

######httpserver.py
`Transfer-Encoding: chunked`只有响应头才有，请求头只能用`Content-Length`。