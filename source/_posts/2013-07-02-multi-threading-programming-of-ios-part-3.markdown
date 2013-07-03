---
layout: post
title: "iOS多线程编程Part 3/3 - GCD"
date: 2013-07-02 21:20
comments: true
categories: iOS
---

前两部分介绍了NSThread、NSRunLoop和NSOperation的基本支持，本文聊聊iOS4发布时推出的神器GCD。


###前言
GCD: Grand Central Dispatch，是一组用于实现并发编程的C接口。GCD是完全基于Objective-C的Block特性开发的，基本调用逻辑和NSOperation很像，都是将工作添加到一个队列，由系统来负责线程的生成和调度。由于是直接使用Block，因此比NSOperation更加方便，大大降低了多线程开发的门槛。示例：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doTask];
    NSLog(@"Fisinished");
});
```

另外，GCD是开源的喔：[libdispatch](http://libdispatch.macosforge.org/)

###Dispatch Queue
要添加工作到队列Dispatch Queue中，这个队列可以是串行或者并行的，并行队列会尽可能的并发执行其中的工作任务，而串行队列每次只能运行一个工作任务。

<!--more-->

目前GCD中有三种类型的Dispatch Queue：

* Main Queue：关联到主线程的队列，可以使用函数dispatch_get_main_queue()获得，加到这个队列中的工作都会分发到主线程运行。主线程只有一个，因此很明显这个是串行队列，每次运行一个工作。
* Global Queue：全局队列是并发队列，又根据优先级细分为高优先级、默认优先级和低优先级三种。通过dispatch_get_global_queue加上优先级参数获得这个全局队列，例如`dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)`
* 自定义Queue：自己创建一个队列，只能是串行队列，可以理解为最终有一个独立的子线程帮你运行添加到这个队列中的工作任务。通过函数dispatch_queue_create创建，例如`dispatch_queue_create(@"com.kiloapp.test", 0)` ,第二个参数仅作保留，目前没有意义，第一个参数是队列的名字，Apple建议使用反DNS型的名字命名，防止重名。

###添加工作任务
添加工作任务到队列也非常简单，调用函数dispatch_async()，两个参数，一个就是Dispatch Queue，另一个是一个包含工作的Block，就像本文开头的示例一样。dispatch_async函数是非阻塞的，调用后会立刻返回，工作由系统分配线程去执行工作。因此另一种常见的使用模式是：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doHardWorkInBackground];
    dispatch_async(dispatch_get_main_queue(), ^{
        [self updateUI];
    });
});
```
可以将一些耗时的工作添加到全局队列，让系统分配线程在后台中去做，完成后，再次调用GCD的主队列去完成UI相关的工作，这样做就不会因为大量的非UI相关工作加重主线程负担，加快UI事件响应。

与dispatch_async对应的有一个dispatch_sync方法，它是阻塞式的，会一直等到添加的工作完成后才会返回。

NSOperation是没法直接使用的，它只是提供了一个工作的基本逻辑，具体实现还是需要你通过定义自己的NSOperation子类来获得。如果有必要也可以不将NSOperation加入到一个NSOperationQueue中去执行，直接调用起`-start`也可以直接执行。

###Dispatch Group

GCD确实非常简单好用，不过有些情况还是有点问题，例如：

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
问题是`[self doWorkOnArray:array];`原先是在全部数组各个成员的工作完成后才会执行的，现在由于dispatch_async是非阻塞的，`[self doWorkOnArray:array];`很有可能在各个成员的工作完成前就运行了，这明显不符合我们的目的。如果将dispatch_async改成dispatch_sync可以解决问题，但是和原来的方法一样失去了并行的好处，也没有意义了。

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

方法是不是很简单，将并发的工作用dispatch_group_async添加到一个Group和全局队列中，dispatch_group_wait会等待这些工作完成后再返回，这样你就可以再运行`[self doWorkOnArray:array];`。

不过有点不好的是dispatch_group_wait会阻塞当前线程，如果当前是主线程岂不是不好，有更绝的：

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

如果就是要同步的执行对数组元素的逐个操作，GCD也提供了一个简便的dispatch_apply函数：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index){
    [self doWorkOnItem:obj:[array objectAtIndex:index]];
});
[self doWorkOnArray:array];
```

要了解更全的接口，查看《Grand Central Dispatch (GCD) Reference》。

###其他有趣的特性

* Dispatch Source

Run Loop有Input Source，GCD也同样支持一系列事件，就是监听事件发生后会执行一个Block形式的Handler。Dispatch Source支持的事件源类型有：Timer源、signal信号源、描述符(文件或者网络描述符)源、进程源、Port源、自定义源。当然有些源由于iOS系统原因肯定是无法使用的，例如进程源、Port源、、signal信号源。

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

* 单例

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
这个的成本还是有点高，每次访问都会有同步锁，而GCD有个dispatch_once方法(类似于[pthread_once](http://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_once.html))可以保证只运行一次初始化：

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

* 信号量Semaphore

和其他多线程技术一样，GCD也支持信号量，dispatch_semaphore_create用于创建，dispatch_semaphore_signal用于通知，dispatch_semaphore_wait用于等待。


###参考资料
[Grand Central Dispatch (GCD) Reference](https://developer.apple.com/library/mac/#documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html)

[Blocks Programming Topics](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html)
