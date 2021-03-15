---
layout:      post
title:      "Android图形系统分析一"
subtitle:   "Gralloc模块的实现原理分析"
navcolor:   "invert"
date:       2019-07-26
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - Gralloc
    - 图形系统
    - hal
    - hw
    - SurfaceFlinger
---
### 分析平台
本文主要参考了老罗的[Android帧缓冲区（Frame Buffer）硬件抽象层（HAL）模块Gralloc的实现原理分析](https://blog.csdn.net/Luoshengyang/article/details/7747932)，以MT6735（android 5.1）的代码为例做的分析。而能阅读的代码部分只是google释放的hal层代码。其实这部分代码并未真正在系统运行时使用，MTK会修改这部分hal层实现，释放gralloc.mt6735.so库来被Gralloc模块初始化时加载。我们就以google的源码为例做理论层面上的分析吧。   

### 0 Gralloc实现原理概要

为了在屏幕中绘制一个指定的画面，应用层需要通过HAL层与硬件设备（屏幕）打交道，将绘制好的图形显示到屏幕中去。主要封装了/dev/graphics/fb<x>节点的操作，为框架层提供接口，kernel通过图形驱动来操作fb节点。这里的HAL层即为Gralloc模块，其功能的实现原理如下：  

1. 加载Gralloc模块  
2. 打开Gralloc模块中的gralloc设备和fb设备  
3. gralloc设备分配一个匹配屏幕大小的图形缓冲区  
4. 将分配好的图形缓冲区注册（通过内存映射的方式）到当前进程的地址空间中  
5. 将要绘制的画面内容写入到注册好的图形缓冲区中  
6. 由fb设备渲染到系统帧缓冲区中  

### 1 加载Gralloc模块  
HAL层的so库统一加载方式，代码位于hardware/libhardware/hardware.c中  

#### 1.1 hw_get_module

```c
int hw_get_module(const char *id, const struct hw_module_t **module)
{
    // 这里的id是Gralloc模块的id，定义在hardware/libhardware/include/hardware/gralloc.h文件中
    // #define GRALLOC_HARDWARE_MODULE_ID "gralloc"
    // hw_module_t是定义在hardware/libhardware/include/hardware/hardware.h中的结构体
    // 用来描述模块的信息，比较重要的是其中的struct hw_module_methods_t* methods
    return hw_get_module_by_class(id, NULL, module);// 直接调用hw_get_module_by_class
}
```
```c  
int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
    int i;
    char prop[PATH_MAX];
    char path[PATH_MAX];
    char name[PATH_MAX];
    char prop_name[PATH_MAX];

    if (inst)
        snprintf(name, PATH_MAX, "%s.%s", class_id, inst);
    else
        strlcpy(name, class_id, PATH_MAX);

    /*
     * Here we rely on the fact that calling dlopen multiple times on
     * the same .so will simply increment a refcount (and not load
     * a new copy of the library).
     * We also assume that dlopen() is thread-safe.
     */

    /* First try a property specific to the class and possibly instance */
    // 先通过ro.hardware.module_name来找system/lib/hw下面有没有合适的so库
    snprintf(prop_name, sizeof(prop_name), "ro.hardware.%s", name);
    if (property_get(prop_name, prop, NULL) > 0) {
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Loop through the configuration variants looking for a module */
    // 如果上面一步没有找到就轮询variant_keys，它的定义在hardware/libhardware/hardware.c中
    // static const char *variant_keys[] = {
    //	"ro.hardware",
    //	"ro.product.board",
    //	"ro.board.platform",
    //	"ro.arch",
    //	"ro.btstack"
    // }
    // 例如：ro.hardware为mt6735，就会从system/lib/hw中找gralloc.mt6735.so，找到就跳转到found了
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT; i++) {
        if (property_get(variant_keys[i], prop, NULL) == 0) {
            continue;
        }
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;

        }
    }

    /* Nothing found, try the default */
    if (hw_module_exists(path, sizeof(path), name, "default") == 0) {
        goto found;
    }

    return -ENOENT;

found:
    /* load the module, if this fails, we're doomed, and we should not try
     * to load a different variant. */
    return load(class_id, path, module);// 找到so库了就加载
}
```
**从一个项目的实际操作方式来看，hardware/libhardware/module下的模块编译生成的都是一个default，例如这里的Gralloc模块生成的是gralloc.default.so，而通常芯片上提供过来的代码里，会包含一个打包好的so，以mt6735为例，vendor下会有一个gralloc.mt6735.so，在上面查找so时会通过ro.hardware属性拿到mt6735这个值，从而去找gralloc.mt6735.so，而后去加载，default的这个so其实是用不到的。而实际操作中，如果删除掉system/lib/hw和system/lib64/hw下的gralloc.mt6735.so，开机时查找到的so确实是default，但是这个so不足以让图形系统运行起来，应该是芯片上根据其硬件有对Gralloc模块做过定制。所以这里hal层的分析只能依赖Android原生代码做一个理论上设计分析。**    

```c  
/**
 * Load the file defined by the variant and if successful
 * return the dlopen handle and the hmi.
 * @return 0 = success, !0 = failure.
 */
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status;
    void *handle;
    struct hw_module_t *hmi;

    /*
     * load the symbols resolving undefined symbols before
     * dlopen returns. Since RTLD_GLOBAL is not or'd in with
     * RTLD_NOW the external symbols will not be global
     */
    handle = dlopen(path, RTLD_NOW);// dlopen打开so
    if (handle == NULL) {
        char const *err_str = dlerror();
        ALOGE("load: module=%s\n%s", path, err_str?err_str:"unknown");
        status = -EINVAL;
        goto done;
    }

    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);// 从so中拿到hmi(hardware module info)
    if (hmi == NULL) {
        ALOGE("load: couldn't find symbol %s", sym);
        status = -EINVAL;
        goto done;
    }

    /* Check that the id matches */
    if (strcmp(id, hmi->id) != 0) {// 比对下拿到的hardware id和传入的是否一致
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;// hmi中的dso赋值为打开的so库，dso(dynamic so)

    /* success */
    status = 0;

    done:
    if (status != 0) {
        hmi = NULL;
        if (handle != NULL) {
            dlclose(handle);
            handle = NULL;
        }
    } else {
        ALOGV("loaded HAL id=%s path=%s hmi=%p handle=%p",
                id, path, *pHmi, handle);
    }

    *pHmi = hmi;

    return status;
}
```

#### 1.2 HMI

这里加载的是Gralloc模块，来看下加载后出来的hmi到底是个什么结构。在调用dlsym时传入了HAL_MODULE_INFO_SYM_AS_STR变量，来从打开的gralloc.xxx.so中找到hmi，这里的HAL_MODULE_INFO_SYM_AS_STR是一个宏，定义在hardware/libhardware/include/hardware/hardware.h

```c  
/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"
```

而在同一处另有一个宏的定义为

```c

/**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI
```

在hardware/libhardware/modules/gralloc/gralloc.cpp中定义一处结构体

```cpp
struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {
        .common = {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods
        },
        .registerBuffer = gralloc_register_buffer,
        .unregisterBuffer = gralloc_unregister_buffer,
        .lock = gralloc_lock,
        .unlock = gralloc_unlock,
    },
    .framebuffer = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .currentBuffer = 0,
};
```

从结果来看会加载到这处的结构体，作为hmi返回。那么如何通过HAL_MODULE_INFO_SYM和HAL_MODULE_INFO_SYM_AS_STR这两个宏找到的这个结构体信息，应该是dlsym这个函数的玄机了，这里暂不做深究。我们只需知道所有hal模块这部分都是这样实现的。  

#### 1.3 private_module_t

加载部分还有另外一个问题，我们在load函数中看到dlsym函数返回的类型被强制转换成了struct hw_module_t *类型，而gralloc.cpp中的却是private_module_t结构体类型，这两者有什么关系呢？有些文章中会说是结构体的继承，而继承通常是面向对象的用语，更准确的说是结构体的扩展。来看下private_module_t类型的定义，在hardware/libhardware/modules/gralloc/gralloc_priv.h

```cpp
struct private_module_t {
    gralloc_module_t base;
	// 以下描述帧缓冲区的属性
    private_handle_t* framebuffer; // 指向系统帧缓冲区的句柄
    uint32_t flags;// 双缓冲的标志，如果支持双缓冲，PAGE_FLIP位就等于1，否则的话，就等于0
    uint32_t numBuffers;// 表示系统帧缓冲区包含有多少个图形缓冲区。一个帧缓冲区包含有多少个图形缓冲区是与它的可视分辨率以及虚拟分辨率的大小有关的。例如，如果一个帧缓冲区的可视分辨率为800 x 600，而虚拟分辨率为1600 x 600，那么这个帧缓冲区就可以包含有两个图形缓冲区
    uint32_t bufferMask;// 记录系统帧缓冲区中的图形缓冲区的使用情况。例如，假设系统帧缓冲区有两个图形缓冲区，这时候成员变量bufferMask就有四种取值，分别是二进制的00、01、10和11，其中，00分别表示两个图缓冲区都是空闲的，01表示第1个图形缓冲区已经分配出去，而第2个图形缓冲区是空闲的，10表示第1个图形缓冲区是空闲的，而第2个图形缓冲区已经分配出去，11表示两个图缓冲区都已经分配出去
    pthread_mutex_t lock;// 互斥锁，用来保护结构体private_module_t的并行访问
    buffer_handle_t currentBuffer;// 用来描述当前正在被渲染的图形缓冲区
    int pmem_master;
    void* pmem_master_base;

    struct fb_var_screeninfo info;// 没用
    struct fb_fix_screeninfo finfo;// 没用
    float xdpi;// 用来描述设备显示屏在宽度和高度上的密度，即每英寸有多少个像素点
    float ydpi;
    float fps;// 描述显示屏的刷新频率，它的单位的fps，即每秒帧数
};
```

#### 1.4 private_handle_t

附上private_handle_t的结构体定义，也在gralloc_priv.h中  

```h
#ifdef __cplusplus
struct private_handle_t : public native_handle {
#else
struct private_handle_t {
    struct native_handle nativeHandle;
#endif

    enum {
        PRIV_FLAGS_FRAMEBUFFER = 0x00000001
    };

    // file-descriptors
    int     fd;
    // ints
    int     magic;
    int     flags;
    int     size;
    int     offset;

    // FIXME: the attributes below should be out-of-line
    uint64_t base __attribute__((aligned(8)));
    int     pid;

#ifdef __cplusplus
```





#### 1.5 gralloc_module_t

包含了一个gralloc_module_t的变量base，看gralloc_module_t结构体，定义在hardware/libhardware/include/hardware/gralloc.h中，注释太长，篇幅原因就删掉了  

```c
/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
typedef struct gralloc_module_t {
    struct hw_module_t common;
    int (*registerBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    int (*unregisterBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle); 
    int (*lock)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr);
   
    int (*unlock)(struct gralloc_module_t const* module,
            buffer_handle_t handle);


    /* reserved for future use */
    int (*perform)(struct gralloc_module_t const* module,
            int operation, ... );

    int (*lock_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr);
    int (*lockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr, int fenceFd);
    int (*unlockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int* fenceFd);

    int (*lockAsync_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr, int fenceFd);

    /* reserved for future use */
    void* reserved_proc[3];
} gralloc_module_t;
```

#### 1.6 hw_module_t

其中又包含了hw_module_t的一个变量common，来看hw_module_t的定义，在hardware/libhardware/include/hardware/hardware.h中，同样删除了一些注释    

```c
/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;
    uint16_t module_api_version;
#define version_major module_api_version
    uint16_t hal_api_version;
#define version_minor hal_api_version

    /** Identifier of module */
    const char *id;

    /** Name of this module */
    const char *name;

    /** Author/owner/implementor of the module */
    const char *author;

    /** Modules methods */
    struct hw_module_methods_t* methods;

    /** module's dso */
    void* dso;

#ifdef __LP64__
    uint64_t reserved[32-7];
#else
    /** padding to 128 bytes, reserved for future use */
    uint32_t reserved[32-7];
#endif

} hw_module_t;
```

这个就是结构体的扩展关系，注意被扩展的那个结构体变量需要放在新结构体起始地址开始。现在再回过头来看hardware/libhardware/modules/gralloc/gralloc.cpp中struct private_module_t HAL_MODULE_INFO_SYM这个结构体的初始化。  

```c
struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {//gralloc_module_t结构体
        .common = {//hw_module_t结构体
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods//重要的模块方法成员变量，这里会指向gralloc模块的打开设备方法
        },
        .registerBuffer = gralloc_register_buffer,//注册buffer
        .unregisterBuffer = gralloc_unregister_buffer,//注销buffer
        .lock = gralloc_lock,//lock和unlock
        .unlock = gralloc_unlock,
    },
    .framebuffer = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .currentBuffer = 0,
};
```

来看下hw_module_t结构体中最重要的struct hw_module_methods_t* methods变量，在初始化时被赋予了什么。  

```c
static struct hw_module_methods_t gralloc_module_methods = {
        .open = gralloc_device_open
};
```

```c
int gralloc_device_open(const hw_module_t* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
        gralloc_context_t *dev;
        dev = (gralloc_context_t*)malloc(sizeof(*dev));

        /* initialize our state here */
        memset(dev, 0, sizeof(*dev));

        /* initialize the procs */
        dev->device.common.tag = HARDWARE_DEVICE_TAG;
        dev->device.common.version = 0;
        dev->device.common.module = const_cast<hw_module_t*>(module);
        dev->device.common.close = gralloc_close;

        dev->device.alloc   = gralloc_alloc;
        dev->device.free    = gralloc_free;

        *device = &dev->device.common;
        status = 0;
    } else {
        status = fb_device_open(module, name, device);
    }
    return status;
}
```

也就是说hw_module_methods_t method这个结构体对象的open变量指向了gralloc_device_open函数，这个起始就是打开fb设备的函数入口。  

#### 1.7 buffer_handle_t

这里需要补充一个结构体的定义，buffer_handle_t是用来描述一块图形缓冲区的数据结构。它的定义在system/core/include/system/window.h中`typedef const native_handle_t* buffer_handle_t;`  

而native_handle_t定义在system/core/include/cutils/native_handle.h中  

```h
typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file-descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
    int data[0];        /* numFds + numInts ints */
} native_handle_t;
```



### 2 打开gralloc和fb设备

Gralloc模块实现的原理第二步，打开gralloc和fb设备。gralloc设备是用来分配图形缓冲区的，分配之后将缓冲区的内存地址映射到用户进程中，用户进程就可以将绘制的内容写入进去。  

#### 2.1 打开gralloc

Gralloc模块在在文件hardware/libhardware/include/hardware/gralloc.h中定义了一个帮助函数gralloc_open，用来打开gralloc设备（上层应用调用的入口），如下所示：  

```c
/** convenience API for opening and closing a supported device */

static inline int gralloc_open(const struct hw_module_t* module, 
        struct alloc_device_t** device) {
    return module->methods->open(module, 
            GRALLOC_HARDWARE_GPU0, (struct hw_device_t**)device);
}
```

这里module->methods->open就会走到gralloc_device_open函数，通过判断传入的name，这边传入的是GRALLOC_HARDWARE_GPU0，所以会走的代码如下  

```c
int gralloc_device_open(const hw_module_t* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {// 走这里
        // 创建一个gralloc_context_t变量，并初始化
        gralloc_context_t *dev;
        dev = (gralloc_context_t*)malloc(sizeof(*dev));

        /* initialize our state here */
        memset(dev, 0, sizeof(*dev));

        /* initialize the procs */
        // 初始化gralloc_context_t的device成员变量
        dev->device.common.tag = HARDWARE_DEVICE_TAG;
        dev->device.common.version = 0;
        dev->device.common.module = const_cast<hw_module_t*>(module);
        dev->device.common.close = gralloc_close;

        dev->device.alloc   = gralloc_alloc;
        dev->device.free    = gralloc_free;

        *device = &dev->device.common;
        status = 0;
    } else {// 不走
        status = fb_device_open(module, name, device);
    }
    return status;
}
```

来看下gralloc_context_t结构体的定义，在hardware/libhardware/modules/gralloc/gralloc.cpp中  

```cpp
struct gralloc_context_t {
    alloc_device_t  device;
    /* our private data here */
};
```

里面也就一个alloc_device_t类型的成员，接着来看alloc_device_t的定义，在hardware/libhardware/include/hardware/gralloc.h中  

```cpp
/**
 * Every device data structure must begin with hw_device_t
 * followed by module specific public methods and attributes.
 */

typedef struct alloc_device_t {
    struct hw_device_t common;

    /* 
     * (*alloc)() Allocates a buffer in graphic memory with the requested
     * parameters and returns a buffer_handle_t and the stride in pixels to
     * allow the implementation to satisfy hardware constraints on the width
     * of a pixmap (eg: it may have to be multiple of 8 pixels). 
     * The CALLER TAKES OWNERSHIP of the buffer_handle_t.
     *
     * If format is HAL_PIXEL_FORMAT_YCbCr_420_888, the returned stride must be
     * 0, since the actual strides are available from the android_ycbcr
     * structure.
     * 
     * Returns 0 on success or -errno on error.
     */
    
    int (*alloc)(struct alloc_device_t* dev,
            int w, int h, int format, int usage,
            buffer_handle_t* handle, int* stride);

    /*
     * (*free)() Frees a previously allocated buffer. 
     * Behavior is undefined if the buffer is still mapped in any process,
     * but shall not result in termination of the program or security breaches
     * (allowing a process to get access to another process' buffers).
     * THIS FUNCTION TAKES OWNERSHIP of the buffer_handle_t which becomes
     * invalid after the call. 
     * 
     * Returns 0 on success or -errno on error.
     */
    int (*free)(struct alloc_device_t* dev,
            buffer_handle_t handle);

    /* This hook is OPTIONAL.
     *
     * If non NULL it will be caused by SurfaceFlinger on dumpsys
     */
    void (*dump)(struct alloc_device_t *dev, char *buff, int buff_len);

    void* reserved_proc[7];
} alloc_device_t;
```

里面包含了一个用来描述gralloc设备的hw_device_t结构体以及分配和释放图形缓冲区的函数alloc和free。这三个变量在gralloc_device_open函数（打开gralloc设备操作）中被初始化。alloc和free分别被初始化为  

```cpp
		dev->device.alloc   = gralloc_alloc;
        dev->device.free    = gralloc_free;
```

这个为后面分配和释放图形缓冲区的操作做好了准备。  

#### 2.2 打开fb  

fb设备的打开辅助函数在hardware/libhardware/include/hardware/fb.h中  

```h
static inline int framebuffer_open(const struct hw_module_t* module,
        struct framebuffer_device_t** device) {
    return module->methods->open(module,
            GRALLOC_HARDWARE_FB0, (struct hw_device_t**)device);
}
```

这里module->methods->open依然走到gralloc.cpp的gralloc_device_open函数，而这边传入的name就是fb设备的了GRALLOC_HARDWARE_FB0，所以会走的代码如下。  

```c
int gralloc_device_open(const hw_module_t* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {// 这里是gralloc的，不走
        ...
    } else {// 走这个
        status = fb_device_open(module, name, device);
    }
    return status;
}
```

fb_device_open定义在hardware/libhardware/module/gralloc/framebuffer.cpp中  

```cpp
int fb_device_open(hw_module_t const* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_FB0)) {
        /* initialize our state here */
        fb_context_t *dev = (fb_context_t*)malloc(sizeof(*dev));
        memset(dev, 0, sizeof(*dev));// 初始化一个fb_context_t变量

        /* initialize the procs */
        dev->device.common.tag = HARDWARE_DEVICE_TAG;
        dev->device.common.version = 0;
        dev->device.common.module = const_cast<hw_module_t*>(module);
        dev->device.common.close = fb_close;
        dev->device.setSwapInterval = fb_setSwapInterval;
        dev->device.post            = fb_post;
        dev->device.setUpdateRect = 0;

        private_module_t* m = (private_module_t*)module;
        status = mapFrameBuffer(m);
        if (status >= 0) {
            int stride = m->finfo.line_length / (m->info.bits_per_pixel >> 3);
            int format = (m->info.bits_per_pixel == 32)
                         ? (m->info.red.offset ? HAL_PIXEL_FORMAT_BGRA_8888 : HAL_PIXEL_FORMAT_RGBX_8888)
                         : HAL_PIXEL_FORMAT_RGB_565;
            const_cast<uint32_t&>(dev->device.flags) = 0;
            const_cast<uint32_t&>(dev->device.width) = m->info.xres;
            const_cast<uint32_t&>(dev->device.height) = m->info.yres;
            const_cast<int&>(dev->device.stride) = stride;
            const_cast<int&>(dev->device.format) = format;
            const_cast<float&>(dev->device.xdpi) = m->xdpi;
            const_cast<float&>(dev->device.ydpi) = m->ydpi;
            const_cast<float&>(dev->device.fps) = m->fps;
            const_cast<int&>(dev->device.minSwapInterval) = 1;
            const_cast<int&>(dev->device.maxSwapInterval) = 1;
            *device = &dev->device.common;
        }
    }
    return status;
}
```

##### 2.2.1 fb_context_t  

fb_context_t结构体定义在framebuffer.cpp中，用来描述fb设备的句柄  

```cpp
struct fb_context_t {
    framebuffer_device_t  device;
};
```

fb_context_t中只包含了一个framebuffer_device_t的变量，来看下framebuffer_device_t结构体  

##### 2.2.2 framebuffer_device_t

framebuffer_device_t结构体定义在fb.h中，用来描述fb设备的  

```h
typedef struct framebuffer_device_t {
    /**
     * Common methods of the framebuffer device.  This *must* be the first member of
     * framebuffer_device_t as users of this structure will cast a hw_device_t to
     * framebuffer_device_t pointer in contexts where it's known the hw_device_t references a
     * framebuffer_device_t.
     */
    struct hw_device_t common;// 一些common的部分，继承了hw_device_t

    /* flags describing some attributes of the framebuffer */
    const uint32_t  flags;

    /* dimensions of the framebuffer in pixels */
    const uint32_t  width;
    const uint32_t  height;

    /* frambuffer stride in pixels */
    const int       stride;

    /* framebuffer pixel format */
    const int       format;

    /* resolution of the framebuffer's display panel in pixel per inch*/
    const float     xdpi;
    const float     ydpi;

    /* framebuffer's display panel refresh rate in frames per second */
    const float     fps;

    /* min swap interval supported by this framebuffer */
    const int       minSwapInterval;

    /* max swap interval supported by this framebuffer */
    const int       maxSwapInterval;

    /* Number of framebuffers supported*/
    const int       numFramebuffers;

    int reserved[7];

    /*
     * requests a specific swap-interval (same definition than EGL)
     *
     * Returns 0 on success or -errno on error.
     */
    // 设置双缓存交换的间隔
    int (*setSwapInterval)(struct framebuffer_device_t* window,
            int interval);

    /*
     * This hook is OPTIONAL.
     *
     * It is non NULL If the framebuffer driver supports "update-on-demand"
     * and the given rectangle is the area of the screen that gets
     * updated during (*post)().
     *
     * This is useful on devices that are able to DMA only a portion of
     * the screen to the display panel, upon demand -- as opposed to
     * constantly refreshing the panel 60 times per second, for instance.
     *
     * Only the area defined by this rectangle is guaranteed to be valid, that
     * is, the driver is not allowed to post anything outside of this
     * rectangle.
     *
     * The rectangle evaluated during (*post)() and specifies which area
     * of the buffer passed in (*post)() shall to be posted.
     *
     * return -EINVAL if width or height <=0, or if left or top < 0
     */
    int (*setUpdateRect)(struct framebuffer_device_t* window,
            int left, int top, int width, int height);

    /*
     * Post <buffer> to the display (display it on the screen)
     * The buffer must have been allocated with the
     *   GRALLOC_USAGE_HW_FB usage flag.
     * buffer must be the same width and height as the display and must NOT
     * be locked.
     *
     * The buffer is shown during the next VSYNC.
     *
     * If the same buffer is posted again (possibly after some other buffer),
     * post() will block until the the first post is completed.
     *
     * Internally, post() is expected to lock the buffer so that a
     * subsequent call to gralloc_module_t::(*lock)() with USAGE_RENDER or
     * USAGE_*_WRITE will block until it is safe; that is typically once this
     * buffer is shown and another buffer has been posted.
     *
     * Returns 0 on success or -errno on error.
     */
    // 渲染图形缓冲区
    int (*post)(struct framebuffer_device_t* dev, buffer_handle_t buffer);


    /*
     * The (*compositionComplete)() method must be called after the
     * compositor has finished issuing GL commands for client buffers.
     */

    int (*compositionComplete)(struct framebuffer_device_t* dev);

    /*
     * This hook is OPTIONAL.
     *
     * If non NULL it will be caused by SurfaceFlinger on dumpsys
     */
    void (*dump)(struct framebuffer_device_t* dev, char *buff, int buff_len);

    /*
     * (*enableScreen)() is used to either blank (enable=0) or
     * unblank (enable=1) the screen this framebuffer is attached to.
     *
     * Returns 0 on success or -errno on error.
     */
    int (*enableScreen)(struct framebuffer_device_t* dev, int enable);

    void* reserved_proc[6];

} framebuffer_device_t;
```

在fb_device_open中可以看到第一部分能初始化的只有common和几个函数指针fb_setSwapInterval和fb_post。后面又初始化了一个private_module_t对象m，private_module_t结构体前文已经介绍，是用来描述gralloc设备图形缓冲区的一个数据结构，通过mapFrameBuffer函数来初始化。  

##### 2.2.3 mapFrameBuffer

```cpp
static int mapFrameBuffer(struct private_module_t* module)
{
    pthread_mutex_lock(&module->lock);
    int err = mapFrameBufferLocked(module);
    pthread_mutex_unlock(&module->lock);
    return err;
}
```

```cpp
int mapFrameBufferLocked(struct private_module_t* module)
{
    // already initialized...
    if (module->framebuffer) {// framebuffer是用来描述系统缓冲区的变量，不为空的就是已经初始化过了
        return 0;
    }
        
    char const * const device_template[] = {
            "/dev/graphics/fb%u",
            "/dev/fb%u",
            0 };

    int fd = -1;
    int i=0;
    char name[64];

    // 1. name=/dev/graphics/fb0
    // 2. 打开/dev/graphics/fb0节点
    // 3. 如果成功了fd就不是-1，while就结束了
    // 4. 如果打开/dev/graphics/fb0失败了，就去尝试打开/dev/fb0
    while ((fd==-1) && device_template[i]) {
        snprintf(name, 64, device_template[i], 0);
        fd = open(name, O_RDWR, 0);
        i++;
    }
    // /dev/graphics/fb0和/dev/fb0两个节点都尝试打开失败了
    if (fd < 0)
        return -errno;
```
```cpp
	// 通过ioctl从fb0节点读取到FSCREENINFO
    struct fb_fix_screeninfo finfo;
    if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1)
        return -errno;

    // // 通过ioctl从fb0节点读取到VSCREENINFO
    struct fb_var_screeninfo info;
    if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1)
        return -errno;

    info.reserved[0] = 0;
    info.reserved[1] = 0;
    info.reserved[2] = 0;
    info.xoffset = 0;
    info.yoffset = 0;
    info.activate = FB_ACTIVATE_NOW;
```
###### 2.2.2.1 fb_var_screeninfo

fb_var_screeninfo是一个描述屏幕属性的结构体  

```h
struct fb_var_screeninfo {
	__u32 xres;			/* visible resolution		*/
	__u32 yres;
	__u32 xres_virtual;		/* virtual resolution		*/
	__u32 yres_virtual;
	__u32 xoffset;			/* offset from virtual to visible */
	__u32 yoffset;			/* resolution			*/

	__u32 bits_per_pixel;		/* guess what			*/
	__u32 grayscale;		/* 0 = color, 1 = grayscale,	*/
					/* >1 = FOURCC			*/
	struct fb_bitfield red;		/* bitfield in fb mem if true color, */
	struct fb_bitfield green;	/* else only length is significant */
	struct fb_bitfield blue;
	struct fb_bitfield transp;	/* transparency			*/	

	__u32 nonstd;			/* != 0 Non standard pixel format */

	__u32 activate;			/* see FB_ACTIVATE_*		*/

	__u32 height;			/* height of picture in mm    */
	__u32 width;			/* width of picture in mm     */

	__u32 accel_flags;		/* (OBSOLETE) see fb_info.flags */

	/* Timing: All values in pixclocks, except pixclock (of course) */
	__u32 pixclock;			/* pixel clock in ps (pico seconds) */
	__u32 left_margin;		/* time from sync to picture	*/
	__u32 right_margin;		/* time from picture to sync	*/
	__u32 upper_margin;		/* time from sync to picture	*/
	__u32 lower_margin;
	__u32 hsync_len;		/* length of horizontal sync	*/
	__u32 vsync_len;		/* length of vertical sync	*/
	__u32 sync;			/* see FB_SYNC_*		*/
	__u32 vmode;			/* see FB_VMODE_*		*/
	__u32 rotate;			/* angle we rotate counter clockwise */
	__u32 colorspace;		/* colorspace for FOURCC-based modes */
	__u32 reserved[4];		/* Reserved for future compatibility */
};
```

需要说明的几个成员是xres、yres，这两个表示屏幕的分辨率，xres_virtual、yres_virtual，这两个表示虚拟分辨率。如何来理解这个虚拟分辨率？这个牵涉到多缓冲技术，假如屏幕的分辨率是600\*800，那么保持width不变，我们的虚拟分辨率未600\*1600，也就是说虚拟缓冲区的分辨率是两倍的屏幕分辨率。如此一来，系统帧缓冲区就可以划分出两块，用来单独渲染一个图形缓冲区的内容。可以交替的做显示和渲染的工作，这个就是双缓冲技术，而目前在Android4.1（Jelly Bean）之后已经是三缓冲了，目的是解决双缓冲带来的显示卡顿问题。更详细的可以参考我介绍[UI Performance](http://chendongqi.me/2017/03/08/android_perf_patterns_ui/)的文章。  

```cpp
/*
     * Request NUM_BUFFERS screens (at lest 2 for page flipping)
     */
    info.yres_virtual = info.yres * NUM_BUFFERS;// NUM_BUFFERS=3

	// enum {
    // PAGE_FLIP = 0x00000001,
    // LOCKED = 0x00000002
	// };
    uint32_t flags = PAGE_FLIP;// 设置多缓冲标志
#if USE_PAN_DISPLAY
    if (ioctl(fd, FBIOPAN_DISPLAY, &info) == -1) {
        ALOGW("FBIOPAN_DISPLAY failed, page flipping not supported");
#else
    if (ioctl(fd, FBIOPUT_VSCREENINFO, &info) == -1) {
        ALOGW("FBIOPUT_VSCREENINFO failed, page flipping not supported");
#endif
        info.yres_virtual = info.yres;
        flags &= ~PAGE_FLIP;
    }

    if (info.yres_virtual < info.yres * 2) {// 如果
        // we need at least 2 for page-flipping
        info.yres_virtual = info.yres;
        flags &= ~PAGE_FLIP;
        ALOGW("page flipping not supported (yres_virtual=%d, requested=%d)",
                info.yres_virtual, info.yres*2);
    }
```

这段代码最终是通过IO控制命令FBIOPUT_VSCREENINFO来设置设备显示屏的虚拟分辨率以及像素格式的。如果设置失败，即调用函数ioctl的返回值等于-1，那么很可能是因为系统帧缓冲区在硬件上不支持双缓冲，因此，接下来的代码就会重新将显示屏的虚拟分辨率的高度值设置为可视分辨率的高度值，并且将变量flags的PAGE_FLIP位置为0。  

另一方面，如果调用函数ioctl成功，但是最终获得的显示屏的虚拟分辨率（ioctl FBIOPUT_VSCREENINFO的时候info的内容还会改变，例如有可能最终拿到的yres_virtual=1.5*yres）的高度值小于可视分辨率的高度值的2倍，那么也说明系统帧缓冲区在硬件上不支持双缓冲。在这种情况下，接下来的代码也会重新将显示屏的虚拟分辨率的高度值设置为可视分辨率的高度值，并且将变量flags的PAGE_FLIP位置为0。  
```cpp
if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1)
        return -errno;

    uint64_t  refreshQuotient =
    (
            uint64_t( info.upper_margin + info.lower_margin + info.yres )
            * ( info.left_margin  + info.right_margin + info.xres )
            * info.pixclock
    );

    /* Beware, info.pixclock might be 0 under emulation, so avoid a
     * division-by-0 here (SIGFPE on ARM) */
    int refreshRate = refreshQuotient > 0 ? (int)(1000000000000000LLU / refreshQuotient) : 0;

    if (refreshRate == 0) {
        // bleagh, bad info from the driver
        refreshRate = 60*1000;  // 60 Hz
    }
```

###### 2.2.2.2 屏幕刷新频率

这段代码再次通过IO控制命令FBIOGET_VSCREENINFO来获得系统帧缓冲区的可变属性信息，并且保存在fb_var_screeninfo结构体info中，接下来再计算设备显示屏的刷新频率。显示屏的刷新频率与显示屏的扫描时序相关。显示屏的扫描时序可以参考内核源代码目录下的kernel-3.10/Documentation/fb/framebuffer.txt文件（很多底层的介绍都可以参考Documentation下的文档说明）。我们结合下图来简单说明上述代码是如何计算显示屏的刷新频率的。    

```txt
  +----------+---------------------------------------------+----------+-------+
  |          |                ↑                            |          |       |
  |          |                |upper_margin                |          |       |
  |          |                ↓                            |          |       |
  +----------###############################################----------+-------+
  |          #                ↑                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |   left   #                |                            #  right   | hsync |
  |  margin  #                |       xres                 #  margin  |  len  |
  |<-------->#<---------------+--------------------------->#<-------->|<----->|
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |yres                        #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                |                            #          |       |
  |          #                ↓                            #          |       |
  +----------###############################################----------+-------+
  |          |                ↑                            |          |       |
  |          |                |lower_margin                |          |       |
  |          |                ↓                            |          |       |
  +----------+---------------------------------------------+----------+-------+
  |          |                ↑                            |          |       |
  |          |                |vsync_len                   |          |       |
  |          |                ↓                            |          |       |
  +----------+---------------------------------------------+----------+-------+


```

中间由xres和yres组成的区域即为显示屏的图形绘制区，在绘制区的上、下、左和右分别有四个边距upper_margin、lower_margin、left_margin和right_margin。此外，在显示屏的最右边以及最下边还有一个水平同步区域hsync_len和一个垂直同步区域vsync_len。电子枪按照从左到右、从上到下的顺序来显示屏中打点，从而可以将要渲染的图形显示在屏幕中。前面所提到的区域信息分别保存在fb_var_screnninfo结构体info的成员变量xres、yres、upper_margin、lower_margin、left_margin、right_margin、hsync_len和vsync_len。  
电子枪每在xres和yres所组成的区域中打一个点所花费的时间记录在fb_var_screnninfo结构体info的成员变量pixclock，单位为pico seconds，即10E-12秒。  
电子枪从左到右扫描完成一行之后，都会处理关闭状态，并且会重新折回到左边去。由于电子枪在从右到左折回的过程中不需要打点，因此，这个过程会比从左到右扫描屏幕的过程要快，这个折回的时间大概就等于在xres和yres所组成的区域扫描（left_margin+right_margin）个点的时间。这样，我们就可以认为每渲染一行需要的时间为（xres + left_margin + right_margin）* pixclock。  
同样，电子枪从上到下扫描完成显示屏之后，需要从右下角折回到左上角去，折回的时间大概等于在xres和yres所组成的区域中扫描（upper_margin + lower_margin）行所需要的时间。这样，我们就可以认为每渲染一屏图形所需要的时间等于在xres和yres所组成的区域中扫描（yres + upper_margin + lower_margin）行所需要的时间。由于在xres和yres所组成的区域中扫描一行所需要的时间为（xres + left_margin + right_margin）* pixclock，因此，每渲染一屏图形所需要的总时间就等于（yres + upper_margin + lower_margin）* （xres + left_margin + right_margin）* pixclock。  
每渲染一屏图形需要的总时间经过计算之后，就保存在变量refreshQuotient中。注意，变量refreshQuotient所描述的时间的单位为1E-12秒。这样，将变量refreshQuotient的值倒过来，就可以得到设备显示屏的刷新频率。将这个频率值乘以10E15次方之后，就得到一个单位为10E-3 HZ的刷新频率，保存在变量refreshRate中。    
当Android系统在模拟器运行的时候，保存在fb_var_screnninfo结构体info的成员变量pixclock中的值可能等于0。在这种情况下，前面计算得到的变量refreshRate的值就会等于0。在这种情况下，接下来的代码会将变量refreshRate的值设置为60 * 1000 * 10E-3 HZ，即将显示屏的刷新频率设置为60HZ。    

###### 2.2.2.3 屏幕密度

```cpp
    if (int(info.width) <= 0 || int(info.height) <= 0) {
        // the driver doesn't return that information
        // default to 160 dpi
        info.width  = ((info.xres * 25.4f)/160.0f + 0.5f);
        info.height = ((info.yres * 25.4f)/160.0f + 0.5f);
    }

    float xdpi = (info.xres * 25.4f) / info.width;
    float ydpi = (info.yres * 25.4f) / info.height;
    float fps  = refreshRate / 1000.0f;
```
这段代码首先计算显示屏的密度，即每英寸有多少个像素点，分别宽度和高度两个维度，分别保存在变量xdpi和ydpi中。注意，fb_var_screeninfo结构体info的成员变量width和height用来描述显示屏的宽度和高度，它们是以毫米（mm）为单位的。所以这里计算公式的理解为：xres表示横向有n个像素，除以屏幕横向尺寸width，算出了每毫米有多少个像素点，因为1英寸等于25.4毫米，所以乘以25.4就算出了每英寸多少个像素点，也就是这里的像素密度dpi的值。  

这一段最后又换算了刷新频率，直接除以1000，拿到了fps的值。  

```cpp
ALOGI(   "using (fd=%d)\n"
            "id           = %s\n"
            "xres         = %d px\n"
            "yres         = %d px\n"
            "xres_virtual = %d px\n"
            "yres_virtual = %d px\n"
            "bpp          = %d\n"
            "r            = %2u:%u\n"
            "g            = %2u:%u\n"
            "b            = %2u:%u\n",
            fd,
            finfo.id,
            info.xres,
            info.yres,
            info.xres_virtual,
            info.yres_virtual,
            info.bits_per_pixel,
            info.red.offset, info.red.length,
            info.green.offset, info.green.length,
            info.blue.offset, info.blue.length
    );

    ALOGI(   "width        = %d mm (%f dpi)\n"
            "height       = %d mm (%f dpi)\n"
            "refresh rate = %.2f Hz\n",
            info.width,  xdpi,
            info.height, ydpi,
            fps
    );


    if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1)
        return -errno;

    if (finfo.smem_len <= 0)
        return -errno;


    module->flags = flags;
    module->info = info;
    module->finfo = finfo;
    module->xdpi = xdpi;
    module->ydpi = ydpi;
    module->fps = fps;
```
这段代码再次通过IO控制命令FBIOGET_FSCREENINFO来获得系统帧缓冲区的固定信息，并且保存在fb_fix_screeninfo结构体finfo中，接下来再使用fb_fix_screeninfo结构体finfo以及前面得到的系统帧缓冲区的其它信息来初始化参数module所描述的一个private_module_t结构体  
```cpp
	/*
     * map the framebuffer
     */

    int err;
    size_t fbSize = roundUpToPageSize(finfo.line_length * info.yres_virtual);
    module->framebuffer = new private_handle_t(dup(fd), fbSize, 0);

    module->numBuffers = info.yres_virtual / info.yres;
    module->bufferMask = 0;

    void* vaddr = mmap(0, fbSize, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    if (vaddr == MAP_FAILED) {
        ALOGE("Error mapping the framebuffer (%s)", strerror(errno));
        return -errno;
    }
    module->framebuffer->base = intptr_t(vaddr);
    memset(vaddr, 0, fbSize);
    return 0;
}
```
表达式finfo.line_length * info.yres_virtual计算的是整个系统帧缓冲区的大小，它的值等于显示屏行数（虚拟分辨率的高度值，info.yres_virtual）乘以每一行所占用的字节数（finfo.line_length）。函数roundUpToPageSize用来将整个系统帧缓冲区的大小对齐到页面边界。对齐后的大小保存在变量fbSize中。  
表达式finfo.yres_virtual / info.yres计算的是整个系统帧缓冲区可以划分为多少个图形缓冲区来使用，这个数值保存在参数module所描述的一个private_module_t结构体的成员变量nmBuffers中。参数module所描述的一个private_module_t结构体的另外一个成员变量bufferMask的值接着被设置为0，表示系统帧缓冲区中的所有图形缓冲区都是处于空闲状态，即它们可以分配出去给应用程序使用。  
系统帧缓冲区是通过调用函数mmap来映射到当前进程的地址空间来的。映射后得到的地址空间使用一个private_handle_t结构体来描述，这个结构体的成员变量base保存的即为系统帧缓冲区在当前进程的地址空间中的起始地址。这样，Gralloc模块以后就可以从这块地址空间中分配图形缓冲区给当前进程使用。  
至此，fb设备的打开过程就分析完成了。在打开fb设备的过程中，Gralloc模块还完成了对系统帧缓冲区的初始化工作。接下来我们继续分析Gralloc模块是如何分配图形缓冲区给用户空间的应用程序使用的。  

### 3 分配图形缓冲区

#### 3.1 gralloc_alloc

在应用层继打开gralloc设备和fb设备之后，会去调用alloc_device_t结构体的alloc方法，在gralloc_device_open函数打开gralloc设备时，将这里的alloc函数赋值成了gralloc_alloc函数，位于hardware/libhardware/module/gralloc/gralloc.cpp中  

```cpp
static int gralloc_alloc(alloc_device_t* dev,
        int w, int h, int format, int usage,
        buffer_handle_t* pHandle, int* pStride)
{
    if (!pHandle || !pStride)
        return -EINVAL;

    size_t size, stride;

    int align = 4;
    int bpp = 0;
    switch (format) {
        case HAL_PIXEL_FORMAT_RGBA_8888:
        case HAL_PIXEL_FORMAT_RGBX_8888:
        case HAL_PIXEL_FORMAT_BGRA_8888:
            bpp = 4;// 需要四个字节来存一个像素
            break;
        case HAL_PIXEL_FORMAT_RGB_888:
            bpp = 3;// 需要三个字节
            break;
        case HAL_PIXEL_FORMAT_RGB_565:
        case HAL_PIXEL_FORMAT_RAW_SENSOR:
            bpp = 2;// 需要连个字节
            break;
        default:
            return -EINVAL;
    }
    // w:屏幕宽度，横向有多少个像素
    // 按照4位对齐，~(align-1)就是00，&运算之后，末两位肯定是00，也就是变成了4的倍数
    // good method
    size_t bpr = (w*bpp + (align-1)) & ~(align-1);
    size = bpr * h;// 一个屏幕的像素需要的位数
    stride = bpr / bpp;// 一个图像缓冲区一行有多少个像素

    int err;
    if (usage & GRALLOC_USAGE_HW_FB) {// usage表示这次分配缓冲区的用途
        err = gralloc_alloc_framebuffer(dev, size, usage, pHandle);// 系统帧缓冲区渲染
    } else {
        err = gralloc_alloc_buffer(dev, size, usage, pHandle);// 内存中分配图形缓冲区
    }

    if (err < 0) {
        return err;
    }

    *pStride = stride;
    return 0;
}
```

参数format用来描述要分配的图形缓冲区的颜色格式。当format值等于HAL_PIXEL_FORMAT_RGBA_8888、HAL_PIXEL_FORMAT_RGBX_8888或者HAL_PIXEL_FORMAT_BGRA_8888的时候，一个像素需要使用32位来表示，即4个字节。当format值等于HAL_PIXEL_FORMAT_RGB_888的时候，一个像素需要使用24位来描述，即3个字节。当format值等于HAL_PIXEL_FORMAT_RGB_565、HAL_PIXEL_FORMAT_RGBA_5551或者HAL_PIXEL_FORMAT_RGBA_4444的时候，一个像需要使用16位来描述，即2个字节。最终一个像素需要使用的字节数保存在变量bpp中。    

usage参数表示要分配的图形缓冲区的用途。如果是用来在系统帧缓冲区中渲染的，即参数usage的GRALLOC_USAGE_HW_FB位为1，那么就在系统帧缓冲区中分配。否则在内存中分配。在内存中分配的图形缓冲区，绘制完之后也是要拷贝到系统帧缓冲区去渲染的。  

函数gralloc_alloc_framebuffer用来在系统帧缓冲区中分配图形缓冲区，而函数gralloc_alloc_buffer用来在内存中分配，接下来分析这两个函数。  

#### 3.2 gralloc_alloc_framebuffer

gralloc_alloc_framebuffer实现在hardware/libhardware/modules/gralloc/gralloc.cpp中  

```cpp
static int gralloc_alloc_framebuffer(alloc_device_t* dev,
        size_t size, int usage, buffer_handle_t* pHandle)
{
    private_module_t* m = reinterpret_cast<private_module_t*>(
            dev->common.module);
    pthread_mutex_lock(&m->lock);
    int err = gralloc_alloc_framebuffer_locked(dev, size, usage, pHandle);
    pthread_mutex_unlock(&m->lock);
    return err;
}
```

锁操作后直接调用同个文件下的gralloc_alloc_framebuffer_locked函数  

```cpp
static int gralloc_alloc_framebuffer_locked(alloc_device_t* dev,
        size_t size, int usage, buffer_handle_t* pHandle)
{
    private_module_t* m = reinterpret_cast<private_module_t*>(
            dev->common.module);

    // allocate the framebuffer
    if (m->framebuffer == NULL) {// 传入的usage是分配系统帧缓冲区，所以如果这里的fb是NULL的话，必须先初始化，初始化的部分参考2.2.3
        // initialize the framebuffer, the framebuffer is mapped once
        // and forever.
        int err = mapFrameBufferLocked(m);
        if (err < 0) {
            return err;
        }
    }

    const uint32_t bufferMask = m->bufferMask;// 用来描述帧缓冲区的使用情况(当前使用的哪个缓冲)
    const uint32_t numBuffers = m->numBuffers;// 描述一个帧缓冲区可以划分几个图形缓冲区
    const size_t bufferSize = m->finfo.line_length * m->info.yres;// 显示一屏内容占用的内存
    if (numBuffers == 1) {
        // If we have only one buffer, we never use page-flipping. Instead,
        // we return a regular buffer which will be memcpy'ed to the main
        // screen when post is called.
        int newUsage = (usage & ~GRALLOC_USAGE_HW_FB) | GRALLOC_USAGE_HW_2D;
        return gralloc_alloc_buffer(dev, bufferSize, newUsage, pHandle);
    }

    // 多缓冲都分配完了
    if (bufferMask >= ((1LU<<numBuffers)-1)) {
        // We ran out of buffers.
        return -ENOMEM;
    }

    // create a "fake" handles for it
    intptr_t vaddr = intptr_t(m->framebuffer->base);// 帧缓冲区的基地址
    private_handle_t* hnd = new private_handle_t(dup(m->framebuffer->fd), size,
            private_handle_t::PRIV_FLAGS_FRAMEBUFFER);

    // find a free slot
    for (uint32_t i=0 ; i<numBuffers ; i++) {
        if ((bufferMask & (1LU<<i)) == 0) {// 从bufferMask低位开始寻找是0的位，表示没有分配出去的那个图形缓冲区
            m->bufferMask |= (1LU<<i);// 设置bufferMask中该为以被分配出去
            break;
        }
        // 没找到的话这里的vaddr地址就加上一片图形缓冲区的内存
        // 开始vaddr是fb的基地址，找到合适的图形缓冲区之后vaddr就是这片缓冲区的起始地址
        vaddr += bufferSize;
    }
    
    hnd->base = vaddr;// 修改hnd中描述的基地址
    hnd->offset = vaddr - intptr_t(m->framebuffer->base);// 修改hnd中描述的地址偏移
    *pHandle = hnd;

    return 0;
}
```

private_module_t结构体参考1.3，包含了一个gralloc_module_t结构体以及后面用来描述系统帧缓冲区的信息。  

如果numBuffers为1，那么也就是说系统帧缓冲区不支持多缓冲技术，那么分配图形缓冲区的请求就无法在帧缓冲区中进行，还是需要分配到内存中。系统帧缓冲区始终用来渲染和post到屏幕上。这里分配到内存图形缓冲区的事就还是交给gralloc_alloc_buffer函数来完成。  

`bufferMask >= ((1LU<<numBuffers)-1)`表示系统帧缓冲区的所有图形缓冲区都分配出去了，这里就无法再分配。来看下这个是怎么计算的。假设系统缓冲区的帧缓冲区数量为2，也就说是双缓冲，那么bufferMask就只有三种可能值：0（两片缓冲都空着）;1（使用了一片）;2（两片都用了）。那么`1LU<<numBuffers)-1`就等于二进制的11，也就是3。那么如果`bufferMask>=3`时，肯定就是超过可分配缓冲区数量了。  

`m->framebuffer->base`参考1.4中，根据m重新构造了一个缓冲区的句柄hnd，来描述这块即将分配出去的缓冲区。注意这里的一个标志PRIV_FLAGS_FRAMEBUFFER，表示这块缓冲区是在系统帧缓冲区中的。  

接下来就是从帧缓冲区中寻找一块合适的图形缓冲区分配出来了。这里的注释“find a free slot”，slot也就意味着图形缓冲区。分配完之后会将这片图形缓冲区的地址和相对帧缓冲区的偏移地址保存起来返回给调用者，这样用户程序就可以直接将要渲染的图形内容拷贝到这个地址上去，也就相当于直接将图形渲染到了系统帧缓冲区中。    

分配帧缓冲区的函数就结束了，再来看下分配图形缓冲区的函数gralloc_alloc_buffer  

#### 3.3 gralloc_alloc_buffer

gralloc_alloc_buffer函数在hardware/libhardware/modules/gralloc/gralloc.cpp中    

```cpp
static int gralloc_alloc_buffer(alloc_device_t* dev,
        size_t size, int /*usage*/, buffer_handle_t* pHandle)
{
    int err = 0;
    int fd = -1;

    size = roundUpToPageSize(size);
    
    fd = ashmem_create_region("gralloc-buffer", size);
    if (fd < 0) {
        ALOGE("couldn't create ashmem (%s)", strerror(-errno));
        err = -errno;
    }

    if (err == 0) {
        private_handle_t* hnd = new private_handle_t(fd, size, 0);
        gralloc_module_t* module = reinterpret_cast<gralloc_module_t*>(
                dev->common.module);
        err = mapBuffer(module, hnd);
        if (err == 0) {
            *pHandle = hnd;
        }
    }
    
    ALOGE_IF(err, "gralloc failed err=%s", strerror(-err));
    
    return err;
}
```

roundUpToPageSize函数是定义在hardware/libhardware/modules/gralloc/gr.h中的一个inline函数，起作用是做字节对齐。  

```h
inline size_t roundUpToPageSize(size_t x) {
    return (x + (PAGE_SIZE-1)) & ~(PAGE_SIZE-1);
}
```

看着是不是特别眼熟，在3.1中刚出现过这种对其的方法，这里就是把size按照PAGE_SIZE来对齐。目的就是在做内存对齐。  

接下来调用ashmem_create_region函数来创建一块匿名共享内存。关于匿名共享内存的知识可以参考罗升阳的博客[Android系统匿名共享内存（Anonymous Shared Memory）C++调用接口分析](https://blog.csdn.net/luoshengyang/article/details/6939890)和[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6651971)   

接着在这块匿名共享内存上分配图形缓冲区，同在系统帧缓冲区中分配一样，这里也是用private_handle_t结构体来保存，差别是这边的flag是0，前面提到的PRIV_FLAGS_FRAMEBUFFER值是0x00000001。  

再接下来需要把这块缓冲区的内存映射到用户空间中，用户空间的程序才能使用绘制内容，来看下这块映射的实现mapBuffer函数。该函数在hardware/libhardware/modules/gralloc/mapper.cpp中，此函数处理的都是gralloc模块中跟内存操作相关的事，分配，释放，注册，注销等。    

```cpp
int mapBuffer(gralloc_module_t const* module,
        private_handle_t* hnd)
{
    void* vaddr;
    return gralloc_map(module, hnd, &vaddr);
}
```

#### 3.4 gralloc_map

直接调用了gralloc_map来映射空间，继续接着看  

```cpp
static int gralloc_map(gralloc_module_t const* /*module*/,
        buffer_handle_t handle,
        void** vaddr)
{
    private_handle_t* hnd = (private_handle_t*)handle;
    if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER)) {// 非帧缓冲区
        size_t size = hnd->size;
        void* mappedAddress = mmap(0, size,
                PROT_READ|PROT_WRITE, MAP_SHARED, hnd->fd, 0);
        if (mappedAddress == MAP_FAILED) {
            ALOGE("Could not mmap %s", strerror(errno));
            return -errno;
        }
        hnd->base = uintptr_t(mappedAddress) + hnd->offset;
        //ALOGD("gralloc_map() succeeded fd=%d, off=%d, size=%d, vaddr=%p",
        //        hnd->fd, hnd->offset, hnd->size, mappedAddress);
    }
    *vaddr = (void*)hnd->base;
    return 0;
}
```

在1.7中已经描述了buffer_hanlde_t的结构，其实就是的private_handle的指针，而这里的private_handle_t就是继承的private_handle，所以这里有个强转类型的过程。    

如果图形缓冲区不是分配在帧缓冲区里的话（flags中PRIV_FLAGS_FRAMEBUFFER这一位不是1），也就是说分配的匿名共享内存在内存中，那么我们需要把这块内存（hnd->fd描述）通过mmap映射到当前的用户空间中来。  

在这块匿名内存中，图形缓冲区可能只占了部分地址空间，所以要拿到图形缓冲区在内存中的地址，需要把匿名缓冲区的内存地址加上一个偏移量（hnd->offset），然后存到hnd->base中。最终把这个地址返回给用户空间。  

那么如果图形缓冲区是分配在帧缓冲区的情况呢？由于在系统帧缓冲区中分配的图形缓冲区只在SurfaceFlinger服务中使用，而SurfaceFlinger服务在初始化系统帧缓冲区的时候，已经将系统帧缓冲区映射到自己所在的进程中来了，因此，函数gralloc_map如果发现要注册的图形缓冲区是在系统帧缓冲区分配的时候，那么就不需要再执行映射图形缓冲区的操作了。  

### 4 释放图形缓冲区

图形缓冲区使用完了之后就需要释放，释放的函数为gralloc_free，在hardware/libhardware/modules/gralloc/gralloc.cpp中  

#### 4.1 gralloc_free  

```cpp
static int gralloc_free(alloc_device_t* dev,
        buffer_handle_t handle)
{
    if (private_handle_t::validate(handle) < 0)// 校验是不是gralloc分配的
        return -EINVAL;

    private_handle_t const* hnd = reinterpret_cast<private_handle_t const*>(handle);// 转成private_handle_t类型
    if (hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) {// 如果是帧缓冲区中分配的
        // free this buffer
        private_module_t* m = reinterpret_cast<private_module_t*>(
                dev->common.module);
        const size_t bufferSize = m->finfo.line_length * m->info.yres;
        int index = (hnd->base - m->framebuffer->base) / bufferSize;
        m->bufferMask &= ~(1<<index); 
    } else { 
        gralloc_module_t* module = reinterpret_cast<gralloc_module_t*>(
                dev->common.module);
        terminateBuffer(module, const_cast<private_handle_t*>(hnd));
    }

    close(hnd->fd);
    delete hnd;
    return 0;
}
```

如果是帧缓冲区中分配的图形缓冲区，那么要知道这个图形缓冲区是帧缓冲区中的第几个。一个图形缓冲区的内存大小是一屏的高和一行的字节数乘积。而`hnd->base - m->framebuffer->base`则计算的是分配的这个图形缓冲区的内存地址减去帧缓冲区的基地址，也就是这片图形缓冲区相对于帧缓冲区基地址偏移了多少个内存，除以bufferSize之后就是偏移了多少个图形缓冲区，也就是这片图形缓冲区的index。然后将`m->bufferMask`中的index位置0，标示这片帧缓冲区空闲了。  

如果图形缓冲区是分配在内存中的，那么调用terminateBuffer来释放。  

#### 4.2 terminateBuffer

该函数在hardware/libhardware/modules/gralloc/mapper.cpp  

```cpp
int terminateBuffer(gralloc_module_t const* module,
        private_handle_t* hnd)
{
    if (hnd->base) {
        // this buffer was mapped, unmap it now
        gralloc_unmap(module, hnd);
    }

    return 0;
}
```

#### 4.3 gralloc_unmap

```cpp
static int gralloc_unmap(gralloc_module_t const* /*module*/,
        buffer_handle_t handle)
{
    private_handle_t* hnd = (private_handle_t*)handle;
    if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER)) {
        void* base = (void*)hnd->base;
        size_t size = hnd->size;
        //ALOGD("unmapping from %p, size=%d", base, size);
        if (munmap(base, size) < 0) {
            ALOGE("Could not unmap %s", strerror(errno));
        }
    }
    hnd->base = 0;
    return 0;
}
```

这个函数的实现与前面所分析的函数gralloc_map的实现是类似的，只不过它执行的是相反的操作，即将解除一个指定的图形缓冲区在当前进程的地址空间中的映射，从而完成对这个图形缓冲区的注销工作。  

### 5 注册图形缓冲区

SurfaceFlinger负责所有图形缓冲区的分配，而前面分析过的，分配图形缓冲区的过程中，在匿名共享内存创建出来之后会映射到请求分配的地址空间中去，那么请求的是SurfaceFlinger，所以映射的地址也是到了它的进程中。而用户程序并不是跑在SurfaceFlinger中的，他们之间通过binder来做通讯。那么如何让用户程序拿到这块共享内存呢，就需要用户程序调用一次这里的注册图形缓冲区函数来将这块内存地址再映射到自己的进程内存中。  

注册图形缓冲区，用户程序是通过1.5中的grallock_molude_t结构体变量来操作的，调用成员变量registerBuffer函数指针。而在1.2中hmi初始化的时候可以看到`.registerBuffer = gralloc_register_buffer,`registerBuffer是被赋值成gralloc_register_buffer。所以注册图形缓冲区的函数实现就在gralloc_register_buffer中，位于hardware/libhardware/modules/gralloc/mapper.cpp。  

```cpp
int gralloc_register_buffer(gralloc_module_t const* module,
        buffer_handle_t handle)
{
    if (private_handle_t::validate(handle) < 0)
        return -EINVAL;

    // *** WARNING WARNING WARNING ***
    //
    // If a buffer handle is passed from the process that allocated it to a
    // different process, and then back to the allocator process, we will
    // create a second mapping of the buffer. If the process reads and writes
    // through both mappings, normal memory ordering guarantees may be
    // violated, depending on the processor cache implementation*.
    //
    // If you are deriving a new gralloc implementation from this code, don't
    // do this. A "real" gralloc should provide a single reference-counted
    // mapping for each buffer in a process.
    //
    // In the current system, there is one case that needs a buffer to be
    // registered in the same process that allocated it. The SurfaceFlinger
    // process acts as the IGraphicBufferAlloc Binder provider, so all gralloc
    // allocations happen in its process. After returning the buffer handle to
    // the IGraphicBufferAlloc client, SurfaceFlinger free's its handle to the
    // buffer (unmapping it from the SurfaceFlinger process). If
    // SurfaceFlinger later acts as the producer end of the buffer queue the
    // buffer belongs to, it will get a new handle to the buffer in response
    // to IGraphicBufferProducer::requestBuffer(). Like any buffer handle
    // received through Binder, the SurfaceFlinger process will register it.
    // Since it already freed its original handle, it will only end up with
    // one mapping to the buffer and there will be no problem.
    //
    // Currently SurfaceFlinger only acts as a buffer producer for a remote
    // consumer when taking screenshots and when using virtual displays.
    //
    // Eventually, each application should be allowed to make its own gralloc
    // allocations, solving the problem. Also, this ashmem-based gralloc
    // should go away, replaced with a real ion-based gralloc.
    //
    // * Specifically, associative virtually-indexed caches are likely to have
    //   problems. Most modern L1 caches fit that description.

    private_handle_t* hnd = (private_handle_t*)handle;
    ALOGD_IF(hnd->pid == getpid(),
            "Registering a buffer in the process that created it. "
            "This may cause memory ordering problems.");

    void *vaddr;
    return gralloc_map(module, handle, &vaddr);
}
```

hnd->pid是分配图形缓冲区的进程，如果它和当前进程相同，那么就没有必要再做一次映射了。  

否则就调用gralloc_map再映射一次，gralloc_map函数的实现参考3.4。  

这里的注释很重要，可惜我没看懂，后面实战中再领会吧。  

### 6 注销图形缓冲区

有注册就有注销，原理同注册一样，直接看下代码    

```cpp
int gralloc_unregister_buffer(gralloc_module_t const* module,
        buffer_handle_t handle)
{
    if (private_handle_t::validate(handle) < 0)
        return -EINVAL;

    private_handle_t* hnd = (private_handle_t*)handle;
    if (hnd->base)
        gralloc_unmap(module, handle);

    return 0;
}
```

先验证缓冲区是否是gralloc模块分配的，后面就是直接调用unmap函数来做释放了，参考4.3。  

### 7 渲染

用户空间的程序把图形绘制到图形缓冲区之后，还需要调用渲染函数来将其在帧缓冲区中渲染，然后才能显示到屏幕上。用户缓冲区是通过framebuffer_device_t这个结构体来操作渲染的，参考2.2.2中的post函数。  

在2.2打开fb设备时，会对framebuffer_device_t这个结构体做初始化，其中就包含了将post函数赋值为fb_post：`dev->device.post            = fb_post;`。代码位于hardware/libhardware/modules/gralloc/framebuffer.cpp。  

```cpp
static int fb_post(struct framebuffer_device_t* dev, buffer_handle_t buffer)
{
    if (private_handle_t::validate(buffer) < 0)
        return -EINVAL;

    fb_context_t* ctx = (fb_context_t*)dev;

    private_handle_t const* hnd = reinterpret_cast<private_handle_t const*>(buffer);
    private_module_t* m = reinterpret_cast<private_module_t*>(
            dev->common.module);

    if (hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) {// 如果分配的是帧缓冲区
        const size_t offset = hnd->base - m->framebuffer->base;
        m->info.activate = FB_ACTIVATE_VBL;
        m->info.yoffset = offset / m->finfo.line_length;
        if (ioctl(m->framebuffer->fd, FBIOPUT_VSCREENINFO, &m->info) == -1) {
            ALOGE("FBIOPUT_VSCREENINFO failed");
            m->base.unlock(&m->base, buffer); 
            return -errno;
        }
        m->currentBuffer = buffer;
        
    } else {
        // If we can't do the page_flip, just copy the buffer to the front 
        // FIXME: use copybit HAL instead of memcpy
        
        void* fb_vaddr;
        void* buffer_vaddr;
        
        m->base.lock(&m->base, m->framebuffer, 
                GRALLOC_USAGE_SW_WRITE_RARELY, 
                0, 0, m->info.xres, m->info.yres,
                &fb_vaddr);// 拿到帧缓冲区的地址，放到fb_vaddr

        m->base.lock(&m->base, buffer, 
                GRALLOC_USAGE_SW_READ_RARELY, 
                0, 0, m->info.xres, m->info.yres,
                &buffer_vaddr);// 图形缓冲区的地址放到buffer_vaddr

        memcpy(fb_vaddr, buffer_vaddr, m->finfo.line_length * m->info.yres);// 拷贝
        
        m->base.unlock(&m->base, buffer); 
        m->base.unlock(&m->base, m->framebuffer); 
    }
    
    return 0;
}
```

如果分配的图形缓冲区就在帧缓冲区中，那么无法拷贝内容了，但是需要告诉帧缓冲区设备将要渲染的图形缓冲区作为当前系统的输出图形缓冲区，这样才可以将要渲染的图形缓冲区的内容绘制到设备显示屏中。这个操作是通过ioctl命令FBIOPUT_VSCREENINFO来实现的。    
在执行IO控制命令FBIOPUT_VSCREENINFO之前，还会将作为参数的fb_var_screeninfo结构体的成员变量activate的值设置FB_ACTIVATE_VBL，表示要等到下一个垂直同步事件出现时，再将当前要渲染的图形缓冲区的内容绘制出来。这样做的目的是避免出现屏幕闪烁，即避免前后两个图形缓冲区的内容各有一部分同时出现屏幕中。     

 成功地执行完成IO控制命令FBIOPUT_VSCREENINFO之后，函数还会将当前被渲染的图形缓冲区保存在private_module_t结构体m的成员变量currentBuffer中，以便可以记录当前被渲染的图形缓冲区是哪一个。  

如果是分配在内存的图形缓冲区中，那么需要把这块地址的内容拷贝到帧缓冲区中。首先通过lock函数拿到系统帧缓冲区的地址，放到fb_vaddr中，然后同样的操作，拿到图形缓冲区中的地址放到buffer_vaddr中。然后就是将要渲染的图形缓冲区的内容通过memcpy来拷贝到帧缓冲区中去了。  

那么内容拷贝到帧缓冲区之后呢？为什么没有将帧缓冲区的切换到当前的显示缓冲区？这个答案还未知，猜想可能是后面继续走的话又会回到之前的流程，分配的图形缓冲区就存在与帧缓冲区中了。这个后面可以调试下看看日志。  

图形缓冲区的渲染也就到这里，最后再来看下渲染中用到的lock和unlock函数。  

在操作图形缓冲区和帧缓冲区时，需要用lock函数来拿到这两块地址，我想通过这个函数的设计名称可以猜想，应该有锁定内存而不让其他进程访问的这一层意思吧。来看下实际的方法。在1.2中可以看到lock和unlock实际的实现是gralloc_lock和gralloc_unlock，代码位于hardware/libhardware/modules/gralloc/mapper.cpp中。  

### 8 lock&unlock

```cpp
int gralloc_lock(gralloc_module_t const* /*module*/,
        buffer_handle_t handle, int /*usage*/,
        int /*l*/, int /*t*/, int /*w*/, int /*h*/,
        void** vaddr)
{
    // this is called when a buffer is being locked for software
    // access. in thin implementation we have nothing to do since
    // not synchronization with the h/w is needed.
    // typically this is used to wait for the h/w to finish with
    // this buffer if relevant. the data cache may need to be
    // flushed or invalidated depending on the usage bits and the
    // hardware.

    if (private_handle_t::validate(handle) < 0)
        return -EINVAL;

    private_handle_t* hnd = (private_handle_t*)handle;
    *vaddr = (void*)hnd->base;
    return 0;
}
```

实际上没有什么锁定的操作，只是把hnd描述的基地址传给了最后一个参数vaddr，所以我们看到通过lock参数拿缓冲区地址时，传入的最后一个参数是fb_vaddr和buffer_vaddr，最后地址就是存放在这两个变量中。  

```cpp
int gralloc_unlock(gralloc_module_t const* /*module*/,
        buffer_handle_t handle)
{
    // we're done with a software buffer. nothing to do in this
    // implementation. typically this is used to flush the data cache.

    if (private_handle_t::validate(handle) < 0)
        return -EINVAL;
    return 0;
}
```

lock中并未有什么真正的锁定操作，所以unlock这边的实现就很没有必要了，什么都没有。  

### 9 附1

本篇中无数次提到了图形缓冲区和帧缓冲区，那么这两个概念的区别是什么？ 以下摘自老罗博客回复 

>  图形缓冲区（GraphicBuffer）是Android的术语，帧缓冲区（FrameBuffer）是Linux Kernel的术语。帧缓冲区是对显卡的抽象，图形缓冲区可以是在帧缓冲区上分配，即显存内部分配，也可以在普通内存上分配，或者其它地方分配，这完全是由厂商提供的gralloc驱动决定。   



### 10 附2 

本篇中剖析了gralloc模块中各个函数的实现，那么如何将这些函数串起来使用？用户程序是怎么调用这些函数的呢？以下通过网上的一篇案例给予一定的说明。  

```cpp
framebuffer_device_t* fbDev;
alloc_device_t* grDev;

hw_module_t const* module;
buffer_handle_t handle;
gralloc_module_t const *mAllocMod;
void* vaddr;
int stride;
int err;
if (hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module) == 0) {//加载gralloc模块
       
        err = framebuffer_open(module, &fbDev); //打开fb设备
        if(err) LOGE("couldn't open framebuffer HAL (%s)", strerror(-err));
        err = gralloc_open(module, &grDev); //打开gralloc设备
        if(err) LOGE("couldn't open gralloc HAL (%s)", strerror(-err));
        
        err = grDev->alloc(grDev, display.w, display.h, HAL_PIXEL_FORMAT_RGBA_8888, GRALLOC_USAGE_HW_FB/*决定申请的是系统图形内存还是普通内存*/, &handle, &stride); //分配图形缓冲区
        // err = grDev->alloc(grDev, 1024, 600, HAL_PIXEL_FORMAT_RGBA_8888, 0/*决定申请的是系统图形内存还是普通内存*/, &handle, &stride); //分配图形缓冲区
        
        mAllocMod = (gralloc_module_t const *)module;
        err = mAllocMod->registerBuffer(mAllocMod, handle); //映射内存到进程中
        
        
        err = mAllocMod->lock(mAllocMod, handle, HAL_PIXEL_FORMAT_RGBA_8888, 0, 0, display.w, display.h, &vaddr);
        LOGE("++++++++++++++++> vaddr = %p\n", vaddr);

         err = mAllocMod->lock(mAllocMod, handle, HAL_PIXEL_FORMAT_RGBA_8888, 0, 0, 1024, 600, &vaddr);
        LOGE("++++++++++++++++> vaddr = %p\n", vaddr);
        //这就绘图即可，将绘制的图的内存直接拷贝到vaddr里面即可
        bitmap.lockPixels();
    　　canvas->drawPath(path, paint);
    　　memcpy(vaddr, bitmap.getPixels(), bitmap.getSize()); 
    　　bitmap.unlockPixels(); 
        err = mAllocMod->unlock(mAllocMod, handle);
        
        err = fbDev->post(fbDev, handle); //图形缓冲区的渲染 
        
        err = mAllocMod->unregisterBuffer(mAllocMod, handle); //解除映射内存
        
        grDev->free(grDev, handle);//释放图形缓冲区
        
        gralloc_close(grDev);//关闭gralloc设备
        framebuffer_close(fbDev);//关闭fb设备 
    } 
```

原文出处[Android gralloc模块实例](<https://www.cnblogs.com/winfu/p/>)  




