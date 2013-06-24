---
layout: post
title: "iOS多线程编程Part 2/3 - NSOperation"
date: 2013-06-02 23:11
comments: true
categories: iOS
published:false
---
###前言
上一篇Blog介绍了NSThread创建线程以及线程中令人迷惑的Run Loop概念，这篇Blog介绍下另一种多线程编程技术：NSOperation。


###NSThread
使用NSThead创建线程有很多方法：

* +detachNewThreadSelector:toTarget:withObject:类方法直接生成一个子线程

```objc
[NSThread detachNewThreadSelector:@selector(threadRoutine:) toTarget:self withObject:nil];
```

* 创建一个NSThread类实例，然后调用start方法。

```objc
NSThread* aThread = [[NSThread alloc] initWithTarget:self selector:@selector(threadRoutine:) object:nil];
[aThread start]; 
```

* 调用NSObject的`+performSelectorInBackground:withObject:`方法生成子线程。

```objc
[myObj performSelectorInBackground:@selector(threadRoutine:) withObject:nil];
```

* 创建一个NSThread子类，然后调用子类实例的start方法。

<!-- more -->

创建线程也是有开销的，iOS下主要成本包括构造内核数据结构（大约1KB）、栈空间（子线程512KB、主线程1MB，不过可以使用方法`-setStackSize:`自己设置，注意必须是4K的倍数，而且最小是16K），创建线程大约需要90毫秒的创建时间。

第二种和第四种方法创建的线程有个好处是拥有线程的对象，因此可以使用`performSelector:onThread:withObject:waitUntilDone:`在该线程上执行方法，这是一种非常方便的线程间通讯的方法（相对于设置麻烦的NSPort用于通讯），所要执行的方法可以直接添加到目标线程的Runloop中执行。Apple建议使用这个接口运行的方法不要是耗时或者频繁的操作，以免导致子线程负载过重。

第三种方法其实与第一种方法是一样的，都会直接生成一个子线程。

上面四种方法生成的子线程都是detached状态，即主线程结束时这些线程都会被直接杀死；如果要生成joinable状态的子线程，只能使用pthread接口啦。

如果在线程的routine入口方法或者拥有线程对象时，还可以设置线程的优先级(`-setThreadPriority:`)，如果需要在线程中保存一些状态信息，还可以使用到`-threadDictionary`得到一个NSMutableDictionary，以key-value的方式保存信息用于线程内读写。

###NSThread的入口方法
要写一个有效的子线程入口方法需要注意很多问题，以下面代码举例：


```

- (void)threadRoutine
{
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; 
 	
	BOOL moreWorkToDo = YES;
    BOOL exitNow = NO;
    NSRunLoop* runLoop = [NSRunLoop currentRunLoop];
 
    NSMutableDictionary* threadDict = [[NSThread currentThread] threadDictionary];
    [threadDict setValue:[NSNumber numberWithBool:exitNow] forKey:@"ThreadShouldExitNow"];
 
    [self myInstallCustomInputSource];
 
    while (moreWorkToDo && !exitNow)
    {
        //执行线程真正的工作方法，如果完成了可以设置moreWorkToDo为False
 
        [runLoop runUntilDate:[NSDate distantFuture]];

        exitNow = [[threadDict valueForKey:@"ThreadShouldExitNow"] boolValue];
    }
 
    [pool release];  
}

```

* 必须创建一个NSAutoreleasePool，因为子线程不会自动创建。同时要注意这个pool因为是最外层pool，如果线程中要进行长时间的操作生成大量autoreleased的对象，则只有在该子线程退出时才会回收，因此如果线程中会大量创建autoreleased对象，那么需要创建额外的NSAutoreleasePool，可以在NSRunloop每次迭代时创建和销毁一个NSAutoreleasePool。
* 如果你的子线程会抛出异常，最好在子线程中设置一个异常处理函数，因为如果子线程无法处理抛出的异常，会导致程序直接Crash关闭。
* (可选)设置Run Loop，如果子线程只是做个一次性的操作，那么无需设置Run Loop；如果子线程进入一个循环需要不断处理一些事件，那么设置一个Run Loop是最好的处理方式，如果需要Timer，那么Run Loop就是必须的。
* 如果需要在子线程运行的时候让子线程结束操作，子线程每次Run Loop迭代中检查相应的标志位来判断是否还需要继续执行，可以使用threadDictionary以及设置Input Source的方式来通知这个子线程。那么什么是RunLoop呢？

