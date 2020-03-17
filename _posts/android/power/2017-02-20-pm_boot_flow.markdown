---
layout:      post
title:      "Android电源管理之开机流程"
subtitle:   "介绍Android系统的启动流程和相关代码"
navcolor:   "invert"
date:       2017-02-20
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - PowerManager
    - 开机
---

### 适用平台

Android Version： 6.0
Platform： MTK6580/MTK6735/MTK6753

### 0. 概述

&emsp;&emsp;一个完整的开机流程，从用户的角度上看是这样的：
![boot_flow_user](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/boot_flow_user.png)
&emsp;&emsp;长按power键感受到手机带给你的震动之后,奇妙之旅就此开始了。正常三秒之内呈现第一屏,一般呈现的是制造厂商的 logo;八秒左右进入第二屏,一般定制为品牌商 logo(从 ODM的角度看就是客户);然后播放开机动画和开机铃声;之后就进入了锁屏界面,开机过程引导完成。    
&emsp;&emsp;而从一个工程师的角度需要看到现象背后的实质,那么其实它是这样的:
![boot_flow_engineer](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/boot_flow_engineer.png)
&emsp;&emsp;（1）在第一屏显示之前和显示过程中，背后是uboot在引导os启动然后加载kernel的过程；（2）当kernel加载完成进入init进程之后，显示第二屏，这个现象的背后则是Android系统的第一个进程init启动，fork出zygote，然后由zygote去启动SystemServer；（3）当显示开机动画时，SystemServer在启动系统运行所需的众多核心服务和普通服务，以及初始化和加载一些应用；（4）播放完开机动画进入到锁屏或者launcher之后系统开机过程就基本结束了。    
&emsp;&emsp;以上是一个从开机现象看到的结果，简单的提到了开机现象背后的几个流程，以下我们将系统的来讨论整个开机流程。

### 1. 写在Android Boot之前

&emsp;&emsp;Android系统的开机基本上可以分成三个大的部分：Bootloader引导、Linux kernel启动、Android启动。Android启动对应的也就是现象中的第二屏显示开始，即Android的第一个进程init启动开始。这一章写在Android启动之前，那么我们就来谈谈Bootloader引导和Kernel的启动过程。

#### 1.1 Bootloader

&emsp;&emsp;首先来弄明白Bootloader到底是什么东西？抽象的说，Bootloader是在操作系统运行之前运行的一段程序，它可以将系统的软硬件环境带到一个合适状态，为运行操作系统做好准备。它的终极任务就是把OS带起来。    
&emsp;&emsp;在嵌入式系统的世界里，存在各种各样的Bootloader，划分方式有根据处理器的体系结构，也有功能的复杂程度。对于不同的体系结构都有一些可选的Bootloader源码可以选择：    
1. X86：X86的工作站和服务器上一般使用LILO和GRUB。    
2. ARM：最早有为ARM720处理器开发板所做的固件，又有了armboot，StrongARM平台的blob，还有S3C2410处理器开发板上的vivi等。现在armboot已经并入了U-Boot，所以U-Boot也支持ARM/XSCALE平台。U-Boot已经成为ARM平台事实上的标准Bootloader。    
3. PowerPC：最早使用于ppcboot，不过现在大多数直接使用U-boot。    
4. MIPS：最早都是MIPS开发商自己写的bootloader，不过现在U-boot也支持MIPS架构。    
5. M68K：Redboot能够支持m68k系列的系统。

&emsp;&emsp;在这里我们讨论下ARM架构的Bootloader。

#### 1.2 Arm特定平台的Bootloader

&emsp;&emsp;不同的ARM平台可能会有不同的Bootloader方案，这个方案中不仅包含了通常的Bootloader程序，一般还包含了其他loader程序。下面列举几个平台的例子:    
&emsp;&emsp;marvell(pxa935) :&emsp;&emsp;boot ROM + OBM [l4] + BLOB    
&emsp;&emsp;informax(im9815) :&emsp;&emsp;boot ROM + barbox + U-boot    
&emsp;&emsp;mediatek(mt6516/6517) :&emsp;&emsp;boot ROM + pre-loader[l5]  + U-boot    
&emsp;&emsp;broadcom(bcm2157) :&emsp;&emsp;boot ROM + boot1/boot2 + U-boot    
&emsp;&emsp;可以看出一般在Bootlaoder方案中包含了两个loader，然后才进入通常的Bootloader。由于不同处理器芯片厂商对ARM Core的封装差异比较大，所以不同的ARM处理器，对于上电引导都是由特定处理器芯片厂商自己开发的程序，这个上电引导程序通常比较简单，会初始化硬件，提供下载模式等，然后才会加载通常的Bootloader。    

#### 1.3 Boot Rom

