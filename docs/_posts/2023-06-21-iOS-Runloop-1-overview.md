---
layout: post
title:  "iOS Runloop(1) - 机制解读"
date:   2023-06-21 23:13:51 +0800
categories: iOS Foundation
---
# 前言
>1. **__CFRunLoopDoSources0 是如何收集 block 的？**
没有收集，只是单纯取出被标记为 signaled 的source0 事件，假如匹配当前模式则执行

```C
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(perform, info)
```
> 2. **CFRUNLOOP_WAKEUP_FOR_TIMER 里面执行了 timer 回调了吗**？执行了    
> 3. **__CFRunLoopDoTimers 里收集 Timer 的回调了？**    
没有收集，取出符合要求的即执行，即 Timer 对应的端口唤醒后， 通过 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ 执行了对应的回调函数
> 4. Source0 有哪些？timer 算吗     
source0 是指只需要唤醒线程一次就可以处理完的时间，包括用户触摸/手势事件等，timer 不算 source0 事件
> 5. Gesture Event 与 UIEvent 是独立处理的吗？
都在触摸事件对应的 Source0 对应的回调函数中处理 __eventFetcherSourceCallback，优先级手势 > UIResponder
> 6. __CFRunLoopDoBlocks 是执行什么?
一些在当前 Block 上待执行的 Block
> 7. 手势触摸事件属于 Source0 还是 Source1？
__CFRunLoopDoSources0 是当前 Runloop 上添加的source0事件，(app 内部事件处理：UIEvent 时间，CFSocket 事件，触摸事件其实是Source1接收系统事件后在回调 __IOHIDEventSystemClientQueueCallback() 内触发的 Source0，Source0 再触发的 _UIApplicationHandleEventQueue()。source0一定是要唤醒runloop及时响应并执行的，如果runloop此时在休眠等待系统的 mach_msg事件，那么就会通过source1来唤醒runloop执行。)  
> 8. source1 ？基于端口，当收到端口消息时立即唤醒，并执行，属于系统 mach_msg 事件

```C
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__
```

> 9. 唤醒 Runloop 使用哪个方法？

```C
CFRunLoopWakeUp(rl);
```
带着上面的问题，开始我们的 Runloop 之旅。
# 一句话描述
> Runloop 即 Event Loop，是循环的处理事件的模型。
# 整体运行模式
AppDelegate - CFRunLoopRunInMode - CFRunLoopRunSpecific - __CFRunLoopRun

## CFRunLoopRunInMode

```C
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```
CFRunLoopRunInMode 无法指定 runloop 实例，所以需要在 runloop 对应的线程才能去切换 Mode.

## CFRunLoopRunSpecific
让指定 runloop 实例按照指定模式 `mode` 运行一次循环，运行结束后恢复到上一个运行的模式。
`seconds`，同时该接口可以设置唤醒超时时间  (假如进入 runloop 后没有任何唤醒事件，则该超时时间对应的定时器超时后会唤醒一次该unloop);
`returnAfterSourceHandled`，即每执行一个 Source0 回调后是否退出。
```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    // 假如正在释放 Runloop 则返回完成退出
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 1. 获取要运行的 Mode 对象
    CFRunLoopModeRef currentMode = __CFRunLoopCopyMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
    if (currentMode) {
            __CFRunLoopModeUnlock(currentMode);
            CFRelease(currentMode);
        }
    __CFRunLoopUnlock(rl);
    return kCFRunLoopRunFinished;
    }
    __CFRunLoopModeLock(currentMode);
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    // 2. 保留之前运行Mode，待本次运行后恢复
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
    
    // 3. 通知 Observers kCFRunLoopEntry 事件
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    
    // 4. 在runloop 上执行指定 Mode
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);

    // 5. 通知 Observers kCFRunLoopExit 退出事件
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    __CFRunLoopModeUnlock(currentMode);
    CFRelease(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    // 6. 恢复到上一次运行的 Mode
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

## __CFRunLoopRun
在指定 Mode 下开启一次循环，直到因为某些原因(returnAfterSourceHandled 执行一次、外部停止、无source/timer 等)而退出循环。
退出原因：

```C
typedef CF_ENUM(SInt32, CFRunLoopRunResult) {
    kCFRunLoopRunFinished = 1, // 无 Mode
    kCFRunLoopRunStopped = 2, // 手动停止了
    kCFRunLoopRunTimedOut = 3, // 超时
    kCFRunLoopRunHandledSource = 4 // returnAfterSourceHandled 执行一次之后退出
};
```


``` c
/* rl, rlm are locked on entrance and exit */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();

    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
    return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
    rlm->_stopped = false;
    return kCFRunLoopRunStopped;
    }
    
    // 1. 为主线程 Runloop 指定 dispatch_port，用于主线程有 GCD 任务时唤醒 Runloop
    #if __HAS_DISPATCH__
    __CFPort dispatchPort = CFPORT_NULL;
    Boolean libdispatchQSafe = (pthread_main_np() == 1) && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
