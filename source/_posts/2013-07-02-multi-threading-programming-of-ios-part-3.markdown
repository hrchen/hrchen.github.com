---
layout: post
title: "iOS多线程编程Part 3/3 - GCD"
date: 2013-07-02 21:20
comments: true
categories: iOS
---

前两部分介绍了NSThread、NSRunLoop和NSOperation，本文聊聊2011年WWDC时推出的神器GCD。GCD: Grand Central Dispatch，是一组用于实现并发编程的C接口。GCD是基于Objective-C的Block特性开发的，基本业务逻辑和NSOperation很像，都是将工作添加到一个队列，由系统来负责线程的生成和调度。由于是直接使用Block，因此比NSOperation子类使用起来更方便，大大降低了多线程开发的门槛。另外，GCD是开源的喔：[libdispatch](http://libdispatch.macosforge.org/)

###基本用法
首先示例：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doTask];
    NSLog(@"Fisinished");
});
```
GCD的调用接口非常简单，就是将Job提交至Queue中，主要的提交Job接口为：

* dispatch_sync(queue, block)同步提交job* dispatch_async (queue, block) 异步提交job* dispatch_after(time, queue, block) 同步延迟提交job
其中第一个参数类型是dispatch_queue_t，就是一个表示队列的数据结构`typedef struct dispatch_queue_s *dispatch_queue_t;`；block就是表示任务的Block`typedef void (^dispatch_block_t)( void);`。

dispatch_async函数是异步非阻塞的，调用后会立刻返回，工作由系统在线程池中分配线程去执行工作。
dispatch_sync和dispatch_after是阻塞式的，会一直等到添加的工作完成后才会返回。

除了添加Block到Dispatch Queue，iOS 4之后新增了添加函数到Dispatch Queue的接口，例如dispatch_async对应的有dispatch_async_f：

```
dispatch_async_f(dispatch_queue_t queue,
	             void *context,
	             dispatch_function_t work);
```
其中第三个参数就是个函数指针，即`typedef void (*dispatch_function_t)(void *);`；第二个参数是传给这个函数的参数。

<!--more-->

###Dispatch Queue
要添加工作到队列Dispatch Queue中，这个队列可以是串行或者并行的，并行队列会尽可能的并发执行其中的工作任务，而串行队列每次只能运行一个工作任务。


目前GCD中有三种类型的Dispatch Queue：

* Main Queue：关联到主线程的队列，可以使用函数dispatch_get_main_queue()获得，加到这个队列中的工作都会分发到主线程运行。主线程只有一个，因此很明显这个是串行队列，每次运行一个工作。
* Global Queue：全局队列是并发队列，又根据优先级细分为高优先级、默认优先级和低优先级三种。通过dispatch_get_global_queue加上优先级参数获得这个全局队列，例如`dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)`
* 自定义Queue：自己创建一个队列，通过函数dispatch_queue_create创建，例如`dispatch_queue_create("com.kiloapp.test", NULL)`。第一个参数是队列的名字，Apple建议使用反DNS型的名字命名，防止重名；第二个参数是创建的queue的类型，iOS 4.3以前只支持串行，即DISPATCH_QUEUE_SERIAL(就是NULL)，iOS4.3以后也开始支持并行队列，即参数DISPATCH_QUEUE_CONCURRENT。

由于有这些种不同类型的队列，一种常见的使用模式是：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [self doHardWorkInBackground];
    dispatch_async(dispatch_get_main_queue(), ^{
        [self updateUI];
    });
});
```
将一些耗时的工作添加到全局队列，让系统分配线程去做，工作完成后再次调用GCD的主线程队列去完成UI相关的工作，这样做就不会因为大量的非UI相关工作加重主线程负担，从而加快UI事件响应。


###Dispatch Group

GCD确实非常简单好用，不过有些场景下还是有点问题，例如：

