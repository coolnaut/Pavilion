> 君子坦荡荡，小人长戚戚。

# 1. RunLoop Mode

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

# 2. __CFRunLoopMode

```c
struct __CFRunLoopMode {
    CFRuntimeBase _base;		
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this 锁mode前，先锁定RunLoop*/
    CFStringRef _name;			 
    Boolean _stopped;
    char _padding[3];
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

# 3. __CFRunLoop

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

