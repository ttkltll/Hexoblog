---
title: '阅读flask源码2：Local,LocalStack'
date: 2020-04-09 07:18:02
tags: flask
Categories: 阅读flask源码
---

我们还是以flask0.1的代码来阅读,先提出几个常见的问题：

1上下文是怎么被压入栈的？

2为什么在不同的程序中通过相同的变量request可以拿到对应的请求

我们先看第一个问题：上下文是怎么被压入栈的？

服务器传过来的请求参数，被封装成了一个_RequestContext对象，这个对象里有这个请求相关联的一组互相“绑架”的参数，它们组成一个上下文环境。比如request,被实例化的app。如下：

```python
class _RequestContext(object):
   

    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None

    def __enter__(self):
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        # do not pop the request stack if we are in debug mode and an
        # exception happened.  This will allow the debugger to still
        # access the request object in the interactive shell.
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()

```

这个对象实现了__enter__,所以通过with context语句。对象被压入到一个栈中。

```python
 def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, "stack", None) # 如果没有stack,那么就返回none,然后给_local创建一个stack属性。
        if rv is None:
            self._local.stack = rv = [] # 新建一个列表，把这个列表给自己的属性_local,它是一个对象。
        rv.append(obj) # 给自己的属性_local的属性，加上一个上下文环境对象，这个对象在请求进来时已经通过with实例化了一个。
        return rv
```

字典的键是线程Id,值就是被压入的，封装了特定请求的上下文对象。那么问题来了，正常情况下，context被压入栈中，栈里应该是context对象啊。是怎么实现里面有字典的呢？



压入栈中，这个压入，可不是压入一个普通的列表，这个列表，将做为Local对象的一个属性stack

这个LocalStack初始化的时候，封装了一个Local对象。为什么要这样做呢？这就回答了上面的问题，Local可以实现线程隔离，拿到当前线程的上下文。

这里Local实现了一个__setattr__方法，这样可以加上线程了，现在上下文按放在一个Local对象的字典中，



上面是存储好了上下文环境 ，下面是讲解，怎么通过全局变量拿到request的呢？

我们进入视图函数了，我们输入：request.url。

```python
@app.route('/')
def index():
    req = request.url
    return req
```

这个时候，request是一个代理对象，

```python
request = LocalProxy(lambda: _request_ctx_stack.top.request)
```

 _request_ctx_stack.top.request)会从栈中取出最上面的对象，并且拿到栈最上面的上下文对象。

它本身没有url属性，所以会触发LocalProxy的__getattr__属性。这个时候，拿到request对象的url属性。















