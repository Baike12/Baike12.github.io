---
title: "迭代器、生成器和经典协程"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---


# 1	iter函数
## 1.1	iter函数调用顺序
- 如果对象实现了__iter__函数，就调用他
- 否则如果实现了__getitem__，iter就创建一个迭代器尝试从0号索引开始
- 否则抛出typeerror异常
迭代器耗尽就没用了，需要重新调用iter构建迭代器
## 1.2	哨符
iter的调用形式
```js
iter（可调用对象，哨符）
```
- 当可调用对象返回的值和哨符相等时停止迭代
>[!note]
>序列都可以迭代

# 2	可迭代对象与迭代器
python从可迭代对象中获取迭代器
# 3	迭代器的__iter__方法
## 3.1	迭代器与可迭代对象
可迭代对象有个__iter__方法，每次都要实例化一个新迭代器
迭代器要实现__next__方法返回单个元素，还要实现__iter__方法，返回迭代器本身
>[!note]
>迭代器也是可迭代对象，可迭代对象不一定是迭代器

## 3.2	迭代器模式
- 访问一个聚合对象不用暴露内部表示
- 支持聚合对象的多种遍历
- 为遍历不同聚合结构提供统一的接口
# 4	生成器函数
调用生成器函数返回生成器，生成器产出值；
__ iter__可以作为生成器函数，返回一个生成器，如果在一个类中定义__iter__函数并且其中有yield，那么__iter__就是生成器函数
# 5	惰性生成器
避免一次将所有内容加载到内存，而是产生一个生成器按需加载到内存
正则表达式包提供了finditer函数来返回一个生成器，可以用来返回
```js
import re
import reprlib

RE_WORD = re.compile(r'\w+')

class Sentence:
    def __init__(self, text):
        self.text = text

    def __iter__(self):
        for match in RE_WORD.finditer(self.text):
            yield match.group()

ss = Sentence("ni hao, chi fan la ma, hahaha")
for s in ss:
    print(s)
```
for循环每次迭代都相当于调用next()
## 5.1	生成器表达式
```js
class Sentence:
    def __init__(self, text):
        self.text = text

    def __iter__(self):
        return (match.group() for match in RE_WORD.finditer(self.text))
```
## 5.2	生成器与迭代器
迭代器：
- 实现了__next__方法
生成器：
- 是由编译器构建的迭代器
- 生成器对象是迭代器
**标准库和itertools库中内置了很多生成器**
# 6	yield from
把一个生成器的工作委托给一个子生成器
# 7	经典协程
## 7.1	元组
- 用作记录：项数固定，每一项可以是不同类型
- 用作不可变序列：项数任意，每一项都是相同类型
## 7.2	生成器
- 用户迭代器
- 用作协程：就是生成器函数，主体中有yield
## 7.3	协程计算器
计算平均值：
- 定义一个函数返回一个生成器
协程可以在多次激活之间保持局部状态
```js
from collections.abc import Generator


def avg_coroutine() -> Generator[float, float, None]:
    total = 0.0
    count = 0
    average = 0.0 
    while True:
        term = yield average
        total += term
        count += 1
        average = total / count

avg1 = avg_coroutine()
avg1.send(None)
print(avg1.send(10))
```
**通过哨符可以让协程返回结果，不产生数据**
## 7.4	经典协程的返回类型提示
协变与逆变：
- 描述类型之间的继承关系在泛型中的传递方式
- 协变：如果B是A的子类，且泛型类型G\[B]也是G\[A]的子类，G是协变的，比如Tuple
- 逆变：如果B是A的子类，且泛型类型G\[A]是G\[B]的子类，G是逆变的，python中函数的参数类型是逆变的
- 不变：如果B是A的子类，且泛型类型G\[A]和G\[B]没有关系
型变经验法则：
- 如果一个形式类型参数定义的是从对象中获取的类型，那么这个形式类型参数可能是协变的
- 如果一个形式类型参数定义的是对象初始化之后向对象中输入的数据类型，那么该形式类型参数可能是逆变的
Generator的定义
```js
Generator[YieldType, SendType, ReturnType]
```
- YieldType：协变的
- SendType：逆变的

