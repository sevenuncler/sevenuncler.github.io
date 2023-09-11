---
layout: post
title:  "iOS GCD(2) - dispatch_group_t"
date:   2023-09-11 10:13:51 +0800
categories: iOS Foundation
---

# 使用举例

```Swift
for i in 1...10000 {
    group.enter()
    DispatchQueue.global().async {
        print("\(i)");
        self.group.leave()
    }
}

or

for i in 1...10000 {
    DispatchQueue.global().async(group: group, execute: DispatchWorkItem(block: {
        print("\(i)");
    }))
}


group.notify(queue: queue, work: DispatchWorkItem(block: {
    print("notify");
}))
```

## API


```C

// 创建 group
dispatch_group_t
dispatch_group_create(void)

// 状态同步控制
void dispatch_group_enter(dispatch_group_t dg)
void dispatch_group_leave(dispatch_group_t dg)

// or，内部其实使用 enter、leave 实现
dispatch_group_async(dispatch_group_t dg, dispatch_queue_t dq,
        dispatch_block_t db) 

// 函数指针版
DISPATCH_NOINLINE
void
dispatch_group_notify_f(dispatch_group_t dg, dispatch_queue_t dq, void *ctxt,
        dispatch_function_t func);

// Block 版    
void
dispatch_group_notify(dispatch_group_t dg, dispatch_queue_t dq,
        dispatch_block_t db)
```


# 一句话总结
使用状态值维护异步任务的执行时机；使用锁阻塞线程与唤醒的方式进行线程间同步。
# 源码分析

## 相关数据结构

