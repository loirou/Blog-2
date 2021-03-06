## 多线程 の 拾遗

![SamuelChan/20180712235815.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20180712235815.png)

### 0x00  线程的状态
![SamuelChan/20170926161551.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20170926161551.png)

- 新建    :
- Runnable: 加入可调度线程池等待调用
- Running : CPU 负责调度可调度线程池中线程的执行;线程执行完成之前，状态可能会在就绪和运行之间来回切换;就绪和运行之间的状态变化由 CPU 负责，程序员`不能干预`
- Blocked : 当满足某个预定条件时，可以使用休眠或锁阻塞线程执行
`sleepForTimeInterval：休眠指定时长`
`sleepUntilDate：休眠到指定日期`
`@synchronized(self)：互斥锁`
- Dead    : [NSThread exit],一旦终止,后续代码都不会执行
正常死亡 线程执行完毕  
非正常死亡
当满足某个条件后，在线程内部中止执行
当满足某个条件后，在主线程中止线程对象
注意：一旦线程停止（死亡）了，就不能再次开启任务
- cancel  : **并不会直接取消线程;只是给线程对象添加 isCancelled 标记;需要在线程内部的关键代码位置,增加判断,决定是否取消当前线程**

  > Changes the cancelled state of the receiver to indicate that it should exit.
The semantics of this method are the same as those used for `NSOperation`. This method sets state information in the receiver that is then reflected by the cancelled property. Threads that support cancellation should `periodically` call the cancelled method to determine if the thread has in fact been cancelled, and exit if it has been.
For more information about cancellation and operation objects, see NSOperation SDKs`



### 0x01GCD

- 串行队列 FIFO
- 并行队列
- 主队列   FIFO
- 全局队列

```objc
/*Creates a new dispatch queue to which blocks may be submitted.
 * @param label
 * A string label to attach to the queue.
 * This parameter is optional and may be NULL.
 *
 * @param attr
 * A predefined attribute such as DISPATCH_QUEUE_SERIAL,
 * DISPATCH_QUEUE_CONCURRENT, or the result of a call to
 * a dispatch_queue_attr_make_with_* function.

 */
dispatch_queue_create(const char *_Nullable label,dispatch_queue_attr_t _Nullable attr);

//串行队列
dispatch_queue_t queue = dispatch_queue_create("serialSync", DISPATCH_QUEUE_SERIAL);

//并行队列
dispatch_queue_t queue = dispatch_queue_create("concurrentAsnc",DISPATCH_QUEUE_CONCURRENT);

//主队列是特殊的串行队列,主队列的任务都只能在主线程中执行
dispatch_queue_t queue = dispatch_get_main_queue()

//全局队列 由系统创建, 队列名称固定为 com.apple.root.default-qos,
//dispatch_queue_t dispatch_get_global_queue(long identifier, unsigned long flags); == (标志位 ,未来扩展位)
dispatch_queue_t queue = dispatch_get_global_queue(0,0)

```

> 死锁 : 主线程中,向主队列添加任务,同步执行(队列引起的循环等待)

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"%s", __FUNCTION__);
    });
}
```

**一些面试问题**

