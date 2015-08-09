# Python协程
## yield协程

`yield`左边为接收参数，右边为计算表达式。每次执行至yield右表达计算完毕。等待下`send(传递参数给左值)`或`next`
```
import time
def coroutine():
    def pr():
        time.sleep(3)
        print('func runing')
        return 3
    print('ready for yield')    
    while True:
        print('...')
        n=yield(pr())
        print('running')
        print(n)

x=coroutine()
x.next()
print('---')
t=x.send(5)
print(t)
print('---')
t=x.send(10)
print(t)
print('---')
t=x.send(15)
print(t)


// 输出结果
ready for yield
...
#sleep 3sec
func runing
---
running
5
...
#sleep 3sec
func runing
3
---
running
10
...
#sleep 3sec
func runing
3
```
为了避免第一次出现'ready for yield'，即跳过无用开头。可以采用装饰器包裹
```
def gencoroutine(func):
    # 装饰器需要传递参数，则用两层包裹
    def start(*args,**kwargs):
        g=func(*args,**kwargs)
        g.next()
        return g
    return start
    
@gencoroutine
def coroutine():
    ...
```
