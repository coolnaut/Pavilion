> 知之为知之，不知为不知，是知也。

# 1. RunLoopSource

事件源，它是唤醒线程和线程任务关键之一，事件源有两种，一个是输入事件源，一个是定时器事件源。首先看看输入事件源。

> 线程在一个loop执行完后，若没有其他任务。将会进入休眠，此时需要其他事件源进行唤醒

# 2. __CFRunLoopSource

```c
struct __CFRunLoopSource {
    CFRuntimeBase _base;        // core foundation"对象"都是以这个开始
    uint32_t _bits;  			      //  基本保留位的位 1 用于信号状态
    pthread_mutex_t _lock;
    CFIndex _order;			        // 通用的顺序索引 
    CFMutableBagRef _runLoops;	// ref ---> 指向当前的loop
    union {	                    // 联合体类型的context版本
				CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```

## 2.1 __CFRuntimeBase

```c
typedef struct __CFRuntimeBase {
    uintptr_t _cfisa;   // 类型
    uint8_t _cfinfo[4]; // 表示 run loop 状态如：Sleeping/Deallocating 等等很多信息
#if __LP64__
    uint32_t _rc;       // 引用计数
#endif
} CFRuntimeBase;

struct __CFBoolean {
    CFRuntimeBase _base;
};
struct __CFString {
    CFRuntimeBase base;
    union {    // In many cases the allocated structs are smaller than these
        struct __inline1 {
    ...
};
// 会发现其实每个结构体中都会有 __CFRuntimeBase 对象
```

## 2.2 CFRunLoopSourceContext

```c
typedef struct {
  // 结构的版本号。必须为 0。
    CFIndex version;
  // 指向程序定义数据的任意指针，可以在创建时与 CFRunLoopSource 关联。
  // 这个指针被传递给上下文中定义的所有回调。
    void *  info;
  // 程序定义info指针的保留回调。
    const void *(*retain)(const void *info);
  // 程序定义info指针的释放回调。
    void    (*release)(const void *info);
  // 程序定义info指针的复制描述回调。
    CFStringRef (*copyDescription)(const void *info);
  // 程序定义info指针的相等测试回调。
    Boolean (*equal)(const void *info1, const void *info2);
  // 程序定义info指针的哈希计算回调。
    CFHashCode  (*hash)(const void *info);
    void    (*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
  // 运行循环源的取消回调。 当源从运行循环模式中删除时，将调用此回调。
    void    (*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
  // 运行循环源的执行回调。当源触发时调用此回调。
    void    (*perform)(void *info);
} CFRunLoopSourceContext;
```

在输入事件源`__CFRunLoopSource`的结构体中可以看到一个`version0`和一个`version1`。后面我们把`version0`版本的`source`称之为`source0`，反之为`source1`。
`version0`：简单理解就是自定义的source，我们可以自己定义一个`source0`。按钮点击，手势等都已经被苹果帮我们定义好了(这个我们后面讨论一下)， Cocoa 还定义了一个自定义输入源，允许在任何线程上执行选择器(selector)。就是`performSelector`系列方法。这里不列举了。
`version1`：这个是和内核相关了，基于`Mach por`t。其中需要到一个port，每个进程都有一个port，就是进程间通信需要用的。

说到这个就要说到一个点，就是我们知道屏幕触摸是硬件相关的，当我们点击屏幕时，此时并不知道是哪个APP的。因此呢这个时候内核先通过source1来这接收个硬件event，后面再分发到source0处理。因此如果追根溯源，点击时间终究还是source1。

```c
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(rls->_context.version0.perform, rls->_context.version0.info);

static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info) {
    if (perform) {
        perform(info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```



# 3. __CFRunLoopTimer

```c
// 仔细和__CFRunLoopSource对比就会发现有一些一样的成员
// 意义大多一样
struct __CFRunLoopTimer {
    CFRuntimeBase _base;					//	 说过了
    uint16_t _bits;								// 标志位
/* 基本保留位的位 0 用于触发状态 */
/* 基本保留位的第 1 位用于调用期间触发状态 */
/* 基本保留位的第 2 位用于唤醒状态 */
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;					// 记录下一次触发时间
    CFTimeInterval _interval;							//  理想时间间隔
    CFTimeInterval _tolerance;          	/* 执行的误差范围 */
    uint64_t _fireTSR;										/* Terminate and stay resident，缩写为TSR */
    CFIndex _order;												
    // 当 CFRunLoopTimer 对象触发时调用回调。
    CFRunLoopTimerCallBack _callout;			
    // 包含程序定义的数据和回调的结构，您可以使用它来配置 CFRunLoopTimer 的行为。
    CFRunLoopTimerContext _context;	
};
```

