---
layout:      post
title:      "Android LowMemoryKill"
subtitle:   "从源码剖析安卓LowMemoryKill机制"
navcolor:   "invert"
date:       2017-02-16
author:     "Cheson"
catalog: true
tags:
    - Android
    - Performance
    - LowMemoryKill
---


### 1 概述  

&emsp;&emsp;Linux kernel中有一个OOM的机制，就是Out Of Memory，用来应对系统内存不足的情况。当出现OOM时，内核有两种处理方式，1）死给你看；2）选择部分合适的process，杀掉之后空出些内存再积极的活下去。对于系统而言，第二种方式无疑是比较合适的处理。那么如何选择杀谁呢？内核中有一个oom_badness的函数用来给process打分，综合一些参数计算出分数，谁的分高就杀谁。这个就是linux kernel中处理低内存的一个方式。  
&emsp;&emsp;这篇文章中要介绍的LowMemoryKiller（后面简称LMK）机制是Android加入的处理低内存的一种机制，其思路也是仿照内核的OOM来设计的。在Android中，进程的生命周期都是系统来控制的，用户结束应用之后并不代表着进程就被杀死了，这也是为了下次启动应用时能更加快速。在LMK机制中，定义了两组数组，一组为系统的内存警戒值oom_minfree，定义了6个内存警戒值；一组为进程分数oom_adj，定义了6个进程的分值。LMK的机制为：当系统的内存达到一个警戒值时，去杀掉大于等于对应分值的进程。  
&emsp;&emsp;下面将从代码的角度来学习下LMK机制的实现。  

### 2 frameworks层

  oom_adj的分值是在用户层的代码中定义的，然后从上至下写入到sysfs文件系统的节点中。在Android4.0之后，frameworks中对oom_adj的定义都放在了ProcessList.java这个文件中（网上很多资料会说是在AMS和init.rc中，这个也是4.0之前的实现方式）。  
  下表中为ProcessList.java中对各种不同的process定义的分值。每种进程类型的含义资料待补充。  

| UNKNOWN_ADJ            | 16   |
| ---------------------- | ---- |
| UNKNOWN_ADJ            | 16   |
| CACHED_APP_MAX_ADJ     | 15   |
| CACHED_APP_MIN_ADJ     | 9    |
| SERVICE_B_ADJ          | 8    |
| PERVIOUS_APP_ADJ       | 7    |
| HOME_APP_ADJ           | 6    |
| SERVICE_ADJ            | 5    |
| HEAVY_WEIGHT_APP_ADJ   | 4    |
| BACKUP_APP_ADJ         | 3    |
| PERCEPTIBLE_APP_ADJ    | 2    |
| VISIBLE_APP_ADJ        | 1    |
| FOREGROUND_APP_ADJ     | 0    |
| PERSISTENT_SERVICE_ADJ | -11  |
| PERSISTENT_PROC_ADJ    | -12  |
| SYSTEM_ADJ             | -16  |
| NATIVE_ADJ             | -17  |

  在ProcessList.java中和LMK相关的四个数组：  
```java
private int[] mOomAdj;

// These are the low-end OOM level limits.  This is appropriate for an
// HVGA or smaller phone with less than 512MB.  Values are in KB.
private final int[] mOomMinFreeLow = new int[] {
        12288, 18432, 24576,
        36864, 43008, 49152
};
// These are the high-end OOM level limits.  This is appropriate for a
// 1280x800 or larger screen with around 1GB RAM.  Values are in KB.
/**
private final int[] mOomMinFreeHigh = new int[] {
        73728, 92160, 110592,
        129024, 147456, 184320
};
   */
//modify by chendongqi for LG performance:[HQ_LMK_OPT] @{
private final int[] mOomMinFreeHigh = new int[] {
        36864, // 36 * 1024 (ADJ 0 -> 36MB)
        49152, // 48 * 1024 (ADJ 1 -> 48MB)
        61440, // 60 * 1024 (ADJ 2 -> 60MB)
        73728, // 72 * 1024 (ADJ 3 -> 72MB)
        204800, // 200 * 1024 (ADJ 9 -> 200MB) (based on performance test)
        296960	// 290 * 1024 (ADJ 15 -> 290MB (=200MB x 1.25 x 1.75 /1.5)
};
///@}
// The actual OOM killer memory levels we are using.
private int[] mOomMinFree;
```
  第一个mOomAdj数组，就是用来存放达到境界值时被杀掉的对应进程的分值。它的初始化在ProcessList的构造方法中：  
