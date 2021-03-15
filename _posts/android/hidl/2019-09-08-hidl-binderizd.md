---
layout:      post
title:      "HIDL系列四"
subtitle:   "绑定式的案例及理解"
navcolor:   "invert"
date:       2019-09-08
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

本篇来介绍如何在源码中添加一个绑定式(binderized)的实例，包括了hal文件的编写，hidl-gen工具自动化生成hidl服务端文件，服务端代码的编写，bp文件的配置，c++测试客户端的编写，selinux配置和vndk配置。  
绑定式HIDL，HAL作为服务端，直接以一个Service的形式运行，所有HAL的实现也都在服务进程中实现。在客户端使用时选择服务获取方式来从hwservicemanager中取到服务端。  

# 1 hal文件的新增  

在AOSP目录下的hardware/interfaces建立目录hidl_test，在它之下再建立目录1.0，在1.0目录中新建文件ITest.hal，内容如下  

```hal
package android.hardware.binderized_test@1.0;

interface IBinderizedTest{
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

- fqname： 完全限定名称的输入文件。比如本例中android.hardware.binderized_test@1.0，要求在源码目录下必须有hardware/interfaces/binderized_test /1.0/目录。对于单个文件来说，格式如下：package@version::fileName，比如android.hardware.binderized_test@1.0::types.Feature。对于目录来说。格式如下package@version，比如android.hardware.binderized_test@1.0。  

- -r： 格式package:path，可选，对fqname对应的文件来说，用来指定包名和文件所在的目录到Android系统源码根目录的路径。如果没有制定，前缀默认是：android.hardware，目录是Android源码的根目录。  

- -o：存放hidl-gen产生的中间文件的路径。  


## 2.3 生成服务端代码模板

运行  

```xml
LOC=hardware/interfaces/binderized_test/1.0/default/
PACKAGE=android.hardware.binderized_test@1.0
hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces/ -randroid.hidl:system/libhidl/transport/ $PACKAGE
```

会在指定的LOC路径下生成BinderizedTest.cpp和BinderizedTest.h文件如下  

```cpp
// file: BinderizedTest.cpp
#include "BinderizedTest.h"

namespace android {
namespace hardware {
namespace binderized_test {
namespace V1_0 {
namespace implementation {

// Methods from ::android::hardware::binderized_test::V1_0::IBinderizedTest follow.
Return<void> BinderizedTest::helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) {
    // TODO implement
    // 做了简单实现
    printf("Test helloWorld");
    char buf[100];
    memset(buf, 0, 100);
    snprintf(buf, 100, "Hello World, %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    return Void();
}


// Methods from ::android::hidl::base::V1_0::IBase follow.

//IBinderizedTest* HIDL_FETCH_IBinderizedTest(const char* /* name */) {
	// 绑定式这里保持默认注释掉就行了
    //return new BinderizedTest();
//}
//
}  // namespace implementation
}  // namespace V1_0
}  // namespace binderized_test
}  // namespace hardware
}  // namespace android
```

```h
#ifndef ANDROID_HARDWARE_BINDERIZED_TEST_V1_0_BINDERIZEDTEST_H
#define ANDROID_HARDWARE_BINDERIZED_TEST_V1_0_BINDERIZEDTEST_H

#include <android/hardware/binderized_test/1.0/IBinderizedTest.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace binderized_test {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct BinderizedTest : public IBinderizedTest {
    // Methods from ::android::hardware::binderized_test::V1_0::IBinderizedTest follow.
    Return<void> helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.

};

