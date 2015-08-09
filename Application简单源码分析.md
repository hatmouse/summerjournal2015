# Application简单源码分析
### 1. 初始化Application
`self.add_handlers(".*$", handlers)`以默认通配符形式初始化handlers，

```
# web.py
class Application(httputil.HTTPServerConnectionDelegate):
    def __init__(self, handlers=None, default_host="", transforms=None,
                 **settings):
        if transforms is None:
            self.transforms = []
            if settings.get("compress_response") or settings.get("gzip"):
                self.transforms.append(GZipContentEncoding)
        else:
            self.transforms = transforms
        self.handlers = []
        self.named_handlers = {}
        self.default_host = default_host
        self.settings = settings
        self.ui_modules = {'linkify': _linkify,
                           'xsrf_form_html': _xsrf_form_html,
                           'Template': TemplateModule,
                           }
        self.ui_methods = {}
        self._load_ui_modules(settings.get("ui_modules", {}))
        self._load_ui_methods(settings.get("ui_methods", {}))
        if self.settings.get("static_path"):
            path = self.settings["static_path"]
            handlers = list(handlers or [])
            static_url_prefix = settings.get("static_url_prefix",
                                             "/static/")
            static_handler_class = settings.get("static_handler_class",
                                                StaticFileHandler)
            static_handler_args = settings.get("static_handler_args", {})
            static_handler_args['path'] = path
            for pattern in [re.escape(static_url_prefix) + r"(.*)",
                            r"/(favicon\.ico)", r"/(robots\.txt)"]:
                handlers.insert(0, (pattern, static_handler_class,
                                    static_handler_args))
        # 不设置虚拟主机地址，默认通配符'.*$'
        if handlers:
            self.add_handlers(".*$", handlers)

        if self.settings.get('debug'):
            self.settings.setdefault('autoreload', True)
            self.settings.setdefault('compiled_template_cache', False)
            self.settings.setdefault('static_hash_cache', False)
            self.settings.setdefault('serve_traceback', True)

        if self.settings.get('autoreload'):
            from tornado import autoreload
            autoreload.start()
```
### 2. `application.listen(8888)`开始监听
```
# web.py
def listen(self, port, address="", **kwargs):
    # 封装httpserver的监听
    from tornado.httpserver import HTTPServer
    server = HTTPServer(self, **kwargs)
    server.listen(port, address)
```
#### `server = HTTPServer(self, **kwargs)`初始化Server
继承`Configurable`，调用`initialize`初始化，然后初始化父类`TCPServer`
```
# httpserver.py
# 建议阅读一下Configurable的代码，才知道是如何实例化HTTPServer
class HTTPServer(TCPServer, Configurable,httputil.HTTPServerConnectionDelegate):
    # 这里request_callback即application，包含url对应的handlers
    def initialize(self, request_callback, no_keep_alive=False, io_loop=None,
                   xheaders=False, ssl_options=None, protocol=None,
                   decompress_request=False,
                   chunk_size=None, max_header_size=None,
                   idle_connection_timeout=None, body_timeout=None,
                   max_body_size=None, max_buffer_size=None):
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.xheaders = xheaders
        self.protocol = protocol
        self.conn_params = HTTP1ConnectionParameters(
            decompress=decompress_request,
            chunk_size=chunk_size,
            max_header_size=max_header_size,
            header_timeout=idle_connection_timeout or 3600,
            max_body_size=max_body_size,
            body_timeout=body_timeout)
        # 初始化父类
        TCPServer.__init__(self, io_loop=io_loop, ssl_options=ssl_options,
                           max_buffer_size=max_buffer_size,
                           read_chunk_size=chunk_size)
        self._connections = set()
        
class TCPServer(object):
    def __init__(self, io_loop=None, ssl_options=None, max_buffer_size=None,
                 read_chunk_size=None):
        self.io_loop = io_loop
        self.ssl_options = ssl_options
        self._sockets = {}  # fd -> socket object
        self._pending_sockets = []
        self._started = False
        self.max_buffer_size = max_buffer_size
        self.read_chunk_size = read_chunk_size
        ...

```
#### `server.listen(port, address)` 开始监听
```
# tcpserver.py
# 为每个网卡创建socket进行监听
def listen(self, port, address=""):
    sockets = bind_sockets(port, address=address)
    # 将sockets加ioloop事件循环
    self.add_sockets(sockets)
    
# tcpserver.py
def add_sockets(self, sockets):
    if self.io_loop is None:
        self.io_loop = IOLoop.current()
    for sock in sockets:
        self._sockets[sock.fileno()] = sock
        # 添加建立建立连接回调函数
        add_accept_handler(sock, self._handle_connection,
                           io_loop=self.io_loop)

# tcpserver.py
# 建立连接后的回调函数
def _handle_connection(self, connection, address):
    if self.ssl_options is not None:
        assert ssl, "Python 2.6+ and OpenSSL required for SSL"
        try:
            connection = ssl_wrap_socket(connection,
                                         self.ssl_options,
                                         server_side=True,
                                         do_handshake_on_connect=False)
        except ssl.SSLError as err:
            if err.args[0] == ssl.SSL_ERROR_EOF:
                return connection.close()
            else:
                raise
        except socket.error as err:
            if errno_from_exception(err) in (errno.ECONNABORTED, errno.EINVAL):
                return connection.close()
            else:
                raise
    try:
        if self.ssl_options is not None:
            stream = SSLIOStream(connection, io_loop=self.io_loop,
                                 max_buffer_size=self.max_buffer_size,
                                 read_chunk_size=self.read_chunk_size)
        else:
            stream = IOStream(connection, io_loop=self.io_loop,
                              max_buffer_size=self.max_buffer_size,
                              read_chunk_size=self.read_chunk_size)
        future = self.handle_stream(stream, address)
        if future is not None:
            self.io_loop.add_future(future, lambda f: f.result())
    except Exception:
        app_log.error("Error in connection callback", exc_info=True)
```
```
# netutil.py
def bind_sockets(port, address=None, family=socket.AF_UNSPEC,
                 backlog=_DEFAULT_BACKLOG, flags=None):
    # 为每个网卡建立socket，并监听
    sockets = []
    if address == "":
        address = None
    if not socket.has_ipv6 and family == socket.AF_UNSPEC:

        family = socket.AF_INET
    if flags is None:
        flags = socket.AI_PASSIVE
    bound_port = None
    for res in set(socket.getaddrinfo(address, port, family, socket.SOCK_STREAM,
                                      0, flags)):
        af, socktype, proto, canonname, sockaddr = res
        if (sys.platform == 'darwin' and address == 'localhost' and
                af == socket.AF_INET6 and sockaddr[3] != 0):
        try:
            sock = socket.socket(af, socktype, proto)
        except socket.error as e:
            if errno_from_exception(e) == errno.EAFNOSUPPORT:
                continue
            raise
        set_close_exec(sock.fileno())
        if os.name != 'nt':
            sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        if af == socket.AF_INET6:
            if hasattr(socket, "IPPROTO_IPV6"):
                sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)
        host, requested_port = sockaddr[:2]
        if requested_port == 0 and bound_port is not None:
            sockaddr = tuple([host, bound_port] + list(sockaddr[2:]))

        sock.setblocking(0)
        sock.bind(sockaddr)
        bound_port = sock.getsockname()[1]
        sock.listen(backlog)
        sockets.append(sock)
    return sockets
    
# netutil.py
# 为socket添加，连接建立之后的回调处理器
def add_accept_handler(sock, callback, io_loop=None):
    if io_loop is None:
        io_loop = IOLoop.current()

    def accept_handler(fd, events):
        for i in xrange(_DEFAULT_BACKLOG):
            try:
                connection, address = sock.accept()
            except socket.error as e:
                if errno_from_exception(e) == errno.ECONNABORTED:
                    continue
                raise
            # 这里的callback为`TCPServer._handle_connection(...)`
            callback(connection, address)
    # 注册到io_loop
    io_loop.add_handler(sock, accept_handler, IOLoop.READ)
```

```
# ioloop.py
def add_handler(self, fd, handler, events):
    fd, obj = self.split_fd(fd)
    self._handlers[fd] = (obj, stack_context.wrap(handler))
    self._impl.register(fd, events | self.ERROR)
```
