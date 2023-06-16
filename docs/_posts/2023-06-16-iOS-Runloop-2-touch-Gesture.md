---
layout: post
title:  "iOS Runloop(2) - 触摸与手势事件"
date:   2023-06-16 09:58:51 +0800
categories: iOS Foundation
---

触摸、按压等用户事件接收处理依靠的是两个 Runloop 配合完成对事件的响应。

# 1. 整体流程
- 硬件设备 -> 主屏幕程序(SpringBoard) -> 当前应用程序 ->     
- 唤醒 com.apple.uikit.eventfetch-thread 中 Runloop 的 Source1 - __IOHIDEventSystemClientQueueCallback 处理事件数据封装 IOHIDEvent ->
- 标记对应主线程中 Source0 事件 -> 手动唤醒主线程 Runloop ->     
- __eventFetcherSourceCallback 处理事件队列 -> hitTest -> 收集到关联手势、UIResponder -> 
- 按照优先级: 手势 > UIControl > UIResponder 响应对应用户回调，或者一直回传给 UIApplication 对象进而丢弃。

## 1.1子线程接收系统的触摸事件
![image](/assets/imgs/runloop_touch_eventfetch.png)
com.apple.uikit.eventfetch-thread 子线程专门负责接收系统(SpringBoard.app)发给应用内的事件通知,内部就是一个 Source1(基于端口)，当 SpringBoard 将一个如触摸事件发送给当前应用，com.apple.uikit.eventfetch-thread 子线程内部的 Runloop 被唤醒，并通过 `__IOHIDEventSystemClientQueueCallback` 函数响应，
将事件数据封装成 IOHIDEvent(IOHID 即 IO Human Interface Device 的缩写)，并通过唤醒主线程 Runloop (`NSRunloopRunInMode`)去执行对应 Source0 回调(`__eventFetcherSourceCallback`)

## 1.2 唤醒主线程 Runloop
![image](/assets/imgs/runloop_touch_source0_iohidevent.png)
上面提到子线程在接收到 SpringBoard.app 发送过来的消息后，将其封装为 IOHIDEvent 后，标记对应的 Source0 为待处理状态？，并且手动唤醒主线程 Runloop？，进而处理目标 Source0 的回调函数 `__eventFetcherSourceCallback`，该回调函数会将 IOHIDEvent 转换成 UIEvent 事件，同时交给当前 UIApplication 的 window 处理

## 1.3 处理 UIEvent
### 寻找最佳响应 UIResponder
众所周知，hitTest 是 UIResponder 中的方法，用于寻找能够响应当前触摸、按压等事件的最佳响应视图 UIView，于此同时也会收集视图关联的手势识别器(UIGestureRecognizer)？，
那么 __eventFetcherSourceCallback 在收到事件后，将会将事件放入事件队列中，事件队列在调度期间会去寻找目标响应对象。
![image](/assets/imgs/runloop_touches_hittest.png)

默认情况下，hitTest: 方法会进一步调用 pointInside 判断当前触摸位置是否在自身区域范围内，假如是则继续在子视图中寻找，直到找到一个最佳目标响应视图，否则 hitTest: 方法返回空代表当前 UIView 及子视图不是响应目标。

在上述过程中同时还会收集链路上的手势识别器，待目标视图、关联手势识别器都搜寻完毕后，下一步将对 UIEvent 事件进行发送

### 发送 UIEvent 事件
将 UIEvent 分别按照优先级顺序发送给 UIControl、UIGesgtureRecognizer、UIResponder，默认情况下他们对应的 touchesBeginXX：、touchesMovedXX: 等方法将被调用，
__dispatchPreprocessedEventFromEventQueue 在上面步骤 UIEventHIDEnumerateChildren 寻找目标响应者后，会调用 `-[UIApplication sendEvent:]` 方法，给目标响应对象发送事件，接下来手势的 touchesBegan(_:with:) 先响应，接下来才是 UIView 的
![image](/assets/imgs/runloop_gesture_touches.png)
![image](/assets/imgs/runloop_respond_touches.png)

