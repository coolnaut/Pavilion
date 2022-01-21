> Runloop是iOS的一个老生常谈的话题，网上的资料也是很多，但是我想了解的关键地方总是一笔带过，因此自己看源码，进行了资料补充，以读者的身份完成这篇博文。欢迎以轻松的心情阅读本篇博文。
>
> Ps：网上大量存在且基础的内容不会出现在本文。结尾可能会尝试讨论NSTimer，autoreleasePool等常见面试题。

### 1. RunLoop

首先需要明白的是为什么需要RunLoop。

```c
void func1() {
  
}
void func() {
   func1();
}
int main() {
   func();
  return 0;
}
```

上面这个程序，在线程执行完main函数后，就结束了，什么也没了。但是手机APP不行，手机APP需要响应用户的操作，总不能打开主界面后，APP就结束了。因此我们需要让程序不结束，不结束的方案就是循环。

```c
do
{
}while(exit)
```

这里面的问题就是：不能一直空循环，因为空循环是对CPU的浪费。

废话不多说，我们直接一步一步走进源码，NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。我们可看CFRunLoop源码，这里不介绍整个CFRunLoop.h，CFRunLoop.c的源码，主要是关键的部分

### 2. 前提须知

在介绍进入源码的时候，我们需要了解一些数据结构，这些数据结构虽然网络都有，但是我总不能边搜（有些字段，网上不一定会介绍）边看本文吧。因此会把数据结构放在这里，这样我们忘记了某些结构的意义直接翻上来看也可以的，主要也是为了铺垫基础。下面列举的为Runloop Mode、Runloop Source，Runloop Observer、部分Runloop Call out等基础介绍（实际顺序为准，call out名字太长不好记，后面遇到再说）。我会尽量详细与尽量注释。另外，为了能过程流畅，因此其他一些结构体我放在附录中介绍，注释中会提到哪个在附录。

另外：这里只是自己的理解，大概率存在错误与疏漏，望不吝指正。

#### 2.1 RunLoopSource

事件源，它是唤醒线程和线程任务关键之一，事件源有两种，一个是输入事件源，一个是定时器事件源。首先看看输入事件源。

##### 2.1.1 __CFRunLoopSource

输入事件源__CFRunLoopSource，在结构体中可以看到一个version0和一个version1。后面我们把version0版本的source称之为source0，反之为source1。
version0：简单理解就是自定义的source，我们可以自己定义一个source0。我们的按钮点击，手势等都已经被苹果帮我们定义好了(这个我们后面讨论一下)， Cocoa 还定义了一个自定义输入源，允许在任何线程上执行选择器(selector)。就我们performSelector系列方法。这里不列举了。
version1：这个是和内核相关了，基于Mach port。其中需要到一个port，每个进程都有一个port，就是进程间通信需要用的。

说到这个就要说到一个点，就是我们知道屏幕触摸是硬件相关的，当我们点击屏幕时，此时并不知道是哪个APP的。因此呢这个时候内核先通过source1来这接收个硬件event，后面再分发到source0处理。因此如果追根溯源，点击啥的其实可以说是source1。

```c
struct __CFRunLoopSource {
    CFRuntimeBase _base; // core foundation"对象"都是以这个开始，详情见附录
    uint32_t _bits;  			//  基本保留位的位 1 用于信号状态
    pthread_mutex_t _lock;
    CFIndex _order;			/* 通用的顺序索引 */
    CFMutableBagRef _runLoops;	// ref ---> 指向当前的loop
    union {	// 这是个联合体，细节，哈哈
				CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```

##### 2.2.2 __CFRunLoopTimer

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

