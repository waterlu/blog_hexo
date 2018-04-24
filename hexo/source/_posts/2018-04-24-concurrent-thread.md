---
title: 多线程(1) 线程状态
date: 2018-04-24 09:47:39
categories: 并发编程
tags: [并发, 线程]
toc: true
description: 本文介绍并发中最基本的概念-线程，包括我对进程和线程的理解，线程的状态的变化。其中，理解线程的状态变化是关键点，深入理解线程状态迁移是理解并发锁的前提。最后，通过一个经典的生产者-消费者例子来说明线程状态和多线程之间的切换。
comments: false
---

## 线程

### 进程与线程

提到线程大家就会想起进程，先来看看进程和线程的联系和区别。

*什么是进程？*

通俗的讲，进程就是操作系统中运行的程序，当你的程序代码被操作系统加载到内存中运行的时候，它就成为了一个进程。以Linux系统为例，每个进程有自己的进程号(pid)，有独立的地址空间。每个进程有独立的地址空间意味着，一个进程不能够访问到另外一个进程的内存空间，内存地址空间是进程私有的。

> 进程间可以通过共享内存通信，共享内存空间并非进程私有的。
>
> 

*什么是线程?*

线程理解起来比进程要抽象一些。线程与操作系统的任务调用相关，可以简单认为线程是操作系统任务调用的最小单位。现在的大部分操作系统采用的都是时间片和抢占式的任务调用机制，当多个任务共享CPU资源时，一个任务执行一段时间后被挂起(这个任务执行的这段时间就称为时间片)，切换到另外一个任务开始执行。通常任务调度的单位不是进程，而是线程，也就是说，线程是进程可以独立交给CPU执行的一个任务。

这样，一个进程可以包含多个线程，每个线程作为一个任务交给操作系统调用。实际上，Java进程至少要包含一个线程，如果我们没有显示的创建线程，main()方法是在[main]线程中执行的。对于Java进程来说，每个线程有自己的栈空间，多个线程共享进程的堆空间。

### JAVA线程

通常有三种创建线程的方法：继承Thread类，实现Runnable接口和实现Callable接口。继承Thread类和实现Runnable接口都需要实现run()方法，实现Callable接口需要实现call()方法。继承Thread类的，直接调用start()方法就可以启动线程；实现Runnable接口的，需要new Thread(task).start()启动线程；实现Callable接口的，需要new Thread(new FutureTask<>(task)).start()启动线程。

> 实战中，我们不会直接使用new Thread().start()来启动线程，而是交给Executor去处理。因为每次new一个新的线程显然是低效的，放到线程池里执行是更好的选择。同样原因，实际编码中很少使用继承Thread类的办法来创建线程，因为Executor只接收接口。并且，由于Java不支持多重继承，实现Runnable接口显然是比继承Thread类更好的设计。
>
> 

实战中我们几乎不会使用直接继承Thread类的方式，剩下的要么实现Runnable，要么实现Callable，选择哪个取决于具体的业务需求。Callable与Runnable的不同之处在于，它可以返回结果，通俗点说，call()方法是有返回值的，run()方法是void的。所以，当需要线程执行完成后返回结果时，请选择Callable。

下面给出一个使用Callable的例子，计算1+2+...+10的和。

```java
public class ProcessThread {
    public static void main(String[] args) throws Exception {
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
        System.out.println("sum=" + futureTask.get().toString());
    }
}
```

>注意：虽然方法名字叫做start()，但是这个方法并不能保证线程立即开始执行，start()操作只能保证线程进入runnable状态，具体什么时间能够被执行由操作系统控制；futureTask.get()方法是同步等待的，等待Task的call()方法执行完成才返回。
>
>

线程是可以设置优先级的，常见的优先级有以下三种，默认为NORMAL_PRIORITY ：

-	MAX_PRIORITY = 10
  -NORMAL_PRIORITY = 5
  -MIN_PRIORITY = 1

理论上优先级高的线程先被执行，具体情况也要看操作系统如何调度。

## 线程状态

### 线程状态迁移

下面的这个线程状态迁移图是比较经典的，画的很详细。


![图1-线程状态迁移图](/images/thread_state.png)

线程的几个状态

- [new]：初始状态，新建一个线程对象后的状态
- [runnable]：就绪状态，调用了start()方法，等待CPU资源(操作系统调度)
- [running]：运行状态，得到了CPU时间片，执行run()方法或者call()方法
- [dead]：销毁状态，线程任务执行结束或者exit(1)强制退出
- [blocked]：阻塞状态，被动失去CPU时间片，等待进入runnable状态
- [waiting]：等待状态，主动失去CPU时间片，等待进入runnable状态

>waiting状态和blocked状态的区别在于一个是主动交出时间片，另外一个是被动交出时间片；当然前提是已经获得了时间片，得到了执行；从runnable状态到running状态是不受程序控制的，完全靠操作系统来调度，虽然我们可以设置线程的优先级，但是不一定达到预期。
>
>

