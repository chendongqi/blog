---
layout:      post
title:      "HIDL系列三"
subtitle:   "绑定化直通式的案例及理解"
navcolor:   "invert"
date:       2019-09-01
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - HIDL
    - Treble
    - Android P
    - MTK 6771
    - binderized
    - Binderizing passthrough
    - same-process passthrough
---

#  0 概述

本篇来介绍如何在源码中添加一个绑定化直通式(binderizing passthrough)的实例，包括了hal文件的编写，hidl-gen工具自动化生成hidl服务端文件，服务端代码的编写，bp文件的配置，c++测试客户端的编写，selinux配置和vndk配置。  
绑定化直通式，也就是在same-process直通式的基础上，添加了一个服务，在服务启动时将HIDL的服务注册到hwservicemanager中，在客户端使用时选择服务获取方式来从hwservicemanager中取到服务端。  

# 1 hal文件的新增  

在AOSP目录下的hardware/interfaces建立目录hidl_test，在它之下再建立目录1.0，在1.0目录中新建文件ITest.hal，内容如下  

```hal
package android.hardware.hidl_test@1.0;

interface ITest{
    helloWorld(string name) generates (string result);
};
```

# 2 hidl-gen工具

该工具用来根据我们建立的ITest.hal文件来帮助我们生成服务端的实现

## 2.1 hidl-gen工具生成

- source build/envsetup.sh  

- lunch \<project\>  

- make hidl-gen  


## 2.2 使用方法

基本用法： hidl-gen -o output-path -L language (-r interface-root) fqname    

参数说明：    
- -L： 语言类型，包括c++, c++-headers, c++-sources, export-header, c++-impl, java, java-constants, vts, makefile, androidbp, androidbp-impl, hash等。hidl-gen可根据传入的语言类型产生不同的文件。  

- fqname： 完全限定名称的输入文件。比如本例中android.hardware.hidl_test@1.0，要求在源码目录下必须有hardware/interfaces/hidl_test/1.0/目录。对于单个文件来说，格式如下：package@version::fileName，比如android.hardware.hidl_test@1.0::types.Feature。对于目录来说。格式如下package@version，比如android.hardware.hidl_test@1.0。  

- -r： 格式package:path，可选，对fqname对应的文件来说，用来指定包名和文件所在的目录到Android系统源码根目录的路径。如果没有制定，前缀默认是：android.hardware，目录是Android源码的根目录。  

- -o：存放hidl-gen产生的中间文件的路径。  


## 2.3 生成服务端代码模板

运行  

```xml
LOC=hardware/interfaces/hidl_test/1.0/default/
PACKAGE=android.hardware.hidl_test@1.0
hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces/ -randroid.hidl:system/libhidl/transport/ $PACKAGE
```

会在指定的LOC路径下生成Test.cpp和Test.h文件如下  

```cpp
// file: Test.cpp
#include "Test.h"

namespace android {
namespace hardware {
namespace hidl_test {
namespace V1_0 {
namespace implementation {

// Methods from ::android::hardware::hidl_test::V1_0::ITest follow.
Return<void> Test::helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) {
    // TODO implement
    // 做了简单实现 start @{
    printf("Test helloWorld");
    char buf[100];
    memset(buf, 0, 100);
    snprintf(buf, 100, "Hello World, %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    // 做了简单实现 end @}
    return Void();
}


// Methods from ::android::hidl::base::V1_0::IBase follow.
// 直通式需要放开的部分  start @{}
ITest* HIDL_FETCH_ITest(const char* /* name */) {
    return new Test();
} 
// end @}
}  // namespace implementation
}  // namespace V1_0
}  // namespace hidl_test
}  // namespace hardware
}  // namespace android
```

```h
#ifndef ANDROID_HARDWARE_HIDL_TEST_V1_0_TEST_H
#define ANDROID_HARDWARE_HIDL_TEST_V1_0_TEST_H

#include <android/hardware/hidl_test/1.0/ITest.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace hidl_test {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct Test : public ITest {
    // Methods from ::android::hardware::hidl_test::V1_0::ITest follow.
    Return<void> helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.

};

// FIXME: most likely delete, this is only for passthrough implementations
// 直通式把这部分注释打开就行
extern "C" ITest* HIDL_FETCH_ITest(const char* name);

}  // namespace implementation
}  // namespace V1_0
}  // namespace hidl_test
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_HIDL_TEST_V1_0_TEST_H
```

两个文件按照如上注释做简单修改  

然后执行  

```xml
hidl-gen -o $LOC -Landroidbp-impl -randroid.hardware:hardware/interfaces/ -randroid.hidl:system/libhidl/transport/ $PACKAGE
```

来生成bp文件，会在default下生成Android.bp，内容如下，默认使用直通式。    

