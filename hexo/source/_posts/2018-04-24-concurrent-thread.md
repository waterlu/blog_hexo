---
title: 多线程基础篇(1) 线程
date: 2018-04-24 09:47:39
categories: 并发编程
tags: [并发, 线程]
toc: true
description: 本文介绍并发中最基本的概念-线程，包括我对进程和线程的理解，以及如何创建并运行一个新的线程。
comments: false
---

## 进程与线程

提到线程大家就会想起进程，先来看看进程和线程的联系和区别。

**什么是进程？**

通俗的讲，进程就是操作系统中运行的程序，当你的程序代码被操作系统加载到内存中运行的时候，它就成为了一个进程。以Linux系统为例，每个进程有自己的进程号(pid)，有独立的地址空间。每个进程有独立的地址空间意味着，一个进程不能够访问到另外一个进程的内存空间，内存地址空间是进程私有的。

> 进程间可以通过共享内存通信，共享内存空间并非进程私有的。
>

**什么是线程?**

线程理解起来比进程要抽象一些。线程与操作系统的任务调用相关，可以简单认为线程是操作系统任务调度的基本单位。现在的大部分操作系统采用的都是时间片和抢占式的任务调用机制，当多个任务共享CPU资源时，一个任务执行一段时间后被挂起(这个任务执行的这段时间就称为时间片)，切换到另外一个任务开始执行。通常任务调度的单位不是进程，而是线程，也就是说，线程是进程可以独立交给CPU执行的一个任务。线程有自己的栈和程序计数器。

这样，一个进程可以包含多个线程，每个线程作为一个任务交给操作系统调用。实际上，Java进程至少要包含一个线程，如果我们没有显示的创建线程，main()方法是在[main]线程中执行的。对于Java进程来说，每个线程有自己的栈空间，多个线程共享进程的堆空间。

## JAVA线程

### 创建线程

通常有三种创建线程的方法：继承Thread类，实现Runnable接口和实现Callable接口。继承Thread类和实现Runnable接口都需要实现run()方法，实现Callable接口需要实现call()方法。以下为示例代码。

**Thread类**

```java
public class Task extends Thread {
  @Override
  public void run() {
    int sum = 0;
    for (int i=1; i<=10; i++) {
      sum += i;
    }
    System.out.println("sum=" + sum);
  }
}
```

**Runnable接口**

- 注意，run()方法没有返回值，且不抛出异常

```java
public class Task implements Runnable {
  @Override
  public void run() {
    int sum = 0;
    for (int i=1; i<=10; i++) {
      sum += i;
    }
    System.out.println("sum=" + sum);
  }
}
```

**Callable接口**

- 注意，call()方法有返回值，而且可以抛出异常，这是Callable和Runnable最重要的区别

```java
public class Task implements Callable<Integer> {
  @Override
  public Integer call() throws Exception {
    int sum = 0;
    for (int i=1; i<=10; i++) {
      sum += i;
    }
    return sum;
  }
}
```

### 启动线程

继承Thread类的，直接调用start()方法就可以启动线程；

```java
public class Task extends Thread {}
Task task = new Task();
task.start();
```

实现Runnable接口的，需要new Thread(task).start()启动线程；

```java
public class Task implements Runnable {}
Task task = new Task();
new Thread(task).start();
```

>  注意：虽然方法名字叫做start()，但是这个方法并不能保证线程立即开始执行，start()操作只能保证线程进入runnable状态，具体什么时间能够被执行由操作系统控制。

也可以使用匿名内部类

```java
Thread task = new Thread(new Runnable() {
  @Override
  public void run() {
    System.out.println("run");
  }
});
task.start();
```

实现Callable接口的，需要new Thread(new FutureTask<>(task)).start()启动线程。

```java
FutureTask<Integer> futureTask = new FutureTask<>(new Callable<Integer>() {
  @Override
  public Integer call() throws Exception {
    int sum = 0;
    for (int i=1; i<=10; i++) {
      sum += i;
    }
    return sum;
  }
});
new Thread(futureTask).start();
```

注意：虽然直接调用run()方法看上去也可以有正确的结果，但是这是在当前线程内执行的，没有启动一个新的线程来执行。

> 实战中，我们通常不会直接使用new Thread().start()来启动线程，而是交给Executor去处理。因为每次new一个新的线程显然是低效的，放到线程池里执行是更好的选择。同样原因，实际编码中很少使用继承Thread类的办法来创建线程，因为Executor只接收接口。并且由于Java不支持多重继承，实现Runnable接口是比继承Thread类更好的设计。
>

### 等待线程

有时我们需要在主线程中等待新线程执行完成，并获得执行结果，这个时候可以使用join()方法。调用join()方法后当前线程被挂起，直到join线程执行完成后再被唤醒。

