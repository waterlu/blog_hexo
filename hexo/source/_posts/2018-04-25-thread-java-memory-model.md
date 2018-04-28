---
title: 多线程(4) Java内存模型
date: 2018-04-25 17:49:04
categories: 并发编程
tags: [并发, 线程, 内存模型]
toc: true
description: 什么是内存模型？首先，不要和JVM运行时数据区混淆了。JVM运行时数据区描述的是堆、栈等内存空间的组成，而内存模型是和多线程息息相关的，解决的是可见性和线程安全问题。
comments: false
---

## 内存模型

什么是内存模型？首先，不要和JVM运行时数据区混淆了。JVM运行时数据区描述的是堆、栈等内存空间的组成，而内存模型是和多线程息息相关的，解决的是可见性和线程安全问题。

![图1-Java内存模型](/images/thread_jmm.png)

上图描述了Java内存模型，

http://tutorials.jenkov.com/java-concurrency/



https://blog.csdn.net/suifeng3051





https://www.jianshu.com/u/f8e9b1c246f1



重排序：

编译器重排序

处理器重排序



volatile解决的是多线程之间读的可见性问题，如果只有一个线程写共享变量，多个线程读取共享变量，那么volatile可以保证读线程能够即使读取到共享变量的最新值，也就是说，一旦写线程修改了共享变量的值，那么读线程可以立即读取到最新的值，没有脏读。

但是，volatile不是用来解决多线程一起写的。只有在一种情况下，使用volatile多线程写共享变量不会出问题，那就是共享变量的新值与旧值没有关联，或者说新值是直接设置的，不需要通过旧值计算得到，这个时候即使有多线程写，volatile共享变量也是正确的。

实际情况中，这样的场景并不常见，多数场景新值都是依赖于旧值的。

volatile可以保证一旦写了，其他线程可以立即读取到；但是如果写之前其他线程做了读取操作，然后再一起写，这样就出问题了。



从内存语义的角度来说，volatile的写-读与锁的释放-获取有相同的内存效果

当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存

当读一个volatile变量时，JMM会把该线程对应的本地内存中的值置为无效，从主内存中读取共享变量



Load1 LoadLoad Load2	确保Load1在Load2之前加载（从主内存读取到工作内存）

Store1 StoreStore Store2	确保Store1在Store2之前存储（从工作内存回写到主内存）

Load1 LoadStore Store2	先加载Load1，然后在存储Store2

Store1 StoreLoad Load2	最重要，刷新工作内存中所有共享变量到主内存，然后再读取



为实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

在每一个volatile写操作前面插入StoreStore，后面插入StoreLoad

在每一个volatile



https://sourceforge.net/projects/fcml/files/fcml-1.1.1/hsdis-1.1.1-win32-amd64.zip/download

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp VolatileExample$Task > VolatileExample$Task.asm

```
-Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*LazySingleton.getInstance
```

-Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly



Object	wait(), notify(), notifyAll()

必须先获得锁，然后才可以调用wait()和notify()方法

Spurious Wakeup

虚假唤醒，没有notify()，wait()就被唤醒。为避免这个问题，使用while()进行判断，不要使用if()进行判断。

注意：使用Object.wait()方法时，这个Object可以是任何对象，但不要使用字符串常量和全局对象。因为字符串常量内部会指向同一个对象。

原本是两个对象，两个锁。但是实际上引用指向一个对象。可能引发问题。如果使用notifyAll()还没有什么大问题，如果使用notify()就有问题了。



Synchronized 语句块是可以重入的(reentrance)











一旦一个共享变量（类的成员变量、 类的静态成员变量） 被 volatile 修饰之后， 那么就具备了两层语义：

- 保证了不同线程对这个变量进行读取时的可见性， 即一个线程修改了某个变量的值， 这新值对其他线程来说是立即可见的。 (volatile 解决了线程间共享变量的可见性问题)。
- 禁止进行指令重排序， 阻止编译器对代码的优化。

# 内存可见性

