---
title:  tornado源码之StackContext（一）
date: 2018-12-21 16:50:38
updated: 2019-01-11 15:29:36
tags: [Tornado]
author: TheBigFish
---
# tornado 源码之 StackContext（一）

> tornado 的异步上下文机制分析

## contents

-   [tornado 源码之 StackContext（一）](#tornado-%E6%BA%90%E7%A0%81%E4%B9%8B-stackcontext%E4%B8%80)
    -   [contents](#contents)
    -   [MyIOLoop](#myioloop)
    -   [异步回调异常的捕获](#%E5%BC%82%E6%AD%A5%E5%9B%9E%E8%B0%83%E5%BC%82%E5%B8%B8%E7%9A%84%E6%8D%95%E8%8E%B7)
    -   [使用 wrap](#%E4%BD%BF%E7%94%A8-wrap)
    -   [使用 contextlib](#%E4%BD%BF%E7%94%A8-contextlib)
    -   [inspired by](#inspired-by)
    -   [copyright](#copyright)

我们实现一个简单的 MyIOLoop 类，模仿 tornado 的 IOLoop，实现异步回调  
实现一个简单的 MyStackContext 类，模仿 tornado 的 StackContext，实现上下文

## MyIOLoop

模拟 tornado IOLoop

```python
class MyIOLoop:
    def __init__(self):
        self._callbacks = []

    @classmethod
    def instance(cls):
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance

    def add_callback(self, call_back):
        self._callbacks.append(call_back)

    def start(self):
        callbacks = self._callbacks
        self._callbacks = []
        for call_back in callbacks:
            call_back()
```

## 异步回调异常的捕获

由输出可以看到，回调函数 call_func 中抛出的异常，在 main 函数中无法被捕获  
main 函数只能捕获当时运行的 async_task 中抛出的异常, async_task 只是向 MyIOLoop 注册了一个回调，并没有当场调用回调  
call_func 函数最终在 MyIOLoop.start 中调用，其异常没有被捕获

```python
my_io_loop = MyIOLoop.instance()
times = 0


def call_func():
    print 'run call_func'
    raise ValueError('except in call_func')


def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    my_io_loop.add_callback(call_back=call_func)


def main():
    try:
        async_task()
    except Exception as e:
        print 'main exception {}'.format(e)
        print 'end'


if __name__ == '__main__':
    main()
    my_io_loop.start()

# run async task 1
# Traceback (most recent call last):
# run call_func
#   File "E:/learn/python/simple-python/stack_context_example.py", line 56, in <module>
#     my_io_loop.start()
#   File "E:/learn/python/simple-python/stack_context_example.py", line 26, in start
#     call_back()
#   File "E:/learn/python/simple-python/stack_context_example.py", line 36, in call_func
#     raise ValueError('except in call_func')
# ValueError: except in call_func
```

## 使用 wrap

可以使用 wrap 的方式，把函数调用和异常捕捉写在一起，回调实际调用的是带异常捕捉的函数 wrapper

```python
my_io_loop = MyIOLoop.instance()
times = 0


def call_func():
    print 'run call_func'
    raise ValueError('except in call_func')


def wrapper(func):
    try:
        func()
    except Exception as e:
        print 'wrapper exception {}'.format(e)


def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    my_io_loop.add_callback(call_back=functools.partial(wrapper, call_func))


def main():
    try:
        async_task()
    except Exception as e:
        print 'main exception {}'.format(e)
        print 'end'


if __name__ == '__main__':
    main()
    my_io_loop.start()

# run async task 1
# run call_func
# wrapper exception except in call_func
```

由此，可以想到，构造一个上下文环境，使用全局变量保存这个执行环境，等回调函数执行的时候，构造出这个环境

## 使用 contextlib

下面模仿了 tornado 异步上下文实现机制

1.  MyStackContext 使用 \_\_enter\_\_ \_\_exit\_\_ 支持上下文
2.  MyStackContext 构造函数参数为一个上下文对象
3.  with MyStackContext(context) 进行如下动作：  
    **在 MyStackContext(context) 构造时，把 context 注册进全局工厂 MyStackContext.context_factory**
    1.  进入 MyStackContext 的\_\_enter
    2.  构造一个 context 对象
    3.  调用 context 对象的 \_\_enter，进入真正 context 上下文
    4.  执行 context 上下文，my_context yield 语句前的部分
    5.  执行上下文包裹的语句，async_task
    6.  async_task 中 add_callback，实际保存的 wrap, wrap 将此时的全局上下文环境 MyStackContext.context_factory 保存，以方便 call_back 调用
    7.  调用 context 对象的 \_\_exit, 退出 context 上下文
    8.  进入 MyStackContext 的\_\_exit
4.  my_io_loop.start() 执行, 调用注册的 \_call_back
5.  实际调用 wrapped 函数
    1.  获取保存的 context 环境
    2.  with context
    3.  调用真正的 callback

这样，在 main 函数中执行

```python
with MyStackContext(my_context):
    async_task()
```

构造一个执行上下文 my_context，异步函数将在这个上下文中调用  
效果上相当于在 my_context 这个上下文环境中调用 async_task  
类似：

```python
def my_context():
    print '---enter my_context--->>'
    try:
        async_task()
    except Exception as e:
        print 'handler except: {}'.format(e)
    finally:
        print '<<---exit my_context ---'
```

```python
import contextlib
import functools


class MyIOLoop:
    def __init__(self):
        self._callbacks = []

    @classmethod
    def instance(cls):
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance

    def add_callback(self, call_back):
        self._callbacks.append(wrap(call_back))

    def start(self):
        callbacks = self._callbacks
        self._callbacks = []
        for call_back in callbacks:
            self._call_back(call_back)

    @staticmethod
    def _call_back(func):
        func()


class MyStackContext(object):
    context_factory = []

    def __init__(self, context):
        if context:
            MyStackContext.context_factory.append(context)

    def __enter__(self):
        try:
            self.context = self.context_factory[0]()
            self.context.__enter__()
        except Exception:
            raise

    def __exit__(self, type, value, traceback):
        try:
            return self.context.__exit__(type, value, traceback)
        finally:
            pass


def wrap(fn):
    def wrapped(callback, contexts, *args, **kwargs):
        context = contexts[0]()
        with context:
            callback(*args, **kwargs)

    contexts = MyStackContext.context_factory
    result = functools.partial(wrapped, fn, contexts)
    return result


my_io_loop = MyIOLoop.instance()

times = 0


def call_func():
    print 'run call_func'
    raise ValueError('except in call_func')


def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    my_io_loop.add_callback(call_back=call_func)


@contextlib.contextmanager
def my_context():
    print '---enter my_context--->>'
    try:
        yield
    except Exception as e:
        print 'handler except: {}'.format(e)
    finally:
        print '<<---exit my_context ---'


def main():
    with MyStackContext(my_context):
        async_task()
    print 'end main'


if __name__ == '__main__':
    main()
    my_io_loop.start()

# ---enter my_context--->>
# run async task 1
# <<---exit my_context ---
# end main
# ---enter my_context--->>
# run call_func
# handler except: except in call_func
# <<---exit my_context ---
```

## inspired by

[Tornado 源码分析（二）异步上下文管理（StackContext）](https://www.jianshu.com/p/3e58f977b908)

## copyright

author：bigfish  
copyright: [许可协议 知识共享署名 - 非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)


***
Sync From: https://github.com/TheBigFish/blog/issues/9