#endif

    // 2. 根据传入的 seconds 设置 Runloop 超时唤醒逻辑
    uint64_t termTSR = 0ULL;
#if __HAS_DISPATCH__
    dispatch_source_t timeout_timer = NULL;
#endif
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
        termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
#if __HAS_DISPATCH__
    dispatch_queue_t queue = (pthread_main_np() == 1) ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
    timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

        CFRetain(rl);
        dispatch_source_set_event_handler(timeout_timer, ^{
            CFRUNLOOP_WAKEUP_FOR_TIMEOUT();
            cf_trace(KDEBUG_EVENT_CFRL_DID_WAKEUP_FOR_TIMEOUT, rl, 0, 0, 0);
            CFRunLoopWakeUp(rl);
            // The interval is DISPATCH_TIME_FOREVER, so this won't fire again
        });
        dispatch_source_set_cancel_handler(timeout_timer, ^{
            CFRelease(rl);
        });

        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
#endif
    }
    
    // 3. 开始进入内部循环
    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
    __CFPortSet waitSet = rlm->_portSet;

        // 3.1 设置忽略唤醒标志
        __CFRunLoopUnsetIgnoreWakeUps(rl);

        // 3.2 通知 Observers kCFRunLoopBeforeTimers 事件
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) {
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        }
        
        // 3.3 通知 Observers kCFRunLoopBeforeSources 事件
        if (rlm->_observerMask & kCFRunLoopBeforeSources) {
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        }
    
    // 3.4 处理？？？
    __CFRunLoopDoBlocks(rl, rlm);
    
    // 3.5 处理 Source0 事件？？？
    Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
    
    // 3.6 ？？？？
    if (sourceHandledThisLoop) {
        __CFRunLoopDoBlocks(rl, rlm);
    }
    
    // 3.7 执行 dispatchPort 事件？？？
    Boolean poll = sourceHandledThisLoop || (0ULL == termTSR);
    if (CFPORT_NULL != dispatchPort && !didDispatchPortLastTime) {
        msg = (mach_msg_header_t *)msg_buffer;
        if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL, rl, rlm)) {
            goto handle_msg;
        }
    }
    didDispatchPortLastTime = false;
    
    // 3.8 通知将要进入休眠
    if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
    __CFRunLoopSetSleeping(rl);
    
    msg = (mach_msg_header_t *)msg_buffer;
    // 3.9 休眠等待 waitSet 端口集任意端口唤醒
    __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy, rl, rlm);
    __CFRunLoopLock(rl);
    __CFRunLoopModeLock(rlm);
    // 记录累计休眠时长
    rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));
    __CFRunLoopSetIgnoreWakeUps(rl);

    // user callouts now OK again
    __CFRunLoopUnsetSleeping(rl);
    
    // 3.10 通知休眠结束
    if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
    
    // 3.11 消息处理
    handle_msg:;
    __CFRunLoopSetIgnoreWakeUps(rl);
    
    // livePort 在__CFRunLoopServiceMachPort 中处理，为当前唤醒事件对应的端口
    if (CFPORT_NULL == livePort) { // 3.12 仅唤醒无需处理对应事件
        CFRUNLOOP_WAKEUP_FOR_NOTHING();
    } else if (livePort == rl->_wakeUpPort) { // 3.13 runloop 超时唤醒
        CFRUNLOOP_WAKEUP_FOR_WAKEUP();
    } else if (rlm->_timerPort != CFPORT_NULL && livePort == rlm->_timerPort) { // 3.14 定时器唤醒
        CFRUNLOOP_WAKEUP_FOR_TIMER();
        if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                // Since we'll be resetting the same timer as before
                // with the same deadlines, we need to reset these
                // values so that the arm next timer code can
                // correctly find the next timer in the list and arm
                // the underlying timer.
                rlm->_timerSoftDeadline = UINT64_MAX;
                rlm->_timerHardDeadline = UINT64_MAX;
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
    } else if (livePort == dispatchPort) { // 3.15 主线程 GCD 任务唤醒 /* --- DISPATCHES  --- */
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            cf_trace(KDEBUG_EVENT_CFRL_DID_WAKEUP_FOR_DISPATCH, rl, rlm, livePort, 0);
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
            CFRUNLOOP_ARP_BEGIN(NULL)
            cf_trace(KDEBUG_EVENT_CFRL_IS_CALLING_DISPATCH | DBG_FUNC_START, rl, rlm, msg, livePort);
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            cf_trace(KDEBUG_EVENT_CFRL_IS_CALLING_DISPATCH | DBG_FUNC_END, rl, rlm, msg, livePort);
            CFRUNLOOP_ARP_END()

            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
    } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            cf_trace(KDEBUG_EVENT_CFRL_DID_WAKEUP_FOR_SOURCE, rl, rlm, 0, 0);
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
                mach_msg_header_t *reply = NULL;
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            } else {
                os_log_error(_CFOSLog(), "__CFRunLoopModeFindSourceForMachPort returned NULL for mode '%@' livePort: %u", rlm->_name, livePort);
            }
        }
        
        /* --- BLOCKS --- */
    
    __CFRunLoopDoBlocks(rl, rlm);
        
    if (sourceHandledThisLoop && stopAfterHandle) {
        retVal = kCFRunLoopRunHandledSource;
        } else if (termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
    } else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
        retVal = kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        rlm->_stopped = false;
        retVal = kCFRunLoopRunStopped;
    } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
        retVal = kCFRunLoopRunFinished;
    }

    } while (0 == retVal);
    
    return retVal;
}

