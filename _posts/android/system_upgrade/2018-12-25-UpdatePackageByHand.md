---
layout:      post
title:      "系统升级系列四"
subtitle:   "手动制作升级包"
navcolor:   "invert"
date:       2018-12-25
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - 升级包制作
    - 手动
---

### 前言

在系列二中介绍了增量包和全量包的制作方式以及从制作脚本剖析了升级包生成的完整过程和细节，这一篇来介绍一个比较偏的制作升级包的方法，虽然方法偏，但是有大用处。  

#### 问题来由

这个方式的产生来源于项目中遇到的一个真实的问题：售后接到多起投诉是设备无法开机，或是反复重启。经过一轮分析之后发现语音分区被写满了，而进一步的原因是讯飞语音的日志将语音分区空间给塞满了。而经过和讯飞一轮的讨论之后，希望能由系统方来删除设备端的日志并且将讯飞日志的配置文件替换成一个修复后的配置文件。经过一轮苦思冥想，决定采用OTA升级的方式来解决此问题，但是遇到几个难点：1、用命令做出来的升级包包含了其他不需要的升级，例如build.prop；2、OTA升级做出来的包是不升级讯飞语音分区的（这个是关键的难点）  

### 解决思路

经过一轮思考，最终根据对updater-script的理解，猜想了一种方式，应该可以手动裁剪和修改updater-script脚本来达到需要的功能：将讯飞的配置文件拷贝到语音分区的对应位置；裁剪掉所有不必要的升级语句。然后就开始实践自己的方案了。这种方案特别像医学上所用的靶向药，可以定向针对某些部位和问题释放药效，而我把这种升级包叫做**定向包**。  

### 原料

需要用到的是，一个现成的升级包、讯飞新的配置文件、签名工具和秘钥  

### 实现方案

#### 1 解压裁剪升级包

将现成的升级包解压开，然后删除patch下的所有内容（不需要对系统打patch），删除system下的所有文件（也不需要拷贝任何任务）  

#### 2 替换讯飞配置文件

将新的讯飞配置文件isstts_log.cfg放到system/etc下，在updater-script下添加（原来有的就保留）

```python
package_extract_dir("data/update_s/system", "/data/update_s/system");
```

来实现升级时isstts_log.cfg会被解压的设备的system/etc/下。添加  

```python
assert(run_program("/system/bin/busybox", "rm", "/data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg"));
assert(run_program("/system/bin/busybox", "mv", "/data/update_s/system/etc/isstts_log.cfg", "/data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg"));
```

来实现将原先的配置文件删除，将system/etc下新的配置文件移动过去，注意要用mv，来保证升级之后system/etc下是没有新增的东西，保证系统的完整性。  

#### 3 删除讯飞日志

添加

```python
run_program("/system/bin/busybox", "rm", "-rf", "/data/update_s/ivres/iflytek/res/Active/TTSRes/log*/*");
```

在升级时删除指定目录  

还有一种思路是，我可以写一个shell脚本  

```shell
#!/system/bin/sh

/system/bin/busybox rm -rf /data/update_s/ivres/iflytek/res/Active/TTSRes/log*/*
/system/bin/busybox rm /data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg
/system/bin/busybox mv /data/update_s/system/etc/isstts_log.cfg /data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg
```

将isstts_log.cfg和这个脚本都放到system/etc/下，升级时解压到设备/system/etc/下，然后在updater-script中添加执行该脚本的语句，这样就可以把要做的事封装起来，做完之后把该脚本isstts_log.cfg和脚本删除掉。  

#### 4 升级校验添加修改

为了满足升级时的校验条件，需要修改一些判断的条件，google原生的设计部分是META-INF/com/android/metadata  

```xml
post-build=Freescale/xxxx/xxxx/xxxx/638:user/dev-keys
post-timestamp=1530351425
pre-build=Freescale/xxxx/xxxx/xxxx/610:user/dev-keys
pre-device=xxxx
```

