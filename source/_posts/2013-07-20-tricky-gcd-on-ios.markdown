---
layout: post
title: "iOS中GCD的那些坑"
date: 2013-07-20 17:20
comments: true
categories: iOS
---

之前一个系列中[iOS多线程编程Part 3/3 - GCD](http://www.hrchen.com/2013/07/multi-threading-programming-of-ios-part-3/)介绍了GCD的各类接口，别仅仅使用了最基本的dispatch_async和dispatch_sync接口提交个Block完事，那就白瞎GCD的强大功能了。要用高级接口，照旧会有坑在那里，绝大多数坑并不是设计缺陷，而是自身特性造成的误用，本文会记录下这些坑。

###坑一
GCD需要自己生成AutoreleasePool吗？正常我们用NSThread的接口生成一个子线程时，都会在入口方法里生成NSAutoreleasePool或者用@@autoreleasepool{}来回收autoreleased的对象，那么在GCD的Block中呢？

GCD会自动管理每个Queue的autorelease pool，但是我们无法保证什么时候它回去drain(文档中没有说明)，有可能在一个Block执行结束后，也可能很多个Block执行结束后。因此如果仅仅生成少量对象，那就没有必要去自己生成NSAutoreleasePool；否则就自己生成一个NSAutoreleasePool来控制drain pool。


<!--more-->

###坑二
dispatch_once提供是和pthread库中类似pthread_once的功能，dispatch_once接口是指在整个App运行期间运行且仅运行一次提交的Block，但是由于dispatch_once会导致调试非常困难，因为最好少用dispatch_once，就像尽量少用NSObject的类方法+initialize()和+(void)load。

目前用到dispatch_once比较多的地方是在实现singleton单例模式的时候，要注意第一个参数dispatch_once_t必须是个全局或者static变量。

###坑三
dispatch_after(when, queue, block)接口用于在一个时间间隔后执行提交的Block，其中第一个参数类型是dispatch_time_t，可以支持纳秒级的延迟执行，例如：

```
double delayInSeconds = 0.01;
dispatch_time_t delayTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t) (delayInSeconds * NSEC_PER_SEC));
```

然而如果在App中用dispatch_after来控制UI的显示顺序时确实非常危险的，可能并不见的严格按照你期望的延迟量去显示UI，所以最好还是少碰dispatch_after，而是通过合适的回调来控制UI先后顺序，例如利用-viewWillAppear和-viewDidAppear来处理UI的先后顺序。

###坑四
在使用Dispatch Barrier(dispatch_barrier_async或者dispatch_barrier_sync)时，必须注意它只对dispatch_queue_create(label, attr)接口创建的并发队列有作用，如果是Global Queue(dispatch_get_global_queue)，这个barrier就不起作用，想想也正常，是全局的队列，凭什么你一个barrier就同步其他任务的执行呢？所以必须得是私有并发队列才有barrier的作用。如果是私有串行队列呢？那不是和主队列一样，已经是串行的了，还要barrier做啥？

###坑五
如果使用dispatch_get_global_queue来生成全局队列时，可以设置4种优先级设置，但是如果没有明确的必要，不要在程序中使用不同的优先级来控制Block的执行，尤其是在特殊情况可能会导致这篇[Blog](http://www.objc.io/issue-2/concurrency-apis-and-pitfalls.html)提到的Priority Inversion问题，具体为什么直接查看这篇Blog的说明，记住一点尽量只用DISPATCH_QUEUE_PRIORITY_DEFAULT默认的优先级创建全局队列。

###坑六
多线程开发最危险的两件事就是死锁和公共资源访问问题。使用GCD的场景如果很复杂，就有非常大的可能遇到死锁问题，尤其是在使用dispatch_sync的时候：

```
dispatch_sync(queue, ^(){
    dispatch_sync(queue, ^(){
        foo();
    });
});
```
上面代码就会导致死锁，当然我们很少会这么写代码，但是如果这样用用dispatch_sync：

```
void test1()
{
	dispatch_sync(queueA, ^(){
	    test2();
	});
}

void test2()
{
    dispatch_sync(queueB, ^(){
        test3();
    });
}

void test3()
{
    dispatch_sync(queueA, ^(){
        //do something
    });
}
```

如果上面的代码调用test1()，就会导致死锁，这种在某些巧妙的调用关系发生才会导致的死锁可能很难发现，所以如果没有必要，尽量不要使用dispatch_sync。

如果使用dispatch_async，就不会导致死锁，即使像这样调用：

```
dispatch_async(queue, ^(){
    dispatch_async(queue, ^(){
        foo();
    });
});
```



	





