---
title: python decorators
date: 2018-11-29 09:19:21
updated: 2018-11-29 09:19:21
tags: []
author: TheBigFish
---
# python decorators

## 装饰器基础

### Decorator 本质

@ 本质是语法糖 - Syntactic Sugar
使用 @decorator 来修饰某个函数 func 时：

```python
@decorator
def func():
    pass
```

其解释器会解释成：

```python
func = decorator(func)
```

注意这条语句会被执行

**多重装饰器**

```python
@decorator_one
@decorator_two
def func():
    pass
```

相当于：

```python
func = decorator_one(decorator_two(func))
```

**带参数装饰器**

```python
@decorator(arg1, arg2)
def func():
    pass
```

相当于：

```python
func = decorator(arg1,arg2)(func)
```

### 使用 `*args、**kwargs` 给被装饰函数传递参数

```python
def wrapper(func):
    def wrapper_in(*args, **kwargs):
        # args是一个数组，kwargs一个字典
        print("%s is running" % func.__name__)
        return func(*args, **kwargs)
    return wrapper_in

@wrapper
def func(parameter1, parameter2, key1=1):
    print("call func with {} {} {}".format(parameter1, parameter2, key1))


func("haha", None, key1=2)

# func is running
# call func with haha None 2
```

### 带参数的装饰器

```python
def log(level):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if level == "warn":
                print("%s with warn is running" % func.__name__)
            elif level == "info":
                print("%s with info is running" % func.__name__)
            return func(*args, **kwargs)
        return wrapper

    return decorator


@log("warn")
def foo(*args, **kwargs):
    print("args {}, kwargs{}".format(args, kwargs))

foo(1, 2, a = 3)

# foo with warn is running
# args (1, 2), kwargs{'a': 3}
```

等同于

```python
def foo(name='foo'):
    print("args {}, kwargs{}".format(args, kwargs))

foo = log("warn")(foo)
```

### 方法装饰器

类方法是一个特殊的函数，它的第一个参数 self 指向类实例
所以我们同样可以装饰类方法

```python
def decorate(func):
   def wrapper(self):
       return "<p>{0}</p>".format(func(self))
   return wrapper

class Person(object):
    def __init__(self):
        self.name = "John"
        self.family = "Doe"

    @decorate
    def get_fullname(self):
        return self.name+" "+self.family

my_person = Person()
print my_person.get_fullname()

# <p>John Doe</p>
```

上例相当于固定了 self 参数, 不太灵活  
使用 `*args, **kwargs`传递给 wrapper 更加通用：

```python
def pecorate(func):
   def wrapper(*args, **kwargs):
       return "<p>{0}</p>".format(func(*args, **kwargs))
   return wrapper

class Person(object):
    def __init__(self):
        self.name = "John"
        self.family = "Doe"

    @pecorate
    def get_fullname(self):
        return self.name+" "+self.family

my_person = Person()

print my_person.get_fullname()
```

### 类装饰器

类实现 `__call__` 方法后变成可调用对象，故可以用类做装饰器

```python
class EntryExit(object):

    def __init__(self, f):
        self.f = f

    def __call__(self):
        print "Entering", self.f.__name__
        self.f()
        print "Exited", self.f.__name__

@EntryExit
def func1():
    print "inside func1()"

@EntryExit
def func2():
    print "inside func2()"

def func3():
    pass

print type(EntryExit(None))
# func1 变为类实例
print type(func1)
print type(EntryExit)
# func3 是普通函数
print type(func3)
func1()
func2()

# <class '__main__.EntryExit'>
# <class '__main__.EntryExit'>
# <type 'type'>
# <type 'function'>
# Entering func1
# inside func1()
# Exited func1
# Entering func2
# inside func2()
# Exited func2
```

类装饰器

```python
@EntryExit
def func1():
    print "inside func1()"
```

等同于

```python
def func1():
    print "inside func1()"
# 此处可以看出 func1 是类EntryExit的一个实例
func1 = EntryExit(myfunc1)
```

### 装饰器装饰类

```python
register_handles = []


def route(url):
    global register_handles

    def register(handler):
        register_handles.append((".*$", [(url, handler)]))
        return handler

    return register

@route("/index")
class Index():
    def get(self, *args, **kwargs):
        print("hi")

# Index 仍然为原来定义的类实例
# 相当于在定义类的同时调用装饰器函数 route， 将该类注册到全局路由 register_handles
@route("/main")
class Main():
    def get(self, *args, **kwargs):
        print("hi")

print (register_handles)

print(type(Index))

# [('.*$', [('/index', <class __main__.Index at 0x0000000002A49828>)]), ('.*$', [('/main', <class __main__.Main at 0x0000000002FBABE8>)])]
# <type 'classobj'>
```

```python
@route("/index")
class Index():
    def get(self, *args, **kwargs):
        print("hi")
```

```python
Index = route("/index")(Index)
# register 返回传入的 handler,故 Index 仍然为类对象
```

## functools

上述装饰器实现有个问题，就是被装饰函数的属性被改变


***
Sync From: https://github.com/TheBigFish/blog/issues/7
