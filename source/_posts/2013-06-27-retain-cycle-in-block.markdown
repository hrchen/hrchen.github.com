---
layout: post
title: "Block的引用循环问题 (ARC & non-ARC)"
date: 2013-06-27 00:32
comments: true
categories: iOS

---

2010年WWDC发布iOS4时Apple对Objective-C进行了一次重要的升级：支持Block。说到底这东西就是闭包，其他高级语音例如Java和C++已有支持，第一次使用Block感觉满简单好用的，但是慢慢也遇到很多坑。本文聊聊ARC和non-ARC下Block使用中的引用循环问题，最近遇到了好几次这种问题，还是深入记录下。先来套[题目](http://blog.parse.com/2013/02/05/objective-c-blocks-quiz/)热热身，貌似能够全部答对的人蛮少的

###Block实现原理

首先探究下Block的实现原理，由于Objective-C是C语言的超集，既然OC中的NSObject对象其实是由C语言的struct+isa指针实现的，那么Block的内部实现估计也一样，以下三篇Blog对Block的实现机制做了详细研究：

* [A look inside blocks: Episode 1](http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-1/)
* [A look inside blocks: Episode 2](http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-2/)
* [A look inside blocks: Episode 3](http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-3/)

虽然实现细节看着头痛，不过发现Block果然是和OC中的NSObject类似，也是用struct实现出来的东西。这个是LLVM项目compiler-rt分析的block头文[Block_private.h](https://llvm.org/svn/llvm-project/compiler-rt/trunk/BlocksRuntime/Block_private.h)头文件中关于Block的struct声明：

<!--more-->

```
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};

```

我们发现Block_layout中也有一个isa指针，像极了NSobject内部实现struct中的isa指针。这里的isa可能指向三种类型之一的Block：

* _NSConcreteGlobalBlock：全局类型Block，在编译器就已经确定，直接放在代码段__TEXT上。直接在NSLog中打印的类型为\_\_NSGlobalBlock\_\_。
* _NSConcreteStackBlock：位于栈上分配的Block，即\_\_NSStackBlock\_\_。
* _NSConcreteMallocBlock：位于堆上分配的Block，即\_\_NSMallocBlock\_\_。

为什么会有这么多种类呢？首先来看全局类型Block，看例子：

```
void addBlock(NSMutableArray *array) {
  [array addObject:^{
    printf("global block\n");
  }];
}
 
void example() {
  NSMutableArray *array = [NSMutableArray array];
  addBlock(array);
  void (^block)() = [array objectAtIndex:0];
  block();
}

```
为什么addBlock中添加到array中的Block属于全局Block呢？因为它不需要运行时(Runtime)任何的状态来改变行为，不需要放在堆上或者栈上，直接编译后在代码段中即可，就像个c函数一样。这种类型的Block在ARC和non-ARC情况下没有差别。

这个Block访问了作用域外的变量d，在实现上就是这个block会多一个成员变量对应这个d，在赋值^block时会将方法exmpale中的d变量值复制到成员变量中，从而实现访问。

```
void example() {
	int d = 5;
	void (^block)() = ^() {
		printf("%d\n", d);
	};
	block();
}
```

如果要修改d呢？：

```
void example() {
	int d = 5;
	void (^block)() = ^() {
		d++;
		printf("%d\n", d);
	};
	block();
	printf("%d\n", d);
}
```
由于局部变量d和这个block的实现不在同一作用域，仅仅在调用过程中用到了值传递，所以不能直接修改，而需要加一个标识符`__block int d = 5;`，那么block就可以实现对这个局部变量的修改了。如果是这种__block标识的变量，在Block实现中不再是简单的一个成员变量，而是对应一个新的结构体表示这个__block变量。__block的本质是引入了一个新的__Block_byref_{$var_name}_{$index}结构体，被__block关键字修饰的变量就被放到这个结构体中。另外，block结构体通过引入__Block_byref_{$var_name}_{$index}指针类型的成员，得以间接访问到Block的外部变量。这样对Block外的变量访问从值传递转变为引用，从而有了修改内容的能力。

正常我们使用Block是在栈上生成的，离开了栈作用域便释放了，如果copy一个Block，那么会将这个Block copy到堆上分配，这样就不再受栈的限制，可以随意使用啦。例如：

```
typedef void (^TestBlock)();
 
TestBlock getBlock() {
  char e = 'E';
  void (^returnedBlock)() = ^{
    printf("%c\n", e);
  };
  return returnedBlock;
}
 
void example() {
  TestBlock block = getBlock();
  block();
}
```

函数getBlock中声明并赋值的returnedBlock，一开始是在栈上分配的，属于NSStackBlock，如果是non-ARC情况下return这个NSStackBlock，那么其实已经被销毁了，在函数中example()使用时就会crash。如果是ARC情况下，getBlock返回的block会自动copy到堆上，那么block的类型就是NSMallocBlock，可以在example()中继续使用。要在Non-ARC情况下正常运行，那么就应该修改为：

```
TestBlock getBlock() {
  char e = 'E';
  void (^returnedBlock)() = ^{
    printf("%c\n", e);
  };
  return [[returnedBlock copy] autorelease];
}
```

###Block中的循环引用问题
扯了这么多，回到Block的循环引用问题，由于我们很多行为会导致Block的copy，而当Block被copy时，会对block中用到的对象产生强引用(ARC下)或者引用计数加一(non-ARC下)。

如果遇到这种情况：

```
@property(nonatomic, readwrite, copy) completionBlock completionBlock;

//========================================
self.completionBlock = ^ {
        if (self.success) {
            self.success(self.responseData);
        }
    }
};
```
对象有一个Block属性，然而这个Block属性中又引用了对象的其他成员变量，那么就会对这个变量本身产生强应用，那么变量本身和他自己的Block属性就形成了循环引用。在ARC下需要修改成这样：

```
@property(nonatomic, readwrite, copy) completionBlock completionBlock;

//========================================
__weak typeof(self) weakSelf = self;
self.completionBlock = ^ {
    if (weakSelf.success) {
        weakSelf.success(weakSelf.responseData);
    }
};
```
也就是生成一个对自身对象的弱引用，如果是倒霉催的项目还需要支持iOS4.3，就用\_\_unsafe_unretained替代\_\_weak。如果是non-ARC环境下就将\_\_weak替换为\_\_block即可。non-ARC情况下，\_\_block变量的含义是在Block中引入一个新的结构体成员变量指向这个\_\_block变量，那么`__block typeof(self) weakSelf = self;`就表示Block别再对self对象retain啦，这就打破了循环引用。
