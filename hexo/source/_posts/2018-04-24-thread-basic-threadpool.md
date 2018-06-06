---
title: 多线程基础篇(2) 线程池
date: 2018-04-24 12:37:24
updated: 2018-04-25
categories: 并发编程
tags: [并发, 线程, 线程池, 任务调度]
toc: true
description: Executor和ExecutorService为我们提供了线程异步执行的接口。其中比较重要的是submit()和shutdown()，分别实现任务的提交和线程池的关闭。
comments: true
---

## 任务调度

任务是一组逻辑工作单元，线程则是任务异步执行的机制。前面说过，实战中我们不会直接调用Thread类的start()方法来启动线程，那么线程应该如何启动呢？

### Executor

使用Executor.execute(task);来替代new Thread(task).start();

```java
package java.util.concurrent;
public interface Executor {
    void execute(Runnable command);
}
```

Executor将任务提交给线程池来处理，线程池的处理是异步的，任务会交给新的线程来执行，本地线程可以继续做其他事情。

### ExecutorService

Executor只有一个没有返回值的接口execute()，为了更好的控制线程的行为，并行包中为我们提供了功能更强大的ExecutorService，实际使用过程中，更多使用的是ExecutorService的接口。

```java
package java.util.concurrent;

public interface ExecutorService extends Executor {
	<T> Future<T> submit(Callable<T> task);
	Future<?> submit(Runnable task);
	<T> Future<T> submit(Runnable task, T result);
	void shutdown();  
  	List<Runnable> shutdownNow();
}
```

下面详细看一下ExecutorService的几个接口

#### submit

提交任务有三个接口，区别在于任务类实现了哪个接口，以及是否需要读取返回结果

- Callable：下面的例子用来计算1+2+...+N的和，任务完成后返回计算结果

```java
    public class Task implements Callable<Integer> {
        private int count = 0;
        public Task(int count) {
            this.count = count;
        }

        @Override
        public Integer call() throws Exception {
            int sum = 0;
            for (int i=1; i<=count; i++) {
                sum += i;
            }
            LOGGER.info("done");
            return sum;
        }
    }
```

再来看看任务调度，通过future.get()可以获取返回的计算结果55。future.get()是一个阻塞方法，如果task线程执行需要很长时间，那么main线程将会停在这里一直到task线程处理完成。 

```java
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Task task = new Task(10);
        Future<Integer> future = executor.submit(task);
        Integer result = future.get();
        LOGGER.info("result=" + result);
```

这里和new Thread().start()没有本质的区别，都达到相同的效果，区别在于这里使用了线程池。

- Runnable：完成相同的功能，由于run()方法没有返回值，我们在构造Task对象时增加了Data参数

```java
    public class Task implements Runnable {
        private final Data data;
        private int count = 0;
        public Task(int count, Data data) {
            this.count = count;
            this.data = data;
        }

        @Override
        public void run() {
            int sum = 0;
            for (int i=1; i<=count; i++) {
                sum += i;
            }
            data.setResult(sum);
        }
    }

    public static class Data {
        private Integer result;
        public Integer getResult() {
            return result;
        }
        public void setResult(Integer result) {
            this.result = result;
        }       
    }	
```

再来看看任务调度，通过future.get()可以获得Data对象，然后从Data对象中获取计算结果55。

这里调用的是submit(Runnable task, T result)方法，如果调用submit(Runnable task)方法是无法获取到计算结果的，future.get()返回null，只能表明计算任务已经完成。

```java
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Data data = new Data();
        Task task = new Task(10, data);
        Future<Data> future = executor.submit(task, data);
        Integer result = future.get().getResult();
        LOGGER.info("result=" + result);
```

> 这里重点演示submit()的用法，线程池使用了最简单的SingleThreadExecutor

#### shutdown

shutdown()方法在关闭ExecutorService之前等待提交的任务执行完成，shutdownNow()方法阻止开启新的任务并且尝试停止当前正在执行的线程。

下面通过例子验证一下

- 定义任务：没有实际逻辑，只是sleep 1秒钟，任务开始和结束后输出日志

```java
    public class Task implements Runnable {
        @Override
        public void run() {
            LOGGER.info("run");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                LOGGER.error(e.getMessage());
            }
            LOGGER.info("done");
        }
    }
```

