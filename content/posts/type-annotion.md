---
title: "类型注解"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---

# 1	渐进式类型
## 1.1	特性
### 1.1.1	可选
如果类型检查工具无法确认类型，会默认为Any
### 1.1.2	不在运行时检测
### 1.1.3	不能改善性能
理论上可以通过类型来改善字节码生成，但是还没有实现
## 1.2	mypy
定义一个函数
```js
def show_count(count, word):
    if count == 1:
        return f"1 {word}"
    count_str = str(count) if count else "no"
    return f"{count_str} {word}s"

print(show_count(0, "apple"))
```
使用mypy检查
```js
mypy python_ex/8_func_type.py
```
- 如果没有定义类型注解不会发现问题
加上严格检查参数
```js
((.venv) ) baike@baikedeMac-mini my-ex % mypy --disallow-untyped-defs python_ex/8_func_type.py 
python_ex/8_func_type.py:1: error: Function is missing a type annotation  [no-untyped-def]
Found 1 error in 1 file (checked 1 source file)
```
- 报错没有为函数加上类型注解
```js
def show_count(count: int, word: str) -> str:
    if count == 1:
        return f"1 {word}"
    count_str = str(count) if count else "no"
    return f"{count_str} {word}s"

print(show_count(0, "apple"))
```
加上注解
```js
(.venv) ) baike@baikedeMac-mini my-ex % mypy --disallow-untyped-defs python_ex/8_func_type.py
Success: no issues found in 1 source file
```
- 不报错了
**使用flake8检查类型，使用blue自动归整代码风格，类似于gofmt**
## 1.3	使用None表示默认值
使用None表示参数为可变类型
```js
from typing import Optional
def show_count(count: int, word: str, plural: Optional[str] = None) -> str:
```
- plural可以为str或者None，必须显示提供默认类型
# 2	类型由受支持的操作定义
类型检查只关心显式声明的类型
```js
def double(x: abc.Sequence):
    return x * 2
```
- 因为abc.Sequence没有实现__mul__方法，所以类型检查不会通过
## 2.1	渐进式类型系统
有两种风格的类型解读
### 2.1.1	鸭子类型
python这样的：
- 对象有类型，变量没有类型
- 对象的类型不重要，重要的是对象支持什么类型
- 运行时才会检查类型
- 更灵活，但是潜在错误更多
### 2.1.2	名义类型
c++这样的：




