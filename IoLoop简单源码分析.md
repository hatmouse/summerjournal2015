# IoLoop简单源码分析
#### 1. `tornado.ioloop.IOLoop.instance()`返回全局唯一单例
返回全局唯一单例，调用IOLoop()实例化
```
# 返回全局唯一单例
@staticmethod
def instance():
    # 判断是否存在全局实例`_instance`否则创建
    if not hasattr(IOLoop, "_instance"):
        with IOLoop._instance_lock:
            if not hasattr(IOLoop, "_instance"):
                # New instance after double check
                IOLoop._instance = IOLoop()
    return IOLoop._instance
```

注意：`with some_lock`等价与
```
some_lock.acquire()
try:
    # do something...
finally:
    some_lock.release()
```

#### 2. ` IOLoop()`进行实例化
调用`Configurable.__new__()`方法，最终返回EPollIOLoop，即io=EPollIOLoop
```
class Configurable(object):
    # 配置的子类必须实现configurable_default和configurable_base方法
    __impl_class = None
    __impl_kwargs = None
    # 正在实例话的类，这里指IOLoop
    def __new__(cls, *args, **kwargs):
        # base=IOLoop
        base = cls.configurable_base()
        init_kwargs = {}
        if cls is base:
            # 重要！，是否配置，否则调用子类实现的configurable_default
            # 根据不同平台返回，linux返回EPollIOLoop
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        # 实例化EPollIOLoop
        # 继承PollIOLoop
        instance = super(Configurable, cls).__new__(impl)
        # 初始化PollIOLoop，epoll.py
        instance.initialize(*args, **init_kwargs)
        return instance
    @classmethod
    def configured_class(cls):
        # 返回默认，若没有则调用configurable_default
        base = cls.configurable_base()
        if cls.__impl_class is None:
            base.__impl_class = cls.configurable_default()
        return base.__impl_class
```
#### 3. 初始化EPollIOLoop、PollIOLoop、IOLoop
```
class EPollIOLoop(PollIOLoop):
    def initialize(self, **kwargs):
        super(EPollIOLoop, self).initialize(impl=select.epoll(), **kwargs)

class PollIOLoop(IOLoop):
    def initialize(self, impl, time_func=None, **kwargs):
        super(PollIOLoop, self).initialize(**kwargs)
        self._impl = impl
        if hasattr(self._impl, 'fileno'):
            set_close_exec(self._impl.fileno())
        self.time_func = time_func or time.time
        self._handlers = {}
        self._events = {}
        self._callbacks = []
        self._callback_lock = threading.Lock()
        self._timeouts = []
        self._cancellations = 0
        self._running = False
        self._stopped = False
        self._closing = False
        self._thread_ident = None
        self._blocking_signal_threshold = None
        self._timeout_counter = itertools.count()
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
                         
```



