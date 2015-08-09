# 异步在Tornado中的实现(coroutine and yield in Tornado)
对异步I/O而言，当遇到I/O需要等待时，就注册一个事件告诉系统，当数据准备好了，就调用这个函数就行，然后就立刻返回，继续执行其他，然后当事件循环机制发现数据可读啦，就调用回调函数。所以一般来说实现异步函数需要**事件循环机制**、**回调函数**。

##### 几个注意的点
* 无论是回调还是协程，都必须在过程中使用异步函数。
* 通过future来代表处理结果，简单来说用来代表执行结果，包含判断是否执行完毕、是否还在运行、是否有结果等方法。异步函数通过`set_result()`与`future`联系起来

#### 1. 基于回调函数实现
1. `fetch`返回`future`
2. `fetch_impl`，非阻塞socket连接(注册`ioloop`，等待数据可读)
3. 有数据可以读取(response或error)，如果有response则调用回调函数即`handle_response()`，设置`future`的结果：`future.set_result(response)`，
4. `future.set_result(response)`调用所有_callbacks这里即`handle_future()`
5. `handle_future()`注册真正的回调函数处理结果，下次调用



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

#### 2. `@tornado.gen.coroutine`配合yield实现异步
和回调某些方面类似的，但使用协程可以通过`send(result)`，将数据回传给`response`。然而`send(result)`则为当future有结果后的回调函数中的语句。可以与上面进行类比。

```
    class GenAsyncHandler(RequestHandler):
        @gen.coroutine
        def get(self):
            http_client = AsyncHTTPClient()
            # 挂起等待,send(result)
            response = yield http_client.fetch('http://example.com')
            do_something_with_response(response)
            self.render('template.html')
```
#### 下面是具体的代码分析
```
# gen.py
# @gen.coroutine
def _make_coroutine_wrapper(func, replace_callback):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        future = TracebackFuture()

        if replace_callback and 'callback' in kwargs:
            callback = kwargs.pop('callback')
            IOLoop.current().add_future(
                future, lambda future: callback(future.result()))

        try:
            # result是包含yield的那个函数生成器
            result = func(*args, **kwargs)
        except (Return, StopIteration) as e:
            result = getattr(e, 'value', None)
        except Exception:
            future.set_exc_info(sys.exc_info())
            return future
        else:
            if isinstance(result, types.GeneratorType):
                try:
                    orig_stack_contexts = stack_context._state.contexts
                    ＃ 启动生成器
                    # 返回异步函数的(这里为fetch)的future，通过该future将异步函数和yiled联系起来。重要！
                    yielded = next(result)
                    if stack_context._state.contexts is not orig_stack_contexts:
                        yielded = TracebackFuture()
                        yielded.set_exception(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                # 生成器函数执行完毕
                    future.set_result(getattr(e, 'value', None))
                except Exception:
                    future.set_exc_info(sys.exc_info())
                else:
                    Runner(result, future, yielded)
                try:
                    return future
                finally:
                    future = None
        future.set_result(result)
        #　返回future
        return future
    ＃　返回修饰后的函数
    return wrapper
```

```
# gen.py
class Runner(object):
    def handle_yield(self, yielded):
        ...
        ＃　跳过其他处理code
        try:
            self.future = convert_yielded(yielded)
        except BadYieldError:
            self.future = TracebackFuture()
            self.future.set_exc_info(sys.exc_info())
    
        if not self.future.done() or self.future is moment:
            # 注册回调函数，当fetch调用set_result()，调用run()
            self.io_loop.add_future(
                self.future, lambda f: self.run())
            return False
        return True
    
    def run(self):
            if self.running or self.finished:
                return
            try:
                self.running = True
                while True:
                    future = self.future
                    if not future.done():
                    #执行run时generator返回的那个future必须已经有结果，否则就没必要传回到generator中了
                        return
                    self.future = None
                    try:
                        orig_stack_contexts = stack_context._state.contexts
                        exc_info = None
    
                        try:
                            value = future.result()
                        except Exception:
                            self.had_exception = True
                            exc_info = sys.exc_info()
    
                        if exc_info is not None:
                            yielded = self.gen.throw(*exc_info)
                            exc_info = None
                        else:
                            # send(value)将结果回传给response
                            yielded = self.gen.send(value)
    
                        if stack_context._state.contexts is not orig_stack_contexts:
                            self.gen.throw(
                                stack_context.StackContextInconsistentError(
                                    'stack_context inconsistency (probably caused '
                                    'by yield within a "with StackContext" block)'))
                    except (StopIteration, Return) as e:
                        #generator执行完毕并成功的处理
                    except Exception:
                        #generator执行过程中异常的处理
                    if not self.handle_yield(yielded):
                        #这里generator还没有执行完毕，yielded是generator迭代过一次之后返回的新yielded。如果yieled还没有被解析出结果就通过handle_yield给yieled设置完成时的重启run的回调,否则yielded已经有结果，就再次运行run，所以run中才会有一个循环
                        return
            finally:
                self.running = False
```
continue...