```
for(id obj in array)
{
    [self doWorkOnItem:obj];
}
[self doWorkOnArray:array];
```

前半部分可以用GCD得到处理性能的提升：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for(id obj in array)
    dispatch_async(queue, ^{
        [self doWorkOnItem:obj];
    });
[self doWorkOnArray:array];
```
问题是`[self doWorkOnArray:array];`原先是在全部数组各个成员的工作完成后才会执行的，现在由于dispatch_async是异步的，`[self doWorkOnArray:array];`很有可能在各个成员的工作完成前就开始运行，这明显不符合原先的语义。如果将dispatch_async改成dispatch_sync可以解决问题，但是和原来的方法一样没有并行处理数组，使用GCD也就没有意义了。

针对这种情况，GCD提供了Dispatch Group可以将一组工作集合在一起，等待这组工作完成后再继续运行。dispatch_group_create函数可以用来创建这个Group：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doWorkOnItem:obj];
    });
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
dispatch_release(group);
[self doWorkOnArray:array];
```

方法是不是很简单，将并发的工作用dispatch_group_async异步添加到一个Group和全局队列中，dispatch_group_wait会等待这些工作完成后再返回，这样你就可以再运行`[self doWorkOnArray:array];`。

不过有点不好的是dispatch_group_wait会阻塞当前线程，如果当前是主线程岂不是不好，有更绝的dispatch_group_notify接口：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doWorkOnItem:obj];
    });
dispatch_group_notify(group, queue, ^{
    [self doWorkOnArray:array];
});
dispatch_release(group);
```

dispatch_group_notify函数可以将这个Group完成后的工作也同样添加到队列中（如果是需要更新UI，这个队列也可以是主队列），总之这样做就完全不会阻塞当前线程了。

Dispatch Group还有两个接口可以显式的告知group要添加block操作：
dispatch_group_enter(group)和dispatch_group_leave(group)，这两个接口的调用数必须平衡，否则group就无法知道是不是处理完所有的Block了。


###Dispatch Apply

如果就是要同步的执行对数组元素的逐个操作，GCD也提供了一个简便的dispatch_apply函数：

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index){
    [self doWorkOnItem:obj:[array objectAtIndex:index]];
});
[self doWorkOnArray:array];
```

###Dispatch Barrier

在使用dispatch_async异步提交时，是无法保证这些工作的执行顺序的，如果需要某些工作在某个工作完成后再执行，那么可以使用Dispatch Barrier接口来实现，barrier也有同步提交dispatch_barrier_async(queue, block)和异步提交dispatch_barrier_sync(queue, block)两种方式。例如：

```
dispatch_async(queue, block1);
dispatch_async(queue, block2);
dispatch_barrier_async(queue, block3);
dispatch_async(queue, block4);
dispatch_async(queue, block5);
```
dispatch_barrier_async是异步的，调用后立刻返回，即使block3到了队列首部，也不会立刻执行，而是等到block1和block2的并行执行完成后才会执行block3，完成后再会并行运行block4和block5。注意这里的queue应该是一个并行队列，否则dispatch_barrier_async操作就失去了意义。

###Dispatch Source

Run Loop有Input Source，GCD也同样支持一系列事件监听和处理，GCD有一组Dispatch Source接口可以监听底层系统对象(例如文件描述符、网络描述符、Mach Port、Unix信号、VFS文件系统的vnode等)的事件，可以设置这些事件的处理函数，如果事件发生时，Dispatch Source就可以将事件的处理方法提交到队列中执行。

dispatch_source_t是Dispatch Source的数据结构，使用dispatch_source_create(type, handle, mask, queue)来创建，第一个参数是source的类型：

