---
layout:      post
title:      "一场Android Performance的追根溯源之旅"
subtitle:   "Android Performance Patterns系列学习思考和实践笔记"
navcolor:   "invert"
date:       2017-03-08
author:     "Cheson"
#header-img: "https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_common.png"
catalog: true
tags:
    - Android
    - Performance
    - Android Performance Patterns
---

### 前言
  本篇写在该系列文章之前，作为本系列的前言以及介绍Android Performance Patterns的原始资料的出处，感谢google developer对此的贡献和胡凯大神对该系列资料的翻译和整理。    
  本人在Android系统性能优化上摸爬滚打了一年有余，尽得片鳞只爪，尤觉管中窥豹，自愁前路暗淡。偶得暇兴致起，搜罗各大论坛android性能相关资料，遇到了Android Performance Patterns系列视频和其胡凯对该系列的翻译，初阅之，不胜欣喜，正和我追求原理之本心，弥补拼凑之前零散的性能知识。值此女神佳节留下此篇以开始记录对该系列的学习思考和实践的历程和心得。   
  附上个人在从事安卓以来建立的性能的知识框架。  
  ![Performance_knowledge_architecture.png](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/Performance_knowledge_architecture.png)
  下面将附上原始的google资料地址和胡凯翻译的文章地址，而本人在我此系列的文章中更多的是记录个人的感悟和理解和在程序上的实践以供自己和他人的参考。    

### Android Performance Patterns 1

**google youtube videos：**     
[Android Performance Patterns Season 1](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_s1](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_s1.png)
**胡凯翻译：**    
[Android性能优化典范 - 第1季](http://hukai.me/android-performance-patterns/)
> 2015新年伊始，Google发布了关于Android性能优化典范的专题，一共16个短视频，每个3-5分钟，帮助开发者创建更快更优秀的Android App。课程专题不仅仅介绍了Android系统中有关性能问题的底层工作原理，同时也介绍了如何通过工具来找出性能问题以及提升性能的建议。主要从三个方面展开，Android的渲染机制，内存与GC，电量优化。    

### Android Performance Patterns 2

**google youtube videos：**     
[Android Performance Patterns Season 2](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_s2](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_s2.png)

**胡凯翻译：**    
[Android性能优化典范 - 第2季](http://hukai.me/android-performance-patterns-season-2/)
> Google前几天刚发布了Android性能优化典范第2季的课程，一共20个短视频，包括的内容大致有：电量优化，网络优化，Wear上如何做优化，使用对象池来提高效率，LRU Cache，Bitmap的缩放，缓存，重用，PNG压缩，自定义View的性能，提升设置alpha之后View的渲染性能，以及Lint，StictMode等等工具的使用技巧。    

### Android Performance Patterns 3

**google youtube videos：**     
[Android Performance Patterns Season 3](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_s3](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_s3.png)

**胡凯翻译：**    
[Android性能优化典范 - 第3季](http://hukai.me/android-performance-patterns-season-3/)
> Android性能优化典范的课程最近更新到第三季了，这次一共12个短视频课程，包括的内容大致有：更高效的ArrayMap容器，使用Android系统提供的特殊容器来避免自动装箱，避免使用枚举类型，注意onLowMemory与onTrimMemory的回调，避免内存泄漏，高效的位置更新操作，重复layout操作的性能影响，以及使用Batching，Prefetching优化网络请求，压缩传输数据等等使用技巧。    

### Android Performance Patterns 4

**google youtube videos：**     
[Android Performance Patterns Season 4](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_s4](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_s4.png)

**胡凯翻译：**    
[Android性能优化典范 - 第4季](http://hukai.me/android-performance-patterns-season-4/)
> Android性能优化典范第4季的课程学习笔记终于在2015年的最后一天完成了，文章共17个段落，包含的内容大致有：优化网络请求的行为，优化安装包的资源文件，优化数据传输的效率，性能优化的几大基础原理等等。   

### Android Performance Patterns 5

**google youtube videos：**     
[Android Performance Patterns Season 5](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_s5](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_s5.png)

**胡凯翻译：**    
[Android性能优化典范 - 第5季](http://hukai.me/android-performance-patterns-season-5/)
> 这是Android性能优化典范第5季的课程学习笔记，拖拖拉拉很久，记录分享给大家，请多多包涵担待指正！文章共10个段落，涉及的内容有：多线程并发的性能问题，介绍了AsyncTask，HandlerThread，IntentService与ThreadPool分别适合的使用场景以及各自的使用注意事项，这是一篇了解Android多线程编程不可多得的基础文章，清楚的了解这些Android系统提供的多线程基础组件之间的差异以及优缺点，才能够在项目实战中做出最恰当的选择。     

### Android Performance Patterns 6

**google youtube videos：**     
[Android Performance Patterns Season 6](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)    
![android_perf_patterns_common](https://chendongqi.github.io/blog/img/2017-03-08-android_perf_patterns_overview/android_perf_patterns_common.png)

**胡凯翻译：**    
[Android性能优化典范 - 第6季](http://hukai.me/android-performance-patterns-season-6/)
> 这里是Android性能优化典范第6季的课程学习笔记，从被@知会到有连载更新，这篇学习笔记就一直被惦记着，现在学习记录分享一下，请多多指教包涵！这次一共才6个小段落，涉及的内容主要有：程序启动时间性能优化的三个方面：优化activity的创建过程，优化application对象的启动过程，正确使用启动显屏达到优化程序启动性能的目的。另外还介绍了减少安装包大小的checklist以及如何使用VectorDrawable来减少安装包的大小。         