- 第一： 使用 volatile 关键字会强制将修改的值立即写入主存；
- 第二： 使用 volatile 关键字的话， 当线程 2 进行修改时， 会导致线程 1 的工作内存中缓存变量 stop 的缓存行无效（反映到硬件层的话， 就是 CPU 的 L1或者 L2 缓存中对应的缓存行无效） ；
- 第三： 由于线程 1 的工作内存中缓存变量 stop 的缓存行无效， 所以线程 1再次读取变量 stop 的值时会去主存读取。

那么， 在线程 2 修改 stop 值时（当然这里包括 2 个操作， 修改线程 2 工作内存中的值， 然后将修改后的值写入内存） ， 会使得线程 1 的工作内存中缓存变量 stop 的缓存行无效， 然后线程 1 读取时， 发现自己的缓存行无效， 它会等待缓存行对应的主存地址被更新之后， 然后去对应的主存读取最新的值。

# 禁止重排序

volatile 关键字禁止指令重排序有两层意思：

- 当程序执行到 volatile 变量的读操作或者写操作时， 在其前面的操作的更改肯定全部已经进行， 且结果已经对后面的操作可见； 在其后面的操作肯定还没有进行
- 在进行指令优化时， 不能把 volatile 变量前面的语句放在其后面执行，也不能把 volatile 变量后面的语句放到其前面执行。

为了实现 volatile 的内存语义， 加入 volatile 关键字时， 编译器在生成字节码时，会在指令序列中插入内存屏障， 会多出一个 lock 前缀指令。 内存屏障是一组处理器指令， 解决禁止指令重排序和内存可见性的问题。 编译器和 CPU 可以在保证输出结果一样的情况下对指令重排序， 使性能得到优化。 处理器在进行重排序时是会考虑指令之间的数据依赖性。

内存屏障， 有 2 个作用：

- 1.先于这个内存屏障的指令必须先执行， 后于这个内存屏障的指令必须后执行。
- 2.使得内存可见性。 所以， 如果你的字段是 volatile， 在读指令前插入读屏障， 可以让高速缓存中的数据失效， 重新从主内存加载数据。 在写指令之后插入写屏障， 能让写入缓存的最新数据写回到主内存。

lock 前缀指令在多核处理器下会引发了两件事情：

1.将当前处理器中这个变量所在缓存行的数据会写回到系统内存。 这个写回内存的
操作会引起在其他 CPU 里缓存了该内存地址的数据无效。 但是就算写回到内存， 如果其他处理器缓存的值还是旧的， 再执行计算操作就会有问题， 所以在多处理器下， 为了保证各个处理器的缓存是一致的， 就会实现缓存一致性协议， 每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了， 当处理器发现自己缓存行对应的内存地址被修改， 就会将当前处理器的缓存行设置成无效状态， 当处理器要对这个数据进行修改操作的时候， 会强制重新从系统内存里把数据读到处理器缓存里

2.它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置， 也不会把前面的指令排到内存屏障的后面； 即在执行到内存屏障这句指令时， 在它前面的操作已经全部完成。

内存屏障可以被分为以下几种类型：

- LoadLoad 屏障： 对于这样的语句 Load1; LoadLoad; Load2， 在 Load2 及后续读取操作要读取的数据被访问前， 保证 Load1 要读取的数据被读取完毕。
- StoreStore 屏障： 对于这样的语句 Store1; StoreStore; Store2， 在 Store2 及后续写入操作执行前， 保证 Store1 的写入操作对其它处理器可见。
- LoadStore 屏障： 对于这样的语句 Load1; LoadStore; Store2， 在 Store2 及后续写入操作被刷出前， 保证 Load1 要读取的数据被读取完毕。
- StoreLoad 屏障： 对于这样的语句 Store1; StoreLoad; Load2， 在 Load2 及后续所有读取操作执行前， 保证 Store1 的写入对所有处理器可见。 它的开销是四种屏障中最大的。 在大多数处理器的实现中， 这个屏障是个万能屏障， 兼具其它三种内存屏障的功能。





