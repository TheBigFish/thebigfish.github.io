---
title: python 多线程编程
date: 2018-11-15 13:41:30
updated: 2018-11-15 13:41:30
tags: [python]
author: TheBigFish
---
# python 多线程编程

## 使用回调方式

```python
import time
def countdown(n):
    while n > 0:
        print('T-minus', n)
        n -= 1
        time.sleep(5)

# Create and launch a thread
from threading import Thread
t = Thread(target=countdown, args=(10,))
t.start()
```

## 使用继承方式

```python
from threading import Thread

class CountdownTask:
    def __init__(self):
        self._running = True

    def terminate(self):
        self._running = False

    def run(self, n):
        while self._running and n > 0:
            print('T-minus', n)
            n -= 1
            time.sleep(5)

c = CountdownTask()
t = Thread(target=c.run, args=(10,))
t.start()
c.terminate() # Signal termination
t.join()      # Wait for actual termination (if needed)
```

注意使用变量 `self._running` 退出线程的方式

## 使用 Queue 进行线程间通信

```python
import Queue
import threading
import time

task_queue = Queue.Queue()


class ThreadTest(threading.Thread):
    def __init__(self, queue):
        threading.Thread.__init__(self)
        self.queue = queue

    def run(self):
        while True:
            msg = self.queue.get()
            print(msg)
            time.sleep(0.1)
            self.queue.task_done()


def main():
    start = time.time()
    # populate queue with data
    for i in range(100):
        task_queue.put("message")

    # spawn a pool of threads, and pass them queue instance
    for i in range(5):
        t = ThreadTest(task_queue)
        t.setDaemon(True)
        t.start()

    # wait on the queue until everything has been processed
    task_queue.join()
    print "Elapsed Time: {}".format(time.time() - start)


if __name__ == "__main__":
    main()
```

setDaemon 设置为 True, run 函数中不需要退出，主线程结束后所有子线程退出  
如果 setDaemon 设置为 False, 则改为

```python
def run(self):
    while not self.queue.empty():
        msg = self.queue.get()
        print(msg)
        time.sleep(0.1)
        self.queue.task_done()
```

并且在主函数结束前 join 所有线程

### **注意**

-   向队列中添加数据项时并不会复制此数据项，线程间通信实际上是在线程间传递对象引用。如果你担心对象的共享状态，那你最好只传递不可修改的数据结构（如：整型、字符串或者元组）或者一个对象的深拷贝。

    ```python
      from queue import Queue
      from threading import Thread
      import copy

      # A thread that produces data
      def producer(out_q):
          while True:
              # Produce some data
              ...
              out_q.put(copy.deepcopy(data))

      # A thread that consumes data
      def consumer(in_q):
          while True:
              # Get some data
              data = in_q.get()
              # Process the data
              ...
    ```

-   q.qsize() ， q.full() ， q.empty() 等实用方法可以获取一个队列的当前大小和状态。但要注意，这些方法都不是线程安全的。可能你对一个队列使用 empty() 判断出这个队列为空，但同时另外一个线程可能已经向这个队列中插入一个数据项。

## 参考

-   [**_python3-cookbook_** Chapter 12 'Concurrency-Starting and Stopping Threads'](https://python3-cookbook.readthedocs.io/zh_CN/latest/c12/p01_start_stop_thread.html)

-   [**_Practical threaded programming with Python_**](https://www.ibm.com/developerworks/aix/library/au-threadingpython/)


***
Sync From: https://github.com/TheBigFish/blog/issues/5