###RunLoop
NSRunLoop一直都是个比较让人迷惑的概念，相关的介绍资料也很少，主要的特性如下：

* 每个线程都有一个Run Loop，主线程的Run Loop会在App运行时被运行，子线程需要手动设置运行。
* 每个Run Loop都会以一个模式mode来运行，，可以使用`- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate` 来运行某个特定模式mode。
* Run Loop的主要作用是监听Timer和Input Source，每个source都会绑定在Run Loop的某个特定模式mode上，而且只有RunLoop在这个模式运行的时候才会触发该Timer和Input Source。

Run Loop的作用是什么呢？在上一节NSThread的入口函数中已经说明了一种NSRunLoop的使用场景，下面再看一例：

```
- (void)main
{
    @autoreleasepool {
        NSLog(@"starting thread.......");
        NSTimer *timer = [NSTimer timerWithTimeInterval:2 target:self selector:@selector(doTimerTask) userInfo:nil repeats:YES];
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
        [timer release];
        while (! self.isCancelled) {
            [self doOtherTask];
            BOOL ret = [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
            NSLog(@"after runloop counting.........: %d", ret);
        }
        NSLog(@"finishing thread.........");
    }
}

- (void)doTimerTask
{
    NSLog(@"do timer task");
}

- (void)doOtherTask
{
    NSLog(@"do other task");
}
```

我们看到入口方法里创建了一个NSTimer，并且以NSDefaultRunLoopMode模式加入到当前子线程的NSRunLoop中，正常情况肯定会执行`-doOtherTask`方式法一次，然后再以NSDefaultRunLoopMode模式运行NSRunLoop，你觉得他会返回吗？答案是不会。NSRunLoop的底层实现是CFRunLoop，内部实现不清楚，感觉应该类似Linux下Select实现机制，当在Timer执行一次后，Run Loop会让子线程进入sleep等待状态，当有新的Timer Source或者Input Source事件发生时，子线程才会被唤醒，继续执行一次触发的Timer事件，但是Timer source比较特殊，存在Timer Source时，`- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate;`方法自身仍然不会返回，有趣吧。如果在生成Timer时repeats参数设置为NO，那么`-runMode:BeforDate`方法在处理以此Timer事件会返回YES，之后由于没有了Timer及其他事件源，所以NSRunLoop运行`-runMode:BeforDate`方法会立刻返回NO，不再等待事件，打印出来的Log像下面这样：

```
2013-06-18 14:59:44.447 CtripTest[13456:3007] after runloop counting.........: 0
2013-06-18 14:59:44.448 CtripTest[13456:3007] do other task
2013-06-18 14:59:44.448 CtripTest[13456:3007] after runloop counting.........: 0
2013-06-18 14:59:44.448 CtripTest[13456:3007] do other task
2013-06-18 14:59:44.449 CtripTest[13456:3007] after runloop counting.........: 0
2013-06-18 14:59:44.449 CtripTest[13456:3007] do other task
2013-06-18 14:59:44.450 CtripTest[13456:3007] after runloop counting.........: 0
2013-06-18 14:59:44.450 CtripTest[13456:3007] do other task
```

当你将子线程的Run Loop运行在一个模式时，如果该模式下没有事件源，运行Run Loop会立刻返回NO。如果你添加一个Input Source，例如`[[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];`或者持有子线程的对象在这个子线程上运行方法，例如`[self performSelector:@selector(doThreadTask) onThread:thread withObject:nil waitUntilDone:NO];`就会让这个Run Loop进入等待状态，即`-runMode:BeforDate`处于阻塞状态不返回。

