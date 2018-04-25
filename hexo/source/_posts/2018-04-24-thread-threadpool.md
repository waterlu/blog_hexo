---
title: 多线程(2) 线程池
date: 2018-04-24 12:37:24
updated: 2018-04-25
categories: 并发编程
tags: [并发, 线程, 线程池]
toc: true
description: 。
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

> Executor将任务提交给线程池来处理，线程池的处理是异步的，任务会交给新的线程来执行，本地线程可以继续做其他事情。
>
> Executor只有一个没有返回值的接口execute()，为了更好的控制线程的行为，并行包中为我们提供了功能更强大的ExecutorService，实际使用过程中，更多使用的是ExecutorService的接口。

```java
package java.util.concurrent;
public interface ExecutorService extends Executor {
  	// 优雅关闭
	void shutdown();  
	// 提交任务执行
	<T> Future<T> submit(Callable<T> task);
	Future<?> submit(Runnable task);	
}
```

