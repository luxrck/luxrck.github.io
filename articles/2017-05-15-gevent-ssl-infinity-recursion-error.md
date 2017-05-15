---
title: gevent与requests混用时出现的ssl无限递归错误
date: 2017-05-15 16:35
category: debug
tags: gevent, ssl
---

### environ
* os: archlinux
* python: 3.6.1
* gevent: 1.2.1
* requests: 2.14.2


### reference
* [SSLContext infinite recursion in Python 3.6 (gevent#903)](https://github.com/gevent/gevent/issues/903)


### reproduce
`vim test.py` :
```python
import gevent.monkey
from requests.packages.urllib3.util.ssl_ import create_urllib3_context

# 将monkey patch置于create_urllib3_context引入之前可防止此错误出现。
gevent.monkey.patch_all()

create_urllib3_context()
# output:
# Traceback (most recent call last):
#   File "test.py", line 7, in <module>
#     create_urllib3_context()
#   File "/usr/lib/python3.6/site-packages/requests/packages/urllib3/util/ssl_.py", line 265, in create_urllib3_context
#     context.options |= options
#   File "/usr/lib/python3.6/ssl.py", line 459, in options
#     super(SSLContext, SSLContext).options.__set__(self, value)
#   File "/usr/lib/python3.6/ssl.py", line 459, in options
#     super(SSLContext, SSLContext).options.__set__(self, value)
#   File "/usr/lib/python3.6/ssl.py", line 459, in options
#     super(SSLContext, SSLContext).options.__set__(self, value)
#   [Previous line repeated 329 more times]
# RecursionError: maximum recursion depth exceeded while calling a Python object
```


### debug
`create_urllib3_context`代码片段：
```python
...
# requests/packages/urllib3/util/ssl_.py#L250

context = SSLContext(ssl_version or ssl.PROTOCOL_SSLv23)

# Setting the default here, as we may have no ssl module on import
cert_reqs = ssl.CERT_REQUIRED if cert_reqs is None else cert_reqs

if options is None:
    options = 0
    # SSLv2 is easily broken and is considered harmful and dangerous
    options |= OP_NO_SSLv2
    # SSLv3 has several problems and is now dangerous
    options |= OP_NO_SSLv3
    # Disable compression to prevent CRIME attacks for OpenSSL 1.0+
    # (issue #309)
    options |= OP_NO_COMPRESSION

context.options |= options
...
```
这里的`SSLContext`的类型为`ssl.SSLContext'`,而不是`gevent._ssl3.SSLContext`。
即`gevent.monkey.patch_all`对其没起作用。

查看`SSLContext`相关代码：
```python
...
# requests/packages/urllib3/util/ssl_.py#L86

try:
    from ssl import SSLContext  # Modern SSL?
except ImportError:
    import sys

    class SSLContext(object):  # Platform-specific: Python 2 & 3.1
        supports_set_ciphers = ((2, 7) <= sys.version_info < (3,) or
                                (3, 2) <= sys.version_info)
...
```
`from ssl import SSLContext`是`gevent.monkey.patch_all`失效的原因所在。
修正为：
```python
import ssl
SSLContext = lambda *args, **kw: ssl.SSLContext(*args, **kw)
```
当然最好的方式还是在引入`gevent`库后立即执行`monkey patch`代码。


### analysis
`gevent.monkey.patch_all`的位置放错导致了`create_urllib3_context`中的`context`
的类型为`ssl.SSLContext`而不是`gevent._ssl3.SSLContext`。

因此，`context.options |= options`最后会调用：
```python
# ssl.py#L457

@options.setter
def options(self, value):
    super(SSLContext, SSLContext).options.__set__(self, value)
```
而不是：
```python
# gevent/_ssl3.py#L67

# In 3.6, these became properties. They want to access the
# property __set__ method in the superclass, and they do so by using
# super(SSLContext, SSLContext). But we rebind SSLContext when we monkey
# patch, which causes infinite recursion.
# https://github.com/python/cpython/commit/328067c468f82e4ec1b5c510a4e84509e010f296
# pylint:disable=no-member
@orig_SSLContext.options.setter
def options(self, value):
    super(orig_SSLContext, orig_SSLContext).options.__set__(self, value)
```
但是，**由于已经调用过了`monkey patch`代码**，故此时的`SSLContext`实际上是`gevent._ssl3.SSLContext`，
是`ssl.SSLContext`的子类。

所以，`super(SSLContext, SSLContext).options`实际上是`super(gevent._ssl3.SSLContext, gevent._ssl3.SSLContext).options`，
得到的结果是`ssl.SSLContext.options`，而不是我们所希望的`_ssl._SSLContext.options`。

同时，这也造成了无限递归。
