---
title: "接口和抽象基类"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---

>[!note]
>猴子补丁：在运行时修改类或模块，比如修改类的魔法方法

# 1	防御性编程和快速失败
如果要求传入特定类型的参数，不应该要求调用方传入指定类型，应该接收调用方传递的参数之后构建参数，如果无法构建就报错，或者用isinstance检查
如果接收可迭代对象作为参数，要早一点调用iter
## 1.1	鸭子类型
>[!note]
>如果一个对象表现的像一个类型，那他就是这个类型

有时候鸭子类型比静态类型更有表现力
```js
def conv(field_name):
    try:
        field_name = field_name.replace(',', '-').split()
    except AttributeError:
        print(f'{field_name} is not a string')
        pass
    else:
        return field_name
```
- 如果参数不能像string那样使用，说明不是string
- 类型提示不能表达“参数必须是string“
# 2	大鹅类型
定义：只要cls是抽象基类，就可以使用isinstance(obj,cls）
利用抽象基类实现的运行时检查方式
## 2.1	继承抽象基类的方式
>[!note]
>应该尽量少的创建抽象基类，抽象基类是一种强契约，会增加代码量和认知负担
>python更提倡鸭子类型，通过实现方法来判断对象类型
- 实现必要方法
- 注册虚拟子类
注册虚拟子类
```js
from collections.abc import Sequence
class User:
    def __init__(self, name, age, gender):
        self.name = name
        self.age = age
        self.gender = gender


Sequence.register(User)
```
- Sequence和User没有继承关系，但是User是Sequence的子类
此时
```js
print(isinstance(User('John', 20, 'male'), Sequence))
print(issubclass(User, Sequence))
```
- 都是True
>[!note]
>不应该经常使用isinstance来根据类型执行操作，应该使用多态，让解释器把调用分派给正确的方法

## 2.2	标准库中的抽象基类
![[Pasted image 20250905163256.png]]
## 2.3	实现一个抽象基类
### 2.3.1	抽象基类的虚拟子类
虚拟子类：指通过注册成为子类的类
使用register注册，解释器不会去检查虚拟子类是否实现了抽象基类的方法，如果没有实现，会在运行时报错
用于注册不是自己维护的、却需要满足指定接口的类
### 2.3.2	抽象基类实现结构类型
就算不显示注册，一个类也可以被识别为抽象基类的子类
比如abc.Size类，如果一个其他类型实现了__len__方法，会被认为是abc.Size类型的虚拟子类，因为Size类中有一个__subclasshook__方法，这个方法检测一个类是否有__len__方法，如果有，就认为这个类是Size类的虚拟子类
# 3	静态协议
## 3.1	为double添加类型提示
double函数将参数翻一倍，数值类型* 2，列表是元素个数* 2
使用名义类型系统时，注解类型名称和参数的类型名称匹配，支持类型的必要参数的类型可能有很多，所以类型提示无法描述鸭子类型
有了typing.Protocol之后类型提示可以描述鸭子类型了，可以告诉mypy工具，double函数接受支持* 2运算的参数x：
- 导入TypeVar和Protocol
- 使用TypeVar定义泛型类型T
- 定义一个类Repeatable，这个类继承Protocol，这个类中就一个__nul__方法，传入T和重复次数，返回T
- 定义RT，上界为Repeatable
- 定义一个函数，接受RT类型
这样只要实现了__mul__的类型都可以作为double的参数
```js
T = TypeVar('T')

class Repeatable(Protocol):
    def __mul__(self: T, other: int) -> T:

RT = TypeVar('RT', bound=Repeatable)

def twice(x: RT) -> RT:
    return x * 2

print(twice(10))
print(twice('Hello'))
print(twice([1, 2, 3]))
```
>[!note]
>isinstance和issubclass只检查有咩有特定的方法，不会检查 方法签名和类型注解

# 4	总结
- 鸭子类型和静态鸭子类型（go）：不需要显式声明对任何特定协议的支持，只要实现协议要求的方法

