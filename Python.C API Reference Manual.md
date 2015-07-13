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