- 主线程：一共5个任务，为方便观察每提交一个任务后等待100毫秒；当提交完成3个任务后，分别调用shutdown()和shutdownNow()关闭线程池，然后观察日志输出

```java
        ExecutorService executor = new ThreadPoolExecutor(5, 10, 10, TimeUnit.SECONDS, 
                new LinkedBlockingDeque<>(5), new ThreadPoolExecutor.AbortPolicy());
        for (int i=0; i<5; i++) {
            Task task = new Task();
            try {
                executor.submit(task);
            } catch (Exception e) {
                LOGGER.error(e.getMessage());
            }
            Thread.sleep(100);
            if (i == 2) {
                executorService.shutdown();
              	//executorService.shutdownNow();
            }
        }
        LOGGER.info("done");
```

- shutdown结果

```shell
17:34:01:347 INFO [pool-2-thread-1] run
17:34:01:456 INFO [pool-2-thread-2] run
17:34:01:566 INFO [pool-2-thread-3] run
17:34:01:675 ERROR [main] Task rejected from ThreadPoolExecutor@36b4cef0[Shutting down]
17:34:01:784 ERROR [main] Task rejected from ThreadPoolExecutor@36b4cef0[Shutting down]
17:34:01:894 INFO [main] done
17:34:02:347 INFO [pool-2-thread-1] done
17:34:02:456 INFO [pool-2-thread-2] done
17:34:02:566 INFO [pool-2-thread-3] done
```

从日志中可以看出，shutdown()后再submit()任务时抛出了异常，提示正在“Shutting down”；正确启动了三个任务，这三个任务在shutdown()后继续执行，正常结束。

- shutdownNow结果

```shell
17:37:00:663 INFO [pool-2-thread-1] run
17:37:00:756 INFO [pool-2-thread-2] run
17:37:00:866 INFO [pool-2-thread-3] run
17:37:00:975 ERROR [pool-2-thread-3] sleep interrupted
17:37:00:975 ERROR [pool-2-thread-2] sleep interrupted
17:37:00:975 ERROR [pool-2-thread-1] sleep interrupted
17:37:00:975 INFO [pool-2-thread-3] done
17:37:00:975 INFO [pool-2-thread-1] done
17:37:00:975 INFO [pool-2-thread-2] done
17:37:00:975 ERROR [main] Task rejected from ThreadPoolExecutor@36b4cef0[Terminated]
17:37:01:085 ERROR [main] Task rejected from ThreadPoolExecutor@36b4cef0[Terminated]
17:37:01:194 INFO [main] done
```

首先，shutdownNow()后再submit()的任务也抛出了异常，但提示信息有差别；另外，虽然也输出了"donw"的信息，但是正确启动的三个任务并没有正常结束（时间不到1秒），日志显示中断了sleep操作，导致线程提前结束。

综上，我们应该使用shutdown()来关闭线程池。这也是web服务可以优雅关闭的基础，当tomcat接收到网络请求后，会提交给线程池进行处理，如果我们通知tomcat关闭服务，那么只需调用线程池的shutdown()方法，这样新的网络请求就没有线程来处理了（需要处理异常），而且正在工作的线程可以正常完成自己的工作后结束。



### ThreadPoolExecutor

平时我们最常用到的是ThreadPoolExecutor类，它实现了ExecutorService接口。

```java
package java.util.concurrent;
public class ThreadPoolExecutor extends AbstractExecutorService {
}
```

```java
package java.util.concurrent;
public abstract class AbstractExecutorService implements ExecutorService {
}
```

我们知道，Executor接口含有execute()方法，ExecutorService接口含有submit()方法，这样就有两种方法提交任务给线程池ThreadPoolExecutor执行。当我们不关注任务的返回结果时，可以通过execute()方法提交任务；当我们需要拿到任务的返回结果时，就必须通过submit()方法提交任务了。

> 除了get()方法可以得到返回结果以外，Future类还提供了其他方法来查询线程的执行状态，所以当我们关系任务的执行状态时，也应该使用submit()，个人建议可以都使用submit()来提交任务。另外，submit()最后还是执行的execute()，只不过执行前做了Future的准备工作。

```java
public abstract class AbstractExecutorService implements ExecutorService {    
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }  
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}

public class FutureTask<V> implements RunnableFuture<V> {
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }  
}
```

> TODO: Future原理