---
title: iOS多线程详解：实践篇
date: 2018-03-23 20:13:10
tags: iOS多线程
---


![](http://upload-images.jianshu.io/upload_images/301129-b23b2a2ecc8c16be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

iOS多线程实践中，常用的就是子线程执行耗时操作，然后回到主线程刷新UI。在iOS中每个进程启动后都会建立一个主线程（UI线程），这个线程是其他线程的父线程。由于在iOS中除了主线程，其他子线程是独立于Cocoa Touch的，所以只有主线程可以更新UI界面。iOS多线程开发实践方式有4种，分别为Pthreads、NSThread、GCD、NSOperation，下面分别讲一讲各自的使用方式，以及优缺点。

![](https://upload-images.jianshu.io/upload_images/301129-bdbbb7dc36c1ab26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

pthread: 跨平台，适用于多种操作系统，可移植性强，是一套纯C语言的通用API，且线程的生命周期需要程序员自己管理，使用难度较大，所以在实际开发中通常不使用。
NSThread: 基于OC语言的API，使得其简单易用，面向对象操作。线程的声明周期由程序员管理，在实际开发中偶尔使用。
GCD: 基于C语言的API，充分利用设备的多核，旨在替换NSThread等线程技术。线程的生命周期由系统自动管理，在实际开发中经常使用。
NSOperation: 基于OC语言API，底层是GCD，增加了一些更加简单易用的功能，使用更加面向对象。线程生命周期由系统自动管理，在实际开发中经常使用。

### Pthreads

> 引自 [维基百科](https://zh.wikipedia.org/wiki/POSIX%E7%BA%BF%E7%A8%8B)
> 实现POSIX 线程标准的库常被称作Pthreads，一般用于Unix-like POSIX 系统，如Linux、 Solaris。但是Microsoft Windows上的实现也存在，例如直接使用Windows API实现的第三方库pthreads-w32；而利用Windows的SFU/SUA子系统，则可以使用微软提供的一部分原生POSIX API。

其实，这就是一套在很多操作系统上都通用的多线程API，所以移植性很强，基于C封装的一套线程框架，iOS上也是适用的。


#### Pthreads创建线程

```
- (void)onThread {
    // 1. 创建线程: 定义一个pthread_t类型变量
    pthread_t thread;
    // 2. 开启线程: 执行任务
    pthread_create(&thread, NULL, run, NULL);
    // 3. 设置子线程的状态设置为detached，该线程运行结束后会自动释放所有资源
    pthread_detach(thread);
}

void * run(void *param) {
    NSLog(@"%@", [NSThread currentThread]);

    return NULL;
}
```

> 打印结果：
> 2018-03-16 11:06:12.298115+0800 ocgcd[13744:5710531] <NSThread: 0x1c026f100>{number = 4, name = (null)}

如果出现`'pthread_create' is invalid in C99`报错，原因是没有导入`#import <pthread.h>`

——`pthread_create(&thread, NULL, run, NULL);` 中各项参数含义：——

* 第一个参数`&thread`是线程对象，指向线程标识符的指针
* 第二个是线程属性，可赋值`NULL`
* 第三个`run`表示指向函数的指针(`run`对应函数里是需要在新线程中执行的任务)
* 第四个是运行函数的参数，可赋值`NULL`

#### Pthreads其他相关方法

* `pthread_create()`：创建一个线程
* `pthread_exit()`：终止当前线程
* `pthread_cancel()`：中断另外一个线程的运行
* `pthread_join()`：阻塞当前的线程，直到另外一个线程运行结束
* `pthread_attr_init()`：初始化线程的属性
* `pthread_attr_setdetachstate()`：设置脱离状态的属性（决定这个线程在终止时是否可以被结合）
* `pthread_attr_getdetachstate()`：获取脱离状态的属性
* `pthread_attr_destroy()`：删除线程的属性
* `pthread_kill()`：向线程发送一个信号

#### Pthreads常用函数与功能

* pthread_t

pthread_t用于表示Thread ID，具体内容根据实现的不同而不同，有可能是一个Structure，因此不能将其看作为整数。

* pthread_equal

pthread_equal函数用于比较两个pthread_t是否相等。

```
int pthread_equal(pthread_t tid1, pthread_t tid2)
```

* pthread_self

pthread_self函数用于获得本线程的thread id。

```
pthread _t pthread_self(void);
```

#### Pthreads实现互斥锁

> 锁可以被动态或静态创建，可以用宏PTHREAD_MUTEX_INITIALIZER来静态的初始化锁，采用这种方式比较容易理解，互斥锁是`pthread_mutex_t`的结构体，而这个宏是一个结构常量，如下可以完成静态的初始化锁：`pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;`也可以用pthread_mutex_init函数动态的创建，函数原型如：`int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t * attr)`

总共有100张火车票，开启两个线程，北京和上海两个窗口同时卖票，卖一张票就减去库存，使用锁，保证北京和上海卖票的库存是一致的。实现如下。

```
#import "ViewController.h"
#include <pthread.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    [self onThread];
}

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
NSMutableArray *tickets;

- (void)onThread {
    tickets = [NSMutableArray array];
    //生成100张票
    for (int i = 0; i < 100; i++) {
        [tickets addObject:[NSNumber numberWithInt:i]];
    }
    
    //线程1 北京卖票窗口
    // 1. 创建线程1: 定义一个pthread_t类型变量
    pthread_t thread1;
    // 2. 开启线程1: 执行任务
    pthread_create(&thread1, NULL, run, NULL);
    // 3. 设置子线程1的状态设置为detached，该线程运行结束后会自动释放所有资源
    pthread_detach(thread1);
    
    //线程2 上海卖票窗口
    // 1. 创建线程2: 定义一个pthread_t类型变量
    pthread_t thread2;
    // 2. 开启线程2: 执行任务
    pthread_create(&thread2, NULL, run, NULL);
    // 3. 设置子线程2的状态设置为detached，该线程运行结束后会自动释放所有资源
    pthread_detach(thread2);

}

void * run(void *param) {
    while (true) {
        //锁门，执行任务
        pthread_mutex_lock(&mutex);
        if (tickets.count > 0) {
            NSLog(@"剩余票数%ld, 卖票窗口%@", tickets.count, [NSThread currentThread]);
            [tickets removeLastObject];
            [NSThread sleepForTimeInterval:0.2];
        }
        else {
            NSLog(@"票已经卖完了");

            //开门，让其他任务可以执行
            pthread_mutex_unlock(&mutex);

            break;
        }
        
        //开门，让其他任务可以执行
        pthread_mutex_unlock(&mutex);
    }

    return NULL;
}

@end
```

> 打印结果：
> 2018-03-16 11:47:01.069412+0800 ocgcd[13758:5723862] 剩余票数100, 卖票窗口<NSThread: 0x1c0667600>{number = 5, name = (null)}
2018-03-16 11:47:01.272654+0800 ocgcd[13758:5723863] 剩余票数99, 卖票窗口<NSThread: 0x1c0672b40>{number = 6, name = (null)}
2018-03-16 11:47:01.488456+0800 ocgcd[13758:5723862] 剩余票数98, 卖票窗口<NSThread: 0x1c0667600>{number = 5, name = (null)}
2018-03-16 11:47:01.691334+0800 ocgcd[13758:5723863] 剩余票数97, 卖票窗口<NSThread: 0x1c0672b40>{number = 6, name = (null)}
> .................
> 2018-03-16 11:47:12.110962+0800 ocgcd[13758:5723862] 剩余票数46, 卖票窗口<NSThread: 0x1c0667600>{number = 5, name = (null)}
2018-03-16 11:47:12.316060+0800 ocgcd[13758:5723863] 剩余票数45, 卖票窗口<NSThread: 0x1c0672b40>{number = 6, name = (null)}
2018-03-16 11:47:12.529002+0800 ocgcd[13758:5723862] 剩余票数44, 卖票窗口<NSThread: 0x1c0667600>{number = 5, name = (null)}
2018-03-16 11:47:12.731459+0800 ocgcd[13758:5723863] 剩余票数43, 卖票窗口<NSThread: 0x1c0672b40>{number = 6, name = (null)}
> .................
> 2018-03-16 11:47:21.103237+0800 ocgcd[13758:5723862] 剩余票数2, 卖票窗口<NSThread: 0x1c0667600>{number = 5, name = (null)}
2018-03-16 11:47:21.308605+0800 ocgcd[13758:5723863] 剩余票数1, 卖票窗口<NSThread: 0x1c0672b40>{number = 6, name = (null)}
2018-03-16 11:47:21.511062+0800 ocgcd[13758:5723862] 票已经卖完了
2018-03-16 11:47:21.511505+0800 ocgcd[13758:5723863] 票已经卖完了

对锁的操作主要包括加锁`pthread_mutex_lock()`、解锁`pthread_mutex_unlock()`和测试加锁`pthread_mutex_trylock()`三个。
`pthread_mutex_trylock()`语义与`pthread_mutex_lock()`类似，不同的是在锁已经被占据时返回`EBUSY`而不是挂起等待。

### NSThread

> `NSThread`是面向对象的，封装程度最小最轻量级的，使用更灵活，但要手动管理线程的生命周期、线程同步和线程加锁等，开销较大。
> `NSThread`的基本使用比较简单，可以动态创建初始化`NSThread`对象，对其进行设置然后启动；也可以通过`NSThread`的静态方法快速创建并启动新线程；此外`NSObject`基类对象还提供了隐式快速创建`NSThread`线程的`performSelector`系列类别扩展工具方法；`NSThread`还提供了一些静态工具接口来控制当前线程以及获取当前线程的一些信息。

#### NSThread创建线程

NSThread有三种创建方式：

* `initWithTarget`方式，先创建线程对象，再启动

```
- (void)onThread {
    // 创建并启动
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    // 设置线程名
    [thread setName:@"thread1"];
    // 设置优先级，优先级从0到1，1最高
    [thread setThreadPriority:0.9];
    // 启动
    [thread start];
}

- (void)run {
    NSLog(@"当前线程%@", [NSThread currentThread]);
}
```

> 打印结果：
> 2018-03-16 13:47:25.133244+0800 ocgcd[13811:5776836] 当前线程<NSThread: 0x1c0264480>{number = 4, name = thread1}

* detachNewThreadSelector显式创建并启动线程

```
- (void)onThread {
    // 使用类方法创建线程并自动启动线程
    [NSThread detachNewThreadSelector:@selector(run) toTarget:self withObject:nil];
}

- (void)run {
    NSLog(@"当前线程%@", [NSThread currentThread]);
}
```

> 打印结果：
> 2018-03-16 13:49:34.620546+0800 ocgcd[13814:5777803] 当前线程<NSThread: 0x1c026a940>{number = 5, name = (null)}

* performSelectorInBackground隐式创建并启动线程

```
- (void)onThread {
    // 使用NSObject的方法隐式创建并自动启动
    [self performSelectorInBackground:@selector(run) withObject:nil];
}

- (void)run {
    NSLog(@"当前线程%@", [NSThread currentThread]);
}
```

> 打印结果：
> 2018-03-16 13:54:33.451895+0800 ocgcd[13820:5780922] 当前线程<NSThread: 0x1c4460280>{number = 4, name = (null)}

#### NSThread方法

```
//获取当前线程
 +(NSThread *)currentThread; 
//创建线程后自动启动线程
+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(id)argument;
//是否是多线程
+ (BOOL)isMultiThreaded;
//线程字典
- (NSMutableDictionary *)threadDictionary;
//线程休眠到什么时间
+ (void)sleepUntilDate:(NSDate *)date;
//线程休眠多久
+ (void)sleepForTimeInterval:(NSTimeInterval)ti;
//取消线程
- (void)cancel;
//启动线程
- (void)start;
//退出线程
+ (void)exit;
//线程优先级
+ (double)threadPriority;
+ (BOOL)setThreadPriority:(double)p;
- (double)threadPriority NS_AVAILABLE(10_6, 4_0);
- (void)setThreadPriority:(double)p NS_AVAILABLE(10_6, 4_0);
//调用栈返回地址
+ (NSArray *)callStackReturnAddresses NS_AVAILABLE(10_5, 2_0);
+ (NSArray *)callStackSymbols NS_AVAILABLE(10_6, 4_0);
//设置线程名字
- (void)setName:(NSString *)n NS_AVAILABLE(10_5, 2_0);
- (NSString *)name NS_AVAILABLE(10_5, 2_0);
//获取栈的大小
- (NSUInteger)stackSize NS_AVAILABLE(10_5, 2_0);
- (void)setStackSize:(NSUInteger)s NS_AVAILABLE(10_5, 2_0);
// 获得主线程
+ (NSThread *)mainThread;
//是否是主线程
- (BOOL)isMainThread NS_AVAILABLE(10_5, 2_0);
+ (BOOL)isMainThread NS_AVAILABLE(10_5, 2_0); // reports whether current thread is main
+ (NSThread *)mainThread NS_AVAILABLE(10_5, 2_0);
//初始化方法
- (id)init NS_AVAILABLE(10_5, 2_0); // designated initializer
- (id)initWithTarget:(id)target selector:(SEL)selector object:(id)argument NS_AVAILABLE(10_5, 2_0);
//是否正在执行
- (BOOL)isExecuting NS_AVAILABLE(10_5, 2_0);
//是否执行完成
- (BOOL)isFinished NS_AVAILABLE(10_5, 2_0);
//是否取消线程
- (BOOL)isCancelled NS_AVAILABLE(10_5, 2_0);
- (void)cancel NS_AVAILABLE(10_5, 2_0);
//线程启动
- (void)start NS_AVAILABLE(10_5, 2_0);
- (void)main NS_AVAILABLE(10_5, 2_0); // thread body method
@end
//多线程通知
FOUNDATION_EXPORT NSString * const NSWillBecomeMultiThreadedNotification;
FOUNDATION_EXPORT NSString * const NSDidBecomeSingleThreadedNotification;
FOUNDATION_EXPORT NSString * const NSThreadWillExitNotification;

@interface NSObject (NSThreadPerformAdditions)
//与主线程通信
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array;
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait;
  // equivalent to the first method with kCFRunLoopCommonModes
//与其他子线程通信
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array NS_AVAILABLE(10_5, 2_0);
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait NS_AVAILABLE(10_5, 2_0);
  // equivalent to the first method with kCFRunLoopCommonModes
//隐式创建并启动线程
- (void)performSelectorInBackground:(SEL)aSelector withObject:(id)arg NS_AVAILABLE(10_5, 2_0);
```

#### NSThread线程状态

* 启动线程

```
// 线程启动
- (void)start;
```

* 阻塞线程

```
// 线程休眠到某一时刻
+ (void)sleepUntilDate:(NSDate *)date;
// 线程休眠多久
+ (void)sleepForTimeInterval:(NSTimeInterval)ti;
```

* 结束线程

```
// 结束线程
+ (void)exit;
```

关于`cancel`的疑问，当使用`cancel`方法时，只是改变了线程的状态标识，并不能结束线程，所以我们要配合`isCancelled`方法进行使用。

```
- (void)onThread {
    // 使用NSObject的方法隐式创建并自动启动
    [self performSelectorInBackground:@selector(run) withObject:nil];
}

- (void)run {
    NSLog(@"当前线程%@", [NSThread currentThread]);
    
    for (int i = 0 ; i < 100; i++) {
        if (i == 20) {
            //取消线程
            [[NSThread currentThread] cancel];
            NSLog(@"取消线程%@", [NSThread currentThread]);
        }
        
        if ([[NSThread currentThread] isCancelled]) {
            NSLog(@"结束线程%@", [NSThread currentThread]);
            //结束线程
            [NSThread exit];
            NSLog(@"这行代码不会打印的");
        }
        
    }
}
```

> 打印结果：
> 2018-03-16 14:11:44.423324+0800 ocgcd[13833:5787076] 当前线程<NSThread: 0x1c4466840>{number = 4, name = (null)}
2018-03-16 14:11:44.425124+0800 ocgcd[13833:5787076] 取消线程<NSThread: 0x1c4466840>{number = 4, name = (null)}
2018-03-16 14:11:44.426391+0800 ocgcd[13833:5787076] 结束线程<NSThread: 0x1c4466840>{number = 4, name = (null)}

线程的状态如下图：

![](https://upload-images.jianshu.io/upload_images/301129-6a83411d68633102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1、新建：实例化对象

2、就绪：向线程对象发送`start`消息，线程对象被加入“可调度线程池”等待`CPU`调度；`detach`方法和`performSelectorInBackground`方法会直接实例化一个线程对象并加入“可调度线程池”

3、运行:`CPU`负责调度“可调度线程池”中线程的执行,线程执行完成之前，状态可能会在“就绪”和“运行”之间来回切换,“就绪”和“运行”之间的状态变化由`CPU`负责，程序员不能干预

4、阻塞:当满足某个预定条件时，可以使用休眠或锁阻塞线程执行,影响的方法有：`sleepForTimeInterval`，`sleepUntilDate`，`@synchronized(self)`线程锁；线程对象进入阻塞状态后，会被从“可调度线程池”中移出，CPU 不再调度

5、死亡

死亡方式：

正常死亡：线程执行完毕
非正常死亡：线程内死亡--->`[NSThread exit]`:强行中止后，后续代码都不会在执行
线程外死亡：`[threadObj cancel]`--->通知线程对象取消,在线程执行方法中需要增加`isCancelled`判断,如果`isCancelled == YES`，直接返回

死亡后线程对象的`isFinished`属性为`YES`；如果是发送`cancle`消息，线程对象的`isCancelled`属性为`YES`；死亡后`stackSize == 0`，内存空间被释放

#### NSThread线程间通信

> 在开发中，我们经常会在子线程进行耗时操作，操作结束后再回到主线程去刷新UI。这就涉及到了子线程和主线程之间的通信。看一下官方关于NSThread的线程间通信的方法。

```
// 在主线程上执行操作
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait;
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray<NSString *> *)array;
  // equivalent to the first method with kCFRunLoopCommonModes

// 在指定线程上执行操作
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array NS_AVAILABLE(10_5, 2_0);
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait NS_AVAILABLE(10_5, 2_0);

// 在当前线程上执行操作，调用 NSObject 的 performSelector:相关方法
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

下面通过一个经典的下载图片DEMO来展示线程之间的通信。具体步骤如下：
1、开启一个子线程，在子线程中下载图片。
2、回到主线程刷新UI，将图片展示在`UIImageView`中。

```
func onThread() {
    let urlStr = "http://tupian.aladd.net/2015/7/2941.jpg"
    self.performSelector(inBackground: #selector(downloadImg(_:)), with: urlStr)
}

@objc func downloadImg(_ urlStr: String) {
    //打印当前线程
    print("下载图片线程", Thread.current)
    
    //获取图片链接
    guard let url = URL.init(string: urlStr) else {return}
    //下载图片二进制数据
    guard let data = try? Data.init(contentsOf: url) else {return}
    //设置图片
    guard let img = UIImage.init(data: data) else {return}
    
    //回到主线程刷新UI
    self.performSelector(onMainThread: #selector(downloadFinished(_:)), with: img, waitUntilDone: false)
}

@objc func downloadFinished(_ img: UIImage) {
    //打印当前线程
    print("刷新UI线程", Thread.current)

	// 赋值图片到imageview
    self.imageView.image = image
}
```

#### NSThread线程安全

线程安全，也可以被称为线程同步，主要是解决多线程争抢操作资源的问题，就比如火车票，全国各地多个售票窗口同事去售卖同一列火车票。
怎么保证，多地售票的票池保持一致，就需要用到多线程同步的技术去实现了。

NSThread线程安全和GCD、NSOperation线程安全都是一样的，实现方法无非就是加锁（各种锁的实现）、信号量、GCD栅栏等。
具体实现，可以看`iOS多线程详解：概念篇`线程同步段落。

### GCD

> GCD（Grand Central Dispatch是苹果为多核并行运算提出的C语言并发技术框架。
> GCD会自动利用更多的CPU内核；
会自动管理线程的生命周期（创建线程，调度任务，销毁线程等）；
程序员只需要告诉GCD想要如何执行什么任务，不需要编写任何线程管理代码。

#### GCD底层实现

>我们使用的GCD的API是C语言函数，全部包含在LIBdispatch库中，DispatchQueue通过结构体和链表被实现为FIFO的队列；而FIFO的队列是由dispatch_async等函数追加的Block来管理的；Block不是直接加入FIFO队列，而是先加入Dispatch Continuation结构体，然后在加入FIFO队列，Dispatch Continuation用于记忆Block所属的Dispatch Group和其他一些信息（相当于上下文）。
>Dispatch Queue可通过dispatch_set_target_queue()设定，可以设定执行该Dispatch Queue处理的Dispatch Queue为目标。该目标可像串珠子一样，设定多个连接在一起的Dispatch Queue,但是在连接串的最后必须设定Main Dispatch Queue，或各种优先级的Global Dispatch Queue,或是准备用于Serial Dispatch Queue的Global Dispatch Queue

Global Dispatch Queue的8种优先级： 

.High priority 
.Default Priority 
.Low Priority 
.Background Priority 
.High Overcommit Priority 
.Default Overcommit Priority 
.Low Overcommit Priority 
.Background Overcommit Priority 

附有Overcommit的Global Dispatch Queue使用在Serial Dispatch Queue中，不管系统状态如何，都会强制生成线程的 Dispatch Queue。 这8种Global Dispatch Queue各使用1个pthread_workqueue

* GCD初始化

> GCD初始化时，使用pthread_workqueue_create_np函数生成pthread_workqueue。pthread_workqueue包含在Libc提供的pthreads的API中，他使用bsthread_register和workq_open系统调用，在初始化XNU内核的workqueue之后获取workqueue信息。

其中XNU有四种workqueue： 

WORKQUEUE_HIGH_PRIOQUEUE 
WORKQUEUE_DEFAULT_PRIOQUEUE 
WORKQUEUE_LOW_PRIOQUEUE 
WORKQUEUE_BG_PRIOQUEUE 

这四种workqueue与Global Dispatch Queue的执行优先级相同

* Dispatch Queue执行block的过程

1、当在Global Dispatch Queue中执行Block时，libdispatch从Global Dispatch Queue自身的FIFO中取出Dispatch Continuation，调用pthread_workqueue_additem_np函数，将该Global Dispatch Queue、符合其优先级的workqueue信息以及执行Dispatch Continuation的回调函数等传递给pthread_workqueue_additem_np函数的参数。

2、thread_workqueue_additem_np()使用workq_kernreturn系统调用，通知workqueue增加应当执行的项目。

3、根据该通知，XUN内核基于系统状态判断是否要生成线程，如果是Overcommit优先级的Global Dispatch Queue，workqueue则始终生成线程。 

4、workqueue的线程执行pthread_workqueue(),该函数用libdispatch的回调函数，在回调函数中执行执行加入到Dispatch Continuatin的Block。 

5、Block执行结束后，进行通知Dispatch Group结束，释放Dispatch Continuation等处理，开始准备执行加入到Dispatch Continuation中的下一个Block。

#### GCD使用步骤

GCD 的使用步骤其实很简单，只有两步。

1、创建一个队列（串行队列或并发队列）
2、将任务追加到任务的等待队列中，然后系统就会根据任务类型执行任务（同步执行或异步执行）

##### 队列的创建方法/获取方法

iOS系统默认已经存在两种队列，主队列(串行队列)和全局队列(并发队列)，那我们可以利用GCD提供的接口创建并发和串行队列。

__关于同步、异步、串行、并行的概念和区别，在`iOS多线程详解：概念篇`中有详细说明__

* 创建串行队列

```
//创建串行队列
let que = DispatchQueue.init(label: "com.jacyshan.thread")
```

使用`DispatchQueue`初始化创建队列，默认是串行队列。第一个参数是表示队列的唯一标识符，用于 DEBUG，可为空，Dispatch Queue 的名称推荐使用应用程序 ID 这种逆序全程域名。

* 创建并发队列

```
//创建并发队列
let que = DispatchQueue.init(label: "com.jacyshan.thread", attributes: .concurrent)
```

第二个参数输入`.concurrent`标示创建的是一个并发队列

* 获取主队列

主队列（Main Dispatch Queue）是GCD 提供了的一种特殊的串行队列
所有放在主队列中的任务，都会放到主线程中执行。

```
//获取主队列
let que = DispatchQueue.main
```

* 获取全局队列

GCD 默认提供了全局并发队列（Global Dispatch Queue）。

```
//获取全局队列
let que = DispatchQueue.global()
```

##### 任务的创建方法

GCD 提供了同步执行任务的创建方法sync和异步执行任务创建方法async。

```
//同步执行任务创建方法
que.sync {
    print("任务1", Thread.current)
}

//异步执行任务创建方法
que.async {
    print("任务2", Thread.current)
}
```

有两种队列（串行队列/并发队列），两种任务执行方式（同步执行/异步执行），那么我们就有了四种不同的组合方式。这四种不同的组合方式是：
> 1、同步执行 + 并发队列
2、异步执行 + 并发队列
3、同步执行 + 串行队列
4、异步执行 + 串行队列

系统还提供了两种特殊队列：全局并发队列、主队列。全局并发队列可以作为普通并发队列来使用。但是主队列因为有点特殊，所以我们就又多了两种组合方式。这样就有六种不同的组合方式了。
> 5、同步执行 + 主队列
6、异步执行 + 主队列

六中组合方式区别通过显示如下。

| 区别 | 并发队列 | 串行队列 | 主队列 |
|:-:|:-:|:-:|:-:|
| 同步执行 | 没有开启新线程，串行执行任务 | 没有开启新线程，串行执行任务 | 主线程调用：死锁卡住不执行 其他线程调用：没有开启新线程，串行执行任务 |
| 异步执行 |有开启新线程，并发执行任务 | 有开启新线程(1条)，串行执行任务 | 没有开启新线程，串行执行任务 |


#### GCD六种组合实现

##### 同步+并发队列

在当前线程中执行任务，任务按顺序执行。

```
    @IBAction func onThread() {
        //打印当前线程
        print("currentThread---", Thread.current)
	    print("代码块------begin")
        
        //创建并发队列
        let que = DispatchQueue.init(label: "com.jackyshan.thread", attributes: .concurrent)
        
        //添加任务1
        que.sync {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
                print("任务1Thread---", Thread.current) //打印当前线程
            }
        }
        
        //添加任务2
        que.sync {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
                print("任务2Thread---", Thread.current)
            }
        }
	    print("代码块------end")
    }
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0078f00>{number = 1, name = main}
> 代码块------begin
任务1Thread--- <NSThread: 0x1c0078f00>{number = 1, name = main}
任务1Thread--- <NSThread: 0x1c0078f00>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c0078f00>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c0078f00>{number = 1, name = main}
代码块------end

可以看到任务是在主线程执行的，因为同步并没有开启新的线程。因为同步会阻塞线程，所以当我们的任务操作耗时的时候，我们界面的点击和滑动都是无效的。
__因为UI的操作也是在主线程，但是任务的耗时已经阻塞了线程，UI操作是没有反应的__。

##### 异步+并发队列

开启多个线程，任务交替执行。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let que = DispatchQueue.init(label: "com.jackyshan.thread", attributes: .concurrent)
    
    //添加任务1
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0062180>{number = 1, name = main}
> 代码块------begin
代码块------end
任务1Thread--- <NSThread: 0x1c02695c0>{number = 4, name = (null)}
任务2Thread--- <NSThread: 0x1c02694c0>{number = 5, name = (null)}
任务1Thread--- <NSThread: 0x1c02695c0>{number = 4, name = (null)}
任务2Thread--- <NSThread: 0x1c02694c0>{number = 5, name = (null)}

可以看到任务是在多个新的线程执行完成的，并没有在主线程执行，因此任务在执行耗时操作的时候，并不会影响UI操作。
异步可以开启新的线程，并发又可以执行多个线程的任务。因为异步没有阻塞线程，`代码块------begin 代码块------end`立即执行了，其他线程执行完耗时操作之后才打印。

##### 同步+串行队列

在当前线程中执行任务，任务按顺序执行。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建串行队列，DispatchQueue默认是串行队列
    let que = DispatchQueue.init(label: "com.jackyshan.thread")
    
    //添加任务1
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c40658c0>{number = 1, name = main}
代码块------begin
任务1Thread--- <NSThread: 0x1c40658c0>{number = 1, name = main}
任务1Thread--- <NSThread: 0x1c40658c0>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c40658c0>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c40658c0>{number = 1, name = main}
代码块------end

同样的，串行执行同步任务的时候，也没有开启新的线程，在主线程上执行任务，耗时操作会影响UI操作。

##### 异步+串行队列

开启一个新的线程，在新的线程中执行任务，任务按顺序执行。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建串行队列，DispatchQueue默认是串行队列
    let que = DispatchQueue.init(label: "com.jackyshan.thread")
    
    //添加任务1
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c407b700>{number = 1, name = main}
代码块------begin
代码块------end
任务1Thread--- <NSThread: 0x1c0462440>{number = 4, name = (null)}
任务1Thread--- <NSThread: 0x1c0462440>{number = 4, name = (null)}
任务2Thread--- <NSThread: 0x1c0462440>{number = 4, name = (null)}
任务2Thread--- <NSThread: 0x1c0462440>{number = 4, name = (null)}

从打印可以看到只开启了一个线程(串行只会开启一个线程)，任务是在新的线程按顺序执行的。
任务是在`代码块------begin 代码块------end`后执行的(异步不会等待任务执行完毕)

下面是讲__主队列__，主队列是一种特殊的串行队列，所有任务(异步同步)都会在主线程执行。

##### 同步+主队列

任务在主线程中调用会出现死锁，其他线程不会。

* 在主线程执行`同步+主队列`

界面卡死，所有操作没有反应。任务互相等待造成死锁。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //获取主队列
    let que = DispatchQueue.main
    
    //添加任务1
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c00766c0>{number = 1, name = main}
代码块------begin

可以看到`代码块------begin`执行完，后面就不执行的，卡住不动了，等一会还会崩溃。

感觉死锁很多文章讲的不是很清楚，其实流程就是互相等待，简单解释如下：

原因是`onThread()`这个任务是在主线程执行的，__`任务1`被添加到主队列，要等待队列`onThread()`任务执行完才会执行__。
然后，__`任务1`是在`onThread()`这个任务中的__，按照`FIFO`的原则，__`onThread()`先被添加到主队列，应该先执行完，但是`任务1`在等待`onThread()`执行完才会执行__。
这样就造成了死锁，互相等待对方完成任务。

* 在其他线程执行`同步+主队列`

主队列不会开启新的线程，任务按顺序在主线程执行

```
override func viewDidLoad() {
    super.viewDidLoad()
    // Do any additional setup after loading the view, typically from a nib.
    
    //开启新的线程执行onThread任务
    performSelector(inBackground: #selector(onThread), with: nil)
}

@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //获取主队列
    let que = DispatchQueue.main
    
    //添加任务1
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.sync {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0076a80>{number = 4, name = (null)}
代码块------begin
任务1Thread--- <NSThread: 0x1c406d680>{number = 1, name = main}
任务1Thread--- <NSThread: 0x1c406d680>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c406d680>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c406d680>{number = 1, name = main}
代码块------end

`onThread`任务是在其他线程执行的，没有添加到主队列，所有也不会等待`任务1、2`的完成，因此不会死锁。

这里有个疑问，有的人会想`串行队列+同步`和`并发队列+同步`为什么不会死锁呢，其实如果`onThread`任务和`同步任务`在同一个队列中，而且`同步任务`是在`onThread`中执行的，也会造成死锁。
在一个队列中，就会出现互相等待的现象，刚好同步又不好开启新的线程，这样就会死锁了。

##### 异步+主队列

主队列不会开启新的线程，任务按顺序在主线程执行

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //获取主队列
    let que = DispatchQueue.main
    
    //添加任务1
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务1Thread---", Thread.current) //打印当前线程
        }
    }
    
    //添加任务2
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2) //模拟耗时操作
            print("任务2Thread---", Thread.current)
        }
    }
    print("代码块------end")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c4076500>{number = 1, name = main}
代码块------begin
代码块------end
任务1Thread--- <NSThread: 0x1c4076500>{number = 1, name = main}
任务1Thread--- <NSThread: 0x1c4076500>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c4076500>{number = 1, name = main}
任务2Thread--- <NSThread: 0x1c4076500>{number = 1, name = main}

可以看到`onThread`任务执行完了，没有等待`任务1、2`的完成(异步立即执行不等待)，所以不会死锁。
主队列是串行队列，任务是按顺序一个接一个执行的。

#### GCD的其他方法

##### asyncAfter延迟执行

很多时候我们希望延迟执行某个任务，这个时候使用`DispatchQueue.main.asyncAfter`是很方便的。
这个方法并不是立马执行的，延迟执行也不是绝对准确，可以看到，他是在延迟时间过后，把任务追加到主队列，如果主队列有其他耗时任务，这个延迟任务，相对的也要等待任务完成。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //主线程延迟执行
    let delay = DispatchTime.now() + .seconds(3)
    DispatchQueue.main.asyncAfter(deadline: delay) {
        print("asyncAfter---", Thread.current)
    }
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c407f900>{number = 1, name = main}
代码块------begin
asyncAfter--- <NSThread: 0x1c407f900>{number = 1, name = main}

##### DispatchWorkItem

`DispatchWorkItem`是一个代码块，它可以在任意一个队列上被调用，因此它里面的代码可以在后台运行，也可以在主线程运行。
它的使用真的很简单，就是一堆可以直接调用的代码，而不用像之前一样每次都写一个代码块。我们也可以使用它的通知完成回调任务。

做多线程的业务的时候，经常会有需求，当我们在做耗时操作的时候完成的时候发个通知告诉我这个任务做完了。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建workItem
    let workItem = DispatchWorkItem.init {
        for _ in 0..<2 {
            print("任务workItem---", Thread.current)
        }
    }
    
    //全局队列(并发队列)执行workItem
    DispatchQueue.global().async {
        workItem.perform()
    }
    
    //执行完之后通知
    workItem.notify(queue: DispatchQueue.main) {
        print("任务workItem完成---", Thread.current)
    }
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c4079300>{number = 1, name = main}
代码块------begin
代码块------结束
任务workItem--- <NSThread: 0x1c42705c0>{number = 5, name = (null)}
任务workItem--- <NSThread: 0x1c42705c0>{number = 5, name = (null)}
任务workItem完成--- <NSThread: 0x1c4079300>{number = 1, name = main}

可以看到我们使用全局队列异步执行了`workItem`，任务执行完，收到了通知。

##### DispatchGroup队列组

有些复杂的业务可能会有这个需求，几个队列执行任务，然后把这些队列都放到一个组`Group`里，当组里所有队列的任务都完成了之后，`Group`发出通知，回到主队列完成其他任务。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建 DispatchGroup
    let group =  DispatchGroup()

    group.enter()
    //全局队列(并发队列)执行任务
    DispatchQueue.global().async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务1------", Thread.current)//打印线程
        }
        
        group.leave()
    }
    
    //如果需要上个队列完成后再执行可以用wait
    group.wait()
    group.enter()
    //自定义并发队列执行任务
    DispatchQueue.init(label: "com.jackyshan.thread").async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务2------", Thread.current)//打印线程
        }
        
        group.leave()
    }
    
    //全部执行完后回到主线程刷新UI
    group.notify(queue: DispatchQueue.main) {
        print("任务执行完毕------", Thread.current)//打印线程
    }
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0261bc0>{number = 1, name = main}
代码块------begin
任务1------ <NSThread: 0x1c046b900>{number = 5, name = (null)}
任务1------ <NSThread: 0x1c046b900>{number = 5, name = (null)}
代码块------结束
任务2------ <NSThread: 0x1c0476b00>{number = 6, name = (null)}
任务2------ <NSThread: 0x1c0476b00>{number = 6, name = (null)}
任务执行完毕------ <NSThread: 0x1c0261bc0>{number = 1, name = main}

两个队列，一个执行默认的全局队列，一个是自己自定义的并发队列，两个队列都完成之后，group得到了通知。
如果把`group.wait()`注释掉，我们会看到两个队列的任务会交替执行。

##### dispatch_barrier_async栅栏方法

`dispatch_barrier_async`是`oc`的实现，`Swift`的实现`que.async(flags: .barrier)`这样。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let que = DispatchQueue.init(label: "com.jackyshan.thread", attributes: .concurrent)

	//并发异步执行任务
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务0------", Thread.current)//打印线程
        }
    }

    //并发异步执行任务
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务1------", Thread.current)//打印线程
        }
    }
    
    //栅栏方法：等待队列里前面的任务执行完之后执行
    que.async(flags: .barrier) {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务2------", Thread.current)//打印线程
        }
        //执行完之后执行队列后面的任务
    }
    
    //并发异步执行任务
    que.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务3------", Thread.current)//打印线程
        }
    }
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c4078a00>{number = 1, name = main}
代码块------begin
代码块------结束
任务0------ <NSThread: 0x1c427d5c0>{number = 5, name = (null)}
任务1------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}
任务0------ <NSThread: 0x1c427d5c0>{number = 5, name = (null)}
任务1------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}
任务2------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}
任务2------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}
任务3------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}
任务3------ <NSThread: 0x1c0470f80>{number = 4, name = (null)}

可以看到由于`任务2`执行的`barrier`的操作，`任务0和1`交替执行，`任务2`等待`0和1`执行完才执行，`任务3`也是等待`任务2`执行完毕。
也可以看到由于`barrier`的操作，并没有开启新的线程去跑任务。

##### Quality Of Service（QoS）和优先级

> 在使用 GCD 与 dispatch queue 时，我们经常需要告诉系统，应用程序中的哪些任务比较重要，需要更高的优先级去执行。当然，由于主队列总是用来处理 UI 以及界面的响应，所以在主线程执行的任务永远都有最高的优先级。不管在哪种情况下，只要告诉系统必要的信息，iOS 就会根据你的需求安排好队列的优先级以及它们所需要的资源（比如说所需的 CPU 执行时间）。虽然所有的任务最终都会完成，但是，重要的区别在于哪些任务更快完成，哪些任务完成得更晚。

> 用于指定任务重要程度以及优先级的信息，在 GCD 中被称为 Quality of Service（QoS）。事实上，QoS 是有几个特定值的枚举类型，我们可以根据需要的优先级，使用合适的 QoS 值来初始化队列。如果没有指定 QoS，则队列会使用默认优先级进行初始化。要详细了解 QoS 可用的值，可以参考这个文档，请确保你仔细看过这个文档。下面的列表总结了 Qos 可用的值，它们也被称为 QoS classes。第一个 class 代码了最高的优先级，最后一个代表了最低的优先级：

* userInteractive
* userInitiated
* default
* utility
* background
* unspecified

创建两个队列，优先级都是userInteractive，看看效果：

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列1
    let que1 = DispatchQueue.init(label: "com.jackyshan.thread1", qos: .userInteractive, attributes: .concurrent)

    //创建并发队列2
    let que2 = DispatchQueue.init(label: "com.jackyshan.thread2", qos: .userInteractive, attributes: .concurrent)
    
    que1.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务1------", Thread.current)//打印线程
        }
    }
    
    que2.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务2------", Thread.current)//打印线程
        }
    }

    print("代码块------结束")
}
```

> 打印结果：
currentThread--- <NSThread: 0x1c0073680>{number = 1, name = main}
代码块------begin
代码块------结束
任务1------ <NSThread: 0x1c047cd80>{number = 5, name = (null)}
任务2------ <NSThread: 0x1c0476c40>{number = 3, name = (null)}
任务1------ <NSThread: 0x1c047cd80>{number = 5, name = (null)}
任务2------ <NSThread: 0x1c0476c40>{number = 3, name = (null)}

两个队列的优先级一样，任务也是交替执行，这和我们预测的一样。

下面把`queue1`的优先级改为`background`，看看效果：

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列1
    let que1 = DispatchQueue.init(label: "com.jackyshan.thread1", qos: .background, attributes: .concurrent)

    //创建并发队列2
    let que2 = DispatchQueue.init(label: "com.jackyshan.thread2", qos: .userInteractive, attributes: .concurrent)
    
    que1.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务1------", Thread.current)//打印线程
        }
    }
    
    que2.async {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("任务2------", Thread.current)//打印线程
        }
    }

    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c006afc0>{number = 1, name = main}
代码块------begin
代码块------结束
任务2------ <NSThread: 0x1c4070180>{number = 5, name = (null)}
任务1------ <NSThread: 0x1c006d400>{number = 6, name = (null)}
任务2------ <NSThread: 0x1c4070180>{number = 5, name = (null)}
任务1------ <NSThread: 0x1c006d400>{number = 6, name = (null)}

可以看到`queue1`的优先级调低为`background`，`queue2`的任务就优先执行了。

还有其他的优先级，从高到低，就不一一相互比较了。

##### DispatchSemaphore信号量

> GCD 中的信号量是指 Dispatch Semaphore，是持有计数的信号。类似于过高速路收费站的栏杆。可以通过时，打开栏杆，不可以通过时，关闭栏杆。在 Dispatch Semaphore 中，使用计数来完成这个功能，计数为0时等待，不可通过。计数为1或大于1时，计数减1且不等待，可通过。

* `DispatchSemaphore(value: )`：用于创建信号量，可以指定初始化信号量计数值，这里我们默认1.
* `semaphore.wait()`：会判断信号量，如果为1，则往下执行。如果是0，则等待。
* `semaphore.signal()`：代表运行结束，信号量加1，有等待的任务这个时候才会继续执行。

可以使用`DispatchSemaphore`实现线程同步，保证线程安全。

加入有一个票池，同时几个线程去卖票，我们要保证每个线程获取的票池是一致的。
使用`DispatchSemaphore`和刚才讲的`DispatchWorkItem`来实现，我们看看效果。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建票池
    var tickets = [Int]()
    for i in 0..<38 {
        tickets.append(i)
    }
    
    //创建一个初始计数值为1的信号
    let semaphore = DispatchSemaphore(value: 1)
    
    let workItem = DispatchWorkItem.init {
        semaphore.wait()
        
        if tickets.count > 0 {
            Thread.sleep(forTimeInterval: 0.2)//耗时操作
            print("剩余票数", tickets.count, Thread.current)
            
            tickets.removeLast()//去票池库存
        }
        else {
            print("票池没票了")
        }
        
        semaphore.signal()
    }

    //创建并发队列1
    let que1 = DispatchQueue.init(label: "com.jackyshan.thread1", qos: .background, attributes: .concurrent)

    //创建并发队列2
    let que2 = DispatchQueue.init(label: "com.jackyshan.thread2", qos: .userInteractive, attributes: .concurrent)
    
    que1.async {
        for _ in 0..<20 {
            workItem.perform()
        }
    }
    
    que2.async {
        for _ in 0..<20 {
            workItem.perform()
        }
    }

    print("代码块------结束")
}
```

> currentThread--- <NSThread: 0x1c407a6c0>{number = 1, name = main}
代码块------begin
代码块------结束
剩余票数 38 <NSThread: 0x1c44706c0>{number = 8, name = (null)}
剩余票数 37 <NSThread: 0x1c0264c80>{number = 9, name = (null)}
剩余票数 36 <NSThread: 0x1c44706c0>{number = 8, name = (null)}
> ................
剩余票数 19 <NSThread: 0x1c0264c80>{number = 9, name = (null)}
剩余票数 18 <NSThread: 0x1c44706c0>{number = 8, name = (null)}
剩余票数 17 <NSThread: 0x1c0264c80>{number = 9, name = (null)}
剩余票数 16 <NSThread: 0x1c44706c0>{number = 8, name = (null)}
> ................
> 剩余票数 2 <NSThread: 0x1c44706c0>{number = 8, name = (null)}
剩余票数 1 <NSThread: 0x1c0264c80>{number = 9, name = (null)}
票池没票了
票池没票了

可以看到我们的资源没有因为造成资源争抢而出现数据紊乱。信号量很好的实现了多线程同步的功能。

##### DispatchSource

> DispatchSource provides an interface for monitoring low-level system objects such as Mach ports, Unix descriptors, Unix signals, and VFS nodes for activity and submitting event handlers to dispatch queues for asynchronous processing when such activity occurs.
DispatchSource提供了一组接口，用来提交hander监测底层的事件，这些事件包括Mach ports，Unix descriptors，Unix signals，VFS nodes。


Tips: DispatchSource这个class很好的体现了Swift是一门面向协议的语言。这个类是一个工厂类，用来实现各种source。比如DispatchSourceTimer（本身是个协议）表示一个定时器。


* DispatchSourceProtocol

基础协议，所有的用到的[DispatchSource](https://developer.apple.com/documentation/dispatch/dispatchsourceprotocol)都实现了这个协议。这个协议的提供了公共的方法和属性：
由于不同的source是用到的属性和方法不一样，这里只列出几个公共的方法

- activate //激活
- suspend //挂起
- resume //继续
- cancel //取消(异步的取消，会保证当前eventHander执行完)
- setEventHandler //事件处理逻辑
- setCancelHandler //取消时候的清理逻辑

* DispatchSourceTimer

在Swift 3中，可以方便的用GCD创建一个Timer(新特性)。DispatchSourceTimer本身是一个协议。
比如，写一个timer，1秒后执行，然后10秒后自动取消，允许10毫秒的误差

```
PlaygroundPage.current.needsIndefiniteExecution = true