```C
#define DISPATCH_GROUP_GEN_MASK         0xffffffff00000000ULL
#define DISPATCH_GROUP_VALUE_MASK       0x00000000fffffffcULL
#define DISPATCH_GROUP_VALUE_1          DISPATCH_GROUP_VALUE_MASK
#define DISPATCH_GROUP_VALUE_INTERVAL   0x0000000000000004ULL
#define DISPATCH_GROUP_HAS_NOTIFS       0x0000000000000002ULL
#define DISPATCH_GROUP_HAS_WAITERS      0x0000000000000001ULL

#define DISPATCH_CLASS_DECL(name, cluster) \
        _OS_OBJECT_DECL_PROTOCOL(dispatch_##name, dispatch_object) \
        _OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(dispatch_##name, dispatch_##name) \
        DISPATCH_CLASS_DECL_BARE(name, cluster)
        
#define _OS_OBJECT_DECL_PROTOCOL(name, super) \
        OS_OBJECT_DECL_PROTOCOL(name, <OS_OBJECT_CLASS(super)>)
        
#define OS_OBJECT_DECL_PROTOCOL(name, ...) \
        @protocol OS_OBJECT_CLASS(name) __VA_ARGS__ \
        @end

#define OS_OBJECT_CLASS(name) OS_##name

#define _OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(name, super) \
        OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(name, super)
        
#define OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(name, proto) \
        OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL_IMPL( \
                OS_OBJECT_CLASS(name), OS_OBJECT_CLASS(proto))

#define OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL_IMPL(name, proto) \
        @interface name () <proto> \
        @end

#define DISPATCH_CLASS_DECL_BARE(name, cluster) \
        OS_OBJECT_CLASS_DECL(dispatch_##name, \
        DISPATCH_##cluster##_VTABLE_HEADER(dispatch_##name))
        
#define OS_OBJECT_CLASS_DECL(name, ...) \
        struct name##_s; \
        struct name##_extra_vtable_s { \
            __VA_ARGS__; \
        }; \
        struct name##_vtable_s { \
            _OS_OBJECT_CLASS_HEADER(); \
            struct name##_extra_vtable_s _os_obj_vtable; \
        }; \
        OS_OBJECT_EXTRA_VTABLE_DECL(name, name) \
        extern const struct name##_vtable_s OS_OBJECT_CLASS_SYMBOL(name) \
                asm(OS_OBJC_CLASS_RAW_SYMBOL_NAME(OS_OBJECT_CLASS(name)))
                
#define DISPATCH_OBJECT_VTABLE_HEADER(x) \
    unsigned long const do_type; \
    const char *const do_kind; \
    void (*const do_dispose)(struct x##_s *, bool *allow_free); \
    size_t (*const do_debug)(struct x##_s *, char *, size_t); \
    void (*const do_invoke)(struct x##_s *, dispatch_invoke_context_t, \
            dispatch_invoke_flags_t)
            
#define OS_OBJECT_CLASS_DECL(name, ...) \
        struct name##_s; \
        struct name##_extra_vtable_s { \
            __VA_ARGS__; \
        }; \
        struct name##_vtable_s { \
            _OS_OBJECT_CLASS_HEADER(); \
            struct name##_extra_vtable_s _os_obj_vtable; \
        }; \
        OS_OBJECT_EXTRA_VTABLE_DECL(name, name) \
        extern const struct name##_vtable_s OS_OBJECT_CLASS_SYMBOL(name) \
                asm(OS_OBJC_CLASS_RAW_SYMBOL_NAME(OS_OBJECT_CLASS(name)))


// 原始定义
DISPATCH_CLASS_DECL(group, OBJECT);
// 展开
_OS_OBJECT_DECL_PROTOCOL(dispatch_group, dispatch_object) \
_OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(dispatch_##name, dispatch_##name) \
DISPATCH_CLASS_DECL_BARE(group, OBJECT)

OS_OBJECT_DECL_PROTOCOL(dispatch_group, <OS_dispatch_object>)
OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL(dispatch_group, dispatch_object)\
DISPATCH_CLASS_DECL_BARE(group, OBJECT)

OS_OBJECT_DECL_PROTOCOL(dispatch_group, <OS_dispatch_object>)
OS_OBJECT_CLASS_IMPLEMENTS_PROTOCOL_IMPL(OS_dispatch_group, OS_dispatch_object)\
DISPATCH_CLASS_DECL_BARE(group, OBJECT)

OS_OBJECT_DECL_PROTOCOL(dispatch_group, <OS_dispatch_object>)
@interface OS_dispatch_group () <OS_dispatch_object>
@end
DISPATCH_CLASS_DECL_BARE(group, OBJECT)

OS_OBJECT_DECL_PROTOCOL(dispatch_group, <OS_dispatch_object>)
@interface OS_dispatch_group () <OS_dispatch_object>
@end
        
struct dispatch_group_s;
struct dispatch_group_extra_vtable_s {
    unsigned long const do_type;
    const char *const do_kind;
    void (*const do_dispose)(struct dispatch_group_s *, bool *allow_free);
    size_t (*const do_debug)(struct dispatch_group_s *, char *, size_t);
    void (*const do_invoke)(struct dispatch_group_s *, dispatch_invoke_context_t, dispatch_invoke_flags_t)
};
struct dispatch_group_vtable_s {
    void *_os_obj_objc_class_t[5];
    struct dispatch_group_extra_vtable_s _os_obj_vtable;
};
extern const struct dispatch_group_vtable_s _OS_dispatch_group_vtable;
extern const struct dispatch_group_vtable_s OS_dispatch_group_class(dispatch_group) 
        asm("_OBJC_CLASS_$_" OS_STRINGIFY(OS_dispatch_group))
        

typedef struct dispatch_group_s *dispatch_group_t;
struct dispatch_group_s {
    DISPATCH_OBJECT_HEADER(group);
    DISPATCH_UNION_LE(uint64_t volatile dg_state,
            uint32_t dg_bits,
            uint32_t dg_gen
    ) DISPATCH_ATOMIC64_ALIGN;
    struct dispatch_continuation_s *volatile dg_notify_head;
    struct dispatch_continuation_s *volatile dg_notify_tail;
};

struct dispatch_group_s {
struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_group_vtable_s *do_vtable;
    int volatile do_ref_cnt; \
    int volatile do_xref_cnt
    struct dispatch_group_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer
    DISPATCH_UNION_LE(uint64_t volatile dg_state,
            uint32_t dg_bits,
            uint32_t dg_gen
    ) DISPATCH_ATOMIC64_ALIGN;
    struct dispatch_continuation_s *volatile dg_notify_head;
    struct dispatch_continuation_s *volatile dg_notify_tail;
};

```

