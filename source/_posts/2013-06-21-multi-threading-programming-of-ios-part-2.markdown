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


###NSOperation & NSOperationQueue
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



###NSBlockOperation & NSInvocationOperation

###Sample Code
看的晕乎乎？理解概念最好的方式当然还是动手写代码，写了个例子放在[GitHub](https://github.com/hrchen/NSThreadExample)上，欢迎讨论。

Apple官方也有一个基于Run Loop的异步网络连接样例程序[SimpleURLConnections](http://developer.apple.com/library/ios/#samplecode/SimpleURLConnections/Listings/Read_Me_About_SimpleURLConnections_txt.html)，值得一看。


###参考文献
[Threading Programming Guide](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/Multithreading/CreatingThreads/CreatingThreads.html)
