---
layout:      post
title:      "Android开机系列二"
subtitle:   "开机情景下Network的部分初始化任务"
navcolor:   "invert"
date:       2017-11-15
author:     "Cheson"
catalog: true
tags:
    - Android
    - frameworks
    - NetworkManagement
    - dhcp
    - 开机
    - netd
    - ConnectivityService
    - libnetutils
---

### Android Version

Android version: 4.4  
Platform: freescale imx6  

### 问题背景

&emsp;&emsp;这篇是关于网络初始化相关的，而内容的来源是出于调查一个产线的问题，花了两周左右才把涉及到的相关代码流程搞得七七八八，不记录下来也真的有点对不住这次经历了。  
&emsp;&emsp;这个问题是这样的，在产线生产测试时发现开机后有10%的概率出现网络无法访问，但是车机的4G网络连接显示是正常的。进一步测试后发现去ping域名（例如www.baidu.com）是直接提示unknown host，而去ping它的ip地址是可以正常收到包的。  
&emsp;&emsp;而另一个背景就是车机里的4G网络实现是通过ZTE的Tbox模块和车机进行usb连接通过Ethernet的方式给车机提供网络访问，实际的网络接口名称是tbox0。

### 初步分析

&emsp;&emsp;背景知识都清楚了之后下面就跟着来回顾下此问题的整个分析周期，开始一个网络管理从入门到放弃的过程。虽然对网络管理非常的陌生，但好歹也是科班出身，一点点的基础还是有的，了解到ping域名不通而ip地址正常的这个信息之后，应该能推测到是dns解析这块出现了问题，用getprop去查看了车机里网络相关的属性值，和正常的一份对比了下发现了几个地方。  
```xml
//正常的
[dhcp.tbox0.dns1]: [192.168.225.1]
[dhcp.tbox0.gateway]: [192.168.225.1]
[dhcp.tbox0.ipaddress]: [192.168.225.153]
[dhcp.tbox0.mask]: [255.255.255.0]
[dhcp.tbox0.reason]: [REBOOT]
[dhcp.tbox0.result]: [ok]
[net.change]: [net.dns1]
[net.dns1]: [192.168.225.1]
```
```xml
//异常的
[dhcp.tbox0.dns1]: [192.168.225.1]
[dhcp.tbox0.gateway]: [192.168.225.1]
[dhcp.tbox0.ipaddress]: [192.168.225.153]
[dhcp.tbox0.mask]: [255.255.255.0]
[dhcp.tbox0.reason]: [REBOOT]
[dhcp.tbox0.result]: []
[net.change]: [net.qtaguid_enabled]
```
&emsp;&emsp;可以看到的其实异常情况下tbox0这个网络接口的ip地址，网关，掩码和dns都是有的，而对比的结果显示的是dhcp.tbox0.result这个值是空的，net.dns1这个字段没有被设置。那么结合现象可以推测net.dns1这个字段应该是决定了设备是否有能力做dns解析的关键，那么这应该是一个方向去查为什么没有生成这个字段。而另一个方向是发现dhcp.tbox0.result这个字段为什么是空的，机缘巧合我选择了第二条路。回过头来回想了整个排查过程，由于之前对这部分的流程一无所知，所以要查一个字段在哪里设置的，还真的是非常难，况且到后面还能看到这个字段其实是拼接的，所以全局搜索都很难。那么在有了一个方向之后却没有任何头绪时该如何去入手这样一个问题？**谈下我自己在这种情况下做法：去看log，从log中找网络相关的信息，然后根据关键日志找到那个代码的位置，从这里开始可以往前往后捋流程了，不清楚的地方就添加log来印证自己的猜测，直到找到头和尾或是得到你想要的结果**。当然还有个快速的方法是去网上找介绍这个模块流程的文章，过滤你想要的东西。而这个问题我正是结合了这两种方法。下面就会开始讲解开机时网络初始化时相关的代码流程。  

### 正常流程

&emsp;&emsp;我这边只介绍和这个问题相关的流程，无法涵盖整个网络管理，但是可以保证是一条完整的线走下来，看完之后就觉得这是一个圆满的故事。而这个故事的完整标题应该是“以太网络初始化时dhcp的相关流程”。先来解释下整个故事的大纲，其实这个问题的本质就是在以太网（也就是车机里的tbox网络）在初始化时需要通过dhcp去解析网络相关信息，解析完整之后设置到系统属性值里，设置完之后同时也会设置一个result（前面提到的dhcp.tbox0.result）为ok，然后这些属性值会被读取，返回给到framework层的网络相关服务，最后更新网络连接正常的状态。一点点来看吧。  

