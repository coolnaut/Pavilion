> 本文重在讨论苹果公司在IOS4引入的全新多线程编程技术——GCD。一个人理解来说一说GCD。

### 1. 引入GCD

​	首先在学习GCD之前，一定要对线程有一定的理解，例如，线程等待，死锁，线程同步等等，如需了解请点击【[线程的介绍](https://blog.csdn.net/Void_leng/article/details/89641606)】。另外GCD是一中与Block有关的技术，因此还需要学习一下Block，请点击【[IOS的Block](https://blog.csdn.net/Void_leng/article/details/103485049)】

​	好了，下面进入正文。GCD全称是"Grand Central Dispatch"，译为大中枢派发(来自《Effective Objective-C》)。GCD对线程进行了抽象，开发者只需要定义想要执行的任务(Block)并追加到合适的任务队列(Dispatch Queue)中，GCD就能生成必要的线程并执行任务。这里要着重的注意：**任务(Block) + 任务队列(Dispatch Queue)**。GCD用一种非常简洁的方式来实现了负责的多线程编程。

  很多IOS的资料都会推荐使用GCD，因为使用GCD会带来如下好处：

- GCD 可用于多核的并行运算；
- GCD 会自动利用更多的 CPU 内核（比如双核、四核）；
- GCD 会自动管理线程的生命周期（创建线程、调度任务、销毁线程）；
- 程序员只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码。

上述的好处会在后面的介绍中一一体现。

### 2. GCD的任务与队列

#### 2.1 任务

​	任务就是要执行的Block，其类型为`dispatch_block_t`。如果查看定义其实一个无返回值无参数的Block：`typedef void (^dispatch_block_t)(void);`。

#### 2.2 队列

​	队列就是数据结构中满足先进先出(FIFO)特性的队列，不过该队列中存储的数据是代码块，其类型为`dispatch_queue_t`。任务队列分为两种：

1. DISPATCH_QUEUE_SERIAL，串行队列

2. DISPATCH_QUEUE_CONCURRENT，并发队列

   对于上述两种队列。如果对于串行和并发有一定了解的话，就能明白。串行队列中，任务的执行是按部就班的一个一个的执行；并发队列中，任务可以同时发生，但任务执行是根据当前系统的状况，创建新线程或使用空闲线程来取到队列中的任务并执行。因此串行队列中就只有一个线程来执行任务，而并行队列中可能会有多个线程来并行执行，且且是无序的

`ps：并行是一个相对的概念，如果当前系统不是多核的，那么就不支持同一时间执行多个任务，因此就不支持并行`



![image-20201123165857010](/Users/lengguocheng/Library/Application Support/typora-user-images/image-20201123165857010.png)

![image-20201123165917051](/Users/lengguocheng/Library/Application Support/typora-user-images/image-20201123165917051.png)

### 3. GCD的基本使用

基本的介绍就在前面结束，下面通过介绍API来进一步了解和使用。

#### 3.1 创建队列

```objective-c
dispatch_queue_create(const char *_Nullable label,dispatch_queue_attr_t _Nullable attr);
```

第一个参数是队列的名字，通常采用公司域名的逆置，例如:com.xxx.xxx.queue或者团队的逆置名。第二个参数是队列的类型`DISPATCH_QUEUE_CONCURRENT`和`DISPATCH_QUEUE_SERIAL`。返回值类型为`dispatch_queue_t`，即就是返回一个队列

#### 3.2 任务的追加

```objective-c
dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

```objective-c
dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
```

async是asynchronous的简写，译为非同步，反之就是同步sync。第一个参数是任务队列，第二个参数是要执行的Block

1. `dispatch_sync`,往一个派发队列上提交一个**同步执行**的Block,**阻塞当前线程**.不开新线程.

2. `dispatch_async`,往一个派发队列上提交一个**异步执行**的Block,**不阻塞当前线程**.可能会开新线程.

#### 3.3 队列的获取

```objective-c
dispatch_get_main_queue()
```

获取主线程队列。对于串行队列，GCD 默认提供了主队列（Main Dispatch Queue）。**主队列其实并不特殊。** 主队列的实质上就是一个普通的串行队列，只是因为默认情况下，当前代码是放在主队列中的，然后主队列中的代码，又都会放到主线程中去执行，所以才造成了主队列特殊的现象

```
dispatch_get_global_queue(long identifier, unsigned long flags);
```

获取全局队列。对于并发队列，GCD 默认提供了全局并发队列（Global Dispatch Queue）。第一个参数表示队列优先级，一般用 `DISPATCH_QUEUE_PRIORITY_DEFAULT`。第二个参数为保留字段备用（一般为0）

#### 3.3 组合

GCD的基本使用，这里不再介绍，因为看到前面的API就可以使用了。这里讨论一下注意事项。首先，前面提到队列有两种队列，如果加上后面GCD默认提供的队列则有 4 种，但是全局并发队列可以作为普通并发队列来使用。由于当前代码默认放在主队列中，在使用中，会有差异，所以主队列很有必要专门来研究一下。由于追加又分为异步追加和同步追加，因此总共有 6 中组合方式。

>1. 同步执行 + 并发队列
>2. 异步执行 + 并发队列
>3. 同步执行 + 串行队列
>4. 异步执行 + 串行队列
>5. 同步执行 + 主队列
>6. 异步执行 + 主队列

`dispatch_sync` 提交的Block因为优化的原因**几乎总是**在当前线程执行的(`注意是当前线程`)，也就是说在哪个线程调用的这个方法，那么该Block就在哪个线程上执行，唯一的一个例外就是如果在子线程中调用并提交到主队列则block是在主线程执行。之所以说这个，是因为同步执行在某些情况下会导致死锁就是这个原因。

##### 3.3.1 同步执行 + 并发队列

在当前这一个线程中执行任务，不具有派发的能力，当前线程执行完一个任务，再执行下一个任务。

任务按顺序执行的。按顺序执行的原因：虽然 `并发队列` 可以开启多个线程，并且同时执行多个任务。但是因为本身不能创建新线程，只有当前线程这一个线程（`同步任务` 不具备开启新线程的能力），所以也就不存在并发。而且当前线程只有等待当前队列中正在执行的任务执行完毕之后，才能继续接着执行下面的操作（`同步任务` 需要等待队列的任务执行结束）。所以任务只能一个接一个按顺序执行，不能同时被执行。

##### 3.3.2 异步执行 + 并发队列

`异步执行` 具备开启新线程的能力。且 `并发队列` 可开启多个线程，同时执行多个任务

##### 3.3.3 同步执行 + 串行队列

`同步执行` 不具备开启新线程的能力，`串行队列` 每次只有一个任务被执行，任务一个接一个按顺序执行。请勿在同一个串行队列中执行同步派发。

![image-20201123201640764](/Users/lengguocheng/Library/Application Support/typora-user-images/image-20201123201640764.png)

在当前串行queue上调用sync函数，sync指定的queue也是当前的串行queue。那么需要执行的block被放到当前queue的队尾等待被执行，由于调用sync函数会阻塞当前队列，需要等待block被执行完毕，但是队列已经被阻塞，因此这个block一直不会被执行，那么sync函数一直不返回，于是就会造成死锁。

##### 3.3.4 异步执行 + 串行队列

`异步执行` 具备开启新线程的能力，`串行队列` 只开启一个线程，`串行队列` 每次只有一个任务被执行，任务一个接一个按顺序执行

##### 3.3.5 同步执行 + 主队列

main_queue为串行队列，因此不要在主线程使用sync向主队列派发任务。理由同 【3.3.3】。

##### 3.3.6 异步执行 + 主队列

同 3.3.4

### 4. GCD栅栏方法

当我们在并发队列异步执行几组任务时，可能会遇到需要第一组操作执行完之后，才能开始执行第二组操作这种情况，那么我们可以使用GCD的栅栏方法。

`dispatch_barrier_async` 方法会等待前边追加到并发队列中的任务全部执行完毕之后，再将指定的任务追加到该异步队列中。然后在 `dispatch_barrier_async` 方法追加的任务执行完毕之后，异步队列才恢复为一般动作，接着追加任务到该异步队列并开始执行。具体如下图所示：
![image-20211103200635439](/Users/lengguocheng/Library/Application Support/typora-user-images/image-20211103200635439.png)

### 5. 队列组

#### 5.1 dispatch_group_notify

通过`dispatch_group_create`创建队列组，通过`dispatch_group_async`将任务异步添加进group。`dispatch_group_notify`会在队列组的任务执行完毕后回到指定的队列执行任务。

> `dispatch_group`没有同步添加，因为同步执行+串行队列有死锁情况，具体参考 [3.3]小节。

```objective-c
- (void)groupNotify {
    // 创建group
    dispatch_group_t group =  dispatch_group_create();
    // 在group添加全局队列的任务
    dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:3]; 
        Log(@"dispatch_get_global_queue");
    });
    // 在group添加主队列的任务
    dispatch_group_async(group, dispatch_get_main_queue(), ^{
        [NSThread sleepForTimeInterval:3];              
        Log(@"dispatch_get_main_queue");
    });
    // 等前面的异步任务 1、任务 2 都执行完毕后，回到主线程执行下边任务    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        [NSThread sleepForTimeInterval:2]; 
        Log(@"group---end"); 
    });
}
```

#### 5.2 dispatch_group_wait

通过`dispatch_group_create`创建队列组，通过dispatch_group_async将任务添加进group。`dispatch_group_wait`会在队列组的任务执行时进行阻塞，其第二个参数为阻塞时间，是`dispatch_time_t`类型，当超过该时间则不再阻塞，可以使用超时，倒计时等。但是为什么不使用`dispatch_after`呢。

```objective-c
- (void)groupWait {
    dispatch_group_t group =  dispatch_group_create();
    dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:3];
        Log(@"dispatch_get_global_queue");
    });
    
    dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [NSThread sleepForTimeInterval:3];
        Log(@"dispatch_get_global_queue");
    });
    // 等待上面的任务全部完成后，会往下继续执行（会阻塞当前线程）
    // 这里用的是 DISPATCH_TIME_FOREVER 因此会永久阻塞
    // 使用DISPATCH_TIME_FOREVER，一定要注意死锁问题，因为这里会阻塞
    // 例如在 dispatch_group_async 中使用主线程队列，而当前线程也是主线程
    // 那么dispatch_group_wait阻塞主线程，导致添加进queue的任务也就无法执行，就会死锁
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    Log(@"group---end");
    
}
```

#### 5.3  dispatch_group_enter/leave

`dispatch_group_enter` 标志着一个任务追加到 group，执行一次，相当于 group 中未执行完毕任务数 +1

`dispatch_group_leave` 标志着一个任务离开了 group，执行一次，相当于 group 中未执行完毕任务数 -1

当 group 中未执行完毕任务数为0的时候，才会使 `dispatch_group_wait` 解除阻塞，以及执行追加到 `dispatch_group_notify` 中的任务。

```objc
- (void)groupEnterAndLeave {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:2];
        Log(@"dispatch_group_enter 1");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:2];
        Log(@"dispatch_group_enter 2");
        dispatch_group_leave(group);
    });
    // 这里使用 dispatch_group_wait 也可以
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        Log(@"group---end");
    });
}
```

这里我们可以用上面两个方法实现`dispatch_group_async`。也可以讨论一下为什么只有异步添加，没有同步添加。

```objective-c
- (void)dispatch_group_async:(dispatch_group_t)group queue:(dispatch_queue_t)queue block:(dispatch_block_t)block {
    if (block) {
        dispatch_group_enter(group);
        // 如果这里是 dispatch_sync
        // 且queue是串行队列, 那么当queue的线程在执行这个任务时就会死锁
        // 例子: 主线程同步执行dispatch_async, 且queue为主队列时, 就会死锁
        dispatch_async(queue, ^{
            block();
            dispatch_group_leave(group);
        });
    }
}
```

### 6. 信号量

GCD 中的信号量是指 **Dispatch Semaphore**，是用于控制线程同步，和其他线程同步原理是一致的，通过设置信号量来控制线程的执行顺序，当信号量为 1 时，可视为互斥锁。 **Dispatch Semaphore** 提供了三个方法：

`dispatch_semaphore_create`：创建一个 Semaphore 并初始化信号的总量

`dispatch_semaphore_signal`：发送一个信号，让信号总量加 1

`dispatch_semaphore_wait`：可以使总信号量减 1，信号总量小于 0 时就会一直等待（阻塞所在线程），否则就可以正常执行。

```objective-c
- (void)semaphoreSync {
    Log(@"semaphore---begin");
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:2];
        Log(@"semaphore do");      // 打印当前线程
        dispatch_semaphore_signal(semaphore);
    });
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
}
```

