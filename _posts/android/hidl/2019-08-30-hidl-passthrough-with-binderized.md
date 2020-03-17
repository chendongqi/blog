---
layout:      post
title:      "HIDL系列一"
subtitle:   "深入理解直通式和绑定式"
navcolor:   "invert"
date:       2019-08-30
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

hidl的实现分为了直通式(passthough)和绑定式(binderized)，google官网和其他论坛上对这两种方式做了一定的描述，但是以我等悟性，实践出真知。本篇来详解下直通式和绑定式的一些细节。  

# 1 Treble

很多人可能像我一样，第一个问题就会问什么是绑定式，什么是直通式？其实这个应该算是个究极问题，内容太多，难以回答。我们把它拆开来，从整个事情的起源开始，不如先来问问为什么要有HIDL。  

原始资料参考google的官网[Android Architecture](https://source.android.google.cn/devices/architecture)  

在Android8.0中google重新架构了framework，称之为Treble项目。目的是为了让系统更简单，更快速，以及开发商可以以更小的代价升级到新的Android版本。在这个新的架构中，定义了HIDL语言来做了调用HAL层的接口，从而实现对于HAL层的调用，调用者和实现可以做分离。  

在这种架构下，开发商或者芯片商编译的HAL放置到专用的vendor分区下，framework部分还是放置在它自己的分区中。也就能够实现Treble项目中的只升级framework部分，而HAL层的实现无需再重新编译和升级。     

在Android7.x和更早的版本中，设备制造商必须更新所有的vendor和framework部分  

![treble_blog_before](https://chendongqi.github.io/blog/img/2019-08-30-hidl/treble_blog_before.png)  

在Android8.0及以后的版本中，因为设计了HIDL是的HAL的使用和实现分离，而HAL层的接口稳定之后实现部分就无需再做更新，所以可以保持vendor分区不做变更而只升级framework部分。  

![treble_blog_after](https://chendongqi.github.io/blog/img/2019-08-30-hidl/treble_blog_after.png)    

# 2 HIDL

Treble就是HIDL诞生的契机，但是HAL的模块如果要在Android8.0上线时就完成所有模块从传统方式向HIDL的方式转换，显然对设备生产商来说是不现实的。于是，HIDL也就变化出来直通式（passthrough）和绑定式（binderized）之分。  

![treble_cpp_legacy_hal_progression](https://chendongqi.github.io/blog/img/2019-08-30-hidl/treble_cpp_legacy_hal_progression.png)   

1. 图中第一部分：为旧版的实现方式，framework的进程去打开HAL模块的so库来调用HAL的实现。    
2. 图中第二部分： 通过直通式HIDL调用到HIDL的实现层，而这里的实现并非真正去完成HAL层的实现，而是去调用旧版的HAL实现，最终实现HAL层的功能。   
3. 图中第三部分：通过绑定式HIDL连接到HIDL的服务，然后在服务中去打开HIDL中的实现so，在so中再去操作真正HAL的实现。    
4. 图中第四部分：通过绑定式HIDL直接连接到HIDL的服务，而服务中也做了真正的HAL实现。  

这个也就是google规划的HIDL应该的发展历程，直通式只是过度，而最终的目标是完成所有HAL实现的迁移，以绑定式HIDL的方式来从客户端连接到服务端，而老式的HAL实现也都将迁移到服务端中。  

# 3 直通式和绑定式

了解了Treble和HIDL，以及绑定式和直通式的作用，那么我们可以来考虑一个问题了，直通式和绑定式的区别是什么？从上面HIDL的发展图中，通过最直接的对比可以发现第二部分和第三部分的区别在于，直通式和绑定式的差异就是是否存在HW Service。以上帝视角来解释，绑定式HIDL的实现其实也就是binder的方式，所以跟这个绑定式的名字也是契合的。  

下面我们来重点区别几个概念，同一进程直通式（same-process passthrough）、绑定化直通式（binderizing passthrough）和绑定式。  

## 3.1 same-process passthrough

same-process passthrough，同一进程的直通式调用，也就是对应着HIDL发展图中的第二部分的实现方式。HIDL实现部分以一个so库的方式提供，然后在实现处去打开真正要操作的HAL Module的so，将返回的实例封装返回出去。so库的使用者，用户客户端来打开这个HIDL的so库，真正拿到的就是要操作的HAL Module的实例。这一系列都是运行在客户端的进程中，所以也就称之为same-process。  

具体的实现可以参考我实践案例的文章。  

## 3.2 binderizing passthrough

绑定化的直通式其实是一个伪概念，其真实的面目是在直通式的基础上，加上了一个服务来实现绑定式，从而客户端可以使用直通式和绑定式两种方式来调用。绑定化的直通式不是直通式，而是直通&绑定式。  

这里的直通式式的原理同上，为什么要增加一个绑定式？我猜测，正式因为HIDL有这样一个发展过程，所以在直通式向绑定式转变时，又不能一刀切的直接抛弃直通式，就在保留直通式的基础上，为此增加了绑定式。这样可以满足不同客户端既能使用绑定式又能使用直通式来调用到同一个HAL Module。  

那么这里的绑定式是如何增加上去的呢？首先我们需要在HIDL中增加一个服务，这个服务的作用就是去向hwservicemanager去注册HIDL的服务，当客户端去使用时，可以选择绑定式或者直通式的方式来获取到HIDL层实现的实例，而这里HIDL对于HAL Module的封装还是和纯绑定式一样的。

为了支持这里的service能够开机运行，我们需要为其添加rc。    

为了有运行的权限，我们另需要配置selinux。   

具体的案例可以参考我实践案例的文章。   

## 3.3 binderized

这种就是纯绑定式的实现，是HIDL发展的终极形态，在这种方式下，HIDL只会以一个service的形态被编译（不会再有so库）。而服务中也不必再去封装一层对HAL Module的打开操作，服务就是真正的HAL层实现。客户端通过向hwservicemanager查询该服务，拿到服务实例之后就是直接用对应的接口操作HAL Module了。也就是HIDL发展图中的第四部分。  

具体实现可以我实践案例的文章。