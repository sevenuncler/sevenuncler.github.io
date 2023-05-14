# TODO
如何拆分 Runloop 查看耗时

1.  图片后台绘制进 CGBitmapContext
VSync 垂直同步

HSync 水平同步的目的？
https://github.com/skyming/iOS-Performance-Optimization
<https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/>
Time profiler WWDC 解读 https://juejin.cn/post/6844903853880524814
Xcode Instruments API 学习
# 发现问题
1. Time Profiler
2. Animation Hitches
3. FPS 监控
- 使用 CADisplayLink 检测 FPS，如 低于30时 dump 堆栈，如何反解堆栈？
4. 线上监控
- 防止子线程 UI 操作，Hook UIView/CALayer 的 setNeedsLayout等方法
- 监测主线程 Runloop 的单次执行耗时，假如超过了 3s 就会捕获所有线程堆栈


# 分析问题

# Time Profiler
## 什么是 Time Profiler
> app spending time
> main thread responsiveness issues
> profile difficult workload

用于检测应用内各方法的耗时占比的性能分析工具，利用它可以找到最耗时的方法进行优化有着后续分析。

## 为什么需要 Time Profiler
- 检测发现主线程响应慢的问题，提升用户体验
- 检测/分析 App 内耗时的方法，不合理的调用，进而优化/降低 CPU 使用，降低能耗、发热 
- 可以精确的分析方法的耗时

### 与 System Trace 的区别

## 如何使用
1. 准备工程 
2. Xcode - Product - Profile - Time Profiler
![image] /assets/imgs/timeprofiler_select_app.png
> profile releases build
1. 选择目标 App，左边按钮开启检测
  - Time Profiler 轨道
  - Points of Interest 轨道，用于精准的分析函数耗时
  - Thermal State
  - 
2. App 内操作
3. 查找 CPU 耗时的位置
4. 找到最耗时的调用函数
5. 计算精确耗时

添加额外的模版
Window mode
Top: CPU 占用

开始使用
选择追踪的进程(即目标 App)
放大
CPU 占用
找到大 CPU
+ 号pin track
"Spinning" 在instruments 中代表卡顿
主线程用来接收用户谁让、更新用户界面
查看主线程
选中时间区域
optional + 前面三角形展开全部调用树
最厚树，白色代表自己的代码
"thunk" 编译器生成的帮助代码，与你App 中代码没有关系
双击进入具体代码
打开 Xcode，修改
总结

模拟器其实是你的Mac，用真实硬件
Hight CPU: 电量、温度、风扇

使用Signposts
Time Profiler 不精确
如何精确代码
signposts
insttuments 支持
最厚堆栈
Point of interests 
list: regions of interest
duration 每个事件的耗时
找到root caller
measure api ?
三击



Time profiler 每隔1ms采样一次，假如在采样的时候目标函数在调用中则weight + 1ms
weight: 该调用栈被采用到的时间比例
self weight: 该调用方法本身的耗时
charge xxx: 将耗时计算到调用函数中
prune "symbol" and subtree: 移除该符号以及调用子树
flatten library to boundaries: 区分目标与个人代码的边界
## 原理向
## 实战
用于分析大耗时的函数，通常是 CPU 的耗时瓶颈
基本流程
每个指标栏详细作用
勾选 revert 之后怎么weight 排序不对了


## More Information
Creating Custom Instruments
Practical Approaches to Great App Performance
https://www.bilibili.com/video/BV15J411K7h8/?spm_id_from=333.337.search-card.all.click&vd_source=8f271a629cebbd20f105df6e6fae8a88

# System Trace

# 原理
一次渲染包含 CPU 与 GPU

CPU：视图创建、布局、图片解码、文本绘制？

优化思路

*   对象创建、更改、销毁

轻量对象、StoryBoard、推迟对象创建、重任务分散

修改 UIView\CALayer 属性消耗高，需要尽量避免

将对象销毁放到子线程

*   布局计算

    *   子线程提前计算好视图布局
    *   componentKit、Texture
*   图片解码

    *   UIImage、CGImageSource 创建时并不是立即解码，而是在CALayer 被提交 GPU 前解码，才会解码

        *   子线程将图片绘制到 CGBitmapContext，然后从 Bitmap 直接创建图片
*   文本绘制？

    *   大量文本内容时，文本高度计算耗时大

        *   boundingRectWithSize：计算文本宽高
        *   NSAttributedString drawWithRect: 绘制文本
    *   文本渲染

        *   UILabel、UITextView 排版与绘制均在主线程

            *   自定义文本控件，底层用TextKit、CoreText 异步绘制
        *   图片绘制

            *   CoreGraphics 线程安全，放到后台

&#x20;GPU：变换、合成、渲染

*   尽量减少短时间内大量图片显示
*   减少视图数量与层级
*   离屏渲染



