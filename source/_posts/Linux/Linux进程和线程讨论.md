---
title: Linux进程和线程讨论
date: 2019-02-18 12:12:28
tags:
- linux
---

![processthread](https://qiniu.li-rui.top/processthread.png)

<!--more-->

# 多道程序设计系统

多道程序设计系统是操作系统类型的一种，多道程序设计技术是在计算机内存中同时存放几道相互独立的程序，使它们在管理程序控制下，相互穿插运行，两个或两个以上程序在计算机系统中同处于开始到结束之间的状态, 这些程序共享计算机系统资源。与之相对应的是单道程序，即在计算机内存中只允许一个的程序运行。

对于一个单CPU系统来说，程序同时处于运行状态只是一种宏观上的概念，他们虽然都已经开始运行，但就微观而言，任意时刻，CPU上运行的程序只有一个。

在多道程序设计系统实现上就要引入进程和线程的概念，从系统资源利用角度来看，进程和线程有着不同的资源利用维度。

# 进程和线程

在操作系统中，一个任务就是一个进程，比如打开chrome浏览器就是一个进程。一个任务里面还会有很多子任务，这些子任务为线程，比如chrome浏览器一边看网页，还有相关插件也在运行。

一个进程至少有一个线程，从上面多道程序设计系统可知，操作系统多任务并行在cpu执行层面上某一个时间点上，cpu只能执行摸一个进程或者线程，这个调度管理由操作系统来完成。

因此多任务实现上有三类

- 多进程
- 多线程
- 多进程加多线程

# 多进程

## Linux进程模型

linux进程模型最著名的就是fork调用。系统的初始进程id为0，之后的所有进程都是由0号进程通过fork调用创建的。因此，就会形成一个进程树。本文使用python来讨论多进程和多线程

## 多进程使用

多进程使用本质上是编程语言调用系统底层fork()函数。fork函数调用一次会出现两个返回值

- 父进程返回子进程id
- 子进程返回0


### 进程信息查看

```python
import os

print("my partent pid is %s" % os.getppid())

print("my pid is %s" % os.getpid())

pid = os.fork()

if pid == 0:
    print('I am child process (%s) and my parent is %s.' % (os.getpid(), os.getppid()))
else:
    print('I (%s) just created a child process (%s).' % (os.getpid(), pid))
```

返回值

```bash
my partent pid is 2518148
my pid is 2519879
I (2519879) just created a child process (2519880).
I am child process (2519880) and my parent is 2519879.
```

### 创建子进程

```python
from multiprocessing import Process
import os

# 子进程要执行的代码
def run_proc(name):
    print('Run child process %s (%s)...' % (name, os.getpid()))

if __name__=='__main__':
    print('Parent process %s.' % os.getpid())
    p = Process(target=run_proc, args=('test',))
    print('Child process will start.')
    p.start()
    p.join()
    print('Child process end.')
```

# 多线程

一个进程内可以创建多个线程

## 创建多线程

```python
import time, threading

# 新线程执行的代码:
def loop():
    print('thread %s is running...' % threading.current_thread().name)
    n = 0
    while n < 5:
        n = n + 1
        print('thread %s >>> %s' % (threading.current_thread().name, n))
        time.sleep(1)
    print('thread %s ended.' % threading.current_thread().name)

print('thread %s is running...' % threading.current_thread().name)
t = threading.Thread(target=loop, name='LoopThread')
t.start()
t.join()
print('thread %s ended.' % threading.current_thread().name)
```

## 线程中的变量

在多进程中每当传创建一个进程，变量就会进行复制到新的的进程中去。而线程是一个进程中的所有线程共享同一个变量。因此需要适当加lock。

## python中的GIL锁

在python中Global Interpreter Lock机制会把所有线程的执行代码都给上了锁，所以，多线程在Python中只能交替执行，因此pyton多线程并不能使用多核。使用多核只能使用多进程。

# 多进程和多线程

## 讨论

- 多进程稳定性高，因为一个子进程崩溃了，不会影响主进程和其他子进程
- 多进程创建进程的代价大。大量的进程切换会占用cpu
- 多线程模式致命的缺点就是任何一个线程挂掉都可能直接造成整个进程崩溃



