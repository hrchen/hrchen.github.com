---
layout: post
title: "iOS多线程编程Part 3/3 - GCD"
date: 2013-07-02 21:20
comments: true
categories: iOS
---

前两部分介绍了NSThread、NSRunLoop和NSOperation，本文聊聊2011年WWDC时推出的神器GCD。GCD: Grand Central Dispatch，是一组用于实现并发编程的C接口。GCD是基于Objective-C的Block特性开发的，基本业务逻辑和NSOperation很像，都是将工作添加到一个队列，由系统来负责线程的生成和调度。由于是直接使用Block，因此比NSOperation子类使用起来更方便，大大降低了多线程开发的门槛。另外，GCD是开源的喔：[libdispatch](http://libdispatch.macosforge.org/)

###基本用法
首先示例：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doTask];
    NSLog(@"Fisinished");
});
```
GCD的调用接口非常简单，就是将Job提交至Queue中，主要的提交Job接口为：

* dispatch_sync(queue, block)同步提交job* dispatch_async (queue, block) 异步提交job* dispatch_after(time, queue, block) 同步延迟提交job
其中第一个参数类型是dispatch_queue_t，就是一个表示队列的数据结构`typedef struct dispatch_queue_s *dispatch_queue_t;`；block就是表示任务的Block`typedef void (^dispatch_block_t)( void);`。

dispatch_async函数是异步非阻塞的，调用后会立刻返回，工作由系统在线程池中分配线程去执行工作。
dispatch_sync和dispatch_after是阻塞式的，会一直等到添加的工作完成后才会返回。

除了添加Block到Dispatch Queue，iOS 4之后新增了添加函数到Dispatch Queue的接口，例如dispatch_async对应的有dispatch_async_f：

```
dispatch_async_f(dispatch_queue_t queue,
	             void *context,
	             dispatch_function_t work);
```
其中第三个参数就是个函数指针，即`typedef void (*dispatch_function_t)(void *);`；第二个参数是传给这个函数的参数。

<!--more-->

###Dispatch Queue
要添加工作到队列Dispatch Queue中，这个队列可以是串行或者并行的，并行队列会尽可能的并发执行其中的工作任务，而串行队列每次只能运行一个工作任务。


目前GCD中有三种类型的Dispatch Queue：

* Main Queue：关联到主线程的队列，可以使用函数dispatch_get_main_queue()获得，加到这个队列中的工作都会分发到主线程运行。主线程只有一个，因此很明显这个是串行队列，每次运行一个工作。
* Global Queue：全局队列是并发队列，又根据优先级细分为高优先级、默认优先级和低优先级三种。通过dispatch_get_global_queue加上优先级参数获得这个全局队列，例如`dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)`
* 自定义Queue：自己创建一个队列，通过函数dispatch_queue_create创建，例如`dispatch_queue_create("com.kiloapp.test", NULL)`。第一个参数是队列的名字，Apple建议使用反DNS型的名字命名，防止重名；第二个参数是创建的queue的类型，iOS 4.3以前只支持串行，即DISPATCH_QUEUE_SERIAL(就是NULL)，iOS4.3以后也开始支持并行队列，即参数DISPATCH_QUEUE_CONCURRENT。

由于有这些种不同类型的队列，一种常见的使用模式是：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doHardWorkInBackground];
    dispatch_async(dispatch_get_main_queue(), ^{
        [self updateUI];
    });
});
```
将一些耗时的工作添加到全局队列，让系统分配线程去做，工作完成后再次调用GCD的主线程队列去完成UI相关的工作，这样做就不会因为大量的非UI相关工作加重主线程负担，从而加快UI事件响应。


###Dispatch Group

GCD确实非常简单好用，不过有些场景下还是有点问题，例如：

```
for(id obj in array)
{
    [self doWorkOnItem:obj];
}
[self doWorkOnArray:array];
```

前半部分可以用GCD得到处理性能的提升：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for(id obj in array)
    dispatch_async(queue, ^{
        [self doWorkOnItem:obj];
    });
[self doWorkOnArray:array];
```
问题是`[self doWorkOnArray:array];`原先是在全部数组各个成员的工作完成后才会执行的，现在由于dispatch_async是异步的，`[self doWorkOnArray:array];`很有可能在各个成员的工作完成前就开始运行，这明显不符合原先的语义。如果将dispatch_async改成dispatch_sync可以解决问题，但是和原来的方法一样没有并行处理数组，使用GCD也就没有意义了。

针对这种情况，GCD提供了Dispatch Group可以将一组工作集合在一起，等待这组工作完成后再继续运行。dispatch_group_create函数可以用来创建这个Group：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doWorkOnItem:obj];
    });
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
dispatch_release(group);
[self doWorkOnArray:array];
```

方法是不是很简单，将并发的工作用dispatch_group_async异步添加到一个Group和全局队列中，dispatch_group_wait会等待这些工作完成后再返回，这样你就可以再运行`[self doWorkOnArray:array];`。

不过有点不好的是dispatch_group_wait会阻塞当前线程，如果当前是主线程岂不是不好，有更绝的dispatch_group_notify接口：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doWorkOnItem:obj];
    });
