---
title: 并行包源码分析
date: 2018-05-17 16:33:00
categories: 并发编程
tags: [Reentrantlock]
toc: true
description: Reentrantlock类源码和实现原理分析。
comments: true

---



## 基础理论

### CAS

CAS是英文Compare And Swap的简称，从字母也能够理解其含义：比较并替换。回想一下数据库中经常使用的乐观锁就是CAS的思想。CAS同时也是Java并行包的基础，可以简单认为syschronized是悲观锁，基于CAS的Java并行包是乐观锁。

CAS对应Unsafe类中的compareAndSwap方法，这个方法实际上需要三个参数：(1)数据的内存地址、(2)预期当前值和(3)即将更新值；执行compareAndSwap方法时会判断当前内存值，如果等于预期当前值，那么替换为新值并返回成功；如果不等于预期当期值，那么不做更新操作，返回失败。

>  这里的comapre和swap是一组原子操作，不会被打断。

我们知道，syschronized可以实现原子性，为避免多个线程同时修改一个数据引发问题，我们在每个线程修改数据前加锁，保证同时只有一个线程操作数据，这样肯定是线程安全的，但是效率低。因为，大多数情况下不存在多个线程同时修改数据的情况，所以大多数情况下加锁和解锁操作是废操作。CAS的思路默认没有线程同时执行，如果出现线程并发修改情况，只有一个线程可以compareAndSwap成功，其他线程出错后重试。

> 凡事没有绝对，如果真的存在多个线程频繁修改同一个数据的业务场景，那么syschronized效率是高于CAS的，因为一旦CAS需要频繁重试，效率自然下降。

#### Unsafe类

CAS的核心实现是Unsafe类，Unsafe类功能强大，使Java拥有了像C一样直接访问内存空间的能力，所以是非常危险的，慎用。Unsafe类的核心方法都是native的，下面看看代码。

```java
package sun.misc;
public final class Unsafe {
  private static final Unsafe theUnsafe;
  public static Unsafe getUnsafe() {
    return theUnsafe;
  }
  static {
    theUnsafe = new Unsafe();
  }
  private Unsafe() {}
  public final native boolean compareAndSwapObject(Object obj, long offset, 
                                                   Object expect, Object update);
  public final native boolean compareAndSwapInt(Object obj, long offset, 
                                                int expect, int update);
  public final native boolean compareAndSwapLong(Object obj, long offset, 
                                                 long expect, long update); 
  public final int getAndAddInt(Object obj, long offset, int update) {
    int expect;
    do {
      expect = this.getIntVolatile(obj, offset);
    } while(!this.compareAndSwapInt(obj, offset, expect, expect + update));
    return var5;
  }
  public native int getIntVolatile(Object var1, long var2);
}
```

我们以`compareAndSwapInt` 为例，这个方法有四个参数：

- Object obj，类实例对象；
- long offset，变量相当于类实例对象的偏移地址（obj+offset定位int变量的内存地址）；
- int expect，预期当前int值；
- int update，即将更新int值。

> 如果有一点C语言知识，就很好理解这种定位内存地址的方法了。

我们再来看一下`getAndAddInt` 方法，也很好理解

- `getIntVolatile(obj, offset)` 方法可以理解为读取int变量的值（obj+offset定位int变量的内存地址）；
- 首先，读取内存中变量的当前值放入expect中；
- 下一步，执行compareAndSwapInt()，比较并交换，预期当前值为expect，新值为expect+update；
- 如果compareAndSwapInt()成功直接返回；
- 如果compareAndSwapInt()失败，重新尝试，直至成功为止。

再来看一下Unsafe的源码

```c
sun::misc::Unsafe::compareAndSwapInt (jobject obj, jlong offset, jint expect, jint update)
{
  jint *addr = (jint *)((char *)obj + offset);
  return compareAndSwap(addr, expect, update);
}

static inline bool compareAndSwap (volatile jint *addr, jint old, jint new_val)
{
  jboolean result = false;
  spinlock lock;
  if ((result = (*addr == old)))
    *addr = new_val;
  return result;
}
```

#### ABA问题

CAS存在一个很明显的问题，即ABA问题。也就是说，compare时虽然内存地址上的值是A，与compareAndSwap前读取到的值一样，但这不能说明内存值没有变化，可能发生了A->B->A的变化。

ABA问题本身不难理解，但是这样就一定出问题吗？对于数值来说，一般也没啥问题。但是，对于链表来说就有问题了。例如：

- 线程1读取当前链表结构为A->B，head指向A，线程1试图执行compareAndSwap(A, B)，将链表头指针head指向A的下一个元素B；
- 当线程1读取链表完成执行compareAndSwap前被挂起，线程2开始执行；
- 线程2从链表中删除了元素A和元素B，又加入了元素A和元素C，链表结构为A->C；
- 此时线程1开始执行，头指针head还是执行C，但链表内容已经变了；如果还是执行head=B就出错了。

ABA问题一般通过加入版本号或者时间戳就可以解决。

### AQS



## 源码解析

### AtomicInteger

明白了CAS原理，AtomicInteger类就很容易理解了。

```java
package java.util.concurrent.atomic;
public class AtomicInteger extends Number implements java.io.Serializable {
  private static final Unsafe unsafe = Unsafe.getUnsafe();
  private static final long valueOffset;
  static {
    valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
  }
  private volatile int value;
  public AtomicInteger(int initialValue) {
    value = initialValue;
  }
  public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
  }
}
```

我们以getAndIncrement()方法为例，目标为实现自增加1操作：

- 在AtomicInteger类构造时获取了Unsafe单例，并计算value变量的内存地址偏移量valueOffset；
- 当调用自增+1方法时，实际调用了Unsafe的getAndAddInt()方法，前两个参数定位到类实例对象的value变量内存地址，变化值为1；
- getAndAddInt()方法在CAS中介绍过，一直尝试compareAndSwapInt()，直到成功为止；
  - 读取value变量的当前值；
  - 执行比较和交换操作，预期值为刚刚读到的value值，更新值为value+1；
  - 如果没有其他线程竞争，那么成功写入value+1；
  - 如果已经有其他线程修改了value的内存值，那么比较和交换操作失败；重新读取value变量值重试比较和交换；
  - 理论上，如果一直有其他线程在修改，会陷入死循环出不来？

AtomicInteger类存在ABA问题，并行包提供了带时间戳的AtomicStampedReference类解决ABA问题。

### AbstractQueuedSynchronizer



### ReentrantLock

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

AQS

Java并行包的锁机制基于AQS框架实现



## 参考

[ReentrantLock实现原理深入探究](http://www.cnblogs.com/xrq730/p/4979021.html)

[Java 重入锁 ReentrantLock 原理分析](http://www.cnblogs.com/nullllun/p/9004309.html)

[AbstractQueuedSynchronizer 原理分析 - 独占/共享模式](http://www.coolblog.xyz/2018/05/01/AbstractQueuedSynchronizer-%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90-%E7%8B%AC%E5%8D%A0-%E5%85%B1%E4%BA%AB%E6%A8%A1%E5%BC%8F/)

https://segmentfault.com/a/1190000008471362

