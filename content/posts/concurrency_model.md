---
title: "并发模型"
date: 2025-09-02T21:36:29+08:00
draft: false
categories: ["python"]
tags: ["python"]
author: "Baike"
description: "《流畅的python》阅读笔记"
---


# 1	术语
## 1.1	进程
进程间通信只能通过原始字节进行，所以要序列化
## 1.2	协程
协程在事件循环的监督下载单个线程中运行
必须一个线程使用yield或者await关键字放弃控制权另一个协程才能并发工作；
如果协程中有阻塞代码，事件循环和其他协程都会被阻塞
## 1.3	python中的进程、线程和gil
- python解释器是一个进程，使用multiprocessing或者concurrent.futures可以启动额外的python进程，subprocess可以启动外部程序的进程（其他语言）
- python解释器只使用一个线程运行用户程序和垃圾回收程序，使用threading或者concurrent.futures可以启动额外的python线程
- 对对象引用计数和其他内部状态的访问受一个锁gil控制，一个时间点只有一个线程可以持有gil，所以只有一个现场可以执行python代码，就算多核也是这样
- 解释器每5ms暂停当前python线程避免一个线程一直持有gil
- python代码不能控制gil，耗时的任务可以由内置函数或者c扩展释放gil
- python标准库中所有发起系统调用的函数都能释放gil，比如磁盘io、网络io、time.sleep等
- 在python的c层集成的扩展可以启动不受gil影响的非python线程，这些线程不能修改python对象，但是可以读写内存中支持缓冲协议的底层对象比如bytearray、numpy数组等
- gil对网络编程影响很小，因为io函数会释放gil
- 计算密集型任务直接使用单线程进程执行
- 要利用多核必须用多进程
>[!note]
>协程和时间循环都运行在同一个线程中

python中没有终止线程的api，只能通过向线程发送消息终止
# 2	例子
实现旋转效果
## 2.1	用线程
```js
import itertools
import time
from threading import Event, Thread


def spin(msg: str, done: Event)-> None:
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, end=' ', flush=True)

        if done.wait(.1):
            break
    
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end=' ')


def slow()->int:
    time.sleep(3)
    return 42
    

def supervisor()->int:
    done = Event()
    spinner = Thread(target=spin, args=("thinking", done))
    print(f'spinner obj : {spinner}')
    spinner.start()
    res = slow()
    done.set()
    spinner.join()
    return res


def main()->None:
    result = supervisor()
    print(f'answer: {result}')


if __name__ == "__main__":
    main()
```
## 2.2	用进程
```js
import itertools
import time
from multiprocessing import Event, Process, synchronize


def spin(msg: str, done: synchronize.Event)-> None:
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, end=' ', flush=True)

        if done.wait(.1):
            break
    
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end=' ')


def slow()->int:
    time.sleep(3)
    return 42
    

def supervisor()->int:
    done = Event()
    spinner = Process(target=spin, args=("thinking", done))
    print(f'spinner obj : {spinner}')
    spinner.start()
    res = slow()
    done.set()
    spinner.join()
    return res


def main()->None:
    result = supervisor()
    print(f'answer: {result}')


if __name__ == "__main__":
    main()
```
- 从api来看：与线程的主要区别在Event是函数，threading中的Event是类
- 实际实现由很大区别，比如跨进程通信需要序列化、Event是用信号量实现的
>[!note]
>multiprocessing中有一个shared_memory包可以让多进程共享一个ShareableList，这是一个可变序列，可以存放固定数量的项
## 2.3	协程实现
时间循环管理所有协程，监视协程发起的io操作，当事件发生时把控制权传回对应协程
协程运行的方式：
- asyncio.run(coro())：驱动一个协程对象，作为所有异步代码的入口
- asyncio.create_task(coro())：在协程中调用，调度领一个协程最终执行，不终止当前协程，返回一个Task实例，包装协程对象
- await coro()：只能在协程中调用，把控制权交给coro()返回的协程对象，终止当前协程，知道coro的主体返回
>[!note]
>coro()调用立即返回一个协程对象，但是不云讯coro函数的主体，coro主体由事件循环驱动

```js
import asyncio
import itertools

async def spin1(msg: str) -> None:
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, flush=True, end=' ')
        try:
            await asyncio.sleep(.1)
        except asyncio.CancelledError:
            break
    blanks = " " * len(status)
    print(f'\r{blanks}\r', end=' ')

async def slow1() -> int:
    await asyncio.sleep(3)
    return 40

async def supervisor1() -> int:
    spinner = asyncio.create_task(spin1("thinking!"))
    print(f'spinner object: {spinner}')
    res = await slow1()
    spinner.cancel()
    return res


def main1() -> None:
    res = asyncio.run(supervisor1())
    print(f'Answer: {res}')

if __name__ == "__main__":
    main1()
```
- 注意使用asyncio创建的协程中不能使用time.sleep，这会阻塞整个线程，需要使用asyncio.sleep，这会把控制权还给事件循环
- 协程可以通过cancel终止，并在终止协程主体的await出抛出CancelledError异常
# 3	GIL
计算密集型任务的影响：
- 多进程时不会影响
- 多线程时也不会，因为线程调度是抢占式的，每5ms抢占一次
- 协程会阻塞，因为所有协程都在时间循环的线程上
# 4	多核世界中的python
## 4.1	系统管理
## 4.2	WSGI
python框架和应用程序接受http请求并响应的标准api，WSGI管理多个进程
比如有gunicorn
## 4.3	分布式任务队列

