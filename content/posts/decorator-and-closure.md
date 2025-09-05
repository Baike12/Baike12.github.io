---
title: "闭包和装饰器"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---

# 1	装饰器基础
是一种可调用对象，参数是另一个函数，对被装饰的函数做一些处理返回一个函数，或者把函数替换成另一个函数或者可调用的对象
装饰器把一个函数替换成另一个函数
```js
def deco(func):
    def inner():
        print("decorator")
    return inner


@deco
def target():
    print("target")


target()
```
- 会打印decorator，说明target被替换为了inner函数
**装饰器在加载模块的时候立即执行了**
# 2	python很是执行装饰器
在加载模块的时候就运行了，在所有函数之前，如果装饰器运行带来一些全局变量的变化，这种变化在装饰器运行的时候就发生了
被装饰的函数在被调用的时候才运行
# 3	注册装饰器
>[!note]
>装饰器返回的函数与通过参数传入的函数相同，这种方式的应用场景：
>把被装饰的函数注册到中央注册处

# 4	变量作用域规则
python不要求声明变量，但是假定在函数中复制的变量是局部变量
```js
b = 6


def func(a):
    print(a)
    print(b)
    b = 8
```
- 会打印a，但是打印b时报错
- 这里b在编译函数主体时被认为是局部变量，打印b时会认为b没有绑定值
要让解释器认识作为全局变量的b，需要加上global
```js
def func(a):
    global b
    print(a)
    print(b)
    b = 8
```
- b为6
## 4.1	作用域类型
- 模块全局作用域
- 函数局部作用域
### 4.1.1	汇编分析
```js
from dis import dis


def func(a):
    print(a)
    print(b)
    b = 8
    
    
 22           0 RESUME                   0

 23           2 LOAD_GLOBAL              1 (NULL + print)
             12 LOAD_FAST                0 (a)
             14 CALL                     1
             22 POP_TOP

 24          24 LOAD_GLOBAL              1 (NULL + print)
             34 LOAD_FAST_CHECK          1 (b)
             36 CALL                     1
             44 POP_TOP

 25          46 LOAD_CONST               1 (8)
             48 STORE_FAST               1 (b)
             50 RETURN_CONST             0 (None)
```
- 可以看到b被当做局部变量
# 5	闭包
闭包是延伸了作用域的函数，包括函数主体中引用的非全局变量和局部变量
```js
def make_averager():
    series = []
    def averager(new_value):
        series.append(new_value)
        total = sum(series)
        return total / len(series)
    return averager
    
avg = make_averager()
```
- series作为自由变量保存avg中
>[!note]
>自由变量指未在局部作用域中绑定的变量，局部作用域就是averager这个作用域

可以通过__code__中看到freevars
```js
print(avg.__code__.co_freevars)
('series',)
```
series中的值保存在对象的__closure__[0].cell_contents中
```js
print(avg.__closure__[0].cell_contents)
[10, 11, 12]
```
# 6	nonlocal声明
可以不保存所有值，只保存总值和项数
如果这样写
```js
def make_averager2():
    count = 0
    total = 0
    def averager(new_value):
        count += 1
        total += new_value
        return total / count
    return averager
```
- 这样写会报错，因为count+=1相当于一次赋值，会让count变成局部变量而不是自由变量
使用nonlocal可以将变量设置为自由变量
```js
def make_averager2():
    count = 0
    total = 0
    def averager(new_value):
	    nonlocal count, total
        count += 1
        total += new_value
        return total / count
    return averager
```
## 6.1	变量查找逻辑
- 如果被global修饰则x来自全局作用域
- 被nonlocal修饰在最近的外层函数中查找
- x是参数，或者在函数主体赋值了，就是局部变量
- 如果引用了x但是没有赋值也不是参数，比如使用append的列表
	- 在外部函数主体的局部作用域中找
	- 没在外层函数主体中，则在全局作用域中找
	- 全局作用域中还是没有，就在__builtins__.__dict__中找

