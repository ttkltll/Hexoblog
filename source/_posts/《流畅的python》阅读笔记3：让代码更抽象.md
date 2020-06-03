---
title: 《流畅的python》阅读笔记3：让代码更抽象
date: 2020-06-03 10:09:49
tags:
categories: 《流畅的python》
---


>  注释：下面代码演示了如何让代码更抽象，更有应对变化的可能。下面（1）（2）（3）（4）（5）都是被否定的写法，也是一些初学者容易写出的代码。
```python
class Vector2d():
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __iter__(self):
        #return iter(tuple(self.x, self.y))        （1）
        #这里可以改成用生成器：
        return (i for i in (self.x, self.y))
    def __repr__(self):
        #return 'Vector2d(' + str(self.x) + ', ' + str(self.y) +')'    （2）
        # 上面这种写法，没有用到这个类的可迭代，如参数太多，你一个个赋值，那就难了,还有就是直接写Vector2d,如果以后这个类变了名字，就麻烦了。这种用+号拼接，是最直观的，但是字符串是不可变序列，它会占有用新的内存。
        #下面是一种好点的写法，但也不够好
        #return '{}({!r}, {!r})'.format(class_name, self.x, self.y)    （3）
        #下面是更好的，能更好地应对变化，够抽象。
        class_name = type(self).__name__
        return '{}({!r}, {!r})'.format(class_name, *self)
    def __eq__(self, other):
        #if self.x == other.x and self.y == other.y:       （4）
        #    return True
        #return False
        #上面的写法很大问题，先不说不简捷，可以直接
        #return self.x == other.x and self.y == other.y      （5）
        #更好地写法是加直接调用内置类型来判断：
        return tuple(self) == tuple(other)  


    def __str__(self):
        return str(tuple(self.x, self.y))
```

