---
layout: post
title: "iOS App性能优化"
date: 2013-05-29 14:20
comments: true
categories: iOS
---

### iOS App的性能关注点
虽然iPhone的机能越来越好，但是app的功能也越来越复杂，性能从来都是移动开发的核心关注点之一。我们说一个app性能好，不是简单指感觉运行速度快，而应该是指应用启动快速、UI反馈响应及时、列表滚动操作流畅、内存使用合理，当然更不能随随便便Crash啦。工程师开发应用时除了在设计上要避免性能“坑”的出现，在实际遇到“坑”时也要能很快定位原因所在。定位性能问题原因当然不能靠猜，合理的方法是使用工具测量评估出投资回报最高的问题点，然后再加以优化。

本文会从以下几点介绍如何分析和优化iOS app的性能：启动时间、用户响应、内存、图形动画、文件和网络I/O。其中会用到Apple出品的性能分析神器“Instruments”。

### 启动时间
应用启动时间长短对用户第一次体验至关重要，同时系统对应用的启动、恢复等状态的运行时间也有严格的要求，在应用超时的情况下系统会直接关闭应用。以下是几个常见场景下系统对app运行时间的要求：

* Launch	20秒
* Resume	10秒

* Suspend	10秒

* Quit	6秒

* Background Task	10分钟


<!-- more -->

要获取准确的app启动所需时间，最简单的方法时首先在main.c中添加如下代码：

```objc
CFAbsoluteTime StartTime;int main(int argc, char **argv) {	StartTime = CFAbsoluteTimeGetCurrent();
```然后在AppDelegate的回调方法`application:didFinishLaunchingWithOptions`中添加：

	dispatch_async(dispatch_get_main_queue(), ^{
		NSLog(@”Lauched in %f seconds.”,  (CFAbsoluteTimeGetCurrent() – StartTime)); 
	});
