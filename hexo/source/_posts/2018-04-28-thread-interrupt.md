---
title: 多线程中级篇(2) 线程中断
date: 2018-04-28 14:24:57
categories: 并发编程
tags: [并发, 多线程]
toc: true
description: 我们在基础篇的线程介绍里面已经提到过如何中断线程，本文将详细展开，描述其中的细节，让大家对线程中断有一个更深刻的理解。
comments: false
---

我们在基础篇的线程介绍里面已经提到过如何中断线程，本文将详细展开，描述其中的细节，让大家对线程中断有一个更深刻的理解。

## 线程中断

### 停止线程

虽然在Thread中提供了stop()方法，但是已经声明为@Deprecated不再使用了。所以，其实Java并没有给我们提供一种简单和方便的优雅停止线程的方法。

```java
@Deprecated
public final void stop() {
}
```

Java没有提供，那只能我们自己来实现了，比较容易想到的是加一个标志位stop=false，在线程执行过程中不断判断这个标志位，当标志为被置为true时，正常退出线程。

注意，这个标志位是典型的volatile关键字的应用场景：所有线程共享这个标志位，每个线程在自己的工作内存中有一个备份，当外部修改主内存中的标志位后，各个线程需要立即读取到变化，完成线程退出。

```java
public class Task implements Runnable {
  private volatile boolean stop = false;
  @Override
  public void run() {
    while (!stop) {
      // do something
    }
  }
  public void stop() {
    this.stop = true;
  }
}
```

```java
Task task = new Task();
new Thread(task).start();
task.stop();
```

### 中断线程

以上代码可以实现线程的正常停止，没有问题。但是有一种情况无法处理，那就是如果在while()语句块出现了I/O或者同步相关的阻塞操作，那么就无法判断stop的值了，上面停止线程的方法就失效了。怎么办呢？这个时候就需要中断线程了。

首先，这些阻塞方法是可以被中断的，一旦被中断将抛出InterruptedException异常；从下面的方法声明中可以看到，join()、sleep()和wait()都声明抛出InterruptedException异常，也就是说这些阻塞方法会检查线程的中断状态，如果发现被中断就抛出异常，变相提供了一种退出机制。

```java
public class Thread {
  public final void join() throws InterruptedException {}
  public static native void sleep(long millis) throws InterruptedException;
}
```

```java
public class Object {
  public final native void wait(long timeout) throws InterruptedException;  
}
```

Thread类提供了方法来查询和设置中断状态。

```java
public class Thread {
  public boolean isInterrupted() {}
  public void interrupt() {}
}
```

当调用了一个线程的interrupt()方法后，当前线程的状态被置为中断状态，但是线程不会自动停止，需要我们来进行处理，停止线程的代码可以修改为：

```java
public class MyThread extends Thread {
  @Override
  public void run() {
    while (!isInterrupted()) {
      // do something
    }
  }
}
```

```java
MyThread thread = new MyThread();
thread.start();
thread.interrupt();
```

这里将while(!stop)修改为while (!isInterrupted())，停止方法由调用自定义的stop()方法改为调用Thread类实例对象的interrupt()方法。

实战中，我们不会在线程执行过程中每执行一步就判断一次isInterrupted()，所以更加贴近现实场景的情况是中断阻塞操作。前面讲过，这时会抛出异常，我们捕获异常并退出程序即可。

```java
public class MyThread extends Thread {
  @Override
  public void run() {
    while (!isInterrupted()) {
      try {
        Thread.sleep(1000);
      } catch (InterruptedException e) {
        break;
      }
    }
  }
}
```

我们以sleep()阻塞为例，wait()和join()也是一样的；当外部调用myThread.interrupt()时，发现MyThread正处于sleep的阻塞状态，会让sleep()抛出InterruptedException异常，我们捕获异常退出即可。

注意，如果外部调用interrupt()方法时，线程处于非阻塞状态，那么状态被设置为interrupted；如果线程处于阻塞状态，那么抛出异常，但是线程状态不会被修改。所以，我们可以在捕获异常后自行设置中断状态，统一退出，看上去是更优雅的做法，代码如下。

```java
public class MyThread extends Thread {
  @Override
  public void run() {
    while (!isInterrupted()) {
      try {
        Thread.sleep(1000);
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

> Thread.currentThread().interrupt()手工设置中断状态，然后通过isInterrupted()判断退出。

### 实战

原来停止或者中断一个线程如此麻烦，那实战中如何使用呢？

其实，实战反而简单。因为实战中我们通常只实现Runnable接口，并把任务提交给Executor去处理。虽然我们自己没有创建Thread对象，但最后线程池里面保存的肯定是Thread对象，也就是说Executor帮我们创建了Thread对象，它实现了一种中断机制来保证任务可以被中断或取消。

下面通过例子代码看看如何使用。

> TODO FutureTask的取消 