&emsp;&emsp;以下是以我了解到的MTK方案来讨论的。    
&emsp;&emsp;每个MTK 处理器芯片都内嵌有Boot ROM，用于储存简单的启动程序代码。复位时如果boot引脚（GPIO0）被拉低，内部Boot ROM则被选择。Boot ROM里面储存着一个通过串口下载的小程序，此特性可用于下载或工厂测试。    
&emsp;&emsp;在MPCore（常用的一种ARM架构）中，每个ARM的处理器开始的存储地址都是0x00000000，通常有两种方式来提供程序代码执行：1）NOR Flash；2）Boot ROM。由于NOR Flash单位的存储成本比较高，所以当涉及到大容量存储空间的产品时，会选择用NAND Flash来存储Bootloader和操作系统，为了让操作系统能够启动，就会通过芯片上的Boot ROM定位到地址0x00000000，并在启动存储MPCore的程序代码。    
&emsp;&emsp;在系统未启动时，只有RTC Clock的时钟脉冲为32.768KHz，而在系统启动时，在PLL（Phase Locked Loop）起震前，只有Boot ROM或是NOR Flash这类设备可以用来执行处理器的指令。因此，在Boot ROM或NOR Flash中的代码必须让系统PLL正常，以便于可以达到最佳的处理器和平台性能。,在系统初始化外部存储（DRAM）之前，所使用的Stack或是可写入的存储区域就必须是芯片内部存储（SRAM），直到DRAM被初始化后才可以被使用。    
&emsp;&emsp;从支持NAND Boot的行为来说，Boot Rom需要执行以下操作：    
1. 让CPU0执行主要开机流程，其他的处理器进入WFI（在启动时，每个处理器可以通过CPU ID得知自己是否为CPU0，如果不是就进入WFI的程序代码中）    
2. 初始化外部存储和执行系统的初始化       
3. 设定Stack    
4. 把BootRom程序复制到外部存储中    
5. 重新定位存储位置，把0x00000000地址对应到外部存储中    
6. 把第二阶段的Bootloader载入到外部存储或者CPU内部存储中。    
7. 执行第二阶段的Bootloader

&emsp;&emsp;在MTK的Bootloader方案中，加入了preloader，它是MTK 内部的一个loader，其主要作用在于：1）保护芯片避免被黑；2）准备代码执行的安全环境；3）包含了许多安全功能和一些安全检查的步骤；4）加载和启动U-Boot
![mtk_bootloader](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/mtk_bootloader.png)

#### 1.4 U-Boot

&emsp;&emsp;U-Boot的官方名称为Universal boot loader，也是目前ARM上应用最广的一种Bootloader，是目前事实上的ARM Bootloader标准。在U-Boot阶段主要完成以下工作：1）配置系统的Memory；2）加载kernel镜像；3）准备一些传递给linux kernle的参数；4）获取机器类型；5）跳转到kernel
![uboot](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/uboot.png)

### 2 Init进程

&emsp;&emsp;Android的启动过程是从init开始的，所以它是后续所有进程的祖先进程。
![init_pid](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/init_pid.png)
&emsp;&emsp;可以看到init进程的pid是1。
![android_boot_flow](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/android_boot_flow.png)
&emsp;&emsp;从这张图Android启动流程图中可以看出init进程需要做的事情有：    
1. fork出一些系统关键服务（如mediaserver、servicemanager等）    
2. fork出zygote    
3. 提供属性服务来管理系统属性    
&emsp;&emsp;下面我们从代码角度来具体看看init线程都做了哪些事情。Init的代码位于system/core/init目录下，先来看init.cpp文件。在这个文件的main方法中可以初探端倪，以下代码为MTK AOSP Android6.0。    

* 1.将kernel启动过程中建立好的文件系统框架mount到相应目录   

```cpp
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sysfs", "/sys", "sysfs", 0, NULL);
```
* 2.调用util.cpp的open_devnull_stdio方法，用来将init进程的标准输入、输出、出错设备设置为新建的设备节点/dev/__null__。    
* 3.调用klog_init()和klog_set_level(KLOG_NOTICE_LEVEL)方法创建并打开设备节点/dev/__kmsg__来作为kernel log的输出节点。    
* 4.调用selinux_initialize方法来建立SELinux，如果处于内核空间则同时加载SELinux策略。    
* 5.调用signal_handler.cpp的signal_handler_init方法重新设置子进程终止时信号SIGCHLD的处理函数。    

