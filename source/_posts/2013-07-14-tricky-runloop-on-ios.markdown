---
layout: post
title: "iOS中Run Loop的那些坑"
date: 2013-07-14 21:20
comments: true
categories: iOS
---

前段时间写了个关于iOS多线程编程的系列：

* [iOS多线程编程Part 1/3 - NSThread & Run Loop](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)
* [iOS多线程编程Part 2/3 - NSOperation](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-2/)
* [iOS多线程编程Part 3/3 - GCD](http://www.hrchen.com/2013/07/multi-threading-programming-of-ios-part-3/)

[iOS多线程编程Part 1/3 - NSThread & Run Loop](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)中讨论了Run Loop的机制、接口和需要注意的坑，不过由于内容较多，描述Run Loop的坑写得很零散，本文做一个集中归纳。

{% img /images/post/runloop_source.jpg %}

回顾一下，Run Loop就是个监听事件的循环，会不停的检查它的事件源(Timer和Input Source)有没有事件发生，如果有事件发生就处理事件或者调用事件的处理方法。

###坑一
在线程中添加Timer时，肯定需要先生成Timer对象啦，有类方法也有实例方法，如果是使用scheduledTimerWith***接口生成的Timer对象，会自动添加到当前线程的NSDefaultRunLoopMode中；如果是其他接口生成的Timer对象，则需要用`-addTimer:forMode`添加Timer，这样做的好处是可以指定添加Timer的Run Loop以及模式。

<!-- more -->

###坑二
如果Run Loop中添加的是Timer而没有其他Input Source，而这个Timer只运行一次，那么Timer事件触发后Timer事件源就会从Run Loop删除，那么再运行Run Loop就会立刻返回；同时Timer事件触发是不会让Run Loop返回的，即使使用CF层的CFRunLoopRef运行接口`SInt32 CFRunLoopRunInMode (mode, seconds, returnAfterSourceHandled); `运行Run Loop，其第三个参数为YES，Timer事件触发仍然不会导致当前Run Loop的运行返回。

###坑三
如果是使用NSRunLoop，有三个运行的接口：

```
//运行 NSRunLoop，运行模式为默认的NSDefaultRunLoopMode模式，没有超时限制
- (void)run;

//运行 NSRunLoop: 参数为运行模式、时间期限，返回值为YES表示是处理事件后返回的，NO表示是超时或者停止运行导致返回的- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate;
//运行 NSRunLoop: 参数为运时间期限，运行模式为默认的NSDefaultRunLoopMode模式 
- (void)runUntilDate:(NSDate *)limitDate;
```

建议是使用第二个接口来运行，因为它能够设置Run Loop的运行参数最多，而且最重要的是可以使用CFRunLoopStop(runLoopRef)来停止Run Loop的运行，而第一个和第三个接口无法使用CFRunLoopStop(runLoopRef)来停止Run Loop的运行。

###坑四
在使用NSURLConnection或者NSStream时，需要考虑到Run Loop的问题，默认情况下这两个对象生成后都是运行在当前线程的NSDefaultRunLoopMode模式的，如果是在主线程，那么在滚动ScrollView或者TableView时，主线程的Run Loop会运行在UITrackingRunLoopMode模式，那么NSURLConnection或者NSStream的回调就无法运行。因此最好是指定NSURLConnection或NSStream在Run Loop中的运行模式，两者有相同的接口`- (void)scheduleInRunLoop:(NSRunLoop *)aRunLoop forMode:(NSString *)mode;`来设置NSURLConnection和NSStream的Run Loop以及模式，而且设置的Mode要设置为NSRunLoopCommonModes，因为NSRunLoopCommonModes默认会包含NSDefaultRunLoopMode和UITrackingRunLoopMode，这样无论Run Loop运行在哪个模式，都可以保证NSURLConnection或者NSStream的回调可以被调用。另外如果是在子线程中你设置了自定义的Run Loop模式，还可以用接口`CFRunLoopAddCommonMode(runLoopRef, mode)`添加到NSRunLoopCommonModes。

不过NSURLConnection的使用有点特殊，必须使用它的designated initializer `- (id)initWithRequest:(NSURLRequest *)request delegate:(id)delegate startImmediately:(BOOL)startImmediately `生成NSURLConnection对象，而且第三个参数是否立刻启动NSURLConnection要设置为NO，之后再用`- (void)scheduleInRunLoop:(NSRunLoop *)aRunLoop forMode:(NSString *)mode;`设置Run Loop与模式，再调用`[NSURLConnectionObject start]`启动。如果是其他接口生成NSURLConnection，用`- (void)scheduleInRunLoop:(NSRunLoop *)aRunLoop forMode:(NSString *)mode;`设置Run Loop和mode都不会起作用。


