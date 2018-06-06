---
title: 【置顶】Java多线程一篇就够
date: 2018-05-21 18:01:50
categories: 并发编程
tags: [多线程]
toc: true
description: Java多线程知识点汇总，全部干货。
comments: true
top: true
---

## 线程

线程是操作系统任务调度的基本单位，Java进程至少有一个[main]线程。需要知道如何创建和启动一个线程，如何停止线程以及线程常见的属性。

### 启动线程

```java
public class Task extends Thread {
  @Override
  public void run() {}
}
new Task().start();
```

```java
public class Task implements Runnable {
  @Override
  public void run() {}
}
Thread thread = new Thread(new Task());
thread.start();
thread.join();
```

```java
public class Task implements Callable<Integer> {
  @Override
  public Integer call() throws Exception {
    return 1;
  }
}
FutureTask<Integer> futureTask = new FutureTask<Integer>(new Task());
new Thread(futureTask).start();
int value = futureTask.get();
```

`new Thread()`  只接受Runnable参数，`FutureTask implements Runnable` 。

FutureTask.get()方法起到和Thead.join()相同的效果，阻塞当前线程直到子线程执行完成并返回执行结果。

> `Thread.currentThread()` 静态方法获取当前线程。

### 停止线程

使用标志位主动退出线程，需要自己编码实现

```java
public class Task implements Runnable {
  private volatile boolean stop = false;
  @Override
  public void run() {
    while (!stop) {}
  }
}
```

能主动退出是最好的，但不是所有情况都可以主动退出，例如：调用了wait()方法后当前线程被阻塞，就无法判断标志位了，这个时候需要通过外部调用Thread.interrupt()方法来中断线程。

```java
public class Task implements Runnable {
  @Override
  public void run() {
    while (true) {
      if (Thread.currentThread().isInterrupted()) {
        break;
      }
      try {
        // Thread.sleep();
        // Object.wait();
      } catch (InterruptedException e) {
        break;
        // Thread.currentThread().interrupt();
      }
    }
  }
}
Thread thread = new Thread(new Task());
thread.start();
thread.interrupt();
```

> 注意这种代码写法：如果线程正常执行中，那么可以判断isInterrupted()标志位；如果线程阻塞中，外部调用interrupt()方法的效果是中断阻塞操作，抛出InterruptedException异常，捕获异常后需要自己处理退出。换句话说，调用Thread.interrupt()方法后系统不会杀死这个线程，如果当前线程是running状态，系统修改isInterrupted()标志位，如果当前线程是blocked状态，系统抛出InterruptedException异常。捕获异常后，我们可以选择直接退出`break` ，也可以`Thread.currentThread().interrupt();` 重置标志位退出。

### 线程属性

每个线程可以设置一个名字，主线程默认名字是[main]，线程池启动线程默认名字是[pool-1-thread-1]。

线程可以设置优先级，最大MAX_PRIORITY=10，最小MIN_PRIORITY=1，默认NORM_PRIORITY=5。

通过设置daemon属性可以将一个线程设置为守护线程，与守护线程相对的是用户线程，当所有用户线程结束的时候，守护线程自动结束。GC线程是典型的守护线程，用户线程没了，GC线程也没有存在的意义了。

## 线程状态

![](/images/thread_state.png)

如果我们输出Thread.currentThread().getState()，与上图是对不上的，下面是源码中线程状态的定义。

```java
public enum State {
  NEW,
  RUNNABLE,
  BLOCKED,
  WAITING,
  TIMED_WAITING,
  TERMINATED;
}
```

getState()不区分RUNNABLE和RUNNING状态，我们只能通过日志输出NEW，RUNNABLE和TERMINATED三个状态。

## 线程池

实战中一般不会使用`new Thread().start()` 来启动一个线程，通常都使用线程池。使用线程池的原因是创建和销毁线程是有开销的，所以当然就会想到线程的复用，一次创建，多次使用。同时线程池也可以保证在突发大量任务时不会因为创建大量线程耗尽系统资源。

### ExecutorService

线程池实现ExecutorService接口，主要有以下四个启动线程的方法：

- execute(Runnable)，不关注返回值，也不与启动的线程交互；
- submit(Runnable)，得不到返回值，但可以通过Future与启动的线程交互；
- submit(Runnable, T)，Runnable非要得到返回值（直接用Callable吧）
- submit(Callable)，有返回值，可以通过Future与启动的线程交互。