#### Framework

&emsp;&emsp;Framework层中可以以ConnectivityService为起点，此服务为网络连接的相关服务，在SystemServer中启动，在启动是来看它的构造函数。  
```Java
//ConnectivityService->ConnectivityService
// 主线流程1-1 ConnectivityService启动
public ConnectivityService(Context context, INetworkManagementService netManager,
            INetworkStatsService statsService, INetworkPolicyManager policyManager,
            NetworkFactory netFactory) {
	...
	// Create and start trackers for hard-coded networks
    for (int targetNetworkType : mPriorityList) {
        final NetworkConfig config = mNetConfigs[targetNetworkType];
        final NetworkStateTracker tracker;
        try {
            tracker = netFactory.createTracker(targetNetworkType, config);
            mNetTrackers[targetNetworkType] = tracker;
        } catch (IllegalArgumentException e) {
            Slog.e(TAG, "Problem creating " + getNetworkTypeName(targetNetworkType)
                    + " tracker: " + e);
            continue;
        }
		// 主线流程1-2 开始监听EthernetDataTrack
        tracker.startMonitoring(context, mTrackerHandler);
        if (config.isDefault()) {
             tracker.reconnect();
        }
    }
    ...
```
```Java
//EthernetDataTrack->startMonitoring
public void startMonitoring(Context context, Handler target) {
	...
	try {
		//支线流程1-1： 从NetworkManagementService中获取当前的interface列表
        final String[] ifaces = mNMService.listInterfaces();
        for (String iface : ifaces) {
            if (iface.matches(sIfaceMatch)) {
                mIface = iface;
                mNMService.setInterfaceUp(iface);
                InterfaceConfiguration config = mNMService.getInterfaceConfig(iface);
                if (config != null && mHwAddr == null) {
                    mHwAddr = config.getHardwareAddress();
                    if (mHwAddr != null) {
                        mNetworkInfo.setExtraInfo(mHwAddr);
                    }
                }

                // if a DHCP client had previously been started for this interface, then stop it
                Log.d(TAG, "===========startMonitoring stopDHCP");
                // 主线流程1-3： 停止dhcp
                NetworkUtils.stopDhcp(mIface);
				// 主线流程1-4： 重连网络
                reconnect();
            }
        }
    } catch (RemoteException e) {
        Log.e(TAG, "Could not get list of interfaces " + e);
    }

    try {
    	// 主线流程1-5
        mNMService.registerObserver(mInterfaceObserver);
    } catch (RemoteException e) {
        Log.e(TAG, "Could not register InterfaceObserver " + e);
    }
	...
}
```
&emsp;&emsp;启动了startMonitoring之后先来看下支线流程1-1这个地方，是从NetworkManagementService中获取当前存在的网络接口，也就是说判断当前已经起来的网卡，包括了wlan，tbox等，方法的内容就不深入下去了，介绍下如何从log中来判断某个网卡是否已经起来了。可以查看如下信息：  
```xml
I SystemServer: Connectivity Service
D CommandListener: Setting iface cfg
D CommandListener: Trying to bring up tbox0
```
&emsp;&emsp;这个CommandListener是来自netd的消息，也就是从kernel传上来的通过netd告诉framework层tbox起来了，可以去对比tbox0起来的这个时机和ConnectivityService起来的时机。如果tbox是在ConnectivityService之后起来的，那么从log里能看到其实主线流程1-3和1-4是不走的，也就是支线流程1-1这边取到的interface list中并不包含tbox，下面就走不到了。后面主线流程1-5会把EthernetDataTrack注册到NetworkManagementService中去。  
```Java
// EthernetDataTrack->registerObserver
@Override
public void registerObserver(INetworkManagementEventObserver observer) {
    mContext.enforceCallingOrSelfPermission(CONNECTIVITY_INTERNAL, TAG);
    Slog.d(TAG, " registerObserver :" + observer);
    mObservers.register(observer);
}
```
&emsp;&emsp;以上工作是开机时服务的初始化和注册的工作，而主线流程1-5这里注册的作用就是让NetworkManagementService知道有EthernetDataTrack这个监听器，后面有网络状态的变化，就会通知各个注册的监听器。准备工作完成了，然后就来看下此时来自底层的网络状态变化到来时，framework中是如何去做相应事情的。  
&emsp;&emsp;其实这块流程最初我也是从日志中发现有NetworkManagementService相关的内容如下：  
```xml
D NetworkManagementService:  onEvent========= code:600 raw:600 Iface linkstate tbox0 up cooked:[Ljava.lang.String;@41d4d080
D NetworkManagementService:  linkstate:tbox0 :up
D NetworkManagementService: ======== notifyInterfaceLinkStateChanged iface:tbox0 up:true length:8
```
&emsp;&emsp;然后去跟了下代码，把这块的流程就衔接起来了。有了解过daemon进程和framework通信方式的话，看到onEvent这样的关键字应该是不会陌生。这里的方式就是netd接收kernel的事件，然后通过socket来通知framework的服务，同样的方式也存在与vold和MountService之间，vold感知来自kernel的usb状态变化，然后通知给MountService发送广播通知各个注册器。这里不对netd里的代码做剖析，也没什么必要，了解这个机制就够了，那么就从NetworkManagementService中接收netd的消息开始。 
```Java
// NetworkManagementService->onEvent
// 主线流程2-1： NMS收到netd的event
@Override
public boolean onEvent(int code, String raw, String[] cooked) {
    Slog.d(TAG, " onEvent========= code:" + code + " raw:" + raw + " cooked:" + cooked);
    switch (code) {
    case NetdResponseCode.InterfaceChange:
            /*
             * a network interface change occured
             * Format: "NNN Iface added <name>"
             *         "NNN Iface removed <name>"
             *         "NNN Iface changed <name> <up/down>"
             *         "NNN Iface linkstatus <name> <up/down>"
             */
            if (cooked.length < 4 || !cooked[1].equals("Iface")) {
                throw new IllegalStateException(
                        String.format("Invalid event from daemon (%s)", raw));
            }
            if (cooked[2].equals("added")) {
                notifyInterfaceAdded(cooked[3]);
                return true;
            } else if (cooked[2].equals("removed")) {
                notifyInterfaceRemoved(cooked[3]);
                return true;
            } else if (cooked[2].equals("changed") && cooked.length == 5) {
                notifyInterfaceStatusChanged(cooked[3], cooked[4].equals("up"));
                return true;
            } else if (cooked[2].equals("linkstate") && cooked.length == 5) {
                Slog.d(TAG, " linkstate:" +  cooked[3] + " :" + cooked[4]);
                // 主线流程2-2
                notifyInterfaceLinkStateChanged(cooked[3], cooked[4].equals("up"));
                return true;
            }
     ...
```
&emsp;&emsp;从log里看这里是linkstate的事件，那么接着来跟notifyInterfaceLinkStateChanged这个方法。  
```Java
// NetworkManagementService->notifyInterfaceLinkStateChanged
/**
 * Notify our observers of an interface link state change
 * (typically, an Ethernet cable has been plugged-in or unplugged).
 */
private void notifyInterfaceLinkStateChanged(String iface, boolean up) {
    final int length = mObservers.beginBroadcast();
    Slog.d(TAG, "======== notifyInterfaceLinkStateChanged iface:" + iface + " up:" + up + " length:" +length);
    for (int i = 0; i < length; i++) {
        try {
          	// 主线流程2-3
            mObservers.getBroadcastItem(i).interfaceLinkStateChanged(iface, up);
        } catch (RemoteException e) {
            Slog.e(TAG, " notifyInterfaceLinkStateChange remoteException:", e);
        } catch (RuntimeException e) {
            Slog.e(TAG, " notifyInterfaceLinkStateChange RuntimeException:", e);
        }
    }
    mObservers.finishBroadcast();
}
```
&emsp;&emsp;这里就是获取到前面主线流程1-5中注册的监听器，然后通过注册的回调方法来通知了，那么走到的地方肯定就是EthernetDataTrack里的interfaceLinkStateChanged  
```Java
@Override
// EthernetDataTrack->interfaceLinkStateChanged
public void interfaceLinkStateChanged(String iface, boolean up) {
    Log.d(TAG, "========= interfaceLinkStateChange mIface:" + mIface + " iface:" + iface);

    if (mIface.equals(iface)) {
        Log.d(TAG, "Interface " + iface + " link " + (up ? "up" : "down"));
        mLinkUp = up;
        mNfsmode = "yes".equals(SystemProperties.get("ro.nfs.mode", "no"));
        mAlwayson = "yes".equals(SystemProperties.get("ro.ethernet.alwayson.mode", "false"));
        mTracker.mNetworkInfo.setIsAvailable(up);

        // use DHCP
        if (up) {
            if (mAlwayson) {
                mEthernetWakeLock.acquire();
            }
            // 主线流程2-4： 准备发起网络重连
            mTracker.reconnect();
        } else {
            if (mAlwayson)
                mEthernetWakeLock.release();
            // 支线流程2-1： 在收到down的时候去断开网络
            mTracker.disconnect();
        }
    }
}
```
```Java
// EthernetDataTrack->reconnect
public boolean reconnect() {
    Log.d(TAG, "=========reConnet==========mLinkUp:" + mLinkUp);
    if (mLinkUp) {
        mTeardownRequested.set(false);
        // 主线流程2-5： framework准备发起dhcp请求
        runDhcp();
    }
    return mLinkUp;
}
```
```Java
// EthernetDataTrack->runDhcp
private void runDhcp() {
    Log.d(TAG, "=========runDhcp==========");
    Thread dhcpThread = new Thread(new Runnable() {
        public void run() {
        	// 用来保存dhcp请求完成之后返回上来的结果（网卡的ip address、mask、dns等信息）
            DhcpResults dhcpResults = new DhcpResults();

			// 主线流程2-7： 通向native的runDhcp，传入的是interface名称和保存结果的引用
            if (!NetworkUtils.runDhcp(mIface, dhcpResults)) {
                Log.e(TAG, "====================DHCP request error:" + NetworkUtils.getDhcpError());
                return;
            }

            mLinkProperties = dhcpResults.linkProperties;
            Log.d(TAG, "=========runDhcp=mLinkeProp:" + mLinkProperties);

			// 主线流程2-8： dhcp正常完成之后设置网络状态并做消息通知
            mNetworkInfo.setIsAvailable(true);
            mNetworkInfo.setDetailedState(DetailedState.CONNECTED, null, mHwAddr);
            Message msg = mCsHandler.obtainMessage(EVENT_STATE_CHANGED, mNetworkInfo);
            msg.sendToTarget();
            }
        });
    // 主线流程2-6： 启动线程去申请做dhcp
    dhcpThread.start();
    }
```
&emsp;&emsp;以上就是在收到网络up的消息之后，framework需要发起dhcp请求，然后由dhcp的服务端去做解析，如果正常返回，framework这边就会保存解析完成之后的网络信息并通知网络连接正常的消息。主线流程2-8之后的内容这边暂时也不涉及到。那么接下来看主线流程2-7调下去之后的流程。  