```
大致流程：

0. 手动调用某种模式运行
1. 通知 Observers 将要进入 Runloop
2. 设置 Runloop 超时器，假如超过指定时间则唤醒 Runloop，内部循环退出时取消定时器
3. 通知 Observers 将要执行 source0 和 timers 事件
4. 执行 Blocks
5. 执行 source0 事件
6. 通知将要进入休眠
7. 休眠并等待端口消息唤醒
8. 通知休眠结束
9. 开始 handle_msg: 处理
10. 如果没有活跃端口，不做处理
11. 如果是 runloop 的端口消息，则什么也不做，并跳到步骤3
12. 如果是 Timer 端口，则执行该事件附近的定时器回调，并设置下一个定时器事件
13. 如果是 GCD 主线程事件，则执行主线 Block
14. 否则是其他 source1 则去取出对应的 source1 , 执行 source1 回调
15. 执行期间产生的 Blocks
16. 返回到步骤3
17. 当前模式运行退出，则恢复到上一个运行模式

> CFRunLoopRun() 内部是循环、CFRunLoopRunInMode() 只运行一次，两者之间运行不会冲突吗？
> 不会，内部使用     __CFRunLoopLock(rl) 对目标 runloop 进行加锁，保证一个时间点不会同时有两个调用在执行; 通知通过 __CFRunLoopSetIgnoreWakeUps 设置标志位，当前正在执行中时，不会重复唤起循环。

## CFRunLoopWakeUp
Runloop 创建时会默认创建 `_wakeUpPort` 端口，用于 runloop 超时未运行时唤醒

``` C
void CFRunLoopWakeUp(CFRunLoopRef rl) {
    // 1. 假如已经唤醒则直接返回
    if (__CFRunLoopIsIgnoringWakeUps(rl)) {
        __CFRunLoopUnlock(rl);
        return;
    }
    
    // 2. 通过给端口发消息唤醒 Runloop
#if TARGET_OS_MAC
    kern_return_t ret;
    /* We unconditionally try to send the message, since we don't want
     * to lose a wakeup, but the send may fail if there is already a
     * wakeup pending, since the queue length is 1. */
    ret = __CFSendTrivialMachMessage(rl->_wakeUpPort, 0, MACH_SEND_TIMEOUT, 0);
    if (ret != MACH_MSG_SUCCESS && ret != MACH_SEND_TIMED_OUT) CRASH("*** Unable to send message to wake up port. (%x) ***", ret);
#elif TARGET_OS_BSD
    __CFPortTrigger(rl->_wakeUpPort);
#else
#error "required"
#endif
}
```


## __CFRunLoopDoBlocks
1. 该函数内部的待执行的 Block 是从哪里来？是否在执行 source0、source1、timer等期间也会产生 block
那么我们先来看一下改函数的源码：

```C
static Boolean __CFRunLoopDoBlocks(CFRunLoopRef rl, CFRunLoopModeRef rlm) { // Call with rl and rlm locked
    // 1. 存储待执行 Blocks 的链表结构，如果为空则返回
    if (!rl->_blocks_head) return false;
    if (!rlm || !rlm->_name) return false;
    Boolean did = false;
    struct _block_item *head = rl->_blocks_head;
    struct _block_item *tail = rl->_blocks_tail;
    rl->_blocks_head = NULL;
    rl->_blocks_tail = NULL;
    CFSetRef commonModes = rl->_commonModes;
    CFStringRef curMode = rlm->_name;
    __CFRunLoopModeUnlock(rlm);
    __CFRunLoopUnlock(rl);
    struct _block_item *prev = NULL;
    struct _block_item *item = head;
    // 2. 遍历待执行 blocks 链表
    while (item) {
        struct _block_item *curr = item;
        item = item->_next;
        Boolean doit = false;
        // 2.1 判断当前 block 是否需要在该模式下执行
        if (_kCFRuntimeIDCFString == CFGetTypeID(curr->_mode)) {
            doit = CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        } else {
            doit = CFSetContainsValue((CFSetRef)curr->_mode, curMode) || (CFSetContainsValue((CFSetRef)curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        }
        if (!doit) prev = curr;
        // 2.2 
        if (doit) {
            if (prev) prev->_next = item; // 释放当前节点 curr，将上一个链表节点(假如有)连接到下一个未执行节点
            if (curr == head) head = item; // 假如当前执行的为 head，则将 head 往后移动一个 item(因为当前该 item 在执行完后就要被释放内存)
            if (curr == tail) tail = prev; // 同上
            void (^block)(void) = curr->_block;
            CFRelease(curr->_mode);
            free(curr);
            if (doit) {
                CFRUNLOOP_ARP_BEGIN(rl);
                // 2.3 执行 Block
                __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
                CFRUNLOOP_ARP_END();
                did = true;
            }
            Block_release(block); // do this before relocking to prevent deadlocks where some yahoo wants to run the run loop reentrantly from their dealloc
    }
    }
    __CFRunLoopLock(rl);
    __CFRunLoopModeLock(rlm);
    if (head && tail) { // 假如还有未执行的 block，重新形成新的链表
        tail->_next = rl->_blocks_head;
        rl->_blocks_head = head;
        if (!rl->_blocks_tail) rl->_blocks_tail = tail;
    }
    return did;
}
```
可以看出，执行的 blocks 都来自 rl->_blocks_head，那么看下哪些情况会往 _blocks_head 添加内容，搜索了下有 `CFRunLoopPerformBlock`, 对应到上层 API 如

```Swift
RunLoop.main.perform(_ block: @escaping @Sendable () -> Void)

```

## __CFRunLoopDoSources0
据了解：source0 事件源的运行方式为标记 source0 待处理，然后手动唤醒 runloop，那么如何验证呢？原因：
1. `__CFRunLoopDoSource0` 执行的时候根据 `__CFRunLoopSourceIsSignaled(rls)` 判断是否需要执行；
2. 

```C
static Boolean __CFRunLoopDoSources0(CFRunLoopRef rl, CFRunLoopModeRef rlm, Boolean stopAfterHandle) {    /* DOES CALLOUT */
    CFTypeRef sources = NULL;
    Boolean sourceHandled = false;
    
    /* Fire the version 0 sources */
    if (NULL != rlm->_sources0 && 0 < CFSetGetCount(rlm->_sources0)) {
        // 收集 source0 对象放到 sources 中，如果存在多个source0则为数组
        CFSetApplyFunction(rlm->_sources0, (__CFRunLoopCollectSources0), &sources);
    }
    if (NULL != sources) {
        __CFRunLoopModeUnlock(rlm);
        __CFRunLoopUnlock(rl);
        // sources is either a single (retained) CFRunLoopSourceRef or an array of (retained) CFRunLoopSourceRef
        // 判断 sources 类型是否为单个类型(非数组)
        if (CFGetTypeID(sources) == CFRunLoopSourceGetTypeID()) {
            CFRunLoopSourceRef rls = (CFRunLoopSourceRef)sources;
            
            sourceHandled = __CFRunLoopDoSource0(rl, rls);
            
        } else { // 数组则遍历
            CFIndex cnt = CFArrayGetCount((CFArrayRef)sources);
            CFArraySortValues((CFMutableArrayRef)sources, CFRangeMake(0, cnt), (__CFRunLoopSourceComparator), NULL);
            for (CFIndex idx = 0; idx < cnt; idx++) {
                CFRunLoopSourceRef rls = (CFRunLoopSourceRef)CFArrayGetValueAtIndex((CFArrayRef)sources, idx);
                
                sourceHandled = __CFRunLoopDoSource0(rl, rls);
                
                if (stopAfterHandle && sourceHandled) {
                    break;
                }
            }
        }
        CFRelease(sources);
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);
    }

    return sourceHandled;
}

```
> 问题1：那么添加到 rlm->_sources0 有哪些呢？是用哪个符号可以断点？
> 可以使用符号断点 CFRunLoopAddSource 配合 bt 看添加 source 的堆栈

> 问题2：modes 是存在哪个属性？commomModes、commonModeItems 是什么？  
> 从 `__CFRunLoopCopyMode` 函数可以看出，具体的某一个 Mode 是存在 runloop->_modes 中
>
> `_commonModes` 内容为字符串，即具体的每一个 Mode 的 ModeName，  
> 
> `_commonModeItems` 用于存放 mode 为 common 的 source

Source0 使用步骤：

``` C
// 0. 创建 source0 
  CFRunLoopSourceContext source_context = CFRunLoopSourceContext();
  source_context.info = this;
  source_context.perform = RunWorkSource;
  work_source_ = CFRunLoopSourceCreate(NULL,  // allocator
                                       1,     // priority
                                       &source_context);
                                       
// 1. 添加到目标 Rlunloop
  CFRunLoopAddSource(run_loop_, work_source_, mode_);
  
// 2. 标记 source0 事件待处理
  CFRunLoopSourceSignal(work_source_);

// 3. 唤醒 source0 所在目标 runloop
  CFRunLoopWakeUp(run_loop_);
```

>目前系统的 Source0 事件有哪些?    
> 1. PurpleEventSignalCallback。  
> 2. __eventFetcherSourceCallback，接收触摸、手势
> 3. FBSSerialQueueRunLoopSourceHandler
> 4. __eventQueueSourceCallback

除了 3. 用于处理触摸事件用，其他暂不知道作用。
## __CFRunLoopServiceMachPort
监听端口列表任意端口的消息

```C
// 接收消息
    ret = mach_msg(msg, MACH_RCV_MSG|MACH_RCV_VOUCHER|MACH_RCV_LARGE|((TIMEOUT_INFINITY != timeout) ? MACH_RCV_TIMEOUT : 0)|MACH_RCV_TRAILER_TYPE(MACH_MSG_TRAILER_FORMAT_0)|MACH_RCV_TRAILER_ELEMENTS(MACH_RCV_TRAILER_AV), 0, msg->msgh_size, port, timeout, MACH_PORT_NULL);

```

## CFRUNLOOP_WAKEUP_FOR_WAKEUP
// 啥也没做
``` C
#define    CFRUNLOOP_WAKEUP_FOR_WAKEUP() do { } while (0)

```
## __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
执行 DispatchQueue.main.async 中的 block

## 如何切 Mode

```Swift
    CFRunLoopRunInMode(CFRunLoopMode.defaultMode, 1, true)
```




# 引用
官网：https://github.com/apple/swift-corelibs-foundation/  
iOS 博客：https://juejin.cn/user/1591748569076078    
Runloop 内部详解：http://www.xbhp.cn/news/160874.html    
手势详解：https://juejin.cn/post/6844904175415853064     
牛：https://www.jianshu.com/p/c294d1bd963d     
iOS 事件传递与处理：https://juejin.cn/post/6956756476559884301     
