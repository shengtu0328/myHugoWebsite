---
title: "volatile（易变关键字）"
date: 2023-06-19T18:01:48+08:00
draft: true
---

## 一.可见性

它可以用来修饰成员变量和静态成员变量，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取

它的值，线程操作 volatile 变量都是直接操作主存









## 二.有序性



### cpu角度

为了在一个时间内，并行执行多条指令，会出现指令次序变化的情况。充分利用cpu



现代 CPU 支持**多级指令流水线**，例如支持同时执行 取指令 - 指令译码 - 执行指令 - 内存访问 - 数据写回 的处理

器，就可以称之为**五级指令流水线**。这时 CPU 可以在一个时钟周期内，同时运行五条指令的不同阶段（相当于一

条执行时间最长的复杂指令），IPC = 1，本质上，流水线技术并不能缩短单条指令的执行时间，但它变相地提高了

指令地吞吐率。





### jvm角度

指令重排，是 JIT 编译器在运行时的一些优化。为了JVM 会在不影响正确性的前提下，可以调整语句的执行顺序。但是在**多线程**下 会有问题



volatile 的底层实现原理是内存屏障，Memory Barrier（Memory Fence） 

对 volatile 变量的写指令后会加入写屏障

对 volatile 变量的读指令前会加入读屏障







**1.** **如何保证可见性**

写屏障（sfence）保证在该屏障之前的，对共享变量的改动，都同步到主存当中

```
public void actor2(I_Result r) {
    num = 2;
    ready = true; // ready 是 volatile 赋值带写屏障
    // 写屏障
}
```

而读屏障（lfence）保证在该屏障之后，对共享变量的读取，加载的是主存中最新数据

```
public void actor1(I_Result r) {
    // 读屏障
    // ready 是 volatile 读取值带读屏障
    if(ready) {
    r.r1 = num + num;
     } else {
    r.r1 = 1;
     }
}
```

**2.** **如何保证有序性**

**写屏障会确保指令重排序时**，不会将写屏障之前的代码排在写屏障之后

```
public void actor2(I_Result r) {
    num = 2;
    ready = true; // ready 是 volatile 赋值带写屏障
    // 写屏障
}
```

**读屏障会确保指令重排序时**，不会将读屏障之后的代码排在读屏障之前

```
public void actor1(I_Result r) {
    // 读屏障
    // ready 是 volatile 读取值带读屏障
    if(ready) {
    r.r1 = num + num;
     } else {
    r.r1 = 1;
     }
}
```







### synchronized



synchronized里的代码 也有可能会发生指令重排序。在多线程环境中 ，synchronized的代码会被多个线程串行执行。所以不会有重排序的问题。但如果在synchronized保护的代码块之外有其他线程在操作共享变量，那就有可能会出现问题。有时可以通过volatile来解决