```java
public interface ExecutorService extends Executor {
  void execute(Runnable command);	// 从Executor继承
  Future<?> submit(Runnable task);
  <T> Future<T> submit(Runnable task, T result);
  <T> Future<T> submit(Callable<T> task);  
}
```

一般常用的就两种，启动任务就行，那就execute(Runnable)；关心返回值，需要交互，那就submit(Callable)。

```java
public class Task implements Callable<Integer> {
  @Override
  public Integer call() throws Exception {
    return 1;
  }
}
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(new Task());
Integer result = future.get();
```

> 不用创建FutureTask了，Callable用线程池更直观。

### Executors

使用Executors可以得到ExecutorService接口，常见的有如下三种。Executors实际创建ThreadPoolExecutor实例，但屏蔽了参数细节，不建议使用，建议自己创建ThreadPoolExecutor。

```java
public class Executors {
  // 线程池中就1个核心线程，忙就排队
  public static ExecutorService newSingleThreadExecutor() {}  
  // 核心线程数是nThreads的线程池，无界排队队列
  public static ExecutorService newFixedThreadPool(int nThreads) {} 
  // 核心线程数0，接收任务就创建新线程执行
  public static ExecutorService newCachedThreadPool() {}
}
```

> newSingleThreadExecutor()和newCachedThreadPool()基本无实战价值。

### ThreadPoolExecutor

几个最重要的参数：

- corePoolSize，核心线程数
- maximumPoolSize，最大线程数
- workQueue，排队策略
- handler，饱和策略

![](/images/threadpool_threadpoolexecutor.png)

首先，当线程池中的线程数小于核心线程数时，会为新任务创建新线程，直到达到核心线程数；核心线程完成任务后不会回收；当有新任务到来时，从线程池中选择空闲的核心线程来执行任务，当无核心线程空闲时进入排队逻辑。

#### 排队逻辑

ArrayBlockingQueue

有界队列，核心线程无空闲时开始排队，队列满以后开启非核心线程，直到最大线程数，超过最大线程数进入饱和逻辑。

LinkedBlockingQueue

无界队列，核心线程无空闲时开始排队，由于队列无界，会一直排队到资源耗尽，最大线程数不起作用。

SynchronousQueue

零界队列，相当于长度为0的队列，不排队直接开启非核心线程，超过最大线程数进入饱和策略。

#### 非核心线程

开启的非核心线程是可以回收的，当空闲时间超过设定时间keepAliveTime+unit即可回收。线程池不会标记哪个线程是核心线程，哪个线程是非核心线程，线程池关心的是数量。

#### 饱和逻辑

AbortPolicy

默认的饱和策略，抛出RejectedExecutionException异常。

DiscardPolicy

丢弃当前任务，就当没提交过。

CallerRunsPolicy

在调用者线程中执行run()方法。

> 我觉得默认就很好，一般就不要修改饱和策略了。

## 线程同步

前面一直关注如何创建和启动线程来完成任务，线程同步关心的是多个线程之间通信的交互的事情。这部分的难度要大一些。

### 竞态条件

首先引入竞态条件和临界区的概念，当多个线程同时访问共享变量时，如果执行顺序不同导致结果可能不同，那么我们就说存在竞态条件（Race Condition），引起竞态条件的代码就称为临界区（Critical Section）。

线程同步的三个问题：原子性、可见性和有序性。

### synchronized

synchronized关键字起到锁的作用，可以保证原子性、可见性和有序性。synchronized语句块同时只能有一个线程执行，其他线程阻塞等待。synchronized本质上是对一个对象加锁，可以显示声明这个对象，也可以是隐式的。synchronized修饰类的成员方法，相当于对类的实例对象加锁；synchronized修饰类的静态方法，相当于对类对象加锁。

```java
public synchronized int add(int value) {}
```

等价于

```java
public int add(int value) {
  synchronized(this) {}
}
```

同样的，

```java
public class Math {
  public static synchronized void add(int value) {}
}
```

等价于

```java
public class Math {
  public static void add(int value) {
    synchronized(Math.this) {}
  }
}
```

#### 偏向锁

Synchronized实现原理：每个对象有一个监视器锁（monitor），当monitor被占用时就会处于锁定状态。进入同步代码块是通过monitorenter指令获取锁，退出同步代码块时通过monitorexit指令释放锁。monitor本身基于操作系统的Mutex和Lock来实现，成本高，所以通常我们称Synchronized为重量级锁。

