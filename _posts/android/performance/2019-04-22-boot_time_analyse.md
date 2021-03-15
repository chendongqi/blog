---
layout:      post
title:      "开机时间分析总结"
subtitle:   "分析实例"
navcolor:   "invert"
date:       2019-04-22
author:     "Cheson"
catalog: true
tags:
    - Android
    - Performance
---

### 1 开机时间定义
分析开机时间,首先来看开机时间是如何定义的,此前提未搞清楚会让测试结果没有准确性可言.

#### 1.1 计时方法
一次完整又准确的开机时间测试,必须包含一个明确的计时起点和计时终点.  
在我们的项目中,终点可能会有好几种,例如最常见的是launcher显示出来,但这里有时也会有分歧,站在客户的角度上,launcher加载并不是终点,他们会已launcher上所有的icon和widget加载完整作为结束计时的终点.在有锁屏的项目上,会以锁屏显示会终点.而我们车机项目上添加了倒计时5秒的警告画面,所以我们测试时通常以此界面显示为终点.  
而起点的定义通常就是测试会跟研发出现分歧的区域了,通常测试人员测试时会通过长按power键来触发重启,在设备黑屏之后就开始计时,这个方式是不准确的.同样不准确的方式是我们在串口或者adb,通过reboot指令去让设备重启,然后从设备黑屏开始计时.此两种方式是异曲同工的.原因在于,真正我们应该作为开机的起点是uboot上电,而通过长按power键或者reboot指令让设备重启,在设备黑屏到下一次上电中间还是有1-2秒的延迟时间.所以最准确的方式就是通过串口观察uboot上电的第一行日志来作为开机计时的起点.如果没有此条件的测试,可以通过直接将电源下电,然后上电的同时开始计时.如果是实车测试,也没有条件直接让电瓶下电,只能通过长按power键重启测试,那就在测试结果上减掉1-2秒的时间.  

#### 1.2 分段计时
整体计时可以给出一次完整的开机时间结果,但有时候分析需要知道各个大阶段大致的耗时,这个大的阶段这里可以分割成kernel启动完整,systemserver开始启动.开机阶段可以细分成很多段,为什么这里只有两个点,因为我们从现象上只能观察到这两个点,而对应的现象就是kernel logo的显示和开机动画的播放.当看到kernel logo后可以掐一个时间,这个点大致就是kernel加载完毕的时间.开机动画开始播放可以掐一个时间,这个点大致就是systemserver开始启动的时间.通过对比这两段时间可以对那些开机时间很离谱的设备做一个大致的定性了.  

#### 1.3 测试标准
测试标准包括了以下几点:1.计时起点和终点必须统一;2.烧完版本后的前两次开机不计入到统计时间内;3.设备必须是不安装第三方应用且是出厂设置的状态;4.统计5次有效时间,计算平均值.  

### 2. 分析手段
分析原则就是对比,必须要有一个标杆,才能说开机时间是长还是短,需要优化的目标是多少.通常如果是MTK的平台,会有driveronly的版本作为一个标准时间,在此基础上加入我们客制化导致的影响时间,在容差范围内的就可以.  

#### 2.1 日志
要分析开机时间,必须先搞懂开机都分哪些阶段,这个可以参考之前分享的开机流程.大致从串口日志后者开机日志可以观察到以下几个阶段:1.uboot启动;2.kernel init done;3.zygote启动耗时;4.systemserver从开始到system ready;5.pms扫描的耗时;6.launcher启动.  
uboot启动可以从串口日志看到其耗时  
kernel init done在日志中可以看到同样的一条日志  
zygote启动耗时主要是看其加载资源的耗时,日志中可以搜到preload字样  
systemserver启动耗时可以在日志中搜systemserver,找到各个服务的启动时间  
pms扫描为systemserver启动阶段的耗时大户,可以通过搜索scan等字样找到  
launcher启动时间到关闭开机动画界面之间的耗时可以通过launcher进程启动日志和carsignal_daemon通知关闭开机动画的时间来计算  

#### 2.2 bootprof   
MTK平台在代码中植入了记录开机阶段的日志,可以在一次开机后,pull出来/proc/bootprof文件来直接观察,免去了自己从日志中搜索各个阶段的工作量.  
```xml
----------------------------------------                                 
0           BOOT PROF (unit:msec)                                         
----------------------------------------                                 
      3574        : preloader                                             
       936        : lk                                                   
       140        : lk->Kernel                                           
----------------------------------------                                 
      2274.252923 : Kernel_init_done                                     
      2562.207615 : INIT: on init start                                   
      2568.442230 : INIT:Mount_START                                     
      2946.715153 : INIT:Mount_END                                       
      3005.563153 : post-fs-data: on modem start                         
      4729.618846 : BOOT_Animation:START                                 
      5372.336615 : Zygote:Preload Start                                 
      5386.493538 : Zygote:Preload Start                                 
      6971.636538 : Zygote:Preload 343 obtain resources in 964ms         
      6985.962692 : Zygote:Preload 41 resources in 14ms                   
      6997.509615 : Zygote:Preload 343 obtain resources in 941ms         
      7020.312461 : Zygote:Preload 41 resources in 22ms                   
      7363.110461 : Zygote:Preload 3005 classes in 1986ms                 
      7367.374000 : Zygote:Preload 3005 classes in 1957ms                 
      7418.989461 : Zygote:Preload End                                   
      7442.382769 : Zygote:Preload End                                   
      7946.403615 : Android:PackageManagerService_Start                   
      8033.941077 : Android:PMS_scan_START                         
      8221.195923 : Android:PMS_scan_data_done:/system/framework       
      8837.749615 : Android:PMS_scan_data_done:/system/priv-app          
     11399.471308 : Android:PMS_scan_data_done:/system/app           
     11407.951385 : Android:PMS_scan_data_done:/system/plugin           
     11411.211462 : Android:PMS_scan_data_done:/data/app             
     11707.197769 : Android:PMS_scan_END 
     11777.238539 : Android:PMS_READY                                     
     24958.796386 : BOOT_Animation:END                     
----------------------------------------
```