```cpp
    // Write to signal_write_fd if we catch SIGCHLD.
    struct sigaction act;
    memset(&act, 0, sizeof(act));
    act.sa_handler = SIGCHLD_handler;
    act.sa_flags = SA_NOCLDSTOP;
    sigaction(SIGCHLD, &act, 0);

    reap_any_outstanding_children();

    register_epoll_handler(signal_read_fd, handle_signal);
```    
* 6.调用property_service.cpp的property_load_boot_defaults方法从文件中加载默认启动属性    
* 7.调用start_property_service方法，创建Property服务建立socket通信，并开始监听属性服务请求     
```cpp
    void start_property_service() {
        property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC |             SOCK_NONBLOCK, 0666, 0, 0, NULL);
        if (property_set_fd == -1) {
            ERROR("start_property_service socket creation failed: %s\n", strerror(errno));
            exit(1);
        }

        listen(property_set_fd, 8);
        register_epoll_handler(property_set_fd, handle_property_set_fd);
    }
```    
&emsp;&emsp;到这里也就是init线程提供的属性服务了。    
* 8.然后就是启动init.rc中定义的服务，这里分为几个步骤。首先是解析init.rc文件：      
```cpp
    init_parse_config_file("/init.rc");
```    
&emsp;&emsp;接下来就是根据init.rc中定义的不同触发器类型的服务都加入到service_list中，这里并不启动服务。      
```cpp
    action_for_each_trigger("early-init", action_add_queue_tail);
    // Trigger all the boot actions to get us started.
    action_for_each_trigger("init", action_add_queue_tail);
    // Don't mount filesystems or start core system services in charger mode.
    char bootmode[PROP_VALUE_MAX];
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }
```    
&emsp;&emsp;启动服务在builtins.c中：       
```cpp
    int do_class_start(int nargs, char **args)
    {
        /* Starting a class does not start services
         * which are explicitly disabled.  They must
         * be started individually.
         */
        service_for_each_class(args[1], service_start_if_not_disabled);
        return 0;
    }
```        
&emsp;&emsp;然后在init_parser.cpp中的service_for_each_class方法中遍历service_list中的所有class，将非disabled的服务启动起来。      
```cpp
    void service_for_each_class(const char *classname,
                            void (*func)(struct service *svc))
    {
        struct listnode *node;
        struct service *svc;
        list_for_each(node, &service_list) {
            svc = node_to_item(node, struct service, slist);
            if (!strcmp(svc->classname, classname)) {
                func(svc);
            }
        }
    }
```     
&emsp;&emsp;到这里就是init线程启动众多服务的过程了，服务的定义还需要参考init.rc	能够更好的理解。    
* 9.到这里init就进入了死循环中一直在监听ufds中的4个文件描述符的动静，如果有POLLIN的事件，就做相应的处理，所以init并没有退出或者进入idle，而是被当做一个服务在运行。       
```cpp
    while (true) {
        if (!waiting_for_exec) {
            execute_one_command();
            restart_processes();
        }

        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (!action_queue_empty() || cur_action) {
            timeout = 0;
        }

        bootchart_sample(&timeout);

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
```

#### 2.1 init.rc

