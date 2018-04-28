---
title: 多线程基础篇(4) 线程同步
date: 2018-04-28 11:25:40
categories: 并发编程
tags: [并发, 线程, synchronized, wait, notify]
toc: true
description: Executor和ExecutorService为我们提供了线程异步执行的接口。其中比较重要的是submit()和shutdown()，分别实现任务的提交和线程池的关闭。
comments: false
---

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