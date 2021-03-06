---
title: 多线程基础篇(3) ThreadPoolExecutor
date: 2018-04-24 16:37:24
updated: 2018-04-25
categories: 并发编程
tags: [并发, 线程, 线程池, 排队策略, 饱和策略]
toc: true
description: 理解了ThreadPoolExecutor类的各个参数和处理流程也就理解了线程池机制。核心线程数、最大线程数、排队策略和饱和策略是最重要的参数，其中排队策略决定了处理流程。通过测试代码可以很好的验证各种参数设置条件下线程池的处理策略。
comments: true
---

## 原理

理解了ThreadPoolExecutor类的各个参数和内部原理也就理解了线程池机制。

### 构造参数

```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
		TimeUnit unit, BlockingQueue<Runnable> workQueue, 
		ThreadFactory threadFactory, RejectedExecutionHandler handler) {}
```

先来看ThreadPoolExecutor类的参数

| 参数名称            | 含义    | 详解                                |
| --------------- | ----- | --------------------------------- |
| corePoolSize    | 核心线程数 | 当前线程数小于corePoolSize时，会一直创建新线程     |
| maximumPoolSize | 最大线程数 | 最多可以创建maximumPoolSize个线程，超过进入饱和策略 |
| keepAliveTime   | 空闲时间  | 当超过corePoolSize时，回收线程时使用          |
| unit            | 时间单位  | 配合keepAliveTime一起使用               |
| workQueue       | 排队策略  | 最重要，排队策略决定了线程池的处理流程               |
| threadFactory   | 工厂    | 创建线程时进行自定义操作                      |
| handler         | 饱和策略  | 配合workQueue使用，处理线程池满的情况           |

### 处理流程

![图1-线程池原理](/images/threadpool_threadpoolexecutor.png)

#### 基本逻辑

- 当线程池里面的线程数小于corePoolSize时，不管当前线程池中的线程是否空闲，都创建新的线程来执行任务，并加入到线程池中；这样随着任务的增加，线程池的线程数会达到corePoolSize个；
- 达到corePoolSize后，当新任务到来时，会选择空闲的线程来执行；
- 如果没有空闲线程，进入排队策略，不同的排队策略有不同逻辑；
- 当核心线程空闲时，会从排队队列中取出任务来执行；
- 当核心线程没有空闲，并且排队队列满时，创建新线程执行任务，最大不超过maximumPoolSize个；
- 当达到maximumPoolSize个线程，且都在忙，新任务到来时进入饱和策略，不同的饱和策略有不同逻辑；如果没有配置饱和策略，抛出RejectedExecutionException异常；
- 当线程数超过corePoolSize后，启动回收逻辑，空闲时间超过keepAliveTime的线程将被回收；回收时不区分核心线程和非核心线程，减少到corePoolSize个后不再回收。

排队策略不同，处理流程也不同，下面分别介绍常见三种排队策略的处理流程：SynchronousQueue、ArrayBlockingQueue和LinkedBlockingQueue。

#### SynchronousQueue

简单说就是没有排队队列，或者队列长度为0，所以通常使用SynchronousQueue作为排队策略时，为避免出现线程执行被拒绝的情况，maximumPoolSize的值会被设置的很大。

```flow
s=>start: 开始
e=>end: 结束
condCore=>condition: 小于核心线程数?
condMax=>condition: 小于最大线程数?
addCore=>operation: 增加核心线程
addNormal=>operation: 增加非核心线程
full=>operation: 饱和逻辑处理

s->condCore
condCore(yes, right)->addCore
condCore(no)->condMax
condMax(yes, right)->addNormal
condMax(no)->full
addNormal->e
addCore->e
full->e
```

#### ArrayBlockingQueue

有界排队队列，必须设置队列长度；当线程数达到corePoolSize时，开始排队；当排队队列满时增加非核心线程直到maximumPoolSize。

