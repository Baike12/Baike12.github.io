---
title: "with、match和else"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---

# 1	with
设置一个临时上下文，交给上下文管理器对象控制，管理器对象要负责清理上下文
包含__enter__, __ exit__两个方法
```js
>>> with open("coldturkey.py") as f:
...     f.read(100)
...
'import json\nimport os\nimport sqlite3\n\nDB_PATH = r"/Library/Application Support/Cold Turkey/data-app.'
>>> f
<_io.TextIOWrapper name='coldturkey.py' mode='r' encoding='UTF-8'>
```
- 上下文管理器对象是一个TextIOWrapper实例，它调用__enter__方法返回的值赋值给f
- 退出时上下文管理器调用__exit__
## 1.1	contextmanager装饰器
把简单的生成器函数变成上下文管理器
只需实现一个有yield语句的生成器，yield将函数分成两部分，yield前面的所有代码在enter时执行，后面的在exit时执行
使用这个装饰器时要把yield放到try、finally中来处理可能的异常，否则异常会导致exit不会被执行，无法完成恢复
contextmansger装饰的生成器也可以用作装饰器
# 2	match
保留关键字的作用：
- 引入特殊求值法则
- 变更控制流
- 管理环境
>[!note]
>自由变量：在函数体中出现的除参数、局部变量和全局变量之外的变量
>zip：将多个可迭代对象的对应位置打包成一个个元组

## 2.1	if之外的else
for else：for执行完之后执行
while else：while为假退出后执行
try else：try没有抛出异常才执行，如果有其他如return、break等分支处理了异常，也会跳过else