但是，从JDK1.6开始逐步优化Synchronized的实现，将实现方式分为三级：偏向锁、轻量级锁和重量级锁。加锁过程是一个失败后的升级过程：先尝试加偏向锁，不成功再尝试加轻量级锁，还不成功才使用重量级锁。

轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能,以下为个人理解。

偏向锁相当于在Object对象锁上记录线程ID，实际上就是最后一个锁定这个对象的线程ID。加锁时先判断偏向锁，也就是把当前线程ID和锁上记录的最后访问线程ID做比较，如果一样说明只有一个线程在使用，不用加锁。偏向锁在只有一个线程访问synchronized同步块代码时提高了效率。如果线程ID不一样，使用CAS操作（更新LockRecord指针）来抢锁，抢锁成功即加上了轻量级锁，抢锁的操作是自旋的，多次失败进入重量级锁逻辑。

### volatile

volatile关键字比synchronized轻量级，可以保证可见性和有序性，但不能保证复合操作的原子性。synchronized相当于限制同一段代码同一时刻只有一个线程执行，把并行改成了串行，所以可以保证没有竞态条件。volatile并不互斥，多线程可以同时直接。volatile修饰的变量保证在写入时立即更新到主内存中，在读取时从主内存中读取，所以volatile可以保证在多个线程中这个变量的值是一样的。

由于volatile关键字不能保证复合操作的原子性，所以volatile最适合修饰boolean变量。

```java
public static volatile boolean ready = false;
```

下面的count++就是复合操作的例子，虽然volatile可以保证count值立即刷新到主内存中，但是由于++操作不是原子操作，所以count++的原子性无法保证。

```java
public static volatile int count = 0;
public void add() {
    count++;
}
```

虽然在实际工作中volatile的使用远比synchronized少，但理解volatile原理是非常重要的。

### wait/notify

通过修改共享变量的值可以完成线程间通信，但这样读取线程需要一直尝试读取，无法让出CPU，所以我们要引入wait()和notify()方法。

调用wait()方法将使当前线程进入等待状态，直到有其他线程调用了notify()或者notifyAll()方法后唤醒。这里有几点必须注意：

- 虽然实现了线程阻塞和唤醒，但wait()和notify()不是Thread类的方法，是Object类的方法；
- wait()和notify()必须在synchronized代码块内使用，也就是说你的先得到对象的锁，然后才可以阻塞或唤醒它；
- 调用一个对象的notify()方法后，将唤醒一个在这个对象上wait()的线程，如果有多个线程等待，操作系统决定唤醒哪一个；notifyAll()方法唤醒所有在这个对象上wait()的线程；
- 由于存在虚假唤醒和信号丢失等可能，所以实战中建议使用notifyAll()，比较安全。

### ThreadLocal

ThreadLocal可以认为是一种特殊的线程同步操作，为避免共享变量出问题，在每个线程中都创建一个不同的副本，互不影响，实际上就是不共享了。

> 代码看上去是共享的，但实际上给每个线程创建了一个对象，最后以线程为key放到一个map中使用。

在Thread里面有一个静态成员变量`ThreadLocal.ThreadLocalMap threadLocals = null;` threadLocals是一个所有线程共享的map，这个map的key是Thread.currentThread()，value是ThreadLocal的值。

```java
public class ThreadLocal<T> {
  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else
      createMap(t, value);
  }  
  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }  
  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      ThreadLocalMap.Entry e = map.getEntry(this);
      if (e != null) {
        T result = (T)e.value;
        return result;
      }
    }
    return setInitialValue();
  }  
}
```

重写initialValue()方法可以初始化ThreadLocal的值。

```java
ThreadLocal<String> uuid = new ThreadLocal<String>(){
  @Override
  protected String initialValue() {
    return UUID.randomUUID().toString();
  }
};
```

### final

final关键字也是避免多线程共享出现竞态条件的一种方法，把共享变量声明为final，不变当然不会出问题。

## Java并发包

Java从1.5开始提供了并行开发包java.util.concurrent，也有人简称JUC。ThreadPoolExecutor就是并发包中的类，下面来看看其他重要的类。

### AtomicInteger

保证读写是原子操作的Integer。

如果只是为了避免int共享变量的竞态条件，可以使用AtomicInteger。因为synchroznied虽然可以实现，但是太重量级，volatile又不能保证++操作原子性。AtomicInteger类的核心方法是compareAndSet()，也就是CAS。

### CAS

