---
layout: post
title:  "iOS Runloop(3) - NSTimer"
date:   2023-06-22 14:41:51 +0800
categories: iOS Foundation
---
# NSTimer 原理
定时器基于 source1 (基于端口) 输入事件，依赖 Runloop 运行的一种定时执行任务的功能对象。
## NSTimer 的基本使用
### 创建、调度(schedule)

```Swift
// scheduledXXX 系列方法会创建定时器的同时自动将 Timer 对象添加到当前线程所在的 Runloop 中，runloop 会强引用 Timer 对象
// 不同的方法提供了不同的回调执行方式(selector、block、invocation)
open class Timer : NSObject {

    open class func scheduledTimer(withTimeInterval interval: TimeInterval, repeats: Bool, block: @escaping @Sendable (Timer) -> Void) -> Timer
    
    open class func scheduledTimer(timeInterval ti: TimeInterval, invocation: NSInvocation, repeats yesOrNo: Bool) -> Timer
    
    open class func scheduledTimer(timeInterval ti: TimeInterval, target aTarget: Any, selector aSelector: Selector, userInfo: Any?, repeats yesOrNo: Bool) -> Timer
}

```

```Swift
// init 系列方法需要在创建 Timer 对象后手动的添加到目标 Runloop 中
open class Timer : NSObject {

    public /*not inherited*/ init(timeInterval ti: TimeInterval, target aTarget: Any, selector aSelector: Selector, userInfo: Any?, repeats yesOrNo: Bool)
    
    public /*not inherited*/ init(timeInterval interval: TimeInterval, repeats: Bool, block: @escaping @Sendable (Timer) -> Void)
    
    public convenience init(fire date: Date, interval: TimeInterval, repeats: Bool, block: @escaping @Sendable (Timer) -> Void)
    
    public init(fireAt date: Date, interval ti: TimeInterval, target t: Any, selector s: Selector, userInfo ui: Any?, repeats rep: Bool)
}

```

注：目标 Runloop 需要运行起来，Timer 才会生效，否则 Timer 的回调不会被执行。


### 手动触发
>You can use this method to fire a repeating timer without interrupting its regular firing schedule. If the timer is non-repeating, it is automatically invalidated after firing, even if its scheduled fire date has not arrived.

对于重复执行的定时器，可以手动调用 `func fire()` 触发一次回调而不打断原来正常的定时逻辑(类似定时器被延时执行的效果，可看下面的分析)；对于非重复执行的定时器将在该方法后自动使定时器失效，即时可能触发时间点还没到达。
### 从 Runloop 移除定时器
`func invalidate() `是唯一可以将 Timer 从 Runloop 移除的方法，同时也会解除 Timer 初始化时对 target、userInfo 对象的强引用。
> You must send this message from the thread on which the timer was installed. If you send this message from another thread, the input source associated with the timer may not be removed from its run loop, which could prevent the thread from exiting properly.

注意该方法必须在与 timer 所在runloop 的线程上调用，否则将导致 timer 对应的输入事件源(source1)不能正常的从 runloop 被移除，这将进而导致线程无法正常退出。

### 不补偿错过的多次触发
 `var tolerance `该属性用于指定定时器允许执行的时间误差，默认为0。利用该属性，系统可以灵活的优化电量消耗与即时响应性能。
实际执行时间 = 计划执行时间 + x (x <= tolerance)，因此定时器不会提前与计划时间执行。

通常建议：Time tolerance >= interval * 10%, 即时设置很小的数值也可以可以显著降低耗电量。
>The system reserves the right to apply a small amount of tolerance to certain timers regardless of the value of this property.

即系统在某些情况下会忽略该属性。
### var fireDate
用户指定定时器执行的时间，该方法线程不安全(所有 NSTimer 的方法线程不安全)。

## Timer 原理
在添加定时器(`CFRunLoopAddTimer`)、执行完定时器回调后(`__CFRunLoopDoTimer`)、移除定时器后(`CFRunLoopRemoveTimer`)、手动设置触发定时器时间(`CFRunLoopTimerSetNextFireDate`)，都会重新设置定时器下一次触发的时间 `__CFArmNextTimerInMode`，而 __CFArmNextTimerInMode 的逻辑也比较简单，找到当前运行 Mode 下所有定时器中最早需要执行的定时器，通过 DISPATCH_SOURCE_TYPE_TIMER 或者 mk_timer_arm 设定事件源，定时触发, 进而执行相关定时器回调