public let timer = DispatchSource.makeTimerSource()

timer.setEventHandler {
    //这里要注意循环引用，[weak self] in
    print("Timer fired at \(NSDate())")
}

timer.setCancelHandler {
    print("Timer canceled at \(NSDate())" )
}

timer.scheduleRepeating(deadline: .now() + .seconds(1), interval: 2.0, leeway: .microseconds(10))

print("Timer resume at \(NSDate())")

timer.resume()

DispatchQueue.main.asyncAfter(deadline: .now() + .seconds(10), execute:{
    timer.cancel()
})
```

`deadline`表示开始时间，`leeway`表示能够容忍的误差。

DispatchSourceTimer也支持只调用一次。

```
func scheduleOneshot(deadline: DispatchTime, leeway: DispatchTimeInterval = default)
```

* UserData

DispatchSource中UserData部分也是强有力的工具，这部分包括两个协议，两个协议都是用来合并数据的变化，只不过一个是按照+(加)的方式，一个是按照|(位与)的方式。

DispatchSourceUserDataAdd
DispatchSourceUserDataOr

> 在使用这两种Source的时候，GCD会帮助我们自动的将这些改变合并，然后在适当的时候（target queue空闲）的时候，去回调EventHandler,从而避免了频繁的回调导致CPU占用过多。

```
let userData = DispatchSource.makeUserDataAddSource()