可能你会觉得为什么这样可拿到系统启动的时间，因为这个dispatch_async中提交的工作会在app主线程启动后的下一个run lopp中运行，此时app已经完成了载入并且将要显示第一帧画面，也就是系统会运行到`-[UIApplication _reportAppLaunchFinished]`之前。下图是用Instruments工具Time Profiler跑的调用栈，Instruments的使用方法建议看WWDC中与performance相关的[session录像](https://developer.apple.com/videos/wwdc)，文字写起来太单薄不够直观哈。

{% img /images/post/loading_app.png %}

从图中我们可以看到在系统调用`[UIApplication _reportAppLaunchFinished]`之前完成了系统回调`application:didFinishLaunchingWithOptions`。

App的启动会包括以下几个部分（来自[WWDC 2012 Session 235](https://developer.apple.com/videos/wwdc/2012/)）:

1）链接和载入：可以在Time Profile中显示dyld载入库函数，库会被映射到地址空间，同时完成绑定以及静态初始化。

2）UIKit初始化：如果应用的Root View Controller是由XIB实现的，也会在启动时被初始化。

3）应用回调：调用UIApplicationDeleagte的回调：`application:didFinishLaunchingWithOptions`

4）第一次Core Animation调用：在启动后的方法`-[UIApplication _resportAppLaunchFinished]`中调用`CA::Transaction::commit`实现第一帧画面的绘制。如果你的程序启动很慢，能     做的首先是将与显示第一屏画面无关的操作放到之后执行；如果是用XIB文件load第一屏，XIB文件中的View层也要如果扁平，不要有太多图层。

### 用户响应
如何能够让用户觉得你的app响应迅速呢？当然是app用户所触发的操作都能得到立刻响应，即用户事件(User Event)能够被主线程的run loop及时处理。什么是run loop？可以想象成一个处理事件的select多路复用。主线程中的run loop当然主要是为了处理用户产生的事件啦，例如点击、滚动等。以后我们会详细聊聊run loop这个让人迷惑的东东。

要让主线程的run loop更好的响应用户事件，工程师应该尽量减少主线程干重活的时间，尤其是读文件啊，网络操作啊，大量运算啊这类重活，如果是阻塞操作，那就更是大忌了。我们可以用多线程(NSThread、NSOperationQueue, GCD，下一篇Blog就会聊到这多线程)将重活移出主线程，这属于显式并发。还有种隐式并发，例如view和layer的动画、layer的绘制以及PNG图片的解码都是在另一个子线程中执行的。除了使用多线程技术减轻主线程的负担外，减少主线程中阻塞也是提升用户体验的一个方法。使用Instruments中Time Profiler工具中的"Recod thread waiting"选项可以统计出app运行时各个线程中的阻塞系统调用情况，例如文件读写read/write，网络读写send/recv，加锁psynch_mutex_wait等。Instruments中的System Trace工具则能够记录所有的底层系统调用。
    
{% img /images/post/record_thread_waiting.png %}


### 内存
内存问题从来都是iOS app的老大难问题，搞不好程序就爆了。由于iOS系统没有Swap文件(知道为啥不？留给悬念)，在内存不足时会将只读数据(例如code page)从内存中移出，需要的时候再从disk上读如内存；可读写数据不会被系统从内存中移出，然而如果占用的内存达到一个阈值，系统会发出相应的通知和回调让应用release对象以回收内存，如果仍然不能减少内存使用量，系统会直接关闭应用。尤其是iOS 5.0之后，如果你的app收到了memory warning，那么脑袋也是和其他app一样放在了案板上，随时有可能被kill掉，并不是说一定会先Kill掉在后台的app。

App使用的内存除了我们在堆上分配的内存外（`+[NSobject alloc]/malloc`），还会有更多使用内存的地方，比如代码和全局数据（__TEXT和__DATA），线程栈，图片，view 的layer backing store等等。因此处理内存问题，绝不仅仅是我们开发app时尽量少申请内存那么简单。

现在有了超炫的ARC，内存问题相对少了很多，开发效率也得到了提高。但是很多公司的项目仍然由于历史原因采用了手动管理内存，该做的活还是少不了。Xcode自带的静态分析功能可以帮你提前发现一些问题，然而有些内存问题是无法用静态分析来发现的，例如我们不断使用内存没有及时释放的问题，就无法使用静态分析器分析出来。此时可以使用Instruments的Allocations和Leaks工具来检查运行时的的内存使用以及泄露问题。


{% img /images/post/allocations.png %}

Allocations工具可以很直观的反应app的内存使用情况，还有个很赞“Mark Heap”功能，在上图左边下半部分中的Heapshot Analysis中。例如你在进入一个页面前点击一下“Mark Heap”，然后再退回上一页面点击一下“Mark Heap”，如果你在进出这个页面里所申请的内存都得到了合理的释放，那么堆的内存增长量就应该降至0（见上图右下部分）。

另一种严重的内存使用问题是引用了已经释放的内存，直接导致应用崩溃，而Allocation有一个选项Enable NSZombie detection能够在应用使用已经释放的内存时标注出来，同时显示错误发生的调用栈信息。这为解决问题提供了最直接的帮助，当然缺点是必须能够重现EXEC_BAD_ACCESS错误。


{% img /images/post/zombies.png %}

工具Leaks可以在应用运行时直接标示出存在内存泄露的代码，如果发生了内存泄露，可以从泄露详细信息中查看泄露的具体对象以及方法调用栈，大部分问题还是很好解决的。


{% img /images/post/leak.png %}

### 图形和动画

图形性能对用户体验有直接的影响，Instruments中的Core Animation工具用于测量物理机上的图形性能，通过视图的刷新频率大小来判断应用的图形性能。例如一个复杂的列表滚动时它的刷新率应该努力趋近于60fps才能让用户觉得够流畅，从这个数字也可以算出run loop最长的响应时间应该是16毫秒。

启动Instruments的Core Animation工具后可以发现左下部分有一堆选项，我们来逐个介绍：

{% img /images/post/core_animation.png %}

1) Color Blended Layers

Instruments可以在物理机上显示出被混合的图层Blended Layer(用红色标注)，Blended Layer是因为这些Layer是透明的(Transparent)，系统在渲染这些view时需要将该view和下层view混合(Blend)后才能计算出该像素点的实际颜色，如果这种blended layer很多，那么在滚动列表时就甭想有流畅的效果。


{% img /images/post/color_blended_layer.png %}

解决blended layer问题也很简单，检查红色区域view的opaque属性，记得设置成YES；检查backgroundColor属性是不是`[UIColor clearColor]`，要知道背景颜色为clear color那可是图形性能的大敌，基本意味着blended layer是跑不了的了，为什么？自己思考一下:)

2) Color Hits Green and Misses Red

很多视图Layer由于Shadow、Mask和Gradient等原因渲染很高，因此UIKit提供了API用于缓存这些Layer：[layer setShouldRasterize:YES]，系统会将这些Layer缓存成Bitmap位图供渲染使用，如果失效时便丢弃这些Bitmap重新生成。图层Rasterization栅格化好处是对刷新率影响较小，坏处是删格化处理后的Bitmap缓存需要占用内存，而且当图层需要缩放时，要对删格化后的Bitmap做额外计算。
使用这个选项后时，如果Rasterized的Layer失效，便会标注为红色，如果有效标注为绿色。当测试的应用频繁闪现出红色标注图层时，表明对图层做的Rasterization作用不大。

3) Color Misaligned Images

