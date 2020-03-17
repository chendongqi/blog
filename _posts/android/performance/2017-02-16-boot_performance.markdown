---
layout:      post
title:      "Android性能优化之Boot Performance"
subtitle:   "Android整机项目中开机速度优化"
navcolor:   "invert"
date:       2017-02-16
author:     "Cheson"
catalog: true
tags:
    - Android
    - Performance
---

### 代码和平台版本

Android Version： 6.0
Platform： MTK MT6735/6737

### 1. 首次开机时间优化

#### 1.1 检查默认加密

Android L平台的关闭方式参照[MTK FAQ12148](https://onlinesso.mediatek.com/Pages/FAQ.aspx?List=SW&FAQID=FAQ14128)
> 关闭加密功能有两种情况：
1 How to disable default encryption in your own image
(1) Modify fstab.{ro.hardware} in ‘out’ folder
alps\out\target\product\[project]\root\ fstab.{ro.hardware}
Set the flag back to encryptable for /data
![mtk_fstab1](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/mtk_fstab1.gif)
(2) Re-pack boot.img
make ramdisk-nodeps; make bootimage-nodpes
(3) Download the new boot.img by flashtool
2 How to disable default encryption in your codebase
a) Modify fstab.{ro.hardware} in your codebase
device\mediatek\ [project]\ fstab.{ro.hardware}
If the project doesn’t have it own fstab.{ro.hardware} . Please create it Modify device.mk to use the modified fstab.{ro.hardware}.
![mtk_fstab2](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/mtk_fstab2.gif)
Set the flag back to encryptable for /data
![mtk_fstab3](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/mtk_fstab3.gif)
b) Re-build boot.img
make bootimage
c) Download the new boot.img by flashtool

Android M平台上修改了默认关闭加密的方式，以MT6580平台为例
检查文件： vendor/mediatek/proprietary/hardware/fstab/mt6580/fstab.in

```Bash
/* Can overwrite FDE setting by defining __MTK_FDE_NO_FORCE and __MTK_FDE_TYPE_FILE in this file */
/* For example, you can un-comment the following line to disable FDE for all projects in this platform. */
#define __MTK_FDE_NO_FORCE /* disable FDE because AES crypto performance is about 22.5 MiB/sec */
#ifdef __MTK_FDE_NO_FORCE
  #define FLAG_FDE_AUTO encryptable
#else
  #define FLAG_FDE_AUTO forceencrypt
#endif
#ifdef __MTK_FDE_TYPE_FILE
  #define FLAG_FDE_TYPE fileencryption
#else
  #define FLAG_FDE_TYPE
#endif
/dev/block/platform/mtk-msdc.0/11120000.msdc0/by-name/system     /system      __MTK_SYSIMG_FSTYPE   ro	        wait
/dev/block/platform/mtk-msdc.0/11120000.msdc0/by-name/userdata   /data        ext4   noatime,nosuid,nodev,noauto_da_alloc,discard               wait,check,resize,FLAG_FDE_AUTO=/dev/block/platform/mtk-msdc.0/11120000.msdc0/by-name/metadata,FLAG_FDE_TYPE
```
当__MTK_FDE_NO_FORCE被define时，就关闭了默认加密功能，所以此平台上默认就是关闭加密功能的

#### 1.2 检查odex优化是否开启

参考[MTK FAQ14131]（https://onlinesso.mediatek.com/Pages/FAQ.aspx?List=SW&FAQID=FAQ14131）
要点如下：

1. 打开WITH_DEXPREOPT := true宏

2. 若user版本未提取odex，DONT_DEXPREOPT_PREBUILTS := true  //此句注释掉

3. 针对64位芯片，只能作为32位运行的apk需要在预置时加入LOCAL_MULTILIB :=32

4. 如果某应用不想使用odex优化，则可以在\build\core\dex_preopt_odex_install.mk做例外处理

5. 开启odex优化，如果预置过多apk则会导致system.img过大，编译不过，则需要修改system分区

检查第一次开机log，搜索PackageManager.DexOptimizer: Running dexopt (dex2oat) on，看看是否还有未优化的apk。

#### 1.3 检查是否关闭patchodex功能

尽管开启了odex优化功能，但是在首次开机时，android会去做patchodex的动作，对odex稍作修改帮放到data/dalvik/&isa目录下。可以采用google的方案，开启WITH_DEXPREOPT_PIC：=true，这样既可以加速，又可以减少data分区。典型的问题log如下：
![patchodex](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/patchodex.png)

#### 1.4 对大型apk的编译优化