### 添加定时器
```Swift
override func viewDidLoad() {
        super.viewDidLoad()
        self.view.backgroundColor = .white
        Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { timer in
            NSLog("timer\n");
        }
    }
```
上面添加定时器的代码
```C
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 4.1
    frame #0: 0x000000018036f118 CoreFoundation`CFRunLoopAddTimer
    frame #1: 0x0000000180bf6130 Foundation`+[NSTimer(NSTimer) scheduledTimerWithTimeInterval:repeats:block:] + 112
  * frame #2: 0x0000000100715e40 MyLord`ProfileViewController.viewDidLoad(self=0x000000013ed06c00) at ProfileViewController.swift:14:15
```


```C
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {
    // 1. 如果定时器指定的模式 是 Common，则对每个 Mode 添加该 Timer
if (modeName == kCFRunLoopCommonModes) {
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        if (NULL == rl->_commonModeItems) {
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        CFSetAddValue(rl->_commonModeItems, rlt);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlt};
            /* 添加新的 Timer 到所有的支持 Common的 Mode 中，是个递归调用，对于Timer 还是会走到下面 Else 逻辑 */
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
    } else { // 添加到目标Mode，并设置下一次触发时间
        __CFRepositionTimerInMode(rlm, rlt, false);
    }
}

```

### 设置下一个定时器触发时间
```C
static void __CFArmNextTimerInMode(CFRunLoopModeRef rlm, CFRunLoopRef rl) {    
    if (rlm->_timers) {
        // 1. 遍历该 Runloop Mode 下的所有定时器，找到下一个最早需要执行的定时器，并且计算出该定时器最早执行时间、最晚执行时间(最早+tolerance)       
        for (CFIndex idx = 0, cnt = CFArrayGetCount(rlm->_timers); idx < cnt; idx++) {
            CFRunLoopTimerRef t = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(rlm->_timers , idx);
            // discount timers currently firing
            if (__CFRunLoopTimerIsFiring(t)) continue;
            
            uint64_t oneTimerHardDeadline;
            uint64_t oneTimerSoftDeadline = t->_fireTSR;
            if (os_add_overflow(t->_fireTSR, __CFTimeIntervalToTSR(t->_tolerance), &oneTimerHardDeadline)) {
                oneTimerHardDeadline = UINT64_MAX;
            }
            
            // We can stop searching if the soft deadline for this timer exceeds the current hard deadline. Otherwise, later timers with lower tolerance could still have earlier hard deadlines.
            if (oneTimerSoftDeadline > nextHardDeadline) {
                break;
            }
            
            if (oneTimerSoftDeadline < nextSoftDeadline) {
                nextSoftDeadline = oneTimerSoftDeadline;
            }
            
            if (oneTimerHardDeadline < nextHardDeadline) {
                nextHardDeadline = oneTimerHardDeadline;
            }
        }
        
        // 2. 更新下一次将要触发的时间
        if (nextSoftDeadline < UINT64_MAX && (nextHardDeadline != rlm->_timerHardDeadline || nextSoftDeadline != rlm->_timerSoftDeadline)) {
            uint64_t leeway = __CFTSRToNanoseconds(nextHardDeadline - nextSoftDeadline);
            dispatch_time_t deadline = __CFTSRToDispatchTime(nextSoftDeadline);
            if (leeway > 0) {  // 2.1 可以看出当定时器支持有误差时，底层使用的是 dispatch_source 进行定时触发，而不是Mode 上的 _timerPort
                // Only use the dispatch timer if we have any leeway
                // <rdar://problem/14447675>
                
                // Cancel the mk timer
                if (rlm->_mkTimerArmed && rlm->_timerPort) {
                    AbsoluteTime dummy;
                    mk_timer_cancel(rlm->_timerPort, &dummy);
                    rlm->_mkTimerArmed = false;
                }
                
                // Arm the dispatch timer
                dispatch_source_set_timer(rlm->_timerSource, deadline, DISPATCH_TIME_FOREVER, leeway);
                rlm->_dispatchTimerArmed = true;
            } else { // 2.2 当没有误差时，使用的是 mk_timer_arm 与 mode 上端口 _timerPort
                // Cancel the dispatch timer
                if (rlm->_dispatchTimerArmed) {
                    // Cancel the dispatch timer
                    dispatch_source_set_timer(rlm->_timerSource, DISPATCH_TIME_FOREVER, DISPATCH_TIME_FOREVER, 888);
                    rlm->_dispatchTimerArmed = false;
                }
                
                // Arm the mk timer
                if (rlm->_timerPort) {
                    mk_timer_arm(rlm->_timerPort, nextSoftDeadline);
                    rlm->_mkTimerArmed = true;
                }
            }
        } 
        
        // 3. 更新下一次更新时间对应的属性
        rlm->_timerHardDeadline = nextHardDeadline;
        rlm->_timerSoftDeadline = nextSoftDeadline;
}

```
## Q&A
### a. NSTimer 常见的循环引用问题？

```Swift
    open class func scheduledTimer(timeInterval ti: TimeInterval, target aTarget: Any, selector aSelector: Selector, userInfo: Any?, repeats yesOrNo: Bool) -> Timer

```
根据上面的内容可知道，当通过类似上述方法初始化一个 Timer 时，Timer 会强引用持有 target 对象以及 userInfo 对象，那么当一个 ViewController 持有一个 Timer，并且传入的target 又是 self 时，就形成了 ViewController -> Timer -> target(ViewController) 的循环引用。那么解决方法呢？

- 通用通过上面的介绍，调用 Timer 的 invalid() 方法，将使 Timer 解除对 target 以及 userInfo 实例对象的引用，那么这样就解除了循环引用；
- 另外也可以通过带 block 参数的方法去初始化 Timer，这样将没有上述的问题。


### b. 重复的定时器是按照原始的时间轴定每一次触发的时间点，还是根据本次触发的时间点+间隔决定下一次触发的时间点？
总结：当定时器某n个时间点被延时后，会合并为只执行一次，不会将期间错过的触发时间点都补偿回来；    
并且后续的触发时间点不是根据最后一次实际触发的时间去计算，还是按照原计划的最近一次时间点去计算。
>For repeating timers, the next fire date is calculated from the original fire date regardless of tolerance applied at individual fire times, to avoid drift

对于重复的定时器，下一个触发时间点是根据原始触发时间点开始计算，而不是把 `tolerance` 计算在内

> After a repeating timer fires, it schedules the next firing for the nearest future date that is an integer multiple of the timer interval after the last scheduled fire date, within the specified tolerance

下一次时间 = interval * n + 最近上一次计划触发时间(~~上一次实际出发时间~~)

测试代码：

```Swift
class ProfileViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        self.view.backgroundColor = .white
        // 1. 每隔5秒定时器触发一次
        Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { timer in
            NSLog("timer\n");
        }
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesEnded(touches, with: event)
        // 2. 手动主线程卡死 5s 左右，用于 delay 主线程上定时器
        NSLog("hang\n");
        var a = 0
        for i in 1...3333 {
            for j in 1...9999 {
                a = i + j
            }
        }
        NSLog("hang end:\(a)\n");
    }
}
```

``` C
2023-06-22 11:33:05.593976+0800 MyLord[30076:4938024] timer

