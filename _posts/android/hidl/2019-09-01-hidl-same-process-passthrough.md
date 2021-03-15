---
layout:      post
title:      "HIDL系列二"
subtitle:   "same-process直通式的案例及理解"
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

# 0 概述

本篇来介绍如何构建一个最基础的same-process passthrough实例。    

# 1 same-process passthrough简介

same-process passthrough是谷歌在Treble结构中，为了分离framework和vendor所引入的HIDL的实现的第一种形态。在这种方式下，把HIDL的定义放在framework中，真正的实现放在vendor中，减少了升级中对vendor所做的修改。在这个HIDL实现的最初级形态中，因为还存在着很多老版的HAL模块的实现，为了兼容这些模块，所以HIDL实现中真正做的也就是去打开这些模块的so，客户端真正操作的还是旧版的HAL实现。因为这种方式是在一个进程中完成的，所以被成为same-process，而passthrough的意思呢，直通，是谁和谁的直通？对比绑定式的形态，我的理解是客户端和HIDL的直通。所以这里passthrough和same-process并没有递进的关系，两者说的其实是一个意思，正因为直通了，所以是same-process。  



# 2 实现案例

来看一个最基础的案例实现  



## 2.1 添加hal接口  

  在AOSP目录下的hardware/interfaces建立目录helloworld，在它之下再建立目录1.0，在1.0目录中新建文件IHelloWorld.hal，内容如下   

```hal
package android.hardware.helloworld@1.0;

interface IHelloWorld{
    helloWorld(string name) generates (string result);
};
```

## 2.2 hidl-gen工具

该工具用来根据我们建立的ITest.hal文件来帮助我们生成服务端的实现

### 2.2.1 hidl-gen工具生成

- source build/envsetup.sh  

- lunch \<project\>  

- make hidl-gen  


### 2.2.2 使用方法

基本用法： hidl-gen -o output-path -L language (-r interface-root) fqname    

参数说明：    
- -L： 语言类型，包括c++, c++-headers, c++-sources, export-header, c++-impl, java, java-constants, vts, makefile, androidbp, androidbp-impl, hash等。hidl-gen可根据传入的语言类型产生不同的文件。  

- fqname： 完全限定名称的输入文件。比如本例中android.hardware.helloworld@1.0，要求在源码目录下必须有hardware/interfaces/helloworld/1.0/目录。对于单个文件来说，格式如下：package@version::fileName，比如android.hardware.helloworld@1.0::types.Feature。对于目录来说。格式如下package@version，比如android.hardware.helloworld@1.0。  

- -r： 格式package:path，可选，对fqname对应的文件来说，用来指定包名和文件所在的目录到Android系统源码根目录的路径。如果没有制定，前缀默认是：android.hardware，目录是Android源码的根目录。  

- -o：存放hidl-gen产生的中间文件的路径。  


### 2.2.3 生成服务端代码模板

运行  

```xml
LOC=hardware/interfaces/helloworld/1.0/default/
PACKAGE=android.hardware.helloworld@1.0
hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces/ -randroid.hidl:system/libhidl/transport/ $PACKAGE
```

会在指定的LOC路径下生成HelloWorld.cpp和HelloWorld.h文件如下  

```cpp
// file: HelloWorld.cpp
#include "HelloWorld.h"

namespace android {
namespace hardware {
namespace helloworld {
namespace V1_0 {
namespace implementation {

// Methods from ::android::hardware::helloworld::V1_0::IHelloWorld follow.
Return<void> HelloWorld::helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) {
    // TODO implement
    printf("Test helloWorld");
    char buf[100];
    memset(buf, 0, 100);
    snprintf(buf, 100, "Hello World, %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    return Void();
}


// Methods from ::android::hidl::base::V1_0::IBase follow.

IHelloWorld* HIDL_FETCH_IHelloWorld(const char* /* name */) {
    return new HelloWorld();
    // 在正式实现时，我们会在这里去打开hardware module的，例如下面来自源码Gnss的例子
    /*
    hw_module_t* module;
    IGnss* iface = nullptr;
    int err = hw_get_module(GPS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);

    if (err == 0) {
        hw_device_t* device;
        err = module->methods->open(module, GPS_HARDWARE_MODULE_ID, &device);
        if (err == 0) {
            iface = new Gnss(reinterpret_cast<gps_device_t*>(device));
        } else {
            ALOGE("gnssDevice open %s failed: %d", GPS_HARDWARE_MODULE_ID, err);
        }
    } else {
      ALOGE("gnss hw_get_module %s failed: %d", GPS_HARDWARE_MODULE_ID, err);
    }
    return iface;
    */
}
//
}  // namespace implementation
}  // namespace V1_0
}  // namespace helloworld
}  // namespace hardware
}  // namespace android
```