目前有些apk例如facebook，微信等，apk较大且代码复杂度较高，往往安装慢，是因为在dex2oat中编译慢，M中处理的方法是加入到白名单中，让它编译的时候做的简单些。
参考FAQ：[MTK FAQ15597](https://onlinesso.mediatek.com/Pages/FAQ.aspx?List=SW&FAQID=FAQ15597)
![big_apk1](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/big_apk1.png)
![big_apk2](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/big_apk2.png)

#### 1.5 检查kernel配置是否有问题

查看bootprof，如果发现kernel启动时间较长，如7秒左右，其他每个启动阶段耗时都较长，可以让驱动排查下kernel是否配置成了eng版本

### 2. 首次和正常开机都有影响的检查项

#### 2.1 检查启动过程中PMS扫描的时间

搜索log关键字scan package，查找是否有elapsed time>100ms的apk，这类apk需要减少

#### 2.2 开机动画包，图片多或者占用内存多，会影响到开机时间

在kernel_log.boot中是否出现lowmemory，搜索关键字

```java
[239.165493]<3>.(1)[70:kswapd0]lowmemorykiller: Candidate 432 (bootanimation), adj -18, score_adj -1000, rss 156937, rswap 2351, to kill
```
Solution：尽量控制开机动画包中的图片数量及每张图片的大小，part1部分循环播放的图片数量需要控制在10张以内

#### 2.3 在开机动画阶段检查是否出现camera I2C transfer timeout

搜索log: 

```java
[292:mediaserver][mt-i2c]ERROR, 484: id=0, addr:10, transfer timeout
```
如果出现此问题，需要排查camera加载慢

#### 2.4 排查media.camera.proxy服务

在升级到M版本之后，谷歌在camera新增了一个叫“media.camera.proxy”的service，在开机过程中会去连接该service。当连接不上时会try 5次，持续5秒左右。影响开机的performance（实测并无优化）。
如下是连接不上的Log：

```java
01940 01-01 08:35:59.563987   222   222 I ServiceManager: Waiting for service media.camera.proxy...
02086 01-01 08:36:00.564399   222   222 I ServiceManager: Waiting for service media.camera.proxy...
02294 01-01 08:36:01.564777   222   222 I ServiceManager: Waiting for service media.camera.proxy...
02387 01-01 08:36:02.565194   222   222 I ServiceManager: Waiting for service media.camera.proxy...
02494 01-01 08:36:03.565630   222   222 I ServiceManager: Waiting for service media.camera.proxy...
```
解决方案：
可以打开/frameworks/av/services/camera/libcameraservice/CameraService.cpp找到pingCameraServiceProxy这个函数，将

```java
sp<IBinder> binder = sm->getService(String16("media.camera.proxy"));
```
改为

```java
sp<IBinder> binder = sm->checkService(String16("media.camera.proxy"));
```

#### 2.5 检查keyguard和launcher初始化时间

![keyguard_boot](https://chendongqi.github.io/blog/img/2017-02-16-boot_performance/keyguard_boot.png)

#### 2.6 kernel部分优化

参考提交记录aa91be1d3dc52a24835bcc25eb7af600ffeb6d38
// 这支文件中取出掉tp初始化的log
kernel-3.18 / drivers/input/touchscreen/mediatek/hx8527_zal1066/himax_852xES.c
// 这里减少tp初始化时的休眠时间
kernel-3.18 / drivers/input/touchscreen/mediatek/hx8527_zal1066/himax_platform.c
// 后面没怎么懂

#### 2.7 检查开机时cpu是否全速运转

烧eng或者userdebug的boot，然后在开机时查看CPU的运行频率：

```Bash
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
```
查看cpu1~3是否online：

```Bash
cat /sys/devices/system/cpu/cpu1/online
```

### 3. 更深层次的排查

#### 3.1 kernel启动时间检查

* kernel init时间长，需要先看一下版本上init.rc文件相对DriverOnly版本是否有添加新的init，这些是否都是必须添加的。在uartlog中，需要排查关键字-----[cut here]-----，找到在kernel init过程中频繁打出的这些call stack，看这些call stack，排查下客制化的点。
* 在uartlog中排查驱动设备初始化是否有完成或延时较长。
* kernel启动时间加长，可以先价差emmc性能，如果emmc质量不好也会影响到performance，memory写速度慢的时候，开机速度也会变慢。如果emmc性能良好在进一步排查其他原因。

#### 3.2 zygote启动时间排查

MTK平台的bootprof中，或者从mailog.boot（mtk平台的开机段main log）中可以找到zygote启动时的主要耗时信息，如下：

```java
Zygote  : Preloading classes...
Zygote  : ...preloaded 3831 classes in 1231ms.
Zygote  : ...preloaded 342 resources in 570ms.
Zygote  : ...preloaded 41 resources in 11ms.
Zygote  : System server process 907 has been created
Zygote  : end preload
```
一般有两个耗时点：

1. 预加载class/resource的时间

需要确认是否有添加很多系统资源，preload classes在frameworks/base/preloaded-classes中定义，适当减少不需要的预加载类预资源。此处需要进一步了解--如何判断哪些为不必要的资源？

2. 这期间是否有很多GC的动作

```C
art     : Starting a blocking GC Explicit
art     : Explicit concurrent mark sweep GC freed 393(51KB) AllocSpace objects, 6(2MB) LOS objects, 44% free, 4MB/8MB, paused 354us total 22.183ms
art     : Starting a blocking GC HeapTrim
art     : Starting a blocking GC Background
```
以上log为Android切到ART虚拟机后的最新GC log形式(之前为dalvikvm打出的log，因为版本过老这里不列出)，这里需要进一步了解的一个标准--什么情况下判定为频繁GC，又该如何优化？

#### 3.3 SystemServer启动阶段排查

在Syslog.boot中删选出如下log

```java
01-01 08:41:56.038416   907   907 I SystemServer: Entered the Android system server!
01-01 08:41:56.567401   907   907 I SystemServer: Recovery Manager
01-01 08:41:56.583414   907   907 I SystemServer: Package Manager
01-01 08:42:02.298454   907   907 I SystemServer: User Service
01-01 08:42:02.763182   907   907 I SystemServer: Reading configuration...
01-01 08:42:02.763269   907   907 I SystemServer: Scheduling Policy
01-01 08:42:02.768264   907   907 I SystemServer: Telephony Registry
01-01 08:42:02.785559   907   907 I SystemServer: Entropy Mixer
01-01 08:42:02.804014   907   907 I SystemServer: Camera Service
01-01 08:42:02.814142   907   907 I SystemServer: Account Manager
01-01 08:42:02.823328   907   907 I SystemServer: Content Manager
01-01 08:42:02.830051   907   907 I SystemServer: System Content Providers
01-01 08:42:02.892894   907   907 I SystemServer: Vibrator Service
01-01 08:42:02.895958   907   907 I SystemServer: Consumer IR Service
01-01 08:42:02.931174   907   907 I SystemServer: Init ECIDManagerService
01-01 08:42:02.931255   907   907 I SystemServer: Init Watchdog
01-01 08:42:02.931907   907   907 I SystemServer: Input Manager
01-01 08:42:02.937843   907   907 I SystemServer: Window Managerart     : Starting a blocking GC Background
01-01 08:42:03.099417   907   907 I SystemServer: Bluetooth Service
01-01 08:42:03.129069   907   907 I SystemServer: Input Method Service
01-01 08:42:03.197145   907   907 I SystemServer: Accessibility Manager
01-01 08:42:03.281191   907   907 I SystemServer: LockSettingsService
01-01 08:42:03.881840   907   907 I SystemServer: Status Bar
01-01 08:42:03.888389   907   907 I SystemServer: Clipboard Service
01-01 08:42:03.891592   907   907 I SystemServer: NetworkManagement Service
01-01 08:42:03.904503   907   907 I SystemServer: Text Service Manager Service
01-01 08:42:03.923227   907   907 I SystemServer: Network Score Service
01-01 08:42:03.925274   907   907 I SystemServer: NetworkStats Service
01-01 08:42:03.954630   907   907 I SystemServer: NetworkPolicy Service
01-01 08:42:04.439141   907   907 I SystemServer: Connectivity Service
01-01 08:42:04.473060   907   907 I SystemServer: Network Service Discovery Service
01-01 08:42:04.481470   907   907 I SystemServer: Start DataShaping Service
01-01 08:42:04.488136   907   907 I SystemServer: UpdateLock Service
01-01 08:42:04.535355   907   907 I SystemServer: Location Manager
01-01 08:42:04.541617   907   907 I SystemServer: Country Detector
01-01 08:42:04.546307   907   907 I SystemServer: Search Service
01-01 08:42:04.549310   907   907 I SystemServer: Search Engine Service
01-01 08:42:04.552144   907   907 I SystemServer: DropBox Service
01-01 08:42:04.555753   907   907 I SystemServer: Wallpaper Service
01-01 08:42:04.564307   907   907 I SystemServer: Audio Service
01-01 08:42:04.653917   907   907 I SystemServer: Wired Accessory Manager
01-01 08:42:04.684407   907   907 I SystemServer: Serial Service
01-01 08:42:04.753316   907   907 I SystemServer: DiskStats Service
01-01 08:42:04.754758   907   907 I SystemServer: SamplingProfiler Service
01-01 08:42:04.783550   907   907 I SystemServer: NetworkTimeUpdateService
01-01 08:42:04.785135   907   907 I SystemServer: CommonTimeManagementService
01-01 08:42:04.787915   907   907 I SystemServer: CertBlacklister
01-01 08:42:04.794061   907   907 I SystemServer: Assets Atlas Service
01-01 08:42:04.826953   907   907 I SystemServer: Media Router Service
01-01 08:42:04.847138   907   907 I SystemServer: BackgroundDexOptService
01-01 08:42:04.871482   907   907 I SystemServer: PerfService state notifier
01-01 08:42:05.272625   907   907 I SystemServer: Making services ready
01-01 08:42:05.330764   907   907 I SystemServer: WebViewFactory preparation
```
从两个方向去排查，1、在init.rc或者是init.platxxxx.rc中去排查不需要加载的服务并删除掉；2、如果某个service的启动中出现异常或者耗时数据过大，可以找该service的owner协助排查。

### 4. 多线程开机方案

此方案来自展讯平台，在我们的项目中已经移植到MTK平台，并有相关的优化数据

#### 4.1 安装应用的多线程优化方案

优化了首次开机时安装应用的速度，在ZAW600 TECNO项目上的实测数据优化了14s左右，以下该方案的patch：

```java
From 66c67a4d9265b3d36c3dbed7377ca463c7ed7079 Mon Sep 17 00:00:00 2001
From: chendongqi <chendongqi@huaqin.com>
Date: Sat, 23 Jul 2016 17:12:50 +0800
Subject: [PATCH] [性能][开机速度]加入展讯多线程安装apk方案，自测第一次开机速度提升14秒左右

Change-Id: I41f8a495a6b964b952ed972bd88a90a534776124
---

diff --git a/core/java/android/content/pm/PackageParser.java b/core/java/android/content/pm/PackageParser.java
index 0784ed4..af708dd 100644
--- a/core/java/android/content/pm/PackageParser.java
+++ b/core/java/android/content/pm/PackageParser.java
@@ -97,7 +97,20 @@
 
 import java.lang.ref.WeakReference;
 
+/*SPRD:Modify the process of the certificate verification to improve the speed of installing application{@*/
+import java.lang.ref.WeakReference;
+import java.util.Enumeration;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.TimeUnit;
+import java.util.jar.JarEntry;
+import java.util.jar.JarFile;
+
 /** @} */
+// add by chendongqi from SPRD modify for install tuning start
+import java.util.concurrent.ConcurrentLinkedQueue;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.atomic.AtomicInteger;
+// add by chendongqi from SPRD modify for install tuning end
 
 /**
  * Parser for package files (APKs) on disk. This supports apps packaged either
@@ -1054,6 +1067,14 @@
     public void collectManifestDigest(Package pkg) throws PackageParserException {
         pkg.manifestDigest = null;
 
+        // add by chendongqi from SPRD modify for isntall tuning start
+        if(pkg.preManifestDigest != null) {
+            pkg.manifestDigest = pkg.preManifestDigest;
+            pkg.preManifestDigest = null;
+            return;
+        }
+        // add by chendongqi from SPRD modify for isntall tuning end
+
         // TODO: extend to gather digest for split APKs
         try {
             final StrictJarFile jarFile = new StrictJarFile(pkg.baseCodePath);
@@ -1099,6 +1120,15 @@
         }
     }
 
+    /* SPRD: Modify the process of the certificate verification to improve the speed of installing application{@ */
+
+    //modify for install tuning start
+    private static final int CERTIFICAT_VERIFY_OK = 0;
+    private static final int CERTIFICAT_VERIFY_FAIL = 1;
+
+    static class VerifyResult {
+        public volatile int ret = CERTIFICAT_VERIFY_OK;
+    }
     private static void collectCertificates(Package pkg, File apkFile, int flags)
             throws PackageParserException {
         final String apkPath = apkFile.getAbsolutePath();
@@ -1107,8 +1137,195 @@
         try {
             /// TODO: [ALPS01655835][L.AOSP.EARLY.DEV] Bring-up build error on frameworks/base @{
             jarFile = new StrictJarFile(apkPath);
-            /// @}
+            // Always verify manifest, regardless of source
+            final ZipEntry manifestEntry = jarFile.findEntry(ANDROID_MANIFEST_FILENAME);
+            if (manifestEntry == null) {
+                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
+                        "Package " + apkPath + " has no manifest");
+            }
 
+            //modify for intall tuning start
+            pkg.preManifestDigest = ManifestDigest.fromInputStream(jarFile.getInputStream(manifestEntry));
+            //modify for intall tuning end
+
+            //final List<ZipEntry> toVerify = new ArrayList<>();
+            final ConcurrentLinkedQueue<ZipEntry> toVerify = new ConcurrentLinkedQueue<ZipEntry>();
+            //wangsl
+            //toVerify.add(manifestEntry);
+            //System.out.println("wangsl,collectCertificates 1");
+            final AtomicInteger verifyRet = new AtomicInteger(CERTIFICAT_VERIFY_OK);
+            //int verifyRet = CERTIFICAT_VERIFY_OK
+            //final VerifyResult verifyRet = new VerifyResult();
+            //verifyRet.set(CERTIFICAT_VERIFY_OK);
+            
+            final StrictJarFile jarFileForTh = jarFile;
+            final Package pkgForTh = pkg;
+            //check mainfest
+            Thread manifestVerify = new Thread(new Runnable() {
+                @Override
+                public void run() {
+                    //System.out.println("wangsl,collectCertificates 2");
+                    boolean ret = doVerify(jarFileForTh,manifestEntry,pkgForTh);
+                    if(!ret) {
+                        //System.out.println("wangsl,collectCertificates 3");
+                        verifyRet.set(CERTIFICAT_VERIFY_FAIL);
+                        //verifyRet.ret = CERTIFICAT_VERIFY_FAIL;
+                        //System.out.println("wangsl,collectCertificates 4");
+                    }
+                }
+            });
+            manifestVerify.start();
+            //wangsl
+
+            // If we're parsing an untrusted package, verify all contents
+            if ((flags & PARSE_IS_SYSTEM) == 0) {
+                final Iterator<ZipEntry> i = jarFile.iterator();
+                while (i.hasNext()) {
+                    final ZipEntry entry = i.next();
+
+                    if (entry.isDirectory()) continue;
+                    if (entry.getName().startsWith("META-INF/")) continue;
+                    if (entry.getName().equals(ANDROID_MANIFEST_FILENAME)) continue;
+
+                    toVerify.add(entry);
+                }
+            }
+            //System.out.println("wangsl,collectCertificates 5");
+            try {
+                manifestVerify.join();
+            }catch(InterruptedException e) {}
+            
+            if(verifyRet.get() == CERTIFICAT_VERIFY_FAIL) {
+                //System.out.println("wangsl,collectCertificates 6");
+                throw new PackageParserException(
+                        INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
+                                + " has mismatched certificates at entry ");
+                
+            }
+
+            //check others
+            int maxCpu = Runtime.getRuntime().availableProcessors()*2;
+            final CountDownLatch latch = new CountDownLatch(maxCpu);
+            
+            //for(int poison = 0;poison<maxCpu;poison++) {
+            //       final ZipEntry entry = new ZipEntry((String)null);
+            //       toVerify.add(entry);//add po
+            //}
+
+            //System.out.println("wangsl,collectCertificates 7");
+            for(int i = 0;i<maxCpu;i++) {
+                new Thread(new Runnable() {
+                    @Override
+                    public void run() {
+                        while(true) {
+                            if(verifyRet.get() == CERTIFICAT_VERIFY_FAIL) {
+                            //if(verifyRet.ret == CERTIFICAT_VERIFY_FAIL) {
+                                latch.countDown();
+                                return;
+                            }
+                            
+                            ZipEntry entry = toVerify.poll();
+                            if(entry == null) {
+                                latch.countDown();
+                                return;
+                            }
+                            
+                            boolean ret = doVerify(jarFileForTh,entry,pkgForTh);
+                            
+                            if(!ret) {
+                                verifyRet.set(CERTIFICAT_VERIFY_FAIL);
+                                //verifyRet.ret = CERTIFICAT_VERIFY_FAIL;
+                                latch.countDown();
+                                return;
+                            }
+                        }
+                    }
+                }).start();
+            }
+            //System.out.println("wangsl,collectCertificates 8");
+            try {
+                latch.await();
+            }catch(InterruptedException e){}
+            //System.out.println("wangsl,collectCertificates 9");
+            if(verifyRet.get() == CERTIFICAT_VERIFY_FAIL) {
+            //if(verifyRet.ret == CERTIFICAT_VERIFY_FAIL) {
+                throw new PackageParserException(
+                        INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
+                                + " has mismatched certificates at entry ");
+                
+            }
+
+        } //catch (GeneralSecurityException e) {
+          //  throw new PackageParserException(INSTALL_PARSE_FAILED_CERTIFICATE_ENCODING,
+          //          "Failed to collect certificates from " + apkPath, e);
+        //} 
+        catch (IOException | RuntimeException e) {
+            throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
+                    "Failed to collect certificates from " + apkPath, e);
+        } finally {
+            closeQuietly(jarFile);
+        }
+
+    }
+
+    private static boolean doVerify(StrictJarFile jarfile,ZipEntry entry,Package pkg) {
+        // Verify that entries are signed consistently with the first entry
+        // we encountered. Note that for splits, certificates may have
+        // already been populated during an earlier parse of a base APK.
+        Certificate[][] entryCerts = null;
+        //System.out.println("wangsl,doVerify 1");
+        try {
+            entryCerts = loadCertificates(jarfile, entry);
+        } catch (Exception e) {
+            //throw new PackageParserException(INSTALL_PARSE_FAILED_CERTIFICATE_ENCODING,
+            //        "Failed to collect certificates from");
+            return false;
+        } 
+        //System.out.println("wangsl,doVerify 2");
+        if (ArrayUtils.isEmpty(entryCerts)) {
+                //throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
+                //        "Package " + apkPath + " has no certificates at entry "
+                //        + entry.getName());
+                return false;
+        }
+        //System.out.println("wangsl,doVerify 3");
+        Signature[] entrySignatures = null;
+        try {
+            entrySignatures = convertToSignatures(entryCerts);
+        }catch(CertificateEncodingException e) {
+            return false;
+        }
+        //System.out.println("wangsl,doVerify 4");
+        if (pkg.mCertificates == null) {
+            pkg.mCertificates = entryCerts;
+            pkg.mSignatures = entrySignatures;
+            pkg.mSigningKeys = new ArraySet<PublicKey>();
+            for (int i=0; i < entryCerts.length; i++) {
+                pkg.mSigningKeys.add(entryCerts[i][0].getPublicKey());
+            }
+        } else {
+            if (!Signature.areExactMatch(pkg.mSignatures, entrySignatures)) {
+            //   throw new PackageParserException(
+            //            INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + apkPath
+            //                        + " has mismatched certificates at entry "
+            //                        + entry.getName());
+                return false;
+            }
+        }
+        //System.out.println("wangsl,doVerify 5");
+        return true;
+    }
+    //modify for install tuning start
+
+    //modify for install tuning start
+    private static void collectCertificatesUgly(Package pkg, File apkFile, int flags)
+            throws PackageParserException {
+    //modify for install tuning end
+        final String apkPath = apkFile.getAbsolutePath();
+        StrictJarFile jarFile1 =null;
+        try {
+            final StrictJarFile jarFile= new StrictJarFile(apkPath);
+            jarFile1=jarFile;
             // Always verify manifest, regardless of source
             final ZipEntry manifestEntry = jarFile.findEntry(ANDROID_MANIFEST_FILENAME);
             if (manifestEntry == null) {
@@ -1136,14 +1353,15 @@
             // Verify that entries are signed consistently with the first entry
             // we encountered. Note that for splits, certificates may have
             // already been populated during an earlier parse of a base APK.
-            for (ZipEntry entry : toVerify) {
-                final Certificate[][] entryCerts = loadCertificates(jarFile, entry);
-                if (ArrayUtils.isEmpty(entryCerts)) {
-                    throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
-                            "Package " + apkPath + " has no certificates at entry "
-                            + entry.getName());
-                }
-                final Signature[] entrySignatures = convertToSignatures(entryCerts);
+            if ((flags & PARSE_IS_SYSTEM) != 0) {
+                for (ZipEntry entry : toVerify) {
+                    final Certificate[][] entryCerts = loadCertificates(jarFile, entry);
+                     if (ArrayUtils.isEmpty(entryCerts)) {
+                         throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
+                                 "Package " + apkPath + " has no certificates at entry "
+                                 + entry.getName());
+                      }
+                     final Signature[] entrySignatures = convertToSignatures(entryCerts);
 
                 if (pkg.mCertificates == null) {
                     pkg.mCertificates = entryCerts;
@@ -1161,6 +1379,44 @@
                     }
                 }
             }
+            }else{
+                int length = toVerify.size();
+                final Package mPkg=pkg;
+                Slog.i(TAG, "collectCertificates length = " + length);
+                mThreshold = length / 2;
+                final CountDownLatch mConnectedSignal = new CountDownLatch(2);
+                new Thread("asynccollectCertificates1") {
+                    @Override
+                    public void run() {
+                        mIsasync1=asynccollectCertificates(mPkg,toVerify, jarFile, 1);
+                        mIsSync1Finish = true;
+                        if (!mIsasync1) {
+                               mConnectedSignal.countDown();
+                           }
+                        mConnectedSignal.countDown();
+                    }
+                }.start();
+
+                new Thread("asynccollectCertificates2") {
+                    @Override
+                    public void run() {
+                        mIsasync2=asynccollectCertificates(mPkg,toVerify, jarFile, 2);
+                        if(mIsSync1Finish){
+                            mIsSync1Finish = false;
+                        }
+                        if (!mIsasync2) {
+                            mConnectedSignal.countDown();
+                           }
+                        mConnectedSignal.countDown();
+                    }
+                }.start();
+                waitForLatch(mConnectedSignal);
+                Slog.e(TAG,"mIsasync1 = " + mIsasync1 + " mIsasync2 = " + mIsasync2);
+                if (!mIsasync1 || !mIsasync2) {
+                    throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
+                             "Package " + apkPath + " has no certificates ");
+                  }
+            }
         } catch (GeneralSecurityException e) {
             throw new PackageParserException(INSTALL_PARSE_FAILED_CERTIFICATE_ENCODING,
                     "Failed to collect certificates from " + apkPath, e);
@@ -1168,9 +1424,10 @@
             throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
                     "Failed to collect certificates from " + apkPath, e);
         } finally {
-            closeQuietly(jarFile);
+            closeQuietly(jarFile1);
         }
     }
+    /* @} */
 
     private static Signature[] convertToSignatures(Certificate[][] certs)
             throws CertificateEncodingException {
@@ -4554,6 +4811,10 @@
          */
         public ManifestDigest manifestDigest;
 
+        // add by chendongqi from SPRD modify for install tuning start(add manifestdigest cache)
+        private ManifestDigest preManifestDigest = null;
+        // dd by chendongqi from SPRD modify for install tuning end
+
         public String mOverlayTarget;
         public int mOverlayPriority;
         public boolean mTrustedOverlay;
@@ -5645,4 +5906,99 @@
         return enabled ;
     }
     /// @}
+	
+	// add by chendongqi from SPRD start
+	/* SPRD: Modify the process of the certificate verification to improve the speed of installing application{@ */
+    private static void waitForLatch(CountDownLatch latch) {
+        for (;;) {
+            try {
+                if (latch.await(1000 * 500, TimeUnit.MILLISECONDS)) {
+                    Slog.e(TAG, "waitForLatch done!");
+                    return;
+                } else {
+                    Slog.e(TAG, "Thread " + Thread.currentThread().getName()
+                            + " still waiting for asynccollectCertificates ready...");
+                }
+            } catch (InterruptedException e) {
+                Slog.e(TAG, "Interrupt while waiting for asynccollectCertificates to be ready.");
+            }
+        }
+    }
+    private static boolean mIsasync1 = true;
+    private static boolean mIsasync2 = true;
+    private static boolean mIsSync1Finish = false;
+    private static int mThreshold;
+    private static boolean asynccollectCertificates(Package pkg,List<ZipEntry> entries, StrictJarFile jarFile,
+            int asyncNum){
+        int elementNum = 0;
+        Slog.e(TAG, "asyncNum = " + asyncNum);
+        try{
+            final Iterator<ZipEntry> i = entries.iterator();
+            while (i.hasNext()) {
+                final ZipEntry entry = i.next();
+                elementNum++;
+                if (asyncNum == 1) {
+                    if (elementNum > mThreshold) {
+                        break;
+                    }
+                } else {
+                    if (elementNum <= mThreshold) {
+                        continue;
+                    }
+                }
+                if (entry.isDirectory()) continue;
+                if (entry.getName().startsWith("META-INF/")) continue;
+
+
+
+               final Certificate[][] localCerts = loadCertificates(jarFile, entry);
+
+               if (ArrayUtils.isEmpty(localCerts)) {
+                   throw new PackageParserException(INSTALL_PARSE_FAILED_NO_CERTIFICATES,
+                           "Package " + pkg.packageName + " has no certificates at entry "
+                           + entry.getName());
+               }
+               if((pkg.mCertificates == null) && (asyncNum == 2)){
+                   Slog.e(TAG, "certs == null");
+                   while(pkg.mCertificates == null){
+                       try{
+                           Thread.sleep(20);
+                           if(mIsSync1Finish){
+                               break;
+                           }
+                           continue;
+                       }catch(Exception e){
+                           Slog.i(TAG, "exception occured");
+                       }
+                   }
+               }
+               final Signature[] entrySignatures = convertToSignatures(localCerts);
+
+               if (pkg.mCertificates == null) {
+                   pkg.mCertificates = localCerts;
+                   pkg.mSignatures = entrySignatures;
+                   pkg.mSigningKeys = new ArraySet<PublicKey>();
+                   for (int i1=0; i1 < localCerts.length; i1++) {
+                       pkg.mSigningKeys.add(localCerts[i1][0].getPublicKey());
+                     }
+               } else {
+                   if (!Signature.areExactMatch(pkg.mSignatures, entrySignatures)) {
+                       throw new PackageParserException(
+                               INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES, "Package " + pkg
+                                       + " has mismatched certificates at entry "
+                                       + entry.getName());
+                   }
+               }
+           }
+
+       } catch (GeneralSecurityException e) {
+            Slog.w(TAG, "Failed to collect certificates from " + pkg.packageName, e);
+            return false;
+       } catch (Exception  e) {
+            Slog.w(TAG, "Failed to collect certificates from " + pkg.packageName, e);
+            return false;
+        }
+       return true;
+    }
+    // add by chendongqi from SPRD end
 }
diff --git a/core/java/com/android/internal/content/PackageHelper.java b/core/java/com/android/internal/content/PackageHelper.java
index 3e0c0b9..816fb12 100644
--- a/core/java/com/android/internal/content/PackageHelper.java
+++ b/core/java/com/android/internal/content/PackageHelper.java
@@ -311,7 +311,9 @@
 
     private static void copyZipEntry(ZipEntry zipEntry, ZipFile inZipFile,
             ZipOutputStream outZipStream) throws IOException {
-        byte[] buffer = new byte[4096];
+        // add by chendongqi from SPRD modify for install tuning start
+        byte[] buffer = new byte[4096*8];
+        // add by chendongqi from SPRD modify for install tuning end
         int num;
 
         ZipEntry newEntry;
diff --git a/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java b/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java
index 1ec3618..1c5d7f7 100644
--- a/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java
+++ b/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java
@@ -62,6 +62,10 @@
 import java.io.IOException;
 import java.io.InputStream;
 import java.io.OutputStream;
+// add by chendongqi from SPRD modify for install tuning start
+import java.io.FileOutputStream;
+import java.util.concurrent.LinkedBlockingQueue;
+// add by chendongqi from SPRD modify for install tuning end
 
 
 /**
@@ -388,6 +392,88 @@
         return PackageManager.INSTALL_SUCCEEDED;
     }
 
+    // add by chendongqi from SPRD modify for install tuning start
+    static class threadcopy {
+        public int length;
+        public byte[] buffer = new byte[1024*512];
+    }
+     private static void copyByThread(final InputStream in,final OutputStream out) throws IOException {
+
+        final LinkedBlockingQueue<threadcopy>list = new LinkedBlockingQueue(8);
+
+        Thread t1 = new Thread(new Runnable() {
+
+            @Override
+            public void run() {
+                int count = 0;
+                while(true) {
+                    try {
+                        threadcopy temp = new threadcopy();
+                        temp.length = in.read(temp.buffer);
+                        list.put(temp);
+                        
+                        if(temp.length <= 0) {
+                            break;
+                        }
+                    } catch (Exception e1) {
+                        // TODO Auto-generated catch block
+                        e1.printStackTrace();
+                    }
+                }
+                
+            }
+        });
+
+        Thread t2 = new Thread(new Runnable() {
+
+            @Override
+            public void run() {
+                while(true) {
+                    threadcopy data;
+                    try {
+                         data = list.take();
+                         if(data.length < 0) {
+                             break;
+                         }
+                         out.write(data.buffer , 0, data.length);
+                    } catch (Exception e) {
+                        // TODO Auto-generated catch block
+                        e.printStackTrace();
+                    }
+                }
+            }
+        });
+
+        try {
+            t1.start();
+            t2.start();
+
+            t1.join();
+            t2.join();
+        }catch(Exception e) {
+            e.printStackTrace();
+
+        }
+    }
+
+    public static boolean copySDFile(File srcFile, File destFile) {
+        boolean result = true;
+        try {
+            InputStream in = new FileInputStream(srcFile);
+            OutputStream out = new FileOutputStream(destFile);
+            try {
+                copyByThread(in, out);
+            } finally  {
+                in.close();
+                out.close();
+            }
+        } catch (IOException e) {
+            result = false;
+        }
+        return result;
+    }
+
+    // add by chendongqi from SPRD modify for install tuning end
     private void copyFile(String sourcePath, IParcelFileDescriptorFactory target, String targetName)
             throws IOException, RemoteException {
         Slog.d(TAG, "Copying " + sourcePath + " to " + targetName);
@@ -397,7 +483,10 @@
             in = new FileInputStream(sourcePath);
             out = new ParcelFileDescriptor.AutoCloseOutputStream(
                     target.open(targetName, ParcelFileDescriptor.MODE_READ_WRITE));
-            Streams.copy(in, out);
+            // add by chendongqi from SPRD modify for install tuning start
+            //Streams.copy(in, out);
+            copyByThread(in,out);
+            // add by chendongqi from SPRD modify for install tuning end
         } finally {
             IoUtils.closeQuietly(out);
             IoUtils.closeQuietly(in);
@@ -410,9 +499,14 @@
         final File targetFile = new File(targetDir, targetName);
 
         Slog.d(TAG, "Copying " + sourceFile + " to " + targetFile);
-        if (!FileUtils.copyFile(sourceFile, targetFile)) {
+        // add by chendongqi from SPRD modify for install tuning start
+        //if (!FileUtils.copyFile(sourceFile, targetFile)) {
+        //    throw new IOException("Failed to copy " + sourceFile + " to " + targetFile);
+        //}
+        if (!copySDFile(sourceFile, targetFile)) {
             throw new IOException("Failed to copy " + sourceFile + " to " + targetFile);
         }
+        // add by chendongqi from SPRD modify for install tuning end
 
         if (isForwardLocked) {
             final String publicTargetName = PackageHelper.replaceEnd(targetName,
```

#### 4.2 多线程扫描应用优化方案

优化了第一次开机和正常开机时扫描目录的速度，在ZAW600 TECNO项目上的测试数据，首次开机提速17秒，正常开机提速3秒，以下为该方案的patch：

```java
From 6b3798059b2fb159db1761de58fe07c603fc8b7d Mon Sep 17 00:00:00 2001
From: chendongqi <chendongqi@huaqin.com>
Date: Mon, 25 Jul 2016 11:15:25 +0800
Subject: [PATCH] [性能][开机速度]移植展讯开机pms多线程扫描方案，第一次开机提速17秒，正常开机提速3秒左右

Change-Id: Ice2cbb2f028b51ff2d923988600f40335559c976
---

diff --git a/services/core/java/com/android/server/pm/PackageManagerService.java b/services/core/java/com/android/server/pm/PackageManagerService.java
index 9cacb07..3119b3f 100644
--- a/services/core/java/com/android/server/pm/PackageManagerService.java
+++ b/services/core/java/com/android/server/pm/PackageManagerService.java
@@ -247,6 +247,7 @@
 import java.io.FileNotFoundException;
 import java.io.FileOutputStream;
 import java.io.FileReader;
+import java.io.FileWriter;
 import java.io.FilenameFilter;
 import java.io.IOException;
 import java.io.InputStream;
@@ -459,6 +460,12 @@
     // This is where all application persistent data goes for secondary users.
     final File mUserAppDataDir;
 
+	// add by chendongqi from SPRD for boot performance --start @{
+    /* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+    final File mPreloadInstallDir;
+    final File mDeleteRecord;
+    /* @} */
+	// add by chendongqi from SPRD for boot performance --end @}
     /** The location for ASEC container files on internal storage. */
     final String mAsecInternalPath;
 
@@ -2027,9 +2034,10 @@
         mSystemPermissions = systemConfig.getSystemPermissions();
         mAvailableFeatures = systemConfig.getAvailableFeatures();
 
+        /* SPRD: Add for boot performance with multi-thread and preload scan{@
         synchronized (mInstallLock) {
         // writer
-        synchronized (mPackages) {
+        synchronized (mPackages) { @}*/
             mHandlerThread = new ServiceThread(TAG,
                     Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
             mHandlerThread.start();
@@ -2043,7 +2051,12 @@
             mAsecInternalPath = new File(dataDir, "app-asec").getPath();
             mUserAppDataDir = new File(dataDir, "user");
             mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
-
+			// add by chendongqi from SPRD for boot performance --start @{
+            /* SPRD: Add for boot performance with multi-thread and preload scan{@ */
+            mPreloadInstallDir = new File(Environment.getRootDirectory(), "preloadapp");
+            mDeleteRecord = new File(mAppInstallDir, ".delrecord");
+             /* @} */
+			 // add by chendongqi from SPRD for boot performance --end @}
             sUserManager = new UserManagerService(context, this,
                     mInstallLock, mPackages);
 
@@ -2356,16 +2369,45 @@
                     scanFlags | SCAN_NO_DEX, 0);
                     
 
+			// modify by chendongqi from SPRD for boot performance --start @{
+            /* SPRD: Add for boot performance with multi-thread and preload scan {@*/
+            mAsecScanFlag = scanFlags & (~SCAN_DEFER_DEX);
             // Collected privileged system packages.
             final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
+            new Thread("scanPrivilegedAppDir"){
+                @Override
+                public void run() {
+                    long startTime = SystemClock.uptimeMillis();
             scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
                     | PackageParser.PARSE_IS_SYSTEM_DIR
                     | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);
+                    Slog.w(TAG, "scanPrivilegedAppDir done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
+                    mConnectedSignal.countDown();
 
+                }
+            }.start();
             // Collect ordinary system packages.
             final File systemAppDir = new File(Environment.getRootDirectory(), "app");
-            scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
-                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+            new Thread("scanSystemAppDir-part0"){
+                @Override
+                public void run() {
+                    long startTime = SystemClock.uptimeMillis();
+                    scanPartDir(0,systemAppDir, PackageParser.PARSE_IS_SYSTEM
+                            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+                    Slog.w(TAG, "scanSystemAppDir-part0 done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
+                    mConnectedSignal.countDown();
+                }
+            }.start();
+            new Thread("scanSystemAppDir-part1"){
+                @Override
+                public void run() {
+                    long startTime = SystemClock.uptimeMillis();
+                    scanPartDir(1,systemAppDir, PackageParser.PARSE_IS_SYSTEM
+                            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+                    Slog.w(TAG, "scanSystemAppDir-part1 done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
+                    mConnectedSignal.countDown();
+                }
+            }.start();
 
             /** M: [Resmon] Enhancement of resmon filter @{ */
             ResmonFilter rf = new ResmonFilter();
@@ -2373,14 +2415,22 @@
             /** @} */
 
             // Collect all vendor packages.
-            File vendorAppDir = new File("/vendor/app");
-            try {
+            final File vendorAppDir = new File("/vendor/app");
+            /*try {
                 vendorAppDir = vendorAppDir.getCanonicalFile();
             } catch (IOException e) {
                 // failed to look up canonical path, continue with original one
-            }
-            scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
-                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+            }*/
+            new Thread("scanVendorAppDir"){
+                @Override
+                public void run() {
+                    long startTime = SystemClock.uptimeMillis();
+                    scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
+                            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+                    Slog.w(TAG, "scanVendorAppDir done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
+                    mConnectedSignal.countDown();
+                }
+            }.start();
 
             /** M: [ALPS00104673][Need Patch][Volunteer Patch]Mechanism for uninstall app from system partition @{ */
             /// M: [ALPS01210636] Unify path from /vendor to /system/vendor/
@@ -2419,8 +2469,18 @@
 
             // Collect all OEM packages.
             final File oemAppDir = new File(Environment.getOemDirectory(), "app");
+            new Thread("scanOemAppDir"){
+                @Override
+                public void run() {
+                    long startTime = SystemClock.uptimeMillis();
             scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
                     | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
+                    Slog.w(TAG, "scanOemAppDir done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
+                    mConnectedSignal.countDown();
+                }
+            }.start();
+            waitForLatch(mConnectedSignal);
+            /* @} */
 
             if (DEBUG_UPGRADE) Log.v(TAG, "Running installd update commands");
             mInstaller.moveFiles();
@@ -2499,11 +2559,15 @@
             if (!mOnlyCore) {
                 EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                         SystemClock.uptimeMillis());
+                /* SPRD: Add for boot performance with multi-thread and preload scan {@*/
                 scanDirLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
-
-                scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
-                        scanFlags | SCAN_REQUIRE_KNOWN, 0);
-
+                final File vitalAppDir = new File(Environment.getRootDirectory(), "vital-app");
+                if(vitalAppDir.exists() && vitalAppDir.isDirectory()){
+                    scanDirLI(vitalAppDir, 0, scanFlags, 0);
+                   }
+                /*scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
+                        scanFlags | SCAN_REQUIRE_KNOWN, 0);*/
+                 /* @} */
                 /**
                  * Remove disable package settings for any updated system
                  * apps that were removed via an OTA. If they're not a
@@ -2583,11 +2647,13 @@
                 }
             }
             mExpectingBetter.clear();
-
+            /* SPRD: Add for boot performance with multi-thread and preload scan {@*/
+            synchronized (mPackages) {
             // Now that we know all of the shared libraries, update all clients to have
             // the correct library paths.
             updateAllSharedLibrariesLPw();
-
+            }
+            /* @} */
             for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                 // NOTE: We ignore potential failures here during a system scan (like
                 // the rest of the commands above) because there's precious little we
@@ -2595,10 +2661,13 @@
                 adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
                         false /* force dexopt */, false /* defer dexopt */);
             }
-
+            /* SPRD: Add for boot performance with multi-thread and preload scan {@*/
+            synchronized (mPackages) {
             // Now that we know all the packages we are keeping,
             // read and update their last usage times.
             mPackageUsage.readLP();
+            }
+            /* @} */
 
             EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                     SystemClock.uptimeMillis());
@@ -2622,7 +2691,11 @@
                         + mSdkVersion + "; regranting permissions for internal storage");
                 updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
             }
-            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
+            /* SPRD: Add for boot performance with multi-thread and preload scan {@*/
+            synchronized (mPackages) {
+            updatePermissionsLPw(null, null, updateFlags);
+            }
+            /* @} */
             ver.sdkVersion = mSdkVersion;
 
             // If this is the first boot or an update from pre-M, and it is a normal
@@ -2676,9 +2749,12 @@
             mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                     mIntentFilterVerifierComponent);
 
+        /* SPRD: Add for boot performance with multi-thread and preload scan
         } // synchronized (mPackages)
-        } // synchronized (mInstallLock)
-
+        } // synchronized (mInstallLock) @}*/
+        /* SPRD: Add for boot performance with multi-thread and preload scan{@*/
+        mAsyncScanThread.start();
+         /* @} */
         // Now after opening every single application zip, make sure they
         // are all flushed.  Not really needed, but keeps things nice and
         // tidy.
@@ -6155,6 +6231,12 @@
         } catch (PackageParserException e) {
             throw PackageManagerException.from(e);
         }
+		
+		/* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+        if(isPreloadOrVitalApp(scanFile.getParent()) && mDeleteRecord.exists()){
+            if(isDeleteApp(pkg.packageName))return pkg;
+        }
+        /* @} */
 
         //add for factory test start lvxudong 20160204
         if(isFactoryKitTest && !mRestoredSettings && !factoryPackages.contains(pkg.packageName)){
@@ -7374,7 +7456,11 @@
                         }
                     }
                     if (!recovered && ((parseFlags&PackageParser.PARSE_IS_SYSTEM) != 0
-                            || (scanFlags&SCAN_BOOTING) != 0)) {
+                            || (scanFlags&SCAN_BOOTING) != 0)
+                            /* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+                            || currentUid > 0
+                            || isPreloadOrVitalApp(pkg.applicationInfo)) {
+                                /* @} */
                         // If this is a system app, we can at least delete its
                         // current data so the application will still work.
                         int ret = removeDataDirsLI(pkg.volumeUuid, pkgName);
@@ -8044,6 +8130,10 @@
                 // This is a regular package, with one or more known overlay packages.
                 createIdmapsForPackageLI(pkg);
             }
+            /* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+            mPreloadPkgList.add(pkg.applicationInfo.packageName);
+            mPreloadUidList.add(pkg.applicationInfo.uid);
+            /* @} */
         }
 
         return pkg;
@@ -8353,8 +8443,11 @@
         info.nativeLibraryRootRequiresIsa = false;
         info.nativeLibraryDir = null;
         info.secondaryNativeLibraryDir = null;
-
-        if (isApkFile(codeFile)) {
+        /* SPRD: Add for boot performance with multi-thread and preload scan
+         * @orig if (isApkFile(codeFile)) {@
+          */
+        if (isApkFile(codeFile) || isPreloadOrVitalApp(codePath)) {
+          /* @} */
             // Monolithic install
             if (bundledApp) {
                 // If "/system/lib64/apkname" exists, assume that is the per-package
@@ -14121,6 +14214,12 @@
             if (DEBUG_REMOVE) Slog.d(TAG, "Removing non-system package:" + ps.name);
             // Kill application pre-emptively especially for apps on sd.
             killApplication(packageName, ps.appId, "uninstall pkg");
+            /* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+            if(ps.pkg != null && ps.pkg.baseCodePath != null){
+                String path = ps.pkg.baseCodePath.substring(0, ps.pkg.baseCodePath.lastIndexOf("/"));
+                if(isPreloadOrVitalApp(path)) delAppRecord(ps.pkg.packageName, flags);
+            }
+            /* @} */
             ret = deleteInstalledPackageLI(ps, deleteCodeAndResources, flags,
                     allUserHandles, perUserInstalled,
                     outInfo, writeSettings);
@@ -17851,4 +17950,167 @@
         return false;
     }
     /** @}*/
+	
+	    /* SPRD: Add for boot performance with multi-thread and preload scan @{*/
+    private  final CountDownLatch mConnectedSignal = new CountDownLatch(5);
+    private  void waitForLatch(CountDownLatch latch) {
+        for (;;) {
+            try {
+                if (latch.await(5000, TimeUnit.MILLISECONDS)) {
+                    Slog.e(TAG, "waitForLatch done!" );
+                    return;
+                } else {
+                    Slog.e(TAG, "Thread " + Thread.currentThread().getName()
+                            + " still waiting for ready...");
+                }
+            } catch (InterruptedException e) {
+                Slog.e(TAG, "Interrupt while waiting for preload class to be ready.");
+            }
+        }
+    }
+    private int mAsecScanFlag ;
+    private ArrayList<String> mPreloadPkgList = new ArrayList<String>();
+    private ArrayList<Integer> mPreloadUidList = new ArrayList<Integer>();
+    Thread mAsyncScanThread = new Thread() {
+        @Override
+        public void run() {
+            File preloadInstallDir = new File(mPreloadInstallDir.getPath());
+            File drmAppPrivateInstallDir = new File(
+                    mDrmAppPrivateInstallDir.getPath());
+            Slog.d(TAG, "preInstallDir start");
+            mPreloadPkgList.clear();
+            mPreloadUidList.clear();
+            scanDirLI(preloadInstallDir, 0, mAsecScanFlag, 0);
+            Slog.d(TAG, "preInstallDir done");
+            scanDirLI(drmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
+                  mAsecScanFlag | SCAN_REQUIRE_KNOWN, 0);
+            Slog.d(TAG, "DrmAppPrivateInstallDir done");
+
+            synchronized (mPackages) {
+                updateAllSharedLibrariesLPw();
+
+                final boolean regrantPermissions = mSettings.getInternalVersion().sdkVersion != mSdkVersion;
+                mSettings.getInternalVersion().sdkVersion = mSdkVersion;
+                updatePermissionsLPw(
+                        null,
+                        null,
+                        UPDATE_PERMISSIONS_ALL
+                                | (regrantPermissions ? (UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL)
+                                        : 0));
+                mSettings.writeLPr();
+            }
+            // Send a broadcast to let everyone know we are done processing
+            int uidArr[] = new int[mPreloadUidList.size()];
+
+            if (mPreloadPkgList.size() > 0) {
+                // Send broadcasts here
+                Bundle extras = new Bundle();
+                extras.putStringArray(Intent.EXTRA_CHANGED_PACKAGE_LIST,
+                        mPreloadPkgList.toArray(new String[mPreloadPkgList.size()]));
+                if (uidArr != null) {
+                    extras.putIntArray(Intent.EXTRA_CHANGED_UID_LIST, uidArr);
+                }
+                sendPackageBroadcast(
+                        Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE, null,
+                        extras, null, null, null);
+            }
+            Runtime.getRuntime().gc();
+        }
+    };
+     private boolean isPreloadOrVitalApp(String path){
+        Slog.i(TAG, "preloadOrVital path : " + path);
+        if(path.startsWith("/system/preloadapp") || path.startsWith("/system/vital-app"))
+              return true;
+        return false;
+   }
+
+    private boolean isPreloadOrVitalApp(ApplicationInfo info) {
+        if(info.sourceDir != null) {
+            return info.sourceDir.startsWith("/system/preloadapp/")
+                    || info.sourceDir.startsWith("/system/vital-app");
+        }
+        return false;
+    }
+
+    private boolean isDeleteApp(String packageName){
+        BufferedReader br = null;
+        try{
+          br = new BufferedReader(new FileReader(mDeleteRecord));
+          String lineContent = null;
+          while( (lineContent = br.readLine()) != null){
+              if(packageName.equals(lineContent)){
+                   return true;
+              }
+          }
+        }catch(IOException e){
+           Log.e(TAG, " isDeleteApp IOException");
+        }finally{
+           try{
+              if(br != null)
+                br.close();
+           }catch(IOException e){
+               Log.e(TAG, " isDeleteApp Close ... IOException");
+           }
+        }
+        return false;
+    }
+
+    private boolean delAppRecord(String packageName,int parseFlags){
+      FileWriter writer = null;
+      try{
+         writer = new FileWriter(mDeleteRecord, true);
+         writer.write(packageName +"\n");
+         writer.flush();
+      }catch(IOException e){
+           Log.e(TAG, "preloadapp unInstall record:  IOException");
+      }finally{
+           try{
+            if(writer != null)
+                writer.close();
+           }catch(IOException e){}
+      }
+       return true;
+    }
+
+    private void scanPartDir(int part, File dir, int flags, int scanMode,
+            long currentTime) {
+        String[] files = dir.list();
+        if (files == null) {
+            Log.d(TAG, "No files in app dir " + dir);
+            return;
+        }
+        if (DEBUG_PACKAGE_SCANNING) {
+            Log.d(TAG, "Scanning preload app dir " + dir + " scanMode="
+                    + scanMode + " flags=0x" + Integer.toHexString(flags));
+        }
+        int i = part == 0 ? 0 : files.length / 2 + 1;
+        int max = part == 0 ? files.length / 2 : files.length - 1;
+        for (; i <= max; i++) {
+            File file = new File(dir, files[i]);
+/*            if (!isPackageFilename(files[i])) {
+                // Ignore entries which are not apk's
+                continue;
+            }*/
+            try{
+                scanPackageLI(file, flags
+                        | PackageParser.PARSE_MUST_BE_APK, scanMode, currentTime,null);
+            } catch (PackageManagerException e) {
+                Slog.w(TAG, "Failed to parse " + file + ": " + e.getMessage());
+
+                // Delete invalid userdata apps
+                if ((flags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
+                        e.error == PackageManager.INSTALL_FAILED_INVALID_APK) {
+                    logCriticalInfo(Log.WARN, "Deleting invalid package at " + file);
+                    if (file.isDirectory()) {
+                        FileUtils.deleteContents(file);
+                    }
+                    file.delete();
+                }
+            }
+        }
+    }
+    private boolean isPackageFilename(String name) {
+        return name != null && name.endsWith(".apk");
+    }
+    /* @} */
 }

```       