## 6.2	一个统计函数运行时长的装饰器
```js
def clock(func):
    def clocked(*args, **kwargs):
        t0 = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_str = ', '.join(repr(arg) for arg in args)
        print(f'[{elapsed:0.8f}s] {name}({arg_str}) -> {result}')
        return result
    return clocked
```
- 定义一个装饰器，接受一个函数作为参数，就是被装饰的函数
使用装饰器装饰一个函数
```js
@clock
def factorial(n):
    return 1 if n < 2 else n * factorial(n - 1)
```
- 实际执行的是clocked函数
被装饰函数都丢失元数据，可以使用functools.wraps来保留被装饰函数的元数据
```js
def clock(func):
    @functools.wraps(func)
    def clocked(*args, **kwargs):
        t0 = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_str = ', '.join(repr(arg) for arg in args)
        print(f'[{elapsed:0.8f}s] {name}({arg_str}) -> {result}')
        return result
    return clocked
```
# 7	标准库中的装饰器
## 7.1	python内置的装饰器
### 7.1.1	property
将方法转换为属性访问，不需要加括号调用方法
```js
class User:
    def __init__(self, name):
        self.name = name
    
    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        self._name = value


user = User('John')
print(user.name)
```
- setter用于写，需要写属性得加上

用途：
- 保持属性访问简洁，隐藏内部实现
- 对属性访问进行控制
### 7.1.2	classmethod
用于定义类方法，第一个参数是类而不是实例：
- 可以通过类名直接调用，当然也可以通过实例调用
- 用于操作类级别数据，或者创建类的实例（相当于构造函数）
```js
class User:
    def __init__(self, name):
        self.name = name

    @classmethod
    def from_str(cls, name):
        return cls(name)


user = User.from_str('John')
print(user.name)
```
- 通常用作备选的构造函数
### 7.1.3	staticmethod
定义静态方法，与类没有关系，参数中没有cls或者self
只是逻辑上属于类，不依赖类和实例，但是可以使用类和实例调用
```js
    @staticmethod
    def add(a, b):
        return a + b


print(User.add(1, 2))
```
## 7.2	标准库中的装饰器
### 7.2.1	functools.cache
缓存计算或者api结果
如果缓存较大，可能撑爆内存
### 7.2.2	functools.lru_cache
参数：
- maxsize：最大条数，最好设置为2的整数次方
- typed：是否把不同参数类型的结果分开保存，如1和1.0
```js
    @lru_cache(maxsize=32, typed=True)
    @staticmethod
    def add(a, b):
        return a + b
```
### 7.2.3	functools.singledispatch
分派函数，被装饰的函数变成返回函数，根据第一个参数的类型执行不同的操作
```js
@singledispatch
def show(obj):
    print(f'{obj} is a {type(obj)}')

@show.register(int)
def _(int_obj):
    print(f'this is a int')

@show.register(str)
def _(str_obj):
    print(f'this is a str')

show(1)
show('1')
```
- 好处在于灵活的注册针对不同类型参数的处理逻辑
# 8	参数化装饰器
装饰器将函数作为第一个参数
创建一个装饰器工厂函数来接收参数，然后返回一个装饰器，应用到被装饰的函数上
```js
registry = set()

def register(active=True):
    def decorate(func):
        print(f'running register(active={active})->decorate({func})')
        if active:
            registry.add(func)
        else:
            registry.discard(func)
        return func
    return decorate

@register(active=False)
def f1():
    print('running f1()')

@register()
def f2():
    print('running f2()')

print(registry)
f1()
f2()
```
- register是一个装饰器工厂，decorate才是装饰器
- register必须作为函数调用，返回真正的装饰器decorate
不适用@语法也可以
```js
register(active=False)(f1)
```
- 踢出f1
## 8.1	基于类的装饰器
通过定义类的__call__方法实现装饰器
```js
class clock2:
    def __init__(self, fmt='%H:%M:%S'):
        self.fmt = fmt

    def __call__(self, func):
        def clocked(*args, **kwargs):
            t0 = time.time()
            result = func(*args, **kwargs)
            elapsed = time.time() - t0
            print(f'[{elapsed:0.8f}s] {func.__name__}({args}, {kwargs}) -> {result}')
            return result
        return clocked
```
- 将参数定义为类的属性，可以在实例化类的时候传入参数
- 定义call方法使类可以调用，返回被clocked装饰的函数
**使用类实现装饰器是一种更好的方法**

