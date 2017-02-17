---
layout:      post
title:      "Android性能优化之Rotation Performance"
subtitle:   "Android整机项目中旋转屏的性能优化"
navcolor:   "invert"
date:       2017-02-17
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - Performance
---

### 代码和平台版本

Android Version： 6.0
Platform： MTK MT6580/MT6737/MT6735

### 1. 测试介绍

旋转屏的性能测试为多数手机品牌客户会关注的一个点，主要已布局重绘的时间来衡量旋转屏的性能，从手机旋转90开始计时到界面出现旋转结束。我接触到过的几个客户如Acer、Tecno标准都在1s以内，通常和平台也会相关，见过的最优的效果为Meizu的1519项目，时间在0.5s左右，旋转非常快速且流畅。

### 2. GSensor数据排查

此项问题的分析，我们从下之上来一步步排查，首先要明确旋转屏的实现是通过GSensor的三轴数据变化来实现的，所以第一步先排查GSensor的数据上报速度是否存在瓶颈。
需要可以root的机器一台，如下操作

```Bash
adb shell
cd dev/input
ls
```
可以看到有几个event
![show event](https://chendongqi.github.io/blog/img/2017-02-17-rotation_performance/show_event.png)
用getevent命令再次查看

```Bash
getevent
```
![get event](https://chendongqi.github.io/blog/img/2017-02-17-rotation_performance/get_event.png)
可以看到event4中为GSensor相关的事件，然后输入

```Bash
getevent event4
```
可以看到如下信息
![gsensor event](https://chendongqi.github.io/blog/img/2017-02-17-rotation_performance/gsensor_event.png)
一直有数据在更新，说明GSensor的数据上报是有的，且刷新非常快。这块是属于正常。

### 3. frameworks层分析

* 屏幕旋转的原理：

屏幕方向的变换是由Sensor决定的。当Sensor变化时，会调用到 
   frameworks/base//services/core/java/com/android/server/policy/WindowOrientationListener.java文件中的onSensorChanged()函数，然后会根据orientation，tiltAngle, mRotation三个变量去做相应判断，如果条件符合，最后会调用onProposedRotationChanged()函数来通知转屏，这个就会走到PhoneWindowManager中。

* 转屏时间的评估：

可以在log中搜索“Screen frozen for”，表示转屏时冻屏的时间，可以作为一个优化后的参考依据，横竖屏切换的时间是不同的
![screen frozen time](https://chendongqi.github.io/blog/img/2017-02-17-rotation_performance/screen_frozen_time.png)

* 转屏优化：

**第一个优化方案是修改旋转屏的触发角度**，此方案来自MTK Online，[ FAQ03882 Android4.0 ICS 怎么修改屏幕自动旋转的角度临界值](https://onlinesso.mediatek.com/Pages/FAQ.aspx?List=SW&FAQID=FAQ03882)
>【默认的角度】
ICS对原本GB坐标矩阵换算的计算方式做了简化，简单地将360度平均划分为四个方向区间:
区间0：[315,45)   // 上
区间1：[45,135)   // 右
区间2：[135,225) // 左
区间3：[225,315) // 下
区间的转换通过一个很简单的语句：
windowOrientationListener.java的onSensorChanged函数中，
int nearestRotation = (orientationAngle + 45) / 90；
但是这并不意味着在45度时屏幕就会旋转，为了防止在临界值附近来回摆动，ICS中还定义了一个gap区域；
顾名思义，在这个gap区域内，不会发生旋转；gap区域在两个相邻方向区间的中间；
windowOrientationListener.java中有个全局变量定义了gap区的大小，
ADJACENT_ORIENTATION_ANGLE_GAP，默认值是45，
通过函数isOrientationAngleAcceptable来控制gap区的作用；
所以真正的默认旋转角度临界值是45 + GAP/2 = 45 + 45/2 = 67.5度
修改默认角度临界值的例子
1.如果要修改成在超过45度的角度旋转，并且横竖屏切换角度一致，就比较简单
比如要在60度旋转，那么只需要计算GAP = (60-45)*2 = 30
将ADJACENT_ORIENTATION_ANGLE_GAP改成30
2.其他情况根据需求会比较复杂，需要重新划分方向区间
需求不同，很难一概而论，但是重新划分区间是一定要做的，
比如需求是垂直转水平临界值是60度，水平转垂直是45度，
那么需要做两个修改：
（1）这个需求实际上已经定义了天然的gap区域，我们不需要ICS本身提供的gap机制，
    将isOrientationAngleAcceptable直接返回true；
（2）修改方向区间，将

```java
    int nearestRotation = (orientationAngle + 45) / 90
```
    改成：
    
```java
    switch (mOrientationListener.mCurrentRotation)
    {
         // 原来是竖直，旋转60度生效
         case 0: 
         case 2:
         {
              if( 60 <= orientationAngle && orientationAngle < 135)
                   nearestRotation = 1;
              else if( 135 <= orientationAngle && orientationAngle < 225)
                   nearestRotation = 2;
              else if( 225 <= orientationAngle && orientationAngle < 300)
                   nearestRotation = 3;
              else
                   nearestRotation = 0;
                   break;
        }
     // 原来是水平，旋转45度生效
         case 1: 
         case 3:
         {
              if( 45 <= orientationAngle && orientationAngle < 135)
                   nearestRotation = 1;
              else if( 135 <= orientationAngle && orientationAngle< 225)
                   nearestRotation = 2;
              else if( 225 <= orientationAngle && orientationAngle< 315)
                   nearestRotation = 3;
              else
                   nearestRotation = 0;
                   break;
        }
        default:
               break;
    }
```
MTK的方案也就是重新定义了四个象限临界角度，在Android6.0版本中依然可以应用，在我的项目中的修改patch如下:

```java
From aa15fcf0fbee9b9ffcdf01107e2934888ba9e568 Mon Sep 17 00:00:00 2001
From: 101002192 <chendongqi@huaqin.com>
Date: Thu, 15 Sep 2016 16:01:14 +0800
Subject: [PATCH] [LV1][BUG][COMMON][UX PERFORMANCE][JIRA][LV1-459] 提高旋转屏幕的灵敏性以加快响应速度
Defect:
NA
Root cause:
NA
How to fix:
调整转屏的象限角度
Impacted group:
SCREEN ROTATION
 On branch ZAL1066_LG
 Your branch is up-to-date with 'gerrit/ZAL1066_LG'.
 Changes to be committed:
	modified:   services/core/java/com/android/server/policy/WindowOrientationListener.java
Change-Id: Id5ec71fb4987a8fa09ddd48d7f87eac28f4d181c
---

diff --git a/services/core/java/com/android/server/policy/WindowOrientationListener.java b/services/core/java/com/android/server/policy/WindowOrientationListener.java
index 409b321..03cc6e7 100644
--- a/services/core/java/com/android/server/policy/WindowOrientationListener.java
+++ b/services/core/java/com/android/server/policy/WindowOrientationListener.java
@@ -507,6 +507,7 @@
             int proposedRotation;
             int oldProposedRotation;
 
+
             synchronized (mLock) {
                 // The vector given in the SensorEvent points straight up (towards the sky) under
                 // ideal conditions (the phone is not accelerating).  I'll call this up vector
@@ -622,9 +623,46 @@
                                 // atan2 returns [-180, 180]; normalize to [0, 360]
                                 orientationAngle += 360;
                             }
-
+                            
+                            // modify by chendongqi for LG Performance--start @{
                             // Find the nearest rotation.
-                            int nearestRotation = (orientationAngle + 45) / 90;
+                            //int nearestRotation = (orientationAngle + 45) / 90;
+                            
+                            int nearestRotation = 0;
+                            switch (mCurrentRotation)
+                            {
+                                 // original is vertical, rotate 45 to be horizontal
+                                 case 0: 
+                                 case 2:
+                                 {
+                                          if( 45 <= orientationAngle && orientationAngle < 135)
+                                               nearestRotation = 1;
+                                          else if( 135 <= orientationAngle && orientationAngle < 225)
+                                               nearestRotation = 2;
+                                          else if( 225 <= orientationAngle && orientationAngle < 315)
+                                               nearestRotation = 3;
+                                          else
+                                               nearestRotation = 0;
+                                          break;
+                                 }
+                                 // original is horizontal, rotate 30 to be vertical
+                                 case 1: 
+                                 case 3:
+                                 {
+                                          if( 30 <= orientationAngle && orientationAngle < 135)
+                                               nearestRotation = 1;
+                                          else if( 135 <= orientationAngle && orientationAngle< 225)
+                                               nearestRotation = 2;
+                                          else if( 225 <= orientationAngle && orientationAngle< 330)
+                                               nearestRotation = 3;
+                                          else
+                                               nearestRotation = 0;
+                                          break;
+                                  }
+                                  default:
+                                          break;
+                             }
+                             // modify by chendongqi for LG Performance--end @}
                             if (nearestRotation == 4) {
                                 nearestRotation = 0;
                             }
@@ -714,6 +752,7 @@
          * for hysteresis.
          */
         private boolean isOrientationAngleAcceptableLocked(int rotation, int orientationAngle) {
+            /*delete by chendongqi for LG performance --start @{
             // If there is no current rotation, then there is no gap.
             // The gap is used only to introduce hysteresis among advertised orientation
             // changes to avoid flapping.
@@ -757,6 +796,8 @@
                     }
                 }
             }
+            * delete by chendongqi --end @}
+            */
             return true;
         }
```
**第二个方案是去除旋转动画**:在WindowManagerService中修改static final boolean CUSTOM_SCREEN_ROTATION = true为false，慎用，自测UX效果较差，但是速度确实快。

**第三个方案是对Activity的生命周期优化**:
切换到横屏时Activity的生命周期会打印如下：

```java
onSaveInstanceState
onPause
onStop
onDestroy
onCreate
onStart
onRestoreInstanceState
onResume
```
切换到竖屏时的生命周期打印如下：

```java
onSaveInstanceState
onPause
onStop
onDestroyonCreate
onStart
onRestoreInstanceState
onResume
onSaveInstanceState
onPause
onStop
onDestroy
onCreate
onStart
onRestoreInstanceState
onResume
```
可以发现切换到竖屏时走了两遍，所以竖屏的时间会比横屏慢
修改AndroidManifest.xml，把该Activity添加 android:configChanges="orientation"，切横屏，只销毁一次。再切回竖屏，发现不会再打印相同信息，但多打印了一行onConfigChanged：

```java
onSaveInstanceState
onPause
onStop
onDestroy
onCreate
onStart
onRestoreInstanceState
onResume
onConfigurationChange
```
更改 android:configChanges="orientation" 改成 android:configChanges="orientation|keyboardHidden"，切横屏，就只打印onConfigChanged；切回竖屏打印两遍onConfigurationChanged。

**Activity的优化小结**：

不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次

设置Activity的android:configChanges="orientation"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次

设置Activity的android:configChanges="orientation|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

### 4. AW875 ACER上的转屏问题经验（ALPS02577316）

一段mtk分析的log:

```java
//Sensor 報轉屏
01-02 13:52:56.052618 813 830 V WindowManager: Application requested orientation -1, got rotation 3 which has compatible metrics
//WMS freeze screen 等 AP 畫完
01-02 13:52:56.175087 813 830 V WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{7937d0e u0 NavigationBar}, surface Surface(name=NavigationBar)
01-02 13:52:56.177301 813 830 V WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{4eda9f8 u0 StatusBar}, surface Surface(name=StatusBar)
01-02 13:52:56.181325 813 830 V WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{67476aa u0 com.android.mms/com.android.mms.ui.ConversationList}, surface Surface 1388c04
01-02 13:52:56.263284 813 834 V WindowManager: Orientation not waiting for draw in Window{67476aa u0 com.android.mms/com.android.mms.ui.ConversationList}, surface Surface 1388c04
//systenUI 畫了 400多 ms
01-02 13:52:56.625260 813 834 V WindowManager: Orientation not waiting for draw in Window{4eda9f8 u0 StatusBar}, surface Surface(name=StatusBar)
01-02 13:52:56.673842 813 834 V WindowManager: Orientation not waiting for draw in Window{7937d0e u0 NavigationBar}, surface Surface(name=NavigationBar)
01-02 13:52:56.673352 813 834 V WindowStateAnimator: Showing WindowStateAnimator{7bd760d NavigationBar} during animation: policyVis=true attHidden=false tok.hiddenRequested=false tok.hidden=false animating=false tok animating=false
01-02 13:52:56.673477 813 834 V WindowStateAnimator: performShowLocked: mDrawState=HAS_DRAWN in WindowStateAnimator{7bd760d NavigationBar}
01-02 13:52:56.687956 813 834 V WindowManager: With display frozen, orientationChangeComplete=true
//開始轉屏
01-02 13:52:56.688076 813 834 I WindowManager: Screen frozen for +633ms due to Window{7937d0e u0 NavigationBar}
```