```flow
s=>start: 开始
e=>end: 结束
condCore=>condition: 小于核心线程数?
condQueue=>condition: 队列未满?
condMax=>condition: 小于最大线程数?
addCore=>operation: 增加核心线程
addNormal=>operation: 增加非核心线程
addQueue=>operation: 排队等待执行
full=>operation: 饱和逻辑处理

s->condCore
condCore(yes, right)->addCore
condCore(no)->condQueue
condQueue(yes, right)->addQueue
condQueue(no)->condMax
condMax(yes, right)->addNormal
condMax(no)->full
addQueue->e
addNormal->e
addCore->e
full->e
```

#### LinkedBlockingQueue

LinkedBlockingQueue如果不指定队列大小，那么就是无界队列；除非系统资源耗尽，将无限增加队列长度，因此无界队列不存在队列满的情况，也就没有非核心线程、饱和逻辑和线程回收逻辑；也就是说当设置workQueue为LinkedBlockingQueue时，keepAliveTime、unit和handler三个参数失效。

LinkedBlockingQueue也可以设置队列大小，就成了有界队列，和ArrayBlockingQueue的处理逻辑一样。

```flow
s=>start: 开始
e=>end: 结束
condCore=>condition: 小于核心线程数?
addCore=>operation: 增加核心线程
addQueue=>operation: 排队等待执行

s->condCore
condCore(yes)->addCore
condCore(no)->addQueue
addQueue->e
addCore->e
```

## 代码示例

### 背景

每个任务开始时输出“run”，结束时输出“exit”，不做具体逻辑，只是sleep1秒钟。

```java
public static class Task implements Runnable {
    @Override
    public void run() {
        LOGGER.info("run");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        LOGGER.info("exit");
    }
}
```

主线程

- 核心线程数2个，最大线程数4个，保活时间3秒
- 依次启动6个任务，每启动一个任务后都输出线程池大小
- 启动任务错误时输出异常信息
- 6个任务都启动完成后sleep 5秒，退出前再次输出线程池大小

```java
public class ThreadExecutor {
    private final static Logger LOGGER = LoggerFactory.getLogger(ThreadExecutor.class);
    public static void main(String[] args) {
        int coreSize = 2;
        int maxSize = 4;
        long time = 3;
        TimeUnit unit = TimeUnit.SECONDS;
        ThreadPoolExecutor executor = new ThreadPoolExecutor(coreSize, maxSize, 
        	time, unit, queue, handler);
        for (int i=0; i<6; i++) {
            Task task = new Task();
            try {
                executor.submit(task);
            } catch (Exception e) {
                LOGGER.error(e.getClass().getSimpleName());
            }
            LOGGER.info("poolSize=" + executor.getPoolSize());
        }
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        LOGGER.info("poolSize=" + executor.getPoolSize());
        LOGGER.info("exit");
    }
}
```

下面看看不同的workQueue和handler配置下的输出结果

### 测试结果

#### 无界队列

```java
LinkedBlockingQueue<Runnable> queue = new LinkedBlockingQueue<>();
```

```shell
10:57:33:055 [pool-2-thread-1] run
10:57:33:055 [main] poolSize=1
10:57:33:055 [main] poolSize=2
10:57:33:055 [main] poolSize=2
10:57:33:055 [pool-2-thread-2] run
10:57:33:055 [main] poolSize=2
10:57:33:055 [main] poolSize=2
10:57:33:055 [main] poolSize=2
10:57:34:068 [pool-2-thread-2] exit
10:57:34:068 [pool-2-thread-1] exit
10:57:34:068 [pool-2-thread-1] run
10:57:34:068 [pool-2-thread-2] run
10:57:35:081 [pool-2-thread-1] exit
10:57:35:081 [pool-2-thread-2] exit
10:57:35:081 [pool-2-thread-1] run
10:57:35:081 [pool-2-thread-2] run
10:57:36:082 [pool-2-thread-1] exit
10:57:36:082 [pool-2-thread-2] exit
10:57:38:056 [main] poolSize=2
10:57:38:056 [main] exit
```

从日志中可以看出

- 当线程池达到核心线程数2后，一直保持在核心线程数不变
- pool-2-thread-1和pool-2-thread-2顺序从队列中取出任务依次执行