假如前面的关联对象如手势识别器识别成功，那么先响应手势的响应函数(通过[UIGestureEnvironment _deliverEvent:toGestureRecognizers:usingBlock:] 触发)，再 cancel UIResponders 后的 UIResponder(通过 -[UIGestureEnvironment _cancelTouches:event:]，
![image](/assets/imgs/runloop_cancel_touch.png)

可以看出 touchesCancelled(_:with:) 方法在这种情况下是由手势管理器触发），
这样后续的触摸事件回调将不会被触发，而是被 canceld。

### Responder 事件回调
UIAppliication 将事件发送给目标响应视图，对应的触摸系列方法将会被调用
``` Objective-c
@interface UIResponder : NSObject <UIResponderStandardEditActions>
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
@end
```

> UIResponder 如何不响应事件？
> 我们经常说一个 UIResponder 或者 UIView 不响应一个触摸事件，是指不额外做一些处理，并不是不实现上述方法，或者指的userInteractionEnabled = false, 而是本身采用默认实现，将事件往响应链路上抛，问问有没有谁需要接收这个事件 UIEvent。

### 手势识别

按照优先级，在发送事件给 UIResponder 之前，UIApplication 会先将 UIEvent 发送给寻找目标视图过程中收集的手势识别器，手势识别器虽然没有继承 UIResponder，但是也有对应的 touch系列方法，可以看到确实是早于 UIView 接收到 UIEvent，手势识别器内部有属性代表识别状态，当状态变成 XXX 时，代表手势识别成功，那么系统设定手势的优先级大于普通 UIResponder 基础的事件响应，将取消后续对 UIResponder 发送 UIEvent，还是发送 cancel 事件。假如识别器识别失败，则继续往 UIResponder 发送 UIEvent

代码验证
> UIGestureRecognizer objects are not in the responder chain, yet observe touches hit-tested to their view and their view's subviews. After observation, the delivery of touch objects to the attached view, or their disposition otherwise, is affected by the cancelsTouchesInView, delaysTouchesBegan, and delaysTouchesEnded properties.

即手势识别器不在响应链中，然而却会观测碰撞到他们关联的视图或者子视图，诉后触摸对象在传递给目标视图时将受cancelsTouchesInView、delaysTouchesBegan、delaysTouchesEnded 等属性设置的影响。

```objective-c
@interface UIGestureRecognizer (UIGestureRecognizerProtected)
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
@end

```

### UIControl 事件回调与响应
UIControl 是 UIView 的子类，因此也能收到 touchXXX 系列方法，通常我们通过 addTarget:action: 方法在 UIControl 添加响应事件回调。

## 1.4 UIControl、UIGestureRecognizer、UIView 优先级

三者的优先级关系如下：    
1. 同视图对象上 UIGestureRecognizer > UIControl >    
2. 父视图对象上 UIGestureRecognizer > UIControl >    
3. ...    

#### 事件传递顺序
> UIGestureRecognizerA touchBegin > UIResponderB touchBegin > UIControlB touchBegin > UIControlB trackingBegin >    
> 父视图的 UIGestureRecognizerC touchBegin > 父视图 UIControlD touchBegin >    
> UIGestureRecognizerA touchEnd > UIGestureRecognizerA Action >    
> UIResponderB touchEnd > UIControlB touchEnd > UIControlB trackingEnd - UIControlB Action    

原则猜测：当前对象(UIGestureRecognizer、UIControl、UIResponder)处理完 touchEnd事件后，则判断当前对象是否可以响应该事件(如当前)，如果是则开始响应；否则才会继续走后续的流程(按照响应链去传递 UIEvent)

#### 同视图 UIGestureRecognizer > UIControl

```
Gesture - touchEnd:
frame #1: 0x00000001028762f4 MyLord`@objc MyGestureRecoginzer.touchesEnded(_:with:) at <compiler-generated>:0
frame #2: 0x0000000109b5f070 UIKitCore`-[UIGestureRecognizer _componentsEnded:withEvent:] + 152
frame #3: 0x000000010a02b648 UIKitCore`-[UITouchesEvent _sendEventToGestureRecognizer:] + 556 // Diff
frame #4: 0x0000000109b54300 UIKitCore`-[UIGestureEnvironment _deliverEvent:toGestureRecognizers:usingBlock:] + 160
frame #5: 0x0000000109b540f0 UIKitCore`-[UIGestureEnvironment _updateForEvent:window:] + 156
frame #6: 0x0000000109fe7f80 UIKitCore`-[UIWindow sendEvent:] + 3168
frame #7: 0x0000000109fc6224 UIKitCore`-[UIApplication sendEvent:] + 692
frame #8: 0x000000010a042b10 UIKitCore`__dispatchPreprocessedEventFromEventQueue + 6824
frame #9: 0x000000010a044774 UIKitCore`__processEventQueue + 5612
frame #10: 0x000000010a03d650 UIKitCore`__eventFetcherSourceCallback + 220
frame #11: 0x00000001803731dc CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 24
```

```
// Gesture - Action
frame #1: 0x0000000102878340 MyLord`@objc PerformanceViewController.largeTimeMethod() at <compiler-generated>:0
frame #2: 0x0000000109b5c790 UIKitCore`-[UIGestureRecognizerTarget _sendActionWithGestureRecognizer:] + 76
frame #3: 0x0000000109b64178 UIKitCore`_UIGestureRecognizerSendTargetActions + 88
frame #4: 0x0000000109b6194c UIKitCore`_UIGestureRecognizerSendActions + 300
frame #5: 0x0000000109b6107c UIKitCore`-[UIGestureRecognizer _updateGestureForActiveEvents] + 496
frame #6: 0x0000000109b55044 UIKitCore`_UIGestureEnvironmentUpdate + 2544 // 这里继续调用响应回调
frame #7: 0x0000000109b54374 UIKitCore`-[UIGestureEnvironment _deliverEvent:toGestureRecognizers:usingBlock:] + 276
frame #8: 0x0000000109b540f0 UIKitCore`-[UIGestureEnvironment _updateForEvent:window:] + 156
frame #9: 0x0000000109fe7f80 UIKitCore`-[UIWindow sendEvent:] + 3168
frame #10: 0x0000000109fc6224 UIKitCore`-[UIApplication sendEvent:] + 692
frame #11: 0x000000010a042b10 UIKitCore`__dispatchPreprocessedEventFromEventQueue + 6824
frame #12: 0x000000010a044774 UIKitCore`__processEventQueue + 5612
frame #13: 0x000000010a03d650 UIKitCore`__eventFetcherSourceCallback + 220
frame #14: 0x00000001803731dc CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 24
```
#### UIControl > 父视图 UIGestureRecognizer

```
// targetAction - Selector
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
  * frame #0: 0x000000010089b914 MyLord`PerformanceViewController.largeTimeMethod1(self=0x0000000102a09890) at PerformanceViewController.swift:87:15
    frame #1: 0x000000010089b990 MyLord`@objc PerformanceViewController.largeTimeMethod1() at <compiler-generated>:0
    frame #2: 0x0000000107fed920 UIKitCore`-[UIApplication sendAction:to:from:forEvent:] + 96
    frame #3: 0x00000001079a79cc UIKitCore`-[UIControl sendAction:to:forEvent:] + 108
    frame #4: 0x00000001079a7d10 UIKitCore`-[UIControl _sendActionsForEvents:withEvent:] + 268
    frame #5: 0x00000001079a49a4 UIKitCore`-[UIButton _sendActionsForEvents:withEvent:] + 120
    frame #6: 0x00000001079a6a30 UIKitCore`-[UIControl touchesEnded:withEvent:] + 392
    frame #7: 0x00000001008997ec MyLord`MyView.touchesEnded(touches=Swift.Set<UIKit.UITouch> @ 0x000000016f57b668, event=0x000060000390c4e0, self=0x0000000102a30dc0) at PerformanceViewController.swift:25:15
    frame #8: 0x000000010089986c MyLord`@objc MyView.touchesEnded(_:with:) at <compiler-generated>:0
    frame #9: 0x0000000108022ae8 UIKitCore`-[UIWindow _sendTouchesForEvent:] + 900
    frame #10: 0x0000000108023f90 UIKitCore`-[UIWindow sendEvent:] + 3184
    frame #11: 0x0000000108002224 UIKitCore`-[UIApplication sendEvent:] + 692
    frame #12: 0x000000010807eb10 UIKitCore`__dispatchPreprocessedEventFromEventQueue + 6824
    frame #13: 0x0000000108080774 UIKitCore`__processEventQueue + 5612
    frame #14: 0x0000000108079650 UIKitCore`__eventFetcherSourceCallback + 220
    frame #15: 0x00000001803731dc CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 24
    frame #16: 0x0000000180373124 CoreFoundation`__CFRunLoopDoSource0 + 172
    frame #17: 0x0000000180372894 CoreFoundation`__CFRunLoopDoSources0 + 232
    frame #18: 0x000000018036cf00 CoreFoundation`__CFRunLoopRun + 756
    frame #19: 0x000000018036c7f4 CoreFoundation`CFRunLoopRunSpecific + 584
    frame #20: 0x0000000188faec98 GraphicsServices`GSEventRunModal + 160
    frame #21: 0x0000000107fe85d4 UIKitCore`-[UIApplication _run] + 868
    frame #22: 0x0000000107fec5cc UIKitCore`UIApplicationMain + 124
    ...
(lldb) 
```
可以看出无论是 UIGestureRecognizer 还是 UIControl，当执行完 touchEnd事件后，假如自己是最高优先级的响应对象，则立即执行对应回调


# 引用
官网：https://github.com/apple/swift-corelibs-foundation/  
iOS 博客：https://juejin.cn/user/1591748569076078    
Runloop 内部详解：http://www.xbhp.cn/news/160874.html    
手势详解：https://juejin.cn/post/6844904175415853064     
牛：https://www.jianshu.com/p/c294d1bd963d     
iOS 事件传递与处理：https://juejin.cn/post/6956756476559884301     