```
#define DISPATCH_SOURCE_TYPE_DATA_ADD
#define DISPATCH_SOURCE_TYPE_DATA_OR
#define DISPATCH_SOURCE_TYPE_MACH_RECV
#define DISPATCH_SOURCE_TYPE_MACH_SEND
#define DISPATCH_SOURCE_TYPE_PROC
#define DISPATCH_SOURCE_TYPE_READ
#define DISPATCH_SOURCE_TYPE_SIGNAL
#define DISPATCH_SOURCE_TYPE_TIMER
#define DISPATCH_SOURCE_TYPE_VNODE
#define DISPATCH_SOURCE_TYPE_WRITE
```
第二个参数handle和第三个参数mask与source的类型相关，有不同的含义，第四个参数是source绑定的queue，由于篇幅问题这些含义请参考《Grand Central Dispatch (GCD) Reference》。

dispatch_source_set_event_handler(source, handler)接口可以添加source的处理方法handler，这里的handler是一个block。如果是dispatch_source_set_event_handler_f(source, handler)，这里的handler就是function。

dispatch_source_cancel(source)接口可以异步取消一个source，取消后上面设置dispatch_source_set_event_handler的evnet handler就不会再执行。取消一个source时，如果之前使用dispatch_source_set_cancel_handler(source, handler)设置了一个取消时的处理block，那么这个block就会在取消source的时候提交至source关联的queue中去执行，可以用来清理资源。

dispatch_source_get_data(source)接口用于返回source需要处理的数据，根据当初创建source类型不同有不同的含义，而且这个接口必须在event handler中调用，否则返回结果可能未定义。

dispatch_source_get_handle(source)和dispatch_source_get_mask(source)接口分布用于获取当初创建source时的两个参数handle和mask。

dispatch_source_merge_data(source, value)接口用于将一个value值合并到souce中，这个source的类型必须是DISPATCH_SOURCE_TYPE_DATA_ADD或者DISPATCH_SOURCE_TYPE_DATA_OR。

下面举个source的例子，使用dispatch_source_get_data和dispatch_source_merge_data，假如我们在处理上面那个数组时要在UI中显示一个进度条：


```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_get_main_queue());

dispatch_source_set_event_handler(source, ^{
    [progressIndicator incrementBy:dispatch_source_get_data(source)];
});
dispatch_resume(source);
    
dispatch_apply([array count], globalQueue, ^(size_t index) {
    [self doWorkOnItem:obj:[array objectAtIndex:index]];
    dispatch_source_merge_data(source, 1);
});
```

注意dispatch source创建后是处于suspend状态的，必须使用dispatch_resume来恢复，dispatch_apply中每处理一个数组元素会调用dispatch_source_merge_data加1，那么这个source的事件handler就可以通过dispatch_source_get_data拿到source的数据。

###Dispatch Once

