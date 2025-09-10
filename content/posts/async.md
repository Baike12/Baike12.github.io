---
title: "异步编程"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---


# 1	一些定义
## 1.1	异步生成器
使用async def定义，在函数主体中有yield，返回一个提供__anext__方法的异步生成器对象
# 2	可异步调用的对象
**for处理可迭代的对象，await处理可异步调用的对象**
获取可异步调用对象的途径：
- 原生协程对象，由async def定义
- asyncio.Task，把协程对象传给create_task得到，不用保存create_task，只要创建就可以调度协程运行，当然，如果要终止协程，需要保存返回对象
## 2.1	原生协程与生成器
用户代码在asyncio和异步库（httpx）之间
asyncio事件循环在背后调用.send驱动用户协程，当await后的表达式返回时，事件循环通过send唤醒外层协程从暂停处继续执行；用户协程使用await等待其他协程
await链最终到达一个底层可异步调用对象，返回一个生成器，由事件循环驱动，对io或计时器等事件做出响应
# 3	异步上下文管理器
在数据库驱动中，执行失败需要回滚，成功要提交，但是涉及和数据库交互，最好用异步操作，这就需要异步上下文管理器
## 3.1	定义
语法：
- sync with
- 以协程实现__enter__和__aexit__
```js
async with connection.transaction():
    await connection.execute("INSERT INTO mytable VALUES (1, 2, 3)")
```
## 3.2	run_in_executor
将同步函数提交到线程池或者进程池中执行，避免它阻塞事件循环
# 4	异步可迭代对象
## 4.1	异步生成器表达式
异步生成器表达式可以在任意位置定义，但是只能在async def主体内或者异步控制台中使用
## 4.2	异步推导式
# 5	异步原理和陷阱