#### native   

&emsp;&emsp;主线流程2-7调用到的是native方法，代码位于frameworks/base/core/jni/android_net_NetUtils.cpp中  
```cpp
// 主线流程3-1： NetworkUtils.runDhcp调用到的native函数
// android_net_NetUtils.cpp->android_net_utils_runDhcp
static jboolean android_net_utils_runDhcp(JNIEnv* env, jobject clazz, jstring ifname, jobject info)
{
	// 主线流程3-2
    return android_net_utils_runDhcpCommon(env, clazz, ifname, info, false);
}
```
```cpp
// android_net_NetUtils.cpp->android_net_utils_runDhcpCommon
static jboolean android_net_utils_runDhcpCommon(JNIEnv* env, jobject clazz, jstring ifname,
        jobject dhcpResults, bool renew)
{
    int result;
    char  ipaddr[PROPERTY_VALUE_MAX];
    uint32_t prefixLength;
    char gateway[PROPERTY_VALUE_MAX];
    char    dns1[PROPERTY_VALUE_MAX];
    char    dns2[PROPERTY_VALUE_MAX];
    char    dns3[PROPERTY_VALUE_MAX];
    char    dns4[PROPERTY_VALUE_MAX];
    const char *dns[5] = {dns1, dns2, dns3, dns4, NULL};
    char  server[PROPERTY_VALUE_MAX];
    uint32_t lease;
    char vendorInfo[PROPERTY_VALUE_MAX];
    char domains[PROPERTY_VALUE_MAX];
    char mtu[PROPERTY_VALUE_MAX];

    const char *nameStr = env->GetStringUTFChars(ifname, NULL);
    if (nameStr == NULL) return (jboolean)false;

    if (renew) {// 这里开机情景下为renew = 0
        result = ::dhcp_do_request_renew(nameStr, ipaddr, gateway, &prefixLength,
                dns, server, &lease, vendorInfo, domains, mtu);
    } else {
    	// 主线流程3-3： 走到libnetutils库里去做dhcp request
        result = ::dhcp_do_request(nameStr, ipaddr, gateway, &prefixLength,
                dns, server, &lease, vendorInfo, domains, mtu);
    }
    if (result != 0) {// 底层的dhcp失败
        ALOGD("dhcp_do_request failed : %s (%s)", nameStr, renew ? "renew" : "new");
    }
	
	// 主线流程3-4： 如果底层dhcp成功，result为0，那么就将ipaddr、gateway等信息填入到dhcpResults，通过环境变量带回给framework层
    env->ReleaseStringUTFChars(ifname, nameStr);
    if (result == 0) {
        env->CallVoidMethod(dhcpResults, dhcpResultsFieldIds.clear);

        // set mIfaceName
        // dhcpResults->setInterfaceName(ifname)
        env->CallVoidMethod(dhcpResults, dhcpResultsFieldIds.setInterfaceName, ifname);

        // set the linkAddress
        // dhcpResults->addLinkAddress(inetAddress, prefixLength)
        result = env->CallBooleanMethod(dhcpResults, dhcpResultsFieldIds.addLinkAddress,
                env->NewStringUTF(ipaddr), prefixLength);
    }

    if (result == 0) {
        // set the gateway
        // dhcpResults->addGateway(gateway)
        result = env->CallBooleanMethod(dhcpResults,
                dhcpResultsFieldIds.addGateway, env->NewStringUTF(gateway));
    }

    if (result == 0) {
        // dhcpResults->addDns(new InetAddress(dns1))
        result = env->CallBooleanMethod(dhcpResults,
                dhcpResultsFieldIds.addDns, env->NewStringUTF(dns1));
    }

    if (result == 0) {
        env->CallVoidMethod(dhcpResults, dhcpResultsFieldIds.setDomains,
                env->NewStringUTF(domains));

        result = env->CallBooleanMethod(dhcpResults,
                dhcpResultsFieldIds.addDns, env->NewStringUTF(dns2));

        if (result == 0) {
            result = env->CallBooleanMethod(dhcpResults,
                    dhcpResultsFieldIds.addDns, env->NewStringUTF(dns3));
            if (result == 0) {
                result = env->CallBooleanMethod(dhcpResults,
                        dhcpResultsFieldIds.addDns, env->NewStringUTF(dns4));
            }
        }
    }

    if (result == 0) {
        // dhcpResults->setServerAddress(new InetAddress(server))
        result = env->CallBooleanMethod(dhcpResults, dhcpResultsFieldIds.setServerAddress,
                env->NewStringUTF(server));
    }

    if (result == 0) {
        // dhcpResults->setLeaseDuration(lease)
        env->CallVoidMethod(dhcpResults,
                dhcpResultsFieldIds.setLeaseDuration, lease);

        // dhcpResults->setVendorInfo(vendorInfo)
        env->CallVoidMethod(dhcpResults, dhcpResultsFieldIds.setVendorInfo,
                env->NewStringUTF(vendorInfo));
    }
    return (jboolean)(result == 0);
}
```
&emsp;&emsp;native的东西比较简单，就是申请了几个变量，通过主线流程3-4传给libnetutils库里去做dhcp request，如果底层的dhcp成功的话，这几个字段就会被填充入正确的信息，同时负责将这些内容填充给dhcpResults，然后送回给framework中去。  

