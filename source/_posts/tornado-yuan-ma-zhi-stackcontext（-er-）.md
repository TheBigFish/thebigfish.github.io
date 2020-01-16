---
title: tornado源码之StackContext（二）
date: 2018-12-21 16:51:30
updated: 2019-01-11 15:29:51
tags: [Tornado]
author: TheBigFish
---
# tornado 源码之 StackContext（二）

> StackContext allows applications to maintain threadlocal-like state
> that follows execution as it moves to other execution contexts.
>
> an exception
> handler is a kind of stack-local state and when that stack is suspended
> and resumed in a new context that state needs to be preserved.
>
> 一个栈结构的上下文处理类  
> 异常处理也是一个栈结构的上下文应用

## contents

-   [tornado 源码之 StackContext（二）](#tornado-%E6%BA%90%E7%A0%81%E4%B9%8B-stackcontext%E4%BA%8C)
    -   [contents](#contents)
    -   [example usage](#example-usage)
    -   [head](#head)
    -   [\_State](#state)
    -   [StackContext](#stackcontext)
    -   [ExceptionStackContext](#exceptionstackcontext)
        -   [example](#example)
    -   [NullContext](#nullcontext)
    -   [wrap](#wrap)
    -   [copyright](#copyright)

## example usage

```python
@contextlib.contextmanager
def die_on_error():
    try:
        yield
    except:
        logging.error("exception in asynchronous operation",exc_info=True)
        sys.exit(1)

with StackContext(die_on_error):
    # Any exception thrown here *or in callback and its desendents*
    # will cause the process to exit instead of spinning endlessly
    # in the ioloop.
    http_client.fetch(url, callback)
ioloop.start()
```

## head

```python
from __future__ import with_statement

import contextlib
import functools
import itertools
import logging
import threading
```

## \_State

```python
class _State(threading.local):
    def __init__(self):
        self.contexts = ()
_state = _State()
```

## StackContext

1.  初始化时保持传入的上下文对象
2.  \_\_enter\_\_
    1.  保存当前的全局上下文
    2.  append 新的上下文到全局上下文
    3.  构造新的上下文
    4.  进入新的上下文 \_\_enter\_\_
3.  \_\_exit\_\_
    1.  调用新的上下文 context \_\_exit\_\_
    2.  回复全局上下文

全局上下文保存整个执行程序的上下文（栈）  
with StackContext(context) 使程序包裹在 (global_context, context) 上执行  
执行完成后恢复全局上下文

```python
class StackContext(object):
    def __init__(self, context_factory):
        self.context_factory = context_factory


    def __enter__(self):
        self.old_contexts = _state.contexts
        _state.contexts = (self.old_contexts +
                           ((StackContext, self.context_factory),))
        try:
            self.context = self.context_factory()
            self.context.__enter__()
        except Exception:
            _state.contexts = self.old_contexts
            raise

    def __exit__(self, type, value, traceback):
        try:
            return self.context.__exit__(type, value, traceback)
        finally:
            _state.contexts = self.old_contexts
```

## ExceptionStackContext

捕获上下文执行中抛出而又未被捕获的异常  
作用类似 finally  
用于执行在程序抛出异常后记录日志、关闭 socket 这些现场清理工作  
如果 exception_handler 中返回 True, 表明异常已经被处理，不会再抛出

### example

```python
from tornado import ioloop
import tornado.stack_context
import contextlib

ioloop = tornado.ioloop.IOLoop.instance()

@contextlib.contextmanager
def context_without_catch():
    print("enter context")
    yield
    print("exit context")


def exception_handler(type, value, traceback):
    print "catch uncaught exception:", type, value, traceback
    return True

def main():
    with tornado.stack_context.ExceptionStackContext(exception_handler):
        with tornado.stack_context.StackContext(context_without_catch):
            print 0 / 0

main()
# enter context
# catch uncaught exception: <type 'exceptions.ZeroDivisionError'> integer division or modulo by zero <traceback object at 0x0000000003321FC8>
```

\_\_exit\_\_ 中捕获 with 语句所包裹的程序执行中所抛出的异常，调用注册的 exception_handler 进行处理  
exception_handler 返回 True，则异常不会蔓延

```python
class ExceptionStackContext(object):
    def __init__(self, exception_handler):
        self.exception_handler = exception_handler

    def __enter__(self):
        self.old_contexts = _state.contexts
        _state.contexts = (self.old_contexts +
                           ((ExceptionStackContext, self.exception_handler),))

    def __exit__(self, type, value, traceback):
        try:
            if type is not None:
                return self.exception_handler(type, value, traceback)
        finally:
            _state.contexts = self.old_contexts
```

## NullContext

临时构造一个空的全局上下文

```python
class NullContext(object):
    def __enter__(self):
        self.old_contexts = _state.contexts
        _state.contexts = ()

    def __exit__(self, type, value, traceback):
        _state.contexts = self.old_contexts
```

## wrap

1.  比较当时的全局上下文（\_state.contexts）和闭包中保存的上下文 (contexts)
    1.  如果当前上下文长度长，表面执行环境需要重新构造
    2.  如果有两者有任何一个上下文不同，执行环境也要重新构造
        1.  新建一个 NullContext(), 清除当前的\_state.contexts(保存原来的，提出时复原)
        2.  以此为基础构造一个 contexts 上下文链
    3.  如果 contexts 是 当前上下文的一个 prefix，则将当前上下文的后续部分作为上下文链，前面共有的无需再构造
2.  在新的上下文链（new_contexts）上执行 with 操作，保证 callback 的执行环境与当时保存时的一模一样

之所以进行这样复杂的操作，是为了对某些前面执行环境相同的情况省略前面的构造，节省时间，否则，可以用一行代替：

`new_contexts = ([NullContext()] + [cls(arg) for (cls,arg) in contexts])`

```python
def wrap(fn):
    if fn is None:
      return None

    def wrapped(callback, contexts, *args, **kwargs):
        # 函数实际调用时，上下文环境发生了变化，与`contexts = _state.contexts`已经有所不同
        if (len(_state.contexts) > len(contexts) or
            any(a[1] is not b[1]
                for a, b in itertools.izip(_state.contexts, contexts))):
            # contexts have been removed or changed, so start over
            new_contexts = ([NullContext()] +
                            [cls(arg) for (cls,arg) in contexts])
        else:
            new_contexts = [cls(arg)
                            for (cls, arg) in contexts[len(_state.contexts):]]
        if len(new_contexts) > 1:
            with contextlib.nested(*new_contexts):
                callback(*args, **kwargs)
        elif new_contexts:
            with new_contexts[0]:
                callback(*args, **kwargs)
        else:
            callback(*args, **kwargs)
    if getattr(fn, 'stack_context_wrapped', False):
        return fn

    # 保存上下文环境
    contexts = _state.contexts
    result = functools.partial(wrapped, fn, contexts)
    result.stack_context_wrapped = True
    return result
```

## copyright

author：bigfish  
copyright: [许可协议 知识共享署名 - 非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)


***
Sync From: https://github.com/TheBigFish/blog/issues/10