由AtomicInteger引出了CAS，compare and swap，比较并交换。CAS乐观锁机制是Java并行包中很多实现的基础。CAS对应Unsafe类中的compareAndSwap方法，这个方法有三个参数：(1)内存地址、(2)当前值、(3)新值；执行compareAndSwap方法时会判断当前内存地址，如果等于当前值，那么替换为新值并返回成功；如果不等于当期值，那么不做任务操作返回失败。

以AtomicInteger类的getAndIncrement()方法为例，实现原子++，并且返回原值。

```java
public class AtomicInteger extends Number implements java.io.Serializable {    
  public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
  }
}
```

原子++操作实际上是调用Unsafe类的getAndAddInt()方法实现的：这个方法一直尝试compareAndSwapInt()。

```java
public final class Unsafe {
  public final native boolean compareAndSwapInt(Object obj, long offset, int expect, 
                                                int update);
  public final int getAndAddInt(Object obj, long offset, int update) {
    int expect;
    do {
      expect = this.getIntVolatile(obj, offset);
    } while(!this.compareAndSwapInt(obj, offset, expect, expect + update));
    return var5;
  }  
}
```

> Unsafe的CAS操作实际上对应了硬件的CAS型指令，这个指令具有原子性。

#### 自旋

上面代码中`do {} while(compareAndSwap())` 这种操作被称为CAS自旋，自旋这个名词在多线程中经常用到，个人理解就是`while(true)` 一直尝试的意思，其实很简单，但起了个高大上的名字。

#### ABA问题

compare时虽然内存地址上的值是A，与compareAndSwap前读取到的值一样，但这不能说明内存值没有变化，可能发生了A->B->A的变化。增加版本号或者时间戳信息可以解决ABA问题。

个人认为，如果内存地址上存储的是int值，那么ABA也不会带来太大问题。但是，如果内存地址上存储的是链表指针，可能就会出大问题。例如：已知链表A->C，head为头指针指向A元素，需求compareAndSwap(A, C)将head移动到下一个元素C。如果在移动前链表发生了变化，变成了：A->B，那么compareAndSwap(A, C)判断head还是A，将head=C。实际上C元素已经在A->B时被删除了，所以这个时候head指到了链表外，就错了。

### ReentrantLock

ReentrantLock类实现了Lock接口，顾名思义实现了可重入锁。从一定意义上讲，可以将ReentrantLock类视作synchronzied的替代者。ReentrantLock提供了显示锁，synchronzied提供了隐式锁；ReentrantLock功能更强大，也更复杂，synchronzied简单好理解，并且也在逐步优化。

> 两者没有好坏之分，synchronzied简单，如果使用synchronzied可以满足要求，那么使用synchronzied就行；如果synchronzied不能满足要求，那么使用ReentrantLock。
>
> 不要想当然认为synchronzied效率低下，synchronzied做了轻量级和偏向锁优化，效率不低。

ReentrantLock常规用法如下，建议在finally中释放锁。

```java
Lock lock = new ReentrantLock();
try {
  lock.lock();
  ......
} finally {
  lock.unlock();
}
```

lock()方法是阻塞的，也可以使用不阻塞的tryLock()方法

```java
Lock lock = new ReentrantLock();
try {
  int count = 0;
  int maxCount = 5;
  while(!lock.tryLock(100, TimeUnit.MILLISECONDS) && count < maxCount) {
    count++;
  }
  ......
} finally {
  lock.unlock();
}
```

tryLock()也是解决死锁问题的一种办法，可以先tryLock()全部资源，都成功后才开始执行。