dispatch_group_notify(group, queue, ^{
    [self doWorkOnArray:array];
});
dispatch_release(group);
```

dispatch_group_notify函数可以将这个Group完成后的工作也同样添加到队列中（如果是需要更新UI，这个队列也可以是主队列），总之这样做就完全不会阻塞当前线程了。

Dispatch Group还有两个接口可以显式的告知group要添加block操作：
dispatch_group_enter(group)和dispatch_group_leave(group)，这两个接口的调用数必须平衡，否则group就无法知道是不是处理完所有的Block了。


###Dispatch Apply

如果就是要同步的执行对数组元素的逐个操作，GCD也提供了一个简便的dispatch_apply函数：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index){
    [self doWorkOnItem:obj:[array objectAtIndex:index]];
});
[self doWorkOnArray:array];
```

###Dispatch Barrier

在使用dispatch_async异步提交时，是无法保证这些工作的执行顺序的，如果需要某些工作在某个工作完成后再执行，那么可以使用Dispatch Barrier接口来实现，barrier也有同步提交dispatch_barrier_async(queue, block)和异步提交dispatch_barrier_sync(queue, block)两种方式。例如：

```
dispatch_async(queue, block1);
dispatch_async(queue, block2);
dispatch_barrier_async(queue, block3);
dispatch_async(queue, block4);
dispatch_async(queue, block5);
```
dispatch_barrier_async是异步的，调用后立刻返回，即使block3到了队列首部，也不会立刻执行，而是等到block1和block2的并行执行完成后才会执行block3，完成后再会并行运行block4和block5。注意这里的queue应该是一个并行队列，否则dispatch_barrier_async操作就失去了意义。

###Dispatch Source

Run Loop有Input Source，GCD也同样支持一系列事件监听和处理，GCD有一组Dispatch Source接口可以监听底层系统对象(例如文件描述符、网络描述符、Mach Port、Unix信号、VFS文件系统的vnode等)的事件，可以设置这些事件的处理函数，如果事件发生时，Dispatch Source就可以将事件的处理方法提交到队列中执行。

dispatch_source_t是Dispatch Source的数据结构，使用dispatch_source_create(type, handle, mask, queue)来创建，第一个参数是source的类型：

```
#define DISPATCH_SOURCE_TYPE_DATA_ADD
#define DISPATCH_SOURCE_TYPE_DATA_OR
#define DISPATCH_SOURCE_TYPE_MACH_RECV
#define DISPATCH_SOURCE_TYPE_MACH_SEND
#define DISPATCH_SOURCE_TYPE_PROC
#define DISPATCH_SOURCE_TYPE_READ
#define DISPATCH_SOURCE_TYPE_SIGNAL
#define DISPATCH_SOURCE_TYPE_TIMER
#define DISPATCH_SOURCE_TYPE_VNODE
#define DISPATCH_SOURCE_TYPE_WRITE
```
第二个参数handle和第三个参数mask与source的类型相关，有不同的含义，第四个参数是source绑定的queue，由于篇幅问题这些含义请参考《Grand Central Dispatch (GCD) Reference》。

dispatch_source_set_event_handler(source, handler)接口可以添加source的处理方法handler，这里的handler是一个block。如果是dispatch_source_set_event_handler_f(source, handler)，这里的handler就是function。

举个自定义源的例子，假如我们在处理上面那个数组时要在UI中显示一个进度条：

```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_get_main_queue());

dispatch_source_set_event_handler(source, ^{
    [progressIndicator incrementBy:dispatch_source_get_data(source)];
});
dispatch_resume(source);
    
dispatch_apply([array count], globalQueue, ^(size_t index) {
    [self doWorkOnItem:obj:[array objectAtIndex:index]];
    dispatch_source_merge_data(source, 1);
});
```
dispatch source创建后是出于suspend状态的，必须使用dispatch_resume来恢复，dispatch_apply中每处理一个数组元素会调用dispatch_source_merge_data加1，那么这个source的事件handler就可以通过dispatch_source_get_data拿到source的数据。

###Dispatch Once

dispatch_once的意思是在App整个生命周期内运行并且只允许一次，类似于pthread库中的[pthread_once](http://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_once.html))。由于dispatch_once的调试非常困难，所以最好还是少用，单例应该是少数值得用的地方了。

传统我们实现单例是这样：

```
+ (id)sharedManager
{
    static Manager *theManager = nil;
    @synchronized([Manager class])
    {
        if(!theManager)
            theManager = [[Manager alloc] init];
    }
    return theManager;
}

```
这个的成本还是有点高，每次访问都会有同步锁，使用dispatch_once可以保证只运行一次初始化：

```
+ (id)sharedWhatever
{
    static dispatch_once_t pred;
    static Manager *theManager = nil;
    dispatch_once(&pred, ^{
        theManager = [[Manager alloc] init];
    });
    return theManager;
}
```
需要注意dispatch_once_t最好使用全局变量或者是static的，否则可能导致无法确定的行为。

###Dispatch Semaphore

和其他多线程技术一样，GCD也支持信号量，dispatch_semaphore_create用于创建，dispatch_semaphore_signal用于通知，dispatch_semaphore_wait用于等待。


###队列Context数据

###Dispatch I/O Channel

###Dispatch Data 对象


###GCD的坑

GCD的常规使用方法很简单，但是同样有很多坑需要绕开：

* 死锁
* 优先级问题


###总结

GCD的API按功能分为：

* 创建管理Queue* 提交Job* Dispatch Group* 管理Dispatch Object* 信号量Semaphore* 队列屏障Barrier* Dispatch Source* Queue Context数据* Dispatch I/O Channel* Dispatch Data 对象

各组接口的详细说明还是参考《Grand Central Dispatch (GCD) Reference》。

###参考资料
[Grand Central Dispatch (GCD) Reference](https://developer.apple.com/library/mac/#documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html)

[Blocks Programming Topics](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html)
