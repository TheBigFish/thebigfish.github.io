---
title: tornado-异步上下文管理（StackContext）
date: 2018-10-17 13:18:50
updated: 2018-10-17 13:53:35
tags: [python]
author: TheBigFish
---
# tornado - 异步上下文管理（StackContext）

## 初步使用

```python
# -*- coding: utf-8 -*-
import tornado.ioloop
import tornado.stack_context

ioloop = tornado.ioloop.IOLoop.instance()

times = 0

def callback():
    print 'run callback'
    raise ValueError('except in callback')


def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    ioloop.add_callback(callback=callback)


def main():
    try:
        async_task()
    except Exception as e:
        print 'main exception {}'.format(e)
    print 'end'

main()
ioloop.start()
```

异常没有在 main 中捕获：

    run async task 1
    end
    run callback
    ERROR:root:Exception in callback <function null_wrapper at 0x7f23ec300488>
    Traceback (most recent call last):
      File "~/learn/tornado/tornado/ioloop.py", line 370, in _run_callback

## 包裹上下文

使用 partial 生成新的函数, 最终调用的函数为 wrapper(callback)，在 wrapper 中捕获异常

```python
# -*- coding: utf-8 -*-
import tornado.ioloop
import tornado.stack_context
import functools

ioloop = tornado.ioloop.IOLoop.instance()

times = 0

def callback():
    print 'run callback'
    raise ValueError('except in callback')

def wrapper(func):
    try:
        func()
    except Exception as e:
        print 'main exception {}'.format(e)

def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    # 使用 partial 生成新的函数
    # 最终 ioloop 调用的函数为 wrapper(callback)
    ioloop.add_callback(callback=functools.partial(wrapper, callback))

def main():
    try:
        async_task()
    except Exception as e:
        print 'main exception {}'.format(e)
    print 'end'

main()
ioloop.start()
```

异常被正确捕获：

    run async task 1
    end
    run callback
    main exception except in callback

## 使用 tornado stack_context 例子

```python
# -*- coding: utf-8 -*-
import tornado.ioloop
import tornado.stack_context
import contextlib

ioloop = tornado.ioloop.IOLoop.instance()

times = 0

def callback():
    print 'Run callback'
    # 抛出的异常在 contextor 中被捕获
    raise ValueError('except in callback')

def async_task():
    global times
    times += 1
    print 'run async task {}'.format(times)
    # add_callback, 会用之前保存的 (StackContext, contextor)，创建一个对象 StackContext(contextor)
    # ioloop 回调的时候使用 
    # with StackContext(contextor)
    #   callback
    # 从而 callback 函数也在 contextor 函数中执行，从而能够在 contextor 中捕获异常
    # 从而实现 async_task() 函数在 contextor 中执行，其引发的异常（其实是 callback）同时在 contextor 被捕获
    ioloop.add_callback(callback=callback)

@contextlib.contextmanager
def contextor():
    print 'Enter contextor'
    try:
        yield
    except Exception as e:
        print 'Handler except'
        print 'exception {}'.format(e)
    finally:
        print 'Release'

def main():
    #使用StackContext包裹住contextor, 下面函数 async_task() 会在 contextor() 环境中执行
    stack_context = tornado.stack_context.StackContext(contextor)
    with stack_context:
        async_task()
    print 'End'


main()
ioloop.start()
```

### tornado.stack_context.StackContext

tornado.stack_context 相当于一个上下文包裹器，它接收一个 context_factory 作为参数并保存  
context_factory 是一个上下文类，拥有 `__enter__` `__exit__`方法

使用 with stack_context 时候，执行自己的 `__enter__`  
`__enter__` 函数根据保存的 context_factory 创建一个 context 对象，并执行对象的 `__enter__`方法  
StackContext 将 (StackContext, context_factory) 保存，将来执行回调的时候再创建一个 StackContext(context_factory) 来执行 call_back

```python
class StackContext(object):
    def __init__(self, context_factory):
        self.context_factory = context_factory

    def __enter__(self):
        # contexts 入栈
        self.old_contexts = _state.contexts
        # _state.contexts is a tuple of (class, arg) pairs
        _state.contexts = (self.old_contexts + 
                           ((StackContext, self.context_factory),))
        try:
            self.context = self.context_factory()
            # 进入 context 对象的执行环境
            self.context.__enter__()
        except Exception:
            _state.contexts = self.old_contexts
            raise

    def __exit__(self, type, value, traceback):
        try:
            return self.context.__exit__(type, value, traceback)
        finally:
            # contexts 出栈
            _state.contexts = self.old_contexts
```

### IOLoop.add_callback

```python
def add_callback(self, callback):
    if not self._callbacks:
        self._wake()
        self._callbacks.append(stack_context.wrap(callback))
```

### IOLoop.start

```python
def start(self):
    if self._stopped:
        self._stopped = False
        return
    self._running = True
    while True:
        # Never use an infinite timeout here - it can stall epoll
        poll_timeout = 0.2

        callbacks = self._callbacks
        self._callbacks = []
        for callback in callbacks:
            # 调用注册的 callback
            self._run_callback(callback)
```

### IOLoop.\_run_callback

```python
def _run_callback(self, callback):
    try:
        callback()
    except (KeyboardInterrupt, SystemExit):
        raise
    except:
        self.handle_callback_exception(callback)
```

### stack_context.wrap

```python
def wrap(fn):

    if fn is None or fn.__class__ is _StackContextWrapper:
        return fn
    # functools.wraps doesn't appear to work on functools.partial objects
    #@functools.wraps(fn)
    def wrapped(callback, contexts, *args, **kwargs):
        if contexts is _state.contexts or not contexts:
            callback(*args, **kwargs)
            return
        
        # 包裹callback, 生成 StackContext(context_factory()) 对象
        if not _state.contexts:
            new_contexts = [cls(arg) for (cls, arg) in contexts]

        elif (len(_state.contexts) > len(contexts) or
            any(a[1] is not b[1]
                for a, b in itertools.izip(_state.contexts, contexts))):
            # contexts have been removed or changed, so start over
            new_contexts = ([NullContext()] +
                            [cls(arg) for (cls,arg) in contexts])
        else:
            new_contexts = [cls(arg)
                            for (cls, arg) in contexts[len(_state.contexts):]]
        if len(new_contexts) > 1:
            with _nested(*new_contexts):
                callback(*args, **kwargs)
        elif new_contexts:
            # 执行 StackContext，调用 fn
            with new_contexts[0]:
                callback(*args, **kwargs)
        else:
            callback(*args, **kwargs)
    # 返回偏函数，绑定 fn, _state.contexts
    return _StackContextWrapper(wrapped, fn, _state.contexts)
```

```python
class _StackContextWrapper(functools.partial):
    pass
```


***
Sync From: https://github.com/TheBigFish/blog/issues/1
