---
title: 《流畅的python》阅读笔记2：切片及原理
date: 2020-06-02 11:50:31
tags:
categories: 《流畅的python》
---

1：什么场景下我们会用到切片？

2：实现切片的原理，基于此，我们如何实现一个支持切片操作的自定义类型？



### 下面说下什么场景下我们会用到切片？

当我们想截取一段代码，怎样操作呢？

比如下面：

```python
list = [0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0]
array1 = array('d', list)
components = reprlib.repr(array1)
# 这个时候返回字符串:“array('d', [0.0, 1.0, 2.0, 3.0, 4.0, ...])”
# 现在我们只要[及其后面到]的字符串，这个时候怎么办？
# 最笨的方法：一个一个取值:components[12],components[13]....,或者用for循环。但是python还提供了切片可以更便捷的操作：

components = components[components.find('['):-1]

# 一行代码解决了问题
```

《流畅的python》示例10.2的__repr__就用到切片来组合一个新的“构造方法字符串”。



我们还知道切片是对象，这可以用于什么场景呢？请看下面一段代码与注释

```python
>>> invoice = """
... 0....5...............
... tan  168  male
... guan 160  female
... """


# 上面是一个字符串，现在要取出每一行的第一段和第二段，怎么办？最笨的思路就是按下标取。
>>> name = slice(0, 5)
>>> height = slice(5,10)
>>> gender = slice(10, None)
# 上面给每个slice对象取个名字，方便以后复用。以后直接写名字就可以了，不用记下标。
>>> list = invoice.split('\n')[2:]
>>> for item in list:
...     print(item[name], item[height])
... 
tan   168  
guan  160  

```

### 切片操作的原理是什么呢？

当我们调用list[0:8]的时候，解释器其实是把slice(0,8）这个对象传给list.__getitem__(self,slice),下面来看 Python 如何把 my_seq[1:3] 句法变成传给 my_seq.__getitem__(...) 的参数：



```python
>>> class MySeq:
...     def __getitem__(self, index):
...         return index  
...
>>> s = MySeq()
>>> s[1]  
1
>>> s[1:4]  
slice(1, 4, None)
>>> s[1:4:2]  
slice(1, 4, 2)
>>> s[1:4:2, 9]  
(slice(1, 4, 2), 9)
>>> s[1:4:2, 7:9]  
(slice(1, 4, 2), slice(7, 9, None))
```

__getitem__会根据传入的数据类型做不同的操作，下面我们通过自定义一个类，让它支持切片，来更好的理解：

```python
class Vector:
    typecode = 'd'

    def __init__(self, components):
        self._components = array(self.typecode, components)

    def __iter__(self):
        return iter(self._components)

    def __repr__(self):
        components = reprlib.repr(self._components)
        components = components[components.find('['):-1]
        return 'Vector({})'.format(components)

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) +
                bytes(self._components))

    def __eq__(self, other):
        return tuple(self) == tuple(other)

    def __abs__(self):
        return math.sqrt(sum(x * x for x in self))

    def __bool__(self):
        return bool(abs(self))
        
    def __len__(self):
        return len(self._components)

    def __getitem__(self, index):
        return self._components[index]
```

执行结果如下：

```python
>>> v7 = Vector(range(7))
>>> v7[1:4]
array('d', [1.0, 2.0, 3.0])”
```

这就有问题了，我们得到的切片应该也是Vector类，可现在是数组类型，如何解决呢？这就要用到我们上面说的切片原理了，我们要根据不同的类型，采取不同的手段。

```python
class Vector:
    typecode = 'd'

    def __init__(self, components):
        self._components = array(self.typecode, components)

    def __iter__(self):
        return iter(self._components)

    def __repr__(self):
        components = reprlib.repr(self._components)
        components = components[components.find('['):-1]
        return 'Vector({})'.format(components)

    def __str__(self):
        return str(tuple(self))
      
    def __len__(self):
        return len(self._components)

    def __getitem__(self, index):
        cls = type(self)  
        if isinstance(index, slice):  
            return cls(self._components[index])  
        elif isinstance(index, numbers.Integral):  
            return self._components[index]  
        else:
            msg = '{cls.__name__} indices must be integers'
            raise TypeError(msg.format(cls=cls))  

```



> 扩展思考：用上面的方法，如何实现一个字典也能切片操作呢？