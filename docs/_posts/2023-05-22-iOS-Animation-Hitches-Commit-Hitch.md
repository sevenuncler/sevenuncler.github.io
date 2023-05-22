---
layout: post
title:  "iOS Animation Hitches(2) - Commit Hitch 优化"
date:   2023-05-22 13:58:51 +0800
categories: iOS Perf
---

# Animation Hitches(2) - Commit Hitch 优化
## 序言
上一篇已经介绍了iOS设备卡顿的阶段主要来自：
1. App 中打包视图树的超时导致卡顿 (*Commit Hitch*)    
2. *The Render Server* 利用 (GPU) 渲染纹理超时导致的卡顿 (*Render Hitch*) 

那么这些卡顿阶段内部又包含了哪些内容，开发者又可以从哪些方面去分析、优化呢，本篇将从原理与实战中的详细介绍 *Commit Hitch* 类型卡顿的优化。
##  Commit Hitch(提交卡顿)
众所周知，在 iOS 系统中，Core Animation 负责 App 的视图与动画的呈现，其基本工作方式即：
- 在主线程 Runloop 中统一收集期间更新后的视图，然后进行重新布局、绘制自定义 `UIView`等
- 接着，将更新后的视图树通过 IPC 进程间通信，打包给 *The Render Server* 系统渲染进程，然后依赖 VSYNC 信号，进行视图、动画的渲染
- 最后 GPU 将渲染后的视图帧提供给显示器展示

而所有发生在主线程 Runloop 中关于视图更新以及打包的流程导致的卡顿，都称之为 "Commit Hitch"，而具体的 Commit 包含以下几个阶段：
1. **Layout**，即视图的布局更新(`layoutSubviews`)
2. **Display**, 对应 draw(rect:) 系列方法
3. **Prepare**, 提交前的准备
4. **commit**，事务提交

既然已经清楚了 Commit Hitch 包含了哪些视图更新阶段，那么针对每一个环节，开发者需要注意的点已经优化的方法有哪些呢？
## Layout(布局)
*Layout* 阶段对应的视图 `layoutSubviews` 系列方法, 
**回顾一下哪些原因会导致视图的触发 `layoutSubviews`？**
- 设置视图位置(frame、bounds、transform)，如屏幕方向变化了、更新了视图的约束、 `frame`，
- 视图层级的改变，即添加、删除视图
- 显示调用 setNeedsLayout

这些视图的变更会导致 Core Animation 在一个 Runloop 的 BeforeWaiting（即将进入休眠）阶段去重新布局更新的视图树。

> 建议：
> - 尽量复用视图避免高昂的添加/删除视图操作
> - 使用 hidden 代替移除视图
> - 减少代价高昂的布局，使用 setNeedsLayout 而不是 layoutIfNeeded
> - 使用最少得约束，降低解析约束布局时的消耗
> - 递归布局代价昂贵，视图只应该使自己会子视图布局失效，而不是父试图？ 

> BeforeWaiting 是不是意味着本次 Runloop 变更的视图，需要等下一个 Runloop 才会被提交到 Render Server 呢？
## Display(显示)
需要 Display 的场景有：
- 重写 `drawRect` 方法，在自定义视图需要更新内容时
- 显示调用 `setNeedsDisplay`
自定义视图绘制、重写CALayer 或 UIView 的 drawRect 系列方法等将在这个阶段被调用

> 因此尽量避免不必要的 drawRect 方法重写, 使用 *CALayer* 的 GPU 加速的属性代替自定义 `draw(rect:)` 的代码，避免使用 CPU (*CoreGraphics*)处理自定义绘图。
> 去除空的 `draw(rect:)` 方法
## prepare(提交前准备)
- 图像解码，如 PNG\JPEG 格式图片解压缩、解码成 bitmap 格式
- 图像转换，某些图像的颜色格式 GPU 无法直接处理，比如 YUV 等图像颜色格式？
在视图树被提交前，未解码的图像资源将在这个时候被解码成 bitmap，对于大图来说，这将花费很多时长，可以参考官方的 《Images and Grphics Best Practices》

当使用 UIImage 或 CGImageSource 的系列方法创建图片时，图片数据不会立刻被解码，而是在被本阶段被解码，这个过程发生在主线程，想要绕过这个机制，常见的做法是使用后台线程吧图片会知道 CGBitmapContext 中，然后直接创建图片，目标网络图片库自带这个功能。

由于 CoreGraphic 是线程安全的，因此图像的绘制可以放到子线程中进行

``` Objc
- (void)display {
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

## commit(提交)
最终视图树将被递归打包给 render server，层级很深的视图树打包时间就需要更久
## 示例
准备一个 Xcode 工程，Xcode - Product - Profile(运行Release配置) - 打开 Instruments，选择 Animation Hitches, 选择目标 App，左边按钮开始录制，App 内进行一些基本操作后停止，等待后展示出分析后的数据，可以看到，在 Animation Hitches 模版中有以下几个轨道：
- Hitch 显示了卡顿及其持续的时长
    ![images](/assets/imgs/commit_hitch_hitch_tracker.png)
- User Events 表示触发卡顿的用户时间
    ![images](/assets/imgs/commit_hitch_user_events_tracker.png)
- Commits 显示了提交事件以及进程
    ![images](/assets/imgs/commit_hitch_commits_tracker.png)
- Renders\GPU， 代表渲染器与 GPU 轨迹
- Frame life times， 构成卡顿的全过程
    ![images](/assets/imgs/commit_hitch_frame_lifetimes_tracker.png)
- Hitch Types，可以推测出帧在哪个阶段被延迟了

1. 在底部 Hitch 找到一个 Hitch
    ![images](/assets/imgs/commit_hitch_select_hitch_record.png)
    选择第一个卡顿记录，从记录中可以得到：
    - Hitch Begin：卡顿开始的时刻
    - Presntation: 所对应的帧呈现的时刻
    - Hitch Duration: 持续时长
    - Buffer Count: 此时是几个缓冲策略，系统会自动切换双缓存或是三缓冲
    - Acceptable latency: 可接受的延时 16.67ms * 3, 即对应上面的三缓冲
    - Severity: 严重程度
    - Hitch Type：即上面提到的卡顿类型
    
可以看出卡顿的延时是按照：对应帧实际展示的时刻 - 预期呈现的时刻？卡顿的类型是 *Expensive Commit*,即 Commit Phase 阶段的卡顿，确实根据上图也可以看出最近一个 Commit Phase 花费了 34.61ms，超出了 2 * SYNC，导致了卡顿。
2. 继续分析，选择目标 App 的主线程，通过 Time Profiler 查看耗时函数
    ![images](/assets/imgs/commit_hitch_time_profile.png)
可以看到方法比较耗时
    ![images](/assets/imgs/commit_hitch_problem_code.png)
从截图可以看出 201 行函数更新子cell比较耗时，因此进行优化代码后，继续分析，直到消灭所有严重的卡顿(黄色、红色)。
## 总结分析方法
- 通过 Animation Hitches 找到卡顿记录
- 查看卡顿类型(按严重程度黄色、红色为需要重点关注)
- 确定是 Commit Phase 卡顿还是 Render Phase 卡顿(决定了后续的代码优化方向)
- 选中此时的 App 主线程，切到 Time Profiler 找到耗时的函数
- 优化代码(根据上面的优化建议)
- 重新分析，重复制导修复所有卡顿




## 参考
官方：https://developer.apple.com/videos/play/tech-talks/10856
Time Profiler 使用：《Using Time Profiler in instruments》
保持界面流畅的技巧：https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/
字节：https://juejin.cn/post/7231731488928399415?from=bytetech