```h
#ifndef ANDROID_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H
#define ANDROID_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H

#include <android/hardware/helloworld/1.0/IHelloWorld.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace helloworld {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct HelloWorld : public IHelloWorld {
    // Methods from ::android::hardware::helloworld::V1_0::IHelloWorld follow.
    Return<void> helloWorld(const hidl_string& name, helloWorld_cb _hidl_cb) override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.

};

// FIXME: most likely delete, this is only for passthrough implementations
extern "C" IHelloWorld* HIDL_FETCH_IHelloWorld(const char* name);

}  // namespace implementation
}  // namespace V1_0
}  // namespace helloworld
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H
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
    name: "android.hardware.helloworld@1.0-impl",
    relative_install_path: "hw",
    // FIXME: this should be 'vendor: true' for modules that will eventually be
    // on AOSP.
    proprietary: true,
    srcs: [
        "HelloWorld.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.helloworld@1.0",
    ],
}
```

## 2.3 update-makefiles.sh

如果是在hardware/interface下建立的新的hal，那么上面操作之后可以调用update-makefiles.sh脚本来更新了编译脚本了。执行脚本的前提是source/lunch过了，然后执行  
```cmd
./hardware/interfaces/update-makefiles.sh
```
会在hardware/interfaces/hidl_test/1.0目录下生成Android.bp  
```bp
// This file is autogenerated by hidl-gen -Landroidbp.

hidl_interface {
    name: "android.hardware.helloworld@1.0",
    root: "android.hardware",
    vndk: {
        enabled: true,
    },
    srcs: [
        "IHelloWorld.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: true,
}
```
## 2.4 vndk

在用update-makefiles.sh生成的Android.bp中，如果配置了vndk的enable为true的话，需要为新增的hidl配置vndk，否则在运行时会出现so库找不到的情况。修改如下：    

build/make/target/product/vndk/28.txt增加一行  

```tx
VNDK-core: android.hardware.health@2.0.so
VNDK-core: android.hardware.helloworld@1.0.so
VNDK-core: android.hardware.ir@1.0.so
```

build/make/target/product/vndk/current.txt增加一行  

```txt
VNDK-core: android.hardware.health@2.0.so
VNDK-core: android.hardware.helloworld@1.0.so
VNDK-core: android.hardware.ir@1.0.so
```

这里要注意的是增加的位置要根据字母排序，否则编译会报错  

## 2.6 manifest

为了保证新增的hidl是可以被客户端访问到的，需要修改device/mediatek/mt6771/manifest.xml，添加如下  
```xml
    <hal format="hidl">
        <name>android.hardware.helloworld</name>
        <transport arch="32+64">passthrough</transport>
        <version>1.0</version>
        <interface>
            <name>IHelloWorld</name>
            <instance>default</instance>
        </interface>
    </hal>
```
要注意这里transport申明成passthrough的方式，而且一定要申明arch属性  

## 2.7 客户端

添加c++的测试客户端  

在1.0目录下建立test目录，test目录下建立TestClient.cpp，内容如下  

```cpp
#include <android/hardware/helloworld/1.0/IHelloWorld.h>
#include <hidl/HidlSupport.h>
#include <stdio.h>

using ::android::hardware::hidl_string;
using ::android::sp;
using ::android::hardware::helloworld::V1_0::IHelloWorld;

int main() {

    android::sp<IHelloWorld> service = IHelloWorld::getService();
    
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
    name: "my_helloworld",
    proprietary: true,
    srcs: ["TestClient.cpp"],

    shared_libs: [
        "liblog", 
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.helloworld@1.0",
    ],
}
```
## 2.8 编译  

直接用make来整编，编译之后烧写镜像。然后开机后执行我们的测试客户端my_helloworld就可以得到正确的返回结果了。  

# 3 附录  

AOSP下完整的代码修改参考我的github： [hidl](https://github.com/chendongqi/hidl)  