&emsp;&emsp;init.rc的一类文件在目录system/core/rootdir下。在device/<vendor>/<project>目录下也可以做init.rc的客制化，一般名称为init.<project>.rc。    
&emsp;&emsp;init.rc由许多的Action和Service组成。每一个语句占据一行，并且各个关键字被空格分开，由#（前面允许有空格）开始的行都是注释行(comment)，一个actions或services的开始隐含声明了一个新的段，所有commands或options属于最近的声明。在第一个段之前的commands或options都会被忽略。每一个actions和services都有不同的名字。后面与前面发生重名的，那么这个后面重名的将被忽略或被认为是一个错误。    
&emsp;&emsp;我们先来看下actions的例子：actions其实就是一组被命名的命令序列。actions都有一个触发条件，触发条件决定了action何时执行。当一个事件发生如果匹配action的触发条件，那么这个action将会被添加到预备执行队列的尾部（除非它已经在队列当中）。每一个action中的命令将被顺序执行，actions的格式如下：       
```bash
    on <trigger>
    <command>
    <command>
	...
```      
&emsp;&emsp;例如init.rc中的例子：      
```bash
    on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000

    # Set the security context of /adb_keys if present.
    restorecon /adb_keys

    start ueventd
```      
&emsp;&emsp;Trigger就是触发条件，标示command执行的时机，具体想了解一共有多少种trigger可以参见资料[ init.rc文件介绍 ](http://note.youdao.com/share/?id=23eed5feb131afe2f167d69bce56c841&type=note)。    
&emsp;&emsp;services是一些由init启动和重新（如果有需要）启动的程序，当然这些程序如果是存在的。    
&emsp;&emsp;每一个service格式如下：      
```bash
    service <name> <pathname> [ <argument> ]*
    <option>
    <option>
    ...
```      
&emsp;&emsp;例如：      
```bash
    service media /system/bin/mediaserver
        class main
        user media
        group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm sdcard_r system net_bt_stack sdcard_rw

        ioprio rt 4
        
    service bootanim /system/bin/bootanimation
        class core
        user graphics
        group graphics audio
        disabled
        oneshot
```     
&emsp;&emsp;options是service的修饰符，用来告诉init怎样及何时启动service。具体选项和含义也可以参照上面的链接。    
&emsp;&emsp;在init.rc中定义和启动的众多服务将会在后续继续讲到。    

#### 2.2 Android属性系统PropertyService

&emsp;&emsp;此节暂时和开机的流程联系不是太紧密，这里先不详述，需要了解可以参考[  Android 属性系统 Property service 设定分析  ](http://blog.csdn.net/andyhuabing/article/details/7381879)。    

### 3. Zygote

&emsp;&emsp;Zygote翻译成中文是受精卵的意思，名字比较奇怪、但是在知道了android进程创建之后就会觉得非常形象。在android中，大部分的应用程序进程都是由zygote来创建的，为什么用大部分，因为还有一些进程比如系统引导进程、init进程等不是由zygote创建的。相反，zygote还是在init进程之后才被创建的。
那么zygote是如何起来的？答案还是在init.rc中。在init.rc文件开始的地方又include了另外几个init*.rc文件:      
```bash
    import /init.environ.rc
    import /init.usb.rc
    import /init.${ro.hardware}.rc
    import /init.${ro.zygote}.rc
    import /init.trace.rc
```     
&emsp;&emsp;其中就包括了zygote相关。         
![zygote_files](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/zygote_files.png)
&emsp;&emsp;在和init.rc同个目录下（system/core/rootdir）中可以看到上面这几个文件，以32位的为例。看到内容不多，就定义了一个service。       
```bash
    service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
        class main
        socket zygote stream 660 root system
        onrestart write /sys/android_power/request_state wake
        onrestart write /sys/power/state on
        onrestart restart media
        onrestart restart netd
```       
&emsp;&emsp;这个service是在系统启动之初来启动一个名为app_process的进程。那么这个app_process和zygote又有何关系？在linux的用户空间，进程app_process会做一些zygote进程启动的前期工作，如，启动runtime运行时环境(实例)，参数分解，设置startSystemServer标志，接着用runtime.start()来执行zygote服务的代码，其实说简单点，就是zygote抢了app_process这个进程的躯壳，改了名字，将后面的代码换成zygote的main函数，这样顺利地过度到了zygote服务进程。这样我们在控制台用ps看系统所有进程，就不会看到app_process，取而代之的是zygote。我们来从代码中核实一下。    
&emsp;&emsp;首先来看一下app_process，其代码在frameworks/base/cmds/app_process目录下，来看app_main.cpp的main函数。      
```cpp
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }    
```    
&emsp;&emsp;在这里就是上文提到过的zygote抢了app_process进程的躯壳，然后改了process的名字，将在后面启动zygote的main函数，过度到zygote进程。     
```cpp
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    }
```       
&emsp;&emsp;通过AndroidRuntime来启动zygote进程，这里比较重要的就是创建VM虚拟机的操作了。代码进入了frameworks/base/core/jni/AndroidRuntime.cpp中的start方法，然后比较关键的代码就是：       
```cpp
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
```       
&emsp;&emsp;Runtime启动的ZygoteInit进程代码在frameworks/base/core/java/com/android/internal/os/ZygoteInit.java。在其main函数中比较重要的几点如下：       
* 调用registerZygoteSocket(socketName)为zygote命令连接注册一个服务器套接字    
* 调用preload预加载类和资源      
```java
    static void preload() {
        Log.d(TAG, "begin preload");
        Log.i(TAG1, "preloadMappingTable() -- start ");
        PluginLoader.preloadPluginInfo();
        Log.i(TAG1, "preloadMappingTable() -- end ");
        preloadClasses();
        preloadResources();
        preloadOpenGL();
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        Log.d(TAG, "end preload");
    }
```       
&emsp;&emsp;preloadClasses方法会预加载一个文本文件中包含的一系列类，这个文件在手机中的目录为:       
```java
    private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";
```       
&emsp;&emsp;在源码中的位置为framework/base/preloaded-classes。    
&emsp;&emsp;preloadResources() preloadResources也意味着本地主题、布局以及android.R文件中包含的所有东西都会用这个方法加载。    
&emsp;&emsp;后面的几个preload方法也会加载OpenGl库，共享库等等。    
* 然后就走到开机流程中关键的一步，启动SystemServer      
```java
    if (startSystemServer) {
        startSystemServer(abiList, socketName);
    }
```     
&emsp;&emsp;startSystemServer方法中调用了Zygote.forkSystemServer，然后子进程就调用handleSystemServerProcess，而父进程则直接返回true并沿着步骤4往下走。      
```java
    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
            parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }
```      
&emsp;&emsp;在handleSystemServerProcess方法中，通过ClassPath来装载Dex文件，然后是构造一些参数，最后通过RuntimeInit.zygoteInit来将参数传递给SystemServer，调用的是frameworks/base/core/java/com/android/internal/os/RuntimeInit.java中的zygoteInit方法。这里可以根据参数传递的走向来跟踪主要流程：      
&emsp;&emsp;在zygoteInit方法中继续调用applicationInit方法:      
```java
    applicationInit(targetSdkVersion, argv, classLoader);
```      
&emsp;&emsp;然后继续调用invokeStaticMain方法：     
```java
    // Remaining arguments are passed to the start class's static main
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
```       
&emsp;&emsp;这里离SystemServer就不远了，最后的几步是这样实现的。先获取到SystemServer的main方法（带参数 system_server），结果就是得到包com.android.server.SystemServer的main()函数：       
```java
    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
            "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
            "Problem getting static main on " + className, ex);
    }
```       
&emsp;&emsp;然后交给ZygoteInit的内部类来处理：       
```java
    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
```      
&emsp;&emsp;MethodAndArgsCaller类实现了Runnable，看它的run方法做的事情：       
```java
    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
```       
&emsp;&emsp;通过Method的invoke来运行包com.android.server.SystemServer的main()函数。这样就启动了SystemServer，代码位于frameworks/base/services/java/com/android/server/SystemServer.java。      
```java
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```       
* runSelectLoop()：这是zygote最后做的事情。进入一个死循环，不再返回，开始孵化整个Android的工作。代码如下：       
```java
    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        try {
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {// 读取新的zygoteConnection
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                boolean done = peers.get(i).runOnce();// 处理command
                if (done) {
                    peers.remove(i);
                    fds.remove(i);
                }
            }
        }
    }
```      
&emsp;&emsp;其工作原理如下图所示：      
![zygote_service_flow](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/zygote_service_flow.png)
&emsp;&emsp;此原理更详细的可以参考[ zygote工作流程分析 ](http://www.cnblogs.com/bastard/archive/2012/09/03/2668579.html)。

### 4. SystemServer

&emsp;&emsp;上面已经讲到了从zygote是如何启动SystemServer的，可以callstack中的函数调用栈再来回顾一下：    
![zygote_callstack](https://chendongqi.github.io/blog/img/2017-02-20-powermanager-boot_flow/zygote_callstack.png)
&emsp;&emsp;这一节就来关注下SystemServer在启动过程中所做的事情。代码位于frameworks/base/services/java/com/android/server/SystemServer.java。流程从run方法开始，下面来看下几个关键步骤：      
* 准备消息队列       
```java
    // Prepare the main looper thread (this thread).
    android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);
    Looper.prepareMainLooper();
```       
* 加载库：android_servers库是frameworks/base/services/core/jni目录下编译出来的共享库。        
```java
    // Initialize native services.
    System.loadLibrary("android_servers");
```      
* 检查上次关机是否失败，如果是，则在这一步直接进行关机，就没有然后了。如果正常则继续往下走，创建系统context。      
```java
    // Check whether we failed to shut down last time we tried.
    // This call may not return.
    performPendingShutdown();

    // Initialize the system context.
    createSystemContext();
```       
* 开始启动各种服务：首先是创建SystemServiceManager用来管理服务的生命周期，然后开始启动三类服务，包括了Boot Strap（启动流程相关服务）、Core（核心服务）和Other（其他服务），详见第5章系统服务。       
* 调用Looper.loop()方法让SystemServer的Looper开始工作，从消息队列里取消息，处理消息。       

### 5. 系统服务

```java
    // Create the system service manager.
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

    // Start services.
    try {
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        /// M: RecoveryManagerService  @{
        if (mRecoveryManagerService != null && ex instanceof RuntimeException) {
            mRecoveryManagerService.handleException((RuntimeException) ex, true);
        } else {
            throw ex;
        }
        /// @}
    }
```      
&emsp;&emsp;**先来看一下startBootstrapServices都启动了哪些服务：**     
&emsp;&emsp;Installer服务：在它起来之后才有合适的权限创建一些重要目录例如data/user，需要在其他服务初始化之前完成这一步。      
```java
    // Wait for installd to finish starting up so that it has a chance to
    // create critical directories such as /data/user with the appropriate
    // permissions.  We need this to complete before we initialize other services.
    Installer installer = mSystemServiceManager.startService(Installer.class);
```       
&emsp;&emsp;启动ActivityManagerService：用来管理应用进程的生命周期以及进程的Activity，Service，Broadcast和Provider等。      
```java
    // Activity manager runs the show.
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
```      
&emsp;&emsp;启动PowerManagerService：PMS也必须很早就启动，因为其他服务也都用来它。      
```java
    // Power manager needs to be started early because other services need it.
    // Native daemons may be watching for it to be registered so it must be ready
    // to handle incoming binder calls immediately (including being able to verify
    // the permissions for those calls).
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
```      
&emsp;&emsp;启动LightService：管理LED等和屏幕背光。     
```java
    // Manages LEDs and display backlight so we need it to bring up the display.
    mSystemServiceManager.startService(LightsService.class);
```       
&emsp;&emsp;启动DisplayManagerService：显示管理服务，支持多种显示类型的多个显示器的镜像显示，包括内建的显示类型（本地）、HDMI显示类型以及支持WIFI Display 协议( MIRACAST)，实现本地设备在远程显示器上的镜像显示。      
```java
    // Display manager is needed to provide display metrics before package manager
    // starts up.
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
```     
&emsp;&emsp;启动PackageManagerService：负责管理系统的Package，包括APK的安装，卸载，信息的查询等等。       
```java
    // Start the package manager.
    Slog.i(TAG, "Package Manager");
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();
```      
&emsp;&emsp;往ServiceManager中注册UserManagerService：提供了创建/删除/擦除用户、用户信息获取、用户句柄获取等用户操作的方法。      
```java
    Slog.i(TAG, "User Service");
    ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());
```       
&emsp;&emsp;在后续启动SensorService。       
```java
    // The sensor service needs access to package manager service, app ops
    // service, and permissions service, therefore we start it after them.
    startSensorService();
```               
**再来看下启动的核心服务都有哪些：**       
* 1.启动BatteryService来跟踪电量以及发送电池广播，需要LightService先启动    
* 2.启动UsageStatsService来提供收集统计应用程序数据使用状态的服务    
* 3.启动WebViewUpdateService来跟踪可用的更新

```java
    /**
     * Starts some essential services that are not tangled up in the bootstrap process.
     */
    private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);

        // Tracks application usage stats.
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        // Update after UsageStatsService is available, needed before performBootDexOpt.
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```

**然后调用startOtherServices()启动一些杂项的服务：**      
1. 注册调度策略服务SchedulingPolicyService      
```java
    Slog.i(TAG, "Scheduling Policy");
    ServiceManager.addService("scheduling_policy", new SchedulingPolicyService());
```       
2. 注册TelecomLoaderService    
```java
    mSystemServiceManager.startService(TelecomLoaderService.class);
```    
3. 注册TelephonyRegistry服务提供电话注册、管理服务，可以获取电话的链接状态、信号强度等等     
```java
    Slog.i(TAG, "Telephony Registry");
    telephonyRegistry = new TelephonyRegistry(context);
    ServiceManager.addService("telephony.registry", telephonyRegistry);
```    
4. 新建EntropyMixer，用于生成随机数    
```java
    Slog.i(TAG, "Entropy Mixer");
    entropyMixer = new EntropyMixer(context);
```    
5. 启动CameraService来提供相机服务    
```java
    Slog.i(TAG, "Camera Service");
    mSystemServiceManager.startService(CameraService.class);
```    
6. AccountManagerService用来管理账号    
```java
    Slog.i(TAG, "Account Manager");
    accountManager = new AccountManagerService(context);
    ServiceManager.addService(Context.ACCOUNT_SERVICE, accountManager);
```    
7. 内容服务，主要是数据库等提供解决方法的服务     
```java
    Slog.i(TAG, "Content Manager");
    contentService = ContentService.main(context,
    mFactoryTestMode == FactoryTest.FACTORY_TEST_LOW_LEVEL);
```    
8. 提供ContentProviders      
```java
    Slog.i(TAG, "System Content Providers");
    mActivityManagerService.installSystemProviders();
```    
9. 震动服务    
```java
    Slog.i(TAG, "Vibrator Service");
    vibrator = new VibratorService(context);
    ServiceManager.addService("vibrator", vibrator);
```    
10. 远程控制，通过红外等控制周围的设备（例如电视等）     
```java
    Slog.i(TAG, "Consumer IR Service");
    consumerIr = new ConsumerIrService(context);
    ServiceManager.addService(Context.CONSUMER_IR_SERVICE, consumerIr);
```    
11. 闹钟服务     
```java
    mSystemServiceManager.startService(AlarmManagerService.class);
    alarm = IAlarmManager.Stub.asInterface(
    ServiceManager.getService(Context.ALARM_SERVICE));
```    
12. Watchdog用来监视进程状态，杀死无返回的进程    
```java
    Slog.i(TAG, "Init Watchdog");
    final Watchdog watchdog = Watchdog.getInstance();
    watchdog.init(context, mActivityManagerService);
```    
13. 提供输入法服务    
```java
    Slog.i(TAG, "Input Manager");
    inputManager = new InputManagerService(context);
```     
14. Android framework框架核心服务，窗口管理服务    
```java
    Slog.i(TAG, "Window Manager");
    wm = WindowManagerService.main(context, inputManager,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
            !mFirstBoot, mOnlyCore);
    ServiceManager.addService(Context.WINDOW_SERVICE, wm);
```     
15. 蓝牙服务    
```java
    mSystemServiceManager.startService(BluetoothService.class);
```    
16. 做dex优化。dex是Android上针对Java字节码的一种优化技术，可提高运行效率    
```java
    mPackageManagerService.performBootDexOpt();
```    
17. 和锁屏界面中的输入密码，手势等安全功能有关。可以保存每个user的相关锁屏信息    
```java
    Slog.i(TAG,  "LockSettingsService");
    lockSettings = new LockSettingsService(context);
    ServiceManager.addService("lock_settings", lockSettings);
```    
18. 提供往一个可持续的存储中读写数据块的操作，如果从设置中恢复出厂设置数据会清空，其他方式恢复出厂设置则数据可保持    
```java
    mSystemServiceManager.startService(PersistentDataBlockService.class);
```     
19. 监测系统空闲并进入省电模式    
```java
    mSystemServiceManager.startService(DeviceIdleController.class);
```    
20. 提供一些系统级别的设置及属性的服务    
```java
    // Always start the Device Policy Manager, so that the API is compatible with
    // API8.
    mSystemServiceManager.startService(DevicePolicyManagerService.Lifecycle.class);
```    
21. 状态栏管理服务    
```java
    Slog.i(TAG, "Status Bar");
    statusBar = new StatusBarManagerService(context, wm);
    ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
```    
22. 粘贴板服务    
```java
    Slog.i(TAG, "Clipboard Service");
    ServiceManager.addService(Context.CLIPBOARD_SERVICE,
    new ClipboardService(context));
```    
23. 网络管理服务    
```java
    Slog.i(TAG, "NetworkManagement Service");
    networkManagement = NetworkManagementService.create(context);
    ServiceManager.addService(Context.NETWORKMANAGEMENT_SERVICE, networkManagement);
```    
24. 文本服务，例如文本检查等    
```java
    Slog.i(TAG, "Text Service Manager Service");
    tsms = new TextServicesManagerService(context);
    ServiceManager.addService(Context.TEXT_SERVICES_MANAGER_SERVICE, tsms);
```    
25. 网络服务相关：ANDROID系统网络连接和管理服务由四个系统服务ConnectivityService、NetworkPolicyManagerService、NetworkManagementService、NetworkStatsService共同配合完成网络连接和管理功能。ConnectivityService、NetworkPolicyManagerService、NetworkStatsService三个服务都通过INetworkManagementService接口跨进程访问NetworkManagementService服务，实现与网络接口的交互及信息读取    
```java
    Slog.i(TAG, "Network Score Service");
    networkScore = new NetworkScoreService(context);
    ServiceManager.addService(Context.NETWORK_SCORE_SERVICE, networkScore);
                
    Slog.i(TAG, "NetworkStats Service");
    networkStats = new NetworkStatsService(context, networkManagement, alarm);
    ServiceManager.addService(Context.NETWORK_STATS_SERVICE, networkStats);
                    
    Slog.i(TAG, "NetworkPolicy Service");
    networkPolicy = new NetworkPolicyManagerService(context, mActivityManagerService,
            (IPowerManager)ServiceManager.getService(Context.POWER_SERVICE),                                 networkStats, networkManagement);
    ServiceManager.addService(Context.NETWORK_POLICY_SERVICE, networkPolicy);
```    
26. wifi p2p、wifi、wifi扫描以及以太网服务等    
```java
    mSystemServiceManager.startService(WIFI_P2P_SERVICE_CLASS);
    mSystemServiceManager.startService(WIFI_SERVICE_CLASS);
    mSystemServiceManager.startService("com.android.server.wifi.WifiScanningService");
    mSystemServiceManager.startService("com.android.server.wifi.RttService");

    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_ETHERNET) ||
                mPackageManager.hasSystemFeature(PackageManager.FEATURE_USB_HOST)) {
        mSystemServiceManager.startService(ETHERNET_SERVICE_CLASS);
    }
```    
27. 升级服务    
```java
    Slog.i(TAG, "UpdateLock Service");
    ServiceManager.addService(Context.UPDATE_LOCK_SERVICE,
                new UpdateLockService(context));
```     
28. 通知服务    
```java
    mSystemServiceManager.startService(NotificationManagerService.class);
    notification = INotificationManager.Stub.asInterface(
                ServiceManager.getService(Context.NOTIFICATION_SERVICE));
    networkPolicy.bindNotificationManager(notification);
```    
29. 位置管理服务    
```java
    Slog.i(TAG, "Location Manager");
    location = new LocationManagerService(context);
    ServiceManager.addService(Context.LOCATION_SERVICE, location);
```    
30. 国家检测服务    
```java
    Slog.i(TAG, "Country Detector");
    countryDetector = new CountryDetectorService(context);
    ServiceManager.addService(Context.COUNTRY_DETECTOR, countryDetector);
```    
31. 搜索服务    
```java
    Slog.i(TAG, "Search Service");
    ServiceManager.addService(Context.SEARCH_SERVICE,
            new SearchManagerService(context));
```    
32. DropBox服务，用于异常信息的搜集    
```java
    Slog.i(TAG, "DropBox Service");
    ServiceManager.addService(Context.DROPBOX_SERVICE,
            new DropBoxManagerService(context, new File("/data/system/dropbox")));
```    
33. 墙纸服务    
```java
    Slog.i(TAG, "Wallpaper Service");
    wallpaper = new WallpaperManagerService(context);
            ServiceManager.addService(Context.WALLPAPER_SERVICE, wallpaper);
```   
34. 音频服务    
```java
    Slog.i(TAG, "Audio Service");
    audioService = new AudioService(context);
    ServiceManager.addService(Context.AUDIO_SERVICE, audioService);
```    
35. 监测接口状态服务    
```java
    if (!disableNonCoreServices) {
        mSystemServiceManager.startService(DockObserver.class);
    }
```    
36. 监测有线耳机    
```java
    Slog.i(TAG, "Wired Accessory Manager");
    // Listen for wired headset changes
    inputManager.setWiredAccessoryCallbacks(
            new WiredAccessoryManager(context, inputManager));
```    
37. MIDI服务    
```java
    // Start MIDI Manager service
    mSystemServiceManager.startService(MIDI_SERVICE_CLASS);
```    
38. 管理USB host    
```java
    // Manage USB host and device support
    mSystemServiceManager.startService(USB_SERVICE_CLASS);
```    
39. 串口服务    
```java
    Slog.i(TAG, "Serial Service");
    // Serial port support
    serial = new SerialService(context);
    ServiceManager.addService(Context.SERIAL_SERVICE, serial);
```    
40. 用来管理夜间模式    
```java
    mSystemServiceManager.startService(TwilightService.class);
```    
41. 管理任务计划    
```java
    mSystemServiceManager.startService(JobSchedulerService.class);
```    
42. 备份服务    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_BACKUP)) {
        mSystemServiceManager.startService(BACKUP_MANAGER_SERVICE_CLASS);
    }
```    
43. 管理Widget    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_APP_WIDGETS)) {
        mSystemServiceManager.startService(APPWIDGET_SERVICE_CLASS);
    }
```    
44. 语音识别    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_VOICE_RECOGNIZERS)) {
        mSystemServiceManager.startService(VOICE_RECOGNITION_MANAGER_SERVICE_CLASS);
    }
