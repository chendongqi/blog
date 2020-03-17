---
layout:      post
title:      "Android性能优化之System Performance"
subtitle:   "Android整机项目中整机系统性能的优化"
navcolor:   "invert"
date:       2017-02-16
author:     "Cheson"
catalog: true
tags:
    - Android
    - Performance
---

### 代码和平台版本

Android Version： 6.0
Platform： MTK MT6735/6737

### 1. LMK参数调整
&emsp;&emsp;LMK(Low Memory Kill)机制，是Android系统继承自Linux的内存管理方案。此机制的实现细节可以参加另一篇文章中的专项介绍。这里介绍工程方面的几点修改方案，修改文件为frameworks/base/services/core/java/com/android/server/am/ProcessList.java。

#### 1.1 调整MAX_CACHED_APPS
先说代码修改，将

```java
static final int MAX_CACHED_APPS = 32;
```
修改成

```java
static final int MAX_CACHED_APPS = 16;
```
备注：MAX_CACHED_APPS标示了后台应用的最大数量，减小这个值之后达到的效果是留出了更多的可用内存，保证系统内存上的流畅性。带来的影响是后台缓存的进程减少，如果有频繁的多个应用轮流运行就会产生频繁的杀进程起进程的动作，对系统也会增加一定负担，所以这个值也是要选取一个最佳的平衡点。例如LG会有一个专门的轮询测试，轮流启动50个app，测试启动时间，此时就需要特别关注这个值的影响和尝试。

#### 1.2 调整OOM门限值

代码修改，将

```java
private final int[] mOomMinFreeHigh = new int[] {
        73728, 92160, 110592,
        129024, 147456, 184320
};
```
修改成

```java
private final int[] mOomMinFreeHigh = new int[] {
        36864, // 36 * 1024 (ADJ 0 -> 36MB)
	    49152, // 48 * 1024 (ADJ 1 -> 48MB)
	    61440, // 60 * 1024 (ADJ 2 -> 60MB)
	    73728, // 72 * 1024 (ADJ 3 -> 72MB)
	    204800, // 200 * 1024 (ADJ 9 -> 200MB) (based on performance test)
	    296960	// 290 * 1024 (ADJ 15 -> 290MB (=200MB x 1.25 x 1.75 /1.5)
};
```
这组数组是通过性能测试的经验和不断调整的效果得出来的，其含义是定义了oom机制中adj等级和剩余内存的对应关系，从这里的例子来说也就是当剩余内存低至290M时，开始杀adj 15的进程。策略的目的也就是为了保证可用内存的充裕。

#### 1.3

代码修改，将

```java
/**
 * Return the maximum pss size in kb that we consider a process acceptable to
 * restore from its cached state for running in the background when RAM is low.
 */
long getCachedRestoreThresholdKb() {
    return mCachedRestoreLevel;
}
```
修改为

```java
/**
 * Return the maximum pss size in kb that we consider a process acceptable to
 * restore from its cached state for running in the background when RAM is low.
 */
long getCachedRestoreThresholdKb() {
    return mCachedRestoreLevel / 2;
}
```
这里的修改含义是削减了返回的最大pss的容量，使得后台进程较之修改前不容易restore，是另一个保持可用内存的方案。

### 2. ZRAM配置

关于ZRAM的技术介绍参考[ZRAM简介](http://kernel.meizu.com/zram-introduction.html)，这里介绍在MTK平台上的配置方法。

**配置脚本**

修改文件device/mediatek/<project>/enableswap.sh，方案如下：

```Bash
#!/bin/sh
# modify by chendonqgi for LG performance [ZRAM change to 384M]
echo 402653184 > /sys/block/zram0/disksize
/system/bin/tiny_mkswap /dev/block/zram0
/system/bin/tiny_swapon /dev/block/zram0
# add by chendongqi for LG performance [Set swappiness]
echo 60 > /proc/sys/vm/swappiness // from 100-->>60
echo 60 > /sys/fs/cgroup/memory/sw/memory.swappiness // from 100-->60
```

**Adjust swappniess（虚拟内存管理中，换页频繁）**

修改文件/kernel/drivers/staging/android/lowmemorykiller.c

```C
vm_swappiness=100
```

### 3. IOWait调整

修改位置为device/mediatek/mt6735/init.mt6735.rc，添加

```Bash
on boot
    write /proc/sys/vm/dirty_background_ratio 3
    write /proc/sys/vm/dirty_ratio 10
```
IOWait调整的意义在于控制io操作的频率，来减少系统的负载，带来的影响为可能会出现iowait频率的变化导致的iowait过高出现的概率增加，这里也需要不断优化参数来找到平衡点。另外此项需要MTK评估平台是否支持，以及评估对flash的寿命影响问题。关于iowait的详细含义目前我也没有更深入的做探究，留待后续分析，提供一篇参考资料[iostat和iowait详细解说](http://blog.csdn.net/lhf_tiger/article/details/8926232)。

### 4. ION

ION为google的下一代内存管理器，具体介绍可以参考[ION基本概念介绍和原理分析](http://blog.csdn.net/zirconsdu/article/details/8969749)
修改方案如下：kernel3.10/drivers/staging/android/ion/mtk/ion_mm_heap.c：

```c
static const unsigned int orders[] = {2, 0};
```
改成

```c
static const unsigned int orders[] = {1, 0};
```

### 5. Rom读取速度优化

此项暂无通用方案，具体可以提交芯片商咨询

### 6. MTK平台调整perfSerice配置

配置不同场景中的CPU运行核数和主频
vendor/mediatek/proprietary/hardware/perfservice/<platform>/scn_tbl/perfservscntbl.txt

```Bash
CMD_SET_CPU_CORE, SCN_APP_TOUCH, 4    //根据项目配置来调整到最高核数，后面相同
CMD_SET_CPU_FREQ, SCN_APP_TOUCH, 1300000    //根据项目配置来调整到最高频率，后面相同
CMD_SET_CPU_CORE, SCN_SW_FRAME_UPDATE, 4
CMD_SET_CPU_FREQ, SCN_SW_FRAME_UPDATE, 1300000
CMD_SET_CPU_CORE, SCN_APP_SWITCH, 4
CMD_SET_CPU_FREQ, SCN_APP_SWITCH, 1300000
CMD_SET_CPU_UP_THRESHOLD, SCN_SW_FRAME_UPDATE, 80
CMD_SET_CPU_DOWN_THRESHOLD, SCN_SW_FRAME_UPDATE, 65
```
此项优化针对的是MTK平台的PerfService，此服务的作用在不同场景下进行cpu频率的调整

### 7. 优化Log输出

* 限制log输出等级： 暂无方案，在魅族项目上有过应用
* 限制不需要的log输出： 例如WindowManagerService中将DEBUG_BOOT = true改为false
* 解决log正的error异常： 搜索system.err
* 解决log中权限异常： 在main log或者kernel log中搜索avc:denied或者avc :denied
