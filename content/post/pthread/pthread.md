---
title: "pthread多线程编程初探"
date: 2019-06-01T21:30:20+08:00
tags: ["C++", "pthread", "并行计算"]
categories: ["Technology"]
featured_image:
draft: false
---

# pthread多线程编程初探

**POSIX线程**（POSIX threads），简称Pthreads，是线程的**POSIX标准**。该标准定义了创建和操纵线程的一整套API。

## 实验要求

**使用pthread实现图像卷积的并行计算**

由于之前使用MPI实现了一遍，相比与MPI，pthread是以共享内存的方式实现并行计算，所有有着系统开销更小，并行损耗更小的优点。但是也限制了pthread只能在一台机器上实现并行，不支持分布式计算。

## 基本概念

### 进程与线程

进程是一个程序运行的实例，一个进程对应系统运行中的一个任务，独占一部分系统资源。

随着人们对于并行计算的需求越来越多，而每次都开多个进程虽然可以实现并行计算，但带来的新的问题——进程间通信。这就需要涉及操作系统提供的进程通信方法（无名管道、有名管道、信号、消息队列、共享内存、Socket、共享文件），无论哪种方式，对于程序原来说需要单独编写通信相关代码还是过于麻烦。因此，懒惰的程序员们便提出了线程的概念。

线程是更小的任务实例，对于进程来说，每个进程有着自己独立的数据段（全局变量）、BSS段（未初始化的全局变量和静态变量）、以及堆栈段（局部变量），同一个程序的进程还有着共享的代码段。而对于线程来说，为了解决进程间通信的痛点，进程的数据段、bss段、堆在线程间共享，这样线程之间可以共享进程中的大部分数据，而且一个线程创建和销毁的开销的十分小，可以说很好的解决了fork多进程的问题。

当然新的机制必然会带来新的问题，比如说共享的数据空间就需要考虑写竞争和一些其他的同步问题，这都需要在实践中一点一点的学习

> 在后面Unix系统分析的课程中了解到，其实linux中对于线程的管理也是按照进程的方式管理，只不过线程的地址空间与普通进程有区别，所以说linux中最小的任务实例还是进程

### Liunx的pthread

**头文件**

```c++
#include <pthread.h>
```

**创建线程**

```c++
int pthread_create(
    pthread_t *restrict_tidp,
    const pthread_attr_t *restrict_attr,
    void *(*start_rtn)(void),
    void *restrict_arg
    );
```

需要四个参数，分别对应：线程信息结构体，线程创建参数，线程入口函数，函数参数），四个参数均按地址传入，创建成功返回0，不成功代表创建失败

看一段代码

```c++
// pthread create
if(pthread_create(&thread_id, NULL, thread_task, &paras[i]) < 0)
{
    cerr << "pthread_create error" << endl;
    exit(-1);
}
```

thread_task为函数指针，对应线程要执行的函数入口，可以看到线程的执行是以一个函数为入口的，将可并行的部分放在函数中单独让线程执行，使用起来确实更加方便

**回收线程**

```c++
int pthread_join(pthread_t thread, void **retval);
```

第一个参数为回收的线程号，会阻塞等待，第二个参数对应线程状态结构体，将回收线程的状态复制到对应地址空间，若不关心线程结束状态，可传入NULL

**编译**

由于`pthread.h`不是标准库函数，之前需要指定`-lpthread`，但我发现gcc 7.3.0这样会报错找不到定义，只需要编译时指定`-pthread`参数即可编译成功

```shell
g++ pthread.cpp -o pthread -pthread
```

## 遇到的问题及解决方法

### Id

有三个东西，pid，thread_id，tid

对于一个进程来说：

* pid就是进程id，和thread_id相等

对于一个进程的许多线程来说：

* thread_id与pid不同，而且不是一个从0开始的数
* thread_id在该进程内不相同，可能与其他进程的线程thread_id相同
* 由于之前提到的，linux对于线程的管理也是按进程的方式，所以线程也有一个pid，称为tid，可以使用系统调用获取。
* 若需要项其他进程的线程发送信息，就需要使用tid来标识对应线程，而不是pid或者thread_id

### 传递参数时地址共享问题

由于需要传递多个参数，采用结构体`task_para`传值

```c++
task_para paras;
// pthread create
for (int i = 0; i < thread_num; i++) 
{
    paras.thread_num = thread_num;
    paras.my_num = i;
    paras.start_line = i * BmpHeight / thread_num;
    paras.end_line = paras[i].start_line + BmpHeight / thread_num - 1;
    if(pthread_create(&thread_id, NULL, thread_task, &paras) < 0)
    {
        cerr << "pthread_create error" << endl;
        exit(-1);
    }
    cout << "Thread " << paras.my_num << " start." << endl;
    threads.push_back(thread_id);
}
```



传递参数时传递不同地址的结构体

```c++
task_para* paras = new task_para[thread_num];    
// pthread create
for (int i = 0; i < thread_num; i++) 
{
    paras[i].thread_num = thread_num;
    paras[i].my_num = i;
    paras[i].start_line = i * BmpHeight / thread_num;
    paras[i].end_line = paras[i].start_line + BmpHeight / thread_num - 1;
    if(pthread_create(&thread_id, NULL, thread_task, &paras[i]) < 0)
    {
        cerr << "pthread_create error" << endl;
        exit(-1);
    }
    cout << "Thread " << paras[i].my_num << " start." << endl;
    threads.push_back(thread_id);
}
```



以上就是这次实验中总结的一些知识点，当然thread还有许多用法没有去实践，之后用OpenMP来实现卷积时再打算总结一下各自优劣吧。