```    
45. 磁盘状态管理服务    
```java
    Slog.i(TAG, "DiskStats Service");
    ServiceManager.addService("diskstats", new DiskStatsService(context));
```    
46. 监视网络时间，当网络时间变化时更新本地时间    
```java
    Slog.i(TAG, "NetworkTimeUpdateService");
    networkTimeUpdater = new NetworkTimeUpdateService(context);
```    
47. 管理本地常见的时间服务的配置，在网络配置变化时重新配置本地服务    
```java
    Slog.i(TAG, "CommonTimeManagementService");
    commonTimeMgmtService = new CommonTimeManagementService(context);
    ServiceManager.addService("commontime_management", commonTimeMgmtService);
```    
48. 屏幕保护    
```java
    // Dreams (interactive idle-time views, a/k/a screen savers, and doze mode)
    mSystemServiceManager.startService(DreamManagerService.class);
```    
49. 图像管理    
```java
    ServiceManager.addService(GraphicsStatsService.GRAPHICS_STATS_SERVICE,
            new GraphicsStatsService(context));
```     
50. 打印服务    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_PRINTING)) {
        mSystemServiceManager.startService(PRINT_MANAGER_SERVICE_CLASS);
    }
```    
51. 限制管理    
```java
    mSystemServiceManager.startService(RestrictionsManagerService.class);
```    
52. 多媒体会话服务    
```java
    mSystemServiceManager.startService(MediaSessionService.class);
```    
53. HDMI控制    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_HDMI_CEC)) {
        mSystemServiceManager.startService(HdmiControlService.class);
    }
