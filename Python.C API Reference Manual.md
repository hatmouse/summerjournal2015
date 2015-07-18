#Introduction
Python/C API通常用在为Python解释器写扩展或者嵌入Python
为Python写扩展比嵌入Python简单一些，并且大部分在写嵌入Python时同样需要写扩展
#头文件
在写程序时应该包含Python.h
在Python.h中定义的所有用户可见的名字都是以Py或者_Py为前缀.其中以_Py开头的名字是Python解释器内部使用并且扩展编写者不应该使用
#Reference Count
The only real reason to use the reference count is to prevent the object from being deallocated as long as our variable is pointing to it
## 所有权规则
[Extending Python with C or C++](http://www.incoding.org/admin/archives/808.html)

大多数返回对象引用的函数，通过引用传递所有权。
* 许多从**其他对象上提取子对象**的函数，通过引用传递所有权，但有一些例外，**PyTuple_GetItem(),PyList_GetItem(),PyDict_GetItem()，和PyDict_GetItemString()**，这些返回的引用是从tuple，list或dict中**借用**的.(借用仅获得拷贝，引用计数并不增加)
* 当你传递一个对象引用给其他函数，这个函数会从借用这个引用，如果需要保存它应该使用Py_INCREF()转换为独立拥有者。但是有例外，**PyTuple_SetItem()和PyList_SetItem()**，直接传递对象所有权
* Python调用一个C函数的返回对象必须拥有引用所有权传递给它的调用者    

### 借用引用
* PyTuple_GetItem()
* PyList_GetItem()
* PyList_GET_ITEM()
* PyList_SET_ITEM()
* PyDict_GetItem()
* PyDict_GetItemString()
* PyErr_Occurred()
* PyFile_Name()
* PyImport_GetModuleDict()
* PyModule_GetDict()
* PyImport_AddModule()
* PyObject_Init()
* Py_InitModule()
* Py_InitModule3()
* Py_InitModule4()
* PySequence_Fast_GET_ITEM()

##引用计数 ob_refcnt

```
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;                    
```

## increment or decrement reference counts
```
#define Py_INCREF(op) (                         \
    _Py_INC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
    ((PyObject *)(op))->ob_refcnt++)

#define Py_DECREF(op)                                   \
    do {                                                \
        PyObject *_py_decref_tmp = (PyObject *)(op);    \
        if (_Py_DEC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
        --(_py_decref_tmp)->ob_refcnt != 0)             \
            _Py_CHECK_REFCNT(_py_decref_tmp)            \
        else                                            \
        _Py_Dealloc(_py_decref_tmp);                    \
    } while (0)
```

#When to Use Py_INCREF() and Py_DECREF()
[Reference Counting in Python](http://edcjones.tripod.com/refcount.html)

##函数返回对象

PyObject* Py_Something(arguments);
绝大部分函数会在返回新对象之前调用Py_INCREF
MyCode必须处理pyo，调用Py_DECREF

```
void MyCode(arguments)
{
    PyObject* pyo;
    ...
    pyo = Py_Something(args);
}
```

但是如果MyCode传递pyo的所有权，则不能调用Py_DECREF

```
PyObject* MyCode(arguments) {
    PyObject* pyo;
    ...
    pyo = Py_Something(args);
    ...
    return pyo;
}
```
如果一个函数返回None，需要INCREF(Py_None)

[Py_INCREF(Py_None) from stackoverflow](http://stackoverflow.com/questions/15287590/why-should-py-increfpy-none-be-required-before-returning-py-none-in-c)

```
Py_INCREF(Py_None);
return Py_None;
```
不要草率的 使用INCREF：不需要对每个本地变量的引用+1，因为当一个变量创建并且有一个指针指向时，默认INC，然而当变量失去作用范围又会DEC，两者抵消。

**仅有当你需要保护这个变量不被释放时才使用INCREF**

* PyList_GetItem仅获得对应项目拷贝，不增加引用计数

```
long sum_list(PyObject *list)
{
 int i, n;
 long total = 0;
 PyObject *item;

 n = PyList_Size(list);
 if (n < 0)
     return -1; /* Not a list */
     /* Caller should use PyErr_Occurred() if a -1 is returned. */
 for (i = 0; i < n; i++) {
     /* PyList_GetItem does not INCREF "item".
        "item" is unprotected and borrowed. IMPORTANT!!! */
     item = PyList_GetItem(list, i); /* Can't fail */
     if (PyInt_Check(item))
         total += PyInt_AsLong(item);
 }
 return total;
}
```
* PySequence_GetItem获得对象所有权，其返回对象+1，因而每次循环后需要-1.

```
long sum_sequence(PyObject *sequence)
{
 int i, n;
 long total = 0;
 PyObject *item;
 n = PySequence_Length(sequence);
 if (n < 0)
     return -1; /* Has no length. */
     /* Caller should use PyErr_Occurred() if a -1 is returned. */
 for (i = 0; i < n; i++) {
     /* PySequence_GetItem INCREFs item.  IMPORTANT!!!*/
     item = PySequence_GetItem(sequence, i);
     if (item == NULL)
         return -1; /* Not a sequence, or other failure */
     if (PyInt_Check(item))
         total += PyInt_AsLong(item);
     Py_DECREF(item);
 }
 return total;
}
```
### INCREF不可马虎
常见的情况是从list中提取对象，一些操作符可以能会替换或者移除list中某个对象，并且假如这个向对象是用户自定义的calss，包含__del__，然而这个__del__可以执行任意的code，但是这些操作可能会无意的DEC 该list[0]的引用计数，导致free。

item利用PyList_GetItem借用list引用
```
bug(PyObject *list) {
 PyObject *item = PyList_GetItem(list, 0);
 //修改措施Py_INCREF(item); /* Protect item. */
 /* This function “steals” a reference to item and discards a reference to an item already in the list at the affected position.
 可能引起list中原list[1]中__del__，导致DEClist[0]*/
 PyList_SetItem(list, 1, PyInt_FromLong(0L));
 PyObject_Print(item, stdout, 0); /* BUG! */
 //修改措施:Py_DECREF(item);
}
```

### 传递给函数的对象

* ** Most functions assume that the arguments passed to them are already protected.Therefore Py_INCREF() is not called inside Function unless Function wants the argument to continue to exist after Caller exits. In the documentation, Function is said to borrow a reference:**
* **When you pass an object reference into another function, in general, the function borrows the reference from you   if it needs to store it, it will use Py_INCREF() to become an independent owner.**
* **大多数函数假定传入函数的参数都是受保护的不需要INCREF，除非希望参数在函数exit后仍然存在。官方文档说法是函数借用引用**
* **你传递对象给一个函数，一般情况来说是函数借用引用，如果你希望保存那么请INCREF，将其变为独立拥有者。**

* PyDict_SetItem()非借用，既然是store变量到dict，因此PyDict_SetItem() INCREF它的kye和value。
* 但是PyTuple_SetItem()和PyList_SetItem()比较特殊，接管所有权(take over responsibility) or 偷取引用(steal a reference)
* PyTuple_SetItem(tuple,i,item)实现：如果tuple[i]存在PyObject则DECREF，然后tuple[i]设置为item。并且Item并没有INCREF
* PyTuple_SetItem(tuple,i,item)既然是steal，那么**Item之前必须有所有权**
* 如果PyTuple_SetItem()插入item失败，则DECREF item引用计数
* PyTuple_SetItem()是设置Tuple中item的唯一方法

### Steal and Borrow

* **steal a referfence(take over responsibilty)**
```
/*你不需要调用DECREF(x)，PyTuple_SetItem()已经自动调用了,当Tuple被DECREF时，它的item也会被DECREF*/
PyObjetc *t;
PyObject *x;
x=PyIntFromLong(1L);
PyTuple_SetItem(t,0,x);
```

* **borrow a reference**
```
/*Better way*/
PyObject *l, *x;
l = PyList_New(3);
x = PyInt_FromLong(1L);
PySequence_SetItem(l, 0, x); Py_DECREF(x);
x = PyInt_FromLong(2L);
PySequence_SetItem(l, 1, x); Py_DECREF(x);
x = PyString_FromString("three");
PySequence_SetItem(l, 2, x); Py_DECREF(x);
```

### More Common Way to Populate a tuple or list
```
PyObject *t, *l; 
t = Py_BuildValue("(iis)", 1, 2, "three");
l = Py_BuildValue("[iis]", 1, 2, "three");
```
## Two Examples

### Example 1

This is a pretty standard example of C code using the Python API.
```
PyObject*
    MyFunction(void)
    {
        PyObject* temporary_list=NULL;
        PyObject* return_this=NULL;

        temporary_list = PyList_New(1);          /* Note 1 */
        if (temporary_list == NULL)
            return NULL;

        return_this = PyList_New(1);             /* Note 1 */
        if (return_this == NULL)
            Py_DECREF(temporary_list);           /* Note 2 */
            return NULL;
        }

        Py_DECREF(temporary_list);               /* Note 2 */
        return return_this;
    }
```
* Note 1: The object returned by PyList_New has a reference count of 1.
* Note 2: Since temporary_list should disappear when MyFunction exits, it must be DECREFed before any return from the function. If a return can be reached both before or after temporary_list is created, then initialize temporary_list to NULL and use Py_XDECREF().

### Example 2
```
This is the same as Example 1 except PyTuple_GetItem() is used.
    PyObject*
    MyFunction(void)
    {
        PyObject* temporary=NULL;
        PyObject* return_this=NULL;
        PyObject* tup;
        PyObject* num;
        int err;

        tup = PyTuple_New(2);
        if (tup == NULL)
            return NULL;

        err = PyTuple_SetItem(tup, 0, PyInt_FromLong(222L));  /* Note 1 */
        if (err) {
            Py_DECREF(tup);
            return NULL;
        }
        err = PyTuple_SetItem(tup, 1, PyInt_FromLong(333L));  /* Note 1 */
        if (err) {
            Py_DECREF(tup);
            return NULL;
        }

        temporary = PyTuple_Getitem(tup, 0);          /* Note 2 */
        if (temporary == NULL) {
            Py_DECREF(tup);
            return NULL;
        }

        return_this = PyTuple_Getitem(tup, 1);        /* Note 3 */
        if (return_this == NULL) {
            Py_DECREF(tup);
            /* Note 3 */
            return NULL;
        }

        /* Note 3 */
        Py_DECREF(tup);
        return return_this;
    }
```
* Note 1: If PyTuple_SetItem fails or if the tuple it created is DECREFed to 0, then the object returned by **PyInt_FromLong is DECREFed**.
* Note 2: PyTuple_Getitem does not increment the reference count for the object it returns.
* Note 3: You have no responsibility for DECFREFing temporary.