## 创建 Group
1. 创建 dispatch_group_t 对象
2. 将当前队列作为 do_targetq
```C
dispatch_group_t
dispatch_group_create(void)
{
    return _dispatch_group_create_with_count(0);
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_group_t
_dispatch_group_create_with_count(uint32_t n)
{
    dispatch_group_t dg = _dispatch_object_alloc(DISPATCH_VTABLE(group),
            sizeof(struct dispatch_group_s));
    dg->do_next = DISPATCH_OBJECT_LISTLESS;
    dg->do_targetq = _dispatch_get_default_queue(false);
    if (n) {
        os_atomic_store2o(dg, dg_bits,
                -n * DISPATCH_GROUP_VALUE_INTERVAL, relaxed);
        os_atomic_store2o(dg, do_ref_cnt, 1, relaxed); // <rdar://22318411>
    }
    return dg;
}

```

## Enter
1. 将 group 中 dg_bits(注：32位)减 DISPATCH_GROUP_VALUE_INTERVAL(减法换成需用补码相加方便理解)
2. 如果为首次进入，则引用计数加1

```C
void
dispatch_group_enter(dispatch_group_t dg)
{
    // The value is decremented on a 32bits wide atomic so that the carry
    // for the 0 -> -1 transition is not propagated to the upper 32bits.
    uint32_t old_bits = os_atomic_sub_orig2o(dg, dg_bits,
            DISPATCH_GROUP_VALUE_INTERVAL, acquire);
    uint32_t old_value = old_bits & DISPATCH_GROUP_VALUE_MASK;
    if (unlikely(old_value == 0)) {
        _dispatch_retain(dg); // <rdar://problem/22318411>
    }
    if (unlikely(old_value == DISPATCH_GROUP_VALUE_MAX)) {
        DISPATCH_CLIENT_CRASH(old_bits,
                "Too many nested calls to dispatch_group_enter()");
    }
}
```

## Leave
1. 将dg.dg_state(即将gd_bits强制转换为64位版) 加 DISPATCH_GROUP_VALUE_INTERVAL
2. 判断旧值是否为最后一个未 leave 前的状态
3. 如果是，则清空 DISPATCH_GROUP_HAS_NOTIFS 和 DISPATCH_GROUP_HAS_WAITERS 状态，并唤醒阻塞的任务
3. 否则，判断是否 enter和leave数量不匹配
4. 退出

```C
void
dispatch_group_leave(dispatch_group_t dg)
{
    // The value is incremented on a 64bits wide atomic so that the carry for
    // the -1 -> 0 transition increments the generation atomically.
    uint64_t new_state, old_state = os_atomic_add_orig2o(dg, dg_state,
            DISPATCH_GROUP_VALUE_INTERVAL, release);
    uint32_t old_value = (uint32_t)(old_state & DISPATCH_GROUP_VALUE_MASK);

    if (unlikely(old_value == DISPATCH_GROUP_VALUE_1)) {
        old_state += DISPATCH_GROUP_VALUE_INTERVAL;
        do {
            new_state = old_state;
            if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
                new_state &= ~DISPATCH_GROUP_HAS_WAITERS;
                new_state &= ~DISPATCH_GROUP_HAS_NOTIFS;
            } else {
                // If the group was entered again since the atomic_add above,
                // we can't clear the waiters bit anymore as we don't know for
                // which generation the waiters are for
                new_state &= ~DISPATCH_GROUP_HAS_NOTIFS;
            }
            if (old_state == new_state) break;
        } while (unlikely(!os_atomic_cmpxchgv2o(dg, dg_state,
                old_state, new_state, &old_state, relaxed)));
        return _dispatch_group_wake(dg, old_state, true);
    }

    if (unlikely(old_value == 0)) {
        DISPATCH_CLIENT_CRASH((uintptr_t)old_value,
                "Unbalanced call to dispatch_group_leave()");
    }
}
```
## 唤醒
1. 判断是否有待 Notify 的任务，存在则取出任务和目标执行队列，然后按照正常的 dispatch 进行任务执行
2. 如果有正在 wait 中的阻塞线程，则 _dispatch_wake_by_address 唤醒目标阻塞线程。
3. 减少对应的线程数