```    
54. TV连接管理    
```java
    if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_LIVE_TV)) {
        mSystemServiceManager.startService(TvInputManagerService.class);
    }
```    
55. 可信设备管理    
```java
    mSystemServiceManager.startService(TrustManagerService.class);
```    
55. FingerPrinter服务    
```java
    mSystemServiceManager.startService(FingerprintService.class);
```    
56. APP启动服务    
```java
    mSystemServiceManager.startService(LauncherAppsService.class);
```    
57. 短信代理服务    
```java
    // MMS service broker
    mmsService = mSystemServiceManager.startService(MmsServiceBroker.class);
```    
58. 在启动完各种服务之后然后调用服务的SystemReady和startBootPhase来调用其他Services中的开机启动流程。

### 6. 开机动画

&emsp;&emsp;开机动画是在SurfaceFlinger中来完成的，所以先来看一下SurfaceFlinger是如何启动的。
在init.rc中定义了SurfaceFlinger服务:      
```bash
    service surfaceflinger /system/bin/surfaceflinger
        class core
        user system
        group graphics drmrpc
        onrestart restart zygote
```      
&emsp;&emsp;所以在init也开始了SurfaceFlinger服务的启动，/system/bin/srufaceflinger可执行文件是由frameworks/native/services/surfaceflinger目录编译生成的。执行时会先走到该目录下main_surfaceflinger.cpp的main方法。     
```cpp
    int main(int, char**) {
        // When SF is launched in its own process, limit the number of
        // binder threads to 4.
        ProcessState::self()->setThreadPoolMaxThreadCount(4);
    
        // start the thread pool
        sp<ProcessState> ps(ProcessState::self());
        ps->startThreadPool();
    
        // instantiate surfaceflinger
        sp<SurfaceFlinger> flinger = new SurfaceFlinger();
    
        setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    
        set_sched_policy(0, SP_FOREGROUND);
    
        // initialize before clients can connect
        flinger->init();
    
        // publish surface flinger
        sp<IServiceManager> sm(defaultServiceManager());
        sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false);
    
        // run in this thread
        flinger->run();
    
        return 0;
    }
