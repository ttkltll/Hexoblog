---
title: '阅读flask源码1：flask请求流程'
date: 2020-04-04 10:56:15
tags: flask 
Categories: 阅读flask源码
---

> 我们先大致的走下flask处理请求的流程，我用的是flask0.1这个版本的源码。之所以用这个，是为了去除不必要枝叶，快速把握flask的主干。后面再迭代，加细节。

```python
from flask import Flask, Request, request, session, flash, abort, _request_ctx_stack

from user import web

app = Flask(__name__)

web.register(app)

#下面会注册路由：
@app.route('/')
def index():
    req = request.url
    return req

print app.url_map
print app.view_functions
app.run(debug=True, port=8004)
```

当我们运行这个入口文件，先实例化一个app对象，后面执行视图函数的路由注册。这个时候，app的url_map,view_function已经有值了。然后运行app.run()。我们进入run()方法内部：

```python
    def run(self, host='localhost', port=5000, **options):
      
        from werkzeug import run_simple
        if 'debug' in options:
            self.debug = options.pop('debug')
        options.setdefault('use_reloader', self.debug)
        options.setdefault('use_debugger', self.debug)
        return run_simple(host, port, self, **options)
```



这个run()方法，会返回run_simple(),这个方法是werkzeug模块里的，会启动一个服务器，run()方法把这个app的实例作为参数传给了run_simple。

服务器会监听浏览器传过来的信息，接着服务器把信息environ,start_response传给WSGI application,也就是实例化的app。environ包括所有的请求信息，start_response是 application 处理完之后需要调用的函数，参数是状态码、响应头部还有错误信息。

服务器端会调用__call__方法，__call__方法返回app自己的wsgi_app()方法：

```python
    def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`"""
        return self.wsgi_app(environ, start_response)
```

我们进入wigi_app内部看下，这里就是WSGI application处理请求的主干了！

```python
    def wsgi_app(self, environ, start_response):
    
        with self.request_context(environ):
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
            response = self.make_response(rv)
            response = self.process_response(response)
            return response(environ, start_response)

```



这些代码做了什么事呢？

先用environ封装一个请求上下文对象，并且把这个上下文压入到栈中。

```python
with self.request_context(environ):
```



为什么这个语句可以做到这件事呢？是因为request_context（）方法会返回一个_RequestContext对象，而这个对象实现了__enter__方法。这里不展开。然后是做调用preprocess_request()函数做一些预处理。现在还没有进入视图函数。然后才是调用dispatch_request()函数，这个函数做什么呢？主要是分发路由，找到匹配的视图函数来处理请求。把结果返回给rv变量。再接着调用make_response()函数对rv封装，把它转换成一个符合格式的Response对象。

```python
            response = self.process_response(response)

```

这句会对response做进一步处理。主要是调用after_request_funcs列表里的函数。这个列表，由一些函数组成。这些函数可以通过after_request（）注册。最后返回return response(environ, start_response)。respsose可以被调用，是因为response是一个实现了__call__方法的类，到此，整个流程已经走完了。



> 先把流程走通了，以后再阅读下蓝图模块，错误处理模块，请求上下文模块的源码





