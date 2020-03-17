---
layout:      post
title:      "Android性能优化之Systrace分析drag响应时间"
subtitle:   "google性能分析工具systrace分析drag事件的响应时间"
navcolor:   "invert"
date:       2017-02-18
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - Performance
    - systrace
---

### 适用平台

Android Version： 6.0及以上
Platform： 通用

### 1. 介绍

本篇介绍的内容，是用systrace来分析drag事件从framework层开始到app第一帧ui绘制完的时间。以contact应用中的drag为例。

### 2. system_server中inputReader接收到down事件的时间

![inputReader_down](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/inputReader_down.png)
![inputReader_down_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/inputReader_down_time.png)

### 3. inputReader的上报move事件的时间

drag动作关注的是第一个move动作产生的时间点：
![inputReader_move](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/inputReader_move.png)
![inputReader_move_time1](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/inputReader_move_time1.png)
![inputReader_move_time2](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/inputReader_move_time2.png)
可以看到从down事件到move事件中间用了0.75ms左右。

### 4. 找contact的UI线程绘制图形

![ui_draw](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/ui_draw.png)
![ui_draw_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/ui_draw_time.png)
所以看到finishDraw的时间点为1003.337ms，但是Drag中画面一直不稳定，不能通过finishDraw来定位最后的点，还需要看送到surfaceflinger中的时间。

### 5. surfaceflinger刷新

因为surfaceflinger刷新buffer中的数据上去的时间间隔如果按每秒60帧来计算的话是16.7秒，所以画面真正刷新的时间不一定为finishDraw的时间，还存在误差，再来看到surfaceflinger的信息。
![sf_post](https:chendongqi.github.io/blog/img/2017-02-18-systrace_drag_time/sf_post.png)
看到postFrameBuffer的开始时间点为1013.573ms，加上耗时0.776ms，这个第一帧刷新的准确时间为1014.349ms，比app中画面绘制结束晚了10ms左右。

### 6. 结论

所以cantact中drag事件framework层到看到画面第一帧的时间为1014.349-828.706=185.643ms。 