#### libnetutils  

&emsp;&emsp;从这里开始才是真正的dhcp业务的核心内容，代码位于system/core/libnetutils目录下，上面主线流程3-4会调用到dhcp_utils.c中去。  
```C
int dhcp_do_request(const char *interface,
                    char *ipaddr,
                    char *gateway,
                    uint32_t *prefixLength,
                    char *dns[],
                    char *server,
                    uint32_t *lease,
                    char *vendorInfo,
                    char *domain,
                    char *mtu)
{
    char result_prop_name[PROPERTY_KEY_MAX];
    char daemon_prop_name[PROPERTY_KEY_MAX];
    char prop_value[PROPERTY_VALUE_MAX] = {'\0'};
    char daemon_cmd[PROPERTY_VALUE_MAX * 2 + sizeof(DHCP_CONFIG_PATH)];
    const char *ctrl_prop = "ctl.start";
    const char *desired_status = "running";
    /* Interface name after converting p2p0-p2p0-X to p2p to reuse system properties */
    char p2p_interface[MAX_INTERFACE_LENGTH];
    // 以下tbox网络为例
    // interface=tbox0，p2p_interface=tbox0
    // 将传入的interface进行统一格式处理
    get_p2p_interface_replacement(interface, p2p_interface);
    // result_prop_name=dhcp.tbox0.result
    // 这里终于看到之前分析时提到过的线索了： dhcp.tbox0.result字段
    snprintf(result_prop_name, sizeof(result_prop_name), "%s.%s.result",
            DHCP_PROP_NAME_PREFIX,
            p2p_interface);
    // daemon_prop_name=init.svc.dhcpcd_tbox0
    snprintf(daemon_prop_name, sizeof(daemon_prop_name), "%s_%s",
            DAEMON_PROP_NAME,
            p2p_interface);

    /* Erase any previous setting of the dhcp result property */
    // 在开始dhcp request之前先清空下之前的结果，如果dhcp.tbox0.result字段还不存在那么这里是无效的
    property_set(result_prop_name, "");

    /* Start the daemon and wait until it's ready */
    // 先去读host是否准备好
    // [net.hostname]: [android-ef3383089cd4ecd]
    if (property_get(HOSTNAME_PROP_NAME, prop_value, NULL) && (prop_value[0] != '\0'))
        //daemon_cmd = dhcpcd_tbox0:-f /system/etc/dhcpcd/dhcpcd.conf -h android-33d27f15b7de052b tbox0
        snprintf(daemon_cmd, sizeof(daemon_cmd), "%s_%s:-f %s -h %s %s", DAEMON_NAME,
                 p2p_interface, DHCP_CONFIG_PATH, prop_value, interface);
    else
        //daemon_cmd = dhcpcd_tbox0:-f /system/etc/dhcpcd/dhcpcd.conf tbox0
        snprintf(daemon_cmd, sizeof(daemon_cmd), "%s_%s:-f %s %s", DAEMON_NAME,
                 p2p_interface, DHCP_CONFIG_PATH, interface);
    memset(prop_value, '\0', PROPERTY_VALUE_MAX);
    // setprop ctrl.start daemon_cmd
    // 主线流程4-1： 启动dhcp的daemon来真正做dhcp
    property_set(ctrl_prop, daemon_cmd);
    ALOGD(" dhcp_do_request daemon_cmd=%s, ", daemon_cmd);
    ALOGD(" dhcp_do_request prop=%s, status=%s ", daemon_prop_name, desired_status);
    // 去读取init.svc.dhcpcd_tbox0的值，目标为读到running就ok
    if (wait_for_property(daemon_prop_name, desired_status, 100) < 0) {
        snprintf(errmsg, sizeof(errmsg), "%s", "Timed out waiting for dhcpcd to start");
        return -1;
    }

    /* Wait for the daemon to return a result */
    // 主线流程4-2： 去读取dhcp.tbox0.result的值，传入的期望值是NULL，如果当该字段存在时理论上只读一次，但是如果不存在的话getprop就会一直失败，一直循环读取
    if (wait_for_property(result_prop_name, NULL, 120) < 0) {
        snprintf(errmsg, sizeof(errmsg), "%s", "Timed out waiting for DHCP to finish");
        return -1;
    }
	
	// 检查dhcp.tbox0.result没有设置
    if (!property_get(result_prop_name, prop_value, NULL)) {
        /* shouldn't ever happen, given the success of wait_for_property() */
        snprintf(errmsg, sizeof(errmsg), "%s", "DHCP result property was not set");
        return -1;
    }
    // dhcp.tbox0.result的值是ok的时候
    if (strcmp(prop_value, "ok") == 0) {
        char dns_prop_name[PROPERTY_KEY_MAX];
        // 主线流程4-3： 将网络信息填充到数据中
        if (fill_ip_info(interface, ipaddr, gateway, prefixLength, dns,
                server, lease, vendorInfo, domain, mtu) == -1) {
            return -1;
        }
        return 0;
    } else {
        snprintf(errmsg, sizeof(errmsg), "DHCP result was %s", prop_value);
        return -1;
    }
}
```
&emsp;&emsp;dhcp_do_request这个函数里做的几件事情归纳下就是：1、构造网络信息相关的字段；2、启动dhcp服务端；3、等待dhcp服务端的结果(dhcp.tbox0.result)；4、result为ok的情况下填充网络信息，这个会送回给native。这里有一个比较关键的函数影响着流程的走向，需要理解下，就是多次出现的wait_for_property。  
```C
static int wait_for_property(const char *name, const char *desired_value, int maxwait)
{
    char value[PROPERTY_VALUE_MAX] = {'\0'};
    int maxnaps = (maxwait * 1000) / NAP_TIME;//NAP_TIME = 200us

    if (maxnaps < 1) {
        maxnaps = 1;
    }
    ALOGD("wait for property name=%s, value=%s, wait loop=%d", name, desired_value, maxnaps);

    while (maxnaps-- > 0) {
    	// 每次睡眠200ms，总共等待时间为maxwait秒
        usleep(NAP_TIME * 1000);
        // 如果property_get读取正常（有字段的情况），那么如果读到了期望值或者是期望值为NULL的时候正常返回
        if (property_get(name, value, NULL)) {
            if (desired_value == NULL || 
                    strcmp(value, desired_value) == 0) {
                return 0;
            }
        }
    }
    // 超时情况
    return -1; /* failure */
}
```
&emsp;&emsp;还有一个填充网络信息的函数fill_ip_info，内容这里不粘贴了，又长又臭，其实思路就是从系统属性值里读取出来内容填充到变量中。那么其实可以推测在dhcp服务端是做了解析网络信息并且写入到系统属性值里面这样的事情。**这整个数据传递的思路其实也可以用来模仿下：守护进程将数据写入到属性值中，so库里的通过读取属性值来填充给native，native通过环境变量带给framework层**。  

