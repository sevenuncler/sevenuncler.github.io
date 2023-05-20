---
layout: post
title:  "iOS Instruments - Animation Hitches"
date:   2023-05-20 08:35:51 +0800
categories: iOS Perf
---

> https://developer.apple.com/videos/play/wwdc2021/10252
> https://developer.apple.com/videos/play/wwdc2021/10181
> https://developer.apple.com/documentation/Xcode/analyzing-the-performance-of-your-shipping-app
> https://developer.apple.com/documentation/xcode/analyzing-responsiveness-issues-in-your-shipping-app
> 
> 高刷屏监控优化(字节)：https://blog.csdn.net/ByteDanceTech/article/details/123437098
> 
> Hitch 基本概念：https://developer.apple.com/videos/play/tech-talks/10855/
> 
> 优化 commit phase: https://developer.apple.com/videos/play/tech-talks/10856
> 
> 优化 Render phase:[ https://developer.apple.com/videos/play/tech-talks/10856](https://developer.apple.com/videos/play/tech-talks/10857)

# Hitch (卡顿) 
> A hitch is any time a frame appears on screen later than expected

Hitch 即卡顿，代表任何时候某一帧画面出现晚了。

iOS 画面显示的原理依赖 VSync 垂直同步硬件时钟，VSync 的工作原理，按照刷新帧率如 60hz，每隔16.67ms秒发出一次信号，协调 App、GPU、显示器进行渲染流程。
刷新帧率设备列表与截图
Render Loop 包含5个阶段：Event、Commit、Render Prepare、Render Excute、
核心流程如下：  
第一个 VSync: App 打包视图树信息给 Render Server
第二个 VSync：Render Server 计算出需要绘制纹理数据给 GPU，GPU 绘制放入 buffer 缓存
第三个 VSync：从 buffer 取出一帧显示
因此，一个视图的属性变更如 frame 改变，需要经历 3个 VSync 信号。可以看出这样的模式需要App、GPU、显示器精确的配合，假如 App 提交不及时或者 GPU 遇到了复杂的绘制导致不能在16.67ms中完成，buffer 将不能及时被填充导致下一帧画面还是上一帧的内容，这样就造成了一帧的卡顿。

- 双缓冲、三缓冲的原理？
三缓冲：为 Render Server 提供一个额外的帧持续时间

- Hitch 从什么时候开始计算？
从 VSync_expected 发出某帧应该展示时，到最终实际展示 的 VSync_show
- 主 Runloop 与 VSync 之间的关系？
主线程 Runloop 接收 VSync 信号唤醒进行视图状态变更收集，因此 Runloop 执行循环并不是与 VSync 完全一一对应，即假如主 Runloop 一直忙碌中在执行任务，则 Runloop 期间接收到的唤醒信号并没有实质上有对应的提交。

- 假如前一个计算了一帧缓存，接下来的 VSync 又改变了，这个时候 GPU 怎么处理缓存？


# 卡顿发生的阶段
由上面介绍我们知道 iOS 应用的 UI 展示是交给一个叫做 Render Server 的进程去完成的，这里面包含了上述的几个步骤：
App 打包视图树信息给 Render Server，即 Commit Phase
Render Server 计算纹理信息交给 GPU 处理，即 Render Phase

**Commit Hitch** ，发生在 App 内，代表 App 花费太多时间调或者处理事件，包含了4个主要阶段：
- Layout
- Display
- Prepare
- Commit,  

**Render Hitch**，发生在 Render Server，代表没有及时完成一次渲染

# 卡顿类型

有哪些类型的卡顿
发生在哪些阶段

# 卡顿测量
卡顿比例
what's new in MeticKit
Eliminate Animation Hitches with XCTest
Cirtical >= 10ms/s 红 严重影响
Warning 5- 10 ms/s 黄 用户会察觉到一些中断
Good <= 5 ms/s 绿 不容易被觉察

# Why
# 如何使用

# Commit Phase 优化
commit transcation
commit phase 包含4个阶段：layout、display、prepare、commit
layout：
layoutsubviews
dislay:ovierride\setneedsdisplay
image decoding、conversion (Ref iamges and graphics best priaces)
commit:package

# find hitches
animation hitches
# Recommendations
keep views lightwieght
hidden
不使用drawRect

rely on setneedslayout notlaytoutif need
最好的约束
递归布局安规

# Render Phase

Frame lifetimes 表示这一帧持续了多久
Render - 选中 - Render Count 代表需要创建离谱渲染通道的数量 
优先关注
Pre-Commit latency 出现这个问题的原因是commit 阶段被延迟了
    view 初始化比较重
Expensive commit
    commit 四个小阶段耗时过长
    主线程上有耗时操作，如数据库读写、播放器的一些操作

    Demystify and eliminate hitches in the render phase
    Render Loop
    hitche: longer than expected
    Render Phase: 
* prepare 
  break down animations to render
  break down layers and effects init a step by step
* excute
  draw each step using GPU
  compiles the final image for display
  Example
  offscreen pass
  entire information
  ## finding hitches
  Hitch type
  expensive GPU time
  render countL offsreen times

  ShadowPath 代替 cornerRidus 原因？

maskTObOUNDS


# 实战
## TTK 冷启卡顿很久是什么原因？
## In-App Push 卡顿分析？
