---
layout: post
title: "关于+initialize和+load的坑"
date: 2013-07-03 22:29
comments: true
categories: iOS
---

NSObject有两个特殊的类方法+initialize和+load。+initialize会在类的任何其他函数调用前被调用，因此也可以利用这个特性实现Singleton单例：

```
static Manager +theManager = nil;

+ (void) initialize 
{
	if (self == [Manager class]) 
	{
		theManager = [[Manager alloc] init];
	}
}

+ (Manager *)sharedObj 
{
  return theManager;
}
```

+load方法是在所在类加载到系统的时候被调用，这通常会比+initialized调用的时机要早，不过通常由于运行环境还有太多不确定性，不建议在+load中调用实际的方法。虽然Apple文档里说+initialized和+load都只会被执行一次，但是这里有坑。

如果子类里没有实现+initialized而父类里面实现了+initialized，那么用到子类时，不是说一定要生成对象，+initialize是调用任何方法，包括类方法，例如[SubClass class]，那么父类的+initialized就会被执行两次！解决办法也很简单，就像开头的写法`if (self == [Manager class]) `，先判断下是不是当前类的类型。

那么对于+load呢？如果你在类的实现中实现了+load，但是在这个类的Category中又实现了一个+load，那么这两个+load都会被调用。

既然这两个方法都是如此的诡异，所以除非必要，最好都不要在这两个方法中执行太多的操作，尤其是+load。绕过坑，远离危险。


