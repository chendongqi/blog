---
layout:      post
title:      "Android开机系列一"
subtitle:   "开机情景下Launcher的启动过程"
navcolor:   "invert"
date:       2017-09-01
author:     "Cheson"
catalog: true
tags:
    - Android
    - frameworks
    - ActivityManager
    - Launcher
    - 开机
---

### Android version

Android version: 4.4
Platform: freescale imx6

### Background

&emsp;&emsp;首先介绍下写这篇文章的起因。之前写过一篇[Android电源管理之开机流程](https://chendongqi.github.io/blog/2017/02/20/pm_boot_flow/)，拆解了从bootloader到android启动的各个阶段，是一篇概要性的综述，又带了点自己的项目经验和理解在里面。后来发现介绍android启动的文章网上其实很多，有深有浅，但是深的其实也不会涉及到很细的细节，不会分析到一个函数调过去还需要走多少步才能达到它预期的结果。时过境迁，回过头看自己这篇开机流程里其实大多数章节也是这样，流程性的东西比较多，当时以为自己懂的东西真的遇到问题时才发现只是懂了皮毛。  
&emsp;&emsp;最近在项目中遇到一个问题，在开机过程中开机动画结束之后会有一段长时间的黑屏，之后launcher才会显示出来。刚开始的时候想去看看别人的博客了解下这段时间系统都在做什么，很遗憾什么都没找到。自己用bootchart结合log看了下，发现其实launcher的进程其实很早就起来了，然后转而去查了launcher启动过程，妄图找到蛛丝马迹。很遗憾的是写开机launcher启动过程的博客都止步在了startHomeActivity方法，然后通过AMS的startActivity启动了launcher，但是真的这样就结束了吗？这个时间点难道就能看到launcher秀出来了？在一开始的分析中就给了我一个否定的答案，于是抱着这样的疑问，催生了这篇文章，作为全网最详细的launcher启动流程介绍。  
&emsp;&emsp;在正式介绍内容之前再说明几点，这次调试的平台是freescale的android4.4代码（之前做手机，紧随google的更新脚步，现在做车机，老司机要稳，所以版本很老）。本文中设计到的framework代码较多，自己也打了很多log，会用代码和log结合的方式来理清开机过程中launcher的启动流程。对activity启动流程比较熟悉的同学可能看起来会轻松点，因为launcher启动本质上也是activity启动流程，因为是在开机阶段，所以有些细节上会有特殊性。主要会涉及到ActivityManagerService、ActivityStack、ActivityStackSupervisor、ActivityThread、WindowManagerService这几个类的交互。  

### First

&emsp;&emsp;故事从SystemServer讲起吧，看过[Android电源管理之开机流程](https://chendongqi.github.io/blog/2017/02/20/pm_boot_flow/)肯定对SystemServer在开机过程中所做的事有一定的了解了，比较重要的就是启动各个系统核心服务，在android4.4上做这件事的函数叫做`initAndLoop`，换汤不换药，在这里初始化了所有系统服务，然后托管给ServiceManager，在之后还做了一件很重要的事，通知各个Service系统准备好了，这里要关注的就是AMS的systemReady。  
```Java
ActivityManagerService.self().systemReady(new Runnable() {
    public void run() {
    	Slog.i(TAG, "Making services ready");//通常我会看这句log来判断systemServer启动好了
      	...
        if (!headless) {//SystemUI起来是很早的
            startSystemUi(contextF);
        }
        try {// 通知MountService系统准备好了
            if (mountServiceF != null) mountServiceF.systemReady();
        } catch (Throwable e) {
            reportWtf("making Mount Service ready", e);
        }
        // 后面全是类似的通知各个服务
        ...
    }
}）；
```
&emsp;&emsp;在调用AMS的systemReady之前，已经优先调用了PowerManagerService、PackageManagerService、WindowManagerService、DisplayService等几个跟硬件更相关的服务的systemReady，AMS是一个承上启下的存在，它的systemReady意味着跟用户更相关的app进程可以陆续起来了，而且后续的通知都是基于AMS的systemReady中回调过来，通过这里的Runnable的run方法去执行的。让我们来看下AMS的systemReady方法  
```Java
public void systemReady(final Runnable goingCallback) {
	synchronized(this) {
		Log.d("chendongqi-boot", "AMS-systemReady start");// 我在这里加了句log
		if (mSystemReady) {// 如果这里已经ready了，那直接可以回调通知
            if (goingCallback != null) goingCallback.run();
            return;
        }
        if (!mDidUpdate) {// 这里有一长段系统升级时候的逻辑
        ...
        }
        ...
        mSystemReady = true;// 这里标志位就改成true了
        ...
        // 又会输出关键性log了，还会打到events log中去
        Slog.i(TAG, "System now ready");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY,
            SystemClock.uptimeMillis());
        ...
        mStackSupervisor.resumeTopActivitiesLocked();// 关键流程1-1
        ...
        Log.d("chendongqi-boot", "AMS-systemReady end");// 又是我加的log
    }
}
```
&emsp;&emsp;跟启动launcher相关的就是这一句`mStackSupervisor.resumeTopActivitiesLocked()`，这里顺便插入一句自己对ActivityStack和ActivityStackSupervisor的理解，看Activity相关情境的流程时都会涉及到这两个类，ActivityStack是Activity的容器，以栈的方式来关系系统里的所有Activity，提供了一些操作的方法，而ActivityStackSupervisor是ActivityStack的监视器，绝对多数对Activity的业务操作逻辑都是从这边发起的。继续来看下一步的流程跳到了ActivityStackSupervisor里  
```Java
boolean resumeTopActivitiesLocked() {//ActivityStackSupervisor
    return resumeTopActivitiesLocked(null, null, null);
}
```
&emsp;&emsp;调用一个重载的三参数方法

```Java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
        Bundle targetOptions) {
    if (targetStack == null) {
        targetStack = getFocusedStack();
    }
    boolean result = false;
    for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
        final ActivityStack stack = mStacks.get(stackNdx);
        if (isFrontStack(stack)) {
            if (stack == targetStack) {//走这里，去启动top stack的top activity
                result = stack.resumeTopActivityLocked(target, targetOptions);
            } else {
                stack.resumeTopActivityLocked(null);
            }
        }
    }
    return result;
}
```
&emsp;&emsp;然后调用走到了ActivityStack中的resumeTopActivityLocked  
```Java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
	android.util.Log.d("chendongqi-boot", "AS-resumeTopActivityLocked start");
	...
	// Find the first activity that is not finishing.
    ActivityRecord next = topRunningActivityLocked(null);
    ...
    if (next == null) {
        // There are no more activities!  Let's just start up the
        // Launcher...
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: No more activities go home");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        // 没有next了，需要去启动launcher
        return mStackSupervisor.resumeHomeActivity(prev);//关键流程1-2
    }
    ...
}
```
&emsp;&emsp;继续ActivityStackSupervisor的resumeHomeActivity方法中  
```Java
boolean resumeHomeActivity(ActivityRecord prev) {
    moveHomeToTop();//将launcher所在stack移到top的位置
    if (prev != null) {
        prev.task.mOnTopOfHome = false;
    }
    ActivityRecord r = mHomeStack.topRunningActivityLocked(null);//取到launcher，如果有的话
    if (r != null && r.isHomeActivity()) {//不是开机情况下回到home就会走这
        mService.setFocusedActivityLocked(r);
        return resumeTopActivitiesLocked(mHomeStack, prev, null);
    }
    return mService.startHomeActivityLocked(mCurrentUser);//开机时走这个，上面都是浮云。关键流程1-3
}
```
&emsp;&emsp;有回到了AMS，调用AMS的startHomeActivityLocked  
```Java
boolean startHomeActivityLocked(int userId) {
	Log.d("chendongqi-boot", "AMS-startHomeActivityLocked");
	...
	Intent intent = getHomeIntent();//获取launcher的Intent，主要是CATEGORY_HOME属性
	...
	ActivityInfo aInfo =
        resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);//根据intent解析出ActivityInfo
    if (aInfo != null) {
    intent.setComponent(new ComponentName(
            aInfo.applicationInfo.packageName, aInfo.name));
    // Don't do this if the home app is currently being
    // instrumented.
    aInfo = new ActivityInfo(aInfo);
    aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
    ProcessRecord app = getProcessRecordLocked(aInfo.processName,
           aInfo.applicationInfo.uid, true);
    if (app == null || app.instrumentationClass == null) {
        intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
        mStackSupervisor.startHomeActivity(intent, aInfo);//调到ActivityStackSupervisor，关键流程1-4
    }
}
```

```Java
void startHomeActivity(Intent intent, ActivityInfo aInfo) {
    Log.d("chendongqi-boot", "ASS-startHomeActivity start");
    moveHomeToTop();
    //开始走到统一的startActivityLocked流程中，关键流程1-5
    startActivityLocked(null, intent, null, aInfo, null, null, 0, 0, 0, null, 0,
                null, false, null);
}
```
&emsp;&emsp;到这里为止流程就走到了统一的startActivityLocked了，也就是网上基本上所有介绍Launcher启动博客的终点，但是这里并不是真正的终点。不像系统开机开机之后，调用到startActivity基本上就等于Activity显示出来了，耗时在几百ms到几秒之间吧。开机流程中调用到startActivityLocked来启动launcher离真正显示出来还是有很大一段距离的，这也是后面的重点，来分析这段时间都在做什么。  
&emsp;&emsp;贴下这个阶段打印出的log  
```xml
08-29 16:20:48.310 D/chendongqi-boot( 1687): WMS-systemReady
08-29 16:20:48.710 D/chendongqi-boot( 1687): AMS-systemReady start
08-29 16:20:50.420 D/chendongqi-boot( 1687): AS-resumeTopActivityLocked start
08-29 16:20:50.420 D/chendongqi-boot( 1687): AMS-startHomeActivityLocked
08-29 16:20:52.130 D/chendongqi-boot( 1687): ASS-startHomeActivity start
```

### second

&emsp;&emsp;第二个阶段接着上面关键流程1-5，调用到ActivityStackSupervisor的startActivityLocked  
```Java
ASS.startActivityLocked
ASS.startActivityUncheckedLocked
ASS.resumeTopActivitiesLocked()
ASS.resumeTopActivitiesLocked(null, null, null)
AS.resumeTopActivityLocked
```
&emsp;&emsp;这里经过一系列的标准流程调用又进入到了ActivityStack的resumeTopActivityLocked，和关键流程1-2不同的是，这次ActivityStack的参数不一样了，也就是收next不是空了，存在launcher的Stack，不用从头开始去启动。  
```Java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    android.util.Log.d("chendongqi-boot", "AS-resumeTopActivityLocked start");
    ...      
    // 会先去pause所有栈，让top stack可以resume，这里launcher是第一个，根本没有其他栈
    // We need to start pausing the current activity so the top one
    // can be resumed...
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving);
    ...
    if (next.app != null && next.app.thread != null) {//一开始launcher的thread都没创建，这个判断都不走
    ...
    } else {
    ...
    }
    ...
    mStackSupervisor.startSpecificActivityLocked(next, true, false);//关键流程1-6
    ...
    completeResumeLocked(next);// 关键流程3-1
    ...
    
```
&emsp;&emsp;流程走到ActivityStackSupervisor的startSpecificActivityLocked，这里可以理解成会走3个分支，第一个是要去把承载Activity真实任务的Thread去创建起来，第二个是继续AM子系统对Activity启动流程控制的逻辑，而第三个是一个控制Activity显示与否的机制。这里就插入先介绍下进程启动的大致流程。  

### third  

&emsp;&emsp;这一节来介绍下一个进程的启动过程，接续上一节中的关键流程1-6，在ActivityStackSupervisor的startSpecificActivityLocked中最后会调用AMS的startProcessLocked  
```Java
void startSpecificActivityLocked(ActivityRecord r,
    boolean andResume, boolean checkConfig) {
    ...
    // 关键流程2-1
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
       "activity", r.intent.getComponent(), false, false, true);
}
```
&emsp;&emsp;然后代码走到AMS中的startProcessLocked，看方法名就知道是启动进程的，这里要留意的是在AMS中有两个startProcessLocked方法，一个是9参数的，另一个是3参数的，这里调用的是9个参数的。继续看代码  
```Java
final ProcessRecord startProcessLocked(String processName,
    ApplicationInfo info, boolean knownToBeDead, int intentFlags,
    String hostingType, ComponentName hostingName, boolean allowWhileBooting,
    boolean isolated, boolean keepIfLarge) {
    ...
    if (app == null) {//开机的时候launcher进程当然是null，这边new一个出来
        app = newProcessRecordLocked(info, processName, isolated);
        ...
    } else {
      ...
    }
    ...
    // 关键流程2-2
    startProcessLocked(app, hostingType, hostingNameStr);//调用3参的方法
    ...
```
```Java
private final void startProcessLocked(ProcessRecord app,
    String hostingType, String hostingNameStr) {
    ...
    // 关键流程2-3
    // 通过zygote去fork出app的进程
    Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
            app.processName, uid, uid, gids, debugFlags, mountExternal,
            app.info.targetSdkVersion, app.info.seinfo, null);
    ...
}
```
&emsp;&emsp;Zygote里启动进程的流程这里就跳过了，启动起来的进程，承载它的类就是ActivityThread。这里是标准的Java类的流程，入口就在main方法  
```Java
public static void main(String[] args) {
...
	Looper.prepareMainLooper();//消息队列

    ActivityThread thread = new ActivityThread();
    thread.attach(false);//关键流程2-4，注意这里传入的参数是false

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();//获取handler
    }

    AsyncTask.init();

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    Looper.loop();//消息循环
    ...
}
```
```Java
private void attach(boolean system) {
	...
	if (!system) {//上面传的参数是false
		...
		RuntimeInit.setApplicationObject(mAppThread.asBinder());
		IActivityManager mgr = ActivityManagerNative.getDefault();
		try {
			//关键流程2-5，调用进入AMS
			mgr.attachApplication(mAppThread);
		} catch (RemoteException ex) {
			// Ignore
		}
	} else {
	...
	}
	...
}
```
&emsp;&emsp;然后又是去到了AMS中，调用attachApplication，这里仅是转而调用了attachApplicationLocked  
```Java
private final boolean attachApplicationLocked(IApplicationThread thread,
    int pid) {
    ...
    // 关键流程2-6，又回到AT中去了
    thread.bindApplication(processName, appInfo, providers,
            app.instrumentationClass, profileFile, profileFd, profileAutoStop,
            app.instrumentationArguments, app.instrumentationWatcher,
            app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
            isRestrictedBackupMode || !normalMode, app.persistent,
            new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
            mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);//更新process的Lru信息
    ...
```
&emsp;&emsp;回到ActivityThread中，调用bindApplication  
```Java
public final void bindApplication(String processName,
        ApplicationInfo appInfo, List<ProviderInfo> providers,
        ComponentName instrumentationName, String profileFile,
        ParcelFileDescriptor profileFd, boolean autoStopProfiler,
        Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
        Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
        Bundle coreSettings) {
        // 关键流程2-7
        setCoreSettings(coreSettings);
        ...
        // 关键流程2-8
        sendMessage(H.BIND_APPLICATION, data);
        ...
}
```
&emsp;&emsp;ActivityThread里面调度任务采用的也是消息队列的机制，在thread启动开始会有几个流程必走的，第一个就是调用setCoreSettings方法，这里面发送了SET_CORE_SETTINGS的message，然后在handleMessage里会去调用handleSetCoreSettings方法来做设置，参数的来源是从AMS调bindApplication时候传入的mCoreSettingsObserver.getCoreSettingsLocked()，具体的细节我就没打出来看了，无非是thread的一些信息。  
&emsp;&emsp;然后继续发送BIND_APPLICATION消息，同样在handleMessage里收到之后消息之后调用handleBindApplication，这个方法做的事情就很多了，重要的包括启动应用的application组件，初始化context等。如果有调试systrace经验的话，可以看到进程开始最前面就是bindApplication这一段，主要工作呢就是在handleBindApplication，而处理activity的那一段还要在后面一段。  
&emsp;&emsp;ActivityThread起来的这一段流程抠出来如下，这一部分就这样。  
```Java
ASS.startSpecificActivityLocked
AMS.startProcessLocked//9个参数
AMS.newProcessRecordLocked
AMS.startProcessLocked//3个参数
Process.start//通过Zygote来创建进程
AT.main
AT.attach
AMS.attachApplication
AMS.attachApplicationLocked
AT.bindApplication
AT.handleSetCoreSettings
AT.handleBindApplication
```

### forth

&emsp;&emsp;ActivityThread启动的分支流程结束之后，继续来看Activity起来的主线流程。接第二节中的流程1-6，到了ASS.startSpecificActivityLocked中，然后继续ASS的realStartActivityLocked  
```Java
final boolean realStartActivityLocked(ActivityRecord r,
    ProcessRecord app, boolean andResume, boolean checkConfig)
    throws RemoteException {
    ...
    // 关键流程1-7
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
            System.identityHashCode(r), r.info,
            new Configuration(mService.mConfiguration), r.compat,
            app.repProcState, r.icicle, results, newIntents, !andResume,
            mService.isNextTransitionForward(), profileFile, profileFd,
            profileAutoStop);
    ...
}
```
&emsp;&emsp;又去到了ActivityThread中，也就是发送了LAUNCH_ACTIVITY消息，然后在handleMessage里调用handleLaunchActivity去处理，这里是分析开机时launcher出来速度的其中给一个关键  
```Java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
	Log.d("chendongqi-boot", "AT-handleLaunchActivity start profiledFd = " + r.profileFd);
	unscheduleGcIdler();//移除掉空闲时的GC任务
	Log.d("chendongqi-boot", "AT-handleLaunchActivity 00000");

    if (r.profileFd != null) {
        mProfiler.setProfiler(r.profileFile, r.profileFd);
        mProfiler.startProfiling();
        mProfiler.autoStopProfiler = r.autoStopProfiler;
    }
    Log.d("chendongqi-boot", "AT-handleLaunchActivity 11111");

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);
    Log.d("chendongqi-boot", "AT-handleLaunchActivity 22222");

    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);
    // 关键流程1-8，这里是耗时之一
    // Activity的onCreate入口
    Activity a = performLaunchActivity(r, customIntent);
    Log.d("chendongqi-boot", "AT-handleLaunchActivity 33333");
    
    if (a != null) {
    	r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        Log.d("chendongqi-boot", "AT-handleLaunchActivity for " + a.getPackageName());
        // 关键流程1-9，这里是耗时之二
        // Activity的onResume入口
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);
        Log.d("chendongqi-boot", "AT-handleLaunchActivity 44444");
        ...
    }
    ...
}
```
&emsp;&emsp;打印1-8流程中的耗时情况  
```Java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
	...
	try {
      ...
      if (activity != null) {
      	Log.d("chendongqi-boot", "AT-performLaunchActivity attach begin");
      	activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config);
        Log.d("chendongqi-boot", "AT-performLaunchActivity attach end");
        ...
        Log.d("chendongqi-boot", "AT-performLaunchActivity callActivityOnCreate begin");
        // 最终调用Activity的onCreate
        mInstrumentation.callActivityOnCreate(activity, r.state);
        Log.d("chendongqi-boot", "AT-performLaunchActivity callActivityOnCreate end");
        ...
	}
	...
}
```
&emsp;&emsp;打印流程1-9中的耗时情况  
```Java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward,
    boolean reallyResume) {
    Log.d("chendongqi-boot", "AT-handleResumeActivity start clearHide = " + clearHide + ", isForward = " + isForward + ", reallyResume = " + reallyResume)
    // 同样在这里跳过GC
    unscheduleGcIdler();
    
    // 耗时操作最终会走到
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    ...
    if (r != null) {
    	...
    	Log.d("chendongqi-boot", "AT-handleResumeActivity hideForNow = " + r.hideForNow + ", finishied = " + a.mFinished);
    	if (!r.onlyLocalRequest) {
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            if (localLOGV) Slog.v(
                TAG, "Scheduling idle handler for " + r);
            Log.d("chendongqi-boot", "AT-handleResumeActivity Scheduling idle handler for " + r);
            // 这里是非常重要的一个机制，添加一个空闲队列，在进程空闲时会调用queueIdle方法
            // 关键流程1-10
            Looper.myQueue().addIdleHandler(new Idler());
        }
        ...
        // Tell the activity manager we have resumed.
        if (reallyResume) {
            try {    
              // 通知AMS，activity已经resumed了，AMS做些收尾工作
              ActivityManagerNative.getDefault().activityResumed(token);
            } catch (RemoteException ex) {
            }
        }
      	...
    } else {
      ...
    }
}
```
&emsp;&emsp;流程走到了1-10，先来看下这段流程中打出来的log，来分析下耗时情况  
```Java
// 从流程1-7开始
08-29 16:20:53.780 D/chendongqi-boot( 2190): AT-scheduleLaunchActivity//发送LAUNCH_ACTIVITY消息
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT.H-handleMessage LAUNCH_ACTIVITY//到处理消息过去了2.8s左右
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity start profiledFd = null//开始处理
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 00000
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 11111
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 22222
08-29 16:20:56.540 D/chendongqi-boot( 2190): AT-performLaunchActivity attach begin
08-29 16:20:56.620 D/chendongqi-boot( 2190): AT-performLaunchActivity attach end
// 这之前的处理都不耗时，准备开始create activity
08-29 16:20:56.630 D/chendongqi-boot( 2190): AT-performLaunchActivity callActivityOnCreate begin
08-29 16:20:56.630 D/chendongqi-boot( 2190): Launcher-onCreate start//在launcher中onCreate开始时加的log
08-29 16:20:58.756 D/chendongqi-boot( 2190): Launcher-onCreate end//在launcher中onCreate结束时加的log
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-performLaunchActivity callActivityOnCreate end//在onCreate中耗时2.1s左右
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleLaunchActivity 33333
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleLaunchActivity for xxx.launcher2
// 准备开始onResume流程
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleResumeActivity start clearHide = false, isForward = false, reallyResume = true
08-29 16:20:58.756 D/chendongqi-boot( 2190): Launcher-onResume start//在launcher中onResume开始时加的log
08-29 16:21:02.906 D/chendongqi-boot( 2190): Launcher-onResume end//在launcher中onResume结束时加的log，onResume耗时4.2s左右
08-29 16:21:02.916 D/chendongqi-boot( 2190): AT-handleResumeActivity hideForNow = false, finishied = false
08-29 16:21:02.916 D/chendongqi-boot( 2190): AT-handleResumeActivity window = null, willBeVisible = true
// 这里是往进程消息队列里addIdleHandler的操作的时间点
08-29 16:21:02.936 D/chendongqi-boot( 2190): AT-handleResumeActivity Scheduling idle handler for ActivityRecord{4196d220 token=android.os.BinderProxy@4196c9e8 {xxx.launcher2/com.xxx.launcher2.Launcher}}
08-29 16:21:02.936 D/chendongqi-boot( 1687): AMS-activityResumed start
08-29 16:21:02.936 D/chendongqi-boot( 2190): AT-handleLaunchActivity 44444
// thread真正空闲然后去执行流程1-10的时间点，过去了4s左右，这里也是瓶颈
08-29 16:21:06.846 D/chendongqi-boot( 2190): AT.Idler-queueIdle start
```
&emsp;&emsp;可见耗时部分有两个地方，其一是launcher的onCreate，其二是launcher的onResume部分，其三是等待thread的空闲处理耗时。再来看下流程1-10之后事情。在ActivityThread中走到queueIdle中时， 去调用AMS的activityIdle   
```Java
public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
    Log.d("chendongqi-boot", "AMS activityIdle start");
    ...
    // 走ASS的activityIdleInternalLocked
    mStackSupervisor.activityIdleInternalLocked(token, false, config);
    ...
}
```
```Java
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
    Configuration config) {
    Log.d("chendongqi-boot", "ASS activityIdleInternalLocked start token = " + token + ", fromTimeout = " + fromTimeout + ", config = " + config);
    ...
}
```
&emsp;&emsp;从观察启动过程中实时log的输出，在走到ASS的activityIdleInternalLocked方法时，launcher就显示出来了，从log的打印来看从AT.Idler-queueIdle方法开始到这里都是不耗时的了。所以这段方法里的具体时间就没去细究了。  


&emsp;&emsp;
这一部分的代码流程抠出来如下  
```Java
ASS.startSpecificActivityLocked
ASS.realStartActivityLocked
AT.scheduleLaunchActivity
AT.handleLaunchActivity
  AT.performLaunchActivity
  Instrumentation.callActivityOnCreate
  Activity.performCreate
  Activity.onCreate
AT.handleResumeActivity
  AT.performResumeActivity
  Activity.performResume
  Activity.performRestart
  Activity.performStart
  Instrumentation.callActivityOnResume
  Activity.onResume
  Activity.onPostResume
Looper.myQueue().addIdleHandler(new Idler());
AMS.activityResumed
AT.Idler.queueIdle
AMS.activityIdle
ASS.activityIdleInternalLocked
```

### fifth

&emsp;&emsp;第4节已经将主线流程介绍完了，一直到了最后看到launcher界面的出来，但是其实Activity能够秀出来跟系统的准备情况有很多千丝万缕的关系，这里来介绍下第2节中派生出来的3-1支线分支所做的事情，虽只是从侧面来给Activity的启动来做支持，但是设计到了更多的系统功能的设计思想，我觉得是这一部分是有非常大的价值的。  
&emsp;&emsp;那么流程就从3-1开始，准备好代码，入口是AS的completeResumeLocked  
```Java
private void completeResumeLocked(ActivityRecord next) {
	android.util.Log.d("chendongqi-boot", "AS-completeResumeLocked start next = " + next.packageName + ", nowVisible = " + next.nowVisible);
    ...
    // 关键流程3-2，发送一个超时消息
    // schedule an idle timeout in case the app doesn't do it for us.
    mStackSupervisor.scheduleIdleTimeoutLocked(next);
    ...
}
```
```Java
void scheduleIdleTimeoutLocked(ActivityRecord next) {
    Log.d("chendongqi-boot", "ASS-scheduleIdleTimeoutLocked start");
    if (DEBUG_IDLE) Slog.d(TAG, "scheduleIdleTimeoutLocked: Callers=" + Debug.getCallers(4));
    Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
    mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);//IDLE_TIMEOUT = 10*1000
    }
```
&emsp;&emsp;先在这里聊一下我自己对于这个超时处理的理解，4.4代码上这里是一个10s的延时消息，两个问题：为什么要设这样一个机制？为什么是10s？这个问题确实困扰了我，跟完整个流程之后我还是不理解这里真实的意义。为什么会有这一个流程存在可以从后面消息触发时所做的事大概探知一二，概括来讲就是去做了系统启动结束的收尾事情，具体的后面会继续介绍流程。而为什么是10s？我做过实验，将10s改成5s或者3s，甚至直接去掉延时，对整体的开机时间并无太大影响。而从log显示来看，这儿延时10s之后，收到消息是在launcher的onResume做完之后，难道设计的意思在此？现在无法理解的话暂时不纠结了，期待某天的顿悟吧。接着来看这个分支的流程，ASS的Handler中收到IDLE_TIMEOUT_MSG消息之后调用activityIdleInternal，转而调了activityIdleInternalLocked。  
```Java
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
    Configuration config) {
    // 这个方法也就是主线分支里的最后一步，显示出launcher的最后一步
    // 但是这里走到的时候看到config的参数是null，这个就是launcher还没显示的差别吧
    Log.d("chendongqi-boot", "ASS activityIdleInternalLocked start token = " + token + ", fromTimeout = " + fromTimeout + ", config = " + config);
    ...
    if (isFrontStack(mHomeStack)) {
    	//AMS的systemReady中mBooting=true
        booting = mService.mBooting;
        mService.mBooting = false;
    }
    ...
    if (!mService.mBooted && isFrontStack(r.task.stack)) {
        mService.mBooted = true;
        enableScreen = true;
    }
    ...
    // 这里booting是true
    if (booting) {
        Log.d("chendongqi-boot", "ASS go AMS->finishBooting");
        // 关键流程3-3
        mService.finishBooting();
    } else if (startingUsers != null) {
        for (int i = 0; i < startingUsers.size(); i++) {
            mService.finishUserSwitch(startingUsers.get(i));
        }
    }
    ...
    if (enableScreen) {//enableScreen==true
    	// 关键流程3-4
        mService.enableScreenAfterBoot();
    }
    ...
```
&emsp;&emsp;现在流程又归集到了AMS中去准备统一调控刚开机时的部分系统初始化工作了，先来看流程3-3之后所做的事  
```Java
final void finishBooting() {
    Log.d("chendongqi-boot", "AMS-finishBooting");
    ...
    if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
    	// Tell anyone interested that we are done booting!
        SystemProperties.set("sys.boot_completed", "1");
        SystemProperties.set("dev.bootcomplete", "1");
        ...
        // BOOT_COMPLETED广播是在这里发的，而且是有序广播
        Intent intent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
        intent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT);
        broadcastIntentLocked(null, null, intent, null,
        	new IIntentReceiver.Stub() {
        		@Override
                public void performReceive(Intent intent, int resultCode,
                		String data, Bundle extras, boolean ordered,
                		boolean sticky, int sendingUser) {
                	synchronized (ActivityManagerService.this) {
                			requestPssAllProcsLocked(SystemClock.uptimeMillis(),
                				true, false);
                    }
                }
            }
            0, null, null,
            android.Manifest.permission.RECEIVE_BOOT_COMPLETED,
            AppOpsManager.OP_NONE, true, false, MY_PID, Process.SYSTEM_UID,
            userId);
    }
}   
```
&emsp;&emsp;然后是3-4  
```Java
void enableScreenAfterBoot() {
    Log.d("chendongqi-boot", "AMS-enableScreenAfter Boot");
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_ENABLE_SCREEN,
            SystemClock.uptimeMillis());
    // 关键流程3-5，走到WMS里
    mWindowManager.enableScreenAfterBoot();

    synchronized (this) {
        updateEventDispatchingLocked();
    }
}
```
```Java
public void enableScreenAfterBoot() {
	Log.d("chendongqi-boot", "WMS-enableScreenAfterBoot start mSystemBooted = " + mSystemBooted);
	...
	// 关键流程3-6，走到PhoneWindowManager中
	mPolicy.systemBooted();
	// 关键流程3-7
	performEnableScreen();
}
```
&emsp;&emsp;3-6的流程又进入到了PhoneWindowManager中  
```Java
public void systemBooted() {
    Log.d("chendongqi-boot", "PWM-systemBooted start");
    if (mKeyguardDelegate != null) {
        Log.d("chendongqi-boot", "PWM-systemBooted KeyguardDelegate ");
        // 通知锁屏系统启动完成
        mKeyguardDelegate.onBootCompleted();
    }
    synchronized (mLock) {
    	// 修改启动完成的标志
        mSystemBooted = true;
    }
}
```
&emsp;&emsp;3-7的流程调用WMS的performEnableScreen 
```Java
public void performEnableScreen() {
	...
    // 跟开机现象默契相关的，通知SurfaceFlinger停止开机动画的时间点
	try {
        IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
        if (surfaceFlinger != null) {
            //Slog.i(TAG, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken("android.ui.ISurfaceComposer");
            surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
                                    data, null, 0);
            data.recycle();
        }
    } catch (RemoteException ex) {
        Slog.e(TAG, "Boot completed: SurfaceFlinger is dead!");
    }
    ...
    // 应用霍尔开关的状态，可以理解，霍尔就是为了皮套控制屏幕，所以要在屏幕ready之后
    mPolicy.enableScreenAfterBoot();
    // Make sure the last requested orientation has been applied.
    updateRotationUnchecked(false, false);
    Log.d("chendongqi-boot", "WMS-performEnableScreen end");
}
```
&emsp;&emsp;这一节获得了几个非常重要的点，在AMS的finishBooting中去发的开机广播，在WMS的performEnableScreen去关闭开机动画。这一节调用到的关键流程如下  
```Java
AS.completeResumeLocked
ASS.scheduleIdleTimeoutLocked
AMS.finishBooting
AMS.enableScreenAfterBoot
WMS.enableScreenAfterBoot
PWM.systemBooted
WMS.performEnableScreen
```
&emsp;&emsp;这一节相关的log输出如下  
```Java
08-29 16:20:53.790 D/chendongqi-boot( 1687): AS-completeResumeLocked start next = xxx.launcher2, nowVisible = false
08-29 16:20:53.820 D/chendongqi-boot( 1687): ASS activityIdleInternalLocked start token = Token{41a4c458 ActivityRecord{41c298e0 u0 xxx.launcher2/com.xxx.launcher2.Launcher t1}}, fromTimeout = true, config = null
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-finishBooting
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-finishBooting end
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-enableScreenAfter Boot
08-29 16:20:53.830 D/chendongqi-boot( 1687): WMS-enableScreenAfterBoot start mSystemBooted = false
08-29 16:20:53.830 D/chendongqi-boot( 1687): PWM-systemBooted start
// 在WMS的performEnableScreen流程里会重复调用4次
08-29 16:20:53.840 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.320 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.330 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.340 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
// 隔了4s左右才会最终终结这个方法，这里的玄机还没去探究
08-29 16:20:58.850 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:58.396 D/chendongqi-boot( 1687): WMS-performEnableScreen end
```

### sixth

&emsp;&emsp;整体的流程到这就全部结束了，虽然其中还有好几个问题没有理解，例如为什么要在ASS中发IDLE_TIMEOUT_MSG；为什么WMS的performEnableScreen会被重复好几次；在AT中通过空闲队列的消息调用activityIdleInternalLocked可以显示出launcher，但是在添加空闲队列的同时手动调用activityIdleInternalLocked不可以？；activityIdleInternalLocked方法是如何最终控制界面显示出来的？有些事当时想不明白就算了，毕竟还有很多工作等着去做，等待哪天在其他问题上有新的理解吧。最后贴下完整的log以供参考  
```Java
08-29 16:20:48.310 D/chendongqi-boot( 1687): WMS-systemReady
08-29 16:20:48.710 D/chendongqi-boot( 1687): AMS-systemReady start
08-29 16:20:50.420 D/chendongqi-boot( 1687): AS-resumeTopActivityLocked start
08-29 16:20:50.420 D/chendongqi-boot( 1687): AMS-startHomeActivityLocked
08-29 16:20:52.130 D/chendongqi-boot( 1687): ASS-startHomeActivity start
08-29 16:20:52.130 D/chendongqi-boot( 1687): AS-resumeTopActivityLocked start
08-29 16:20:52.150 D/chendongqi-boot( 1687): AMS-systemReady end
08-29 16:20:53.780 D/chendongqi-boot( 2190): AT-scheduleLaunchActivity
08-29 16:20:53.790 D/chendongqi-boot( 1687): AS-completeResumeLocked start next = xxx.launcher2, nowVisible = false
08-29 16:20:53.800 D/chendongqi-boot( 1687): ASS-scheduleIdleTimeoutLocked start
08-29 16:20:53.800 D/chendongqi-boot( 1687): ASS-reportResumedActivityLocked ready to goto ensureActivitiesVisibleLocked
08-29 16:20:53.820 D/chendongqi-boot( 1687): ASS IDLE_TIMEOUT_MSG
08-29 16:20:53.820 D/chendongqi-boot( 1687): ASS activityIdleInternal
08-29 16:20:53.820 D/chendongqi-boot( 1687): ASS activityIdleInternalLocked start token = Token{41a4c458 ActivityRecord{41c298e0 u0 xxx.launcher2/com.xxx.launcher2.Launcher t1}}, fromTimeout = true, config = null
08-29 16:20:53.830 D/chendongqi-boot( 1687): ASS go AMS->finishBooting
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-finishBooting
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-finishBooting end
08-29 16:20:53.830 D/chendongqi-boot( 1687): AMS-enableScreenAfter Boot
08-29 16:20:53.830 D/chendongqi-boot( 1687): WMS-enableScreenAfterBoot start mSystemBooted = false
08-29 16:20:53.830 D/chendongqi-boot( 1687): PWM-systemBooted start
08-29 16:20:53.830 D/chendongqi-boot( 1687): PWM-systemBooted KeyguardDelegate 
08-29 16:20:53.840 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.320 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.330 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:54.340 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT.H-handleMessage LAUNCH_ACTIVITY
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity start profiledFd = null
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 00000
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 11111
08-29 16:20:56.530 D/chendongqi-boot( 2190): AT-handleLaunchActivity 22222
08-29 16:20:56.540 D/chendongqi-boot( 2190): AT-performLaunchActivity attach begin
08-29 16:20:56.620 D/chendongqi-boot( 2190): AT-performLaunchActivity attach end
08-29 16:20:56.630 D/chendongqi-boot( 2190): AT-performLaunchActivity callActivityOnCreate begin
08-29 16:20:56.630 D/chendongqi-boot( 2190): Launcher-onCreate start
08-29 16:20:58.850 D/chendongqi-boot( 1687): WMS-performEnableScreen start, performEnableScreen: mDisplayEnabled=false mForceDisplayEnabled=false mShowingBootMessages=false mSystemBooted=true mOnlyCore=false

08-29 16:20:58.396 D/chendongqi-boot( 1687): WMS-performEnableScreen end
08-29 16:20:58.756 D/chendongqi-boot( 2190): Launcher-onCreate end
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-performLaunchActivity callActivityOnCreate end
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleLaunchActivity 33333
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleLaunchActivity for xxx.launcher2
08-29 16:20:58.756 D/chendongqi-boot( 2190): AT-handleResumeActivity start clearHide = false, isForward = false, reallyResume = true
08-29 16:20:58.756 D/chendongqi-boot( 2190): Launcher-onResume start

08-29 16:21:02.906 D/chendongqi-boot( 2190): Launcher-onResume end
08-29 16:21:02.916 D/chendongqi-boot( 2190): AT-handleResumeActivity hideForNow = false, finishied = false
08-29 16:21:02.916 D/chendongqi-boot( 2190): AT-handleResumeActivity window = null, willBeVisible = true
08-29 16:21:02.936 D/chendongqi-boot( 2190): AT-handleResumeActivity Scheduling idle handler for ActivityRecord{4196d220 token=android.os.BinderProxy@4196c9e8 {xxx.launcher2/com.xxx.launcher2.Launcher}}
08-29 16:21:02.936 D/chendongqi-boot( 1687): AMS-activityResumed start
08-29 16:21:02.936 D/chendongqi-boot( 2190): AT-handleLaunchActivity 44444
08-29 16:21:06.846 D/chendongqi-boot( 2190): AT.Idler-queueIdle start
08-29 16:21:06.846 D/chendongqi-boot( 1687): AMS activityIdle start
08-29 16:21:06.846 D/chendongqi-boot( 1687): ASS activityIdleInternalLocked start token = Token{41a4c458 ActivityRecord{41c298e0 u0 xxx.launcher2/com.xxx.launcher2.Launcher t1}}, fromTimeout = false, config = { themeChanged=0 themeChangedFlags=0 1.0 ?mcc?mnc zh_CN ldltr sw720dp w1280dp h654dp 160dpi xlrg long land finger -keyb/v/h -nav/h s.5}
```