var globalData:UInt = 0

userData.setEventHandler {
    let pendingData = userData.data
    globalData = globalData + pendingData
    print("Add \(pendingData) to global and current global is \(globalData)")
}

userData.resume()

let serialQueue = DispatchQueue(label: "com")

serialQueue.async {
    for var index in 1...1000 {
        userData.add(data: 1)
    }

    for var index in 1...1000 {
        userData.add(data: 1)
    }
}
```

> Add 32 to global and current global is 32
Add 1321 to global and current global is 1353
Add 617 to global and current global is 1970
Add 30 to global and current global is 2000


### NSOperation

> NSOperation是基于GCD的一个抽象基类，将线程封装成要执行的操作，不需要管理线程的生命周期和同步，但比GCD可控性更强，例如可以加入操作依赖（addDependency）、设置操作队列最大可并发执行的操作个数（setMaxConcurrentOperationCount）、取消操作（cancel）等。NSOperation作为抽象基类不具备封装我们的操作的功能，需要使用两个它的实体子类：NSBlockOperation和继承NSOperation自定义子类。NSOperation需要配合NSOperationQueue来实现多线程。

#### NSOperation使用步骤

##### 自定义Operation

继承`Operation`创建一个类，并重写`main`方法。当调用`start`的时候，会在适当的时候执行`main`里面的任务。

```
class ViewController: UIViewController {
    
