---
layout: post
title: "Core Animation编程 Part1/2"
date: 2014-03-12 18:13
comments: true
categories: iOS
---

那啥，一忙起来就彻底忘记更新Blog了，目前移动产品开发也有了新的趋势，继续守在一个平台越来越难混了，HTML5在国内App开发中也逐步流行，因此未来移动开发工程师不仅仅要懂iOS/Android，多少还需要了解些H5，也就是Hybrid的开发模式，具体改日再叙。

今天继续扯iOS开发系列，聊聊iOS中牛逼闪闪的Core Animation，会有两个部分。先来了解下iOS中动画的层次：

{% img /images/post/core-animation-architecture.jpg %}

最上层是UIKit，第二层是Core Anmation，然后iOS把这些动画效果用Open GL将在硬件上渲染出来。UIKit层中的动画API是作用于UIView，Core Animation层的动画API是作用于CALayer。我个人学习iOS的一个方法就是看类的头文件和相应的官方Reference文档。头文件可以很快查找出类的继承结构、属性和接口情况；Reference文档会详细介绍类的属性、方法以及需要了解的核心信息，没什么比这个文档说得更清楚了。

[UIView Class Reference](http://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIView_Class/UIView/UIView.html)中讲到UIView主要负责三件事：

(1) 绘图和动画：利用UIKit、Core Graphics或者OpenGL ES绘制视图内容；有些UIView视图属性支持动画。

(2) 布局和子视图管理；一个视图可以关联零个或者多个子视图，如果有子视图，还可以定义子视的大小和位置；每个视图会定义它相对于父视图的尺寸变化(resizing)规则

(3) 事件处理：UIView继承自UIResponder，因此可以处理Touch事件和其他UIResponder中定义的事件；可以添加UIGestureRecognizer到UIView上，从而处理常见的几种手势操作。

<!--more-->

UIView的渲染原理可以参考WWDC 2011的Session：[Understanding UIKit Rendering](https://developer.apple.com/videos/wwdc/2011/)以及objc.io的[Getting Pixels onto the Screen](http://www.objc.io/issue-3/moving-pixels-onto-the-screen.html)，应该算是原理比较透彻了。

UIView中支持动画的属性有frame，bounds，center，alpha，backgroundColor，contentStretch(iOS 6之后不推荐使用，已经是Deprecated)，transform。除了transform外，其他属性都很直白，这里transform是CGAffineTransform类型（[CGAffineTransform Reference](http://developer.apple.com/library/ios/#documentation/GraphicsImaging/Reference/CGAffineTransform/Reference/reference.html)）。这里的transform是Affine Transform，即仿射变换，可以在二维中做各种视图变换。其本质就是个矩阵：

{% img https://developer.apple.com/library/ios/documentation/GraphicsImaging/Reference/CGAffineTransform/Art/equation01_2x.png %}

通过CGAffineTransform.h中的各种TransformMake接口就可以轻松实现二维视图的选择、放大、移动等操作。如果要了解如何使用这些仿射变换，可以参考文档[《Quartz 2D Programming Guide》](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html)。Quartz 2D是iOS上的二维渲染引擎，提供了诸如透明Layer、Path绘图、离屏渲染、颜色管理、反锯齿渲染以及PDF相关的显示。基本上我们用的到这些接口的场景也就是：二维绘图、图形编辑功能、创建和显示位图，还有PDF相关的功能。细节这里就不表了，改日再叙。

UIView层的动画API相信大家都用过(非Block参数的UIView动画API实在太老旧，使用价值已经不大)：

```
+ (void)animateWithDuration:(NSTimeInterval)duration delay:(NSTimeInterval)delay options:(UIViewAnimationOptions)options animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion NS_AVAILABLE_IOS(4_0);

+ (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion NS_AVAILABLE_IOS(4_0); // delay = 0.0, options = 0

+ (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations NS_AVAILABLE_IOS(4_0); // delay = 0.0, options = 0, completion = NULL
```
使用方法也很简单，就是在API的Block中改变需要实现动画的UIView属性值。iOS 7中又增加了针对Keyframe动画的API：

```
+ (void)animateKeyframesWithDuration:(NSTimeInterval)duration delay:(NSTimeInterval)delay options:(UIViewKeyframeAnimationOptions)options animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion NS_AVAILABLE_IOS(7_0);

+ (void)addKeyframeWithRelativeStartTime:(double)frameStartTime relativeDuration:(double)frameDuration animations:(void (^)(void))animations NS_AVAILABLE_IOS(7_0); // start time and duration are values between 0.0 and 1.0 specifying time and duration relative to the overall time of the keyframe animation

```


Core Animation实际上就是作用在UIVIew属性layer上的动画，layer的类型是CALayer，定义在QuartzCore Framework中。从CALayer.h可以发现它和UIView实在太像了。它实现了NSCoding和CAMediaTiming协议，这个CAMediaTiming协议很有意思，稍后再说。QuartzCore Framework中除了CALayer以其各种子类外，比较重要的就是CAAction、CAAnimation、CAMediaTiming和CAMediaTimingFunction。

如何区分UIView和CALayer的关系？每个UIView都有layer属性，并且不会为空，实际是iOS系统为了图形渲染的优化，在Mac OS上NSView需要设置才会生成layer。layer不发生变化时其内容就是个CGImageRef。


CALayer有和UIView类似的属性frame、bounds、hidden、opaque，也有以下这些特别的属性：

- position：
- zPosition
- anchorPoint
- anchorPointZ
- transform
- cornerRadius
- backgroundColor
- borderColor
- opacity
- shadowRadius
- shadowOffset
- shadowColor
- shadowPath
- shouldRasterize
- contents


说完CALayer，可以直接上Example，推荐Github的[CA360](https://github.com/neror/CA360),常见的Core Animation类型有涉及。

- 基础动画类型（BasicAnimation）：使用类CABasicAnimation，例子如下：

```
 CABasicAnimation *pulseAnimation = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
  pulseAnimation.duration = .5;
  pulseAnimation.toValue = [NSNumber numberWithFloat:1.1];
  pulseAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
  pulseAnimation.autoreverses = YES;
  pulseAnimation.repeatCount = FLT_MAX;

```

CABasicAnimation集成至CAPropertyAnimation，表示基于属性变化的动画，其中和时间相关的属性都来自于CAMediaTiming Protocol，理解起来也很直观，

duration表示动画的持续时间，默认为0秒；

speed表示动画的速度，用于将父对象的时间对应当当前时间，默认为1，如果设置为2，就表示动画速度是父对象的2倍；

repeatCount表是动画重复的次数，默认为0；repeatDuration表示重复时候的持续时间，默认为0秒；autoreverses表示是否反向播放动画，默认为NO；


beginTime表示动画开始的时间，默认为0，它是相对于父对象的时间空间而言的，即如果一个Animation Group中的一个animation设置beginTime为2，则表示这个animation在animation开始的2秒后才会开始执行。如果要在不同Layer层级之间计算时间，需要用到CALayer的API：

```
- (CFTimeInterval)convertTime:(CFTimeInterval)t fromLayer:(CALayer *)l;
- (CFTimeInterval)convertTime:(CFTimeInterval)t toLayer:(CALayer *)l;
```

timeOffset表示动画对当前时间的偏移量，默认为0秒；举个例子加强理解，假设有个动画有五部分:1,2,3,4,5（分别占用一秒），如果这组动画的beginTime为1，则动画播放顺序为2，3，4，5，1；如果beginTime为2，则动画播放顺序为3，4，5，1，2；以此类推。如果需要从父对象的时间控件转换到当前对象的时间控件，使用如下公式：

```
t = (tp - begin) * speed + offset
```
timeOffset的一个用途就是同过设置speed为0和offset为一个合适的值来暂停一个layer，可以见这个例子(获取到该layer上的当前时间，将speed降为0，同时将timeoffset置为当前时间，即可暂停动画到):

```
CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];  
layer.speed = 0.0;  
layer.timeOffset = pausedTime;  
```

fillMode表示表示动画的效果在有效期前后的赋值模式，有kCAFillModeForwards、kCAFillModeBackwards、kCAFillModeBoth、kCAFillModeRemoved几类，默认的为kCAFillModeRemoved。继续之前的假设动画有5个状态：1，2，3，4，5（分别占用一秒），如果动画的beginTime为1，同时duration为3，则实际的动画序列为2，3，4。这个动画添加到Animation Group中，Animation Group的duration为5，那么这个动画Group开始和结束的一秒内显示什么呢？如果fillMode是kCAFillModeForwards（动画属性向后拖后），则动画序列为2，3，4，5，5；如果fillMode是kCAFillModeBackwards（与kCAFillModeForwards相反），动画序列为1，2，3，4，4（最后一秒为4是由于动画不消除）；如果fillMode是kCAFillModeBoth，则动画序列为1，2，3，4，5；默认的kCAFillModeRemoved则动画序列为空，2，3，4，空。解释应该比较清楚了吧。

回到最初的CABasicAnimation基础动画：

```
 CABasicAnimation *pulseAnimation = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
  pulseAnimation.duration = .5;
  pulseAnimation.toValue = [NSNumber numberWithFloat:1.1];
  pulseAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
  pulseAnimation.autoreverses = YES;
  pulseAnimation.repeatCount = FLT_MAX;

```

第一句`+ (instancetype)animationWithKeyPath:(NSString *)path;` 表示需要修改的Layer属性，例如“position”，”tansform.scale”，可以对哪些属性进行修改，可以参考[官方文档](),后面几句设置的duration、autoreverses、repeatCount都已经解释过，toValue是设置动画属性的目标值，timingFunction是至动画的时间函数，就是个简单的贝塞尔曲线（Bezier Curve）,如下图所示：

{% img https://developer.apple.com/library/ios/documentation/cocoa/conceptual/animation_types_timing/Art/standardtiming_2x.png %}

分别代表kCAMediaTimingFunctionLinear, 
kCAMediaTimingFunctionEaseIn, 
kCAMediaTimingFunctionEaseOut, 
kCAMediaTimingFunctionEaseInEaseOut四类时间函数，简单点理解就是动画是匀速的(kCAMediaTimingFunctionLinear)还是先快后慢（kCAMediaTimingFunctionEaseOut），还是先慢后快（kCAMediaTimingFunctionEaseIn），还是慢快慢（kCAMediaTimingFunctionEaseInEaseOut）。kCAMediaTimingFunctionDefault对应的时间函数就是四个点决定的三次贝塞尔曲线：

{% img http://ww2.sinaimg.cn/large/65cc0af7gw1dxm21gxjr0j.jpg %}

自定义时间函数当然也是可以的，参考[How to create custom easing function with Core Animation](http://stackoverflow.com/questions/5161465/how-to-create-custom-easing-function-with-core-animation)

如果时间函数无法用简单的贝塞尔曲线表达，那么就得用到另一种动画了CAKeyframeAnimation关键帧动画，Part 2再继续聊。关于时间函数可以参考卢克的Blog [漫谈iOS Animation](http://geeklu.com/2012/09/animation-in-ios/)或者官方文档[Animation Types and Timing Programming Guide](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/animation_types_timing/articles/PropertyAnimations.html)

Example Source：[CA360](https://github.com/neror/CA360)