#### 2.3 bootchart
android原生还支持通过bootchart去图形式的分析开机时间，特点在于可以清晰的看到整个开机过程中的cpu状态，各个进程启动的时间点，各个进程生命周期中对cpu的占用特征。详细的使用可以参见专门的bootchart篇。   

### 3. 优化方法
优化方法需要针对每个项目的问题去做定制,例如uboot启动时间长,就看下uboot,kernel如果启动耗时需要7秒多,那可能是某个驱动模块加载有问题或者是存在延时加载的情况.  
而上面的情况就比较复杂一点,通常需要较多的经验来看出问题所在并给出针对性的解法.举几种遇到过的问题类型.  
加载同样的资源,存在时间差异比较多的情况,那么可以在开机阶段观察cpu的占用情况,通常可以看到有个进程占用了cpu资源导致.而解法就需要此进程的owner来给出了.  
多次测试开机时间差异比较大,这种情况下也可以观察开机cpu占用,之前有发现过一个问题,开机阶段有进程直接while循环在占用cpu,导致其他进程的时间片分配比较不规律产生了此现象.这种情况就需要把写这块代码的工程师祭天了来优化了.  
其他通常都是常规的有迹可循的原因了,没有无缘无故的时间增加,必然是增加了预置apk或者资源,或者开机自启的进程和服务等。  
以下为经验总结的一些可以去检查的checklist  

#### 3.1  检查odex优化是否开启
要点如下：  

1. 打开WITH_DEXPREOPT := true宏  

2. 若user版本未提取odex，DONT_DEXPREOPT_PREBUILTS := true  //此句注释掉  

3. 针对64位芯片，只能作为32位运行的apk需要在预置时加入LOCAL_MULTILIB :=32  

4. 如果某应用不想使用odex优化，则可以在\build\core\dex_preopt_odex_install.mk做例外处理  

5. 开启odex优化，如果预置过多apk则会导致system.img过大，编译不过，则需要修改system分区  

检查第一次开机log，搜索PackageManager.DexOptimizer: Running dexopt (dex2oat) on，看看是否还有未优化的apk。  
#### 3.2 检查是否关闭patchodex功能

尽管开启了odex优化功能，但是在首次开机时，android会去做patchodex的动作，对odex稍作修改帮放到data/dalvik/&isa目录下。可以采用google的方案，开启WITH_DEXPREOPT_PIC：=true，这样既可以加速，又可以减少data分区。  

#### 3.3  对大型apk的编译优化

目前有些apk例如facebook，微信等，apk较大且代码复杂度较高，往往安装慢，是因为在dex2oat中编译慢，M中处理的方法是加入到白名单中，让它编译的时候做的简单些。  

#### 3.4 检查kernel配置是否有问题

查看bootprof，如果发现kernel启动时间较长，如7秒左右，其他每个启动阶段耗时都较长，可以让驱动排查下kernel是否配置成了eng版本.  

#### 3.5 检查启动过程中PMS扫描的时间

搜索log关键字scan package，查找是否有elapsed time>100ms的apk，这类apk需要减少  

#### 3.6 开机动画包，图片多或者占用内存多，会影响到开机时间

在kernel_log.boot中是否出现lowmemory，搜索关键字  

```java
[239.165493]<3>.(1)[70:kswapd0]lowmemorykiller: Candidate 432 (bootanimation), adj -18, score_adj -1000, rss 156937, rswap 2351, to kill
```
Solution：尽量控制开机动画包中的图片数量及每张图片的大小，part1部分循环播放的图片数量需要控制在10张以内  

#### 3.7 检查开机时cpu是否全速运转

烧eng或者userdebug的boot，然后在开机时查看CPU的运行频率：

```xml
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
```
查看cpu1~3是否online：

```xml
cat /sys/devices/system/cpu/cpu1/online
```
### 4 进一步优化手段  
上面的一些方式都是常规的排查项，就算全部都做了也只能达到一般的水准，这部分会介绍一些更加特殊和激进的方式，来让你的项目有别于其他。  

#### 4.1 Zygote多线程扫描资源
在ZygoteInit.java中将preload放到一个thread中去做。这个方法再4.4中可以使用，在MTK平台中已经有此方式了。  
#### 4.2 修改SystemServer的启动时机  
SystemServer在ZygoteInit.java中扫描完资源后启动修改成在扫描之前启动，在4.4中可以使用，MTK平台中尝试了一次未开机成功。  
#### 4.3 限制persistent进程
在AMS和PMS中加入了persistent进程的白名单，只有在此白名单中的进程可以通过persistent自启动，注意可能有service死掉后无法自启动的情况，修改下ActiveServices.java。  
另一种思路，我们可以定义一个黑名单，在黑名单中的进程不允许其开机时的自启动，但是给它死掉后自启的能力。  
#### 4.4 PMS扫描和安装多线程
在PMS scan中采用多线程的方式去减少扫描时间。  
首次开机的安装同样的思路，详情可以参考patch。  