ReentrantLock提供了公平锁FairSync和非公平锁NonfairSync两种实现，默认使用非公平锁，可通过参数fair选择使用。ReentrantLock基于AbstractQueuedSynchronizer实现，AQS单独讲。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
  private final Sync sync;
  public ReentrantLock() {
    sync = new NonfairSync();
  }
  public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
  }  
  public void lock() {
    sync.lock();
  }
  public void unlock() {
    sync.release(1);
  }
  static final class FairSync extends Sync {}
  static final class NonfairSync extends Sync {}
  abstract static class Sync extends AbstractQueuedSynchronizer {}
}
```

ReentrantLock是可重入锁，每lock()一次，计数器加1，每unlock()一次，计数器减1。lock()和unlock()一定要成对出现。

#### Condition

ReentrantLock对应synchronized，ReentrantLock.Condition对应wait/notify，都可以达到相同效果。wait和notify必须在synchronized语句块内，所以Condition对象通过ReentrantLock对象创建。

```java
public class ReentrantConditionDemo {
  private static AtomicInteger sum = new AtomicInteger(0);
  private final static ReentrantLock lock = new ReentrantLock();
  private final static Condition condition = lock.newCondition();  
  public static class Consumer implements Runnable {
    @Override
    public void run() {
      try {
        lock.lock();
        condition.await();
        logger.info("sum=" + sum.get());
      } catch (InterruptedException e) {
        e.printStackTrace();
      } finally {
        lock.unlock();
      }
    }
  }
  public static class Producer implements Runnable {
    @Override
    public void run() {
      try {
        lock.lock();
        sum.getAndIncrement();
        Thread.sleep(1000);
        condition.signalAll();
      } catch (InterruptedException e) {
        e.printStackTrace();
      } finally {
        lock.unlock();
      }
    }
  }
}
```

几个注意的地方：

- 使用`lock.newCondition();` 创建Condition，一个lock可以创建多个不同的Condition；
- `Condition.await=Object.wait ` `Condition.signal=Object.notify` `Condition.signalAll=Object.notifyAll`
- 调用Condition方法前必须lock.lock()；
- wait和notify是Object类的方法，所以Condition类也有这两个方法，不要写错。

ReentrantLock.Condition相对wait/notify的增强点：

- wait/notify是基于Object的，也就是说一个Ojbect被lock以后，如果有多个等待条件，notifyAll都可以触发；例如生产者-消费者中锁定数组对象后，有数组空和数组满两个判断条件，都通过notifyAll触发；
- ReentrantLock.Condition的颗粒度比wait/notify小，一个Lock对象可以new多个Condition，也就是说数组空和数组满可以是两个Condition，分别signalAll触发。

### AQS

由ReentrantLock引出了AbstractQueuedSynchronizer同步器，简称AQS。不仅ReentrantLock，BlockingQueue、CountDownLatch和Semaphore等类都是基于AQS实现的，所以AQS是JUC中最重要的一个类。

先从字面理解，AQS是抽象队列同步器。首先这是一个抽象类，使用时需要像ReentrantLock一样继承抽象基类并实现相应方法；其次，AQS目的是同步，所以基于AQS可以实现各种同步功能；最后，AQS是基于队列实现的。

AQS有两种模式：独占模式和共享模式。独占的意思是只有一个线程能够得到锁，其他线程需要在队列里面排队，ReentrantLock就是独占模式。共享的意思是同时可以有多个线程得到锁，Semaphore就是共享模式。

AQS里面有一个volatile修饰的int型变量state，记录了当前锁的状态。以独占模式为例，通过CAS操作更新state状态保证并发时只能有一个线程得到锁，其他线程放到一个queue里面排队。这个queue是通过双向链表实现的，每个节点是一个Node，Node里面记录了排队Thread信息，前一个Node的指针和后一个Node的指针。AQS里面记录了双向链表的头指针head和尾指针tail。

```java
public abstract class AbstractQueuedSynchronizer {
  static final class Node {
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    Node nextWaiter;
  }
  private transient volatile Node head;
  private transient volatile Node tail;
  private volatile int state;
  protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }
  public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
  }
  public final boolean release(int arg) {
    if (tryRelease(arg)) {
      Node h = head;
      if (h != null && h.waitStatus != 0)
        unparkSuccessor(h);
      return true;
    }
    return false;
  }  
  public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
      doAcquireShared(arg);
  }  
  public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
      doReleaseShared();
      return true;
    }
    return false;
  }  
  protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
  }  
  protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
  }
  protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
  }
  protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
  }
} 
```

注意：这里有一个小技巧。AbstractQueuedSynchronizer是一个抽象基类，独占模式的接口是acquire和release，共享模式的接口是acquireShared和releaseShared。从代码可以看出，独占模式必须实现tryAcquire和tryRelease方法，由于AQS有两种模式，所以tryAcquire和tryRelease方法没有被声明为abstract，而是采用了抛出异常的方法。也就是说，共享模式不会进入tryAcquire和tryRelease代码，没有问题；独占模式如果子类没有实现这两个方法，将抛出异常。

compareAndSetState()方法通过CAS操作更新state状态，通过state状态就可以判断锁定状态。

head是队列的头指针，head声明为volatile是由于指针的移动也是CAS操作。

> AQS是Java并发包的基础，CAS又是AQS的基础。

#### 独占模式

独占模式加锁的处理流程如下：

- 入口acquire()方法；
- 调用tryAcquire()方法（子类实现）尝试加锁，如果加锁成功直接返回成功；
- 加锁失败，将线程信息封装到Node节点中，并添加到队列的队尾（CAS+自旋入队尾）；
- 再次尝试tryAcquire()获取锁，失败后线程进入阻塞；

独占模式解锁的处理流程如下：

- 入口release()方法；
- 调用 tryRelease()方法尝试释放锁；
- 解锁成功唤醒后继线程；

#### 共享模式

> TODO

#### LockSupport

AQS的线程阻塞和唤醒没有使用Object类的wait()和notify()方法，使用的是LockSupport类的park()和unpark()方法。底层通过Unsafe类native的park()和unpark()方法实现。

wait()和notify()是基于Object的，park()和unpark()是基于Thread的，语义上更好理解。

### BlockingQueue

BlockingQueue是一个FIFO的阻塞队列，put()方法往队列末尾增加元素，take()方法从队列头读取元素。BlockingQueue有三种实现：ArrayBlockingQueue、LinkedBlockingQueue和SynchronousQueue。

#### ArrayBlockingQueue

基于数组实现有界队列，底层基于ReentrantLock实现，notEmpty判断空，notFull判断满。

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E> {
  final Object[] items;
  int count;
  public ArrayBlockingQueue(int capacity, boolean fair) {
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
  }  
  public void put(E e) throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
      while (count == items.length)
        notFull.await();
      enqueue(e);
    } finally {
      lock.unlock();
    }
  }
  public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
      while (count == 0)
        notEmpty.await();
      return dequeue();
    } finally {
      lock.unlock();
    }
  }  
}
```

