---
layout:      post
title:      "Android性能优化之Systrace分析基础"
subtitle:   "google性能分析工具systrace的使用指南"
navcolor:   "invert"
date:       2017-02-18
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - Performance
    - systrace
---

### 适用平台

Android Version： 6.0及以上
Platform： 通用

### 1. 分析前准备

最好是在userdebug或者开启root权限的user版本设备上来进行systrace的抓取。在实际的操作用，我一直使用了普通的user版本，需要的信息也都抓到了。

### 2. ubuntu环境下systrace抓取的方法

这里介绍两种比较通用的使用方法，命令行式的和图形界面式的。当然还有其他封装的工具，比如MTK出的手机端的apk以及GAT工具中的插件等，这里就不介绍，我们只需掌握最原始和最基础的，万变不离其宗。
这里使用的Android SDK工具的版本需要升级的6.0也就是level 24。

#### 2.1 命令行式

使用的为Android SDK目录下的systrace.py脚本来做

```Bash
android-sdk-linux/platform-tools/systrace/systrace.py -b xxx -t xxx -o trace.html <tag>
```
介绍下参数的使用：
-b是定义buffer size，单位为KB，-t定义的是抓取的时间，一般抓5秒的systrace就将buffer size设成20480差不多，如果buffer size过小有可能导致信息丢失。-o后面为目标文件的名称，最重要和困难的就是tag，那么有几种tag呢？可以使用命令

```Bash
systrace.py -l
```
来查看，而tag的使用场景在后面会继续介绍。

#### 2.2 图形界面式

google还提供了图形界面式的抓取工具，先启动位于android-sdk-linux/tools目录下的monitor工具，如果sdk的环境变量都配置正确可以直接在命令行中执行：

```Bash
monitor
```
如果有安装android studio，也可以在studio启动monitor，连接手机，会启动如下工具界面(部分相关截图)
![monitor ui](https:chendongqi.github.io/blog/img/2017-02-18-systrace_base/monitor_ui.png)
然后点击红框部分的按钮会启动如下systrace抓取前的配置界面
![systrace config](https:chendongqi.github.io/blog/img/2017-02-18-systrace_base/systrace_config.png)
图中的tag配置为LG项目中测试Touch Performance时抓取的配置，适用于抓取界面启动相关信息的测试项。配置完成之后点击ok，然后就在手机中做测试操作，预设时间结束之后自动保存systrace信息。

#### 2.3 TAG使用

不论是命令式或者是图形界面式的抓取方法，都需要设置抓取的TAG信息。如果是和MTK合作，配合他们抓取systrace的情况，那么可以根据MTK的需求来做，但是要脱离MTK的阴影就必须能自行对各个TAG的含义有一定的了解来做配置，这里介绍各个TAG的使用场景，而真实情况下通常是多种TAG的组合，这就需要经验的积累，例如上文中提到过的Touch Performance测试的适用场景。
命令模式下，TAG的查看可以通过

```Bash
systrace.py -list
```
来取得
![TAG list](https:chendongqi.github.io/blog/img/2017-02-18-systrace_base/tag_list.png)
而图形界面下就方便很多，直接给你列出来做勾选就可以，各个TAG的含义如下：
--这三个是cpu信息，必须带上
    freq sched idle
--测试列表滑动，桌面滑动等流畅性问题
&emsp;&emsp;    gfx input view
--若在上面基础上还需要分析HWUI
&emsp;&emsp;    gfx input view hwui
--测试app启动或者进入某个界面速度
&emsp;&emsp;    gfx input view am wm res
--怀疑有GC或者IO导致卡顿
&emsp;&emsp;    gfx input view dalvik disk
--怀疑有power相关，量灭屏慢等
&emsp;&emsp;    gfx inpu view res am wm power

### 3. systrace的打开方式

systrace是google提供的一种log形式，其打开也比较特殊，需要用chrome浏览器来查看(google的工具生态圈也是越来越强大)。前面提到了在Android 6.0设备上，抓取时需要将SDK版本也升级到最新，那么查看时我们也需要最新版本的Chrome浏览器，否则可能会出现打不开，信息丢失等情况。我这里使用了ubuntu下的Version 53.0.2785.143，配合升级到android-24的SDK版本。
工具准备齐全之后来介绍下正确的打开方式，在android 6.0之前的版本中，非常普遍的方式是右击抓到的systrace文件(例如trace.html)，然后选择用chrome打开，这样就万事大吉了。然后在Android 6.0以后我不建议这么做，因为继续沿用这种方式的话我自己是遭遇过打不开和信息丢失的情况，而正确的方式应该是这样的：

1. 启动Chrome

2. 在地址栏输入chrome://tracing/

3. 点击load，然后选择需要加载的systrace文件就可以了

效果如下图：
![systrace view](https:chendongqi.github.io/blog/img/2017-02-18-systrace_base/systrace_view.png)
全是密密麻麻各种颜色的线条，需要有极强的耐心从这一堆东西里找出线索来，基本的键盘操作方式如下：
A和D--左右移动
W和S--缩放
鼠标操作为界面上悬浮的工具栏
![operate tool](https:chendongqi.github.io/blog/img/2017-02-18-systrace_base/operate_tool.png)
从上至下以此为选择、移动、缩放和测量(时间间隔)的工具，需要先选中，后使用。

#### 4. 如何分析

已经介绍了抓取和打开systrace以及几个基本的操作，那么如何使用systrace来分析问题呢？这是个非常大且困难的课题，也是我本人学习系统异常、性能等问题分析以来遇到的最艰涩的一种分析手段。
我也就我至今接触和掌握比较熟练的两个例子来做部分的介绍，可以参考：
后续补充:
---应用启动时间分析
----drag动作耗时分析






       
