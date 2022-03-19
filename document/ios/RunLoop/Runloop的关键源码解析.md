> Runloop是iOS的一个老生常谈的话题，网上的资料也是很多，但是我想了解的关键地方总是一笔带过，因此自己看源码，进行了资料补充，以读者的身份完成这篇博文。欢迎讨论并以轻松的心情阅读本篇博文。
>
> Ps：网上大量存在且基础的内容不会出现在本文。结尾可能会尝试讨论NSTimer，autoreleasePool等常见面试题。
>
> [toc]

# 1. RunLoop

首先需要明白的是为什么需要RunLoop。

```c
void func() {
	printf("hello world\n");   
}
int main() {
   func();
  return 0;
}
```

上面这个程序，在线程执行完main函数后，就结束了，什么也没了。但是手机APP不行，手机APP需要时刻响应用户的操作（除非没电或者卡住）毕竟总不能打开主界面后，APP就结束了。因此我们需要让程序不结束，不结束的方案就是循环。

```c
do
{
}while(exit)
```

这里面的问题就是：不能一直空循环，因为空循环是对CPU的浪费，所以它需要在不执行任务的时候进行休眠，有任务时被唤醒。因此RunLoop的作用就是让我们能够拥有一个不浪费资源的常驻线程，即不会自动销毁的线程。

废话不多说，我们直接一步一步走进源码，NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。我们可看CFRunLoop源码，这里不介绍整个CFRunLoop.h，CFRunLoop.c的源码，主要是关键的部分

# 2. 数据结构

在介绍进入源码的时候，我们需要了解一些数据结构，这些数据结构虽然网络都有，但是我们总不能边搜（有些字段，网上不一定会介绍）边看本文吧。因此会把数据结构放在这里，这样我们忘记了某些结构的意义直接翻上来看也可以的，主要也是为了铺垫基础。下面列举的为Runloop Mode、Runloop Source，Runloop Observer、部分Runloop Call out等基础介绍（实际顺序为准，call out名字太长不好记，后面遇到再说）。我会尽量详细与尽量注释。另外，为了能过程流畅，因此其他一些结构体我放在附录中介绍，注释中会提到哪个在附录。

> **注：这里只是自己的理解，大概率存在错误与疏漏，望不吝指正。**

## 2.1 RunLoopSource

首先要说明的就是事件源，它是唤醒线程和线程任务关键之一，事件源有两种，一个是输入事件源，一个是定时器事件源。首先看看输入事件源。

### 2.1.1 __CFRunLoopSource

输入事件源__CFRunLoopSource，在结构体中可以看到一个version0和一个version1。后面我们把version0版本的source称之为source0，反之为source1。
**version0：**非基于Port的，只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
**version1：**基于Port的，包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。

说到这个就要说到一个点，就是我们知道屏幕触摸是硬件相关的，当我们点击屏幕时，此时并不知道是哪个APP的。因此呢内核先通过source1来这接收个硬件event，后面再分发到source0处理。所以如果追根溯源，点击等系统事件的可以说是source1。

```c
struct __CFRunLoopSource {
    CFRuntimeBase _base; // core foundation"对象"都是以这个开始，详情见附录
    uint32_t _bits;  			//  基本保留位的位 1 用于信号状态
    pthread_mutex_t _lock;
    CFIndex _order;			/* 通用的顺序索引 */
    CFMutableBagRef _runLoops;	// ref ---> 指向当前的loop
    union {	// 这是个联合体，又是一个小细节，
				CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
// CFRunLoopSourceContext
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

事件源的回调有一个名字特别长的一个函数，其内部其实就是调用了这个perform回调，如下：

> 这是我们见到的第一个回调，如触发点击事件，当你打断点的时候，就会出现它的身影
>
> __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__

```c++
// 示例
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(rls->_context.version0.perform, rls->_context.version0.info);

