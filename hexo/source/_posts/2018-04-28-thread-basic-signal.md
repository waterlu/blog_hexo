---
title: 多线程基础篇(4) 线程同步
date: 2018-04-28 11:25:40
categories: 并发编程
tags: [并发, 线程, synchronized, wait, notify]
toc: true
description: 我们已经了解如何创建一个线程，以及如何交给线程池去调度，接下来看看线程之间同步的问题。本文详细描述了如何使用synchronized和wait/notify实现线程同步。最后通过生产者-消费者的例子进行了深入讨论。
comments: false
---

## 线程同步

我们已经了解如何创建一个线程，以及如何交给线程池去调度，接下来看看线程之间同步的问题。之所以引入线程同步的原因是临界资源和竞争条件，我们先不讨论，放到后面去了解，这里先来看如何实现线程同步。

### synchronized

关键字synchronized用来标识同步语句块，同步语句块同一时间只能有一个线程执行，其他线程都必须等这个线程执行完成，起到了同步锁的作用。并发操作添加synchronized后意味着变成了串行操作，一次只能执行一个。

#### 四种同步方法

synchronized可以用在以下四个位置进行同步

- 类的成员方法上
- 静态方法上
- 类成员方法内部的语句块上
- 静态方法内部的语句块上

synchronized语义可以理解为对一个对象的锁定，同一时刻只有一个线程可以操作这个对象，其他线程被锁定。或者说，synchronized语句意味着synchronized代码块与一个对象进行了关联，需要先锁定这个对象，然后才能执行synchronized代码块，当然这个对象只能被锁定一次，当退出synchronized代码块时可以解锁这个对象。

下面依次来看一下synchronized关键字放置在不同位置的效果

**类成员方法**

```java
public class Math() {
  public synchronized void add(int value){
    this.count += value;
  }
}
```

上述代码意味着，synchronized语句块（this.count += value;）的执行，必须先要获得Math类实例对象的锁，然后才可以执行。当两个线程同时执行一个Math对象的add()方法，后执行的线程加锁失败，必须等到先执行的线程操作完成，退出synchronized语句块解锁后才能开始执行。

> 注意，锁定的是对象，不是类，如果两个线程创建了两个Math对象，那么是可以同时调用它们的add()方法的。

**静态方法**

```java
public class Math() {
  public static synchronized void add(int value){
    Math.count += value;
  }
}
```

可以认为静态方法同步锁的是类对象（区分类对象和实例对象），所以两个线程对静态同步方法的调用一定是互斥的。

**方法内部语句块**

```java
public class Math {
  public void add(int value){
    synchronized(this) {
      this.count += value;
    }
  }
}
```

其效果和第一种把synchronized加到类成员方法上面是一样的。其实，这样写更明显，可以认为是第一种的翻译。

**静态方法内部语句块**

```java
public class Math {
  public void add(int value){
    synchronized(Math.this) {
      this.count += value;
    }
  }
}
```

同上，这种写法和第三种是一样的，效果也一样。

> 思考一下：如果一个类有两个方法，一个静态，一个非静态，都加上synchronized关键字，可以同时执行吗？答案是可以。因为一个锁定的是实例对象this，另外一个锁定的是类对象Math.this，不是一个对象，所以可以同时执行。

**Object对象**

synchronized关键字不一定非要修饰this，也可以自己指定任何一个Object对象实例，如下也可行：

```java
public class Math {
  private Object lock = new Object();
  public void add3(int value){
    synchronized(lock) {
      this.count += value;
    }
  }
}
```

#### 选用哪种方法

有这么多种同步方法，实战中我们应该选择哪一种呢。个人建议选择“方法内部语句块”的同步方法，这样同步对象非常明确，不容易引起混淆。此外，同步语句块的颗粒度也可以小于同步方法，减小同步语句块通常也是提升性能的方法之一。

#### 字符串常量

再来多思考一层，像下面这样写行不行呢？

