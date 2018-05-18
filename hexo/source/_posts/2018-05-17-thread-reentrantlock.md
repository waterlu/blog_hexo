---
title: Reentrantlock源码分析
date: 2018-05-17 16:33:00
categories: 并发编程
tags: [Reentrantlock]
toc: true
description: Reentrantlock类源码和实现原理分析。
comments: false

---

## 源码解析

ReentrantLock顾名思义表示可重入锁，一般视为synchronized的替代者。

废话不说，直接上代码，从代码中可以看出：

- ReentrantLock有两种实现：公平锁FairSync和非公平锁NonfairSync。
- 默认实现是非公平锁NonfairSync；
- 构造ReentrantLock对象时可以传入boolean型参数fair显示指定使用公平锁还是非公平锁；
- 无论公平锁还是非公平锁都集成AbstractQueuedSynchronizer类；
- 结论：关键看AbstractQueuedSynchronizer、FairSync和NonfairSync三个类如何实现。

```java
package java.util.concurrent.locks;
public class ReentrantLock implements Lock, java.io.Serializable {
  public ReentrantLock() {
    sync = new NonfairSync();
  }
  public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
  }  	
  private final Sync sync;
  public void lock() {
    sync.lock();
  }
  public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
  }
  public void unlock() {
    sync.release(1);
  }  
  abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();
  }
  static final class FairSync extends Sync {}
  static final class NonfairSync extends Sync {}
}
```

### NonfairSync

ReentrantLock类的默认实现是NonfairSync，我们先看非公平锁。NonfairSync是ReentrantLock的静态内部类。

我们发现NonfairSync类非常简单，以下已经是这个类的全部代码，核心实现不再这里，继续向下深入。

```java
static final class NonfairSync extends Sync {
  final void lock() {
    if (compareAndSetState(0, 1))
      setExclusiveOwnerThread(Thread.currentThread());
    else
      acquire(1);
  }
  protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
  } 
}
```

看了AbstractQueuedSynchronizer代码以后，我们解释一下lock()方法

- compareAndSetState()是Unsafe的一个CAS方法，AQS初始化的state=0，lock()将state=1
```java
package java.util.concurrent.locks;
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
  private volatile int state;
  private static final Unsafe unsafe = Unsafe.getUnsafe();
  private static final long stateOffset;
  static {
    stateOffset = unsafe.objectFieldOffset
      (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
  }
  protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }  
}
```
- 第一步相当于抢占了一个开关，抢到以后把当前线程对象赋值给了AQS对象作为当前排他线程，仅此而已；

```java
package java.util.concurrent.locks;
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
  private transient Thread exclusiveOwnerThread;
  protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
  }
  protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
  }
}
```

- 第二次执行lock()方法时，compareAndSetState()由于stateOffset=1和预期值0不一样，返回false，进入acquire(1)逻辑；acquire()是AbstractQueuedSynchronizer类的方法，tryAcquire()回到NonfairSync类执行nonfairTryAcquire(1)

```java
package java.util.concurrent.locks;
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
  public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
  }
}
```

- nonfairTryAcquire()是基类Sync的方法，从基类AbstractQueuedSynchronizer中getState()；由于之前lock()时已经将AQS置为1，这里c==1且current !=  exclusiveOwnerThread，返回false；

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
  final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
}
```

```java
package java.util.concurrent.locks;
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
  private volatile int state;
  protected final int getState() {
    return state;
  }
  protected final void setState(int newState) {
    state = newState;
  }  
}
```

- tryAcquire()返回false，执行acquireQueued(addWaiter(Node.EXCLUSIVE), 1)

```java
package java.util.concurrent.locks;
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {    
  private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
      node.prev = pred;
      if (compareAndSetTail(pred, node)) {
        pred.next = node;
        return node;
      }
    }
    enq(node);
    return node;
  }
}
```





### AQS

AbstractQueuedSynchronizer类就是与CAS齐名、大名鼎鼎的AQS。

先看AbstractQueuedSynchronizer基类AbstractOwnableSynchronizer，基类很简单，只记录一个线程对象，从变量名看这个线程对象是排他的。

> 这里的transient是不参加序列化的意思。

```java
package java.util.concurrent.locks;
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
  private transient Thread exclusiveOwnerThread;
  protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
  }
  protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
  }
}
```

下面是AbstractQueuedSynchronizer代码

```java
package java.util.concurrent.locks;
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
  private static final Unsafe unsafe = Unsafe.getUnsafe();
  private static final long stateOffset;
  static {
    stateOffset = unsafe.objectFieldOffset
      (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
  }
  private volatile int state;
  protected final int getState() {
    return state;
  }
  protected final void setState(int newState) {
    state = newState;
  }  
  protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }  
  public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
  }
}
```





## 原理分析

lock时，调用AQS的compareAndSetState(0,1)方法将AQS的state=1，

## 参考

[ReentrantLock实现原理深入探究](http://www.cnblogs.com/xrq730/p/4979021.html)

[Java 重入锁 ReentrantLock 原理分析](http://www.cnblogs.com/nullllun/p/9004309.html)

[AbstractQueuedSynchronizer 原理分析 - 独占/共享模式](http://www.coolblog.xyz/2018/05/01/AbstractQueuedSynchronizer-%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90-%E7%8B%AC%E5%8D%A0-%E5%85%B1%E4%BA%AB%E6%A8%A1%E5%BC%8F/)



