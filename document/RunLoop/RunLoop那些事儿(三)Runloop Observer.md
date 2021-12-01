> 君子坦荡荡，小人长戚戚。

# 1. Runloop Observer

Observer 是非常重要的。观察下面的结构体，感觉像是的timer的结构体搬过来的，这样也可以猜到有一部分和timer是一样的，就是它的回调。其中_activities是observer独有的，代表观察的RunLoop的状态。如果RunLoop的状态为这个状态就会发起通知，调用回调。

# 2. __CFRunLoopObserver

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

## 2.1 _activities

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),         //即将进入run loop
    kCFRunLoopBeforeTimers = (1UL << 1),  //即将处理timer
    kCFRunLoopBeforeSources = (1UL << 2), //即将处理source
    kCFRunLoopBeforeWaiting = (1UL << 5), //即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),  //被唤醒但是还没开始处理事件
    kCFRunLoopExit = (1UL << 7),          //run loop已经退出
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

这里使用`CF_OPTIONS`后，后面进行位与操作后，就可以直接得到状态。

```c
// 截取其中一段代码可以看到
if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
```

## 2.2 _callout

在`__CFRunLoopDoObservers`方法中会调用下面的宏，通知事件

```C
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
// 和timer的是一样的
  static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(CFRunLoopObserverCallBack func, CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (func) {
        func(observer, activity, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

# 3. 应用

## 3.1 AutoreleasePool

App 启动后，苹果在主线程 RunLoop 里注册了下面的 Observer：

- 通知 Observers：**即将进入 Loop** => 调用 _objc_autoreleasePoolPush() 创建自动释放池
- do while
  - …
  - 通知 Observers：**即将进入休眠** => 调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池
  - …
- 通知 Observers: **即将退出** => 调用 _objc_autoreleasePoolPop() 来释放自动释放池

## 3.2 事件响应

苹果注册了一个 **Source1** 来接收触摸、加速、传感器等系统事件，随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

## 3.3 手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断，随后系统将对应的 UIGestureRecognizer 标记为待处理，苹果注册了一个 Observer 监测 **即将进入休眠**，其回调函数会获取所有刚被标记为待处理的 UIGestureRecognizer，并执行UIGestureRecognizer 的回调，当有 UIGestureRecognizer 的状态变化时，这个回调都会进行相应处理。

## 3.4 界面更新

当在操作 UI 时，这个 UIView 或 CALayer 就被标记为待处理，并被提交到一个全局的容器去，苹果注册了一个 Observer 监测 **即将进入休眠** 和 **即将退出**，回调去执行一个很长的函数，这个函数里会遍历所有待处理的 UIView 或 CALayer 以执行实际的绘制和调整，并更新 UI 界面。

## 3.4 关于 GCD

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE() 里执行这个 block，也就是对应 handle_msg 处理消息：如果 dispatch 就执行 block 。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

## 3.5 PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会 创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。当调用 performSelector:onThread: 时，实际上其会 创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。