#### 有界队列

```java
ArrayBlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(2);
// LinkedBlockingQueue<Runnable> queue = new LinkedBlockingQueue<>(2);
```

```shell
11:01:23:147 [main] poolSize=1
11:01:23:147 [pool-2-thread-1] run
11:01:23:147 [main] poolSize=2
11:01:23:147 [main] poolSize=2
11:01:23:147 [pool-2-thread-2] run
11:01:23:147 [main] poolSize=2
11:01:23:147 [main] poolSize=3
11:01:23:147 [main] poolSize=4
11:01:23:147 [pool-2-thread-3] run
11:01:23:147 [pool-2-thread-4] run
11:01:24:147 [pool-2-thread-1] exit
11:01:24:147 [pool-2-thread-1] run
11:01:24:147 [pool-2-thread-2] exit
11:01:24:147 [pool-2-thread-2] run
11:01:24:147 [pool-2-thread-3] exit
11:01:24:147 [pool-2-thread-4] exit
11:01:25:158 [pool-2-thread-2] exit
11:01:25:158 [pool-2-thread-1] exit
11:01:28:154 [main] poolSize=2
11:01:28:154 [main] exit
```

从日志中可以看出

- 当队列（2个）满以后，增加了非核心线程pool-2-thread-3和pool-2-thread-4
- 核心线程pool-2-thread-1和pool-2-thread-2执行完第一个任务后，又从队列中取出第二个任务执行
- 主线程退出前线程池大小又回到了核心线程数，说明空闲线程已经被释放

#### 同步移交+Abort

```java
SynchronousQueue<Runnable> queue = new SynchronousQueue<>();
RejectedExecutionHandler handler = new ThreadPoolExecutor.AbortPolicy();
```

```sh
11:07:26:130 INFO [main] poolSize=1
11:07:26:130 INFO [pool-2-thread-1] run
11:07:26:132 INFO [main] poolSize=2
11:07:26:132 INFO [main] poolSize=3
11:07:26:132 INFO [pool-2-thread-2] run
11:07:26:133 INFO [main] poolSize=4
11:07:26:133 ERROR [main] RejectedExecutionException
11:07:26:133 INFO [main] poolSize=4
11:07:26:134 ERROR [main] RejectedExecutionException
11:07:26:134 INFO [main] poolSize=4
11:07:26:135 INFO [pool-2-thread-3] run
11:07:26:135 INFO [pool-2-thread-4] run
11:07:27:143 INFO [pool-2-thread-3] exit
11:07:27:143 INFO [pool-2-thread-2] exit
11:07:27:143 INFO [pool-2-thread-4] exit
11:07:27:143 INFO [pool-2-thread-1] exit
11:07:31:150 INFO [main] poolSize=2
11:07:31:150 INFO [main] exit
```

从日志中可以看出

- 当线程数到达最大线程数4个以后，再提交的任务抛出了RejectedExecutionException异常
- 所以与前面的结果不同，这次只执行了4个任务
- 最后空闲线程被回收，线程数保持在2个
- AbortPolicy是默认的饱和策略

> 以下已空队列为例，更换饱和策略
>
> 

#### 同步移交+Discard

```java
SynchronousQueue<Runnable> queue = new SynchronousQueue<>();
RejectedExecutionHandler handler = new ThreadPoolExecutor.DiscardPolicy();
```

```shell
11:12:20:113 INFO [pool-2-thread-1] run
11:12:20:113 INFO [main] poolSize=1
11:12:20:113 INFO [main] poolSize=2
11:12:20:113 INFO [pool-2-thread-2] run
11:12:20:113 INFO [main] poolSize=3
11:12:20:113 INFO [main] poolSize=4
11:12:20:113 INFO [pool-2-thread-3] run
11:12:20:113 INFO [main] poolSize=4
11:12:20:113 INFO [main] poolSize=4
11:12:20:113 INFO [pool-2-thread-4] run
11:12:21:128 INFO [pool-2-thread-4] exit
11:12:21:128 INFO [pool-2-thread-2] exit
11:12:21:128 INFO [pool-2-thread-3] exit
11:12:21:128 INFO [pool-2-thread-1] exit
11:12:25:119 INFO [main] poolSize=2
11:12:25:119 INFO [main] exit
```