```java
mOomAdj = new int[] {
    FOREGROUND_APP_ADJ, VISIBLE_APP_ADJ, PERCEPTIBLE_APP_ADJ,
    BACKUP_APP_ADJ, CACHED_APP_MIN_ADJ, CACHED_APP_MAX_ADJ
};
```
  这个代码中已经做了针对1G内存项目的优化方案，将此数据修改为了{0, 1, 2, 3, 9, 15}。  
  然后在ProcessList的构造方法的最后，执行了一次updateOomLevels方法，用来初始化另一个重要的数组，存放内存警戒值的mOomMinFree。  
```java  
updateOomLevels(0, 0, false);
```
  来看下在updateOomLevels方法中，具体做了些什么。其实主要代码就做了一件事，就是根据各种条件来计算出mOomMinFree这个数组，来确定内存的警戒值。然后往下传递。  
```java
for (int i=0; i<mOomAdj.length; i++) {
    int low = mOomMinFreeLow[i];
    int high = mOomMinFreeHigh[i];
    if (is64bit) {
        // Increase the high min-free levels for cached processes for 64-bit
        if (i == 4) high = (high*3)/2;
        else if (i == 5) high = (high*7)/4;
    }
    mOomMinFree[i] = (int)(low + ((high-low)*scale));
}
```
  计算的工作就是在这个循环中完成的，low就是已经初始化过的mOomMinFreeLow数组中的值，high就是已经初始化过的mOomMinFreeHigh数组中的值。  
```java
final boolean is64bit = Build.SUPPORTED_64_BIT_ABIS.length > 0;
```
  如果是64位，则将cached进程的临界值提高。这样设计的原因：两个猜测：1）64位系统中程序更吃内存，尽可能的保证更大的 内存；2）64位的cpu处理速度更快，提高临界值之后会导致更频繁的杀应用，增加的cpu负担在64位中可以接受。  
  然后就是根据low和high的值来计算mOomMinFree的值，这里需要另一个参数scale。  
```java
float scale = scaleMem > scaleDisp ? scaleMem : scaleDisp;
if (scale < 0) scale = 0;
else if (scale > 1) scale = 1;
```
  scale的值为scaleMem和scaleDisp中较大的那个，然后再判断赋值为0或者1。那么又需要看scaleMem和scaleDisp这两个参数。scaleMem如果大于等于700M，那么scaleMem的值就是大于等于1。  
```java
float scaleMem = ((float)(mTotalMemMb-350))/(700-350);
```
  scaleDisp的值是根据传入的分辨率来计算的，还记得在ProcessList构造方法中调用updateOomLevels时传入的参数吗，分辨率为0 \* 0。那么这里的逻辑肯定不对的。  
```java
// Scale buckets from screen size.
int minSize = 480*800;  //  384000
int maxSize = 1280*800; // 1024000  230400 870400  .264
float scaleDisp = ((float)(displayWidth*displayHeight)-minSize)/(maxSize-minSize);
```
  再继续看代码，发现在后面还有一次调用，会传入真正的设备分辨率。   
```java  
void applyDisplaySize(WindowManagerService wm) {
    if (!mHaveDisplaySize) {
        Point p = new Point();
        wm.getBaseDisplaySize(Display.DEFAULT_DISPLAY, p);
        if (p.x != 0 && p.y != 0) {
            updateOomLevels(p.x, p.y, true);
            mHaveDisplaySize = true;
        }
    }
}
```
  然后updateOomLevels方法会再走一次。以Tecno项目的例子，来看下前面讲过的流程中打出来的log。  