```java
public class Math {
  private String lock = "lock";
  public void add(int value){
    synchronized(lock) {
      this.count += value;
    }
  }
}
```

说行也对，应该不会报错；说不行也对，因为这样有隐患，强烈不建议这么用。详细说一下，如果只有一个类这么用是不会出错的，但是如果有两个类这么用就出问题了。

```java
public class Math {
  private String lock = "lock";
  public void add(int value){
    synchronized(lock) {
      this.count += value;
    }
  }
}
public class Work {
  private String lock = "lock";
  public void work(int value){
    synchronized(lock) {
      //doSomething
    }
  }
}
```

如上，完全不相干的两个类，add()方法和work()的调用将会互斥，因为两个方法都将试图锁定相同的字符串实例。这里多说一下，理论上讲，虽然字符串内容一样，但是Math和Work两个类中的lock应该是不同的对象；但是，为了效率，Java提供了常量池的概念，导致两个类中的lock对象都是常量池中"lock"的引用，所以就一样了。

> 正常情况，String也是一个普通的类，那么lock作为String类的实例对象，应该在堆上分配空间，并且 "lock" 应该保存在堆空间上。如果是这样的话，两个类中的lock对象就不一样了。例如：把String lock = "lock"换成 Object lock = new Object()就没有问题了。但是实际情况是，创建String对象时，字符串内容没有保存到堆空间上，而是在方法区开辟了一块空间来保存，这块空间也被成为常量池。常量池是有去重逻辑的，当创建第二个lock对象时，发现常量池中已经存在就不会再创建了，直接返回已有常量字符串。

### wait/nofity/notifyAll

有了synchronized以后，其实我们已经可以进行线程通信了。例如：我们可以创建一个线程共享对象MySignal，一个线程在完成工作后，修改hasDataToProcess标志，另外一个线程轮询这个标志位，当第一个线程任务完成后第二个线程开始自己的任务。

```java
public class MySignal{
  protected boolean hasDataToProcess = false;
  public synchronized boolean hasDataToProcess(){
    return this.hasDataToProcess;
  }
  public synchronized void setHasDataToProcess(boolean hasData){
    this.hasDataToProcess = hasData;  
  }
}
```

以上模型有一个明显的问题，就是第二个线程需要一直查询，占用CPU时间，所以更常用的线程间通信机制是wait和notify，调用wait()方法后当前线程进入阻塞，让出CPU时间片。

调用wait()方法将使当前线程进入等待状态，直到有其他线程调用了notify()方法后唤醒。wait()有点像sleep()，但是也有很大的区别。

首先，一定要注意sleep()是Thread类的静态方法，wait()是Object类的成员方法，这非常重要。虽然wait和notify是用来实现线程通信的，但是它们并不是Thread类的方法。其实这也非常好理解，假设实现方案是调用Thread类的wait()方法进入等待状态，那么如何把它唤醒呢，肯定还得调用这个Thread类对象的notify()方法来唤醒；那么我们知道肯定需要在其他线程中完成某一项工作后唤醒这个线程，那就意味着另外一个线程得拥有这个线程的类实例对象，这明显是不合适的。退一步，两个线程之间共享对象就合理多了。

其次，wait()和notify()方法必须在synchronized代码块内执行，也就是说，首先你得拥有这个对象的锁，然后才可以调用wait()和notify()方法。解析一下，通常的逻辑如下：一个线程获得Object对象锁进入synchronized代码块开始自己的工作，需要时调用wait()方法进入等待；这个时候另外一个线程获得Object对象锁进入synchronized代码块，执行自己的任务，任务完成后调用notify()方法，唤醒正在等待的线程。其实，以上就是生产者-消费者基本的处理逻辑。

最后，调用一个对象的notify()方法后，将唤醒一个在这个对象上wait()的线程，如果有多个现线正在wait()，那么由操作系统来决定唤醒哪一个，可以认为是随机的；如果调用的是notifyAll()方法，那么这个对象上wait()的全部线程都将被唤醒。

