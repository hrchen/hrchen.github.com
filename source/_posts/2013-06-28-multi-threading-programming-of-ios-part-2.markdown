---
layout: post
title: "iOS多线程编程Part 2/3 - NSOperation"
date: 2013-06-28 23:59
comments: true
categories: iOS
---

多线程编程Part 1介绍了NSThread以及NSRunLoop，这篇Blog介绍另一种并发编程技术：NSOPeration。


###NSOperation & NSOperationQueue
从头文件NSOperation.h来看接口是非常的简洁，NSOperation本身是一个抽象类，定义了一个要执行的工作，NSOperationQueue是一个工作队列，当工作加入到队列后，NSOperationQueue会自动按照优先顺序及工作的从属依赖关系(如果有的话)组织执行。

NSOperation是没法直接使用的，它只是提供了一个工作的基本逻辑，具体实现还是需要你通过定义自己的NSOperation子类来获得。如果有必要也可以不将NSOperation加入到一个NSOperationQueue中去执行，直接调用起`-start`也可以直接执行。

<!--more-->

在继承NSOpertaion后，对于非并发的工作，只需要实现NSOperation子类的main方法：

```
-(void)main 
{
   @try 
   {
      // 处理工作任务
   }
   @catch(...) 
   {
      // 处理异常，但是不能再重新抛出异常
   }
}
```
由于NSOperation的工作是可以取消Cancel的，那么你在main方法处理工作时就需要不断轮询`[self isCancelled]`确认当前的工作是否被取消了。

如果要支持并发工作，那么NSOperation子类需要至少override这四个方法:

* start
* isConcurrent
* isExecuting
* isFinished

实现了一个基于Operation的下载器，在Sample Code中可以下载。

```
- (void)operationDidStart
{
    [self.lock lock];
    NSMutableURLRequest* request = [[NSMutableURLRequest alloc] initWithURL:self.URL
                                                                cachePolicy:NSURLRequestReloadIgnoringCacheData
                                                            timeoutInterval:self.timeoutInterval];
    [request setHTTPMethod: @"GET"];
    
    self.connection =[[NSURLConnection alloc] initWithRequest:request
                                                     delegate:self
                                             startImmediately:NO];
    [self.connection scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
    [self.connection start];
    [self.lock unlock];
}

- (void)operationDidFinish
{
    [self.lock lock];
    [self willChangeValueForKey:@"isFinished"];
    [self willChangeValueForKey:@"isExecuting"];
    
    self.executing = NO;
    self.finished = YES;
    
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
    [self.lock unlock];
}

- (void)start
{
    [self.lock lock];
    if ([self isCancelled])
    {
        [self willChangeValueForKey:@"isFinished"];
        self.finished = YES;
        [self didChangeValueForKey:@"isFinished"];
        return;
    }
    
    [self willChangeValueForKey:@"isExecuting"];
    [self performSelector:@selector(operationDidStart) onThread:[[self class] networkThread] withObject:nil waitUntilDone:NO];
    self.executing = YES;
    [self didChangeValueForKey:@"isExecuting"];
    [self.lock unlock];
}

- (void)cancel
{
    [self.lock lock];
    [super cancel];
    if (self.connection)
    {
        [self.connection cancel];
        self.connection = nil;
    }
    
    [self.lock unlock];
}

- (BOOL)isConcurrent {
    return YES;
}

- (BOOL)isExecuting {
    return self.executing;
}

- (BOOL)isFinished {
    return self.finished;
}

```

start方法是工作的入口，通常是你用来设置线程或者其他执行工作任务需要的运行环境的，注意不要调用[super start]；isConcurrent是标识这个Operation是否是并发执行的，这里是个坑，如果你没有实现isConcurrent，默认是返回NO，那么你的NSOperation就不是并发执行而是串行执行的，大多数情况下这可不是你想要的；isExecuting和isFinished是用来报告当前的工作执行状态情况的，注意必须是线程访问安全的。

注意你的实现要发出合适的KVO通知，因为如果你的NSOperation实现需要用到工作依赖从属特性，而你的实现里没有发出合适的“isFinished”KVO通知，依赖你的NSOperation就无法正常执行。NSOperation有许多支持KVO的属性：

* isCancelled
* isConcurrent
* isExecuting
* isFinished
* isReady
* dependencies
* queuePriority
* completionBlock

当然也不是说所有的KVO通知都需要自己去实现，例如通常你用不到addObserver到你工作的“isCancelled”属性，你只需要直接调用cancel方法就可以取消这个工作任务。

实现NSOperation子类后，可以直接调用start或者添加到一个NSOperationQueue里：

```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperation:downloader];
    
```

###NSOperation和NSOperationQueue其他特性

工作是有优先级的，可以通过NSOperation的一下两个接口读取或者设置：

```
- (NSOperationQueuePriority)queuePriority;
- (void)setQueuePriority:(NSOperationQueuePriority)p;
```

工作之间也可有从属依赖关系，只有依赖的工作完成后才会执行：

```
- (void)addDependency:(NSOperation *)op;
- (void)removeDependency:(NSOperation *)op;
```

还可以通过下面接口设置运行NSOpration的子线程优先级：

```
- (void)setQueuePriority:(NSOperationQueuePriority)priority;
```

iOS4之后还可以往NSOperation上添加一个结束block，用于在工作执行结束之后的操作：

```
- (void)setCompletionBlock:(void (^)(void))block;
```

如果需要阻塞等待NSOperation工作结束(别在主线程这么干)，可以使用接口：

```
- (void)waitUntilFinished;
```

NSOperationQueue除了添加NSOperation外，也支持直接添加一个Block(iOS4之后)：

```
- (void)addOperationWithBlock:(void (^)(void))block
```

NSOperationQueue可以取消所有添加的工作：

```
- (void)cancelAllOperations;
```
也可以阻塞式的等待所有工作结束(别在主线程这么干)：

```
- (void)waitUntilAllOperationsAreFinished;
```

在NSOperation对象中获得被添加的NSOperationQueue队列：

```
+ (id)currentQueue
```

要获得一个绑定在主线程的NSOperationQueue队列：

```
+ (id)mainQueue
```


###NSInvocationOperation & NSBlockOperation

其实除非必要，简单的工作完全可以使用官方提供的NSOperation两个子类NSInvocationOperation和NSBlockOperation来实现。

NSInvocationOperation：

```
NSInvocationOperation* theOp = [[NSInvocationOperation alloc] 
                       initWithTarget:self                 
		                  selector:@selector(myTaskMethod:)                                           
                            object:data];
```

NSBlockOperation:

```
NSBlockOperation* theOp = [NSBlockOperation blockOperationWithBlock: ^{
      NSLog(@"Beginning operation.\n");
      // Do some work.
   }];
```

接口非常简单，一看便会。

###Sample Code
本文例子放在[Github](https://github.com/hrchen/ExamplesForBlog)上（工程NSURLConnectionExample中的PTOperationDownloader）。

###参考资料
[Concurrency Programming Guide](http://developer.apple.com/library/mac/#documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html)

[NSOperation Class Reference](http://developer.apple.com/library/mac/#documentation/Cocoa/Reference/NSOperation_class/Reference/Reference.html)