#### LinkedBlockingQueue

基于链表实现无界队列，读和写使用了两个锁，count使用AtomicInteger。

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E> {
  private final int capacity;
  private final AtomicInteger count = new AtomicInteger();
  private final ReentrantLock takeLock = new ReentrantLock();
  private final Condition notEmpty = takeLock.newCondition();
  private final ReentrantLock putLock = new ReentrantLock();
  private final Condition notFull = putLock.newCondition();  
  public LinkedBlockingQueue() {
    this.capacity = Integer.MAX_VALUE;
    last = head = new Node<E>(null);
  }
  static class Node<E> {}
  transient Node<E> head;
  private transient Node<E> last;
  public void put(E e) throws InterruptedException {
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
      while (count.get() == capacity) {
        notFull.await();
      }
      enqueue(node);
      c = count.getAndIncrement();
      if (c + 1 < capacity)
        notFull.signal();
    } finally {
      putLock.unlock();
    }
    if (c == 0)
      signalNotEmpty();
  }
  public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
      while (count.get() == 0) {
        notEmpty.await();
      }
      x = dequeue();
      c = count.getAndDecrement();
      if (c > 1)
        notEmpty.signal();
    } finally {
      takeLock.unlock();
    }
    if (c == capacity)
      signalNotFull();
    return x;
  }  
}
```

#### SynchronousQueue

零界队列。

> TODO

### CountDownLatch

倒计数锁。每次调用`countDown()` 计数器减1，`await();` 方法阻塞到计数器为0时返回，起到和join()方法一样的效果。CountDownLatch比Thread.join()更灵活，Thread.join()只能在线程结束后返回，CountDownLatch可以在任何位置调用countDown()方法。另外，我们通常使用线程池启动子线程，子线程只需要实现runnable接口，这种情况下Thread类对象是封装在线程池里面的，我们不方便拿到，也就不方便调用它的join()方法；使用CountDownLatch就简单多了，只需要在run()方法退出前调用countDown()方法即可。

```java
public static class Task implements Runnable {
  private CountDownLatch counter;
  public Task(CountDownLatch counter) {
    this.counter = counter;
  }
  @Override
  public void run() {
    counter.countDown();
  }
}

CountDownLatch counter = new CountDownLatch(2);
Task task = new Task(counter);
ExecutorService executorService = Executors.newFixedThreadPool(2);
for (int i=0; i<2; i++) {
  executorService.execute(task);
}
counter.await();
```

### Semaphore

信号量和锁有点类似，区别是ReetrantLock是独占锁，只能锁一次，Semaphore是共享锁，可以锁多次。

Semaphore的acquire()/release()方法和lock()/unlock()方法一样需要成对使用。区别在于lock()方法只能有1个线程通过，acquire()方法可以有多个线程通过。

```java
Semaphore semaphore = new Semaphore(5);