> 需要注意，notify()唤醒只是意味着wait()线程进入runnable状态，也就是可以执行的状态，至于什么时间能够被执行，还是看操作系统调度。换句话说，notify()的确可以唤醒线程，但是被唤醒线程什么时间能够被执行是没有保障的，运气不好的话，如果CPU一直在忙，那么被唤醒线程也可能很久都得不到执行。

下面来看一个例子，这里只展示基本用法，通过生产者和消费者的例子可以看的更清楚

```java
public class MonitorObject{
}
public class MyWaitNotify{
  MonitorObject myMonitorObject = new MonitorObject();
  public void doWait(){
    synchronized(myMonitorObject){
      try{
        myMonitorObject.wait();
      } catch(InterruptedException e){...}
    }
  }
  public void doNotify(){
    synchronized(myMonitorObject){
      myMonitorObject.notify();
    }
  }
}
```

#### 丢失问题

这里同样存在使用常量字符串做为同步对象的问题。

```java
public class WaitNotify extends Thread {
  private String myMonitorObject = "lock";
  boolean wasSignalled = false;
  @Override
  public void run() {
    doWait();
  }
  public void doWait(){
    System.out.println("notify doWait start");
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
        } catch(InterruptedException e){
        }
      }
      wasSignalled = false;
    }
    System.out.println("notify doWait done");
  }
  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```

```java
public class WaitNotify2 extends Thread {
  private String myMonitorObject = "lock";
  boolean wasSignalled = false;
  @Override
  public void run() {
    doWait();
  }
  public void doWait(){
    System.out.println("notify2 doWait start");
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
        } catch(InterruptedException e){

        }
      }
      wasSignalled = false;
    }
    System.out.println("notify2 doWait done");
  }
  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```

```java
public class ThreadSignal {
  public static void main(String[] args) throws InterruptedException {
    WaitNotify notify = new WaitNotify();
    WaitNotify2 notify2 = new WaitNotify2();
    notify.start();
    notify2.start();
    Thread.sleep(1000);
    System.out.println("send signal");
    notify.doNotify();
  }
}
```

由于WaitNotify和WaitNotify2实际上使用的是一个锁对象，所以实际上现在myMonitorObject对象上有两个wait()，当我们调用notify.doNotify()时，会随机选择一个来唤醒。如果唤醒的是WaitNotify，那么没有问题；如果唤醒的是WaitNotify2，那么就出问题了。

## 生产者和消费者实例

通过经典的生产者和消费者的例子，可以看到Thread类和Object类如何配合，以及线程的生命周期和状态变化。

### 需求

- 两个线程，一个线程作为生产者生产商品，一个线程作为消费者消费商品；
- 商品有库存，当库存满时生产者暂停生产，当库存空时消费者暂停消费。

### 实现

#### 主线程

- 创建库存列表，启动生产者线程和消费者线程

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

#### 生产-消费逻辑

- 开始库存是空的，先生产一个数据
- 此后如果可以抢到synchronized锁，可以继续生产数据，直到库存满
- 库存满了以后主动让出时间片，进入waiting状态，等待消费者notify()
- 如果生产完第一个数据后没有抢到synchronized锁，那么进入blocked状态，等待消费者退出synchronized代码块；当消费者完成一次消费操作退出synchronized代码块后，生产者线程进入runnable状态，等待CPU调度；抢到CPU时间片后可以继续生产
- 如果生产者线程一直没有抢到CPU时间片，那么消费者一直消费（一次消费一个），直到库存空
- 库存空以后消费者交出时间片，生产者获得时间片，继续生产数据
- 生产数据后通过notify()方法唤醒消费者线程
- 生产者notify()方法只能保证消费者线程进入runnable状态，但不一定能抢到时间片执行
- 大家都是runnable状态时，谁能得到CPU时间片就看人品了，操作系统就可以这么任性

#### 生产者线程

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

#### 消费者线程

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

#### 输出结果分析

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