```C
DISPATCH_NOINLINE
static void
_dispatch_group_wake(dispatch_group_t dg, uint64_t dg_state, bool needs_release)
{
    uint16_t refs = needs_release ? 1 : 0; // <rdar://problem/22318411>

    // 唤醒 dispatch_group_notify 的任务
    if (dg_state & DISPATCH_GROUP_HAS_NOTIFS) {
        dispatch_continuation_t dc, next_dc, tail;

        // Snapshot before anything is notified/woken <rdar://problem/8554546>
        // 获取当前的快照，即获取当前的 tail，方便取任务时做截断，防止执行过程中加入新的任务
        dc = os_mpsc_capture_snapshot(os_mpsc(dg, dg_notify), &tail);
        do {
            dispatch_queue_t dsn_queue = (dispatch_queue_t)dc->dc_data;
            // 使用 tail 作为截断，只取当前时刻的任务(不执行后续新加入的，下一次执行)
            next_dc = os_mpsc_pop_snapshot_head(dc, tail, do_next);
            _dispatch_continuation_async(dsn_queue, dc,
                    _dispatch_qos_from_pp(dc->dc_priority), dc->dc_flags);
            _dispatch_release(dsn_queue);
        } while ((dc = next_dc));

        refs++;
    }
    // 唤醒 dispatch_group_wait 的线程
    if (dg_state & DISPATCH_GROUP_HAS_WAITERS) {
        _dispatch_wake_by_address(&dg->dg_gen);
    }

    if (refs) _dispatch_release_n(dg, refs);
}
```
# 添加异步通知任务
1. 将任务添加到 dg.dg_notify 链表尾部
2. 将gd.dg_state 的标记为 DISPATCH_GROUP_HAS_NOTIFS ，代表有 notify 任务
3. 如果当前没有需要等待的同步操作，则执行唤醒 notify 任务

```C
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_group_notify(dispatch_group_t dg, dispatch_queue_t dq,
        dispatch_continuation_t dsn)
{
    uint64_t old_state, new_state;
    dispatch_continuation_t prev;

    dsn->dc_data = dq;
    _dispatch_retain(dq);

    prev = os_mpsc_push_update_tail(os_mpsc(dg, dg_notify), dsn, do_next);
    if (os_mpsc_push_was_empty(prev)) _dispatch_retain(dg);
    os_mpsc_push_update_prev(os_mpsc(dg, dg_notify), prev, dsn, do_next);
    if (os_mpsc_push_was_empty(prev)) {
        os_atomic_rmw_loop2o(dg, dg_state, old_state, new_state, release, {
            new_state = old_state | DISPATCH_GROUP_HAS_NOTIFS;
            if ((uint32_t)old_state == 0) {
                os_atomic_rmw_loop_give_up({
                    return _dispatch_group_wake(dg, new_state, false);
                });
            }
        });
    }
}
```

## 同步等待
1. 如果当前无同步状态需要等待，则直接返回结束等待。
2. 如果存在需要等待的同步状态，并且超时已经过期，则返回超时状态
3. 否则，更新 dg.dg_states，标记 DISPATCH_GROUP_HAS_WAITERS 状态
4. 阻塞当前线程

```C
long
dispatch_group_wait(dispatch_group_t dg, dispatch_time_t timeout)
{
    uint64_t old_state, new_state;

    os_atomic_rmw_loop2o(dg, dg_state, old_state, new_state, relaxed, {
        if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
            os_atomic_rmw_loop_give_up_with_fence(acquire, return 0);
        }
        if (unlikely(timeout == 0)) {
            os_atomic_rmw_loop_give_up(return _DSEMA4_TIMEOUT());
        }
        new_state = old_state | DISPATCH_GROUP_HAS_WAITERS;
        if (unlikely(old_state & DISPATCH_GROUP_HAS_WAITERS)) {
            os_atomic_rmw_loop_give_up(break);
        }
    });

    return _dispatch_group_wait_slow(dg, _dg_state_gen(new_state), timeout);
}
```


1. 利用 dg->dg_gen 进行线程的阻塞与唤醒

```C
DISPATCH_NOINLINE
static long
_dispatch_group_wait_slow(dispatch_group_t dg, uint32_t gen,
        dispatch_time_t timeout)
{
    for (;;) {
        int rc = _dispatch_wait_on_address(&dg->dg_gen, gen, timeout, 0);
        if (likely(gen != os_atomic_load2o(dg, dg_gen, acquire))) {
            return 0;
        }
        if (rc == ETIMEDOUT) {
            return _DSEMA4_TIMEOUT();
        }
    }
}
```

