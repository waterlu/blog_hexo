---
title: thread-status
date: 2018-04-28 11:30:13
tags:
---

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

> waiting状态和blocked状态的区别在于一个是主动交出时间片，另外一个是被动交出时间片；当然前提是已经获得了时间片，得到了执行；从runnable状态到running状态是不受程序控制的，完全靠操作系统来调度，虽然我们可以设置线程的优先级，但是不一定达到预期。
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

Thread类和Object类相关方法

```java
package java.lang;
public class Thread implements Runnable {
	public static native Thread currentThread();
  	public static native void yield();
  	public static native void sleep(long millis) throws InterruptedException;  
  	public synchronized void start() {
    	start0();  
    }
  	public void interrupt() {
      	interrupt0();
    }
  	public final void setPriority(int newPriority) {
      	setPriority0(priority = newPriority);
    }
  	private native void start0();
  	private native void interrupt0();
  	private native void setPriority0(int newPriority);
}
```

```java
package java.lang;
public class Object {
	public final native void notify();
  	public final native void notifyAll();
  	public final native void wait(long timeout) throws InterruptedException;
}
```