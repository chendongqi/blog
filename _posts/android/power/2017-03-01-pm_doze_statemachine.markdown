---
layout:      post
title:      "Android电源管理之Doze模式专题系列（二）"
subtitle:   "Doze代码分布和状态机介绍"
navcolor:   "invert"
date:       2017-03-01
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - PowerManager
    - Doze
---

### 1. 代码分布

&emsp;&emsp;在Android M版本中为了实现Doze模式，新增了一个DeviceIdle的服务，其实现的代码位于frameworks/base/services/core/java/com/android/server/DeviceIdleController.java中。SystemServer在开机时会启动这个服务。        
```java
    mSystemServiceManager.startService(DeviceIdleController.class);
```
&emsp;&emsp;关于doze模式的控制逻辑都是在这个新增的服务中实现的，在上一篇[初识Doze](http://chendongqi.me/2017/02/28/pm_doze_MeetDoze/)一文中已经提到了进入doze之后的几个功耗策略：限制网络连接、阻止partial类型的wakelock、阻止Alarm、系统不扫描wifi热点、阻止sync任务、不允许JobScheduler进行任务调度。所以除了控制逻辑之外，还在NetworkPolicyManagerService、JobSchedulerService、SyncManager、PowerManagerService和AlarmManagerService中加入了对doze状态的监听和查询接口来进行响应的操作。其控制逻辑和策略实现的代码关系如下图所示。在DeviceIdleController中实现对设备状态的控制和改变，并且通知其他相关注册了AppIdleStateChangeListener接口的服务进行处理，而反过来这些服务也可以向DeviceIdleController查询device的状态，是一种交互的关系。    
![doze_stateMachine.png](https://chendongqi.github.io/blog/img/2017-02-28-pm_doze/doze_stateMachine.png)    

### 2. Doze模式中的状态机和其切换

&emsp;&emsp;oze模式的核心思想涉及了设备状态的切换以及不同状态下的功耗策略处理，所以类似于wifi连接，状态机是整个Doze模式设计中的重要部分，这一章来详细介绍下Doze模式的状态切换以及其代码实现。    
&emsp;&emsp;在[初识Doze](http://chendongqi.me/2017/02/28/pm_doze_MeetDoze/)一文中简单介绍了Doze模式中包含的几个状态，这里讲重点介绍下各个状态的含义以及切换的触发条件，并剖析代码中的实现。    

#### 2.1 Doze状态机介绍

&emsp;&emsp;Doze模式具体包含了7种状态：1）当设备亮屏或者处于正常使用状态时其就为ACTIVE状态；2）ACTIVE状态下不插充电器或者usb且灭屏设备就会切换到INACTIVE状态；3）INACTIVE状态经过30分钟，期间检测没有打断状态的行为Doze就切换到IDLE_PENDING的状态；4）然后再经过30分钟以及一系列的判断，状态切换到SENSING；5）在SENSING状态下会去检测是否有地理位置变化，没有的话就切到LOCATION状态；6）LOCATION状态下再经过30s的检测时间之后就进入了Doze的核心状态IDLE；7）在IDLE模式下每隔一段时间就会进入一次IDLE_MAINTANCE，此间用来处理之前被挂起的一些任务；8）IDLE_MAINTANCE状态持续5分钟之后会重新回到IDLE状态；9）在除ACTIVE以外的所有状态中，检测到打断的行为如亮屏、插入充电器，位置的改变等状态就会回到ACTIVE，重新开始下一个轮回。    
![doze_mode_state.png](https://chendongqi.github.io/blog/img/2017-02-28-pm_doze/doze_mode_state.png)    

&emsp;&emsp;**参考资料：**    
&emsp;&emsp;eCourse：[M Doze&AppStandby](https://onlinesso.mediatek.com/Pages/eCourse.aspx?001=002&002=002002&003=002002001&itemId=560&csId=%257B433b9ec7-cc31-43c3-938c-6dfd42cf3b57%257D%2540%257Bad907af8-9a88-484a-b020-ea10437dadf8%257D)     
&emsp;&emsp;eService：[关于doze模式是否支持的疑问](http://eservice.mediatek.com/eservice-portal/issue_manager/update/2062164)