## 3.1 CFRunLoopTimerCallBack

```c
typedef void (*CFRunLoopTimerCallBack)(CFRunLoopTimerRef timer, void *info);
```

##  3.2 CFRunLoopTimerContext

```objective-c
typedef struct {
    CFIndex version;
    void *  info;
    const void *(*retain)(const void *info);
    void    (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
} CFRunLoopTimerContext;
```

可以通过如下函数使用 `CFRunLoopTimerContext`：

```c
// 这个函数符合 CFRunLoopTimerCallBack
static void _runLoopTimerWithBlockContext(CFRunLoopTimerRef timer, void *opaqueBlock) {
    typedef void (^timer_block_t) (CFRunLoopTimerRef timer);
    timer_block_t block = (timer_block_t)opaqueBlock;
    block(timer);
}

CFRunLoopTimerRef CFRunLoopTimerCreateWithHandler(CFAllocatorRef allocator, CFAbsoluteTime fireDate, CFTimeInterval interval, CFOptionFlags flags, CFIndex order,
                        void (^block) (CFRunLoopTimerRef timer)) {
    
    CFRunLoopTimerContext blockContext;
    blockContext.version = 0;
    blockContext.info = (void *)block;
    blockContext.retain = (const void *(*)(const void *info))_Block_copy;
    blockContext.release = (void (*)(const void *info))_Block_release;
    blockContext.copyDescription = NULL;
    return CFRunLoopTimerCreate(allocator, fireDate, interval, flags, order, _runLoopTimerWithBlockContext, &blockContext);
}
```

可自行阅读如下函数

```c
CFRunLoopTimerRef CFRunLoopTimerCreate(CFAllocatorRef allocator, CFAbsoluteTime fireDate, CFTimeInterval interval, CFOptionFlags flags, CFIndex order, CFRunLoopTimerCallBack callout, CFRunLoopTimerContext *context) {
    CHECK_FOR_FORK();
    if (isnan(interval)) {
        CRSetCrashLogMessage("NaN was used as an interval for a CFRunLoopTimer");
        HALT;
    }
    CFRunLoopTimerRef memory;
    UInt32 size;
    size = sizeof(struct __CFRunLoopTimer) - sizeof(CFRuntimeBase);
    memory = (CFRunLoopTimerRef)_CFRuntimeCreateInstance(allocator, CFRunLoopTimerGetTypeID(), size, NULL);
    if (NULL == memory) {
    return NULL;
    }
    __CFSetValid(memory);
    __CFRunLoopTimerUnsetFiring(memory);
    __CFRunLoopLockInit(&memory->_lock);
    memory->_runLoop = NULL;
    memory->_rlModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    memory->_order = order;
    if (interval < 0.0) interval = 0.0;
    memory->_interval = interval;
    memory->_tolerance = 0.0;
    if (TIMER_DATE_LIMIT < fireDate) fireDate = TIMER_DATE_LIMIT;
    memory->_nextFireDate = fireDate;
    memory->_fireTSR = 0ULL;
    uint64_t now2 = mach_absolute_time();
    CFAbsoluteTime now1 = CFAbsoluteTimeGetCurrent();
    if (fireDate < now1) {
    memory->_fireTSR = now2;
    } else if (TIMER_INTERVAL_LIMIT < fireDate - now1) {
    memory->_fireTSR = now2 + __CFTimeIntervalToTSR(TIMER_INTERVAL_LIMIT);
    } else {
    memory->_fireTSR = now2 + __CFTimeIntervalToTSR(fireDate - now1);
    }
    memory->_callout = callout;
    if (NULL != context) {
    if (context->retain) {
        memory->_context.info = (void *)context->retain(context->info);
    } else {
        memory->_context.info = context->info;
    }
    memory->_context.retain = context->retain;
    memory->_context.release = context->release;
    memory->_context.copyDescription = context->copyDescription;
    } else {
    memory->_context.info = 0;
    memory->_context.retain = 0;
    memory->_context.release = 0;
    memory->_context.copyDescription = 0;
    }
    return memory;
}
```

Timer 最后的回调会在这里

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(CFRunLoopTimerCallBack func, CFRunLoopTimerRef timer, void *info) {
    if (func) {
        func(timer, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}

// 示例
 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(rlt->_callout, rlt, context_info);
/*
	看到这里，其实会发现，真正的callBack其实就是 CFRunLoopTimerContext 中的 info
*/
```