## dispatch_group_async
本质：对 dispatch_group_enter 和 dispatch_group_leave 进行了封装
1. 对任务标记 DC_FLAG_CONSUME | DC_FLAG_GROUP_ASYNC
2. 调度任务前使用 dispatch_group_enter 标记进入同步状态
3. 执行任务时判断 dc_flags 为 DC_FLAG_GROUP_ASYNC，则执行 _dispatch_continuation_with_group_invoke
4. 执行完任务，调用 dispatch_group_leave
```C
void
dispatch_group_async(dispatch_group_t dg, dispatch_queue_t dq,
        dispatch_block_t db)
{
    dispatch_continuation_t dc = _dispatch_continuation_alloc();
    uintptr_t dc_flags = DC_FLAG_CONSUME | DC_FLAG_GROUP_ASYNC;
    dispatch_qos_t qos;

    qos = _dispatch_continuation_init(dc, dq, db, 0, dc_flags);
    _dispatch_continuation_group_async(dg, dq, dc, qos);
}

// 内部本质为 dispatch_group_enter 和 leave
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_group_async(dispatch_group_t dg, dispatch_queue_t dq,
        dispatch_continuation_t dc, dispatch_qos_t qos)
{
    dispatch_group_enter(dg);
    dc->dc_data = dg;
    _dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}

// 与普通 GCD 任务一样的流程
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_async(dispatch_queue_class_t dqu,
        dispatch_continuation_t dc, dispatch_qos_t qos, uintptr_t dc_flags)
{
#if DISPATCH_INTROSPECTION
    if (!(dc_flags & DC_FLAG_NO_INTROSPECTION)) {
        _dispatch_trace_item_push(dqu, dc);
    }
#else
    (void)dc_flags;
#endif
    return dx_push(dqu._dq, dc, qos);
}

```

### 判断 GCD 任务是否为 group 任务

```C
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_invoke_inline(dispatch_object_t dou,
        dispatch_invoke_flags_t flags, dispatch_queue_class_t dqu)
{
    dispatch_continuation_t dc = dou._dc, dc1;
    dispatch_invoke_with_autoreleasepool(flags, {
        uintptr_t dc_flags = dc->dc_flags;
        // Add the item back to the cache before calling the function. This
        // allows the 'hot' continuation to be used for a quick callback.
        //
        // The ccache version is per-thread.
        // Therefore, the object has not been reused yet.
        // This generates better assembly.
        _dispatch_continuation_voucher_adopt(dc, dc_flags);
        if (!(dc_flags & DC_FLAG_NO_INTROSPECTION)) {
            _dispatch_trace_item_pop(dqu, dou);
        }
        if (dc_flags & DC_FLAG_CONSUME) {
            dc1 = _dispatch_continuation_free_cacheonly(dc);
        } else {
            dc1 = NULL;
        }
        // 此处进行执行任务并leave 操作
        if (unlikely(dc_flags & DC_FLAG_GROUP_ASYNC)) {
            _dispatch_continuation_with_group_invoke(dc);
        } else {
            _dispatch_client_callout(dc->dc_ctxt, dc->dc_func);
            _dispatch_trace_item_complete(dc);
        }
        if (unlikely(dc1)) {
            _dispatch_continuation_free_to_cache_limit(dc1);
        }
    });
    _dispatch_perfmon_workitem_inc();
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_with_group_invoke(dispatch_continuation_t dc)
{
    struct dispatch_object_s *dou = dc->dc_data;
    unsigned long type = dx_type(dou);
    if (type == DISPATCH_GROUP_TYPE) {
        // 执行任务
        _dispatch_client_callout(dc->dc_ctxt, dc->dc_func);
        _dispatch_trace_item_complete(dc);
        // 结束同步状态
        dispatch_group_leave((dispatch_group_t)dou);
    } else {
        DISPATCH_INTERNAL_CRASH(dx_type(dou), "Unexpected object type");
    }
}
```
# 适用场景
1. 如等待多个不同的任务都完成才开始的任务，比如一个 UI 需要等待多个请求回来，再组合数据进行 UI 刷新
2. dispatch_group 是针对任务级别的，即 dispatch_group 只关心任务与任务之间的同步关系，不在乎任务是不是在同一个队列; 而 dispatch_barrier 是返回限制在同一个队列中的任务之间的同步关系，即通过队列的同步机制进行任务之间的同步控制。
# 使用注意点
1. enter 和 group 必须配对使用
2. dispatch_group 同步任务是有上限的
# 总结
1. dispatch_group 比较巧妙的使用位运算记录当前同步等待任务的状态，通过 enter、leave 维护该值达到平衡，当执行最后一个 leave 状态后，进行状态清楚，唤醒待执行任务和待唤醒线程。