    @IBAction func onThread() {
        //打印当前线程
        print("currentThread---", Thread.current)
        print("代码块------begin")
        
        let op = JKOperation.init()
        op.start()
        
        print("代码块------结束")
    }
}


class JKOperation: Operation {
    override func main() {
        Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
        print("任务---", Thread.current)
    }
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c007a280>{number = 1, name = main}
代码块------begin
任务--- <NSThread: 0x1c007a280>{number = 1, name = main}
代码块------结束

可以看到自定义`JKOperation`，初始化之后，调用`start`方法，`main`方法里面的任务执行了，是在主线程执行的。
因为我们没有使用`OperationQueue`，所以没有创建新的线程。

##### 使用BlockOperation

初始化`BlockOperation`之后，调用`start`方法。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    let bop = BlockOperation.init {
        Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
        print("任务---", Thread.current)
    }
    bop.start()
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c4070640>{number = 1, name = main}
代码块------begin
任务--- <NSThread: 0x1c4070640>{number = 1, name = main}
代码块------结束

##### 配合OperationQueue实现

初始化`OperationQueue`之后，调用`addOperation`，代码块就会自定执行，调用机制执行是有`OperationQueue`里面自动实现的。
`addOperation`的方法里面其实是生成了一个`BlockOperation`对象，然后执行了这个对象的`start`方法。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    OperationQueue.init().addOperation {
        Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
        print("任务---", Thread.current)
    }
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c006a700>{number = 1, name = main}
代码块------begin
代码块------结束
任务--- <NSThread: 0x1c0469900>{number = 5, name = (null)}

可以看到`OperationQueue`初始化，默认是生成了一个并发队列，而且执行的是一个异步操作，所以打印任务的线程不是在主线程。

##### 队列的创建方法/获取方法

`OperationQueue`没有实现串行队列的方法，也没有像`GCD`那样实现了一个全局队列。
只有并发队列的实现和主队列的获取。

* 创建并发队列

并发队列的任务是并发(几乎同时)执行的，可以最大发挥CPU多核的优势。
看到有的说通过`maxConcurrentOperationCount`设置并发数量1就实现了串行。
实际上是不对的，通过设置优先级可以控制队列的任务交替执行，在下面讲到`maxConcurrentOperationCount`会实现代码。

```
//创建并发队列
let queue = OperationQueue.init()
```

`OperationQueue`初始化，默认实现的是并发队列。

* 获取主队列

我们的主队列是串行队列，任务是一个接一个执行的。

```
//获取主队列
let queue = OperationQueue.main
```

获取主队列的任务是异步执行的。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //获取主队列
    let queue = OperationQueue.main
    