dispatch_once的意思是在App整个生命周期内运行并且只允许一次，类似于pthread库中的[pthread_once](http://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_once.html))。由于dispatch_once的调试非常困难，所以最好还是少用，单例应该是少数值得用的地方了。

传统我们实现单例是这样：

```
+ (id)sharedManager
{
    static Manager *theManager = nil;
    @synchronized([Manager class])
    {
        if(!theManager)
            theManager = [[Manager alloc] init];
    }
    return theManager;
}

```
这个的成本还是有点高，每次访问都会有同步锁，使用dispatch_once可以保证只运行一次初始化：

```
+ (id)sharedWhatever
{
    static dispatch_once_t pred;
    static Manager *theManager = nil;
    dispatch_once(&pred, ^{
        theManager = [[Manager alloc] init];
    });
    return theManager;
}
```
需要注意dispatch_once_t最好使用全局变量或者是static的，否则可能导致无法确定的行为。

###Dispatch Semaphore

和其他多线程技术一样，GCD也支持信号量，dispatch_semaphore_create(value)用于创建一个信号量类型dispatch_semaphore_t，参数是long类型，表示信号量的初始值；dispatch_semaphore_signal(semaphore)用于通知信号量(增加一个信号量)；dispatch_semaphore_wait(semaphore, timeout)用于等待信号量(减少一个信号量)，第二个参数是超时时间，如果返回值小于0，会按照先后顺序等待其他信号量的通知。


###管理GCD数据

所有GCD的对象同样是有引用计数的，如果引用计数为0就被释放，如果你不再需要所创建的GCD对象，就可以使用dispatch_release(object)将对象的引用计数减一；同样可以使用dispatch_retain(object)将对象的引用计数加一。注意由于全局和主线程队列对象都不需要去dispatch_release和dispatch_retain，即使调用了也没有作用。

dispatch_suspend(queue)可以暂停一个GCD队列的执行，当然由于是block粒度的，如果调用dispatch_suspend时正好有队列中block正在执行，那么这些运行的block结束后不会有其他的block再被执行；同理dispatch_resume(queue)可以恢复一个GCD队列的运行。注意dispatch_suspend的调用数目需要和dispatch_resume数目保持平衡，因为dispatch_suspend是计数的，两次调用dispatch_suspend会设置队列的暂停数为2，必须再调用两次dispatch_resume才能让队列重新开始执行block。

可以使用dispatch_set_context(object, context)给一个GCD对象设置一个关联的数据，第二个参数任何一个内存地址；dispatch_set_context(object)就是获得这个关联数据，这样可以方便传递各类上下文数据。

本小节提到的GCD对象不单指队列dispatch_queue_t，是指在GCD中出现的各种类型，声明类型dispatch_object_t是个union：

```
typedef union {
   struct dispatch_object_s *_do;
   struct dispatch_continuation_s *_dc;
   struct dispatch_queue_s *_dq;
   struct dispatch_queue_attr_s *_dqa;
   struct dispatch_group_s *_dg;
   struct dispatch_source_s *_ds;
   struct dispatch_source_attr_s *_dsa;
   struct dispatch_semaphore_s *_dsema;
   struct dispatch_data_s *_ddata;
   struct dispatch_io_s *_dchannel;
   struct dispatch_operation_s *_doperation;
   struct dispatch_fld_s *_dfld;
} dispatch_object_t 
```

###Dispatch Data 对象

GCD是基于C的接口，其内部处理数据是无法直接使用Objective-C的数据类型，如果要使用数据buffer时需要自己malloc一块内存空间来用，因此GCD提供了类似Objective-C中NSData的dispatch_data_t数据结构作为数据buffer。

dispatch_data_t的类型dispatch_data_s的指针，使用dispatch_data_create(buffer, size, queue, destructor)可以创建一个dispatch_data_t，第一个参数是保存数据的内存地址，第二个参数size是数据字节大小，第三个参数queue提交destructor block的队列，第四个参数destructor是用于释放data的block，默认是DISPATCH_DATA_DESTRUCTOR_DEFAULT和DISPATCH_DATA_DESTRUCTOR_FREE，后者在buffer是使用malloc生成的缓冲区时使用。示例：

```
void *buffer = malloc(length);
dispatch_data_t data = dispatch_data_create(buffer, length, NULL, DISPATCH_DATA_DESTRUCTOR_FREE);
```

如果是从NSData转换为dispatch_data_t：

```
nsdata = [nsdata copy];
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
	return dispatch_data_create([nsdata bytes], [nsdata length], queue, ^{
	    [nsdata release];
	});
```

与直接使用己malloc分配的连续内存空间不同，dispatch_data_t可以直接将两块数据用dispatch_data_create_concat(dataA, dataB)拼接起来，还可以用dispatch_data_create_subrange(data, offset, length)获取部分dispatch_data_t。

如果反过来要访问一个dispatch_data_t对应的内存空间，就需要使用dispatch_data_create_map(data, buffer_ptr, size_ptr)接口，示例：

```
const void *buffer;
size_t length;
dispatch_data_t tmpData = dispatch_data_create_map(data, &buffer, &length);

//可以得到dispatch_data_t的内存空间地址和字节大小
//这里我们可以直接使用buffer指针对应的内存
//返回的tmpData是一个新的对应data连续内存空间的dispatch_data_t

dispatch_release(tmpData);
```
   

###Dispatch I/O Channel

GCD提供的这组Dispatch I/O Channel接口用于异步处理基于文件和网络描述符的操作，可以用于文件和网络I/O操作。

Dispatch IO Channel对象dispatch_io_t就是对一个文件或网络描述符的封装，使用dispatch_io_t dispatch_io_create(type, fd, queue, cleanup_hander)接口生成一个dispatch_io_t对象。第一个参数type表示channel的类型，有DISPATCH_IO_STREAM和DISPATCH_IO_RANDOM两种，分布表示流读写和随机读写；第二个参数fd是要操作的文件描述符；第三个参数queue是cleanup_hander提交需要的队列；第四个参数cleanup_hander是在系统释放该文件描述符时的回调。示例：

```
dispatch_io_t fileChannel = dispatch_io_create(DISPATCH_IO_STREAM, STDIN_FILENO, dispatch_get_global_queue(0, 0), ^(int error) {
        if(error)
            fprintf(stderr, "error from stdin: %d (%s)\n", error, strerror(error));
    });
    
```
dispatch_io_close(channel, flag)可以将生成的channel关闭，第二个参数是关闭的选项，如果使用DISPATCH_IO_STOP (0x01)就会立刻中断当前channel的读写操作，关闭channel。如果使用的是0，那么会在正常读写结束后才会关闭channel。

During a read or write operation, the channel uses the high- and low-water mark values to determine how often to enqueue the associated handler block. It enqueues the block when the number of bytes read or written is between these two values.

在channel的读写操作中，channel会使用low_water和high_water值来决定读写了多大数据才会提交相应的数据处理block，可以dispatch_io_set_low_water(channel, low_water)和dispatch_io_set_high_water(channel, high_water)设置这两个值。

Channel的异步读写操作使用接口dispatch_io_read(channel, offset, length, queue, io_handler)和dispatch_io_write(channel, offset, data, queue, io_handler)。dispatch_io_read接口参数分布表示channel，偏移量，字节大小，提交IO处理block的队列，IO处理block；dispatch_io_write接口参数分别表示channel，偏移量，数据(dispatch_data_t)，提交IO处理block的队列，IO处理block。其中io_handler的定义为`^(bool done, dispatch_data_t data, int error)()`。

举个例子，将STDIN读到的数据写到STDERR：

```
dispatch_io_read(stdinChannel, 0, SIZE_MAX, dispatch_get_global_queue(0, 0), ^(bool done, dispatch_data_t data, int error) {
       if(data)
       {
           dispatch_io_write(stderrChannel, 0, data, dispatch_get_global_queue(0, 0), ^(bool done, dispatch_data_t data, int error) {});
       }
});

```

看起来使用上还挺麻烦的，需要创建Channel才能进行读写，因此GCD直接提供了两个方便异步读写文件描述符的接口(参数含义和channel IO的类似)：

```
void dispatch_read(
   dispatch_fd_t fd,
   size_t length,
   dispatch_queue_t queue,
   void (^handler)(dispatch_data_t data, int error));
   
void dispatch_write(
   dispatch_fd_t fd,
   dispatch_data_t data,
   dispatch_queue_t queue,
   void (^handler)(dispatch_data_t data, int error));
   
```

###总结

GCD的API按功能分为：

* 创建管理Queue* 提交Job* Dispatch Group* 管理Dispatch Object* 信号量Semaphore* 队列屏障Barrier* Dispatch Source* Queue Context数据* Dispatch I/O Channel* Dispatch Data 对象

各组接口的详细说明还是参考《Grand Central Dispatch (GCD) Reference》。

###参考资料
[Grand Central Dispatch (GCD) Reference](https://developer.apple.com/library/mac/#documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html)

[Blocks Programming Topics](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html)
