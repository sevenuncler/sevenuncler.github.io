---
layout: post
title:  "iOS Instruments - Animation Hitches 概述"
date:   2023-05-20 10:48:51 +0800
categories: iOS Perf
---

# Hitch (卡顿) 
> A hitch is any time a frame appears on screen later than expected

Hitch 即卡顿，代表任何时候某一帧画面出现晚了，卡顿的出现会给用户带来不好的用户体验，对产品的印象产生负面影响。
## 卡顿示意图
![image](/assets/imgs/animation_hitch_show_case.png)
VSync 表示每一帧初始触发时间的硬件, VSYNC 表示新帧必须准备就绪的时刻
iOS 设备画面显示的原理依赖 VSync 硬件时钟，它的工作原理，按照刷新帧率如 60hz，每隔16.67ms秒发出一次信号，协调 App、GPU、显示器进行渲染流程。
刷新帧率设备列表与截图
## Apple 设备刷新率

| 设备型号 | 刷新帧率 | VSync 间隔 |年份|
| --- | --- | --- |---|
| <= iPhone 13 Pro| 60HZ |  16.67 ms|
| iPad Pro | 120 HZ | 8.33ms | 2017. 2代 |
| >= iPhone 13 Pro 系列 | 120HZ(ProMotion) |8.33 ms| 2021 |

# 渲染循环(The Render Loop)
![image](/assets/imgs/animation_hitch_render_loop.png)
Render Loop 包含5个阶段：
- **App 内**  
Event: 应用内事件，如触摸事件、网络回调、定时器等，App 在响应这些事件时可以变更视图的比如布局、背景色等。    
Commit：统一收集在本次 Runloop 期间变更后的视图树，打包给 Render Server。  
Event 和 Commit 这两个阶段的处理都在应用内，这两个阶段是串行的，需要在1个VSYNC完成，否则会引起卡顿。  
- **Render Server**  
Render Prepare，为在 GPU 上绘制做准备，准备线性管线。  
Render Excute，渲染管线经过 GPU，GPU 绘制出图像。  
这两个阶段都在 Render Server 进程中完成，需要在1个 VSYNC 完成，否则会引起卡顿。
- **Diplay**  
将 GPU 绘制的图像显示出来。

对应到 VSYNC：  
第一个 VSync: App 打包视图树信息给 Render Server  
第二个 VSync：Render Server 计算出需要绘制纹理数据给 GPU，GPU 绘制放入 buffer 缓存  
第三个 VSync：从 buffer 取出一帧显示   

因此，一个视图的属性变更如 frame 改变，需要经历至少 2个 VSync 信号。可以看出这样的模式需要App、GPU、显示器精确的配合，假如 App 提交不及时或者 GPU 遇到了复杂的绘制导致不能在16.67ms中完成，buffer 将不能及时被填充导致下一帧画面还是上一帧的内容，这样就造成了一帧的卡顿。  

# 卡顿类型
由上面介绍我们知道 iOS 应用的 UI 展示是交给一个叫做 Render Server 的进程去完成的，这里面包含了上述的几个步骤：  
App 打包视图树信息给 Render Server，即 Commit Phase，这个阶段发生的卡顿叫 Commit Hitch。  
Render Server 计算纹理信息交给 GPU 处理，即 Render Phase，这个阶段发生的卡顿叫 Render Hitch。  

**Commit Hitch** ，发生在 App 内，代表 App 花费太多时间调或者处理事件，包含了4个主要阶段：
- Layout，layoutSubviews 等  
- Display，自定义绘制，通常与 drawRect 方法相关  
- Prepare，图片在提交前解压等  
- Commit,  App 打包视图树到 Render Server  

![image](/assets/imgs/animation_hitch_duration.png)
上图展示了卡顿耗时(Hitch time)的起始计算方式，以显示器预期显示某帧开始，到实际展示的时刻结束。

**Render Hitch**，发生在 Render Server，代表没有及时完成一次渲染

![image](/assets/imgs/animation_hitch_render_hitch.png)


# 如何衡量卡顿
通常使用卡顿率来衡量卡顿表现，即 hitch time / duration  
**Cirtical** >= 10ms/s，红 严重影响, 代表开发过程中需要立即排查解决  
**Warning** 5- 10 ms/s， 黄 用户会察觉到一些中断，需要关注这里的卡顿问题    
**Good** <= 5 ms/s， 绿 不容易被觉察  

# 自问自答
1. 120HZ 的设备更容易引起卡顿吗？  
不会，举例，假如一次 commit 的耗时为 12ms, render 耗时为8ms，则在 60hz 上需要经过2 * VSYNC = 32ms，而 120HZ 设备上需要 2 * VSYNC + 1 * VSYNC = 24ms，在高刷设备上更早的展示了图像帧，表现出来更加流畅。   
2. 主线程 Runloop 每次运行完成假如有变更都会进行一次 Commit 吗？  
3. 双缓冲、三缓冲的原理？   
双缓冲：即一帧画面在展示前需要经过两个帧时长处理
三缓冲：为了避免卡顿，为 Render Server 提供一个额外的帧持续时间，系统可能会切换到三缓冲
4. Hitch 从什么时候开始计算？  
从 VSync_expected 发出某帧应该展示时，到最终实际展示 的 VSync_show
5. 主 Runloop 与 VSync 之间的关系？  
主线程 Runloop 接收 VSync 信号唤醒进行视图状态变更收集，因此 Runloop 执行循环并不是与 VSync 完全一一对应，即假如主 Runloop 一直忙碌中在执行任务，则 Runloop 期间接收到的唤醒信号并没有实质上有对应的提交。  
6. 假如前一个计算了一帧缓存，接下来的 VSync 又改变了，这个时候 GPU 怎么处理缓存？
猜测也是和 Core Animation一样统一收集变更而不是一有变更就执行渲染。