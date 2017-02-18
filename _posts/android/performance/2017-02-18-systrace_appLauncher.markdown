---
layout:      post
title:      "Android性能优化之Systrace分析app启动分析"
subtitle:   "google性能分析工具systrace分析app启动时间的实例"
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

此篇文章将介绍如何通过systrace来分析在launch界面click一个app的icon后app的启动时间，包括了animation off和animation on的情况，以google music应用为例。

### 2. 寻找InputReader-->AppLaunch_dispatchPtr:Down

根据Android系统事件传递的机制，第一步先找到frameworks层中事件出现的源头，就是在InputReader进程中的AppLaunch_dispatchPtr:Down这个动作。
![AppLaunch_dispatchPtr_down](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/AppLaunch_dispatchPtr_down.png)
下方可以看到其时间
![AppLaunch_dispatchPtr_down_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/AppLaunch_dispatchPtr_down_time.png)

### 3. 寻找InputReader-->AppLaunch_dispatchPtr: Up

然后找到click事件弹起的那个时间点：
![AppLaunch_dispatchPtr_up](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/AppLaunch_dispatchPtr_up.png)
![AppLaunch_dispatchPtr_up_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/AppLaunch_dispatchPtr_up_time.png)
这里看到down和up之间相差了19ms左右，这是因为手动点击时的迟滞时间导致，而每次操作这个值都可能不同。app的launch time应该以up为起始。

### 4. 寻找Launcher接收到Down事件

然后click事件会从frameworks中分发到launcher界面
![launcher_receive_down](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_down.png)
![launcher_receive_down_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_down_time.png)

### 5. 寻找Launcher接收到Up事件，up在down的后面

![launcher_receive_up](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_up.png)
![launcher_receive_up_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_up_time.png)

### 6. Launcher响应click事件

![launcher_receive_onclick](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_onclick.png)
![launcher_receive_onclick_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_receive_onclick_time.png)

### 7. Launcher开始通过AM去启动Activity

![launcher_amStart](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_amStart.png)
![launcher_amStart_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/launcher_amStart_time.png)

### 8.寻找WindowManager中的绘制start window的时间

![wm_start_window](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_start_window.png)
这里有个经验可以参考，一般在systrace的图形中，WindowManager的start在InputReader的后上方
![wm_start_window_experience](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_start_window_experience.png)
先找到wmAddStarting的时间
![wm_wmAdd](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_wmAdd.png)
![wm_wmAdd_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_wmAdd_time.png)
再看下绘制结束的时间
![wm_finishDraw](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_finishDraw.png)
![wm_finishDraw_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/wm_finishDraw_time.png)

### 9. 寻找surfaceflinger最终送帧的时间点

![sf_post](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/sf_post.png)
![sf_post_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/sf_post_time.png)

到这一步就是animation off情况下，启动music的第一帧的上层显示时间，也就是说如果关闭了启动动画，那么第一帧就是WindowManager绘制出的白屏画面。
如果从down开始计算是408.558-196.164=212.394，因为手动down的误差较大，如果从up时间开始算就是408.558-215.671=192.887ms。在animation on的情况下，情况又不同了，在以上的基础上还会走app的启动动画，接着来看。

### 10. 寻找music进程绘制ui的时间

先找到music进程的图形
![music_view](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/music_view.png)
然后找到finishiDraw
![music_finishDraw](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/music_finishDraw.png)
![music_finishDraw_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/music_finishDraw_time.png)
这里就可以计算出music绘制完第一帧的时间，然后继续查看附近的surfaceflinger的送帧时间
![music_sf_post](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/music_sf_post.png)
![music_sf_post_time](https:chendongqi.github.io/blog/img/2017-02-18-systrace_launch_time/music_sf_post_time.png)
这里就可以计算出music应用绘制和显示第一帧花了多少时间
2518.475-408.558=2109.917ms
加上前面绘制startWindow的时间就可以算出总时间
