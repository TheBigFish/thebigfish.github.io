---
title: tornado源码之iostream
date: 2018-12-20 16:46:51
updated: 2018-12-20 16:46:51
tags: [python]
author: TheBigFish
---
# iostream.py

A utility class to write to and read from a non-blocking socket.

IOStream 对 socket 进行包装，采用注册回调方式实现非阻塞。  
通过接口注册各个事件回调

-   \_read_callback
-   \_write_callback
-   \_close_callback
-   \_connect_callback

ioloop 中 socket 事件发生后，调用 IOStream.\_handle_events 方法，对事件进行分发。  
对应的事件处理过程中，如果满足注册的回调条件，则调用回调函数  
回调函数在 IOStream.\_handle_events 中被调用

## contents

-   [iostream.py](#iostreampy)
    -   [contents](#contents)
    -   [example](#example)
    -   [head](#head)
    -   [IOStream.\_\_init\_\_](#iostreaminit)
    -   [IOStream.connect](#iostreamconnect)
    -   [IOStream.read_until](#iostreamreaduntil)
    -   [IOStream.read_bytes](#iostreamreadbytes)
    -   [IOStream.write](#iostreamwrite)
    -   [IOStream.close](#iostreamclose)
    -   [IOStream.\_handle_events](#iostreamhandleevents)
    -   [IOStream.\_run_callback](#iostreamruncallback)
    -   [IOStream.\_run_callback](#iostreamruncallback-1)
    -   [IOStream.\_read_from_socket](#iostreamreadfromsocket)
    -   [IOStream.\_read_to_buffer](#iostreamreadtobuffer)
    -   [IOStream.\_read_from_buffer](#iostreamreadfrombuffer)
    -   [IOStream.\_handle_connect](#iostreamhandleconnect)
    -   [IOStream.\_handle_write](#iostreamhandlewrite)
    -   [IOStream.\_consume](#iostreamconsume)
    -   [IOStream.\_add_io_state](#iostreamaddiostate)
    -   [IOStream.\_read_buffer_size](#iostreamreadbuffersize)
    -   [copyright](#copyright)

## example

一个简单的 IOStream 客户端示例  
**由此可见， IOStream 是一个异步回调链**

1.  创建 socket
2.  创建 IOStream 对象
3.  连接到主机，传入连接成功后回调函数 send_request
4.  socket 输出数据请求页面，读取 head, 传入读取 head 成功后回调函数 on_headers
5.  继续读取 body, 传入读取 body 成功后回调函数 on_body
6.  关闭 stream，关闭 ioloop

```python
from tornado import ioloop
from tornado import iostream
import socket


def send_request():
    stream.write("GET / HTTP/1.0\r\nHost: baidu.com\r\n\r\n")
    stream.read_until("\r\n\r\n", on_headers)


def on_headers(data):
    headers = {}
    for line in data.split("\r\n"):
        parts = line.split(":")
        if len(parts) == 2:
            headers[parts[0].strip()] = parts[1].strip()
    stream.read_bytes(int(headers["Content-Length"]), on_body)


def on_body(data):
    print data
    stream.close()
    ioloop.IOLoop.instance().stop()


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
stream = iostream.IOStream(s)
stream.connect(("baidu.com", 80), send_request)
ioloop.IOLoop.instance().start()


# html>
# <meta http-equiv="refresh" content="0;        url=http://www.baidu.com/">
# </html>
```

## head

```python
from __future__ import with_statement

import collections
import errno
import logging
import socket
import sys

from tornado import ioloop
from tornado import stack_context

try:
    import ssl # Python 2.6+
except ImportError:
    ssl = None
```

## IOStream.\_\_init\_\_

包装 socket 类  
关键语句 `self.io_loop.add_handler(self.socket.fileno(), self._handle_events, self._state)` 将自身的\_handle_events 加入到全局 ioloop poll 事件回调  
此时只注册了 ERROR 类型事件

\_read_buffer: 读缓冲

```python
class IOStream(object):

    def __init__(self, socket, io_loop=None, max_buffer_size=104857600,
                 read_chunk_size=4096):
        self.socket = socket
        self.socket.setblocking(False)
        self.io_loop = io_loop or ioloop.IOLoop.instance()
        self.max_buffer_size = max_buffer_size
        self.read_chunk_size = read_chunk_size
        self._read_buffer = collections.deque()
        self._write_buffer = collections.deque()
        self._write_buffer_frozen = False
        self._read_delimiter = None
        self._read_bytes = None
        self._read_callback = None
        self._write_callback = None
        self._close_callback = None
        self._connect_callback = None
        self._connecting = False
        self._state = self.io_loop.ERROR
        with stack_context.NullContext():
            self.io_loop.add_handler(
                self.socket.fileno(), self._handle_events, self._state)
```

## IOStream.connect

连接 socket 到远程地址，非阻塞模式

1.  连接 socket
2.  注册连接完成回调
3.  poll 增加 socket 写事件

```python
    def connect(self, address, callback=None):
        """Connects the socket to a remote address without blocking.

        May only be called if the socket passed to the constructor was
        not previously connected.  The address parameter is in the
        same format as for socket.connect, i.e. a (host, port) tuple.
        If callback is specified, it will be called when the
        connection is completed.

        Note that it is safe to call IOStream.write while the
        connection is pending, in which case the data will be written
        as soon as the connection is ready.  Calling IOStream read
        methods before the socket is connected works on some platforms
        but is non-portable.
        """
        self._connecting = True
        try:
            self.socket.connect(address)
        except socket.error, e:
            # In non-blocking mode connect() always raises an exception
            if e.args[0] not in (errno.EINPROGRESS, errno.EWOULDBLOCK):
                raise
        self._connect_callback = stack_context.wrap(callback)
        self._add_io_state(self.io_loop.WRITE)
```

## IOStream.read_until

1.  注册读完成回调
2.  尝试从缓冲中读
3.  从 socket 中读到缓冲区
4.  重复 2,3, 没有数据则退出
5.  将 socket 读事件加入 poll

如果缓存中数据满足条件，则直接执行 callback 并返回，  
否则，保存 callback 函数下次 read 事件发生时，\_handle_events 处理读事件时，再进行检测及调用

```python
    def read_until(self, delimiter, callback):
        """Call callback when we read the given delimiter."""
        assert not self._read_callback, "Already reading"
        self._read_delimiter = delimiter
        self._read_callback = stack_context.wrap(callback)
        while True:
            # See if we've already got the data from a previous read
            if self._read_from_buffer():
                return
            self._check_closed()
            if self._read_to_buffer() == 0:
                break
        self._add_io_state(self.io_loop.READ)
```

## IOStream.read_bytes

参考 read_until，读限定字节

```python
    def read_bytes(self, num_bytes, callback):
        """Call callback when we read the given number of bytes."""
        assert not self._read_callback, "Already reading"
        if num_bytes == 0:
            callback("")
            return
        self._read_bytes = num_bytes
        self._read_callback = stack_context.wrap(callback)
        while True:
            if self._read_from_buffer():
                return
            self._check_closed()
            if self._read_to_buffer() == 0:
                break
        self._add_io_state(self.io_loop.READ)
```

## IOStream.write

```python
    def write(self, data, callback=None):
        """Write the given data to this stream.

        If callback is given, we call it when all of the buffered write
        data has been successfully written to the stream. If there was
        previously buffered write data and an old write callback, that
        callback is simply overwritten with this new callback.
        """
        self._check_closed()
        self._write_buffer.append(data)
        self._add_io_state(self.io_loop.WRITE)
        self._write_callback = stack_context.wrap(callback)

    def set_close_callback(self, callback):
        """Call the given callback when the stream is closed."""
        self._close_callback = stack_context.wrap(callback)
```

## IOStream.close

1.  从 ioloop 移除 socket 事件
2.  关闭 socket
3.  调用关闭回调

```python
    def close(self):
        """Close this stream."""
        if self.socket is not None:
            self.io_loop.remove_handler(self.socket.fileno())
            self.socket.close()
            self.socket = None
            if self._close_callback:
                self._run_callback(self._close_callback)

    def reading(self):
        """Returns true if we are currently reading from the stream."""
        return self._read_callback is not None

    def writing(self):
        """Returns true if we are currently writing to the stream."""
        return bool(self._write_buffer)

    def closed(self):
        return self.socket is None
```

## IOStream.\_handle_events

核心回调  
任何类型的 socket 事件触发 ioloop 回调\_handle_events，然后在\_handle_events 再进行分发  
值得注意的是，IOStream 不处理连接请求的 read 事件  
**注意**  
作为服务端，默认代理的是已经建立连接的 socket

```python
# HTTPServer.\_handle_events
# connection 为已经accept的连接
stream = iostream.IOStream(connection, io_loop=self.io_loop)
```

作为客户端，需要手动调用 IOStream.connect，连接成功后，成功回调在 write 事件中处理

这个实现比较别扭

```python
    def _handle_events(self, fd, events):
        if not self.socket:
            logging.warning("Got events for closed stream %d", fd)
            return
        try:
            # 处理读事件，调用已注册回调
            if events & self.io_loop.READ:
                self._handle_read()
            if not self.socket:
                return
            # 处理写事件，如果是刚建立连接，调用连接建立回调
            if events & self.io_loop.WRITE:
                if self._connecting:
                    self._handle_connect()
                self._handle_write()
            if not self.socket:
                return
            # 错误事件，关闭 socket
            if events & self.io_loop.ERROR:
                self.close()
                return
            state = self.io_loop.ERROR
            if self.reading():
                state |= self.io_loop.READ
            if self.writing():
                state |= self.io_loop.WRITE
            if state != self._state:
                self._state = state
                self.io_loop.update_handler(self.socket.fileno(), self._state)
        except:
            logging.error("Uncaught exception, closing connection.",
                          exc_info=True)
            self.close()
            raise
```

## IOStream.\_run_callback

执行回调

```python
    def _run_callback(self, callback, *args, **kwargs):
        try:
            # Use a NullContext to ensure that all StackContexts are run
            # inside our blanket exception handler rather than outside.
            with stack_context.NullContext():
                callback(*args, **kwargs)
        except:
            logging.error("Uncaught exception, closing connection.",
                          exc_info=True)
            # Close the socket on an uncaught exception from a user callback
            # (It would eventually get closed when the socket object is
            # gc'd, but we don't want to rely on gc happening before we
            # run out of file descriptors)
            self.close()
            # Re-raise the exception so that IOLoop.handle_callback_exception
            # can see it and log the error
            raise
```

## IOStream.\_run_callback

读回调

1.  从 socket 读取数据到缓存
2.  无数据, socket 关闭
3.  检测是否满足 read_until read_bytes
4.  满足则执行对应回调

```python
    def _handle_read(self):
        while True:
            try:
                # Read from the socket until we get EWOULDBLOCK or equivalent.
                # SSL sockets do some internal buffering, and if the data is
                # sitting in the SSL object's buffer select() and friends
                # can't see it; the only way to find out if it's there is to
                # try to read it.
                result = self._read_to_buffer()
            except Exception:
                self.close()
                return
            if result == 0:
                break
            else:
                if self._read_from_buffer():
                    return
```

## IOStream.\_read_from_socket

从 socket 读取数据

```python
    def _read_from_socket(self):
        """Attempts to read from the socket.

        Returns the data read or None if there is nothing to read.
        May be overridden in subclasses.
        """
        try:
            chunk = self.socket.recv(self.read_chunk_size)
        except socket.error, e:
            if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                return None
            else:
                raise
        if not chunk:
            self.close()
            return None
        return chunk
```

## IOStream.\_read_to_buffer

从 socket 读取数据存入缓存

```python
    def _read_to_buffer(self):
        """Reads from the socket and appends the result to the read buffer.

        Returns the number of bytes read.  Returns 0 if there is nothing
        to read (i.e. the read returns EWOULDBLOCK or equivalent).  On
        error closes the socket and raises an exception.
        """
        try:
            chunk = self._read_from_socket()
        except socket.error, e:
            # ssl.SSLError is a subclass of socket.error
            logging.warning("Read error on %d: %s",
                            self.socket.fileno(), e)
            self.close()
            raise
        if chunk is None:
            return 0
        self._read_buffer.append(chunk)
        if self._read_buffer_size() >= self.max_buffer_size:
            logging.error("Reached maximum read buffer size")
            self.close()
            raise IOError("Reached maximum read buffer size")
        return len(chunk)
```

## IOStream.\_read_from_buffer

从缓冲中过滤数据  
检测是否满足结束条件 (read_until/read_bytes)，满足则调用之前注册的回调  
采用的是查询方式

```python
    def _read_from_buffer(self):
        """Attempts to complete the currently-pending read from the buffer.

        Returns True if the read was completed.
        """
        if self._read_bytes:
            if self._read_buffer_size() >= self._read_bytes:
                num_bytes = self._read_bytes
                callback = self._read_callback
                self._read_callback = None
                self._read_bytes = None
                self._run_callback(callback, self._consume(num_bytes))
                return True
        elif self._read_delimiter:
            _merge_prefix(self._read_buffer, sys.maxint)
            loc = self._read_buffer[0].find(self._read_delimiter)
            if loc != -1:
                callback = self._read_callback
                delimiter_len = len(self._read_delimiter)
                self._read_callback = None
                self._read_delimiter = None
                self._run_callback(callback,
                                   self._consume(loc + delimiter_len))
                return True
        return False
```

## IOStream.\_handle_connect

调用连接建立回调，并清除连接中标志

```python
    def _handle_connect(self):
        if self._connect_callback is not None:
            callback = self._connect_callback
            self._connect_callback = None
            self._run_callback(callback)
        self._connecting = False
```

## IOStream.\_handle_write

写事件

1.  从缓冲区获取限定范围内数据
2.  调用 socket.send 输出数据
3.  如果数据发送我且已注册回调，调用发送完成回调

```python
    def _handle_write(self):
        while self._write_buffer:
            try:
                if not self._write_buffer_frozen:
                    # On windows, socket.send blows up if given a
                    # write buffer that's too large, instead of just
                    # returning the number of bytes it was able to
                    # process.  Therefore we must not call socket.send
                    # with more than 128KB at a time.
                    _merge_prefix(self._write_buffer, 128 * 1024)
                num_bytes = self.socket.send(self._write_buffer[0])
                self._write_buffer_frozen = False
                _merge_prefix(self._write_buffer, num_bytes)
                self._write_buffer.popleft()
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    # With OpenSSL, after send returns EWOULDBLOCK,
                    # the very same string object must be used on the
                    # next call to send.  Therefore we suppress
                    # merging the write buffer after an EWOULDBLOCK.
                    # A cleaner solution would be to set
                    # SSL_MODE_ACCEPT_MOVING_WRITE_BUFFER, but this is
                    # not yet accessible from python
                    # (http://bugs.python.org/issue8240)
                    self._write_buffer_frozen = True
                    break
                else:
                    logging.warning("Write error on %d: %s",
                                    self.socket.fileno(), e)
                    self.close()
                    return
        if not self._write_buffer and self._write_callback:
            callback = self._write_callback
            self._write_callback = None
            self._run_callback(callback)
```

## IOStream.\_consume

从读缓存消费 loc 长度的数据

```python
    def _consume(self, loc):
        _merge_prefix(self._read_buffer, loc)
        return self._read_buffer.popleft()

    def _check_closed(self):
        if not self.socket:
            raise IOError("Stream is closed")
```

## IOStream.\_add_io_state

增加 socket 事件状态

```python
    def _add_io_state(self, state):
        if self.socket is None:
            # connection has been closed, so there can be no future events
            return
        if not self._state & state:
            self._state = self._state | state
            self.io_loop.update_handler(self.socket.fileno(), self._state)
```

## IOStream.\_read_buffer_size

获取读缓存中已有数据长度

```python
    def _read_buffer_size(self):
        return sum(len(chunk) for chunk in self._read_buffer)


class SSLIOStream(IOStream):
    """A utility class to write to and read from a non-blocking socket.

    If the socket passed to the constructor is already connected,
    it should be wrapped with
        ssl.wrap_socket(sock, do_handshake_on_connect=False, **kwargs)
    before constructing the SSLIOStream.  Unconnected sockets will be
    wrapped when IOStream.connect is finished.
    """
    def __init__(self, *args, **kwargs):
        """Creates an SSLIOStream.

        If a dictionary is provided as keyword argument ssl_options,
        it will be used as additional keyword arguments to ssl.wrap_socket.
        """
        self._ssl_options = kwargs.pop('ssl_options', {})
        super(SSLIOStream, self).__init__(*args, **kwargs)
        self._ssl_accepting = True
        self._handshake_reading = False
        self._handshake_writing = False

    def reading(self):
        return self._handshake_reading or super(SSLIOStream, self).reading()

    def writing(self):
        return self._handshake_writing or super(SSLIOStream, self).writing()

    def _do_ssl_handshake(self):
        # Based on code from test_ssl.py in the python stdlib
        try:
            self._handshake_reading = False
            self._handshake_writing = False
            self.socket.do_handshake()
        except ssl.SSLError, err:
            if err.args[0] == ssl.SSL_ERROR_WANT_READ:
                self._handshake_reading = True
                return
            elif err.args[0] == ssl.SSL_ERROR_WANT_WRITE:
                self._handshake_writing = True
                return
            elif err.args[0] in (ssl.SSL_ERROR_EOF,
                                 ssl.SSL_ERROR_ZERO_RETURN):
                return self.close()
            elif err.args[0] == ssl.SSL_ERROR_SSL:
                logging.warning("SSL Error on %d: %s", self.socket.fileno(), err)
                return self.close()
            raise
        except socket.error, err:
            if err.args[0] == errno.ECONNABORTED:
                return self.close()
        else:
            self._ssl_accepting = False
            super(SSLIOStream, self)._handle_connect()

    def _handle_read(self):
        if self._ssl_accepting:
            self._do_ssl_handshake()
            return
        super(SSLIOStream, self)._handle_read()

    def _handle_write(self):
        if self._ssl_accepting:
            self._do_ssl_handshake()
            return
        super(SSLIOStream, self)._handle_write()

    def _handle_connect(self):
        self.socket = ssl.wrap_socket(self.socket,
                                      do_handshake_on_connect=False,
                                      **self._ssl_options)
        # Don't call the superclass's _handle_connect (which is responsible
        # for telling the application that the connection is complete)
        # until we've completed the SSL handshake (so certificates are
        # available, etc).


    def _read_from_socket(self):
        try:
            # SSLSocket objects have both a read() and recv() method,
            # while regular sockets only have recv().
            # The recv() method blocks (at least in python 2.6) if it is
            # called when there is nothing to read, so we have to use
            # read() instead.
            chunk = self.socket.read(self.read_chunk_size)
        except ssl.SSLError, e:
            # SSLError is a subclass of socket.error, so this except
            # block must come first.
            if e.args[0] == ssl.SSL_ERROR_WANT_READ:
                return None
            else:
                raise
        except socket.error, e:
            if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                return None
            else:
                raise
        if not chunk:
            self.close()
            return None
        return chunk

def _merge_prefix(deque, size):
    """Replace the first entries in a deque of strings with a single
    string of up to size bytes.

    >>> d = collections.deque(['abc', 'de', 'fghi', 'j'])
    >>> _merge_prefix(d, 5); print d
    deque(['abcde', 'fghi', 'j'])

    Strings will be split as necessary to reach the desired size.
    >>> _merge_prefix(d, 7); print d
    deque(['abcdefg', 'hi', 'j'])

    >>> _merge_prefix(d, 3); print d
    deque(['abc', 'defg', 'hi', 'j'])

    >>> _merge_prefix(d, 100); print d
    deque(['abcdefghij'])
    """
    prefix = []
    remaining = size
    while deque and remaining > 0:
        chunk = deque.popleft()
        if len(chunk) > remaining:
            deque.appendleft(chunk[remaining:])
            chunk = chunk[:remaining]
        prefix.append(chunk)
        remaining -= len(chunk)
    deque.appendleft(''.join(prefix))

def doctests():
    import doctest
    return doctest.DocTestSuite()
```

## copyright

author：bigfish  
copyright: [许可协议 知识共享署名 - 非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)


***
Sync From: https://github.com/TheBigFish/blog/issues/8
