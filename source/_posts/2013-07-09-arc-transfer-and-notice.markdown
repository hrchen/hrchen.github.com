---
layout: post
title: "ARC工程转换和开发注意事项"
date: 2013-07-09 17:40
comments: true
categories: iOS
---

本文介绍ARC工程转换流程和开发注意事项，ARC的意义不仅仅是开发效率的提高，更是一种思维方式的转变，无需再考虑什么地方调用retain/release，而是考虑对象的强/弱指针来控制对象所有权，还要继续关注循环引用的问题。###ARC工程转换####Xcode工程设置
1)	查看Xcode工程Build Settings设置中Builing Options中的Compiler选项，确保使用的是Apple LLVM compiler 3.0以上版本的编译器； {% img /images/post/xcode-compiler.jpg %}
2)	在Xcode工程的Build Settings开启ARC：搜索Objective-C Automatic Reference Counting；{% img /images/post/xcode-arc.jpg %} <!-- more -->
3)	打开Xcode的Prefernece设置中的General，开启Coninue building after errors。{% img /images/post/xcode-settings.jpg %}####开启ARC转换工具
1) 打开Xcode->Edit->Refactor->Convert to Objective-C ARC…{% img /images/post/xcode-refactor-arc.jpg %} 2) 注意只选择本工程相关文件，第三方库如果一般不要进行ARC转换，如果有对应ARC版本库可以直接替换。{% img /images/post/xcode-arc-check.jpg %} 正常情况下不会顺利完成Check，会有大量需要手动修改的Error，常见的问题有以下这些：
1)	调用 [cell autorelease]、[object release]、[object retain]，直接删除即可，这种应该属于Checker的误报，正常是可以直接Refactor的。
2)	CoreFoundation对象与NSObject对象的转换，需要添加__bridge, __bridge_retained或者__bridge_trasfer。
CoreFoundation的对象例如CFStringRef有自己的引用计数，和Cocoa框架中的NSObject是不同的方法，ARC只对NSObject对象的引用计数有效。只要是生成CF对象的函数名中有含有Create, Copy, 或者Retain，就表示需要为它的引用计数负责，需要使用结束时CFRelease()将引用计数减一。
例如：如果使用了一个含有reate, Copy, 或者Retain的方法生成了一个CFStringRef name，那么在转换成NSString时，就需要写成NSString *nameString = (__bridge_transfer NSString *)name; 
* __bridge_transfer的含义表示将CF对象的管理权移至NSObject层由ARC负责，无需再用CFRelease()释放name这个CFStringRef。
* __bridge_retained的含义相反，就是将一个NSObject对象转换成CF对象，并且引用计数加一，那么在CF层用完这个CF对象后，就需要使用CFRelease()释放该对象，因为内存管理权已经由NSobject层转移至CF层。*  __bridge的含义表示在NSObject层和CF层引用计数都平衡，无需转移内存管理权。例如如果使用不包含reate, Copy, 或者Retain的函数获得的CFStringRef name转换成NSString时，无需处理引用计数问题，因此可以这样转换：NSString *nameString = (__bridge NSString *)name;
除了上面三个关键字，还有两个宏CFBridgingRetain()和CFBridgingRelease()来控制CF层与NSObject层的引用计数平衡，不过实际上他们就是__bridge_retained和__bridge_transfer。
```CFTypeRef CFBridgingRetain(id X) { return (__bridge_retained CFTypeRef)X; } id CFBridgingRelease(CFTypeRef CF_CONSUMED X) { return (__bridge_transfer id)X; }```
3)	NSInvocation方法
例如：

```NSString *date = @”test”;[writeInvocation setArgument:&data atIndex:2];
```