// FIXME: most likely delete, this is only for passthrough implementations
// extern "C" IBinderizedTest* HIDL_FETCH_IBinderizedTest(const char* name);

}  // namespace implementation
}  // namespace V1_0
}  // namespace binderized_test
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_BINDERIZED_TEST_V1_0_BINDERIZEDTEST_H
```

两个文件按照如上注释做简单修改  

然后执行  

```xml
hidl-gen -o $LOC -Landroidbp-impl -randroid.hardware:hardware/interfaces/ -randroid.hidl:system/libhidl/transport/ $PACKAGE
```

来生成bp文件，会在default下生成Android.bp，内容如下，做如下修改，使用绑定式，将编译成一个可执行文件。    

```xml
cc_binary {
    // FIXME: this should only be -impl for a passthrough hal.
    // In most cases, to convert this to a binderized implementation, you should:
    // - change '-impl' to '-service' here and make it a cc_binary instead of a
    //   cc_library_shared.
    // - add a *.rc file for this module.
    // - delete HIDL_FETCH_I* functions.
    // - call configureRpcThreadpool and registerAsService on the instance.
    // You may also want to append '-impl/-service' with a specific identifier like
    // '-vendor' or '-<hardware identifier>' etc to distinguish it.
    name: "android.hardware.binderized_test@1.0-service",
    relative_install_path: "hw",
    // FIXME: this should be 'vendor: true' for modules that will eventually be
    // on AOSP.
    vendor: true,
    srcs: [
        "BinderizedTest.cpp",
    ],

    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "liblog",
        "android.hardware.binderized_test@1.0",
    ],
}
```

# 3 update-makefiles.sh

如果是在hardware/interface下建立的新的hal，那么上面操作之后可以调用update-makefiles.sh脚本来更新了编译脚本了。执行脚本的前提是source/lunch过了，然后执行  
```cmd
./hardware/interfaces/update-makefiles.sh
```
会在hardware/interfaces/binderized_test/1.0目录下生成Android.bp  
```bp
// This file is autogenerated by hidl-gen -Landroidbp.

hidl_interface {
    name: "android.hardware.binderized_test@1.0",
    root: "android.hardware",
    vndk: {
        enabled: true,
    },
    srcs: [
        "IBinderizedTest.hal",
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
/*
 * Copyright (C) 2017 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
#include <unistd.h>

#include <hidl/HidlTransportSupport.h>
#include <log/log.h>
#include <utils/Errors.h>
#include <utils/StrongPointer.h>

#include "BinderizedTest.h"


// libhidl:
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;

// Generated HIDL files
using android::hardware::binderized_test::V1_0::IBinderizedTest;

// The namespace in which all our implementation code lives
using namespace android::hardware::binderized_test::V1_0::implementation;
using namespace android;


// Main service entry point
int main() {
    // Create an instance of our service class
    // 这里需要去主动向hwservicemanager做注册  
    android::sp<IBinderizedTest> service = new BinderizedTest();
    configureRpcThreadpool(1, true /*callerWillJoin*/);

    if (service->registerAsService() != OK) {
        ALOGE("registerAsService failed");
        return 1;
    }

    // Join (forever) the thread pool we created for the service above
    joinRpcThreadpool();

    // We don't ever actually expect to return, so return an error if we do get here
    return 2;
}
```

修改同目录下的Android.bp，将service.cpp添加进编译  

```xml
cc_binary {
    // FIXME: this should only be -impl for a passthrough hal.
    // In most cases, to convert this to a binderized implementation, you should:
    // - change '-impl' to '-service' here and make it a cc_binary instead of a
    //   cc_library_shared.
    // - add a *.rc file for this module.
    // - delete HIDL_FETCH_I* functions.
    // - call configureRpcThreadpool and registerAsService on the instance.
    // You may also want to append '-impl/-service' with a specific identifier like
    // '-vendor' or '-<hardware identifier>' etc to distinguish it.
    name: "android.hardware.binderized_test@1.0-service",
    relative_install_path: "hw",
    // FIXME: this should be 'vendor: true' for modules that will eventually be
    // on AOSP.
    vendor: true,
    srcs: [
        "BinderizedTest.cpp",
        "service.cpp"
    ],

    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "liblog",
        "android.hardware.binderized_test@1.0",
    ],
}
```

# 5 rc文件

为了让服务端能够开机自启动，添加rc配置，在default下建立android.hardware.binderized_test@1.0-service.rc文件，内容如下  

```rc
service vendor.binderized_test-hal-1.0 /vendor/bin/hw/android.hardware.binderized_test@1.0-service
    class hal
    user system
    group system