注意修改fingerprint（或者直接去掉updater-script里校验fingerprint的语句），如果原料使用的升级包比较老，那么可以把时间戳改成一个很大的值，因为校验时间戳时比这个点晚的版本会无法升级（或者直接去掉updater-script里校验时间戳的语句），设备名称要对应。  

因为这边是厂商定制，校验的时候用了一个定制的Manifest_SYS.xml来存储信息  

```xml
<?xml version="1.0" encoding="utf-8"?>
<sys_info>
<type>sys</type>
<project>xe1109s</project>### 设备名称一定要对应好，在为多个项目做此工作时要注意对应
<vendor>xxxx</vendor>
<isfull>1</isfull>### 我直接改成了全量包，避免了很多增量升级时的校验
<systemVersionName>9.9.9</systemVersionName>### 版本名称写大一点
<systemVersionCode>999</systemVersionCode>### 版本号大一点
<dependencyVersionName>0.0.0</dependencyVersionName>### 不需要
<dependencyVersionCode>000</dependencyVersionCode>### 不需要
<systemVersionDescription></systemVersionDescription>
<ext></ext>
</sys_info>
```



#### 5 签名

把修改后的目录打包，然后用signapk和release来签名  

```xml
java -Xmx2048m -jar out/host/linux-x86/framework/signapk.jar -w build/target/product/security/releasekey.x509.pem build/target/product/security/releasekey.pk8 update.zip signed.zip
```



### updater-script实例

提供下脚本的参考，重点是这种思路，具体如何实现可以根据自己的升级包和问题来专门定制  

```python
ui_print("Mount System partion...");
show_progress(0.100000, 0);
assert(run_program("/system/bin/busybox", "mkdir", "-p", "/data/update_s/system"));
mount("ext4", "EMMC", "/dev/block/mmcblk0p2", "/data/update_s/system");
assert(run_program("/system/bin/busybox", "mkdir", "-p", "/data/update_s/ivres"));
mount("ext4", "EMMC", "/dev/block/mmcblk0p12", "/data/update_s/ivres");
show_progress(0.200000, 0);

ui_print("Skip verifying current system...");
show_progress(0.300000, 0);

# ---- start making changes here ----
ui_print("No files to delete...");
show_progress(0.500000, 0);
ui_print("No system files to patch...");
show_progress(0.600000, 0);
# ---- add new files
ui_print("Unpacking new files...");
package_extract_dir("data/update_s/system", "/data/update_s/system");
show_progress(0.700000, 0);

# ---- permissions
ui_print("Symlinks and permissions...");
#set_metadata("/data/update_s/system/etc/iflytek_update.sh", "uid", 0, "gid", 2000, "mode", 0755, "capabilities", 0x0, "selabel", "u:object_r:system_file:s0");
show_progress(0.800000, 0);

# --- repair iflytek res
ui_print("Repair iflytek res...");
run_program("/system/bin/busybox", "rm", "-rf", "/data/update_s/ivres/iflytek");
assert(run_program("/data/update_s/system/bin/iflytek_res"));
#assert(run_program("/data/update_s/system/etc/iflytek_update.sh"));
assert(run_program("/system/bin/busybox", "rm", "/data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg"));
assert(run_program("/system/bin/busybox", "mv", "/data/update_s/system/etc/isstts_log.cfg", "/data/update_s/ivres/iflytek/res/Active/TTSRes/isstts_log.cfg"));
show_progress(0.900000, 0);

# ---- ready to switch system
#run_program("/system/bin/busybox", "rm", "/data/update_s/system/etc/iflytek_update.sh");
run_program("/system/bin/busybox", "umount", "/data/update_s/system");
run_program("/system/bin/busybox", "rm", "-rf", "/data/update_s/system");
run_program("/system/bin/busybox", "umount", "/data/update_s/ivres");
run_program("/system/bin/busybox", "rm", "-rf", "/data/update_s/ivres");
switch_system(1);
```