static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info) {
    if (perform) {
        perform(info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

### 2.1.2 创建__CFRunLoopSource

__CFRunLoopSource的基本数据结构已经清晰了，下面是输入事件源的一个创建过程。

```c++
CFRunLoopSourceRef CFRunLoopSourceCreate(CFAllocatorRef allocator, CFIndex order, CFRunLoopSourceContext *context) {
    CHECK_FOR_FORK(); // iOS App 只有一个进程，因此这个宏可以忽略
    CFRunLoopSourceRef memory; // 创建了一个指针
    uint32_t size;
    if (NULL == context) CRASH("*** NULL context value passed to CFRunLoopSourceCreate(). (%d) ***", -1);
    // 开辟内存
    // _CFRuntimeCreateInstanc方法里面会将sizeof(CFRuntimeBase)再加上
    // 因此所有使用_CFRuntimeCreateInstanc创建的内存都会先减去sizeof(CFRuntimeBase)这个大小
    size = sizeof(struct __CFRunLoopSource) - sizeof(CFRuntimeBase);
    memory = (CFRunLoopSourceRef)_CFRuntimeCreateInstance(allocator, CFRunLoopSourceGetTypeID(), size, NULL);
    if (NULL == memory) {
    return NULL;
    }
    __CFSetValid(memory);
    __CFRunLoopSourceUnsetSignaled(memory);
    __CFRunLoopLockInit(&memory->_lock);
    memory->_bits = 0;
    memory->_order = order;
    memory->_runLoops = NULL;
    size = 0;
    switch (context->version) {
    case 0:
    size = sizeof(CFRunLoopSourceContext);
    break;
    case 1:
    size = sizeof(CFRunLoopSourceContext1);
    break;
    }
    // 拷贝一份
    objc_memmove_collectable(&memory->_context, context, size);
    if (context->retain) {
    memory->_context.version0.info = (void *)context->retain(context->info);
    }
    return memory;
}
```

### 2.1.3 __CFRunLoopTimer

定时器事件的结构体其实和source结构体定义的差不多，我们先看看其定义和回调。

```c
// 仔细和__CFRunLoopSource对比就会发现有一些一样的成员
// 意义大多一样
struct __CFRunLoopTimer {
    CFRuntimeBase _base;				
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

typedef struct {
    CFIndex	version;
    void *	info;	
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopTimerContext;
```

这里就会看到第二个名字特别长的一个函数。

> __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
>
> 调用timer

其内部的实现是这样。

```c++
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(CFRunLoopTimerCallBack func, CFRunLoopTimerRef timer, void *info) {
    if (func) {
        func(timer, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}

// 示例
 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(rlt->_callout, rlt, context_info);
/*
	同样的，info就是附带的信息参数
*/
```

### 2.1.4 创建__CFRunLoopTimer

对于定时器事件源的创建略微比输入事件源繁琐。

```c++
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
    // 这里是倒计时的时间设置
    // mach_absolute_time 获取基于系统启动后的时钟嘀嗒数，是一个CPU/总线依赖函数，精度纳秒级
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

// 下面是一个timer创建的一个函数，作为示例
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

## 2.2 Runloop Observer

Observer 是非常重要的。Runloop对外的回调都考Observe通知。

观察下面的结构体，仿佛是上面的timer的结构体搬过来的。其中_activities代表观察当前线程在RunLoop中所表示的状态。如果在RunLoop中到达该状态就会发起相应通知，调用回调。

### 2.2.1 __CFRunLoopObserver

```c
typedef struct __CFRunLoopObserver * CFRunLoopObserverRef;
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* RunLoop的状态 */
    CFIndex _order;			
    CFRunLoopObserverCallBack _callout;	
    CFRunLoopObserverContext _context;	
};
```

CFOptionFlags就是改状态的类型，代表着线程在runloop中的表示状态

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0), //即将进入run loop
    kCFRunLoopBeforeTimers = (1UL << 1), //即将处理timer
    kCFRunLoopBeforeSources = (1UL << 2),//即将处理source
    kCFRunLoopBeforeWaiting = (1UL << 5),//即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),//被唤醒但是还没开始处理事件
    kCFRunLoopExit = (1UL << 7),//run loop已经退出
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

这个状态位是怎么使用的呢？在后面的代码中可以看到，它是通过位与的方式来确定当前在何种状态。

```c++
// 截取其中一段代码可以看到
// CFRunLoopActivity内部的枚举都是通过位移，因此我们通过RunloopMode中的掩码进行位与的时候可以直接确定当前线程在runloop的状态
if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
```

下面是观察者的一个宏，这是第三个长名字的回调了。

```c
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__

  static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(CFRunLoopObserverCallBack func, CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (func) {
        func(observer, activity, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

### 2.2.2 创建__CFRunLoopObserver

observe的创建方式和前面的大同小异。

```c++
CFRunLoopObserverRef CFRunLoopObserverCreate(CFAllocatorRef allocator, CFOptionFlags activities, Boolean repeats, CFIndex order, CFRunLoopObserverCallBack callout, CFRunLoopObserverContext *context) {
    CHECK_FOR_FORK();
    CFRunLoopObserverRef memory;
    UInt32 size;
    size = sizeof(struct __CFRunLoopObserver) - sizeof(CFRuntimeBase);
    memory = (CFRunLoopObserverRef)_CFRuntimeCreateInstance(allocator, CFRunLoopObserverGetTypeID(), size, NULL);
    if (NULL == memory) {
    return NULL;
    }
    __CFSetValid(memory);
    __CFRunLoopObserverUnsetFiring(memory);
    if (repeats) {
    __CFRunLoopObserverSetRepeats(memory);
    } else {
    __CFRunLoopObserverUnsetRepeats(memory);
    }
    __CFRunLoopLockInit(&memory->_lock);
    memory->_runLoop = NULL;
    memory->_rlCount = 0;
    memory->_activities = activities;
    memory->_order = order;
    memory->_callout = callout;
    if (context) {
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

### 2.2.3 RunLoopObserver应用

苹果具体的配置在附录中。

- AutoreleasePool

App 启动后，苹果在主线程 RunLoop 里注册了下面的 Observer：

```
即将进入 Loop => 调用 _objc_autoreleasePoolPush() 创建自动释放池
即将进入休眠 => 调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池
即将退出 => 调用 _objc_autoreleasePoolPop() 来释放自动释放池
```

- 事件响应

苹果注册了一个 **Source1** 来接收触摸、加速、传感器等系统事件，随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

- 手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断，随后系统将对应的 UIGestureRecognizer 标记为待处理，苹果注册了一个 Observer 监测 **即将进入休眠**，其回调函数会获取所有刚被标记为待处理的 UIGestureRecognizer，并执行UIGestureRecognizer 的回调，当有 UIGestureRecognizer 的状态变化时，这个回调都会进行相应处理。

- 界面更新

当在操作 UI 时，这个 UIView 或 CALayer 就被标记为待处理，并被提交到一个全局的容器去，苹果注册了一个 Observer 监测 **即将进入休眠** 和 **即将退出**，回调_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv，这个函数里会遍历所有待处理的 UIView 或 CALayer 以执行实际的绘制和调整，并更新 UI 界面。

- 关于 GCD

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE() 里执行这个 block，也就是对应 handle_msg 处理消息：如果 dispatch 就执行 block 。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE是第四个长名字的回调。

- PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会 创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。当调用 performSelector:onThread: 时，实际上其会 创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

## 2.3 RunLoop Mode

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

### 2.3.1 __CFRunLoopMode

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;		
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this 锁mode前，先锁定RunLoop*/
    CFStringRef _name;			 
    Boolean _stopped;
    char _padding[3];、
     // 	核心了啊， 注意看了
    CFMutableSetRef _sources0;	// 事件源set，就是version为0的__CFRunLoopSource
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;	// 观察者array
    CFMutableArrayRef _timers;		// 计时器array
    CFMutableDictionaryRef _portToV1SourceMap;	// port和source1对应
    __CFPortSet _portSet;				// 端口set
    CFIndex _observerMask;			// 这个字段是为了当前mode和匹配观察者状态。
  /*
  	每添加一个观察者，就会
  	rlm->_observerMask |= rlo->_activities; （源代码）
  	这样就保证了这个字段有所有观察者当前的状态
  	然后在通知观察者时
  	rlm->_observerMask & xxxx
  	就可以判断当前mode是否有对应状态的观察者，然后就会进行通知
  	
  	主要是还没有代入环境，后面源码介绍的时候还会介绍。
  */
  
  /*
  	下面两个宏，是这样定义的
  	#if DEPLOYMENT_TARGET_MACOSX		
		#define USE_DISPATCH_SOURCE_FOR_TIMERS 1
		#define USE_MK_TIMER_TOO 1
		#else			
		#define USE_DISPATCH_SOURCE_FOR_TIMERS 0
		#define USE_MK_TIMER_TOO 1
		#endif
  */
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO	// 那这里就是iOS要走的
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS		// 是否是Windows环境
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

### 2.3.2 创建__CFRunLoopFindMode

下面这个方法的第三个参数就是代表着没找到是否创建。首先先看一下mode和runloop类的一个注册。后面遇到的TypeID就是这个类对象在注册表中的下标。

```c++
CFTypeID CFRunLoopGetTypeID(void) {
    static dispatch_once_t initOnce = 0;
    dispatch_once(&initOnce, ^{ __kCFRunLoopTypeID = _CFRuntimeRegisterClass(&__CFRunLoopClass); __kCFRunLoopModeTypeID = _CFRuntimeRegisterClass(&__CFRunLoopModeClass); });
    return __kCFRunLoopTypeID;
}
```

具体创建如下

```c++
static CFRunLoopModeRef __CFRunLoopFindMode(CFRunLoopRef rl, CFStringRef modeName, Boolean create) {
    CHECK_FOR_FORK();
    CFRunLoopModeRef rlm;
    struct __CFRunLoopMode srlm;
    memset(&srlm, 0, sizeof(srlm));
    _CFRuntimeSetInstanceTypeIDAndIsa(&srlm, __kCFRunLoopModeTypeID);
    srlm._name = modeName;
    // 这里传的是srlm，是使用的srlm的modeName，因此新建的也无所谓
    rlm = (CFRunLoopModeRef)CFSetGetValue(rl->_modes, &srlm);
    if (NULL != rlm) {
    __CFRunLoopModeLock(rlm);
    return rlm;
    }
    if (!create) {
    return NULL;
    }
    rlm = (CFRunLoopModeRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, __kCFRunLoopModeTypeID, sizeof(struct __CFRunLoopMode) - sizeof(CFRuntimeBase), NULL);
    if (NULL == rlm) {
    return NULL;
    }
    __CFRunLoopLockInit(&rlm->_lock);
    rlm->_name = CFStringCreateCopy(kCFAllocatorSystemDefault, modeName);
    rlm->_stopped = false;
    rlm->_portToV1SourceMap = NULL;
    rlm->_sources0 = NULL;
    rlm->_sources1 = NULL;
    rlm->_observers = NULL;
    rlm->_timers = NULL;
    rlm->_observerMask = 0;
    rlm->_portSet = __CFPortSetAllocate();
    rlm->_timerSoftDeadline = UINT64_MAX;
    rlm->_timerHardDeadline = UINT64_MAX;
    kern_return_t ret = KERN_SUCCESS;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    rlm->_timerFired = false;
    rlm->_queue = _dispatch_runloop_root_queue_create_4CF("Run Loop Mode Queue", 0);
    mach_port_t queuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
    if (queuePort == MACH_PORT_NULL) CRASH("*** Unable to create run loop mode queue port. (%d) ***", -1);
    rlm->_timerSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, rlm->_queue);
    __block Boolean *timerFiredPointer = &(rlm->_timerFired);
    dispatch_source_set_event_handler(rlm->_timerSource, ^{
        *timerFiredPointer = true;
    });
    _dispatch_source_set_runloop_timer_4CF(rlm->_timerSource, DISPATCH_TIME_FOREVER, DISPATCH_TIME_FOREVER, 321);
    dispatch_resume(rlm->_timerSource);
    
    ret = __CFPortSetInsert(queuePort, rlm->_portSet);
    if (KERN_SUCCESS != ret) CRASH("*** Unable to insert timer port into port set. (%d) ***", ret);
    
#endif
#if USE_MK_TIMER_TOO
    rlm->_timerPort = mk_timer_create();
    ret = __CFPortSetInsert(rlm->_timerPort, rlm->_portSet);
    if (KERN_SUCCESS != ret) CRASH("*** Unable to insert timer port into port set. (%d) ***", ret);
#endif
    ret = __CFPortSetInsert(rl->_wakeUpPort, rlm->_portSet);
    if (KERN_SUCCESS != ret) CRASH("*** Unable to insert wake up port into port set. (%d) ***", ret);
    CFSetAddValue(rl->_modes, rlm);
    CFRelease(rlm);
    __CFRunLoopModeLock(rlm);   /* return mode locked */
    return rlm;
}
```

## 2.4 RunLoop

越过千山万水，终于到RunLoop了。

### 2.4.1 __CFRunLoop

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;				// 唤醒loop的端口 【关键】
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
		/*线程和RunLoop是一一对应的，这个我们在附录解析吧*/
  	pthread_t _pthread;					// 对应的线程
    uint32_t _winthread;				// 对应的Windows的线程
    CFMutableSetRef _commonModes;					// 带有common属性的mode集合
    CFMutableSetRef _commonModeItems;			// 带有common属性的source
    CFRunLoopModeRef _currentMode;				// 当前运行的mode
    CFMutableSetRef _modes;								// mode的集合
    
  	/*
  			链表头指针，该链表保存了所有需要被 run loop 执行的 block。
      	外部通过调用 CFRunLoopPerformBlock 函数来向链表中添加一个 block 节点。
    		run loop 会在 CFRunLoopDoBlock 时遍历该链表，逐一执行 block。
  	*/
  	struct _block_item *_blocks_head;
  	// 链表尾结点，用于插入block
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;			// 运行时间
    CFAbsoluteTime _sleepTime;		// 睡眠时间
    CFTypeRef _counterpart;			
};
```

### 2.4.2 创建__CFRunLoop

```c++
static CFRunLoopRef __CFRunLoopCreate(pthread_t t) {
    CFRunLoopRef loop = NULL;
    CFRunLoopModeRef rlm;
    uint32_t size = sizeof(struct __CFRunLoop) - sizeof(CFRuntimeBase);
    loop = (CFRunLoopRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, CFRunLoopGetTypeID(), size, NULL);
    if (NULL == loop) {
    return NULL;
    }
    (void)__CFRunLoopPushPerRunData(loop);
    __CFRunLoopLockInit(&loop->_lock);
    loop->_wakeUpPort = __CFPortAllocate();
    if (CFPORT_NULL == loop->_wakeUpPort) HALT;
    __CFRunLoopSetIgnoreWakeUps(loop);
    loop->_commonModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    CFSetAddValue(loop->_commonModes, kCFRunLoopDefaultMode);
    loop->_commonModeItems = NULL;
    loop->_currentMode = NULL;
    loop->_modes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    loop->_blocks_head = NULL;
    loop->_blocks_tail = NULL;
    loop->_counterpart = NULL;
    loop->_pthread = t;
#if DEPLOYMENT_TARGET_WINDOWS
    loop->_winthread = GetCurrentThreadId();
#else
    loop->_winthread = 0;
#endif
    rlm = __CFRunLoopFindMode(loop, kCFRunLoopDefaultMode, true);
    if (NULL != rlm) __CFRunLoopModeUnlock(rlm);
    return loop;
}
```

# 3. RunLoop主逻辑

现在正式进入RunLoop，对于关键代码进行注释说明，会剔除部分五官代码。

## 3.1 CFRunLoopRun

```c
// 默认的
// 可以看到基本的循环就是
// do {
//
// } while(条件)
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
// 这个没仔细看，我理解应该是切换mode的时候调用
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```

## 3.2 CFRunLoopRunSpecific

为了充分理解上面的代码，我们把这个CFRunLoopRunSpecific带出来；

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
		// 这个did不知道是干嘛的
    Boolean did = false;
	if (currentMode) __CFRunLoopModeUnlock(currentMode);
	__CFRunLoopUnlock(rl);
	return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
	// 这里就能看到观察者的使用方式了
  // 进入runloop
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
  
  // 
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
  
  
  // 退出runloop
  if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

### 3.2.1 __CFRunLoopRun

__CFRunLoopRun是一个巨长的函数，详细代码我放在附录吧，这里我用注释代替一些代码。

```c++
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

精简部分代码

```c++
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();

    /*
   		这里有一段优化的代码 
   		检查loop是否停止
    */
    /*
  		 这里有一段关于GCD消息端口的设置
    */
    /*
   		这里有一段设置超时时间的代码 
    */

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
#endif
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#elif DEPLOYMENT_TARGET_LINUX
        int livePort = -1;
#endif
    __CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);
// 进行通知
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
// 执行添加的block
// 这个方法的内部，就是遍历_blocks_head这个链表
// 然后执行
// 其中的block是通过 CFRunLoopPerformBlock 添加的
// 这个方法就是 perfom 系列方法的底层
// 官方说 This method enqueues the block only and does not automatically wake up the specified run loop
// 因此有多次执行 __CFRunLoopDoBlocks
    __CFRunLoopDoBlocks(rl, rlm);

        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
          // 执行添加的block
            __CFRunLoopDoBlocks(rl, rlm);
    }
// 这里是否开始轮询，否则要休眠了
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
// 这里是基于port的source1事件，采用goto语法
#if __HAS_DISPATCH__
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_LINUX
            if (__CFRunLoopServiceFileDescriptors(CFPORTSET_NULL, dispatchPort, 0, &livePort)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }
#endif

        didDispatchPortLastTime = false;
// 即将休眠
    if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
    __CFRunLoopSetSleeping(rl);
    // do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

#if __HAS_DISPATCH__
        __CFPortSetInsert(dispatchPort, waitSet);
#endif
        
    __CFRunLoopModeUnlock(rlm);
    __CFRunLoopUnlock(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
      // 调用 mach_msg 等待接受 mach_port 的消息
      // 等待被唤醒
        do {
            if (kCFUseCollectableAllocator) {
                // objc_clear_stack(0);
                // <rdar://problem/16393959>
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator) {
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        
        
#elif DEPLOYMENT_TARGET_WINDOWS
// ………………代码省略
#elif DEPLOYMENT_TARGET_LINUX
// ………………代码省略
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

#if __HAS_DISPATCH__
        __CFPortSetRemove(dispatchPort, waitSet);
#endif
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
    __CFRunLoopUnsetSleeping(rl);
    // CFRunLoopAfterWaiting
    // 循环唤醒之后的事件处理循环内，但在处理唤醒它的事件之前。 仅当运行循环在当前循环期间确实进入睡眠状态时才会发生此活动。
    // kCFRunLoopAfterWaiting
    if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
// source1 事件源
        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);

#if DEPLOYMENT_TARGET_WINDOWS
// ………………代码省略
#endif
      // 啥也不是
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
      // 啥也不是
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
      // 这里是dispatch的timer
      // 如：dispatch_after
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
      // 基于mach的timer
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if __HAS_DISPATCH__
      // dispatch_main
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
#if DEPLOYMENT_TARGET_WINDOWS
            void *msg = 0;
#endif
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        }
#endif
        else {
          // source1 事件源唤醒
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *reply = NULL;
        sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
        if (NULL != reply) {
            (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
            CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
        }
#elif DEPLOYMENT_TARGET_WINDOWS || DEPLOYMENT_TARGET_LINUX
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
        }
            
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
    __CFRunLoopDoBlocks(rl, rlm);
        
// 结束的判定
    if (sourceHandledThisLoop && stopAfterHandle) {
        retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
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

#if __HAS_DISPATCH__
    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else
#endif
    {
        free(timeout_context);
    }

    return retVal;
}

// 打完收工
```

# 附录：

### __CFRuntimeBase

```C
typedef struct __CFRuntimeBase {
    uintptr_t _cfisa; // 类型
    uint8_t _cfinfo[4]; // 表示 run loop 状态如：Sleeping/Deallocating 等等很多信息
#if __LP64__
    uint32_t _rc; // 引用计数
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
```

### _CFRuntimeCreateInstance

```c++
CFTypeRef _CFRuntimeCreateInstance(CFAllocatorRef allocator, CFTypeID typeID, CFIndex extraBytes, unsigned char *category) {
    if (__CFRuntimeClassTableSize <= typeID) HALT;
    CFAssert1(typeID != _kCFRuntimeNotATypeID, __kCFLogAssertion, "%s(): Uninitialized type id", __PRETTY_FUNCTION__);
    CFRuntimeClass *cls = __CFRuntimeClassTable[typeID];
    if (NULL == cls) {
	return NULL;
    }
    if (cls->version & _kCFRuntimeRequiresAlignment) {
        allocator = kCFAllocatorSystemDefault;
    }
    Boolean customRC = !!(cls->version & _kCFRuntimeCustomRefCount);
    if (customRC && !cls->refcount) {
        CFLog(kCFLogLevelWarning, CFSTR("*** _CFRuntimeCreateInstance() found inconsistent class '%s'."), cls->className);
        return NULL;
    }
    CFAllocatorRef realAllocator = (NULL == allocator) ? __CFGetDefaultAllocator() : allocator;
    if (kCFAllocatorNull == realAllocator) {
	return NULL;
    }
    Boolean usesSystemDefaultAllocator = _CFAllocatorIsSystemDefault(realAllocator);
    size_t align = (cls->version & _kCFRuntimeRequiresAlignment) ? cls->requiredAlignment : 16;
    CFIndex size = sizeof(CFRuntimeBase) + extraBytes + (usesSystemDefaultAllocator ? 0 : sizeof(CFAllocatorRef));
    size = (size + 0xF) & ~0xF;	// CF objects are multiples of 16 in size
    // CFType version 0 objects are unscanned by default since they don't have write-barriers and hard retain their innards
    // CFType version 1 objects are scanned and use hand coded write-barriers to store collectable storage within
    CFRuntimeBase *memory = NULL;
    if (cls->version & _kCFRuntimeRequiresAlignment) {
        memory = malloc_zone_memalign(malloc_default_zone(), align, size);
    } else {
        memory = (CFRuntimeBase *)CFAllocatorAllocate(allocator, size, CF_GET_COLLECTABLE_MEMORY_TYPE(cls));
    }
    if (NULL == memory) {
	return NULL;
    }
    if (!kCFUseCollectableAllocator || !CF_IS_COLLECTABLE_ALLOCATOR(allocator) || !(CF_GET_COLLECTABLE_MEMORY_TYPE(cls) & __kCFAllocatorGCScannedMemory)) {
	memset(memory, 0, size);
    }
    if (__CFOASafe && category) {
	__CFSetLastAllocationEventName(memory, (char *)category);
    } else if (__CFOASafe) {
	__CFSetLastAllocationEventName(memory, (char *)cls->className);
    }
    if (!usesSystemDefaultAllocator) {
        // add space to hold allocator ref for non-standard allocators.
        // (this screws up 8 byte alignment but seems to work)
	*(CFAllocatorRef *)((char *)memory) = (CFAllocatorRef)CFRetain(realAllocator);
	memory = (CFRuntimeBase *)((char *)memory + sizeof(CFAllocatorRef));
    }
    uint32_t rc = 0;
#if __LP64__
    if (!kCFUseCollectableAllocator || (1 && 1)) {
        memory->_rc = 1;
    }
    if (customRC) {
        memory->_rc = 0xFFFFFFFFU;
        rc = 0xFF;
    }
#else
    if (!kCFUseCollectableAllocator || (1 && 1)) {
        rc = 1;
    }
    if (customRC) {
        rc = 0xFF;
    }
#endif
    uint32_t *cfinfop = (uint32_t *)&(memory->_cfinfo);
    *cfinfop = (uint32_t)((rc << 24) | (customRC ? 0x800000 : 0x0) | ((uint32_t)typeID << 8) | (usesSystemDefaultAllocator ? 0x80 : 0x00));
    memory->_cfisa = 0;
    if (NULL != cls->init) {
	(cls->init)(memory);
    }
    return memory;
}
```

### apple官方Runloop

```c
CFRunLoop {
    current mode = kCFRunLoopDefaultMode
    common modes = {
        UITrackingRunLoopMode
        kCFRunLoopDefaultMode
    }
 
    common mode items = {
 
        // source0 (manual)
        CFRunLoopSource {order =-1, {
            callout = _UIApplicationHandleEventQueue}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventSignalCallback }}
        CFRunLoopSource {order = 0, {
            callout = FBSSerialQueueRunLoopSourceHandler}}
 
        // source1 (mach port)
        CFRunLoopSource {order = 0,  {port = 17923}}
        CFRunLoopSource {order = 0,  {port = 12039}}
        CFRunLoopSource {order = 0,  {port = 16647}}
        CFRunLoopSource {order =-1, {
            callout = PurpleEventCallback}}
        CFRunLoopSource {order = 0, {port = 2407,
            callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_}}
        CFRunLoopSource {order = 0, {port = 1c03,
            callout = __IOHIDEventSystemClientAvailabilityCallback}}
        CFRunLoopSource {order = 0, {port = 1b03,
             // 系统事件
            callout = __IOHIDEventSystemClientQueueCallback}}
        CFRunLoopSource {order = 1, {port = 1903,
            callout = __IOMIGMachPortPortCallback}}
 
        // Ovserver
        CFRunLoopObserver {order = -2147483647, activities = 0x1, // Entry
           // AutoreleasePool的通知
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
        CFRunLoopObserver {order = 0, activities = 0x20,          // BeforeWaiting
            callout = _UIGestureRecognizerUpdateObserver}
        CFRunLoopObserver {order = 1999000, activities = 0xa0,    // BeforeWaiting | Exit
            callout = _afterCACommitHandler}
        CFRunLoopObserver {order = 2000000, activities = 0xa0,    // BeforeWaiting | Exit
           // 界面更新
            callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
        CFRunLoopObserver {order = 2147483647, activities = 0xa0, // BeforeWaiting | Exit
           // AutoreleasePool的通知
            callout = _wrapRunLoopWithAutoreleasePoolHandler}
 
        // Timer
        CFRunLoopTimer {firing = No, interval = 3.1536e+09, tolerance = 0,
            next fire date = 453098071 (-4421.76019 @ 96223387169499),
            callout = _ZN2CAL14timer_callbackEP16__CFRunLoopTimerPv (QuartzCore.framework)}
    },
 
    modes ＝ {
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 =  { /* same as 'common mode items' */ },
            sources1 =  { /* same as 'common mode items' */ },
            observers = { /* same as 'common mode items' */ },
            timers =    { /* same as 'common mode items' */ },
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = 0, {
                    callout = FBSSerialQueueRunLoopSourceHandler}}
            },
            sources1 = (null),
            observers = {
                CFRunLoopObserver >{activities = 0xa0, order = 2000000,
                    callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}
            )},
            timers = (null),
        },
 
        CFRunLoopMode  {
            sources0 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventSignalCallback}}
            },
            sources1 = {
                CFRunLoopSource {order = -1, {
                    callout = PurpleEventCallback}}
            },
            observers = (null),
            timers = (null),
        },
        
        CFRunLoopMode  {
            sources0 = (null),
            sources1 = (null),
            observers = (null),
            timers = (null),
        }
    }
}
```

https://juejin.cn/post/6907543033751961607

http://joakimliu.github.io/2019/03/09/runloop-note/

https://www.cnblogs.com/kenshincui/p/6823841.html

autoreleaspoll

https://www.jianshu.com/p/0fe8ed6f37ae

https://www.jianshu.com/p/2d9071465a73