#### dhcpcd

&emsp;&emsp;在主线流程4-1中调用了`property_set(ctrl_prop, daemon_cmd);`，是通过属性服务来启动init.rc中的服务，从log看到这里的daemon_cmd的值为`dhcpcd_tbox0:-f /system/etc/dhcpcd/dhcpcd.conf tbox0`，在init.rc中查到dhcpcd_tbox0这个服务的定义如下，是一个oneshot的服务  
```
service dhcpcd_tbox0 /system/bin/dhcpcd -d -ABKL
    class main
    disabled
    oneshot

```
&emsp;&emsp;启动了dhcpcd，这个就是dhcp的服务端，代码位于external/dhcpcd。启动该进程时首先会走到dhcpcd.c的main函数  
```c
//dhcpcd.c->main
int
main(int argc, char **argv)
{
...
//主线流程5-1
//遍历可用的iface，来做初始化，调用init_state
for (iface = ifaces; iface; iface = iface->next) {
	init_state(iface, argc, argv);
	if (iface->carrier != LINK_DOWN)
		opt = 1;
}
...
```
```c
//dhcpcd.c->init_state
static void
init_state(struct interface *iface, int argc, char **argv)
{
	struct if_state *ifs;

	if (iface->state)
		ifs = iface->state;
	else
		ifs = iface->state = xzalloc(sizeof(*ifs));

	ifs->state = DHS_INIT;
	ifs->reason = "PREINIT";
	ifs->nakoff = 1;
	configure_interface(iface, argc, argv);
	if (!(options & DHCPCD_TEST))
		// 主线流程5-2，调用run_script，reason是PREINIT
		run_script(iface);
	...
}
```
&emsp;&emsp;在configure.h里有个宏`configure.h:39:#define run_script(ifp) run_script_reason(ifp, (ifp)->state->reason);`，所以run_script其实是调用的configure.c里的run_script_reason。  
```c
//configure.c->run_script_reason
int
run_script_reason(const struct interface *iface, const char *reason)
{
...
// 主线流程5-3
pid = exec_script(argv, env);
...
```
```c
static int
exec_script(char *const *argv, char *const *env)
{
	pid_t pid;
	sigset_t full;
	sigset_t old;

	/* OK, we need to block signals */
	sigfillset(&full);
	sigprocmask(SIG_SETMASK, &full, &old);
	signal_reset();

	switch (pid = vfork()) {//forc一个进程
	case -1:
		syslog(LOG_ERR, "vfork: %m");
		break;
	case 0:
		sigprocmask(SIG_SETMASK, &old, NULL);
		//主线流程5-4：去执行传入的命令，其实就是跑脚本
		execve(argv[0], argv, env);
		syslog(LOG_ERR, "%s: %m", argv[0]);
		_exit(127);
		/* NOTREACHED */
	}

	/* Restore our signals */
	signal_setup();
	sigprocmask(SIG_SETMASK, &old, NULL);
	return pid;
}
```
&emsp;&emsp;而运行的脚本是位于代码目录下的`external/dhcpcd/dhcpcd-run-hooks`脚本，其实是个钩子  
```shell
#!/system/bin/sh
# dhcpcd client configuration script 

# Handy variables and functions for our hooks to use
from="from"
signature_base="# Generated by dhcpcd"
signature="${signature_base} ${from} ${interface}"
signature_base_end="# End of dhcpcd"
signature_end="${signature_base_end} ${from} ${interface}"
state_dir="/data/misc/dhcpcd"

# We source each script into this one so that scripts run earlier can
# remove variables from the environment so later scripts don't see them.
# Thus, the user can create their dhcpcd.enter/exit-hook script to configure
# /etc/resolv.conf how they want and stop the system scripts ever updating it.
for hook in \
	/system/etc/dhcpcd/dhcpcd.enter-hook \
	/system/etc/dhcpcd/dhcpcd-hooks/* \
	/system/etc/dhcpcd/dhcpcd.exit-hook
do
	for skip in ${skip_hooks}; do
		case "${hook}" in
			*/"${skip}")			continue 2;;
			*/[0-9][0-9]"-${skip}")		continue 2;;
			*/[0-9][0-9]"-${skip}.sh")	continue 2;;
		esac
	done
	if ls "${hook}" >/dev/null 2>&1; then
		. "${hook}"
	fi
done
```
&emsp;&emsp;它会去把/system/etc/dhcpcd/dhcpcd-hooks/下面的所有脚本跑起来，而这个目录下的脚本包括了external/dhcpcd/dhcpcd-hooks/20-dns.conf和external/dhcpcd/dhcpcd-hooks/95-configured。20-dns.conf的作用是配置网卡的dns，例如dhcp.tbox0.dns1。95-configured脚本如下  
```
# This script runs last, after all network configuration
# has completed. It sets a property to let the framework
# know that setting up the interface is complete.

if [[ $interface == p2p* ]]
    then
    intf=p2p
    else
    intf=$interface
fi

# For debugging:
setprop dhcp.${intf}.reason "${reason}"

case "${reason}" in
BOUND|INFORM|REBIND|REBOOT|RENEW|TIMEOUT)
    setprop dhcp.${intf}.ipaddress  "${new_ip_address}"
    setprop dhcp.${intf}.gateway    "${new_routers%% *}"
    setprop dhcp.${intf}.mask       "${new_subnet_mask}"
    setprop dhcp.${intf}.leasetime  "${new_dhcp_lease_time}"
    setprop dhcp.${intf}.server     "${new_dhcp_server_identifier}"
    setprop dhcp.${intf}.vendorInfo "${new_vendor_encapsulated_options}"
    setprop dhcp.${intf}.mtu        "${new_interface_mtu}"

    setprop dhcp.${intf}.result "ok"
    ;;

EXPIRE|FAIL|IPV4LL|STOP)
    setprop dhcp.${intf}.result "failed"
    ;;

RELEASE)
    setprop dhcp.${intf}.result "released"
    ;;
esac
```
&emsp;&emsp;当传入的reason符合条件时，会去设置系统属性值中网络相关的信息，然后将result设置为ok。这里就是属性值中网络信息的来源和result值的来源。  
&emsp;&emsp;这次跑脚本时传入的reason和PREINIT，所以这一不会去设置网络信息，中间跳过一些不重要的流程，然后会从dhcpcd.c里走到dhcpcd的bind.c里的bind_interface  
```c
//bind.c->bind_interface
void
bind_interface(void *arg)
{
...
if (state->reason == NULL) {
	if (state->old) {
		if (state->old->yiaddr == state->new->yiaddr &&
		    lease->server.s_addr)
			state->reason = "RENEW";
		else
			state->reason = "REBIND";
	} else if (state->state == DHS_REBOOT)
		state->reason = "REBOOT";//走到这里，reason赋值为REBOOT
	else
		state->reason = "BOUND";
}
...
// 主线流程5-5
configure(iface);
...
```

