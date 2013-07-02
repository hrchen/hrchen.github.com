---
layout: post
title: "iOS多线程编程Part 3/3 - GCD"
date: 2013-06-28 23:59
comments: true
categories: iOS
---

前两部分介绍了NSThread、NSRunLoop和NSOperation的基本支持，本文聊聊iOS4发布时推出的神器GCD。


###基础
GCD: Grand Central Dispatch，是一组用于实现并发编程的low level的C接口，基本功能非常类似NSOperation，允许将工作分解成互不相关的子任务串行或者并行的添加到工作队列中，由于接近系统底层，因此性能比NSOperation要好，但是不属于Cocoa框架。

GCD是完全基于Objective-C的心

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










###Dispatch Source


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
