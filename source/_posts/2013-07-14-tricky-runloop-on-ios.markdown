---
layout: post
title: "iOS中Run Loop的那些坑"
date: 2013-07-14 21:20
comments: true
categories: iOS
---

前段时间写了个关于iOS多线程编程的系列：

* [iOS多线程编程Part 1/3 - NSThread & Run Loop](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)
* [iOS多线程编程Part 2/3 - NSOperation](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-2/)
* [iOS多线程编程Part 3/3 - GCD](http://www.hrchen.com/2013/07/multi-threading-programming-of-ios-part-3/)

刚好最近[objc.io](http://www.objc.io/)第二期也在谈论"Concurrent Programming"，其中也提到了Run Loop在并发编程中的作用，可惜不是很深入。我们平时在使用Run Loop时仍然会遇到很多坑，本文在[iOS多线程编程Part 1/3 - NSThread & Run Loop](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)基础上把Run Loop的那些坑再总结一下。

{% img /images/post/runloop_source.jpg %}

回顾一下，Run Loop就是个循环，会不停的检查它的事件源有没有事件发生，如果有事件发生就处理事件或者调用事件的处理方法。