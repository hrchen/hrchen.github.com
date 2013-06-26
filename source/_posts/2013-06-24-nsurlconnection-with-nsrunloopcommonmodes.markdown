---
layout: post
title: "一个异步网络请求问题：关于NSURLConnection和NSRunLoopCommonModes"
date: 2013-06-24 16:15
comments: true
categories: iOS
---

我们开发App时，常常需要异步下载网络资源或者实现REST API调用，目前流行的HTTP库有[ASIHTTPRequest](https://github.com/pokeb/asi-http-request/)（已经停止开发维护）和[AFNetWorking](https://github.com/AFNetworking/AFNetworking)。两者实现异步下载的方式不太相同，ASIHTTPRequest使用的是一个公共独立子线程和CFNetWork API的技术：

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
AFNetWorking则是包装了NSOperation和NSURLConnection技术实现异步网络通讯，然而它在NSOperation中真正启动NSURLConnection网络请求时，同样生成了一个公共独立子线程来Kick off网络请求：

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

ASIHTTPRequest和AFNetWorking实际上都是使用一个公共独立子线程配合Run Loop来实现管理网络请求
。由于我在开发中需要一个简单的网络图片下载功能，直接使用第三方库需要添加太多文件，就写了一个类似Sample Code中PTNormalDownloaler下载类，当时遇到个tricky的问题。

首先，如果是直接调用NSURLConnection的initWithRequest:delegate:startImmediately:（第三个参数用YES，这个是designated initializer）或者方法initWithRequest:delegate:时，NSURLConnection会默认运行在NSDefaultRunLoopMode模式下，即使再使用scheduleInRunLoop:forMode:设置运行模式也没有用。如果NSURLConnection运行在NSDefaultRunLoopMode下，何为Run Loop的模式Mode，请参考这篇[Blog](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)），
这篇Blog提到NSDefaultRunLoopMode是Run Loop默认的运行模式，用于处理除了NSConnection对象的事件。
然而如果NSURLConnection是运行在NSDefaultRunLoopMode，而当前线程是主线程，并且UI上有类似滚动这样的操作，那么主线程的Run Loop会运行在UITrackingRunLoopMode下，就无法响应NSURLConnnection的回调。此时需要首先使用initWithRequest:delegate:startImmediately:（第三个参数为NO）生成NSURLConnection，再重新设置NSURLConnection的运行模式为NSRunLoopCommonModes，那么UI操作和回调的执行都将是非阻塞的，因为NSRunLoopCommonModes是一组run loop mode的集合，默认情况下包含了NSDefaultRunLoopMode和UITrackingRunLoopMode。

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

其次在调用PTNormalDownloaler的start方法时，如果是用GCD在其他线程中开始运行：

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

你在GCD的全局队列里运行的PTNormalDownloaler中的不会得到NSURLConnection回调，而从主线程中启动NSURLConnection可以得到回调，这是由于在GCD全局队列中执行时没有运行Run Loop，那么NSURLConnection也就无法触发回调了。

当然在GCD的全局队列里启动NSURLConnection需要这样这样：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^(){
        [downloader start];
        NSLog(@"current worker thread: %@", [NSThread currentThread]);
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        NSLog(@"exit worker thread");
    });

```

如果从主线程启动NSURLConntection，其回调会在主线程中被调用：

```
2013-06-26 19:35:13.309 NSURLConnectionExample[22646:50b] connectionDidFinishLoading in main thread?: 1
```

如果是在子线程启动NSURLConntection，其回调则会在子线程中被调用：

```
2013-06-26 19:38:18.937 NSURLConnectionExample[22670:3903] connectionDidFinishLoading in main thread?: 0
```

在主线程启动NSURLConnection会不会影响主线程的UI？影响肯定会有，但是网络IO本身不会影响，那是底层操作系统的事情，仅仅是网络IO结束后由主线程来处理回调而已。

样例程序里也实现了一个基于子线程的下载器PTThreadDownloader，与ASIHTTPRequest和AFNetWorking中的方法类似，生成一个公共子线程来启动NSURLConnection。这里没有必要对每个网络请求都生成一个后台子线程去启动NSURLConnection（例如上面那种用GCD扔到global queue的方式最终也是每次从线程池找到一个线程来启动NSURLConnection），因为底层网络IO并不是在这个子线程里去执行的，子线程仅仅用于响应NSURLConnection回调。不过公共子线程的方法会导致有一个子线程一直运行在后台，等待用户用它来启动NSURLConnection。


###Sample Code
本文例子放在[Github](https://github.com/hrchen/ExamplesForBlog)上（工程NSURLConnectionExample），可以根据文中的几种情况测试initWithRequest:delegate:startImmediately:第三个参数的影响以及回调问题，例子UI中的按钮每点击一次会产生一个下载PTNormalDownloaler对象并开始执行，如果这个NSURLConnection是运行在NSDefaultRunLoopMode模式下，那么上下滚动UI中的Table View，是不会触发NSURLConnection回调的，只有UI操作结束后才会触发。


