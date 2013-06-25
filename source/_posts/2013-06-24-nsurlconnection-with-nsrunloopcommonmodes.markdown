---
layout: post
title: "一个异步网络请求问题：关于NSURLConnection和NSRunLoopCommonModes"
date: 2013-06-24 16:15
comments: true
categories: iOS
---

通常我们开发App时，常常需要异步下载网络资源或者实现REST API调用，目前流行的HTTP库有[ASIHTTPRequest](https://github.com/pokeb/asi-http-request/)（已经停止开发维护）和[AFNetWorking](https://github.com/AFNetworking/AFNetworking)。两者实现异步下载的方式不太相同，ASIHTTPRequest使用的是每个请求独立子线程和CFNetWork API的技术：

```
+ (NSThread *)threadForRequest:(ASIHTTPRequest *)request
{
	if (networkThread == nil) {
		@synchronized(self) {
			if (networkThread == nil) {
				networkThread = [[NSThread alloc] initWithTarget:self selector:@selector(runRequests) object:nil];
				[networkThread start];
			}
		}
	}
	return networkThread;
}
```
AFNetWorking则是使用NSOperation和NSURLConnection的技术实现异步网络通讯，然而它在NSOperation中真正要实现网络请求时，同样生成了一个子线程来执行实际的网络请求：

```
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}
```

<!-- more -->

ASIHTTPRequest和AFNetWorking实际上都是使用子线程配合Run Loop来实现管理网络请求
。由于在开发中需要一个简单的网络图片下载功能，直接使用第三方库太多文件，也不太想使用NSOpertaion以及子线程，后来写了一个类似Sample Code中PTURLDownloader下载类，当时遇到两个tricky的问题。


首先，如果是直接调用NSURLConnection的initWithRequest:delegate:startImmediately:（第三个参数用YES，这个是designated initializer）或者方法initWithRequest:delegate:时，NSURLConnection会运行在NSDefaultRunLoopMode模式下，即使使用scheduleInRunLoop:forMode:再次设置运行模式也没有用。如果NSURLConnection运行在NSDefaultRunLoopMode下，何为Run Loop的模式Mode，请参考这篇[Blog](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)。
这篇Blog也说到NSDefaultRunLoopMode是Run Loop默认的运行模式，用于处理除了NSConnection对象的事件。
然而如果NSURLConnection是运行在NSDefaultRunLoopMode，而当前线程是主线程，并且UI上有类似滚动这样的操作，那么主线程的Run Loop会运行在UITrackingRunLoopMode(仅仅是猜测，还未验证)下，就无法响应NSURLConnnection的回调。此时需要使用下面的方法重新设置NSURLConnection的运行模式是NSRunLoopCommonModes，那么UI操作和回调的执行都将是非阻塞的。

```
- (void)start
{
    NSMutableURLRequest* request = [[NSMutableURLRequest alloc]   
                initWithURL:self.URL
                cachePolicy:NSURLCacheStorageNotAllowed
                timeoutInterval:self.timeoutInterval];
    [request setHTTPMethod: @"GET"];
    self.connection =[[NSURLConnection alloc] initWithRequest:request
                                                     delegate:self
                                             startImmediately:NO];
    [self.connection scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
    [self.connection start];
}

```

其次在调用PTURLDownloader的start方法时，如果是用GCD在其他线程中开始运行：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^(){
        [downloader start];
    });

```
而非在主线程中运行：

```
[downloader performSelectorOnMainThread:@selector(start) 
                             withObject:nil 
                          waitUntilDone:YES];
```

你在GCD的全局队列里运行的PTURLDownloader中的不会得到NSURLConnection回调，而主线程可以得到回调，这是由于在GCD全局队列中执行时没有运行Run Loop，那么NSURLConnection也就无法触发回调了。

当然GCD的启动PTURLDownloader的start方法也是可以的，这样：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^(){
        [downloader start];
        [[NSRunLoop currentRunLoop] run];
    });

```
不过回调会在主线程中被调用，就好像当初是在主线程调用PTURLDownloader的start一样，这一点我还不太明白，您知道的话欢迎直接留言:)。

有人会觉得在主线程调用调用PTURLDownloader的start会不会影响主线程的UI，影响会有，但是网络IO完全不会影响，那是底层操作系统的事情，仅仅是网络IO结束后由主线程来处理回调而已。

###Sample Code
本文例子继续放在了[Github](https://github.com/hrchen/NSURLConnectionExample)上，可以根据文中的几种情况测试initWithRequest:delegate:startImmediately:第三个参数的影响以及回调问题，例子UI中的按钮每点击一次会产生一个下载PTURLDownloader对象并开始执行，如果这个NSURLConnection是运行在NSDefaultRunLoopMode模式下，那么上下滚动UI中的Table
View，是不会触发NSURLConnection回调的，只有UI操作结束后才会触发。