![log.png](https://github.chendongqi.io/blog/img/2017-05-23-lmk/log.png)
  可以看到最后scale的值计算为1.0。那么mOomMinFree数组的计算结果就是{36864, 49152, 61440, 73728, 204800, 296900}。  
  在计算完成之后还有两处代码可能会修正这个mOomMinFree  
```java
if (minfree_abs >= 0) {
    for (int i=0; i<mOomAdj.length; i++) {
        mOomMinFree[i] = (int)((float)minfree_abs * mOomMinFree[i]
                / mOomMinFree[mOomAdj.length - 1]);
    }
}

if (minfree_adj != 0) {
    for (int i=0; i<mOomAdj.length; i++) {
        mOomMinFree[i] += (int)((float)minfree_adj * mOomMinFree[i]
                / mOomMinFree[mOomAdj.length - 1]);
        if (mOomMinFree[i] < 0) {
            mOomMinFree[i] = 0;
        }
    }
}
```
  minfree_abs和minfree_adj这两个值是从配置文件中读取的  
```java
int minfree_adj = Resources.getSystem().getInteger(
        com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAdjust);
int minfree_abs = Resources.getSystem().getInteger(
        com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAbsolute);
```
  在Tecno项目中（平台默认配置），minfree_adj为0，minfree_abs为-1。所以后面两个修正的代码流程没有走。最终的mOomMinFree就是{36864, 49152, 61440, 73728, 204800, 296900}。  
  计算完之后就需要把mOomMinFree和mOomAdj这两个数组写到设备节点中去。第二次调用传输进来的write标志位为true，所以第二次才真正会写。1）首先需要构建一个ByteBuffer用来存储数据；2）存入cmd的标志，这里为LMK_TARGET，定义的值为0，会在后面根据这个标志来判断命令的类型。3）放入mOomMinFree数组的值，这里会进行一次换算，PAGE_SIZE的值为4 \* 1024，也就是内存管理中一个页的大小为4K。所以最后存放进去的数组为原数组除以4的结果{9216, 12288, 15360, 18432, 51200, 74240}，也就是表示页数。4）存入mOomAdj数组。5）调用writeLmkd方法来进入到下一层。  
```java
if (write) {
    ByteBuffer buf = ByteBuffer.allocate(4 * (2*mOomAdj.length + 1));
    buf.putInt(LMK_TARGET);
    for (int i=0; i<mOomAdj.length; i++) {
        buf.putInt((mOomMinFree[i]*1024)/PAGE_SIZE);
        buf.putInt(mOomAdj[i]);
    }

    writeLmkd(buf);
    SystemProperties.set("sys.sysctl.extra_free_kbytes", Integer.toString(reserve));
}
```
  来看一下writeLmkd方法  
```java
private static void writeLmkd(ByteBuffer buf) {

    for (int i = 0; i < 3; i++) {
        if (sLmkdSocket == null) {
                if (openLmkdSocket() == false) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException ie) {
                    }
                    continue;
                }
        }

        try {
            sLmkdOutputStream.write(buf.array(), 0, buf.position());
            return;
        } catch (IOException ex) {
            Slog.w(TAG, "Error writing to lowmemorykiller socket");

            try {
                sLmkdSocket.close();
            } catch (IOException ex2) {
            }

            sLmkdSocket = null;
        }
    }
}
```
  逻辑很简单，尝试三次，打开lmkd的socket，然后将前面构造的ByteBuffer写进去。  
```java
private static boolean openLmkdSocket() {
    try {
        sLmkdSocket = new LocalSocket(LocalSocket.SOCKET_SEQPACKET);
        sLmkdSocket.connect(
            new LocalSocketAddress("lmkd",
                    LocalSocketAddress.Namespace.RESERVED));
        sLmkdOutputStream = sLmkdSocket.getOutputStream();
    } catch (IOException ex) {
        Slog.w(TAG, "lowmemorykiller daemon socket open failed");
        sLmkdSocket = null;
        return false;
    }

    return true;
}
```
  通过打开socket的方法，可以追寻到是一个叫lmkd的服务端。  
  
### 3 lmkd  

  framework中的代码通过socket调用到system/core/lmkd/lmkd.c中，在上文中构造的ByteBuffer中首先放入了一个cmd的指令值LMK_TARGET，这个值就是在这儿用来判断命令的类型的。在ctrl_command_handler方法中进行处理。  
```C
case LMK_TARGET:
    targets = nargs / 2;
    if (nargs & 0x1 || targets > (int)ARRAY_SIZE(lowmem_adj))
        goto wronglen;
    cmd_target(targets, &ibuf[1]);
    break;
```
  这里判断到cmd就是LMK_TARGET，然后就调用cmd_target方法。在这个方法里：1）将传入的参数中的params中解析出framework中传递下来的内存警戒值和进程分值。  
``` C
for (i = 0; i < ntargets; i++) {
    lowmem_minfree[i] = ntohl(*params++);
    lowmem_adj[i] = ntohl(*params++);
}
```
  2）构造出一组带分隔符的数组，将内存警戒值和进程分值存放到数组中  
```C
for (i = 0; i < lowmem_targets_size; i++) {
    char val[40];

    if (i) {
        strlcat(minfreestr, ",", sizeof(minfreestr));
        strlcat(killpriostr, ",", sizeof(killpriostr));
    }

    snprintf(val, sizeof(val), "%d", lowmem_minfree[i]);
    strlcat(minfreestr, val, sizeof(minfreestr));
    snprintf(val, sizeof(val), "%d", lowmem_adj[i]);
    strlcat(killpriostr, val, sizeof(killpriostr));
}
```
  3）调用writefilestring来将值写入到设备节点中  
```C
writefilestring(INKERNEL_MINFREE_PATH, minfreestr);
writefilestring(INKERNEL_ADJ_PATH, killpriostr);
```
```C
static void writefilestring(char *path, char *s) {
    int fd = open(path, O_WRONLY);
    int len = strlen(s);
    int ret;

    if (fd < 0) {
        ALOGE("Error opening %s; errno=%d", path, errno);
        return;
    }

    ret = write(fd, s, len);
    if (ret < 0) {
        ALOGE("Error writing %s; errno=%d", path, errno);
    } else if (ret < len) {
        ALOGE("Short write on %s; length=%d", path, ret);
    }

    close(fd);
}
```
  而设备节点的路径定义的就是  
```C
#define INKERNEL_MINFREE_PATH "/sys/module/lowmemorykiller/parameters/minfree"
#define INKERNEL_ADJ_PATH "/sys/module/lowmemorykiller/parameters/adj"
```
  可以在手机中查看下  
![adj_minfree.png](https://github.chendongqi.io/blog/img/2017-05-23-lmk/adj_minfree.png)  
  查看设备节点会发现一个问题，minfree的数组确实为我们代码中写入的数组，但是adj的这组值却不是代码中传下去的{0,1,2,3,9,15}。  
  此处有玄机：从JB9.MP 后版本之后，LMK 自动将oom_adj 转换成 oom_score_adj ，即写入时依旧是按照oom_adj 写入，而读取出来时，则是oom_score_adj。这么做的好处是什么？用户空间只是负责配置这样两组数组，并传递下来写入到设备节点中。最后是由系统轮询内存时发现可用内存小于某一警戒值时再去杀死进程，而杀死哪个进程不是简单的由用户空间配置的oom_adj来决定的，这个只是一个计算因子，还会结果其他因素最后计算出一个进程的分数，也就是oom_score_adj，所以这里提前一步将oom_adj转换成了oom_score_adj，便于后面的处理。而每个进程也会有这样一个得分，存放在/proc/\<pid\>/oom_score_adj下面，可以看到下面截图，这个得分是一个动态计算的值，会直接和/sys/module/lowmemorykiller/parameters/minfree中对应。  
![process_score.png](https://github.chendongqi.io/blog/img/2017-05-23-lmk/process_score.png)    
  oom_adj和oom_score_adj数值转换的伪代码逻辑如下  
```C
if(oom_adj == 15) then oom_score_adj = 1000;
else oom_score_adj = oom_adj * 1000 / 17;
```
  简单的来说，有个对应关系：  

| oom_adj | oom_score_adj |
| ------- | ------------- |
| -16     | -947          |
| -12     | -705          |
| 0       | 0             |
| 1       | 58            |
| 2       | 117           |
| 3       | 176           |
| 4       | 235           |
| 6       | 352           |
| 9       | 529           |
| 15      | 1000          |



### 4 kernel  

  上面是用户空间的处理逻辑，做的事情就是配置oom_adj和oom_minfree这两组数组，并写入到设备节点中去。然后后面的事就是在kernel中来做了，主要的代码在kernel-3.18/drivers/staging/android/lowmemorykiller.c文件中。下面来将其中的重要处理逻辑剥离出来介绍一下。  
  首先是初始化  
```C
static int __init lowmem_init(void)
{
#ifdef CONFIG_HIGHMEM
	unsigned long normal_pages;
#endif

#ifdef CONFIG_ZRAM
	vm_swappiness = 100;
#endif


	task_free_register(&task_nb);
	register_shrinker(&lowmem_shrinker);

#ifdef CONFIG_HIGHMEM
	normal_pages = totalram_pages - totalhigh_pages;
	total_low_ratio = (totalram_pages + normal_pages - 1) / normal_pages;
	pr_err("[LMK]total_low_ratio[%d] - totalram_pages[%lu] - totalhigh_pages[%lu]\n",
			total_low_ratio, totalram_pages, totalhigh_pages);
#endif

	return 0;
}
```
  除去一些MTK的宏控制的补充功能部分，剩下的最重要的就是register_shrinker，参数为一个结构体的数组。结构体的定义是这样的。  
```C
static struct shrinker lowmem_shrinker = {
	.scan_objects = lowmem_scan,
	.count_objects = lowmem_count,
	.seeks = DEFAULT_SEEKS * 16
};
```
  注册的动作实现是在kernel-3.18/mm/vmscan.c文件中，将lowmem_shrinker结构体的指针注册进去之后，在内存分页回收时（什么鬼？），会在vmscan.c中调用到shrink_slab_node方法，而后就调用scan_objects和count_objects，最后回调到lowmemorykiller.c中定义的lowmem_scan和lowmem_count方法。  
  于是接下来来看lowmem_count和lowmem_scan方法，首先来看比较简单的lowmem_count方法。  
```C
static unsigned long lowmem_count(struct shrinker *s,
				  struct shrink_control *sc)
{
#ifdef CONFIG_FREEZER
	/* Do not allow LMK to work when system is freezing */
	if (pm_freezing)
		return 0;
#endif
	return global_page_state(NR_ACTIVE_ANON) +
		global_page_state(NR_ACTIVE_FILE) +
		global_page_state(NR_INACTIVE_ANON) +
		global_page_state(NR_INACTIVE_FILE);
}
```
  这里通过global_page_state方法来获取，这个方法的定义在kernel-3.18/include/linux/vmstat.h中。这个方法通过传入一个枚举类型的值来读取到内存状态。这个枚举类型在kernel-3.18/include/linux/mmzone.h中定义的。搜了资料没有比较详细的介绍这里的含义，大概来说就是获取匿名类型的内存（堆和栈）和文件映射的内存。这块的具体作用和参数含义还不是非常清楚。继续来看最重要的lowmem_scan函数吧。  
  首先定义了一些变量，说明几点：1）OOM_SCORE_ADJ_MAX定义的值为1000；  
```C  
unsigned long rem = 0;
int tasksize;
int i;
short min_score_adj = OOM_SCORE_ADJ_MAX + 1;
int minfree = 0;
int selected_tasksize = 0;
short selected_oom_score_adj;
int array_size = ARRAY_SIZE(lowmem_adj);
int other_free = global_page_state(NR_FREE_PAGES) - totalreserve_pages;
int other_file = global_page_state(NR_FILE_PAGES) -
                    global_page_state(NR_SHMEM) -
                    total_swapcache_pages();
```
  2）在lowmemrykiller.c中预设了两个adj和minfree的数组，大小都为9，所以array_size的默认值为9；  
```C
static short lowmem_adj[9] = {
	0,
	1,
	6,
	12,
};
static int lowmem_adj_size = 9;
int lowmem_minfree[9] = {
	3 * 512,	/* 6MB */
	2 * 1024,	/* 8MB */
	4 * 1024,	/* 16MB */
	16 * 1024,	/* 64MB */
};
```
  3）other_free和other_file用来表示可用内存，具体含义不是太清楚。  
  然后继续看代码流程，在这里进行了一个比较，因为用户空间已经配置了两组数据，所以这里lowmem_adj_size和lowmem_minfree_size的大小都为用户空间配置的6，而且lowmem_minfree数组和lowmem_adj数组的值都已经不是lowmemorykiller.c中的初始化值了，已经被重新赋值成用户空间配置的值。  
```C
if (lowmem_adj_size < array_size)
    array_size = lowmem_adj_size;
if (lowmem_minfree_size < array_size)
    array_size = lowmem_minfree_size;
for (i = 0; i < array_size; i++) {
    minfree = lowmem_minfree[i];
    if (other_free < minfree && other_file < minfree) {
        min_score_adj = lowmem_adj[i];
        break;
    }
}
```
  在哪儿改的？应该是调用了下图中两个方法。具体实现没有跟踪过，从log可以确定是已经被配置成用户空间设置的值了。  
```C
module_param_array_named(adj, lowmem_adj, short, &lowmem_adj_size,
			 S_IRUGO | S_IWUSR);
module_param_array_named(minfree, lowmem_minfree, uint, &lowmem_minfree_size,
			 S_IRUGO | S_IWUSR);
```
![lowmem_minfree.png](https://github.chendongqi.io/blog/img/2017-05-23-lmk/lowmem_minfree.png)   
  然后就是从小到大遍历lowmem_minfree数组中的警戒值，如果other_free且other_file这两个可用内存都小于警戒值，那么就记录下min_score_adj的值。如果min_score_adj和初始值相同，那么无需进行lowmemorykiller操作，输出log，解锁，然后返回0结束一次scan流程。否则的话继续往下走，将min_score_adj赋值给selected_oom_score_adj，作为后续选定需要杀死的进程adj。  
```C
if (min_score_adj == OOM_SCORE_ADJ_MAX + 1) {
    lowmem_print(5, "lowmem_scan %lu, %x, return 0\n",
             sc->nr_to_scan, sc->gfp_mask);

#if defined(CONFIG_MTK_AEE_FEATURE) && defined(CONFIG_MT_ENG_BUILD)
    /*
    * disable indication if low memory
    */
    if (in_lowmem) {
        in_lowmem = 0;
        /* DAL_LowMemoryOff(); */
        lowmem_print(1, "LowMemoryOff\n");
    }
#endif
    spin_unlock(&lowmem_shrink_lock);
    return 0;
}

selected_oom_score_adj = min_score_adj;
```
  选定了需要被杀死的adj的值，后面的事就是去杀进程。代码的流程就会走入到一个循环中  
```C
for_each_process(tsk)
```
  For_each_process为kernel-3.18/include/linux/sched.h中定义的一个宏，传入的参数tsk为task_struct类型的一个指针。  
```C
#define for_each_process(p) \
	for (p = &init_task ; (p = next_task(p)) != &init_task ; )
```  
  这里的逻辑就是便利系统中每一个task。然后又定义了个task_struct的指针p用来后面对每个task做处理，定义了一个oom_score_adj来存每个task的得分。用find_lock_task_mm来锁住一个task，将地址赋值给p。  
```C
struct task_struct *p;
short oom_score_adj;

if (tsk->flags & PF_KTHREAD)
    continue;

p = find_lock_task_mm(tsk);
```  
  取得这个task的oom_score_adj，这个就是/proc/\<pid\>/oom_score_adj，也即是在lmkd中写入的进程的得分。  
```C
oom_score_adj = p->signal->oom_score_adj;
```  
  如果task的分值小于系统选出来的分值，那么这个task就不用处理，解锁之后继续下一个。  
```C
if (oom_score_adj < min_score_adj) {
    task_unlock(p);
    continue;
}
```  
  选择要杀死哪个进程？选择的算法如下：  
```C
if (selected) {
    if (oom_score_adj < selected_oom_score_adj)
        continue;
    if (oom_score_adj == selected_oom_score_adj &&
        tasksize <= selected_tasksize)
        continue;
}
```  
  如果当前的task的oom_score_adj小于被选择的那个，那么不做修改；如果adj相同，那么当前task占用的内存小于等于被选择的那个时，也不做更改。换个角度来说也就是，第一个条件是看adj，谁大选谁；如果adj相同，那么选择占用内存大的那个。  
  接下来就是更新被选择的task的内存和adj。  
```C
selected = p;
selected_tasksize = tasksize;
selected_oom_score_adj = oom_score_adj;
lowmem_print(2, "select '%s' (%d), adj %d, score_adj %hd, size %d, to kill\n",
         p->comm, p->pid, REVERT_ADJ(oom_score_adj), oom_score_adj, tasksize);
```  
  然后打印出log记录，设置标志位，这是thread的标志位。  
```C
if (selected) {
    lowmem_print(1, "Killing '%s' (%d), adj %d, score_adj %hd,\n"
            "   to free %ldkB on behalf of '%s' (%d) because\n"
            "   cache %ldkB is below limit %ldkB for oom_score_adj %hd\n"
            "   Free memory is %ldkB above reserved\n",
             selected->comm, selected->pid,
             REVERT_ADJ(selected_oom_score_adj),
             selected_oom_score_adj,
             selected_tasksize * (long)(PAGE_SIZE / 1024),
             current->comm, current->pid,
             other_file * (long)(PAGE_SIZE / 1024),
             minfree * (long)(PAGE_SIZE / 1024),
             min_score_adj,
             other_free * (long)(PAGE_SIZE / 1024));
    lowmem_deathpending = selected;
    lowmem_deathpending_timeout = jiffies + HZ;
    set_tsk_thread_flag(selected, TIF_MEMDIE);
```  
  最后一步就是发送一个signal来杀死进程，并且更新可用内存的值，进入下一次循环了。  
```C
send_sig(SIGKILL, selected, 0);
rem += selected_tasksize;
```  


