# `AsyncHTTPClient`简单的源码分析
* 异步函数，`ioloop`驱动配合callback 或 coroutine(yield)
* 通过`Configurable`实现子类配置

```
import ioloop

def handle_request(response):
    if response.error:
        print "Error:", response.error
    else:
        print response.body
    ioloop.IOLoop.instance().stop()

http_client = httpclient.AsyncHTTPClient()
http_client.fetch("http://www.google.com/", handle_request)
ioloop.IOLoop.instance().start()
```

#### 1.`__new__`初始实例化，调用父类实例化`Configurable`
```
# httpclient.py
class AsyncHTTPClient(Configurable):
    def __new__(cls, io_loop=None, force_instance=False, **kwargs):
        io_loop = io_loop or IOLoop.current()
        if force_instance:
            instance_cache = None
        else:
            instance_cache = cls._async_clients()
        if instance_cache is not None and io_loop in instance_cache:
            return instance_cache[io_loop]
        # 调用父类Configurable的实例化
        instance = super(AsyncHTTPClient, cls).__new__(cls, io_loop=io_loop,
                                                       **kwargs)
        instance._instance_cache = instance_cache
        if instance_cache is not None:
            instance_cache[instance.io_loop] = instance
        return instance
```

#### 2.`Configurable`调用`__new__`

```
class Configurable(object):
    __impl_class = None
    __impl_kwargs = None

    def __new__(cls, *args, **kwargs):
        # cls为正在实例化的类，即AsyncHTTPClient
        base = cls.configurable_base()
        init_kwargs = {}
        if cls is base:
            # 进行默认初始化，调用configurable_default，在子类实现
            # impl即SimpleAsyncHTTPClient
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        instance = super(Configurable, cls).__new__(impl)
        # 调用配置实现类SimpleAsyncHTTPClient的初始化
        instance.initialize(*args, **init_kwargs)
        return instance
```
```
# utils.py
# 返回配置配置类
def configurable_default(cls):
    from tornado.simple_httpclient import SimpleAsyncHTTPClient
    return SimpleAsyncHTTPClient
```
#### 3.初始化`SimpleAsyncHTTPClient`，同时初始化`AsyncHTTPClient`
```
# simple_httpclient.py
# 父类为AsyncHTTPClient
class SimpleAsyncHTTPClient(AsyncHTTPClient):
    def initialize(self, io_loop, max_clients=10,
                   hostname_mapping=None, max_buffer_size=104857600,
                   resolver=None, defaults=None, max_header_size=None,
                   max_body_size=None):
        # 调用父类AsyncHTTPClient初始化
        super(SimpleAsyncHTTPClient, self).initialize(io_loop,
                                                      defaults=defaults)
        self.max_clients = max_clients
        self.queue = collections.deque()
        self.active = {}
        self.waiting = {}
        self.max_buffer_size = max_buffer_size
        self.max_header_size = max_header_size
        self.max_body_size = max_body_size
        if resolver:
            self.resolver = resolver
            self.own_resolver = False
        else:
            self.resolver = Resolver(io_loop=io_loop)
            self.own_resolver = True
        if hostname_mapping is not None:
            self.resolver = OverrideResolver(resolver=self.resolver,
                                             mapping=hostname_mapping)
        self.tcp_client = TCPClient(resolver=self.resolver, io_loop=io_loop)
```
#### 4.`fetch()`，调用子类实现的`fetch_impl()`，最后返回`future`
```
# httpclient.py
def fetch(self, request, callback=None, raise_error=True, **kwargs):
    if self._closed:
        raise RuntimeError("fetch() called on closed AsyncHTTPClient")
    if not isinstance(request, HTTPRequest):
        request = HTTPRequest(url=request, **kwargs)
    request.headers = httputil.HTTPHeaders(request.headers)
    request = _RequestProxy(request, self.defaults)
    future = TracebackFuture()
    if callback is not None:
        callback = stack_context.wrap(callback)
        def handle_future(future):
            exc = future.exception()
            if isinstance(exc, HTTPError) and exc.response is not None:
                response = exc.response
            elif exc is not None:
                response = HTTPResponse(
                    request, 599, error=exc,
                    request_time=time.time() - request.start_time)
            else:
                response = future.result()
                ＃　注册真正的对结果处理的回调函数，下次轮询调用
            self.io_loop.add_callback(callback, response)
        # 添加future的_callbacks，由set_result()调用
        future.add_done_callback(handle_future)

    def handle_response(response):
        if raise_error and response.error:
            future.set_exception(response.error)
        else:
            # 调用handle_future()
            future.set_result(response)
    # fetch_impl具体实现，当有数据可以读取了或出现错误，回调handle_response()
    self.fetch_impl(request, handle_response)
    return future
```
5.`fetch_impl`添加任务到队列
```
# simple_httpclient.py
def fetch_impl(self, request, callback):
    key = object()
    self.queue.append((key, request, callback))
    if not len(self.active) < self.max_clients:
        # 添加超时处理事件
        timeout_handle = self.io_loop.add_timeout(
            self.io_loop.time() + min(request.connect_timeout,
                                      request.request_timeout),
            functools.partial(self._on_timeout, key))
    else:
        timeout_handle = None
    self.waiting[key] = (request, callback, timeout_handle)
    # 处理duixl
    self._process_queue()
    if self.queue:
        gen_log.debug("max_clients limit reached, request queued. "
                      "%d active, %d queued requests." % (
                          len(self.active), len(self.queue)))
```