```xml
cc_library_shared {
    // FIXME: this should only be -impl for a passthrough hal.
    // In most cases, to convert this to a binderized implementation, you should:
    // - change '-impl' to '-service' here and make it a cc_binary instead of a
    //   cc_library_shared.
    // - add a *.rc file for this module.
    // - delete HIDL_FETCH_I* functions.
    // - call configureRpcThreadpool and registerAsService on the instance.
    // You may also want to append '-impl/-service' with a specific identifier like
    // '-vendor' or '-<hardware identifier>' etc to distinguish it.
    name: "android.hardware.hidl_test@1.0-impl",
    relative_install_path: "hw",
    // FIXME: this should be 'vendor: true' for modules that will eventually be
    // on AOSP.
    proprietary: true,
    srcs: [
        "Test.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.hidl_test@1.0",
    ],
}
```

# 3 update-makefiles.sh

如果是在hardware/interface下建立的新的hal，那么上面操作之后可以调用update-makefiles.sh脚本来更新了编译脚本了。执行脚本的前提是source/lunch过了，然后执行  
```cmd
./hardware/interfaces/update-makefiles.sh
```
会在hardware/interfaces/hidl_test/1.0目录下生成Android.bp  
```bp
// This file is autogenerated by hidl-gen -Landroidbp.

hidl_interface {
    name: "android.hardware.hidl_test@1.0",
    root: "android.hardware",
    vndk: {
        enabled: true,
    },
    srcs: [
        "ITest.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: true,
}
```



# 4 service.cpp

在default下建立service.cpp文件，写入内容如下  

```cpp
#define LOG_TAG "android.hardware.hidl_test@1.0-service"
#include <android/hardware/hidl_test/1.0/ITest.h>
#include <hidl/LegacySupport.h>

using android::hardware::hidl_test::V1_0::ITest;
using android::hardware::defaultPassthroughServiceImplementation;

int main() {
    return defaultPassthroughServiceImplementation<ITest>();
}
```

修改同目录下的Android.bp，将service.cpp编译成服务端  

```xml
cc_binary {
    name: "android.hardware.hidl_test@1.0-service",
    relative_install_path: "hw",
    vendor: true,
    init_rc: ["android.hardware.hidl_test@1.0-service.rc"],
    srcs: ["service.cpp"],

    shared_libs: [
        "liblog",
        "libcutils",
        "libdl",
        "libbase",
        "libutils",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "android.hardware.hidl_test@1.0",
    ],
}
```

# 5 rc文件

为了让服务端能够开机自启动，添加rc配置，在default下建立android.hardware.hidl_test@1.0-service.rc文件，内容如下  

```rc
service hidl_test_hal_service /vendor/bin/hw/android.hardware.hidl_test@1.0-service
    class hal
    user system
    group system

```



# 6 vndk

在用update-makefiles.sh生成的Android.bp中，如果配置了vndk的enable为true的话，需要为新增的hidl配置vndk，否则在运行时会出现so库找不到的情况。修改如下：    

build/make/target/product/vndk/28.txt增加一行  

```tx
VNDK-core: android.hardware.health@2.0.so
VNDK-core: android.hardware.hidl_test@1.0.so
VNDK-core: android.hardware.ir@1.0.so
```

build/make/target/product/vndk/current.txt增加一行  

```txt
VNDK-core: android.hardware.health@2.0.so
VNDK-core: android.hardware.hidl_test@1.0.so
VNDK-core: android.hardware.ir@1.0.so
```

这里要注意的是增加的位置要根据字母排序，否则编译会报错  



# 7 Selinux

为了让service能在开机运行，需要配置selinux    
修改device/mediatek/mt6771/sepolicy/basic目录下的问题  
修改attributes     

```xml
# add by chendongqi for hidl test
attribute mytest_hal_hidl_test;
attribute mytest_hal_hidl_test_server;
attribute mytest_hal_hidl_test_client;
```
修改file_contexts    
```xml
# add by chendongqi for hidl test
/vendor/bin/hw/android\.hardware\.hidl_test@1\.0-service u:object_r:mytest_hal_hidl_test_default_exec:s0
```
修改hwservice.te    
```xml
# add by chendongqi for hidl test
type mytest_hidl_test_hwservice, hwservice_manager_type;
```
修改hwservice_contexts   
```xml
# add by chendongqi for hidl test
android.hardware.hidl_test::ITest   u:object_r:mytest_hidl_test_hwservice:s0
```
添加mytest_hal_hidl_test_default.te   
```xml
type mytest_hal_hidl_test_default, domain;

hal_server_domain(mytest_hal_hidl_test_default, mytest_hal_hidl_test)

type mytest_hal_hidl_test_default_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(mytest_hal_hidl_test_default)

add_hwservice(mytest_hal_hidl_test_server, mytest_hidl_test_hwservice)

hwbinder_use(mytest_hal_hidl_test_default)

vndbinder_use(mytest_hal_hidl_test_default)

allow mytest_hal_hidl_test_default hwservicemanager_prop:file { read getattr open };
```



