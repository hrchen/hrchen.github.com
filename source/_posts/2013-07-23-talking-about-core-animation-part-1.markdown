 ---
layout: post
title: "Core Animation编程 Part1/2"
date: 2013-07-23 18:13
comments: true
categories: 
---

今天开始聊聊iOS中牛逼闪闪的Core Animation，这个绝对是最秒杀Android的特性，毕竟iOS源于Mac OS，玩了那么多年，API自然成熟很多。先来了解下iOS中Core Animation的层次，

{% img /images/post/core-animation-architecture.jpg %}

最上层是UIKit，第二层是Core Anmation，然后iOS把这些动画效果用Open GL将在硬件上渲染出来。UIKit层中的动画API是作用于UIView，Core Animation层的动画API是作用于CALayer。我个人学习iOS的一个方法就是看类的头文件和相应的官方Reference文档。头文件可以很快查找出类的继承结构、属性和接口情况；Reference文档会详细介绍类的属性、方法以及需要了解的核心信息，没什么比这个文档说得更清楚了。

[UIView Class Reference](http://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIView_Class/UIView/UIView.html)中提到UIView主要负责三件事：

(1) 绘图和动画：利用UIKit、Core Graphics或者OpenGL ES绘制视图内容；有些UIView视图属性支持动画。

(2) 布局和子视图管理；一个视图可以关联零个或者多个子视图，如果有子视图，还可以定义子视的大小和位置；每个视图会定义它相对于父视图的尺寸变化(resizing)规则

(3) 事件处理：UIView继承自UIResponder，因此可以处理Touch事件和其他UIResponder中定义的事件；可以添加UIGestureRecognizer到UIView上，从而处理常见的几种手势操作。

那么UIView中哪些属性可以支持动画呢？有这些：frame，bounds，center，alpha，backgroundColor，contentStretch(iOS 6之后不推荐使用，已经Deprecated)，transform。除了transform外，其他属性都很直观，transform是CGAffineTransform类型（[CGAffineTransform Reference](http://developer.apple.com/library/ios/#documentation/GraphicsImaging/Reference/CGAffineTransform/Reference/reference.html)）。这里的transform是Affine Transform，即仿射变换，可以将点从一个坐标系映射到另一个坐标系，可以实现图形的拉伸、旋转、平移，详细点请看Reference。

Core Animation实际上就是作用在UIVIew属性layer上的动画，layer的类型是CALayer，定义在QuartzCore Framework中。从CALayer.h可以发现它和UIView实在太像了。需要主要它需要实现NSCoding和CAMediaTiming协议，这个CAMediaTiming协议很有意思，稍后再说。QuartzCore Framework中除了CALayer以其各种子类外，比较重要的就是CAAction、CAAnimation、CAMediaTiming和CAMediaTimingFunction。

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


说完CALayer，可以直接上Example，推荐Github的[CA360](https://github.com/neror/CA360),常见的Core Animation类型都提到了。