    queue.addOperation {
        Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
        print("任务1---", Thread.current)
    }

    queue.addOperation {
        Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
        print("任务2---", Thread.current)
    }
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0064f80>{number = 1, name = main}
代码块------begin
代码块------结束
任务1--- <NSThread: 0x1c0064f80>{number = 1, name = main}
任务2--- <NSThread: 0x1c0064f80>{number = 1, name = main}

##### 任务的创建方法

* 通过BlockOperation创建任务

```
BlockOperation.init {
    Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
    print("任务1---", Thread.current)
}.start()
```

* 通过OperationQueue创建任务

```
//创建并发队列
let queue = OperationQueue.init()

queue.addOperation {
    Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
    print("任务---", Thread.current)
}
```

#### NSOperation相关方法

##### 最大并发操作数：maxConcurrentOperationCount

> maxConcurrentOperationCount 默认情况下为-1，表示不进行限制，可进行并发执行。
maxConcurrentOperationCount这个值不应超过系统限制(64)，即使自己设置一个很大的值，系统也会自动调整为 min{自己设定的值，系统设定的默认最大值}。

* 设置`maxConcurrentOperationCount`为`1`，实现串行操作。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let queue = OperationQueue.init()
    //设置最大并发数为1
    queue.maxConcurrentOperationCount = 1

