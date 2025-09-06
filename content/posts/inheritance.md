---
title: "继承"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---


# 1	super函数
调用super().__ init__函数可以让超类完成它负责的初始化任务，也可以这样
```js
class MyDict(OrderedDict):
    def __init__(self, *args, **kwargs):
        OrderedDict.__init__(self, *args, **kwargs)
```
- 这样如果后面变更了基类可能会忘记更新
python3中super()不用传递参数，但是实际上传递参数也是可以的，在python2中就是需要传递参数的
# 2	子类化内置类型
内置类型（如dict、list）**是c语言编写的**，通常不调用用户定义的类覆盖方法；比如，如果一个类继承自dict，重写了__setitem__方法，那么重写的方法不会生效，还是会用dict自身的方法；
内置类型的方法调用其他类的方法如果被覆盖了，也不会被调用；
- 定义一个类MyDict继承自dict
- 重定义__getitem__返回固定值42
- 直接用MyDict实例化一个d，返回的是42
- 用dict实例化一个d2，然后d2调用update(d)，不会返回42
```js
class MyDict2(dict):
    def __getitem__(self, key):
        return 42
    
d = MyDict2(a=1)
print(d['a'])  # 42

d2 = {}
d2['a'] = 10
d2.update(d)
print(d2) # 1
```
用户自定义的类应该继承collections中的类
# 3	多重继承的方法解析顺序
## 3.1	菱形问题
祖父和两个父都有同名方法，子类调用哪个方法
定义类：
- Root实现ping、pong
- A继承自Root，实现ping并在ping中调用super的ping，实现pong并在pong中调用super的pong
- B继承自Root，实现ping并在ping中调用super的ping，实现pong但是没有调用super的pong
- Leaf继承自A、B，实现ping并在ping中调用super的ping
```js
class Root: 
    def ping(self):
        print(f'{self}.ping() in Root')

    def pong(self):
        print(f'{self}.pong() in Root')

    def __repr__(self):
        cls_name = type(self).__name__
        return f'<instance of {cls_name}>'


class A(Root): 
    def ping(self):
        print(f'{self}.ping() in A')
        super().ping()

    def pong(self):
        print(f'{self}.pong() in A')
        super().pong()


class B(Root): 
    def ping(self):
        print(f'{self}.ping() in B')
        super().ping()

    def pong(self):
        print(f'{self}.pong() in B')


class Leaf(A, B): 
    def ping(self):
        print(f'{self}.ping() in Leaf')
        super().ping()

```
实例化Leaf，调用ping、pong
- ping：Root、A、B、Root
- pong：A、B，没有Root
```js

l = Leaf()
l.ping()
l.pong()

<instance of Leaf>.ping() in Leaf
<instance of Leaf>.ping() in A
<instance of Leaf>.ping() in B
<instance of Leaf>.ping() in Root
<instance of Leaf>.pong() in A
<instance of Leaf>.pong() in B
```
每个类都有__mro__属性，是一个元祖，保存从当前类到object类型的调用顺序：
```js
(<class '__main__.Leaf'>, <class '__main__.A'>, <class '__main__.B'>, <class '__main__.Root'>, <class 'object'>)
(<class '__main__.A'>, <class '__main__.B'>)
```
- 解析顺序使用了c3算法
子列继承的写法顺序也决定解析顺序
>[!note]
>一般都加上super方法就好

# 4	混入类
不能作为具体类的唯一基类，因为没有提供具体对象的全部功能，而是增加或定制子类或同级类的行为
混入类要发生作用需要让他的解析顺序在其他类的前面
# 5	多重继承的应用
tkinter、Django等
# 6	应对多重继承
## 6.1	优先使用对象组合而不是继承
## 6.2	理解不同情况下使用继承的原因
一般有两种创建子类的原因：
- 继承接口，创建子类型，这种情况最好使用抽象基类
- 继承实现，通过重用避免代码重复，这种情况可以利用混入
## 6.3	使用抽象基类显式表示接口
如果类的作用是定义接口，应该把他定义成抽象基类或者typing.Protocol的子类
## 6.4	通过混入明确重用代码
如果一个类只是提供方法，不提供关系，最好用混入：
- 混入只打包方法，便于重用
- 混入类不能实例化
- 混入类不保持内部状态，也就是没有属性
使用混入最好加上Mixin
## 6.5	聚合类
解构继承自混入，没有添加自身结构或行为
聚合类的主题不一定为空，只是经常为空
# 7	总结
**go是最好的语言:)**
继承被推广的原因是为了开发框架