# 8 manifest

为了保证新增的hidl是可以被客户端访问到的，需要修改device/mediatek/mt6771/manifest.xml，添加如下  
```xml
    <hal format="hidl">
        <name>android.hardware.hidl_test</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>ITest</name>
            <instance>default</instance>
        </interface>
    </hal>
```



# 9 客户端

添加c++的测试客户端  

在1.0目录下建立test目录，test目录下建立TestClient.cpp，内容如下  

```cpp
#include <android/hardware/hidl_test/1.0/ITest.h>
#include <hidl/HidlSupport.h>
#include <stdio.h>

using ::android::hardware::hidl_string;
using ::android::sp;
using ::android::hardware::hidl_test::V1_0::ITest;

int main() {

    android::sp<ITest> service = ITest::getService();
    
    if (service == nullptr) {
        printf("Failed to get service\n");
        return -1;
    }

    service->helloWorld("Test", [&](hidl_string result) {
        printf("%s\n", result.c_str());
    });

    return 0;
}
```

建立Android.bp  

```xml
cc_binary {
    relative_install_path: "hw", 
    defaults: ["hidl_defaults"],
    name: "my_hidl_test_client",
    proprietary: true,
    srcs: ["TestClient.cpp"],

    shared_libs: [
        "liblog", 
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.hidl_test@1.0",
    ],
}
```

# 10 编译

直接用make命令整编整个工程，编译完成后会生成的一些有用的文件如下  

```xml
./system/lib64/android.hardware.hidl_test@1.0.so
./system/lib64/vndk-28/android.hardware.hidl_test@1.0.so
./system/lib/android.hardware.hidl_test@1.0.so
./system/lib/vndk-28/android.hardware.hidl_test@1.0.so
./vendor/lib64/hw/android.hardware.hidl_test@1.0-impl.so
./vendor/lib/hw/android.hardware.hidl_test@1.0-impl.so
./vendor/bin/hw/my_hidl_test_client
./vendor/etc/init/android.hardware.hidl_test@1.0-service.rc
./vendor/bin/hw/android.hardware.hidl_test@1.0-service
```

这些就是我们会用到的一些文件，测试时开机会会根据rc文件，把android.hardware.hidl_test@1.0-service自启动起来，然后我们去运行客户端my_hidl_test_client，会调用到system下的so库，然后会调用到vendor下的impl的so库。  

# 11 问题汇总

## 11.1 vndk编译错误

```xml
FAILED: out/target/product/spm8666p1_64/obj/PACKAGING/vndk_intermediates/check-list-timestamp 
/bin/bash -c "(( diff --old-line-format=\"Removed %L\" 	  --new-line-format=\"Added %L\" 	  --unchanged-line-format=\"\" 	  build/make/target/product/vndk/28.txt out/target/product/spm8666p1_64/obj/PACKAGING/vndk_intermediates/libs.txt 	  || ( echo -e \" error: VNDK library list has been changed.\\n\" \"       Changing the VNDK library list is not allowed in API locked branches.\"; exit 1 )) ) && (mkdir -p out/target/product/spm8666p1_64/obj/PACKAGING/vndk_intermediates/ ) && (touch out/target/product/spm8666p1_64/obj/PACKAGING/vndk_intermediates/check-list-timestamp )"
Added VNDK-core: android.hardware.hidl_test@1.0.so
Added VNDK-core: android.hardware.power@1.3.so
 error: VNDK library list has been changed.
```

遇到此类编译错误，参考第6部分添加vndk  

## 11.2 so库找不到

运行客户端时提示android.hardware.hidl_test@1.0.so库找不到  

参考第6部分添加vndk 

## 11.3 服务获取不到

运行客户端时日志提示服务获取失败。  

首先做如下两步，手动运行android.hardware.hidl_test@1.0-service，然后再运行客户端，可以正常运行，则Selinux配置有问题，参考哦啊第7部分Selinux配置。一般这种情况下，运行客户端后会终端会一直卡在那儿，查看日志可以看到一直在等待服务端连接。  

另外一种情况，确保service是运行起来的情况下，执行客户端，打印代码中日志“Failed to get service”，查看系统日志也可以发现  

```xml
W hwservicemanager: getTransport: Cannot find entry android.hardware.hidl_test@1.0::ITest in either framework or device manifest.
```

参考第8部分，配置manifest文件  



# 12 附录

AOSP下完整的代码修改参考我的github： [hidl](https://github.com/chendongqi/hidl)  
