---
title: '阅读flask源码3：仿写Local,LocalStack'
date: 2020-04-16 09:46:06
Tags: flask
categories: 阅读flask源码
---

我们上篇分析了上下文压入栈，出栈的大致流程，现在我们要问，flask是怎么实现的呢？准备来说，Local是怎么管理不同请求对象的，LocalProxy是怎么代理的。因为flask源码太复杂，今天我们自己一步一步实现类似于Local,LocalProxy的代码。通过自己实现，我们学会用代理模式，学会python的魔法方法__setattr__,__getattr__。



```python
class A(object):
    def spam(self, x):
        print('A.spam')
    def foo(self):
        print('A.foo')

# 简单的代理
class AProxy(object):

    def __init__(self):
        self._a = A()
    
    def spam(self,x):
        print('B1.spam')
        return self._a.spam(x)
    
    def foo(self):
        print('B1.foo')
        return self._a.foo()
    
    def bar(self):
        print('B1.bar')

b1 = AProxy()
b1.spam(1)
b1.foo()
b1.bar()
```

这里的b1,是一个对象，我们访问b1.spam(1),它会返回类A的一个实例的方法。b1.foo（）也同理。

这个代理的问题是，我们写代理的时候，要先知道被代理的类A有哪些方法，下面我们实现一种更通用的代理。

```python
class A(object):
    def spam(self, x):
        print('A.spam')
    def foo(self):
        print('A.foo')

# 简单的代理
class AProxy(object):

    def __init__(self):
        self._a = A()
        self.name = ''
    def __getattr__(self,name):
      	return getattr(self._a,key)
    def bar(self):
        print('B1.bar')

b1 = AProxy()
b1.spam(1)
b1.foo()
b1.bar()
```

上面这个代理，比第一个就抽象了，这晨用__getattr__实现了什么呢？只要你访问AProxy没有的方法，它就会去__getattr__中找，这个函数实现了你给一个函数名，它会给你被代理对象的函数。这里是一个抽象。

上面这个函数，只是实现了属性，方法的访问，却不能给它赋值。

```python
class A(object):
  	def __init__(self):
      	self.log = ''
    def spam(self, x):
        print('A.spam')
    def foo(self):
        print('A.foo')

# 简单的代理
class AProxy(object):

    def __init__(self):
        self._a = A()
        self.name = ''
    def __getattr__(self,name):
      	return getattr(self._a,key)
    def __setattr__(self,name,value):
      	return setattr(self._a,name,value)
    def bar(self):
        print('B1.bar')

b1 = AProxy()
b1.name = '我是AProxy的一个实例'
b1.log = '我是A的一个实例'
```

这个代理就比前一个更好用了，因为它可以给A的实例间接赋值。我们调用b1.log,这个时候AProxy没有这个属性，这时就去__setattr__中找，把log,'我是A的一个实例'作为实参，传给__setattr__。

当然这个代理也有些问题，就是它还不够抽象，它现在只能代理类A,如果我们希望它还能代理其它类怎么办呢？

改进如下：

```python
class A(object):
  	def __init__(self):
      	self.log = ''
    def spam(self, x):
        print('A.spam')
    def foo(self):
        print('A.foo')

# 简单的代理
class AProxy(object):

    def __init__(self，object):
        self._a = object
        self.name = ''
    def __getattr__(self,name):
      	return getattr(self._a,key)
    def __setattr__(self,name,value):
      	return setattr(self._a,name,value)
    def bar(self):
        print('B1.bar')
a1 = A
b1 = AProxy(a1)
b1.name = '我是AProxy的一个实例'
b1.log = '我是A的一个实例'
```

这里通过一个给AProxy传参初始化，可以拿到不同的实例代理。

有了上面这些基础，现在我们模仿Local,LocalProxy来写一个代理：

```python
class Request(object):

    def __init__(self):
        self.url = 'tanliang.com'

class User(object):

    def __init__(self):
        self.owner = 'tanliang'

class Local:
    def __init__(self):
        self._objs = {}
        self._objs['request'] = Request()
        self._objs['user'] = User()

    def __getattr__(self, name):
        print('Local.__getattr__')
        return self._objs[name]

    def __call__(self, name):
        return LocalProxy(self, name)

class LocalProxy(object):

    def __init__(self, local, name):
        self._local = local
        self._name = name

    def _get_current_obj(self):
        print('_get_current_obj')
        return getattr(self._local, self._name)

    def __getattr__(self, name):
        print('SpamProxy.__getattr__')
        return getattr(self._get_current_obj(), name)
    
    def __setattr__(self, name ,value):
        if name.startswith('_'):
            super().__setattr__(name, value)
        else:
            setattr(self._obj, name, value)
    
    def __delattr__(self, name):
        print('SpamProxy.__delattr__')
        if name.startswith('_'):
            super().__delattr__(name)
        else:
            print('SpamProxy', name)
            delattr(self._obj, name)

l = Local()

request = l('request')
print(request)
print(request.url)

user = l('user') 
print(user.owner)
```

这里l是一个Local实例，request是一个代理对象，LocalProxy的实例，因为Local实现了__call__方法。当我们访问request.url时，会发生什么呢？先找到__getattr__,然后把url作为参数传给getattr。可是这个时候还没有拿到“当前对象”。self.__get_current_obj()就是拿当前对象的，最后能过Local的__getattr__拿到request对象。