Misaligned Image表示要绘制的点无法直接映射到频幕上的像素点，此时系统需要对相邻的像素点做anti-aliasing反锯齿计算，增加了图形负担，通常这种问题出在对某些View的Frame重新计算和设置时产生的。

{% img /images/post/color_misaligned_image.png %}

上图中被标注为黄色的图层，这是由于图层显示的是被缩放后的图片，如果这些图片是通过网络下载的，可以通过程序更新为确定的绘制大小来解决。还有些系统Navigation Bar和Tool Bar的背景图片使用的是拉伸(Streched)图片，也会被表示为黄色，这是属于正常情况，通常无需修改。这种问题一般对性能影响不大，而是可能会在边缘处虚化。

(4) Color Offscreen-Rendered Yellow

Offscreen-Rendering离屏渲染意思是iOS要显示一个视图时，需要先在后台用CPU计算出视图的Bitmap，再交给GPU做Onscreen-Rendering显示在屏幕上，因为显示一个视图需要两次计算，所以这种Offscreen-Rendering会导致app的图形性能下降。

大部分Offscreen-Rendering都是和视图Layer的Shadow和Mask相关，下列情况会导致视图的Offscreen-Rendering：
1. 使用Core Graphics (CG开头的类)。
2.	使用drawRect()方法，即使为空。
3.	将CALayer的属性shouldRasterize设置为YES。
4.	使用了CALayer的setMasksToBounds(masks)和setShadow*(shadow)方法。
5.	在屏幕上直接显示文字，包括Core Text。
6.	设置UIViewGroupOpacity。

这篇博文[Designing for iOS: Graphics & Performance](http://robots.thoughtbot.com/post/36591648724/designing-for-ios-graphics-performance)对offsreen以及图形性能有个很棒的介绍，(5) Color Copied Images
Copied Image选项可以标注应用绘制时被Core Animation复制的图片，标注成蓝绿色。虽然我在运行时遇到过，不过个人感觉对图形性能影响不大。
(6) Color Immediately，Flash Updated Regions， Color OpenGL Fast Path Blue
Color Immediately选项表示Instruments在做color-flush操作时取消10毫秒的延时。Flash Updated Regions选项用于用红色示标示出在屏幕上使用GPU计算绘制的图层。Color OpenGL Fast Path Blue选项用于用蓝色标示出在屏幕上由OpenGL compositor绘制的内容。
这三个选项对图形性能的分析意义较小，通常仅作为参考。


### 文件和网络I/O

如果需要对app的文件和网络I/O情况做分析，可以用到这三个Instruments工具System Usage、File Activity和Network。

工具System Usage可以统计出运行状态下应用的文件和网络IO操作数据。例如我们发现应用启动后又一个峰值，这可能存在问题，我们可以利用System Usage工具的详细信息栏查看应用是由于对哪些文件的读写操作导致了峰值。

工具File Activity只能在模拟器中运行，因此数据采集可能不是非常准确。它同样可以详细给出读取的文件属性、大小、载入时间等信息，适合与System Usage配合使用。

{% img /images/post/file_activity.png %}

Network工具则可以采集到应用的TCP/IP和UDP的使用信息(传输的数据量、当前所有TCP连接等)，用得不多，做网络使用状况分析时用用还行。

{% img /images/post/instruments_networking.png %}


### 更多阅读

涉及iOS App性能的知识很多，上面只是冰山一角，重点推荐WWDC的session。


WWDC 2012:

* 406: Adopting Automatic Reference Counting
* 238: iOS App Performance: Graphics and Animations
* 242: iOS App Performance: Memory
* 235: iOS App Performance: Responsiveness
* 409: Learning Instruments
* 706: Networking Best Practices
* 514: OpenGL ES Tools and Techniques
* 506: Optimizing 2D Graphics and Animation Performance
* 601: Optimizing Web Content in UIWebViews and Websites on iOS
* 225: Up and Running: Making a Great Impression with Every Launch

WWDC 2011:

* 105: Polishing Your App: Tips and tricks to improve the responsiveness and performance
* 121: Understanding UIKit Rendering
* 131 performance optimization on iphone os
* 308: Blocks and Grand Central Dispatch in Practice
* 323: Introducing Automatic Reference Counting
* 312: iOS Performance and Power Optimization with Instruments

还有几篇不错的blog：


http://oleb.net/blog/2011/11/ios5-tech-talk-michael-jurewitz-on-performance-measurement/

http://eng.pulse.me/tips-for-improving-performance-of-your-ios-application/

http://robots.thoughtbot.com/post/36591648724/designing-for-ios-graphics-performance

http://www.touchwonders.com/en/how-to-make-your-apps-feel-responsive-and-fast-part-2/
