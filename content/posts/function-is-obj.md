---
title: "函数即对象"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---

# 1	支持函数式变成的包
## 1.1	operator函数
reduce：对一个可迭代对象中的元素进行累计计算，得到一个值
```js
def is_obj(arr):
    return reduce(lambda x, y: x + y, arr)
```
- 将arr中的元素相加
```js
from operator import add
def is_obj2(arr):
    return reduce(add, arr)
```
- operator中的add函数可以实现lambda函数的效果
itemgetter可以获取序列中的元素
```js
from operator import itemgetter

arr = [{'name': 'apple', 'price': 10}, {'name': 'banana', 'price': 20}, {'name': 'cherry', 'price': 30}]

print(sorted(arr, key=itemgetter('price')))
```
- 获取序列中的price并按升序打印
同样，lambda函数也可以实现
```js
print(sorted(arr, key=lambda x: x['price']))
```
itemgetter可以传入多个参数
```js
arr = [{'name': 'apple', 'price': 40, 'sales_num' : 100}, {'name': 'banana', 'price': 20, 'sales_num' : 200}, {'name': 'cherry', 'price': 30, 'sales_num' : 300}, {'name': 'date', 'price': 30, 'sales_num' : 400}]

print(sorted(arr, key=itemgetter('price', 'sales_num')))
```
- 会按price先排序，再按sales_num排序
attrgetter可以获取对象属性
```js
from operator import attrgetter

class LatLon:
    def __init__(self, lat, lon):
        self.lat = lat
        self.lon = lon

cities = [LatLon(35.689722, 139.691667), LatLon(37.774929, -122.419418), LatLon(40.712776, -74.005974)]
for city in sorted(cities, key=attrgetter('lat')):
    print(city.lat)
```
- 也可以获取嵌套结构，比普通元祖或者字典更加规范
methodcaller可以在对象上调用制定的方法
```js
from operator import methodcaller

print(methodcaller('upper')('Hello'))
```
- 在对象上调用转大写函数，把字符都变成大写
## 1.2	functools.partial
将接受多个参数的函数改造成接受更少参数的回调api
```js
from operator import mul
from functools import partial

triple = partial(mul, 3)
print(triple(7))
```
- 将mul的第一个参数设置为默认3