对比上面的日志可以发现

- 同样也是只执行了4个任务
- 区别在于没有抛出异常，也就是说Discard策略直接拒绝，来异常都不给，没啥用

#### 同步移交+CallerRun

```java
SynchronousQueue<Runnable> queue = new SynchronousQueue<>();
RejectedExecutionHandler handler = new ThreadPoolExecutor.CallerRunsPolicy();
```

```shell
11:18:11:432 INFO [pool-2-thread-1] run
11:18:11:432 INFO [main] poolSize=1
11:18:11:434 INFO [main] poolSize=2
11:18:11:434 INFO [pool-2-thread-2] run
11:18:11:434 INFO [main] poolSize=3
11:18:11:435 INFO [main] poolSize=4
11:18:11:435 INFO [main] run
11:18:11:435 INFO [pool-2-thread-3] run
11:18:11:435 INFO [pool-2-thread-4] run
11:18:12:447 INFO [pool-2-thread-4] exit
11:18:12:447 INFO [main] exit
11:18:12:447 INFO [pool-2-thread-3] exit
11:18:12:447 INFO [pool-2-thread-2] exit
11:18:12:447 INFO [pool-2-thread-1] exit
11:18:12:447 INFO [main] poolSize=4
11:18:12:447 INFO [main] run
11:18:13:448 INFO [main] exit
11:18:13:448 INFO [main] poolSize=4
11:18:18:454 INFO [main] poolSize=2
11:18:18:454 INFO [main] exit
```

从日志中可以看出

- 当线程数到达最大线程数4个以后，第5个任务开始在main线程中执行
- CallerRunsPolic保证了线程不会被丢弃，但交给主线程运行没有起到线程池的作用，应该也不常用

## 实战

### Executors

```java
package java.util.concurrent;

public class Executors {
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }    
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    } 
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }  
}
```

Executors类提供了常用线程池的创建方法，不过不建议使用Executors来创建线程池，因为这会隐藏最终创建ThreadPoolExecutor类的参数，还是建议手工构造ThreadPoolExecutor，显示指定参数。下面详细看看三个预定义的线程池：

- SingleThread，顾名思义，线程池的核心和最大线程数都是1，无界排队策略；也就是说这个线程池中只有1个线程，其他并发线程排队等待，线程池一个一个来处理，简单，但是效率比较低；
- FixedThread，线程池的核心线程数与最大线程数保持一致，无界排队策略；也就是说这个线程池中没有非核心线程，最多可以同时处理N个任务，其他任务排队等待；SingleThread可以看作N=1的FixedThread；
- CachedThread，核心线程数0，最大线程数无穷大，无排队队列，保活时间60秒；也就是说这个线程池中的线程数是动态变化的，当有新任务时就创建新线程来处理（当前线程都在忙），处理完成后1分钟回收线程；如果没有任务，线程数就是0，如果任务非常多，线程数无限增加，直到耗尽系统资源。

> 以上可以看出，Executors默认提供的几种线程池比较极端，实战中需要自定义ThreadPoolExecutor。

### 核心线程数

虽然排队策略看上去比较复杂，但我认为ThreadPoolExecutor最重要的参数是核心线程数；因为考虑到不拒绝任何任务，我们可以使用LinkedBlockingQueue实现无界队列，这样其他参数基本就都失效了，而核心线程数即使当前没有任务也要保留，就变得异常重要的。

确定核心线程数，一般的思路是根据任务性质来判断的。如果是计算密集型任务，那么通常核心线程数设置为CPU个数+1，通过Runtime.getRuntime().availableProcessors();可以读取CPU个数。如果是I/O密集型任务，那么中断会多一些，核心线程数也可以多一些，一般可以设置为CPU个数的2倍。

> Apache MINA 的核心线程数设置为CPU个数+1