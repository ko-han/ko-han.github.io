---
layout: post
title: "Python metaclass by example"
---
注： 本文中的 python 指 python3。所有代码展示均在 python3.6 下的运行的结果。

## 什么是元类？
元类可以简单的理解为类工厂， 所有的类都是由元类产生的。python 的默认元类是type。type 关键字也接受一个关键字的情况考虑下，如果参数是类，则返回类的元类，如果是实例则返回实例的类。
```
In [4]: class A:
   ...:     pass
   ...:
   ...:

In [5]: type(A)
Out[5]: type

In [6]: type(A())
Out[6]: __main__.A

In [7]: class M(type):
   ...:     pass
   ...:
   ...:

In [8]: class B(metaclass=M):
   ...:     pass
   ...:
   ...:

In [9]: type(B)
Out[9]: __main__.M
```
在上面的代码中，`type(A)` 返回 `type` 是因为 `type` 关键字也是一个元类，是 python 中默认的元类。我们通过继承 `type` 可以自定义一个元类， 通过在类定义的时候添加关键字 `metaclass=M` 来自定义类的元类。如类 B ，所以在我们调用 `type(B)` 的时候返回的元类是 M。接着我们要考虑如何写代码使我们的元类可以做一些事。我们来实现一个python的Enum类。首先我们考虑一个枚举类有什么功能？ 第一， 枚举类的成员名不能重复； 第而，枚举类只能有一个，不能被实例化；第二， 枚举类在定义了之后不能做任何修改， 第三，枚举类应该有auto关键字，因为有时候我们并不关心枚举类具体的值是什么。好了，我们知道了这三点要求，就可以去实现了。
```
from collections import OrderedDict
class EnumDict(OrderedDict):
    def __setitems__(self, key, value):
        if key in self:
            raise TypeError("can't reassign emum items: {}".format(key))
        super().__setitems__(key, value)

```
为了实现元类成员名不能重复，我们首先实现了一个自定义 dict，不能重复设置来保证成员名不重复，那么如何用这个自定义dict呢？ 元类有一个魔法方法 __prepireed