状态变化的过程：

- [new]：MyThread thread = new MyThread();

- [new]->[runnable]：thread.start();

- [runnable]->[running]：操作系统调度，runnable状态意味着任务已经做好执行的准备，但什么时间真正执行（获得时间片）需要由操作系统来控制，任务本身无法控制；

- [running]->[dead]：线程执行完成；

- [running]->[runnable]：Thread.yield(); 调用yield()方法将失去时间片，但是可以立刻进入runnable状态，运气好的话可以立刻又获得时间片进入running状态；yield()给其他任务以执行的机会；

- [running]->[waiting]：

  - thread.join();调用join()方法将失去时间片，进入waiting状态，等到thread线程执行完毕后进入runnable状态；
  - object.wait(); 调用 wait()方法将失去时间片，进入waiting状态，得到其他线程执行object.notify()方法是进入runnable状态；
  - Thread.sleep(); 调用sleep()方法将失去时间片，进入waiting状态，sleep结束后进入runnable状态；

- [ruuning]->[blocked]：调用synchronized()将等待锁，进入blocked状态，得到锁以后进入runnable状态；

- [waiting]->[runnable]：

  - thread.join(); 其他线程执行完成后进入runnable状态；
  - object.wait(); 其他线程调用object.notify()或者object.notifyAll()后进入runnable状态；

- [blocked]->[runnable]：其他线程退出synchronized{}语句块以后进入runnable状态；




> 注意：区分Thread类和Object类的方法，区分静态方法和普通方法。
>
> 

### 生产者和消费者实例

通过经典的生产者和消费者的例子，可以看到Thread类和Object类如何配合，以及线程的生命周期和状态变化。

*需求*

- 两个线程，一个线程作为生产者生产商品，一个线程作为消费者消费商品；
- 商品有库存，当库存满时生产者暂停生产，当库存空时消费者暂停消费。

*实现*

- 主线程，创建库存列表，启动生产者线程和消费者线程

```java
public class ProducerAndConsumer {
    public static void main(String[] args) {
        LinkedList<String> storeList = new LinkedList<>();
      	// 生产者和消费者共享库存列表
        Producer producer = new Producer(storeList);
        Consumer consumer = new Consumer(storeList);
        producer.start();
        consumer.start();
    }
}
```

- 生产-消费逻辑
  - 开始库存是空的，先生产一个数据
  - 此后如果可以抢到synchronized锁，可以继续生产数据，直到库存满
  - 库存满了以后主动让出时间片，进入waiting状态，等待消费者notify()
  - 如果生产完第一个数据后没有抢到synchronized锁，那么进入blocked状态，等待消费者退出synchronized代码块；当消费者完成一次消费操作退出synchronized代码块后，生产者线程进入runnable状态，等待CPU调度；抢到CPU时间片后可以继续生产
  - 如果生产者线程一直没有抢到CPU时间片，那么消费者一直消费（一次消费一个），直到库存空
  - 库存空以后消费者交出时间片，生产者获得时间片，继续生产数据
  - 生产数据后通过notify()方法唤醒消费者线程
  - 生产者notify()方法只能保证消费者线程进入runnable状态，但不一定能抢到时间片执行
  - 大家都是runnable状态时，谁能得到CPU时间片就看人品了，操作系统就可以这么任性

- 生产者

```java
public class Producer extends Thread {
    // 库存最大容量
    private final static int MAX = 10;
    // 计数器，方便观察
    private AtomicInteger count = new AtomicInteger(1);
    // 库存
    private LinkedList<String> storeList;

    public Producer(LinkedList<String> storeList) {
        this.storeList = storeList;
    }

    @Override
    public void run() {
      	// 获得时间片，进入running状态
        while(true) {
            // 必须在synchronized代码块内使用wait()和notify()
            // 如果storeList已经被其他线程锁定，进入blocked状态
            // 如果storeList没有被其他线程锁定，继续running状态
            synchronized (storeList) {
                while(storeList.size() == MAX) {
                    System.out.println("库存满了");
                    try {
                      	// 库存满了，不能再生产了
                      	// 让出时间片，进入waiting状态
                      	// 操作系统负责保存当前线程状态，当恢复时从这里继续执行
                        storeList.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
              	// storeList.size() < MAX 时，直接生产一个新数据
              	// storeList.size() = MAX 时，消费者调用storeList.notify()时,
              	// 进入runnable状态，获得时间片后继续执行，生产一个新数据
                String data = String.format("%04d", count++);
                storeList.add(data);
                System.out.println("生产 " + data);
              	// 让出时间片，进入waiting状态              
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
              	// Sleep 500毫秒后进入runnable状态，获得时间片后继续执行
              	// 通知消费者，如果消费者由于库存空了进入waiting状态，此时将被唤醒
                storeList.notifyAll();
              	// 此后重新尝试进入synchronized代码块
            }
        }
    }
}
```