public void run() {
  try {
    semaphore.acquire();
	......
    semaphore.release();
  } catch (InterruptedException e) {
    e.printStackTrace();
  }
}
```

### CyclicBarrier

循环栅栏，所有等待线程等到最后一个执行完成后再同步前进，可以指定下一步任务。不常用。

```java
CyclicBarrier barrier = new CyclicBarrier(2, new MainTask());

public void run() {
    barrier.await();
}
```

## 多线程活性问题

### 死锁

两个线程互相等待对方的资源而被阻塞。线程1锁定资源A，申请资源B，线程2锁定资源B，申请资源A，就出现死锁。典型的例子：哲学家进餐问题、父节点和子节点互相添加。

哲学家进餐问题：五个哲学家做在一个圆桌上，只有五支筷子，哲学家需要左右手各拿到一支才可以进餐。哲学家全都先拿起左手筷子，再拿起右手筷子，然后再进餐。可能出现五个哲学家全部左手拿筷子的情况，形成死锁。

解决方法一：锁定全部筷子

一次拿走全部筷子，然后再进餐，这样不会死锁，但同时只能有一个人进餐，虽然还富余三支筷子。

解决方法二：严格锁定顺序

都按照一个顺序来加锁。这点可能感到奇怪，先加左手筷子锁，再加右手筷子锁，顺序是一样的啊。主要原因是圆桌，左手和右手是相对的，A的右手可能就是B的左手。解决方案：给所有筷子从1到5编号，先加小号锁，再加大号锁。编号是绝对的，保证从小到大的加锁顺序可以避免死锁。

解决方法三：尝试加锁

使用ReentrantLock类的tryLock()方法尝试给两支筷子加锁，都成功才可以进餐。

### 活锁

一直尝试加锁，一直失败得不到锁，就是活锁现象。

### 线程饥饿

多个线程竞争同一个锁，如果一直有新的线程加入竞争，可能出现某个线程永远得不到锁的情况。

使用ReentrantLock的公平锁可以解决线程饥饿问题，公平锁保证按照排队顺序唤醒下一个线程。

```java
Lock lock = new ReentrantLock(true);
```

注意：公平锁带来了额外的成本，所以通常实现统计意义上的公平即可，无需实现排队公平。

### 虚假唤醒

调用wait()方法后处于blocked状态的线程，在没有其他线程调用notify()方法的情况下被系统唤醒，称为虚假唤醒。

为避免虚假唤醒带来的问题，进入wait()的判断条件不用if，用while，这样唤醒后会再判断一遍。

```java
while(condition) {
    object.wait();
}
```

### 信号丢失

调用wait()方法后等待notify()方法唤醒，但是不幸的是notify()方法已经在wait()方法执行前执行了，那么wait()永远等不到唤醒信息。

## 生产者/消费者

### wait/notify实现

```java
public class ProducerConsumerDemo {
  private final static Logger LOGGER = LoggerFactory.getLogger(ProducerConsumerDemo.class);
  public static void main(String[] args) {
    LinkedList<String> storeList = new LinkedList<>();
    Producer producer = new Producer(storeList, "producer" + (i+1));
    Consumer consumer = new Consumer(storeList, "consumer" + (i+1));
    producer.start();
    consumer.start();
  }
  // 生产者
  public static class Producer extends Thread {
    private LinkedList<String> storeList;
    private final static int MAX = 10;
    private static AtomicInteger count = new AtomicInteger(1);    
    public Producer(LinkedList<String> storeList, String name) {
      this.storeList = storeList;
      this.setName(name);
    }
    @Override
    public void run() {
      while(true) {
        synchronized (storeList){
          // 库存满了阻塞等待消费
          while(storeList.size() == MAX) {
            LOGGER.info("库存满了");
            try {
              storeList.wait();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          // 生产20个以后退出
          if (count.intValue() > 20) {
            break;
          }
          // 生产数据
          String data = String.format("%04d", count.getAndIncrement());
          storeList.add(data);
          LOGGER.info("生产 " + data);
          // 等一会
          try {
            Thread.sleep(500);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          // 唤醒等待消费的线程
          storeList.notifyAll();
        }
      }
    }
  }
  // 消费者
  public static class Consumer extends Thread {
    private LinkedList<String> storeList;
    public Consumer(LinkedList<String> storeList, String name) {
      this.storeList = storeList;
      this.setName(name);
    }
    @Override
    public void run() {
      while(true) {
        synchronized (storeList){
          // 库存空了阻塞等待生产
          while(storeList.size() == 0) {
            LOGGER.info("库存空了");
            try {
              storeList.wait();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          // 消费掉第一个数据
          String data = storeList.get(0);
          LOGGER.info("消费 " + data);
          storeList.remove(0);
          // 等一会
          try {
            Thread.sleep(500);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          // 唤醒等待生产的线程
          storeList.notifyAll();
        }
      }
    }
  }
}
```

### Lock/Condition实现

```java
public class ProducerConsumerDemo {
  private static int count = 0;
  private final static Logger logger = LoggerFactory.getLogger(ProducerConsumerDemo.class);
  public static void main(String[] args) {
    LinkedList<Integer> store = new LinkedList();
    Lock lock = new ReentrantLock();
    Condition full = lock.newCondition();
    Condition empty = lock.newCondition();
    ExecutorService executorService = Executors.newCachedThreadPool();
    Producer producer = new Producer(store, lock, full,empty);
    Consumer consumer = new Consumer(store, lock, full,empty);
    executorService.submit(producer);
    executorService.submit(consumer);
  }
  // 生产者
  public static class Producer implements Runnable {
    private final LinkedList<Integer> store;
    private final Lock lock;
    private final Condition full;
    private final Condition empty;
    public Producer(LinkedList store, Lock lock, Condition full, Condition empty) {
      this.store = store;
      this.lock = lock;
      this.full = full;
      this.empty = empty;
    }
    @Override
    public void run() {
      while (count < 20){
        try {
          lock.lock();
          while (store.size() >= 10) {
            full.await();
          }
          count++;
          Integer data = new Integer(count);
          store.add(data);
          logger.info("生产 " + data);
          Thread.sleep(500);
          empty.signalAll();
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
        }
      }
    }
  }
  // 消费者
  public static class Consumer implements Runnable {
    private final LinkedList<Integer> store;
    private final Lock lock;
    private final Condition full;
    private final Condition empty;
    public Consumer(LinkedList store, Lock lock, Condition full, Condition empty) {
      this.store = store;
      this.lock = lock;
      this.full = full;
      this.empty = empty;
    }
    @Override
    public void run() {
      while (true) {
        try {
          lock.lock();
          while (store.size() == 0) {
            empty.await();
          }
          Integer data = store.get(0);
          store.remove(0);
          logger.info("消费 " + data);
          Thread.sleep(500);
          full.signalAll();
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
        }
      }
    }
  }
}
```

### BlockingQueue实现

```java
public class ProducerConsumerDemo {
  private final static Logger logger = LoggerFatory.getLogger(ProducerConsumerDemo.class);
  public static void main(String[] args) throws InterruptedException {
    BlockingQueue queue = new ArrayBlockingQueue(10);
    Producer producer = new Producer(queue);
    Consumer consumer = new Consumer(queue);
    new Thread(producer).start();
    new Thread(consumer).start();
  }
  // 生产者
  public static class Producer implements Runnable{
    protected BlockingQueue queue = null;
    protected int count = 0;
    public Producer(BlockingQueue queue) {
      this.queue = queue;
    }
    @Override
    public void run() {
      while (count < 20) {
        try {
          count++;
          queue.put(new Integer(count));
          logger.info("生产 " + count);
          Thread.sleep(100);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }
  }
  // 消费者
  public static class Consumer implements Runnable{
    protected BlockingQueue queue = null;
    public Consumer(BlockingQueue queue) {
      this.queue = queue;
    }
    @Override
    public void run() {
      while (true) {
        try {
          logger.info("消费 " + queue.take());
          Thread.sleep(500);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }
  }
}
```

## Java内存模型



## 参考资料

[Java并发编程：Synchronized底层优化（偏向锁、轻量级锁）](http://www.cnblogs.com/paddix/p/5405678.html)

[AbstractQueuedSynchronizer 原理分析 - 独占/共享模式](http://www.coolblog.xyz/2018/05/01/AbstractQueuedSynchronizer-%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90-%E7%8B%AC%E5%8D%A0-%E5%85%B1%E4%BA%AB%E6%A8%A1%E5%BC%8F/)

[【Java并发编程】—–“J.U.C”：LinkedBlockingQueue](https://www.jianshu.com/p/cc2281b1a6bc)

