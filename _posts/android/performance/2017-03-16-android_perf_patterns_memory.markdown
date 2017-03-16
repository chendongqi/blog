---
layout:      post
title:      "Android Performance Patterns——Memory Performance"
subtitle:   "Android Performance Patterns系列学习思考和实践笔记"
navcolor:   "invert"
date:       2017-03-15
author:     "Cheson"
#header-img: "https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_common.png"
catalog: true
tags:
    - Android
    - Performance
    - Android Performance Patterns
---

### 0. Garbage Collection in Android

&emsp;&emsp;Android的内存管理是一个三级Generation的模型，最近分配的对象存放在Young Generation中，在该区域停留达到一定程度之后会被转移到Old Generation中，最后会被存放到Permanent Generation中。    
![android_memory_gc_mode.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/android_memory_gc_mode.png)
&emsp;&emsp;每个级别的区域中都有一定的空间限制，如果分配的对象空间达到阈值时就会触发GC操作。在不同的Generation中做GC操作的速度是不同的，Young Generation中每次GC操作时间最短，Old Generation其次，Permanent Generation最慢。当然GC操作的时间和遍历的对象数量也是相关的。GC是一个阻塞的操作，在进行GC时，其他的线程都是处于暂停状态的。这个知识点和异常分析有很大的关系，通常在分析ANR和SWT的问题时，会优先看callstack中suspend状态的thread，通常这些thread是比较可疑的，卡在native调用或者binder调用，又或者是发生了死锁卡住。然后还有写suspend的情况却不属于异常，其中一种就是恰好在做GC时，其他thread都会被suspend；还有种情况是arm在dumpstacktrace时也会把thread都suspend。  
&emsp;&emsp;单个的GC操作并不耗时，但是频繁的GC就会对性能产生一定影响了。例如对UI Performance的影响，一帧的渲染需要在16ms内完成才能保证有流程的用户体验，如果频繁的GC占用了太多的CPU时间，导致无法及时完成这一帧的渲染，性能问题就随之而来了。这个就是GC这面双刃剑带来的危害，当然正常情况下剑总是冲敌人的，下面来看下什么情况下会对自己造成伤害。  
![gc_overtime.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/gc_overtime.png)
&emsp;&emsp;导致GC频繁发生的原因可能有两个：1、内存抖动，
Android内存三级模型
GC的特点（其他线程暂停，suspend，和ui性能的关系）
GC的触发原因：内存抖动和瞬间大量对象

### 1. Memory Churn & Performance

内存抖动的例子：for循环onDraw方法中分配内存，
案例设计，
查看工具（Memory Monitor和Allocation Tracker），
解决方法（对象池）

### 2. Performance Cost of Memory Leaks

分析的方法
案例
MAT工具的使用实践



### 3. Tools

Memory monitor  
Allocation Tracker  
Heap Tool