> 1 2 3 4 5

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"1");
    dispatch_queue_t global_queue = dispatch_get_global_queue(0, 0);
    dispatch_sync(global_queue, ^{
        NSLog(@"2");
        dispatch_sync(global_queue, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
}
```

> 1 3 

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    dispatch_queue_t global_queue = dispatch_get_global_queue(0, 0);
    
    dispatch_async(global_queue, ^{
        NSLog(@"1");
        // 子线程runloop没开
        [self performSelector:@selector(printLog) withObject:nil afterDelay:0];
        NSLog(@"3");
    });
}

- (void)printLog{
    NSLog(@"2");
}

- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:
(NSTimeInterval)delay;
Description	
Invokes a method of the receiver on the current thread using the default mode after a delay.This 
method sets up a timer to perform the aSelector message on the current thread’s run loop. The 
timer is configured to run in the default mode (NSDefaultRunLoopMode). When the timer fires, 
the thread attempts to dequeue the message from the run loop and perform the selector. It 
succeeds if the run loop is running and in the default mode; otherwise, the timer waits until the 
run loop is in the default mode.

```

#### dispatch\_barrier\_async
- 主要用于在多个异步操作完成之后，统一对非线程安全的对象进行更新
- 适合于大规模的 I/O 操作, 多读单写
- dispatch\_barrier\_async 添加的 block 会在之前添加的 block 全部运行结束之后，统一在同一个线程顺序执行，从而保证对非线程安全的对象进行正确的操作

![SamuelChan/20180712180326.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20180712180326.png)

> 实际使用1

```objc
// 下载
- (void)barrierExample01{
    
    _photoList = [[NSMutableArray alloc] init];
    _photoQueue = dispatch_queue_create("com.samuel.barrier", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 330; ++i) {
        dispatch_async(_photoQueue, ^{
            NSLog(@"下载照片%d", i);
            dispatch_barrier_async(_photoQueue, ^{
                NSLog(@"添加图片 %d,%@", i,[NSThread currentThread]);
                [self.photoList addObject:@(i)];
            });
        });
    }
}

2018-07-12 20:57:50.295409+0800 multiThread[17143:7731419] 下载照片2
2018-07-12 20:57:50.295500+0800 multiThread[17143:7731419] 下载照片3
2018-07-12 20:57:50.295563+0800 multiThread[17143:7731419] 下载照片4
2018-07-12 20:57:50.295623+0800 multiThread[17143:7731419] 下载照片5
2018-07-12 20:57:50.295683+0800 multiThread[17143:7731419] 下载照片6
2018-07-12 20:57:50.295742+0800 multiThread[17143:7731419] 下载照片7
.
.
.
.
2018-07-12 20:57:50.345310+0800 multiThread[17143:7731522] 添加图片 1,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345392+0800 multiThread[17143:7731522] 添加图片 2,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345424+0800 multiThread[17143:7731522] 添加图片 3,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345495+0800 multiThread[17143:7731522] 添加图片 4,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345553+0800 multiThread[17143:7731522] 添加图片 5,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345655+0800 multiThread[17143:7731522] 添加图片 6,<NSThread: 0x1c4664540>{number = 4, name = (null)}
2018-07-12 20:57:50.345687+0800 multiThread[17143:7731522] 添加图片 7,<NSThread: 0x1c4664540>{number = 4, name = (null)}
```

> example 2 : 单读多写

```objc
@interface UserCenter()
{
    // 定义一个并发队列
    dispatch_queue_t concurrent_queue;
    
    // 用户数据中心, 可能多个线程需要数据访问
    NSMutableDictionary *userCenterDic;
}

@end

// 多读单写模型
@implementation UserCenter

- (id)init
{
    self = [super init];
    if (self) {
        // 通过宏定义 DISPATCH_QUEUE_CONCURRENT 创建一个并发队列
        concurrent_queue = dispatch_queue_create("read_write_queue", DISPATCH_QUEUE_CONCURRENT);
        // 创建数据容器
        userCenterDic = [NSMutableDictionary dictionary];
    }
    
    return self;
}

- (id)objectForKey:(NSString *)key
{
    __block id obj;
    // 同步读取指定数据
    dispatch_sync(concurrent_queue, ^{
        obj = [userCenterDic objectForKey:key];
    });
    
    return obj;
}

- (void)setObject:(id)obj forKey:(NSString *)key
{
    // 异步栅栏调用设置数据
    dispatch_barrier_async(concurrent_queue, ^{
        [userCenterDic setObject:obj forKey:key];
    });
}
```

> example 3 : AFNetworking

```objc
- (void)setValue:(NSString *)value
forHTTPHeaderField:(NSString *)field
{
    dispatch_barrier_async(self.requestHeaderModificationQueue, ^{
        [self.mutableHTTPRequestHeaders setValue:value forKey:field];
    });
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    NSString __block *value;
    dispatch_sync(self.requestHeaderModificationQueue, ^{
        value = [self.mutableHTTPRequestHeaders valueForKey:field];
    });
    return value;
}
```


#### dispatch_group

```objc
//第一种方法:使用 dispatch_group_async 和 dispatch_group_notify
      // 1. 调度组
      dispatch_group_t group = dispatch_group_create();

      // 2. 队列
      dispatch_queue_t queue = dispatch_queue_create("com.samuel.group", DISPATCH_QUEUE_CONCURRENT);

      // 将任务添加到调度组中
      dispatch_group_async(group, queue, ^{
          [NSThread sleepForTimeInterval:2];
          NSLog(@"下载音乐 A");
      });

      dispatch_group_async(group, queue, ^{
          NSLog(@"下载音乐 B");
      });

      dispatch_group_async(group, queue, ^{
          NSLog(@"下载音乐 C");
      });

      // 是异步的,等所有任务离开组就调用 dispatch_group_notify
      dispatch_group_notify(group, dispatch_get_main_queue(), ^{
          NSLog(@"所有异步任务下载完成了, 在 主线程更新UI thread = %@", [NSThread currentThread]);
      });
      
      
      
//第二种方法更多常用:因为一般来说任务都是耗时的,所以异步请求,无论有无成功,第一种方式都是直接leave_group了
      // 1. 调度组
      dispatch_group_t group = dispatch_group_create();
      // 2. 队列
      dispatch_queue_t queue = dispatch_queue_create("com.samuel.group", DISPATCH_QUEUE_CONCURRENT);

      // 在 terminal 输入 man(manual) dispatch_group_async可以看到这个函数的详细介绍
      // 进入组
      dispatch_group_enter(group);
      dispatch_async(queue, ^{
          [NSThread sleepForTimeInterval:2];
          NSLog(@"下载音乐 A");
          // 离开组
          dispatch_group_leave(group);
      });

      dispatch_group_enter(group);
      dispatch_async(queue, ^{
          NSLog(@"下载音乐 B");
          // 离开组
          dispatch_group_leave(group);
      });

      dispatch_group_enter(group);
      dispatch_async(queue, ^{
          NSLog(@"下载音乐 C");
          // 离开组
          dispatch_group_leave(group);
      });

      // 是异步的,等所有任务离开组就调用 dispatch_group_notify
      dispatch_group_notify(group, dispatch_get_main_queue(), ^{
          NSLog(@"所有异步任务下载完成了, 在 主线程更新UI thread = %@", [NSThread currentThread]);
      });
```

#### dispatch_after

```objc
     /*
      参数:
          1. dispatch_time_t when,       什么时候执行
                 dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.5 * NSEC_PER_SEC)) 在当前时间过后多少 `纳秒` 执行
          2. dispatch_queue_t queue,     在哪个队列执行
          3. dispatch_block_t block      要执行的代码
         疑问: dispatch_after 是同步还是异步的? -- 异步的
      */
     dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.5 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
         NSLog(@"执行 thread = %@", [NSThread currentThread]);
     });
    
     NSLog(@"end");
```

#### dispatch_once

```objc
+ (LoadingWaitView *)sharedHUD{
    static dispatch_once_t once;
    static LoadingWaitView *sharedHUD;
    // onceToken 默认是 0, 当执行到dispatch_once时, onceToken = 0 就执行 Block 里面的代码,onceToken 变为 -1
    dispatch_once(&once, ^{
        sharedHUD = [[self alloc] initWithFrame:kXMScreen.bounds];
    });
    return sharedHUD;
}
```

### 0x02 NSOperation
1.概述

- 默认是`并发队列` && `异步执行`
- 方便的操作: 
	- `最大并发数` 
	- `队列的暂定/继续` 
	- `取消所有的操作` 
	- `指定操作之间的依赖关系(GCD可以用同步实现)`
- NSOperation 是一个抽象类，不能直接使用，需要使用子类
	- 抽象类的作用:定义子类共有的属性和方法
		- UICollectionViewLayout
		- UIGestureRecognizer
		- CAAnimation
		- CAPropertyAnimation
	- NSInvocationOperation
	- NSBlockOperation
	- 自定义子类继承 NSOperation，实现内部相应的方法


2.使用步骤

- 创建 操作 (NSOperation)：确定要做的事情
- 创建 队列 (NSOperationQueue)：用于存放 操作 (NSOperation)
- 将操作添加到队列中(异步,并行)
	- 系统会自动将 NSOperationQueue 中的 NSOperation取出
	- 将取出的 NSOperation 封装的操作放到新线程中执行
	- 任务的取出遵循队列的 FIFO 原则：先进先出，后进后出

3.代码

```objc
//1.NSInvocationOperation
    NSInvocationOperation *invocationOperation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(invocationOperation:) object:@{@"msg": @"invocationOperation 基本使用", @"parameter": @"呵呵"}];
    //创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    //将操作添加到队列中
    [queue addOperation:invocationOperation];
    
    
//2.NSBlockOperation
    // 创建操作
    NSBlockOperation *blockOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"blockOperation1, thread = %@", [NSThread currentThread]);
    }];
    //一个NSBlockOperation 操作可以添加多个 block 任务
    for (NSInteger i = 2; i < 100; ++i) {
        [blockOperation addExecutionBlock:^{
            NSLog(@"blockOperation%ld, thread = %@",i, [NSThread currentThread]);
        }];
    }
    //创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    //将任务添加到队列中
    [queue addOperation:blockOperation];
    
//3.自定义NSOperation:需要实现main方法
  - (void)main {
      // 要执行的代码
      NSLog(@"自定义操作 线程 = %@", [NSThread currentThread]);
  }

//4.完成的回调
    [operation1 setCompletionBlock:^{
        NSLog(@"end invocation thread = %@", [NSThread currentThread]);
    }];

//5.线程间通讯
    NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];
//6.操作优先级
    op1.qualityOfService = NSQualityOfServiceUserInteractive;

//7.最大并发数:最多同时执行任务的数量,不是线程数量
    queue.maxConcurrentOperationCount = 2;
//8.队列的挂起,恢复,取消操作
    queue.suspended      = NO / YES;
    queue.operationCount;
    [queue cancelAllOperations];//取消队列的所有操作,已经在执行的操作继续执行
//9.设置依赖关系:一定要让 操作A执行完后,才能执行操作B
   [operationB addDependency:operationA]; // 操作B依赖于操作A
```

4.NSOperation状态控制

> [gnustep-base-1.24.9](https://launchpad.net/ubuntu/+archive/primary/+sourcefiles/gnustep-base/1.24.9-3ubuntu1/gnustep-base_1.24.9.orig.tar.gz)

- 重写operation的main(),底层控制变更任务执行完成状态,以及任务退出
- 重写start(),自行控制任务状态

**main()**

```objc
- (void)main;
The default implementation of this method does nothing. You should override this method to 
perform the desired task. In your implementation, do not invoke super. This method will 
automatically execute within an autorelease pool provided by NSOperation, so you do not need 
to create your own autorelease pool block in your implementation.

If you are implementing a concurrent operation, you are not required to override this method but 
may do so if you plan to call it from your custom start method.

- (void) main {
  return;	// OSX default implementation does nothing
}

```

**start()**

```objc
- (void)start;	//Begins the execution of the operation.
The default implementation of this method updates the execution state of the operation and calls 
the receiver’s main method. This method also performs several checks to ensure that the 
operation can actually run. For example, if the receiver was cancelled or is already finished, this 
method simply returns without calling main. (In OS X v10.5, this method throws an exception if 
the operation is already finished.) If the operation is currently executing or is not ready to 
execute, this method throws an NSInvalidArgumentException exception. In OS X v10.5, this 
method catches and ignores any exceptions thrown by your main method automatically. In 
macOS 10.6 and later, exceptions are allowed to propagate beyond this method. You should 
never allow exceptions to propagate out of your main method.

Note
An operation is not considered ready to execute if it is still dependent on other operations that 
have not yet finished.

If you are implementing a concurrent operation, you must override this method and use it to 
initiate your operation. Your custom implementation must not call super at any time. In addition to 
configuring the execution environment for your task, your implementation of this method must 
also track the state of the operation and provide appropriate state transitions. When the 
operation executes and subsequently finishes its work, it should generate KVO notifications for 
the isExecuting and isFinished key paths respectively. For more information about manually 
generating KVO notifications, see Key-Value Observing Programming Guide.

You can call this method explicitly if you want to execute your operations manually. However, it is 
a programmer error to call this method on an operation object that is already in an operation 
queue or to queue the operation after calling this method. Once you add an operation object to a 
queue, the queue assumes all responsibility for it.
	
```

**start() gnuStep源码**

```objc

- (void) start
{
  NSAutoreleasePool	*pool = [NSAutoreleasePool new];
  double		prio = [NSThread  threadPriority];

  [internal->lock lock];
  NS_DURING
    {
      if (YES == [self isConcurrent])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on concurrent operation",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (YES == [self isExecuting])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on executing operation",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (YES == [self isFinished])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on finished operation",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (NO == [self isReady])
	{
	  [NSException raise: NSInvalidArgumentException
		      format: @"[%@-%@] called on operation which is not ready",
	    NSStringFromClass([self class]), NSStringFromSelector(_cmd)];
	}
      if (NO == internal->executing)
	{
	  [self willChangeValueForKey: @"isExecuting"];
	  internal->executing = YES;
	  [self didChangeValueForKey: @"isExecuting"];
	}
    }
  NS_HANDLER
    {
      [internal->lock unlock];
      [localException raise];
    }
  NS_ENDHANDLER
  [internal->lock unlock];

  NS_DURING
    {
      if (NO == [self isCancelled])
	{
	  [NSThread setThreadPriority: internal->threadPriority];
	  [self main];
	}
    }
  NS_HANDLER
    {
      [NSThread setThreadPriority:  prio];
      [localException raise];
    }
  NS_ENDHANDLER;

  [self _finish];  // kvo方式通知queue移除NSOperation
  [pool release];
}
```

### 0x03 NSThread
![SamuelChan/20180712231736.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20180712231736.png)

```objc
- (void) start
{
  pthread_attr_t	attr;
  pthread_t		thr;

  ......
  
  /* Make sure the notification is posted BEFORE the new thread starts.
   */
  gnustep_base_thread_callback();

  /* The thread must persist until it finishes executing.
   */
  RETAIN(self);

  /* Mark the thread as active whiul it's running.
   */
  _active = YES;

  errno = 0;
  pthread_attr_init(&attr);
  /* Create this thread detached, because we never use the return state from
   * threads.
   */
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  /* Set the stack size when the thread is created.  Unlike the old setrlimit
   * code, this actually works.
   */
  if (_stackSize > 0)
    {
      pthread_attr_setstacksize(&attr, _stackSize);
    }
  if (pthread_create(&thr, &attr, nsthreadLauncher, self))
    {
      DESTROY(self);
      [NSException raise: NSInternalInconsistencyException
                  format: @"Unable to detach thread (last error %@)",
                  [NSError _last]];
    }
}

static void *nsthreadLauncher(void* thread)
{
    NSThread *t = (NSThread*)thread;
    setThreadForCurrentThread(t);
	
    ......
  /*
   * Let observers know a new thread is starting.
   */
  if (nc == nil)
    {
      nc = RETAIN([NSNotificationCenter defaultCenter]);
    }
  [nc postNotificationName: NSThreadDidStartNotification
		    object: t
		  userInfo: nil];

  [t _setName: [t name]];

  [t main];

  [NSThread exit];
  // Not reached
  return NULL;
}

- (void) main
{
  if (_active == NO)
    {
      [NSException raise: NSInternalInconsistencyException
                  format: @"[%@-%@] called on inactive thread",
        NSStringFromClass([self class]),
        NSStringFromSelector(_cmd)];
    }

  [_target performSelector: _selector withObject: _arg];
}

```

### 多线程锁相关问题
```objc
// 创建有多少信号量
dispatch_semaphore_create(long value);
// semaphore -1 < 0, 阻塞
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout);
// semaphore + 1 >= 0 唤醒
dispatch_semaphore_signal(dispatch_semaphore_t dsema);

- (NSArray *)tasksForKeyPath:(NSString *)keyPath {
    __block NSArray *tasks = nil;
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return tasks;
}
```

**使用信号量来使用dispatch_async_group**

```objc
    //创建信号量
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
    
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:10];
        dispatch_semaphore_signal(semaphore);
    });
    
    dispatch_group_async(group, queue, ^{
        [NSThread sleepForTimeInterval:10];
        dispatch_semaphore_signal(semaphore);
    });
    
    dispatch_group_notify(group, queue, ^{
        
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
        NSLog(@"信号量为0");
    });
```