```java
public class ThreadJoin {
  private final static Logger LOGGER = LoggerFactory.getLogger(ThreadJoin.class);
  public static void main(String[] args) throws Exception {
    Task1 task1 = new Task1();
    Thread thread1 = new Thread(task1);
    thread1.start();
    thread1.join();
    LOGGER.info("done");
  }
  public static class Task1 implements Runnable {
    @Override
    public void run() {
      int sum = 0;
      for (int i=1; i<=10; i++) {
        sum += i;
      }
      LOGGER.info("sum=" + sum);
    }
  }
}
```

上面为示例代码，执行结果如下

```shell
13:40:15:944 INFO [Thread-1] sum=55
13:40:15:944 INFO [main] done
```

如果去掉thread1.join()，那么执行结果如下

```shell
13:41:21:690 INFO [main] done
13:41:21:690 INFO [Thread-1] sum=55
```

通常，Runnable接口需要调用join()方法，Callable接口就不需要了，直接调用FutureTask的get()方法就可以获取执行结果。FutureTask.get()方法起到和join()相同的效果，当前线程阻塞，直到子线程执行完成并返回执行结果，以下为示例。

```java
public class ThreadJoin {
  private final static Logger LOGGER = LoggerFactory.getLogger(ThreadJoin.class);
  public static void main(String[] args) throws Exception {
    Task2 task2 = new Task2();
    FutureTask<Integer> futureTask = new FutureTask<>(task2);
    Thread thread2 = new Thread(futureTask);
    thread2.start();
    LOGGER.info(futureTask.get().toString());
  }
  public static class Task2 implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
      int sum = 0;
      for (int i=1; i<=10; i++) {
        sum += i;
      }
      return sum;
    }
  }
}
```

以上代码开启一个子线程计算1到10的和，执行结果如下；FutureTask.get()方法是同步等待的，等待Task的call()方法执行完成才返回。

```shell
13:47:02:066 INFO [main] 55
```

### 中断线程

有时我们可能由于某种原因中断子线程任务的执行，在其执行完成前退出。通常用两种方法：一是自己设置标志位，根据标志位退出；二是调用interrupt()方法。

**标志位**

先来看使用标志位退出的代码

```java
public class ThreadStop {
  private final static Logger LOGGER = LoggerFactory.getLogger(ThreadStop.class);
  public static void main(String[] args) {
    Task task = new Task();
    Thread thread = new Thread(task);
    thread.start();
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    task.stop();
  }

  public static class Task implements Runnable {
    private boolean stop = false;
    @Override
    public void run() {
      while (!stop) {
        LOGGER.info("run");
        try {
          Thread.sleep(1000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
      LOGGER.info("stop");
    }
    public void stop() {
      stop = true;
    }
  }
}

```

执行结果如下，在执行过程中不断判断标志位stop，当发现stop被置为true后退出。

```shell
14:02:51:483 INFO [Thread-1] run
14:02:52:483 INFO [Thread-1] run
14:02:53:496 INFO [Thread-1] stop
```

**中断**

通过标志位退出线程有自己的局限性，如果线程调用了阻塞方法（例如wait()），并且无法被唤醒，那么就无法通过判断标志位来实现退出了。这种情况下，我们可以调用Thread的interrupt()方法中断阻塞方法，实现退出。

```java
public class ThreadStop {
    private final static Logger LOGGER = LoggerFactory.getLogger(ThreadStop.class);
    public static void main(String[] args) {
        Task task = new Task();
        Thread thread = new Thread(task);
        thread.start();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread.interrupt();
    }

    public static class Task implements Runnable {
        @Override
        public void run() {
            while (true) {
                if (Thread.currentThread().isInterrupted()) {
                    LOGGER.info("interrupted");
                    break;
                }
                LOGGER.info("run");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    LOGGER.error("interrupted in sleep");
                    break;
                }
            }
        }
    }
}
```

执行结果如下，thread.interrupt()方法中断了阻塞的Thread.sleep()方法，抛出InterruptedException异常；线程捕获异常后退出，注意这里需要手动退出。

```shell
14:11:43:574 INFO [Thread-1] run
14:11:44:580 ERROR [Thread-1] interrupted in sleep
14:11:44:580 INFO [Thread-1] interrupted
```

### 线程优先级

创建一个线程以后，可以设置线程的优先级。线程优先级是从1到10的数字，数字越大优先级越高，常用的优先级有MIN_PRIORITY、MAX_PRIORITY和NORMAL_PRIORITY三种，其中默认值为NORMAL_PRIORITY ：

```java
public final static int MIN_PRIORITY = 1;
public final static int NORM_PRIORITY = 5;
public final static int MAX_PRIORITY = 10;
```

以下为示例代码

```java
MyThread thread = new MyThread();
thread.setPriority(Thread.MAX_PRIORITY);
thread.start();
```

理论上优先级高的线程先被执行，但不一定绝对这样，具体情况要看操作系统如何调度。总体上，优先级高的线程有更多的机会被执行。





