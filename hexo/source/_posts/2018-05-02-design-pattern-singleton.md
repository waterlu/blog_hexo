---
title: Singleton
date: 2018-05-02 09:09:01
categories: 设计模式
tags: [单例, 线程安全]
toc: true
description: 单例是最常见的设计模式，如何实现单例呢？如果实现线程安全的单例呢？本文将展示单例的五种写法，比茴香豆还多一种。
comments: false
---

## Singleton

单例Singleton是最常见的设计模式，如何实现单例呢？如果实现线程安全的单例呢？

### 单线程单例

最常见的单例写法

- 声明一个静态实例
- 构造函数声明为私有，确保不能通过构造函数创建对象
- getInstance()时进行null判断

```java
public class OneThreadSingleton {
  private static OneThreadSingleton instance = null;
  private OneThreadSingleton() {
  }
  public static OneThreadSingleton getInstance() {
    if (null == instance) {
      instance = new OneThreadSingleton();
    }
    return instance;
  }
}
```

以上代码在单线程环境下进行测试，测试代码如下

```java
public class SingletonExample {
  public static void main(String[] args) {
    for (int i=0; i<10; i++) {
      OneThreadSingleton singleton = OneThreadSingleton.getInstance();
      System.out.println(singleton);
    }
  }
}
```

返回结果如下，都是一个实例，说明单例成功

```java
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
cn.waterlu.java.design.OneThreadSingleton@6f75e721
```

但是以上代码在多线程环境下是有问题的，多线程测试代码如下

```java
public class SingletonExample {
  public static void main(String[] args) {
    List<Task> taskList = new ArrayList<>();
    for (int i=0; i<10; i++) {
      Task task = new Task();
      taskList.add(task);
    }
    for (Task task : taskList) {
      task.start();
    }
  }
  public static class Task extends Thread {
    @Override
    public void run() {
      OneThreadSingleton singleton = OneThreadSingleton.getInstance();
      System.out.println(singleton);
    }
  }
}
```

返回结果如下，存在不一样的情况，单例失败

- 注意，结果是随机的，需要多运行几次才能出现返回不一样实例的情况
- 错误原因在于并发进行(null==instance)判断是不准确的，可能存在多个线程判断时instance都是null的情况

```shell
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@dbefca
cn.waterlu.java.design.OneThreadSingleton@6464d21
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
cn.waterlu.java.design.OneThreadSingleton@65d2066b
```

### 同步单例

以上经典的单例代码在多线程情况是有问题的，也就是非线程安全的，如果如何写出线程安全的单例呢？

以下代码是最容易想到的，给getInstance()方法增加synchronized关键字，这样就能保证线程安全了

- 首先肯定这种写法是对的，没有问题，可以实现线程安全的单例模式
- 这种写法虽然简单，但存在效率问题：每一次进入synchronized代码块都是需要加锁的，加锁和释放锁肯定是有开销的；绝大多数情况下，instance不等于null，加锁只在instance等于null时起作用，典型的悲观锁
- 既然这样，把synchronized放到if()判断里面不就行了，这就引出了另外一种实现

```java
public class SynchronizedSingleton {
  private static SynchronizedSingleton instance = null;
  private SynchronizedSingleton() {
  }
  public synchronized static SynchronizedSingleton getInstance() {
    if (null == instance) {
      instance = new SynchronizedSingleton();
    }
    return instance;
  }
}
```

### 双重检查单例

继续上面的思路，代码如下：

- synchronized代码块只在instance等于null时起作用，大多数情况下直接return instance即可
- 注意，这里有两个if (null == instance)判断，所以也被称为双重检查，那么为什么要做第二遍检查呢
- 因为synchronized虽然起到了互斥的作用，但是代码还是会执行的；假设，两个线程同时通过第一层null判断，抢到锁的线程执行了new操作；第一个线程释放锁以后第二个线程从阻塞状态唤醒执行，此时如果没有第二次判断，那么它还是会创建一个新的对象。

```java
public class DoubleCheckSingleton {
  private static DoubleCheckSingleton instance = null;
  private DoubleCheckSingleton() {
  }
  public static DoubleCheckSingleton getInstance() {
    if (null == instance) {
      synchronized (DoubleCheckSingleton.class) {
        if (null == instance) {
          instance = new DoubleCheckSingleton();
        }
      }
    }
    return instance;
  }
}

```

### 静态单例

其实还有更简单的单例实现方法，代码如下

- 代码非常简单，也是线程安全的，因为多线程只发生了读操作，当然是安全的
- 当然，这么做也有缺点，那就是没有延迟加载，俗称lazy load
- 即使StaticSingleton类没有被使用，它也创建对象并占用了内存空间；上面的例子都是延迟加载的，在第一次被使用时创建的对象

> 备注：我真心觉得这么写就挺好，又简单又高效，耗费那点内存空间真不算什么。

```java
public class StaticSingleton {
    private final static StaticSingleton instance = new StaticSingleton();
    private StaticSingleton() {
    }
    public static StaticSingleton getInstance() {
        return instance;
    }
}
```
### 内部静态类单例

- 在前面代码的基础上进行改造，把创建对象操作放到内部类中实现，以完成延迟加载
- 头一次看到这种代码可能会感到奇怪，一步一步分析过来就了解了

```java
public class Singleton {
    private Singleton() {
    }
    private static class SingletonInner {
        private static Singleton instance = new Singleton();
    }
    public static Singleton getInstance() {
        return SingletonInner.instance;
    }
}
```

>  这么多单例的实现方式，个人觉得这就和茴香豆的茴字有几种写法差不多，没啥意义。关键理解其中的思考方法，还有些意义。