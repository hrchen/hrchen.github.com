 ---
layout: post
title: "Core Animation编程 Part1/2"
date: 2013-07-23 18:13
comments: true
categories: 
---

今天开始聊聊iOS中牛逼闪闪的Core Animation，会分为两个部分。先来了解下iOS中动画的层次，

{% img /images/post/core-animation-architecture.jpg %}

最上层是UIKit，第二层是Core Anmation，然后iOS将动画用Open GL将效果在硬件上渲染出来。UIKit层中的动画是作用于UIView，Core Animation层的动画是作用于CALayer。我个人目前学习iOS的一个方法就是看类的头文件和相应的官方Reference文档。头文件可以很快查找出类的继承结构、属性和接口情况；Reference文档会详细介绍类的属性、方法以及需要了解的核心信息，没什么比这个文档说得更清楚了。

[UIView Class Reference](http://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIView_Class/UIView/UIView.html)中讲到UIView主要负责三件事：

(1) 绘图和动画：利用UIKit、Core Graphics或者OpenGL ES绘制视图内容；有些UIView视图属性支持动画。

(2) 布局和子视图管理；一个视图可以关联零个或者多个子视图，如果有子视图，还可以定义子视的大小和位置；每个视图会定义它相对于父视图的尺寸变化(resizing)规则

(3) 事件处理：UIView集成自UIResponder，因此可以处理Touch事件和其他UIResponder中定义的事件；可以添加UIGestureRecognizer到UIView上，从而处理常见的几种手势操作。

那么UIView中哪些属性可以支持动画呢？有这些：frame，bounds，center，alpha，backgroundColor，contentStretch(iOS 6之后不推荐使用，即Deprecated)，transform。除了transform外，其他属性都很直接，这里transform是CGAffineTransform类型（[CGAffineTransform Reference](http://developer.apple.com/library/ios/#documentation/GraphicsImaging/Reference/CGAffineTransform/Reference/reference.html)）。这里的transform是Affine Transform，即仿射变换，
