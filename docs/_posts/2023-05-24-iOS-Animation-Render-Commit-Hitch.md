---
layout: post
title:  "iOS Animation Hitches - Render Hitch 优化(三)"
date:   2023-05-22 13:58:51 +0800
categories: iOS Perf
---
# 序
前一篇文章《Animation Hitches - 概述》已经介绍了 UI 发生在 APP 期间的 CPU 卡顿原理与优化措施，即在 *Commit Phase* 的 Layout、Display、Prepare、Commit 四个环节进行分析与优化。那么对于 *Render Server* 与 GPU ，又有哪些原因导致卡顿，并且如何优化的这些引起卡顿的问题呢？

# Render hitch
顾名思义，*Render Hitch* 即渲染期间的卡顿，iOS 设备的渲染流程是由 *The Render Server* 配合 GPU 完成，而前面的文章《Animation Hitches - 概述》中介绍过一个画面帧在被提交的 *The Render Server* 之后，有 16.67ms (默认情况)可处理的时间而不引起卡顿，假如 Render Server 对这一帧的处理耗时超过了 16.67 ms，那将导致 GPU 缓冲区没法填充，导致显示屏幕上还是上一帧的画面。接下来，我们来看一下 Render Server 内部的流程。

# Render Prepare 阶段
> Breaks Down animations from commit to be rendered over time   
> Breaks down layers and effects into a step-by-step plan of simple operations   

当视图树打包提交给 Render Server， Render Server 需要拆解出其中的动画，将图层数和特效拆分成一个一个简单的绘制操作。

![images](/assets/imgs/render_hitch_prepare.png)  

# Render Excute 阶段
> Draw each step in the pipeline using the GPU  
Compline the final image for display  

Metal/OpenGL ES 按照上面命令执行绘制，具体方法是按照视图层级从底往上的顺序绘制和叠加，先父视图 -> 子视图 -> 兄弟视图的顺序：

![images](/assets/imgs/render_hitch_excute.png)


当遇到比如阴影时，阴影依赖于目标图层的形状，因此虽然一个图层的阴影特效是在该图层之下需要先绘制，但是阴影却需要依赖该图层的形状等信息，无法直接绘制，这个时候，就需要另外开辟一个新的纹理，先绘制目标图层，确定所需信息，在此基础上绘制阴影，然后再把阴影拷贝到原始目标纹理上，再接下来继续阴影所在图层的绘制，这个过程即所谓的离屏渲染。  
![images](/assets/imgs/render_hitch_offscreen.png)

大量的离屏渲染将导致 GPU 性能开销的增加与大量的耗时，进而导致 GPU 无法按时绘制预期的画面，导致卡顿，有四种主要的离屏渲染可以优化：
- Shadowing
- Masking
- Rounded Rectangles
- Visual Effects

> 对应导致离屏渲染的原理以及解决方案是啥？

# 发现与优化
可以看到 Render Hitch 的一个主要来源来自于大量的离屏渲染，那么接下来我们来看看如何找到离谱渲染的图层以及对应的优化方案，使用 Instruments 的 Animation Hitches 模版开始对目标程序分析，进行指定的操作后，停止记录，开始分析数据：
- 选中 Hitch 轨道，在底部选择一个 Hitch 记录,

![images](assets/imgs/render_hitch_select_record.png) 

选中一条 Render Phase 的卡顿记录，如何区分是 Commit Phase 还是 Render Phase，可以根据 Hitch type 以及阶段上方所处是轨道看对应的帧是在哪个阶段被延迟提交了。

- 切换轨道置 Renders，再次看到底部，需要重点关注 Render count，即离屏渲染的次数  
![images](assets/imgs/render_hitch_gpu_render_count.png)

- 切换到 Xcode，Editor - select layer, 可以看到该图层离屏渲染的内容
![images](assets/imgs/render_hitch_layer_off_screen.png)

- 根据记录找到对应的代码，进行离屏渲染排查，代码优化  
Shadowing，不使用shadow 属性，而使用 shadowPath 告知 Render 阴影的形状，这样就避免了渲染阴影时不知道具体形状而需要离谱先绘制图层再绘制阴影的离屏渲染问题    
Masking
Rounded Rectangles
Visual Effects
- 重复上述步骤

# 优化建议


# TODO
双缓冲、三缓冲切换？
渲染被延迟并且 Render Server 视图追赶时会切成3

# 参考

官方: https://developer.apple.com/videos/play/tech-talks/10857