    let bq1 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务1---", Thread.current)
        }
    }
    
    let bq2 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务2---", Thread.current)
        }
    }

    queue.addOperations([bq1, bq2], waitUntilFinished: false)
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0067880>{number = 1, name = main}
代码块------begin
代码块------结束
任务1--- <NSThread: 0x1c046ad40>{number = 4, name = (null)}
任务1--- <NSThread: 0x1c046ad40>{number = 4, name = (null)}
任务2--- <NSThread: 0x1c046ad40>{number = 4, name = (null)}
任务2--- <NSThread: 0x1c046ad40>{number = 4, name = (null)}

从打印结果可以看到队列里的任务是按串行执行的。
这是因为队列里的任务优先级一样，在只有一个并发队列数的时候，任务按顺序执行。

* 设置`maxConcurrentOperationCount`为`1`，实现并发操作。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let queue = OperationQueue.init()
    //设置最大并发数为1
    queue.maxConcurrentOperationCount = 1

    let bq1 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务1---", Thread.current)
        }
    }
    bq1.queuePriority = .low
    
    let bq2 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务2---", Thread.current)
        }
    }
    bq2.queuePriority = .high
    
    let bq3 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务3---", Thread.current)
        }
    }
    bq3.queuePriority = .normal

    queue.addOperations([bq1, bq2, bq3], waitUntilFinished: false)
    
    print("代码块------结束")
}
```

> currentThread--- <NSThread: 0x1c4261780>{number = 1, name = main}
代码块------begin
代码块------结束
任务2--- <NSThread: 0x1c0279340>{number = 4, name = (null)}
任务2--- <NSThread: 0x1c0279340>{number = 4, name = (null)}
任务3--- <NSThread: 0x1c0279340>{number = 4, name = (null)}
任务3--- <NSThread: 0x1c0279340>{number = 4, name = (null)}
任务1--- <NSThread: 0x1c0279340>{number = 4, name = (null)}
任务1--- <NSThread: 0x1c0279340>{number = 4, name = (null)}

可以我们通过设置优先级`queuePriority`，实现了队列的任务交替执行了。

* 设置`maxConcurrentOperationCount`为`11`，实现并发操作。

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let queue = OperationQueue.init()
    //设置最大并发数为1
    queue.maxConcurrentOperationCount = 11

    let bq1 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务1---", Thread.current)
        }
    }
    
    let bq2 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务2---", Thread.current)
        }
    }

    queue.addOperations([bq1, bq2], waitUntilFinished: false)
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c407a200>{number = 1, name = main}
代码块------begin
代码块------结束
任务2--- <NSThread: 0x1c42647c0>{number = 4, name = (null)}
任务1--- <NSThread: 0x1c04714c0>{number = 3, name = (null)}
任务2--- <NSThread: 0x1c42647c0>{number = 4, name = (null)}
任务1--- <NSThread: 0x1c04714c0>{number = 3, name = (null)}

`maxConcurrentOperationCount`大于`1`的时候，实现了并发操作。

##### 等待执行完成：waitUntilFinished

> `waitUntilFinished`阻塞当前线程，直到该操作结束。可用于线程执行顺序的同步。

比如实现两个并发队列按顺序执行。

```
    @IBAction func onThread() {
        //打印当前线程
        print("currentThread---", Thread.current)
        print("代码块------begin")
        
        //创建并发队列1
        let queue1 = OperationQueue.init()

        let bq1 = BlockOperation.init {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
                print("任务1---", Thread.current)
            }
        }
        
        let bq2 = BlockOperation.init {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
                print("任务2---", Thread.current)
            }
        }

        queue1.addOperations([bq1, bq2], waitUntilFinished: true)
        
        //创建并发队列2
        let queue2 = OperationQueue.init()
        
        let bq3 = BlockOperation.init {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
                print("任务3---", Thread.current)
            }
        }
        
        let bq4 = BlockOperation.init {
            for _ in 0..<2 {
                Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
                print("任务4---", Thread.current)
            }
        }
        
        queue2.addOperations([bq3, bq4], waitUntilFinished: true)
        
        print("代码块------结束")
    }