typedef struct {
    CFIndex	version;
    void *	info;														// 执行的回调
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopTimerContext;
```

#### 2.2. Runloop Observer

Observer 是非常重要的。观察下面的结构体，感觉像是上面的timer的结构体搬过来的，这样也可以猜到有一部分和timer是一样的，就是它的回调。其中_activities是observer独有的，代表观察的RunLoop的状态。如果RunLoop的状态为这个状态就会发起通知，调用回调。

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

看看这个CFOptionFlags哈

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

观察者也有一个宏

```c
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
// 和timer的是一样的，哈哈，你可以看看附录的实现。
  static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(CFRunLoopObserverCallBack func, CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (func) {
        func(observer, activity, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```



#### 2.3 RunLoop Mode

前面的几个结构介绍完了，下面看看这回mode

```c
// 带Ref的都是一个该类型的指针
// 比如CFRunLoopModeRef 是__CFRunLoopMode *、 CFRunLoopObserverRef是__CFRunLoopObserver *
// 后面遇到不再写出来了
// ref?? reference
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

#### 2.4 __CFRunLoop

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

### 3. 进入RunLoop

现在正是进入RunLoop的循环开始。入口就是下面两个：

```c
// 默认的
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

为了充分理解上面的代码，我们把这个CFRunLoopRunSpecific带出来；

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
	Boolean did = false;
	if (currentMode) __CFRunLoopModeUnlock(currentMode);
	__CFRunLoopUnlock(rl);
	return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

下面是一个巨长的函数__CFRunLoopRun，详细代码我放在附录吧，这里我用注释代替一些代码：

```c
static Boolean __CFRunLoopServiceMachPort(mach_port_name_t port, mach_msg_header_t **buffer, size_t buffer_size, mach_port_t *livePort, mach_msg_timeout_t timeout, voucher_mach_msg_state_t *voucherState, voucher_t *voucherCopy) {
    Boolean originalBuffer = true;
    kern_return_t ret = KERN_SUCCESS;
    for (;;) {		/* In that sleep of death what nightmares may come ... */
        mach_msg_header_t *msg = (mach_msg_header_t *)*buffer;
        msg->msgh_bits = 0;
        msg->msgh_local_port = port;
        msg->msgh_remote_port = MACH_PORT_NULL;
        msg->msgh_size = buffer_size;
        msg->msgh_id = 0;
        if (TIMEOUT_INFINITY == timeout) { CFRUNLOOP_SLEEP(); } else { CFRUNLOOP_POLL(); }
        ret = mach_msg(msg, MACH_RCV_MSG|(voucherState ? MACH_RCV_VOUCHER : 0)|MACH_RCV_LARGE|((TIMEOUT_INFINITY != timeout) ? MACH_RCV_TIMEOUT : 0)|MACH_RCV_TRAILER_TYPE(MACH_MSG_TRAILER_FORMAT_0)|MACH_RCV_TRAILER_ELEMENTS(MACH_RCV_TRAILER_AV), 0, msg->msgh_size, port, timeout, MACH_PORT_NULL);

        // Take care of all voucher-related work right after mach_msg.
        // If we don't release the previous voucher we're going to leak it.
        voucher_mach_msg_revert(*voucherState);
        
        // Someone will be responsible for calling voucher_mach_msg_revert. This call makes the received voucher the current one.
        *voucherState = voucher_mach_msg_adopt(msg);
        
        if (voucherCopy) {
            if (*voucherState != VOUCHER_MACH_MSG_STATE_UNCHANGED) {
                // Caller requested a copy of the voucher at this point. By doing this right next to mach_msg we make sure that no voucher has been set in between the return of mach_msg and the use of the voucher copy.
                // CFMachPortBoost uses the voucher to drop importance explicitly. However, we want to make sure we only drop importance for a new voucher (not unchanged), so we only set the TSD when the voucher is not state_unchanged.
                *voucherCopy = voucher_copy();
            } else {
                *voucherCopy = NULL;
            }
        }

        CFRUNLOOP_WAKEUP(ret);
        if (MACH_MSG_SUCCESS == ret) {
            *livePort = msg ? msg->msgh_local_port : MACH_PORT_NULL;
            return true;
        }
        if (MACH_RCV_TIMED_OUT == ret) {
            if (!originalBuffer) free(msg);
            *buffer = NULL;
            *livePort = MACH_PORT_NULL;
            return false;
        }
        if (MACH_RCV_TOO_LARGE != ret) break;
        buffer_size = round_msg(msg->msgh_size + MAX_TRAILER_SIZE);
        if (originalBuffer) *buffer = NULL;
        originalBuffer = false;
        *buffer = realloc(*buffer, buffer_size);
    }
    HALT;
    return false;
}

#elif DEPLOYMENT_TARGET_WINDOWS

#define TIMEOUT_INFINITY INFINITE

// pass in either a portSet or onePort
static Boolean __CFRunLoopWaitForMultipleObjects(__CFPortSet portSet, HANDLE *onePort, DWORD timeout, DWORD mask, HANDLE *livePort, Boolean *msgReceived) {
    DWORD waitResult = WAIT_TIMEOUT;
    HANDLE handleBuf[MAXIMUM_WAIT_OBJECTS];
    HANDLE *handles = NULL;
    uint32_t handleCount = 0;
    Boolean freeHandles = false;
    Boolean result = false;
    
    if (portSet) {
	// copy out the handles to be safe from other threads at work
	handles = __CFPortSetGetPorts(portSet, handleBuf, MAXIMUM_WAIT_OBJECTS, &handleCount);
	freeHandles = (handles != handleBuf);
    } else {
	handles = onePort;
	handleCount = 1;
	freeHandles = FALSE;
    }
    
    // The run loop mode and loop are already in proper unlocked state from caller
    waitResult = MsgWaitForMultipleObjectsEx(__CFMin(handleCount, MAXIMUM_WAIT_OBJECTS), handles, timeout, mask, MWMO_INPUTAVAILABLE);
    
    CFAssert2(waitResult != WAIT_FAILED, __kCFLogAssertion, "%s(): error %d from MsgWaitForMultipleObjects", __PRETTY_FUNCTION__, GetLastError());
    
    if (waitResult == WAIT_TIMEOUT) {
	// do nothing, just return to caller
	result = false;
    } else if (waitResult >= WAIT_OBJECT_0 && waitResult < WAIT_OBJECT_0+handleCount) {
	// a handle was signaled
	if (livePort) *livePort = handles[waitResult-WAIT_OBJECT_0];
	result = true;
    } else if (waitResult == WAIT_OBJECT_0+handleCount) {
	// windows message received
        if (msgReceived) *msgReceived = true;
	result = true;
    } else if (waitResult >= WAIT_ABANDONED_0 && waitResult < WAIT_ABANDONED_0+handleCount) {
	// an "abandoned mutex object"
	if (livePort) *livePort = handles[waitResult-WAIT_ABANDONED_0];
	result = true;
    } else {
	CFAssert2(waitResult == WAIT_FAILED, __kCFLogAssertion, "%s(): unexpected result from MsgWaitForMultipleObjects: %d", __PRETTY_FUNCTION__, waitResult);
	result = false;
    }
    
    if (freeHandles) {
	CFAllocatorDeallocate(kCFAllocatorSystemDefault, handles);
    }
    
    return result;
}

#endif

struct __timeout_context {
    dispatch_source_t ds;
    CFRunLoopRef rl;
    uint64_t termTSR;
};

static void __CFRunLoopTimeoutCancel(void *arg) {
    struct __timeout_context *context = (struct __timeout_context *)arg;
    CFRelease(context->rl);
    dispatch_release(context->ds);
    free(context);
}

static void __CFRunLoopTimeout(void *arg) {
    struct __timeout_context *context = (struct __timeout_context *)arg;
    context->termTSR = 0ULL;
    CFRUNLOOP_WAKEUP_FOR_TIMEOUT();
    CFRunLoopWakeUp(context->rl);
    // The interval is DISPATCH_TIME_FOREVER, so this won't fire again
}

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
    
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
	dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

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
#endif
	__CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);

        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

	__CFRunLoopDoBlocks(rl, rlm);

        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	}

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;

	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);
        
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
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
            // objc_clear_stack(0);
            // <rdar://problem/16393959>
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        
        
#elif DEPLOYMENT_TARGET_WINDOWS
        // Here, use the app-supplied message queue mask. They will set this if they are interested in having this run loop receive windows messages.
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);