```
修改同目录下的android.bp，将rc文件添加进编译  
```xml
cc_binary {
    // FIXME: this should only be -impl for a passthrough hal.
    // In most cases, to convert this to a binderized implementation, you should:
    // - change '-impl' to '-service' here and make it a cc_binary instead of a
    //   cc_library_shared.
    // - add a *.rc file for this module.
    // - delete HIDL_FETCH_I* functions.
    // - call configureRpcThreadpool and registerAsService on the instance.
    // You may also want to append '-impl/-service' with a specific identifier like
    // '-vendor' or '-<hardware identifier>' etc to distinguish it.
    name: "android.hardware.binderized_test@1.0-service",
    relative_install_path: "hw",
    // FIXME: this should be 'vendor: true' for modules that will eventually be
    // on AOSP.
    vendor: true,
    srcs: [
        "BinderizedTest.cpp",
        "service.cpp"
    ],

    init_rc: ["android.hardware.binderized_test@1.0-service.rc"],

    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "liblog",
        "android.hardware.binderized_test@1.0",
    ],
}
```



# 6 vndk

在用update-makefiles.sh生成的Android.bp中，如果配置了vndk的enable为true的话，需要为新增的hidl配置vndk，否则在运行时会出现so库找不到的情况。修改如下：    

build/make/target/product/vndk/28.txt增加一行  

```tx
VNDK-core: android.hardware.automotive.vehicle@2.0.so
VNDK-core: android.hardware.binderized_test@1.0.so
VNDK-core: android.hardware.biometrics.fingerprint@2.1.so
```

build/make/target/product/vndk/current.txt增加一行  

```txt
VNDK-core: android.hardware.automotive.vehicle@2.0.so
VNDK-core: android.hardware.binderized_test@1.0.so
VNDK-core: android.hardware.biometrics.fingerprint@2.1.so
```

这里要注意的是增加的位置要根据字母排序，否则编译会报错  



# 7 Selinux

为了让service能在开机运行，需要配置selinux    
修改device/mediatek/mt6771/sepolicy/basic目录下的问题  
修改attributes     

```xml
# add by chendongqi for binderized hidl test
attribute mytest_binderized_test;
attribute mytest_binderized_test_server;
attribute mytest_binderized_test_client;
```
修改file_contexts    
```xml
# add by chendongqi for binderized hidl test
/vendor/bin/hw/android\.hardware\.binderized_test@1\.0-service u:object_r:mytest_binderized_test_default_exec:s0
```
修改hwservice.te    
```xml
# add by chendongqi for hidl test
type mytest_hidl_test_hwservice, hwservice_manager_type;
```
修改hwservice_contexts   
```xml
# add by chendongqi for binderized hidl test
android.hardware.binderized_test::IBinderizedTest u:object_r:mytest_binderized_test_hwservice:s0
```
添加mytest_binderized_test_default.te   
```xml
type mytest_binderized_test_default, domain;

hal_server_domain(mytest_binderized_test_default, mytest_binderized_test)

add_hwservice(mytest_binderized_test_server, mytest_binderized_test_hwservice)
hwbinder_use(mytest_binderized_test_default)
vndbinder_use(mytest_binderized_test_default)

type mytest_binderized_test_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(mytest_binderized_test_default)


allow mytest_binderized_test_default hwservicemanager_prop:file { read getattr open };
```



# 8 manifest

为了保证新增的hidl是可以被客户端访问到的，需要修改device/mediatek/mt6771/manifest.xml，添加如下  
```xml
    <hal format="hidl">
        <name>android.hardware.binderized_test</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>IBinderizedTest</name>
            <instance>default</instance>
        </interface>
    </hal>
```



# 9 客户端

添加c++的测试客户端  

在1.0目录下建立test目录，test目录下建立TestClient.cpp，内容如下  

```cpp
#include <android/hardware/binderized_test/1.0/IBinderizedTest.h>
#include <hidl/HidlSupport.h>
#include <stdio.h>

using ::android::hardware::hidl_string;
using ::android::sp;
using ::android::hardware::binderized_test::V1_0::IBinderizedTest;

int main() {

    android::sp<IBinderizedTest> service = IBinderizedTest::getService();
    
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
    name: "my_binderized_test_client",
    proprietary: true,
    srcs: ["TestClient.cpp"],

    shared_libs: [
        "liblog", 
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.binderized_test@1.0",
    ],
}
```

# 10 编译

直接用make命令整编整个工程，编译完成后会生成的一些有用的文件如下  

```xml
./system/lib64/android.hardware.binderized_test@1.0.so
./system/lib64/vndk-28/android.hardware.binderized_test@1.0.so
./system/lib/android.hardware.binderized_test@1.0.so
./system/lib/vndk-28/android.hardware.binderized_test@1.0.so
./vendor/bin/hw/my_binderized_test_client
./vendor/etc/init/android.hardware.binderized_test@1.0-service.rc
./vendor/bin/hw/android.hardware.binderized_test@1.0-service
```

这些就是我们会用到的一些文件，测试时开机会会根据rc文件，把android.hardware.binderized_test@1.0-service自启动起来，然后我们去运行客户端my_binderized_test_client，会调用到system下的so库，然后会调用到android.hardware.binderized_test@1.0-service中的实现。  


# 11 附录

AOSP下完整的代码修改参考我的github： [hidl](https://github.com/chendongqi/hidl)  