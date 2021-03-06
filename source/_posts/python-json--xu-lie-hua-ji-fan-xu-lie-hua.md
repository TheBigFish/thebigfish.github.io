---
title: python json 序列化及反序列化
date: 2018-11-01 16:10:50
updated: 2018-11-01 16:10:50
tags: [python]
author: TheBigFish
---
# python json 序列化及反序列化

<!-- TOC -->

-   [python json 序列化及反序列化](#python-json-%E5%BA%8F%E5%88%97%E5%8C%96%E5%8F%8A%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96)
    -   [使用`namedtuple`](#%E4%BD%BF%E7%94%A8namedtuple)
    -   [使用`object_hook`](#%E4%BD%BF%E7%94%A8objecthook)
    -   [获取对象属性](#%E8%8E%B7%E5%8F%96%E5%AF%B9%E8%B1%A1%E5%B1%9E%E6%80%A7)
    -   [获取对象的嵌套属性](#%E8%8E%B7%E5%8F%96%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%B5%8C%E5%A5%97%E5%B1%9E%E6%80%A7)

<!-- /TOC -->

## 使用`namedtuple`

反序列化为 namedtuple

```python
import json
from collections import namedtuple

data = '{"name": "John Smith", "hometown": {"name": "New York", "id": 123}}'

# Parse JSON into an object with attributes corresponding to dict keys.
x = json.loads(data, object_hook=lambda d: namedtuple('X', d.keys())(*d.values()))
print x.name, x.hometown.name, x.hometown.id
```

序列化为 json

```python
json.dumps(x._asdict())
```

输出

```shell
{"hometown": ["New York", 123], "name": "John Smith"}
```

封装：

```python
def _json_object_hook(d): return namedtuple('X', d.keys())(*d.values())
def json2obj(data): return json.loads(data, object_hook=_json_object_hook)

x = json2obj(data)
```

总结：

序列化及反序列化都比较方便，但是 `namedtuple` 不能进行复制，不能修改

## 使用`object_hook`

反序列化为对象

```python
class JSONObject:
    def __init__(self, d):
        self.__dict__ = d

data = '{"name": "John Smith", "hometown": {"name": "New York", "id": 123}}'

a = json.loads(data,
               object_hook=JSONObject)

a.name = "changed"
print a.name
```

## 获取对象属性

-   使用 `getattr`

```python
print getattr(a.hometown, 'id', 321)
# 123
print getattr(a.hometown, 'id1', 321)
# 321
```

-   使用 `try`

```python
try:
    print a.hometown.id2
except AttributeError as ex:
    print ex
```

-   使用 `get`

```python
x = data.get('first', {}).get('second', {}).get('third', None)
```

## 获取对象的嵌套属性

```python
def multi_getattr(obj, attr, default = None):
    """
    Get a named attribute from an object; multi_getattr(x, 'a.b.c.d') is
    equivalent to x.a.b.c.d. When a default argument is given, it is
    returned when any attribute in the chain doesn't exist; without
    it, an exception is raised when a missing attribute is encountered.

    """
    attributes = attr.split(".")
    for i in attributes:
        try:
            obj = getattr(obj, i)
        except AttributeError:
            if default:
                return default
            else:
                raise
    return obj

print multi_getattr(a, "hometown.name")
# New York
print multi_getattr(a, "hometown.name1", "abc")
# abc
```

```python
# coding=utf-8
from __future__ import unicode_literals
import collections
import operator

_default_stub = object()


def deep_get(obj, path, default=_default_stub, separator='.'):
    """Gets arbitrarily nested attribute or item value.

    Args:
        obj: Object to search in.
        path (str, hashable, iterable of hashables): Arbitrarily nested path in obj hierarchy.
        default: Default value. When provided it is returned if the path doesn't exist.
            Otherwise the call raises a LookupError.
        separator: String to split path by.

    Returns:
        Value at path.

    Raises:
        LookupError: If object at path doesn't exist.

    Examples:
        >>> deep_get({'a': 1}, 'a')
        1

        >>> deep_get({'a': 1}, 'b')
        Traceback (most recent call last):
            ...
        LookupError: {u'a': 1} has no element at 'b'

        >>> deep_get(['a', 'b', 'c'], -1)
        u'c'

        >>> deep_get({'a': [{'b': [1, 2, 3]}, 'some string']}, 'a.0.b')
        [1, 2, 3]

        >>> class A(object):
        ...     def __init__(self):
        ...         self.x = self
        ...         self.y = {'a': 10}
        ...
        >>> deep_get(A(), 'x.x.x.x.x.x.y.a')
        10

        >>> deep_get({'a.b': {'c': 1}}, 'a.b.c')
        Traceback (most recent call last):
            ...
        LookupError: {u'a.b': {u'c': 1}} has no element at 'a'

        >>> deep_get({'a.b': {'Привет': 1}}, ['a.b', 'Привет'])
        1

        >>> deep_get({'a.b': {'Привет': 1}}, 'a.b/Привет', separator='/')
        1

    """
    if isinstance(path, basestring):
        attributes = path.split(separator)
    elif isinstance(path, collections.Iterable):
        attributes = path
    else:
        attributes = [path]

    LOOKUPS = [getattr, operator.getitem, lambda obj, i: obj[int(i)]]
    try:
        for i in attributes:
            for lookup in LOOKUPS:
                try:
                    obj = lookup(obj, i)
                    break
                except (TypeError, AttributeError, IndexError, KeyError,
                        UnicodeEncodeError, ValueError):
                    pass
            else:
                msg = "{obj} has no element at '{i}'".format(obj=obj, i=i)
                raise LookupError(msg.encode('utf8'))
    except Exception:
        if _default_stub != default:
            return default
        raise
    return obj
```


***
Sync From: https://github.com/TheBigFish/blog/issues/2
