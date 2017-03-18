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
&emsp;&emsp;导致GC频繁发生的原因可能有两个：1、内存抖动，内存抖动是在短时间内分配和回收大量对象时出现的现象，主要是在循环中创建对象的代码导致，频繁分配大量对象就可能会触发GC；2、瞬间产生大量的对象，会导致Young Generatation区域中空间达到阈值而触发GC。  
![memory_monitor_gc.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/memory_monitor_gc.png)
&emsp;&emsp;分享一篇对GC讲解的比较通俗易懂的文章  
[Android GC那些事](https://zhuanlan.zhihu.com/p/20282779?columnSlug=magilu)
> **回收算法**  
**标记回收算法（Mark and Sweep GC）**  
从"GC Roots"集合开始，将内存整个遍历一次，保留所有可以被GC Roots直接或间接引用到的对象，而剩下的对象都当作垃圾对待并回收，这个算法需要中断进程内其它组件的执行并且可能产生内存碎片。  
**复制算法 (Copying）**  
将现有的内存空间分为两块，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。  
**标记-压缩算法 (Mark-Compact)**  
先需要从根节点开始对所有可达对象做一次标记，但之后，它并不简单地清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。这种方法既避免了碎片的产生，又不需要两块相同的内存空间，因此，其性价比比较高。  
**分代**  
将所有的新建对象都放入称为年轻代的内存区域，年轻代的特点是对象会很快回收，因此，在年轻代就选择效率较高的复制算法。当一个对象经过几次回收后依然存活，对象就会被放入称为老生代的内存空间。对于新生代适用于复制算法，而对于老年代则采取标记-压缩算法。  

&emsp;&emsp;看完这一段也就可以理解为何Young Generation中执行GC会比较快，因为其算法是用Copying，是一种以空间换时间的做法。  
&emsp;&emsp;当然老罗的博客是不可缺少的一手资料    
[Dalvik虚拟机垃圾收集（GC）过程分析](http://blog.csdn.net/luoshengyang/article/details/41822747)    


### 1. Memory Churn & Performance

#### 1.1 Typical Case

&emsp;&emsp;这一节中就来详细看下内存抖动的情况，以一个非常简单的案例开场，来看这样一段代码  
```java
    public class MainActivity extends AppCompatActivity {

        private Context mContext = null;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            mContext = getApplicationContext();

            testMemoryChurn();

        }

        private void testMemoryChurn() {
            for(int i = 0; i < 1000; i++) {
                createBitmap();
            }
        }

        private void createBitmap() {
            Bitmap bitmap = BitmapFactory.decodeResource(mContext.getResources(),R.drawable.android_perf_patterns_common);
        }
    }
```

#### 1.2 Memory Monitor & Allocation Tracker

&emsp;&emsp;这段程序跑起来后在Android Studio中用Memory Monitor工具来查看内存的情况，出现了非常明显的内存抖动情况，其引发的原因就是for循环中的分配对象，每次循环结束后都会被回收，导致内存不停的变化。    
![memory_churn.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/memory_churn.png)
&emsp;&emsp;在实际调试一个应用时，用Memory Monitor可以监视到内存的情况，但是无法定位到代码，如果看到有内存频繁或者长时间的内存抖动情况，如何去确定代码中的根因呢。这里需要用到Android Monitor中的另一个工具：Allocation Tracker。此工具的使用是需要手动抓取一段时间内的内存分配情况的，在操作之前点击下图中的Start Allocation Tracking，结束操作之后再点击stop。  
![allocation_tracker.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/allocation_tracker.png)
&emsp;&emsp;例如抓取到的内存抖动的一段内存情况，可以看到thread-1占了几乎100%的内存，分配的size很大，顺藤摸瓜就能找到问题的代码在于decodeResource方法中。  
![allocation_tracker_result.png](https://chendongqi.github.io/blog/img/2017-03-16-android_perf_patterns_memory/allocation_tracker_result.png)
&emsp;&emsp;这个案例非常简单，但也是此类问题非常典型的案例，通常在for循环中分配内存时就会出现这个问题，另一个可能导致内存抖动的罪魁祸首是在onDraw方法中分配内存，当view被频繁重绘的时候也会导致内存抖动。这里的内存抖动问题解决起来就很简单了，把对象分配的动作放在循环外就可以。  

#### 1.3 What is Memory Churn Exactly 

&emsp;&emsp;直观了解了内存抖动之后来思考这样一个问题，内存抖动为什么会引起GC呢？要思考这个问题之前先要明白GC的一个原理，什么情况下会触发GC。在前一节介绍GC时的老罗博客的参考资料中给出了GC的4中触发条件    
> GC_FOR_MALLOC: 表示是在堆上分配对象时内存不足触发的GC。
GC_CONCURRENT: 表示是在已分配内存达到一定量之后触发的GC。
GC_EXPLICIT: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。
GC_BEFORE_OOM: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。
实际上，GC_FOR_MALLOC、GC_CONCURRENT和GC_BEFORE_OOM三种类型的GC都是在分配对象的过程触发的。  
&emsp;&emsp;在这个背景知识的基础上再来理解内存抖动的情况，以上面案例中的代码来思考，在循环内部分配内存后，当结束一次循环时，局部变量虽然已经失效了，但是其内存空间并没有立即被回收（虽然Java可以通过GC自动回收内存，但也不是即时的），即使在使用完对象之后立即将其置为null，也不会立即回收其所占的内存空间，还是需要等待系统的GC操作。所以我们看到的内存抖动现象其实不应该被称之为频繁引起GC的原因，而应该理解为频繁分配对象的操作，导致了Young Generation中的剩余空间频繁达到阈值（内存抖动截图中的波峰）而触发GC，然后可用内存又降下来（波谷），产生了内存抖动的现象。  

#### 1.4 What Can Help Us

&emsp;&emsp;对



内存抖动的例子：for循环onDraw方法中分配内存，
案例设计，
查看工具（Memory Monitor和Allocation Tracker），
解决方法（对象池）

[Android 内存抖动 性能分析 <10>](http://blog.csdn.net/qq_31726827/article/details/50514263)

### 2. Performance Cost of Memory Leaks

分析的方法
案例
MAT工具的使用实践



### 3. Tools

Memory monitor  
Allocation Tracker  
Heap Tool

