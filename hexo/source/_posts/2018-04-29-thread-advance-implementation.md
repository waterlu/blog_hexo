---
title: 多线程高级篇(1) 线程实现
date: 2018-04-29 11:34:52
categories: 并发编程
tags: [并发, 多线程]
toc: true
description: 本文探究Java线程的底层实现，包括windows平台和Linux平台下的线程理论、原理和基本实现。
comments: false
---

## 线程实现

Java线程的实现有三种策略：

- 内核实现，Java线程相关方法声明为native，所有操作直接调用操作系统内核的API，此时的Java线程与操作系统内核线程一一对应；
- 用户实现，Java线程对操作系统是透明的，自己实现线程调度；
- 混合实现，综合上面两种实现的折中方案。

Java线程在JDK1.2之前采用用户实现策略，之后替换为内核实现。也就是说，一个Java线程就对应操作系统一个实际的线程（轻量级进程）。

Java线程使用抢占式的调度策略，由操作系统来分配执行时间。由于各个操作系统的线程优先级粒度不一样，所以Java线程优先级设置很难与操作系统线程优先级一一对应，所以就不那么靠谱。

![Java线程实现](/images/thread_os.png)

上图中，KLT是Kernel Thread的缩小，表示这是一个操作系统内核线程；LWP是Light Weight Process的缩写，表示轻量级进程。应用程序一般不直接使用内核线程，而是使用轻量级进程作为接口；P表示一个Java进程。

所以，上图的含义为：一个Java进程中可以创建多个线程，每个线程调用LWP接口，实际对应着一个内核线程，内核线程由操作系统调度。

Linux 2.6以后有了一种新的pthread线程库--NPTL(Native POSIX Threading Library)，hotspot虚拟机就是使用NPTL来实现多线程的。