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

ASIHTTPRequest和AFNetWorking实际上都里用了子线程配合Run Loop来实现管理网络请求以及
。由于在开发中需要一个简单的网络图片下载功能，不想添加第三方库，也不太想自己包装NSOpertaion以及子线程，能够直接通过NSURLConnection

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

```
[downloader performSelectorOnMainThread:@selector(start) 
                             withObject:nil 
                          waitUntilDone:YES];
```


###Sample Code
看的晕乎乎？理解概念最好的方式当然还是动手写代码，写了个例子放在[GitHub](https://github.com/hrchen/NSThreadExample)上，欢迎讨论。

Apple官方也有一个基于Run Loop的异步网络连接样例程序[SimpleURLConnections](http://developer.apple.com/library/ios/#samplecode/SimpleURLConnections/Listings/Read_Me_About_SimpleURLConnections_txt.html)，值得一看。


###参考资料
[Threading Programming Guide](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/Multithreading/CreatingThreads/CreatingThreads.html)