2023-06-22 11:33:10.593778+0800 MyLord[30076:4938024] timer

2023-06-22 11:33:12.405400+0800 MyLord[30076:4938024] hang

2023-06-22 11:33:17.081918+0800 MyLord[30076:4938024] hang end:13332

2023-06-22 11:33:17.082326+0800 MyLord[30076:4938024] timer

2023-06-22 11:33:20.594092+0800 MyLord[30076:4938024] timer
```

- 上面打印记录可以看出，从  11:33:05 开始每隔5秒钟执行一次定时器回调，当11:33:12 手动触发一次主线程卡顿，持续5s，卡顿在 11:33:17结束，按原计划定时器将在
11:33:15 执行一次，由于卡顿导致延迟，在延迟之后立马执行了一次 11:33:17.082326；
- 并且恢复正常后下一次执行的时间为最初计划的 11:33:20，而不是最后一次定时器执行的 11:33:17 + 5 = 11:33:22 触发下一次定时器
### c. DISPATCH_SOURCE_TYPE_TIMER 与 NSTimer 的区别
总所周知，通过     dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
 也能创建定时器，从上面定时器内部原来也可以看出来，定时器底层实际由 DISPATCH_SOURCE_TYPE_TIMER 实现，那么他们之间有什么差异呢，坦诚的说，还不了解，敬请期待后面关于 GCD 的源码解读吧。

# 引用

[https://juejin.cn/post/6844903968250789896](https://note.youdao.com/)