- 消费者

```java
public class Consumer extends Thread {
    // 库存
    private LinkedList<String> storeList;

    public Consumer(LinkedList<String> storeList) {
        this.storeList = storeList;
    }

    @Override
    public void run() {
      	// 获得时间片，进入running状态
        while(true) {
            // 如果storeList已经被其他线程锁定，进入blocked状态
            // 如果storeList没有被其他线程锁定，继续running状态          
            synchronized (storeList) {
                while(storeList.size() == 0) {
                    System.out.println("库存空了");
                    try {
                      	// 库存空了，不能再消费了
                      	// 让出时间片，进入waiting状态                      
                        storeList.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
              	// storeList.size() > 0 时，直接消费一个数据
              	// storeList.size() = 0 时，生产者调用storeList.notify()时,
              	// 进入runnable状态，获得时间片后继续执行，消费一个数据
                String data = storeList.get(0);
                System.out.println("消费 " + data);
                storeList.remove(0);
                // 让出时间片，进入waiting状态
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 通知生产者，如果生产者由于库存满了进入waiting状态，此时将被唤醒
                storeList.notifyAll();
              	// 此后重新尝试进入synchronized代码块
            }
        }
    }
}
```

- 下表列出了输出结果和对应的线程状态

| 日志输出    | Producer PC       | Producer状态 | Consumer PC         | Consumer状态 |
| ------- | ----------------- | ---------- | ------------------- | ---------- |
| 生产 0001 | storeList.add();  | running    |                     | runnable   |
| 消费 0001 | synchronized ()   | blocked    | storeList.remove(); | running    |
| 库存空了    | synchronized ()   | blocked    | storeList.wait();   | waiting    |
| 生产 0002 | storeList.add();  | running    | storeList.wait();   | waiting    |
| 生产 0003 | storeList.add();  | running    | storeList.wait();   | waiting    |
|         |                   |            | synchronized ()     | blocked    |
| 消费 0002 | synchronized ()   | blocked    | storeList.remove(); | running    |
| 消费 0003 | synchronized ()   | blocked    | storeList.remove(); | running    |
| 库存空了    | synchronized ()   | blocked    | storeList.wait();   | waiting    |
| 生产 0004 | storeList.add();  | running    | storeList.wait();   | waiting    |
| ......  |                   |            |                     |            |
| 生产 0013 | storeList.add();  | running    | synchronized ()     | blocked    |
| 库存满了    | storeList.wait(); | waiting    | synchronized ()     | blocked    |
| 消费 0004 | storeList.wait(); | waiting    | storeList.remove(); | running    |
| 消费 0005 | synchronized ()   | blocked    | storeList.remove(); | running    |
| 消费 0006 | synchronized ()   | blocked    | storeList.remove(); | running    |
| 生产 0014 | storeList.add();  | running    | synchronized ()     | blocked    |
| 生产 0015 | storeList.add();  | running    | synchronized ()     | blocked    |
|         |                   |            |                     |            |

> 注意：每生产或者消费一个数据以后都通过storeList.notify()来唤醒对手方的storeList.wait()，但是唤醒只能保证对手方线程进入runnable状态；这个时候继续执行当前线程(notify)，还是切换到等待线程(wait)是由操作系统来调度的，是随机的。此外，由于synchronized的存在，多CPU情况下生产者和消费者也不会长期同时处于running状态，短暂同时处于running状态后，后进入synchronized代码块的线程将进入blocked状态，等待前进入的线程退出synchronized代码块。
>
> 

## 线程实现

Java线程的实现有三种策略：

- 内核实现，Java线程相关方法声明为native，所有操作直接调用操作系统内核的API，此时的Java线程与操作系统内核线程一一对应；
- 用户实现，Java线程对操作系统是透明的，自己实现线程调度；
- 混合实现，综合上面两种实现的折中方案。

Java线程在JDK1.2之前采用用户实现策略，之后替换为内核实现。也就是说，一个Java线程就对应操作系统一个实际的线程（轻量级进程）。

Java线程使用抢占式的调度策略，由操作系统来分配执行时间。由于各个操作系统的线程优先级粒度不一样，所以Java线程优先级设置很难与操作系统线程优先级一一对应，所以就不那么靠谱。

![Java线程实现](/images/thread_os.png)

上图中，KLT是Kernel Thread的缩小，表示这是一个操作系统内核线程；LWP是Light Weight Process的缩写，表示轻量级进程。应用程序一般不直接使用内核线程，而是使用轻量级进程作为接口；P表示一个Java进程。

所以，上图的含义为：一个Java进程中可以创建多个线程，每个线程调用LWP接口，实际对应着一个内核线程，内核线程由操作系统调度。

Linux 2.6以后有了一种新的pthread线程库--NPTL(Native POSIX Threading Library)，hotspot虚拟机就是使用NPTL来实现多线程的。