NSInvocation在设置调用参数时会提示：`NSInvocation's setArgument is not safe to be used with an object with ownership other than __unsafe_unretained`修改方法需要将传入的参数添加__unsafe_unretained关键字，即`NSString * __unsafe_unretained date = @”test”;`4)	NSAutoreleasePoolARC下不再支持NSAutoreleasePool，需要使用@autoreleasepool{}替换。5)	错误`Passing address of non-local object to __autoreleasing parameter for write-back`。
此错误通常是由于将非局部变量的地址传递给一个方法导致的，例如：
```//_array 和_dict是成员变量而非局部变量[CTViewController trainInfoList:&_array forSeats:&_dict];```
处理方法也比较简单，生成一个临时局部变量即可：
```NSArray *tempArray = nil;NSDictionary *tempDict = nil;[CTViewController trainInfoList:&tempArray forSeats:&tempDict];_array = tempArray;_dict = tempDict;```
###ARC开发注意事项1)	NSObject的 retain, release和autorelease都无需再调用，ARC会评估NSObject对象的生命周期，在编译器自动添加相应内存相关方法完成内存管理，并且会生成相应的dealloc方法，因此如果自定义的类如果没有需要内存管理外的操作(例如删除NSNotification的Observer以及将指向自己的delegate置为nil)，就无需再实现dealloc。
2)	不能在struct中使用NSObject对象的指针。
3)	使用@autoreleasepool{}取代NSAutoreleasePool。
4)	不再使用NSZone。
5)	新增属性关键字strong、weak、unsafe_unretained
6)	新增变量关键字__strong、__weak、__unsafe_unretained、__autoreleasing：
__strong表示这个变量指针是强指针，指向的对象只要有强指针指向它就不会被销毁；
__weak表示变量指针是弱指针，如果没有其他强指针指向这个对象时，这个对象就会被销毁，同时弱指针会置为nil；
__unsafe_unretained和__weak类似，除了在对象销毁后不会使这个__unsafe_unretained指针置为nil，因此这个指针就变成悬空指针！
__autoreleasing用于传递给方法的参数是引用传值，并且在返回时会autorelease。
正确的关键字写法：

```MyClass * __weak weakReference;MyClass * __unsafe_unretained unsafeReference;
```
7)	避免循环引用问题
循环引用问题一般在两个类对象相互引用和使用Block对象时出现，解决两个类对象相互引用问题，可以将其中一个引用声明为弱引用，就可以打破循环引用问题。
Block对象常见的循环引用问题如下：

```MyViewController *myController = [[MyViewController alloc] init…];myController.completionBlock= ^() { 	[myController dismissViewControllerAnimated:YES completion:nil]; }; ```
上面例子中会导致completionBlock和myController循环引用，正确的处理方法有两种：
* 一是使用__block关键字，之后将该__block变量置为nil
```MyViewController * __block myController = [[MyViewController alloc] init…];myController.completionBlock= ^() {	[myController dismissViewControllerAnimated:YES completion:nil]; 	myController = nil;}; 
```* 二是使用__weak关键字
```MyViewController * __block myController = [[MyViewController alloc] init…];MyViewController * __weak weakMyViewController = myController;myController.completionBlock= ^() {	[weakMyViewController dismissViewControllerAnimated:YES completion:nil]; }; ```8)	toll-free bridge问题
对于要使用Core Foundation的对象时，要注意对象生命周期的所有权问题，重点是三个关键字__bridge, __bridge_retained和__bridge_trasfer对CF对象和NS对象的转换，使用方法参见上节工程转换时候的说明。
9)	ARC时所有临时变量指针(栈内生成)都会初始化为nil。例如：

```- (void)test { NSString *name; NSLog(@"name: %@", name); }```
上面代码会打印nil。
10)	如果有需要ARC管理的文件，可以在Xcode中设置工程Target的Build Phase中Compiler Source，不需要ARC管理的文件添加编译参数“-fno-objc-arc”。
{% img /images/post/xcode-fno-objc-arc.jpg %}11)	在MRC情况下NSString * __block myString是不会被retain的，但是ARC情况下NSString * __block myString实际会被retian，如果需要和MRC下同样的语义，请使用：__block NSString * __unsafe_unretained myString 或者__block NSString * __weak myString。
12)	iOS 4.*系统不支持weak语义，可以使用unsafe_unretained替代，但是可能导致悬空指针问题，需要小心对待。





