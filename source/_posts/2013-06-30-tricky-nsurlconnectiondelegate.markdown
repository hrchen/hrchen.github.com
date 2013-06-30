---
layout: post
title: "一个NSURLConnectionDelegate的坑"
date: 2013-06-30 04:20
comments: true
categories: iOS
---


NSURLConnection的坑还是蛮多的，上次是[RunLoopMode的问题](http://www.hrchen.com/2013/06/nsurlconnection-with-nsrunloopcommonmodes/)，这次是关于NSURLConnectionDelegate。

NSURLConnection的代理Protocol定义有三类：NSURLConnectionDelegate、NSURLConnectionDataDelegate和NSURLConnectionDownloadDelegate。

* NSURLConnectionDelegate：所有类型NSURLConnection的基础代理方法，都是Optional的方法，主要是涉及SSL/TSL加密的相关接口。

```
@optional
- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error;
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection;
- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;

```
* NSURLConnectionDataDelegate：用于将网络请求的数据存放到内存中(以NSData的形式)的代理方法。所有方法都是Optional的。

<!--more-->

```
@optional
- (NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(NSURLResponse *)response;
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data;

- (NSInputStream *)connection:(NSURLConnection *)connection needNewBodyStream:(NSURLRequest *)request;
- (void)connection:(NSURLConnection *)connection   didSendBodyData:(NSInteger)bytesWritten
                                                 totalBytesWritten:(NSInteger)totalBytesWritten
                                         totalBytesExpectedToWrite:(NSInteger)totalBytesExpectedToWrite;

- (NSCachedURLResponse *)connection:(NSURLConnection *)connection willCacheResponse:(NSCachedURLResponse *)cachedResponse;

- (void)connectionDidFinishLoading:(NSURLConnection *)connection;

```
* NSURLConnectionDownloadDelegate：用于将网络资源直接保存到文件中的代理方法，除了connectionDidFinishDownloading:destinationURL:都是Optional的方法。connectionDidFinishDownloading:destinationURL回调可以告知你下载的网络数据最终存放的文件位置，正常都是在iPhone应用沙盒的/tmp目录下。

```
@optional
- (void)connection:(NSURLConnection *)connection didWriteData:(long long)bytesWritten totalBytesWritten:(long long)totalBytesWritten expectedTotalBytes:(long long) expectedTotalBytes;
- (void)connectionDidResumeDownloading:(NSURLConnection *)connection totalBytesWritten:(long long)totalBytesWritten expectedTotalBytes:(long long) expectedTotalBytes;

@required
- (void)connectionDidFinishDownloading:(NSURLConnection *)connection destinationURL:(NSURL *) destinationURL;

```


由于生成NSURLConnectin对象传入delegate参数时类型就是id，而不是传统id<***Delegate>形式，那么如何确定当前代理实现的是什么类型的NSURLConnectionDelegate代理呢？方法也很诡异，如果你的代理实现了connectionDidFinishDownloading:destinationURL:，那么就表示你要实现的是NSURLConnectionDownloadDelegate，NSURLConnectionDataDelegate中的connection:DidReceiveData就不会得到回调，即使你实现了它。道理很简单，这两类代理一个是用于将下载数据保存到文件上，另一个是保存到内存中，只能两者居其一。

故事还没有结束 ，如果你实现了connectionDidFinishDownloading:destinationURL并且想通过回到得到的destinationURL读取保存数据的文件时，令人惊讶的发现这个文件居然不存在，因为这类NSURLConnectionDataDelegate回调是用于Newsstand类型的App开发的，用于将杂志等信息保存到本地文件。实在想不通为什么只有Newsstand类型App才能用这组接口，很多开发者早已发了bug报告给Apple，Apple也已经确认，但是从iOS5到了iOS7，这个“bug”还是没有被修复。

附带我在Stackoverflow上的[回答](http://stackoverflow.com/questions/11047169/how-may-delegate-method-from-one-protocol-prevent-execution-of-another-one-from/17369617#17369617)。

