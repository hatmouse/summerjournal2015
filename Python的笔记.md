# Python的笔记

### 1. class是type的instance
类也是对象
代码解释，解释器会在类创建末尾添加下面一句话啊。type创建一个type实例(即:calss className)，type也存在__new__(实例化type)，__int__(初始化类)
`class = type(className, superClasses, attributeDict)`

### 2. \__new__()实例化
* object是所有类的基类
* 依次向上调用\__new__实例化类，若父没有定义，则继续直至object基类
* 不能调用自身的\__new__来实例化对象。`cls.__new__(cls, *args, **kwargs)`这样不可行

```
# cls当前正在实例化的类
def __new__(cls, *args, **kwargs):
    ...
```

### 3. super继承
对于单继承，当父类修改名字可以避免多次修改
```
class Base(object):
    def __init__(self):
        print "Base created"

class ChildA(Base):
    def __init__(self):
        Base.__init__(self)

class ChildB(Base):
    def __init__(self):
        # Python3 可以直接使用super().__init__()
        super(ChildB, self).__init__()

ChildA() 
ChildB()
```
对于多继承,采用mro顺序调用，相同父类只调用一次
```
class D(A,B,C):
    def __init__(self):
        super(D，self).__init__()
```