归根结底，Run Loop就是个处理事件的机制，事件源分两类：Timer Source和Input Source(包括-performSelector:***API调用簇，Port Input Source、自定义Input Source)。

{% img /images/post/runloop_source.jpg %}

1) Timer Souce就是创建Timer添加到Run Loop中，没啥好说的，Cocoa或者Core Foundation都有相应接口实现。

2) Input Source中的-performSelector:***API调用簇方法，有以下这些接口：

```
performSelectorOnMainThread:withObject:waitUntilDone:
performSelectorOnMainThread:withObject:waitUntilDone:modes:

performSelector:onThread:withObject:waitUntilDone:
performSelector:onThread:withObject:waitUntilDone:modes:

performSelector:withObject:afterDelay:
performSelector:withObject:afterDelay:inModes:

cancelPreviousPerformRequestsWithTarget:
cancelPreviousPerformRequestsWithTarget:selector:object:

```

这些API最后两个是取消当前线程中调用，其他API是在主线程或者当前线程下一个Run Loop循环中执行指定的@selector。

3) Port Input Source：概念上也比较简单，可以用NSMachPort作为线程之间的通讯通道。例如在主线程创建子线程时传入一个NSPort对象，这样主线程就可以和这个子线程通讯啦，如果要实现双向通讯，那么子线程也需要回传给主线程一个NSPort。

NSPort的子类除了NSMachPort，还可以使用NSMessagePort或者Core Foundation中的CFMessagePortRef。

#### 注意：虽然有这么棒的方式实现线程间通讯方式，但是估计由于危及iOS的Sandbox沙盒环境，所以这些API都是私有接口，如果你用到NSPortMessage，XCode会提示`'NSPortMessage' for instance message is a forward declaration`。

4) 自定义Input Source：

向Run Loop添加自定义Input Source只能使用Core Foundation的接口：`CFRunLoopSourceCreate`创建一个source，`CFRunLoopAddSource`向Run Loop中添加source，`CFRunLoopRemoveSource`从Run Loop中删除source，`CFRunLoopSourceSignal`通知source，`CFRunLoopWakeUp`唤醒Run Loop。

Apple官方文档提供了一个自定义Input Source使用模式。

{% img /images/post/input_source.jpg %}

主线程持有子线程的Run Loop和Source，还有一个用于保存需要运行操作的数据buffer。主线程需要子线程干活时，首先将需要的操作数据添加到数据buffer，然后通知source，唤醒子线程Run Loop（因为子线程可能正在sleep状态，`CFRunLoopWakeUp`唤醒Run Loop可以通知线程醒来干活），由于子线程也持有这个source和数据buffer，因此在触发唤醒时可以使用这个数据buffer的数据来执行相关操作（需要注意数据buffer访问时的同步）。

具体的实现参见下面的Sample Code。

###什么时候需要用Run Loop

官方文档的建议是：
* 需要使用port或者自定义Input Source与其他线程进行通讯。
* 需要在线程中使用Timer。
* 需要在线程上使用performSelector***方法。
* 需要让线程执行周期性的工作。

子线程运行过程中被Kill掉了怎么办？加一个run loop observer到线程的Run Loop中，这也线程

###Sample Code
看的晕乎乎？理解概念最好的方式当然还是动手写代码，写了个例子放在[GitHub](https://github.com/hrchen/NSThreadExample)上，欢迎讨论。

Apple官方也有一个基于Run Loop的异步网络连接样例程序[SimpleURLConnections](http://developer.apple.com/library/ios/#samplecode/SimpleURLConnections/Listings/Read_Me_About_SimpleURLConnections_txt.html)，值得一看。


###参考文献
[Threading Programming Guide](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/Multithreading/CreatingThreads/CreatingThreads.html)
