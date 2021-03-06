---
title: thread local in python
date: 2018-11-13 11:50:25
updated: 2018-11-13 11:50:25
tags: [python]
author: TheBigFish
---
<!-- TOC -->

-   [thread local in python](#thread-local-in-python)
    -   [线程局部变量](#%E7%BA%BF%E7%A8%8B%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F)
    -   [主线程也有自己的线程局部变量](#%E4%B8%BB%E7%BA%BF%E7%A8%8B%E4%B9%9F%E6%9C%89%E8%87%AA%E5%B7%B1%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F)
    -   [继承 threading.local](#%E7%BB%A7%E6%89%BF-threadinglocal)
    -   [应用实例](#%E5%BA%94%E7%94%A8%E5%AE%9E%E4%BE%8B)

<!-- /TOC -->

# thread local in python

参考 [Thread Locals in Python: Mostly easy](http://slinkp.com/python-thread-locals-20171201.html)

## 线程局部变量

```python
import threading

mydata = threading.local()
mydata.x = 'hello'

class Worker(threading.Thread):
    def run(self):
        mydata.x = self.name
        print mydata.x

w1, w2 = Worker(), Worker()
w1.start(); w2.start(); w1.join(); w1.join()
```

    Thread-1
    Thread-2

各线程独享自己的变量，但是使用全局变量 mydata

## 主线程也有自己的线程局部变量

```python
import threading

mydata = threading.local()
mydata.x = {}

class Worker(threading.Thread):
    def run(self):
        mydata.x['message'] = self.name
        print mydata.x['message']
w1, w2 = Worker(), Worker()
w1.start(); w2.start(); w1.join(); w2.join()
```

    Exception in thread Thread-1:
    Traceback (most recent call last):
      File "C:\Python27\lib\threading.py", line 801, in __bootstrap_inner
        self.run()
      File "E:/learn/python/test/thread_local.py", line 15, in run
        mydata.x['message'] = self.name
    AttributeError: 'thread._local' object has no attribute 'x'

    Exception in thread Thread-2:
    Traceback (most recent call last):
      File "C:\Python27\lib\threading.py", line 801, in __bootstrap_inner
        self.run()
      File "E:/learn/python/test/thread_local.py", line 15, in run
        mydata.x['message'] = self.name
    AttributeError: 'thread._local' object has no attribute 'x'

线程 w1,w2 没有 x 属性，子线程与主线程拥有各自的变量

## 继承 threading.local

```python
import threading

class MyData(threading.local):
    def __init__(self):
        self.x = {}

mydata = MyData()

class Worker(threading.Thread):
    def run(self):
        mydata.x['message'] = self.name
        print mydata.x['message']

w1, w2 = Worker(), Worker()
w1.start(); w2.start(); w1.join(); w2.join()
```

    Thread-1
    Thread-2

## 应用实例

bottle 0.4.10

```python
class Request(threading.local):
    """ Represents a single request using thread-local namespace. """

    def bind(self, environ):
        """ Binds the enviroment of the current request to this request handler """
        self._environ = environ
        self._GET = None
        self._POST = None
        self._GETPOST = None
        self._COOKIES = None
        self.path = self._environ.get('PATH_INFO', '/').strip()
        if not self.path.startswith('/'):
            self.path = '/' + self.path

#----------------------
request = Request()
#----------------------


def WSGIHandler(environ, start_response):
    """The bottle WSGI-handler."""
    global request
    global response
    request.bind(environ)
    response.bind()
    try:
        handler, args = match_url(request.path, request.method)
        if not handler:
            raise HTTPError(404, "Not found")
        output = handler(**args)
    except BreakTheBottle, shard:
        output = shard.output
```


***
Sync From: https://github.com/TheBigFish/blog/issues/4