```

> 打印结果：
> currentThread--- <NSThread: 0x1c407d1c0>{number = 1, name = main}
代码块------begin
任务1--- <NSThread: 0x1c0467d40>{number = 4, name = (null)}
任务2--- <NSThread: 0x1c0460a00>{number = 3, name = (null)}
任务1--- <NSThread: 0x1c0467d40>{number = 4, name = (null)}
任务2--- <NSThread: 0x1c0460a00>{number = 3, name = (null)}
任务3--- <NSThread: 0x1c0467d40>{number = 4, name = (null)}
任务4--- <NSThread: 0x1c0460a00>{number = 3, name = (null)}
任务3--- <NSThread: 0x1c0467d40>{number = 4, name = (null)}
任务4--- <NSThread: 0x1c0460a00>{number = 3, name = (null)}
代码块------结束

通过设置队列的`waitUntilFinished`为`true`，可以看到`queu1`的任务并发执行完了之后，`queue2`的任务才开始并发执行。
而且所有的执行是在`代码块------begin`和`代码块------结束`之间的。`queue1`和`queue2`阻塞了主线程。

##### 操作依赖：addDependency

```
@IBAction func onThread() {
    //打印当前线程
    print("currentThread---", Thread.current)
    print("代码块------begin")
    
    //创建并发队列
    let queue = OperationQueue.init()

    let bq1 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务1---", Thread.current)
        }
    }
    
    let bq2 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务2---", Thread.current)
        }
    }
    
    let bq3 = BlockOperation.init {
        for _ in 0..<2 {
            Thread.sleep(forTimeInterval: 0.2)//执行耗时操作
            print("任务3---", Thread.current)
        }
    }
    
    bq3.addDependency(bq1)
    
    queue.addOperations([bq1, bq2, bq3], waitUntilFinished: false)
    
    print("代码块------结束")
}
```

> 打印结果：
> currentThread--- <NSThread: 0x1c0065740>{number = 1, name = main}
代码块------begin
代码块------结束
任务1--- <NSThread: 0x1c0660300>{number = 5, name = (null)}
任务2--- <NSThread: 0x1c4071340>{number = 6, name = (null)}
任务1--- <NSThread: 0x1c0660300>{number = 5, name = (null)}
任务2--- <NSThread: 0x1c4071340>{number = 6, name = (null)}
任务3--- <NSThread: 0x1c0660300>{number = 5, name = (null)}
任务3--- <NSThread: 0x1c0660300>{number = 5, name = (null)}

__不添加操作依赖__

> 打印结果：
> currentThread--- <NSThread: 0x1c4072c40>{number = 1, name = main}
代码块------begin
代码块------结束
任务1--- <NSThread: 0x1c0270c80>{number = 7, name = (null)}
任务2--- <NSThread: 0x1c447df00>{number = 4, name = (null)}
任务3--- <NSThread: 0x1c0270cc0>{number = 8, name = (null)}
任务1--- <NSThread: 0x1c0270c80>{number = 7, name = (null)}
任务2--- <NSThread: 0x1c447df00>{number = 4, name = (null)}
任务3--- <NSThread: 0x1c0270cc0>{number = 8, name = (null)}

可以看到`任务3`在添加了操作依赖`任务1`，执行就一直等待`任务1`完成。

##### 优先级：queuePriority

> NSOperation 提供了queuePriority（优先级）属性，queuePriority属性适用于同一操作队列中的操作，不适用于不同操作队列中的操作。默认情况下，所有新创建的操作对象优先级都是NSOperationQueuePriorityNormal。但是我们可以通过setQueuePriority:方法来改变当前操作在同一队列中的执行优先级。

```
public enum QueuePriority : Int {

