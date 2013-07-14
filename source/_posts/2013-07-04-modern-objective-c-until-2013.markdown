---
layout: post
title: "几个有用的Objective-C新特性"
date: 2013-07-04 21:20
comments: true
categories: iOS
---

Objective-C已经稳定在TIOBE编程语言排行榜前五名，2010年刚接触Objective-C还是因为公司在搞Mac版企业IM开发，那时候OC还几乎无人问津，这些年倒是风光无限，只能感叹当初踩对点了，不清楚未来10年又会有哪些语言流行起来，下一个会不会是[go](http://golang.org/)？。这几年来Objective-C的进化速度也是非常快，Apple不断添加新的特性到Objective-C，例如ARC、Block等，以下挑些个人感觉对开发效率影响比较大的新特性来说：

###不用再写sythesize
以前声明属性Property，都要在类的实现@implementation里将属性和成员变量做相应的synthesize，synthesize的含义是将属性和成员变量做关联。早期声明一个属性，需要同样声明一个成员变量，然后`@synthersize date=_date;`将属性与成员变量关联起来，后来可以无需声明成员变量，`@synthersize date=_date;`可以自动帮你声明一个名字为_date的成员变量，`@synthersize date`就是自动声明一个成员变量date。

Xcode4.4以后，synthesize关键字也不需要写了，例如`@property (nonatomic, readwrite, retain) NSDate *date;`编译器可以自动绑定并且生成对应的成员变量`NSDate *_date`，相当于是自动添加了`@synthesize date=_date;`。这样既不用再声明成员变量，也不要费神写@synthesize，方便不少。

<!--more-->


当然凡事有例外，如果同时实现了setter和getter方法，例如上面你实现了`-(void)setDate`和`-(NSDate *)date`，那么编译器就不会自动帮你synthesize。这里同时实现setter和getter方法是针对readwrite属性来说的，对于readonly属性，那么你实现了getter方法即`-(NSDate *)date`也同样不会自动绑定成员变量。


###成员方法的顺序

以前在.m实现文件中实现方法时经常会引用其他成员方法，而如果引用的成员方法未在头文件或者匿名catrgory中声明，同时也不在引用者前面，那么编译器会报未找到该方法的错误。现在新的编译器中，只要在实现文件里的成员方法，在其他任何位置的方法中调用都不再报错，Nice!


###不一样的NSNumber、NSArray和NSDictionary

最新的OC语法里还添加了许多类似脚本语言的特性，例如以前要生成NSNumber满费劲，都是[NSNumber numberWith***]的写法，太多冗余。现在方便了，可以用@符号替代，例如`[NSNumber numberWithChar:‘c’]`可以直接表示为`@'c'`，`[NSNumber numberWithInt:123]`直接表示为`@123`，`[NSNumber numberWithFloat:1.23f]`z直接写为`@1.23f`，其他类型同理变换。

NSArray的变化也是类似的，`[NSArray array]`就是`@[]`，`[NSArray arrayWithObject:x]`就是 `@[x]`，`[NSArray arrayWithObjects:x, y, z, nil]`就是`@[x, y, z]`，不过这种方式生成的是NSArray，要生成NSMutableArray呢？也简单，直接调用mutableCopy即可，例如`[@[x, y, z] mutableCopy]`。如果要访问第1个元素，以前需要写成`[array objectAtIndex:0]`，现在可以直接用`array[0]`访问，像极了脚本语言。

NSDictionary的变化和NSArray类似，不同的是用`@{}`,例如`[NSDictionary dictionaryWithObject:value forKey:key]`可以表示为`@{key: value}`。访问时也和大多数脚本语言一样，用`dict[key]`来获得键值对应的值。

至于以上简化的方法到底要不要用，还是看自己或者项目组的习惯了，个人建议是在符合统一编码规范的情况下，尽量拥抱变化，毕竟这些都是为了优化生产效率的变化。