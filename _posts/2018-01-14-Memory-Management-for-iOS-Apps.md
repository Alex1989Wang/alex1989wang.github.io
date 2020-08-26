---
layout: post
title: iOS应用的内存管理（一）
subtitle: Memory Management for iOS Apps (No.1)
date: 2018-01-14 19:52:57

tags:
- iOS Programming
- Memory 

categories:
- Chinese
---

iOS的工作经验已经有一年半的时间了，对于`iOS内存管理`这个话题并没有去深入的研究过。`ARC`对于刚接触iOS编程的程序员来讲是很友好的，不再需要程序员手动地在正确的地方对对象的生命周期显式地控制；`ARC`的引入使得编译器能够在编译时（`compile-time`）自动地为代码添加`retain`、`release`和`autorelease`来保证对象能够有正确的生命周期。

尽管在大概半年前，自己阅读了[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)，对`MRR (manual retain release)或者MRC (manual reference counting)`的写法实践也基本了解。这些知识在使用`Instruments`中的内存诊断工具`VM Tracker`的时候，会发现`VM Tracker`面板上的统计指标有很多术语是不理解的。原因很简单`VM(vitual memory)`和`reference counting`是不相干的内容。

<!-- more -->

另外，同事发现新的iOS系统（``iOS11``)下跑`VM Tracker`可以看到`Swapped Size`这个类目。这些都触发了自己再去回读[参考资料](#Reference)中的文档和查看WWDC相关session内容。

所以，接下来的几篇博客将从`reference counting`到`Virtual Memory`，然后简单地说明`Instruments`内存诊断的一些概念和使用，基本都是这次文档和资料阅读的一次总结。

<div align='center'>
<img 
src="/images/vm-tracker-swapped-size.png" 
width="800" 
title = "VM Tracker中Swapped Size和细项统计"
alt = "VM Tracker中Swapped Size和细项统计"
align = center
/>
</div>

在以前系统版本(`iOS 11之前`)是没有这个类目的[VM Tracker Instrument](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Instrument-VMTracker.html)。同时，Apple关于`VM`的文档（[Memory Usage Performance Guidelines](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)）和WWDC的一次session（[iOS App Performance: Memory](https://developer.apple.com/videos/play/wwdc2012/242/)）也有明确地提出`iOS`的`VM`机制不存在像`Mac OS`一样的将不活动的进程的一些内存分页转移到`secondary storage`（也就是Mac的硬盘）上从而腾出空闲的虚拟内存分页的所谓`swap`机制：

> Paging Out Process

> In OS X, when the number of pages in the free list dips below a computed threshold, the kernel reclaims physical pages for the free list by swapping inactive pages out of memory. To do this, the kernel iterates all resident pages in the active and inactive lists, performing the following steps:
> 
> - If a page in the active list is not recently touched, it is moved to the inactive list.
> - If a page in the inactive list is not recently touched, the kernel finds the page’s VM object.
> - If the VM object has never been paged before, the kernel calls an initialization routine that creates and assigns a default pager object. The VM object’s default pager attempts to write the page out to the backing store.
> - If the pager succeeds, the kernel frees the physical memory occupied by the page and moves the page from the inactive to the free list.
> 
> Note: In iOS, the kernel does not write pages out to a backing store. When the amount of free memory dips below the computed threshold, the kernel flushes pages that are inactive and unmodified and may also ask the running application to free up memory directly. For more information on responding to these notifications, see Responding to Low-Memory Warnings in iOS.

所以这个问题让我和同事一起开始研究这个`VM`机制和`iOS`的内存管理。

这篇博客会主要介绍[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)中的内容，也就是`reference counting`的基础。而不会涉及到`virtual memory`的知识。计划会在后面的博客中来简单的介绍`iOS`系统下的`VM`机制和`Instruments`在查找内存问题上的使用。

## MRR时代的内存管理策略

不管是MRR(MRC)还是现在都在使用的`ARC`，它们的基础原理都是`retain count`（引用计数）这个东西。简单地讲，就是一个对象的引用计数不为0时，该对象就不会被销毁，其`dealloc`方法就不会被系统调用。在`MRR`时代，程序员需要手动地在代码中添加`retain/release/autorelease`方法来管理一个对象的持有关系（`ownership`)，从而管理该对象的生命周期。

`Objective-C`有一套特殊的方法命名规范来帮助程序员更好更正确地插入`retain/release/autorelease`方法；当然，除此之外还有一些其他的规则：
- <span style="border-bottom:2px dashed red;"><font size = 4><b>You own any object you create</b></font></span> You create an object using a method whose name begins with “alloc”, “new”, “copy”, or “mutableCopy” (for example, alloc, newObject, or mutableCopy). 使用以“alloc”, “new”, “copy”, or “mutableCopy”开头的方法创建返回的对象是你拥有的，也就是`retain count`在创建之后为1，需要你自己去释放。

- <span style="border-bottom:2px dashed red;"><font size = 4><b>You can take ownership of an object using retain</b></font></span> A received object is normally guaranteed to remain valid within the method it was received in, and that method may also safely return the object to its invoker. You use retain in two situations: (1) In the implementation of an accessor method or an init method, to take ownership of an object you want to store as a property value; and (2) To prevent an object from being invalidated as a side-effect of some other operation (as explained in Avoid Causing Deallocation of Objects You’re Using). 可以使用`-(void)retain`方法来持有某个对象（增加对该对象的引用计数）。在某个方法中获取到的对象通常情况下在该方法中是有效的，同时这个方法也可以安全地将该对象返回给其调用者。在两种情况下我们会使用`retain`：1. 在`accessor`或者`init`方法的实现中，使用`retain`来持有某个你想要用作属性的对象。2. 防止某个对象因为其他的操作影响而被释放（比如，对象被容器`NSArray||NSDictionary`唯一持有，然后调用容器的移除该对象的方法之后，我们任然想要使用该对象）。

- <span style="border-bottom:2px dashed red;"><font size = 4><b>When you no longer need it, you must relinquish ownership of an object you own</b></font></span> You relinquish ownership of an object by sending it a release message or an autorelease message. In Cocoa terminology, relinquishing ownership of an object is therefore typically referred to as “releasing” an object. 当你不再需要你持有的某个对象的时候，你必须放弃对它的持有。通过向该对象发送`release||autorelease`消息来放弃对某个对象的持有。在`Cocoa`的术语中，放弃对某个对象的持有在实际使用就就是`释放`某个对象的意思。

- <span style="border-bottom:2px dashed red;"><font size = 4><b>You must not relinquish ownership of an object you do not own</b></font></span> This is just corollary of the previous policy rules, stated explicitly. 如果某个对象并不是你所有，你一定不能够释放该对象。这条规则其实是上一条规则的结果，但是需要明确地说明。（`overrelease`在MRR中会导致应用崩溃）。

在[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)中，对于这些规则，有一个比较好的流程图：

<div align='center'>
<img 
src="/images/memory_management_retain_cout.png" 
width="800" 
title = "内存管理中retain count的变化"
alt = "内存管理中retain count的变化"
align = center
/>
</div>

对这张图实际上相对于原图做了一些修改，将原图标在节点处的`retain count`值放到了相应的``线段（也就是时间段）``上，这样更加符合实际的情况，也更方便理解`retain count`的变化过程。如果需要需要比较原图，这是[原图连接](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Art/memory_management_2x.png)。

在解释MRR的内容管理规则时，有`retain count`、持有和`retain||release||autorelease方法`等概念，实际上它们之间有明确的关系存在：
> The ownership policy is implemented through reference counting—typically called “retain count” after the retain method. Each object has a retain count.
> 
> - When you create an object, it has a retain count of 1.
> - When you send an object a retain message, its retain count is incremented by 1. 当向一个对象发送`retain`消息时，引用计数加1。
> - When you send an object a release message, its retain count is decremented by 1. 当向一个对象发送`release`消息时，引用计数减1。
> - When you send an object a autorelease message, its retain count is decremented by 1 at the end of the current autorelease pool block. 当向一个对象发送`autorelease`消息时，引用计数会在当前的autorelease pool作用域结束的时候减一。
> - If an object’s retain count is reduced to zero, it is deallocated. 当一个对象的引用计数减至0时，它就会被销毁。
> 
> <span style="border-bottom:2px solid red;">Important:</span> There should be no reason to explicitly ask an object what its retain count is (see retainCount). The result is often misleading, as you may be unaware of what framework objects have retained an object in which you are interested. In debugging memory management issues, you should be concerned only with ensuring that your code adheres to the ownership rules. 显示地询问一个对象的引用计数是没有什么鸟用的。而且，该引用计数的结果常常可能会具有误导性，因为一些`framework`的对象也可能持有你目前询问的这个对象。在debug内存问题的时候，你只应该关心你的代码是不是遵守了以上的一些持有规则。

## MRR时代的内存管理实践

内存管理实践实际上就是在实际的代码书写过程中，如何遵循上面的几条规则。针对上面的规则的一条或者几条，Apple的文档也给出了相应的示例：


### 规则1&&3
- <span style="border-bottom:2px dashed red;"><font size = 2><b>你持有你创建的对象</b></font></span> 
- <span style="border-bottom:2px dashed red;"><font size = 2><b>当你不再需要你持有的某个对象的时候，你必须放弃对它的持有。</b></font></span> 

```objc
{
    Person *aPerson = [[Person alloc] init];
    // ...
    NSString *name = aPerson.fullName;
    // ...
    [aPerson release];
}
```

使用`alloc`创建了`aPerson`对象（其`retain count`当前为1），在不再需要改对象时：也就是代码块作用域的末尾，对该对象发送`release`消息，放弃持有。

### 规章4
- <span style="border-bottom:2px dashed red;"><font size = 2><b>如果某个对象并不是你所有，你一定不能够释放该对象</b></font></span> 
 
<span style="border-bottom:2px solid red;">!!!错误示例:</span>
```objc
- (NSString *)fullName {
    NSString *string = [NSString stringWithFormat:@"%@ %@",
                                 self.firstName, self.lastName];
    return [string autorelease]; //!!!!!错误
}

该对象并不是通过`规则1`中的方法创建的，你并不持有该对象，并不需要你对它发送`autorelease`消息来管理内存。
```

### 规则2
- <span style="border-bottom:2px dashed red;"><font size = 2><b>可以使用`-(void)retain`方法来持有某个对象（增加对该对象的引用计数）</b></font></span> 

<span style="border-bottom:2px solid black;">示例1:</span>
```objc
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;

- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    // Make the new assignment.
    _count = newCount;
}
```
在`setter`中对新传入的对象增加持有，同时释放原来的对象。利用`setter`的封装也简化了内存管理，不需要将`retain||release`的代码零散在各个地方。

<span style="border-bottom:2px solid black;">示例2:</span>
```objc
objectOfInterest = [[array objectAtIndex:n] retain];
[array removeObjectAtIndex:n];
// Use heisenObject...
[objectOfInterest release];
```

如果某个容器对象`array`是某个需要使用的对象`objectOfInterest`的唯一持有者，若先进行移除操作`removeObjectAtIndex`，那么该对象将被释放，无法在之后被使用。所以在使用之前我们需要对他进行`retain`。

## autorelease pool的使用

一个`autorelease pool`的代码块是由@autoreleasepool来标记的：
```objc
@autoreleasepool {
    // Code that creates autoreleased objects.
}
```

在`autorelease pool`作用域结束时，之前被发送`autorelease`消息的对象，都会被发送`release`消息。若该对象之前收到过多个`autorelease`消息，在这个时候会收到相应数目的`release`消息。

`autorelease pool`可以嵌套多个：

```objc
@autoreleasepool {
    // . . .
    @autoreleasepool {
        // . . .
    }
    . . .
}
```

通常在iOS的编程中，使用嵌套`autorelease pool`的情形就是：在一次循环的loop中创建大量的临时对象。可以将loop的逻辑放在一个`autorelease pool`中，这样可以在下一次loop开始之前就释放掉上次loop中创建的一些`autorelease`对象，从而避免内存的峰值。当然，这个场景可以推广到任何可能产生大量临时对象的地方。


## <span id="Reference">参考资料</span>
- [Fixing Memory Issues](https://developer.apple.com/videos/play/wwdc2013/410/)
- [iOS App Performance: Memory](https://developer.apple.com/videos/play/wwdc2012/242/)
- [Adopting Automatic Reference Counting](https://developer.apple.com/videos/play/wwdc2012/406/)
- [Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)