    case veryLow

    case low

    case normal

    case high

    case veryHigh
}
```

* 当一个操作的所有依赖都已经完成时，操作对象通常会进入准备就绪状态，等待执行。

* `queuePriority`属性决定了进入准备就绪状态下的操作之间的开始执行顺序。并且，优先级不能取代依赖关系。

* 如果一个队列中既包含高优先级操作，又包含低优先级操作，并且两个操作都已经准备就绪，那么队列先执行高优先级操作。比如上例中，如果 op1 和 op4 是不同优先级的操作，那么就会先执行优先级高的操作。

* 如果，一个队列中既包含了准备就绪状态的操作，又包含了未准备就绪的操作，未准备就绪的操作优先级比准备就绪的操作优先级高。那么，虽然准备就绪的操作优先级低，也会优先执行。优先级不能取代依赖关系。如果要控制操作间的启动顺序，则必须使用依赖关系。


##### NSOperation常用属性和方法

* 取消操作方法

`open func cancel()`可取消操作，实质是标记`isCancelled`状态。

* 判断操作状态方法

`open var isExecuting: Bool { get }`判断操作是否正在在运行。

`open var isFinished: Bool { get }`判断操作是否已经结束。

`open var isConcurrent: Bool { get }`判断操作是否处于串行。

`open var isAsynchronous: Bool { get }`判断操作是否处于并发。

`open var isReady: Bool { get }`判断操作是否处于准备就绪状态，这个值和操作的依赖关系相关。

`open var isCancelled: Bool { get }`判断操作是否已经标记为取消。	

* 操作同步

`open func waitUntilFinished()`阻塞当前线程，直到该操作结束。可用于线程执行顺序的同步。

`open var completionBlock: (() -> Swift.Void)?`会在当前操作执行完毕时执行 completionBlock。

`open func addDependency(_ op: Operation)`添加依赖，使当前操作依赖于操作 op 的完成。

`open func removeDependency(_ op: Operation)`移除依赖，取消当前操作对操作 op 的依赖。

`open var dependencies: [Operation] { get }`在当前操作开始执行之前完成执行的所有操作对象数组。

`open var queuePriority: Operation.QueuePriority`设置当前操作在队列中的优先级。

##### NSOperationQueue常用属性和方法

* 取消/暂停/恢复操作

`open func cancelAllOperations()`可以取消队列的所有操作。

`open var isSuspended: Bool`判断队列是否处于暂停状态。true为暂停状态，false为恢复状态。可设置操作的暂停和恢复，true代表暂停队列，false代表恢复队列。

* 操作同步

`open func waitUntilAllOperationsAreFinished()`阻塞当前线程，直到队列中的操作全部执行完毕。

* 添加/获取操作

`open func addOperation(_ block: @escaping () -> Swift.Void)` 向队列中添加一个 NSBlockOperation 类型操作对象。

`open func addOperations(_ ops: [Operation], waitUntilFinished wait: Bool)`向队列中添加操作数组，wait 标志是否阻塞当前线程直到所有操作结束。

`open var operations: [Operation] { get }`当前在队列中的操作数组（某个操作执行结束后会自动从这个数组清除）。

`open var operationCount: Int { get }`当前队列中的操作数。

* 获取队列

`open class var current: OperationQueue? { get }`获取当前队列，如果当前线程不是在 NSOperationQueue 上运行则返回 nil。

`open class var main: OperationQueue { get }` 获取主队列。