#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            // These Win32 APIs cause a callout, so make sure we're unlocked first and relocked after
            __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);

            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
            
            // To prevent starvation of sources other than the message queue, we check again to see if any other sources need to be serviced
            // Use 0 for the mask so windows messages are ignored this time. Also use 0 for the timeout, because we're just checking to see if the things are signalled right now -- we will wait on them again later.
            // NOTE: Ignore the dispatch source (it's not in the wait set anymore) and also don't run the observers here since we are polling.
            __CFRunLoopSetSleeping(rl);
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            
            __CFRunLoopWaitForMultipleObjects(waitSet, NULL, 0, 0, &livePort, NULL);
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);            
            __CFRunLoopUnsetSleeping(rl);
            // If we have a new live port then it will be handled below as normal
        }
        
        
#endif
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
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
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            
            // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

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
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
	    }
            
            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
            
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);
        

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
        
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```





### ps：

#### __CFRuntimeBase

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

#### CFRunLoopTimerContext

timer的回调不是一个简单的block，而是首先封装了一个下面的这种

```c
// CFRunLoopTimerContext 包含程序定义的数据和回调的结构，您可以使用它配置 CFRunLoopTimer 的行为
static void _runLoopTimerWithBlockContext(CFRunLoopTimerRef timer, void *opaqueBlock) {
    typedef void (^timer_block_t) (CFRunLoopTimerRef timer);
    timer_block_t block = (timer_block_t)opaqueBlock;
    block(timer);
}
```

```c
// 之所以列举这个方法，是因为由于timer存在循环引用，因此苹果新增了带block的api来创建timer。
// 顺便看看CFRunLoopTimerContext对象的创建
CFRunLoopTimerRef CFRunLoopTimerCreateWithHandler(CFAllocatorRef allocator, CFAbsoluteTime fireDate, CFTimeInterval interval, CFOptionFlags flags, CFIndex order,
						void (^block) (CFRunLoopTimerRef timer)) {
    
    CFRunLoopTimerContext blockContext;
    blockContext.version = 0;
    blockContext.info = (void *)block;	// 这才是我们真正的执行的回调
    blockContext.retain = (const void *(*)(const void *info))_Block_copy;
    blockContext.release = (void (*)(const void *info))_Block_release;
    blockContext.copyDescription = NULL;
    return CFRunLoopTimerCreate(allocator, fireDate, interval, flags, order, _runLoopTimerWithBlockContext, &blockContext);
}
```

```c
CFRunLoopTimerRef CFRunLoopTimerCreate(CFAllocatorRef allocator, CFAbsoluteTime fireDate, CFTimeInterval interval, CFOptionFlags flags, CFIndex order, CFRunLoopTimerCallBack callout, CFRunLoopTimerContext *context) {
    CHECK_FOR_FORK();
    if (isnan(interval)) {
        CRSetCrashLogMessage("NaN was used as an interval for a CFRunLoopTimer");
        HALT;
    }
    CFRunLoopTimerRef memory;   // 这里的timer用的是memory，我们换成timer吧
    CFRunLoopTimerRef timer;		// 注意，这个是我加的，不是源代码的
  	UInt32 size;
    size = sizeof(struct __CFRunLoopTimer) - sizeof(CFRuntimeBase);
    timer = (CFRunLoopTimerRef)_CFRuntimeCreateInstance(allocator, CFRunLoopTimerGetTypeID(), size, NULL);
    if (NULL == timer) {
	return NULL;
    }
    __CFSetValid(timer);
    __CFRunLoopTimerUnsetFiring(timer);
    __CFRunLoopLockInit(&timer->_lock);
    timer->_runLoop = NULL;
    timer->_rlModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    timer->_order = order;
    if (interval < 0.0) interval = 0.0;
    timer->_interval = interval;
    timer->_tolerance = 0.0;
    if (TIMER_DATE_LIMIT < fireDate) fireDate = TIMER_DATE_LIMIT;
    timer->_nextFireDate = fireDate;
    timer->_fireTSR = 0ULL;
    uint64_t now2 = mach_absolute_time();
    CFAbsoluteTime now1 = CFAbsoluteTimeGetCurrent();
    if (fireDate < now1) {
	timer->_fireTSR = now2;
    } else if (TIMER_INTERVAL_LIMIT < fireDate - now1) {	// #define TIMER_INTERVAL_LIMIT	504911232.0
	timer->_fireTSR = now2 + __CFTimeIntervalToTSR(TIMER_INTERVAL_LIMIT);
    } else {
	timer->_fireTSR = now2 + __CFTimeIntervalToTSR(fireDate - now1);
    }
    timer->_callout = callout;
    if (NULL != context) {
	if (context->retain) {
	    timer->_context.info = (void *)context->retain(context->info);
	} else {
	    timer->_context.info = context->info;
	}
	timer->_context.retain = context->retain;
	timer->_context.release = context->release;
	timer->_context.copyDescription = context->copyDescription;
    } else {
	timer->_context.info = 0;
	timer->_context.retain = 0;
	timer->_context.release = 0;
	timer->_context.copyDescription = 0;
    }
    return memory;
}
```

```c
// 最终执行这个方法来完成timer的回调
__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(rlt->_callout, rlt, context_info);

