---
layout: blog
title: "JVM中的Safe Point是什么"
catalog: true
header-img: img/post-bg-2.jpg
subtitle: Safe Point的定义和实现机制
date: 2024-08-02
tags: [2024,Java,JVM]
---

## 什么是Safe Point
> A point during program execution at which all GC roots are known and all heap object contents are consistent. From a global point of view, all threads must block at a safepoint before the GC can run.

> 程序执行期间所有 GC 根已知并且所有堆对象内容一致的点。从全局角度来看，所有线程都必须在安全点阻塞，然后 GC 才能运行。

[HotSpot Glossary of Terms](https://openjdk.org/groups/hotspot/docs/HotSpotGlossary.html)

Safe Point正如其名称是一个安全点，是程序执行过程中，线程可以安全停留的一个特殊位置。所有的线程在执行到Safe Point的时候就会去检查是否需要执行`进入Safe Point`操作，如果需要执行，那么所有的线程都将会等待，等待所有的线程进入Safe Point，JVM执行完相关操作后，线程再重新恢复执行。

> JVM通过SafePoint来通知所有线程需要阻塞，以此来实现STW。

当线程进入`Safe Point`时，线程栈上和堆上的映射关系是确定的，只要线程还在Safe Point，JVM就可以安全地修改内存（包括堆和栈），而不会影响线程在恢复执行后对程序状态的一致性认知。
> 线程状态的一致性认知:线程恢复前后，对内存的变化(如进行垃圾回收、内存压缩等)不感知。<br>
> 针对GC场景来说，堆上主要包括堆内存的重新调整，如移动对象，更新引用等。<br>
> 而栈主要是更新栈上对象的引用，确保其指向正确的（可能已被移动的）对象。

 safepoint 不仅仅用在GC上，在比如 deoptimization、Class redefinition 都有使用，只是 GC safepoint 比较知名。

## 线程什么时候会进入Safe Point
当线程在锁竞争(blocked on a lock)、同步阻塞(synchronized block)、被暂停(LockSupport#park())、IO被阻塞(blocked on blocking IO)、等待获得监视器锁状态(waiting on a monitor,如wait()方法)时，线程就处于safepoint状态。

线程在执行JNI(Java Native Interface)代码时，在进入native方法时，当前线程也是处于safepoint状态。在JNI方法调用结束时，会默认进行safepoint检查。

> A Java thread which is executing bytecode is NOT at a safepoint (or at least the JVM cannot assume that it is at a safepoint).

> 正在执行字节码的 Java 线程不处于安全点（或者至少 JVM 不能假设它处于安全点）。

[Under the hood JVM: Safepoints](https://medium.com/software-under-the-hood/under-the-hood-java-peak-safepoints-dd45af07d766)

JVM中线程和safepoint有以下关系：
+ JVM 无法强制任何线程进入安全点状态。
+ JVM 可以阻止线程离开安全点状态。
