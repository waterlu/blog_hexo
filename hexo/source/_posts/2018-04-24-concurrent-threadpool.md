---
title: 多线程(2) 线程池
date: 2018-04-24 16:37:24
updated: 2018-04-25
categories: 并发编程
tags: [并发, 线程, 线程池]
toc: true
description: 本文介绍并发中最基本的概念-线程，包括我对进程和线程的理解，线程的状态的变化。其中，理解线程的状态变化是关键点，深入理解线程状态迁移是理解并发锁的前提。最后，通过一个经典的生产者-消费者例子来说明线程状态和多线程之间的切换。
comments: false
---

## 任务调度

任务是一组逻辑工作单元，线程则是任务异步执行的机制。前面说过，实战中我们不会直接调用Thread类的start()方法来启动线程，那么线程应该如何启动呢？

使用Executor.execute(task);来替代new Thread(task).start();

```java
package java.util.concurrent;
public interface Executor {
    void execute(Runnable command);
}
```

Executor将任务交给线程池来处理，



## 线程池

### ThreadPoolExecutor

理解了ThreadPoolExecutor类的各个参数和内部原理也就理解了线程池机制。

#### 构造参数

```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
		TimeUnit unit, BlockingQueue<Runnable> workQueue, 
		ThreadFactory threadFactory, RejectedExecutionHandler handler) {}
```

先来看ThreadPoolExecutor类的参数

| 参数名称            | 含义   | 详解                                       |
| --------------- | ---- | ---------------------------------------- |
| corePoolSize    | 基本大小 | 即使当前没有待执行任务，线程池也可以保留corePoolSize个线程；当前线程数小于corePoolSize时，会一直创建新线程； |
| maximumPoolSize | 最大大小 |                                          |
|                 |      |                                          |
|                 |      |                                          |
|                 |      |                                          |
|                 |      |                                          |
|                 |      |                                          |

https://blog.csdn.net/lipc_/article/details/52025993

https://blog.csdn.net/hayre/article/details/53291712

https://blog.csdn.net/qq_25806863/article/details/71126867

https://blog.csdn.net/a837199685/article/details/50619311