---
layout:      post
title:      "Android电源管理之Doze模式专题系列（九）"
subtitle:   "状态切换总结"
navcolor:   "invert"
date:       2017-04-10
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: false
tags:
    - Android
    - PowerManager
    - Doze

---

  前面已经把doze模式中各个状态之间的切换过程阐述了一遍，这一篇中就对Doze模式的状态切换做一下总结。  
![state_summary.png](https://chendongqi.github.io/blog/img/2017-02-28-pm_doze/state_summary.png)  
  doze模式的初始状态为ACTIVE，当处于灭屏状态下且未充电时，状态将切换到INACTIVE  
  在INACTIVE状态30分钟之后，Alarm被fire然后切换到IDLE_PENDING状态，做事情  
  在IDLE_PENDING状态30分钟之后再切换到SENSING，监听位置变化  
  在SENSING状态时依赖AnyMotionDector的回调来切换到LOCATING状态  
  LOCATING状态是会通过LocationService来更新location的状态  
  在Alarm再次唤醒时就会切换到IDLE状态，在此状态下就会禁用网络、Job、Sync、Wakelock等  
  在IDLE界面会持续60×N分钟的时间，N为进入的次数，最大不超过6小时。时间到了之后就进入到IDLE_MAINTENANCE  
  在IDLE_MAINTENANCE时处理IDLE状态下被Pending起来的任务，IDLE_MAINTENANCE的持续时间为5×N分钟，最大不超过10分钟  
  IDLE_MAINTENANCE持续时间到了之后再切回到IDLE状态  
  在任何状态下如果有位置变化，充电，亮屏等则状态会切回到ACTIVE重头开始。  