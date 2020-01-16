---
title: python 类函数绑定方法的实现
date: 2019-12-18 15:43:27
updated: 2019-12-18 16:08:11
tags: [python]
author: TheBigFish
---
# python 类函数绑定方法的实现

## 实现一个函数描述器

> This means that all functions are non-data descriptors which return bound methods when they are invoked from an object.

所有的函数都是一个无数据的描述器。类实例调用函数即触发描述器语法，该描述器在类实例被调用时，返回一个绑定的普通方法。

下面实现了一个纯 python 的描述器 BindFunction, 用来绑定方法 f_normal 到函数 f。

```python
class D:
    def f(self, name):
        print(self, name)
d=D()
d.f("hello")
D.f(d, "hello")


import types
from functools import wraps


def f_normal(self, name):
    print(self, name)


class BindFunction(object):
    def __init__(self, func):
        wraps(func)(self)

    def __call__(self, *args, **kwargs):
        return self.__wrapped__(*args, **kwargs)

    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        if obj is None:
            return self
        return types.MethodType(self,  obj)


class D:
    f = BindFunction(f_normal)

d = D()
d.f("world")
D.f(d, "world")
```

类 BindFunction 也可以实现如下：

```python
class BindFunction(object):
    def __init__(self, func):
        self.func = func

    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        if obj is None:
            return self.func
        return types.MethodType(self.func,  obj)
```

## Functions and Methods

[Functions and Methods](https://docs.python.org/3/howto/descriptor.html?highlight=types%20methodtype#functions-and-methods)

>     Python’s object oriented features are built upon a function based environment. Using non-data descriptors, the two are merged seamlessly.

    Class dictionaries store methods as functions. In a class definition, methods are written using def or lambda, the usual tools for creating functions. Methods only differ from regular functions in that the first argument is reserved for the object instance. By Python convention, the instance reference is called self but may be called this or any other variable name.

    To support method calls, functions include the __get__() method for binding methods during attribute access. This means that all functions are non-data descriptors which return bound methods when they are invoked from an object. In pure Python, it works like this:

    class Function(object):
        . . .
        def __get__(self, obj, objtype=None):
            "Simulate func_descr_get() in Objects/funcobject.c"
            if obj is None:
                return self
            return types.MethodType(self, obj)
    Running the interpreter shows how the function descriptor works in practice:

    >>>
    >>> class D(object):
    ...     def f(self, x):
    ...         return x
    ...
    >>> d = D()

    # Access through the class dictionary does not invoke __get__.
    # It just returns the underlying function object.
    >>> D.__dict__['f']
    <function D.f at 0x00C45070>

    # Dotted access from a class calls __get__() which just returns
    # the underlying function unchanged.
    >>> D.f
    <function D.f at 0x00C45070>

    # The function has a __qualname__ attribute to support introspection
    >>> D.f.__qualname__
    'D.f'

    # Dotted access from an instance calls __get__() which returns the
    # function wrapped in a bound method object
    >>> d.f
    <bound method D.f of <__main__.D object at 0x00B18C90>>

    # Internally, the bound method stores the underlying function,
    # the bound instance, and the class of the bound instance.
    >>> d.f.__func__
    <function D.f at 0x1012e5ae8>
    >>> d.f.__self__
    <__main__.D object at 0x1012e1f98>
    >>> d.f.__class__
    <class 'method'>


***
Sync From: https://github.com/TheBigFish/blog/issues/14