static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(CFRunLoopTimerCallBack func, CFRunLoopTimerRef timer, void *info) {
    if (func) {
        func(timer, info); 
      	// 你串一下就可以发现，就是这样的一个操作
      	// _callout是一个函数指针指向了这个函数_runLoopTimerWithBlockContext，就是这一节第一个函数
      	// 然后在这个函数中执行我们的回调：info 
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

#### __CFRunLoopRun

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();

    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
	return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
	rlm->_stopped = false;
	return kCFRunLoopRunStopped;
    }
    
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
	dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

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
#endif
	__CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);

        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

	__CFRunLoopDoBlocks(rl, rlm);

        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	}

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;

	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);
        
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
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
            // objc_clear_stack(0);
            // <rdar://problem/16393959>
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        
        
#elif DEPLOYMENT_TARGET_WINDOWS
        // Here, use the app-supplied message queue mask. They will set this if they are interested in having this run loop receive windows messages.
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);

#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            // These Win32 APIs cause a callout, so make sure we're unlocked first and relocked after
            __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);

            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
            
            // To prevent starvation of sources other than the message queue, we check again to see if any other sources need to be serviced
            // Use 0 for the mask so windows messages are ignored this time. Also use 0 for the timeout, because we're just checking to see if the things are signalled right now -- we will wait on them again later.
            // NOTE: Ignore the dispatch source (it's not in the wait set anymore) and also don't run the observers here since we are polling.
            __CFRunLoopSetSleeping(rl);
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            
            __CFRunLoopWaitForMultipleObjects(waitSet, NULL, 0, 0, &livePort, NULL);
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);            
            __CFRunLoopUnsetSleeping(rl);
            // If we have a new live port then it will be handled below as normal
        }
        
        
#endif
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
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
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            
            // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

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
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
	    }
            
            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
            
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);
        

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
        
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```





https://juejin.cn/post/6907543033751961607

http://joakimliu.github.io/2019/03/09/runloop-note/

https://www.cnblogs.com/kenshincui/p/6823841.html





autoreleaspoll

https://www.jianshu.com/p/0fe8ed6f37ae

https://www.jianshu.com/p/2d9071465a73