```c
int
configure(struct interface *iface)
{
...
run_script(iface);
return 0;
}
```
&emsp;&emsp;这里又去执行了脚本，而且传入的reason是REBOOT，所以会去设置网络信息，并且设置result为ok，到这里这套主线流程就通了。  

### 异常情况  

&emsp;&emsp;介绍下这个问题里出现概率最高的一个现象，就是tbox上报网络状态的时序为down/up/up/up，在收到连续up的时候每次会去做dhcp请求，还记得在dhcp_do_request函数里会先去把result清空的动作吗？没错，出现连续的up时，问题就是这样发生的，首先清空result，然后去启动dhcpcd的时候，因为它是oneshot的，之前没有stop的情况下不会再起来，因此其实dhcp request是没有做的，所以result不会被再设置为ok，那么dhcp_do_request里一直在等读取result的值，但是property_get一直fail，超时之后就返回的不是正确的网络信息，而是超时异常，所以最终framework的网络服务就判断为dhcp失败，也就不会去配置网络。那么为什么ip地址是可以ping通的？因为在第一次dhcp的时候网卡信息就是ok的了，而dhcpcd里也是会配置路由，所以第二次dhcp请求只是清空了result，更直观的说只是影响了系统dns的配置，所以ip链路是正确的。  

### 解决方法  

&emsp;&emsp;在libnetutils库里，调用dhcp_do_request的时候在清空result之前去做一把stop dhcp的动作，调用这里的dhcp_stop就可以了。  







