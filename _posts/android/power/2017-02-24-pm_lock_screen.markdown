---
layout:      post
title:      "Android电源管理之Power键锁屏流程"
subtitle:   "介绍按power之后进入锁屏的代码流程，另外介绍了如何设计灭屏而不锁屏的方案"
navcolor:   "invert"
date:       2017-02-24
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - PowerManager
    - LockScreen
---

### 适用平台

Android Version： 6.0    
Platform： MTK6580/MTK6735/MTK6753    

### 1. Power键锁屏流程

&emsp;&emsp;本章讨论的锁屏流程是指：在亮屏情况下，按power键之后进入系统灭屏休眠过程中绘制锁屏界面的流程。这里需要说明一点，锁屏界面的绘制是在灭屏时完成的，而并非是在下一次亮屏时，这样设计也很容易理解，以便于在亮屏第一时间就能显示锁屏界面。    
![lockscreen_flow](https://chendongqi.github.io/blog/img/2017-02-24-pm_lockscreen/lockscreen_flow.png)    
&emsp;&emsp;整个锁屏流程（自PowerManagerService开始，按Power键到PowerManagerService中的休眠流程可以参见站内文章[ Android电源管理之关机流程 ](https://chendongqi.github.io/blog/2017/02/21/pm_shutdown_flow/)）如上图所见流程图。    
1. 通过Binder Call调用PMS的goToSleep方法开始进入系统休眠流程    
2. 调用PMS的goToSleepInternal方法    
```java
    private void goToSleepInternal(long eventTime, int reason, int flags, int uid) {
        synchronized (mLock) {
            if (mProximityPositive && reason == PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON) {
                Slog.d(TAG, "Proximity positive sleep and force wakeup by power button");
                // 如果pSensor是处于positive状态，则power键将用来唤醒系统
                mDirty |= DIRTY_WAKEFULNESS;
                mWakefulness = WAKEFULNESS_ASLEEP;
                updatePowerStateLocked();
                return;
            }

            if (goToSleepNoUpdateLocked(eventTime, reason, flags, uid)) {
                // 步骤2
                updatePowerStateLocked();
            }
        }
    }
```    
&emsp;&emsp;在goToSleepInternal方法中如果接近传感器是属于near状态，那么power键将不用来休眠系统而是唤醒。否则将调用PMS的goToSleepNoUpdateLocked方法    
3. 调用PMS的goToSleepNoUpdateLocked：    
```java
    switch (reason) {
        case PowerManager.GO_TO_SLEEP_REASON_DEVICE_ADMIN:
            Slog.i(TAG, "Going to sleep due to device administration policy "
                    + "(uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_TIMEOUT:
            Slog.i(TAG, "Going to sleep due to screen timeout (uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_LID_SWITCH:
            Slog.i(TAG, "Going to sleep due to lid switch (uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON:
            Slog.i(TAG, "Going to sleep due to power button (uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_SLEEP_BUTTON:
            Slog.i(TAG, "Going to sleep due to sleep button (uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_HDMI:
            Slog.i(TAG, "Going to sleep due to HDMI standby (uid " + uid +")...");
            break;
        case PowerManager.GO_TO_SLEEP_REASON_PROXIMITY:
            Slog.i(TAG, "Going to sleep due to proximity (uid " + uid +")...");
            break;
        default:
            Slog.i(TAG, "Going to sleep by application request (uid " + uid +")...");
            reason = PowerManager.GO_TO_SLEEP_REASON_APPLICATION;
            break;
    }
    mLastSleepTime = eventTime;
    mSandmanSummoned = true;
    setWakefulnessLocked(WAKEFULNESS_DOZING, reason);
```    
&emsp;&emsp;在这个方法里有一段处理休眠reason的代码，reason除了定义的几种之外都会被处理成GO_TO_SLEEP_REASON_APPLICATION。然后就是调用setWakefulnessLocked。        
4. 调用PMS的setWakefulnessLocked          
5. 调用Notifer的onWakefulnessChangeStarted方法    
6. 调用Notifier的handleEarlyInteractiveChange方法    
7. 通过接口WindowManagerPolicy来调用到PhoneWindowManager的startedGoingToSleep方法    
```java
    // Called on the PowerManager's Notifier thread.
    @Override
    public void startedGoingToSleep(int why) {
        if (DEBUG_WAKEUP) Slog.i(TAG, "Started going to sleep... (why=" + why + ")");
        if (mKeyguardDelegate != null) {
            mKeyguardDelegate.onStartedGoingToSleep(why);// 开始进入到Keyguard的流程中
        }
    }
```    
8. 同步的，调用PMS的updatePowerStateLocked方法进而走到finishWakefulnessChangeIfNeededLocked方法      
9. 调用Notifier的onWakefulnessChangeFinished方法      
10. 调用Notifer的handleLateInteractiveChange方法      
11. 通过接口WindowManagerPolicy来调用到PhoneWindowManager的finishedGoingToSleep方法      
```java
    // Called on the PowerManager's Notifier thread.
    @Override
    public void finishedGoingToSleep(int why) {
        EventLog.writeEvent(70000, 0);
        if (DEBUG_WAKEUP) Slog.i(TAG, "Finished going to sleep... (why=" + why + ")");
        MetricsLogger.histogram(mContext, "screen_timeout", mLockScreenTimeout / 1000);

        // We must get this work done here because the power manager will drop
        // the wake lock and let the system suspend once this function returns.
        synchronized (mLock) {
            mAwake = false;
            // 处理灭屏时的窗口显示，包括更新唤醒手势、更新显示方向、更新屏幕超时时间
            updateWakeGestureListenerLp();
            updateOrientationListenerLp();
            updateLockScreenTimeout();
        }
        if (mKeyguardDelegate != null) {
            // 回调锁屏接口来完成锁屏
            mKeyguardDelegate.onFinishedGoingToSleep(why);
        }
    }
```     
&emsp;&emsp;本篇内容对流程只梳理到从PMS到PhoneWindowManager部分，具体KeyguardDelgate是如何进行锁屏绘制的不做进一步介绍。

### 2. 灭屏锁屏分离

&emsp;&emsp;以上流程分析之后，我们可以思考一个需求，如何去实现在灭屏时不绘制锁屏界面呢？以皮套为例，如何在皮套合上时只灭屏休眠和不绘制锁屏，那样在打开皮套时就不会先显示锁屏界面然后解锁这样一个过程了。    
&emsp;&emsp;1) 方法一：在调用PMS的goToSleep方法时会传入一个int类型的reason值来标示休眠的原因。这个reason会在整个流程中传递下来，那么思路一就可以在皮套中调用goToSleep时传入一个自定义的reason值，在PhoneWindowManager的goingToSleep中判断why的值，如果是皮套合盖的原因，则不调用锁屏入口即可。具体实现如下：    
![first_strategy1](https://chendongqi.github.io/blog/img/2017-02-24-pm_lockscreen/first_strategy1.png)    
![first_strategy2](https://chendongqi.github.io/blog/img/2017-02-24-pm_lockscreen/first_strategy2.png)    
&emsp;&emsp;另外需要注意的是如上一章的步骤3中所述，传入的reason在PMS的goToSleepNoUpdateLocked方法中会被处理一次，所以在HolsterService中传入的值20也会被处理成GO_TO_SLEEP_REASON_APPLICATION，所以需要加一步处理。    
![first_strategy3](https://chendongqi.github.io/blog/img/2017-02-24-pm_lockscreen/first_strategy3.png)     
&emsp;&emsp;在处理reason时加入自己定义的值即可。当然每个应用的reason值是可以自定义的。这样做完之后合上皮套就只灭屏休眠而不会去绘制锁屏了。    
&emsp;&emsp;2) 方法2：这个方法未在代码中调试，只是提供一个思路。     
&emsp;&emsp;&emsp;&emsp;1) 在Settings中加入一个数据库字段，初始化为0，合上皮套时写入1    
&emsp;&emsp;&emsp;&emsp;2) 在PhoneWindowManager的goingToSleep读取这个字段，在判断调用锁屏入口时如果字段的值为1，则不调用锁屏；在皮套开盖时将这个字段写入0    
&emsp;&emsp;&emsp;&emsp;3) 这个方法还有个简化的方案，在PhoneWindowManager中有个变量mLidState用来记录霍尔器件状态。可以在判断调用锁屏条件时加入这个变量的判断，如果hall为通路则不锁屏，否则锁屏。