```       
&emsp;&emsp;在main方法中主要先new了一个SurfaceFlinger的对象，然后调用该对象的init方法，再调用run方法。       
&emsp;&emsp;再来看SurfaceFlinger.cpp的init方法来初始化：在这个方法中就是去初始化GL和图形绘制的动作，在这个方法的最后可以看到调用了startBootAnim方法来尝试播放开机动画。      
```cpp
    // start boot animation
    startBootAnim();
```    
&emsp;&emsp;在这个方法里就是通过属性服务来控制开机动画的播放的。      
```cpp
    void SurfaceFlinger::startBootAnim() {
    #ifdef MTK_AOSP_ENHANCEMENT
        // dynamic disable/enable boot animation
        checkEnableBootAnim();
    #else
        // start boot animation
        property_set("service.bootanim.exit", "0");
        property_set("ctl.start", "bootanim");
    #endif
    }
```      
&emsp;&emsp;把service.bootanim.exit属性设为0，这个属性bootanimation进程里会周期检查，=1时就退出动画，这里=0表示要播放动画。后面通过ctl.start的命令启动bootanimation进程，动画就开始播放了。    
&emsp;&emsp;那么接下来的工作就是在bootanimation进程里实现开机动画的播放，这个进程在手机中的目录为system/bin/bootanimation，代码中的目录为frameworks/base/cmds/bootanimation。详细实现的方法这里不再介绍，具体可以参照[ android开机动画启动流程  ](http://blog.csdn.net/happy_6678/article/details/46236831)。    

### 7. 进入home

&emsp;&emsp;在startOtherServicer方法启动完各种系统服务之后，就会去调用ActivityManagerService.systemReady的方法来通知AMS系统启动ready，可以开始启动第三方应用程序了。这是个回调的方式，我们再来看一下AMS中的systemReady方法中都写了什么，其中可以看到一段    ：

```java
    /// M: power-off Alarm feature @{
    // Start up initial activity.
    /// M: ALPS01855968 Set booting flag to start actviity successfully.
    mBooting = true;

    /// M: AMEventHook event @{
    // phase 300: Before calling startHomeActivityLocked to launch home activity.
    eventData = AMEventHookData.SystemReady.createInstance();
    eventData.set(300, mContext, this);
    eventResult = mAMEventHook.hook(AMEventHook.Event.AM_SystemReady,
            eventData);
    /// M: AMEventHook event @}

    if (!PowerOffAlarmUtility.isAlarmBoot()) {
        startHomeActivityLocked(mCurrentUserId, "systemReady");
    }
    /// @}
```    
&emsp;&emsp;这里有MTK修改的痕迹，google原始的代码只是调用startHomeActivityLocked方法。这个方法就是来启动Home的。



