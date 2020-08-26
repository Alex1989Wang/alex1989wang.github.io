---
layout: post
title: iOS应用的内存管理（二）
subtitle: Memory Management For iOS Apps - No.2
date: 2018-01-15 22:19:59

tags:
- iOS Programming
- Memory 

categories:
- Chinese
---

在[上一篇博客中](http://www.awsomejiang.com/2018/01/14/Memory-Management-for-iOS-Apps/)，主要介绍了对于一个对象的持有关系在MRR下是如何实践的。这篇文章主要介绍iOS系统中`Virtual Memory`的一些相关概念。这两个领域相关性并不大：前者是关于对象生命周期的管理；后者是`virtual memory`是如何使得设备的`RAM`得到了高效的利用。

虚拟内存`Virtual Memory`实际上是操作系统`Operating System`或者[计算机组成](https://www.youtube.com/watch?v=4CI-4aWDzgw)`Computer Structure`学科的概念。

<!-- more -->

## 虚拟内存的产生

虚拟内存产生的主要驱动就是早期计算机的物理内存RAM比较小，同时增大RAM的代价也非常高。在这样的低RAM的条件下，仍然希望操作系统能够多任务工作，运行多个程序。虚拟内存从不同的几个方面比较好的解决了这个问题，使得操作系统在一定程度上脱离了物理的RAM带来的限制。现在的操作系统，几乎无一例外都具备了虚拟内存；但不同的系统平台，虚拟内存的具体机制会有一些区别（如，iOS和Mac OS的机制就不太相同）。

### 关于`VM`的一些概念

具有`VM`机制的操作系统，会对``每个运行的进程``创建一个逻辑地址空间`logical address space`或者叫虚拟地址空间`virtual address space`；该空间的大小由操作系统位数决定：32位的操作系统，其逻辑地址空间的大小为4GB，64位的操作系统为18 exabyes（其计算方式是2^32 || 2^64）。

下面是一张来自[WWDC Session: iOS App Performance](https://developer.apple.com/videos/play/wwdc2012/242/)关于进程地址空间的说明图：

<div align='center'>
<img 
src="/images/vm_address_space_32_bit.png" 
width="600" 
title = "进程虚拟地址空间 - 来自WWDC Session"
alt = "进程虚拟地址空间 - 来自WWDC Session"
align = center
/>
</div>

虚拟地址空间(或者`逻辑地址空间`)会被分为相同大小的块，这些块被称为`内存页（page）`。计算机处理器和它的`内存管理单元（MMU - memory management uinit）`维护着一张将程序的逻辑地址空间映射到物理地址上的`分页表page table`。

在OS X和早版本的iOS中，分页的大小为4kB。在之后的基于A7和A8的系统中，虚拟内存（`64位的地址空间`）地址空间的分页大小变为了16KB，而物理RAM上的内存分页大小仍然维持在4KB；基于A9及之后的系统，虚拟内存和物理内存的分页都是16KB。[官方参考资料](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html#//apple_ref/doc/uid/20001880-BCICIHAB)

当某个进程的代码想要获取某一地址的数据时，MMU会使用分页表将该进程使用的`逻辑地址`转换为明确的物理地址，从而获得物理RAM上的数据。该过程叫做地址转换`address translation`；为了能够高效地转化地址，[这里还有很多技术](https://www.youtube.com/watch?v=ZjKS1IbiGDA&list=PLiwt1iVUib9s2Uo5BeYmwkDFUh70fJPxX&index=6)，不会在博客中讨论。

对于一个进程而言，逻辑地址空间中的地址对它来讲是分配即可获得`(原文为：As far as a program is concerned, addresses in its logical address space are always available.)`。这个意思是说，比如使用`malloc`分配一块10MB内存：
```objc
- (void)allocateSomeMemory {
     void *buf = malloc(10 * 1024 * 1024);
     for (unsigned int i = 0; i < sizeof(buf), i++) {
         buf[i] = (char)random();
	 }
... 
}
```
当执行`void *buf = malloc(10 * 1024 * 1024);`时，我们得到的是逻辑地址空间中的一块区域`region`。然而，如果我们在使用这块内存的时候`buf[i] = (char)random();`，如果该段内存页还没有映射到某段物理内存中，页错误`page fualt`就会发生。当页错误的时候，系统会立即调用`page-fault hanlder`来处理该页错误。`page-fault handler`会停止该进程上的代码运行，找到物理内存上的一段对应大小的空页，从硬盘或者闪存盘中加载相应数据到刚才找到的内存页中；同时，更新页表`page table`中的映射关系。之后，回到该被暂停执行的进程代码，继续执行；这时后，该进程代码又可以正常了获取内存中的数据了。[这整个过程被叫做内存分页（`paging`）](https://zh.wikipedia.org/wiki/%E5%88%86%E9%A0%81)。在`paging`结束之后，我们才真正意义上获得了物理内存。
         
如果在物理RAM上已经没有空白内存页可以使用了，`page-fualt handler`必须要释放空间来构建新的分页。清理内存分页的机制在不同的系统平台上实现可能是不一样的。在OS X，上虚拟内存这套机制在这个时候会将某些不活动的内存页写到后备存储（backing store）中。该后备存储通常就是机器的磁盘。将物理内存中的数据写入到计算机后备存储的过程叫做`paging out`或者`swapping out`，相反过程，将数据从`backing store`中移动至物理内存被称为`paging in`或者`swapping in`。在iOS平台上，`paging out`过程是没有的，但是只读的数据||代码分页在需要的时候是可以从闪存盘被`paging in`到物理内存中。

任何形式的paging过程都是非常耗时的。据说在极端情况下，如果需要读入的数据不在RAM中而且RAM上也没有空闲内存页的情况下：需要将部分数据paging out来腾出物理内存上的空间，同时paing in磁盘上的这些待读取数据来填充这个空间，[之后再读取RAM上的这些数据的这个过程耗时可能大于网络条件较好时，读取远程服务器上的数据](https://www.youtube.com/watch?v=bShqyf-hDfg&list=PLiwt1iVUib9s2Uo5BeYmwkDFUh70fJPxX&index=8)。

## Clean Memory Vs. Dirty Memory

`Memory`的分类中有两个很重要的概念：`clean`和`dirty`。这两个分类区别了在内存不足的时候，iOS系统内核会如何处理它们。前面我们已经提到过，iOS系统的`VM`机制不像许多其他的系统（MAC OS || Linus等）在内存不足的时候采用`Swap Out || Page Out`方式将不活动的进程的内存写入到后备存储中。在iOS上，如果系统已经到了没有内存可以清理腾出空间的时候，前台应用仍然在继续申请内存，系统的唯一手段就是终止这个应用的运行。在调试状态下，XCode的调试控制台会输出`Terminate app due to memory issue`的信息；如果是在非调试状态下，也会在设备上留下低内存的日志。

设备低内存日志：

<div align='center'>
<img 
src="/images/low_memory_syslog.png" 
width="600" 
title = "设备低内存日志"
alt = "设备低内存日志"
align = center
/>
</div>

那`Clean Memory`和应用由于没有办法分配内存而被终止的有什么联系？iOS应用在没有办法分配内存而被系统Terminate应用之前，系统会尝试清理一些`clean memory`。

<div align='center'>
<img 
src="/images/system_response_to_low_memory.png" 
width="600" 
title = "系统如何处理低内存 - WWDC Session 242"
alt = "系统如何处理低内存 - WWDC Session 242"
align = center
/>
</div>

为什么`Clean Memory`能够在内存紧张的时候被回收而对正在运行的前台应用没有影响呢？因为存在于`Clean Memory`上的数据在磁盘上是有一个完整的备份的（`memory for which a copy exsits on disk`），也就是在系统需要的时候还能够完全被重新创建。在[WWDC Session - iOS App Performance: Memory](https://developer.apple.com/videos/play/wwdc2012/242/)中提到，`Clean Memory`包括了：`code`、`frameworks`和`memory-mapped files`（不是很明白这个具体有哪些）。<span style="border-bottom:2px solid red;">相对应的，任何其他的内存都叫做`Dirty Memory`。在内存不足的情况下，系统无论如何都没有办法清除（`evict`）`dirty memory`，因为它们没有办法在需要的时候被原样重建；清除它们的话，正在运行的前台应用一定会受到影响，其结果和terminate前台应用没有区别。</span>

总结来讲，`Clean Memory`和`Dirty Memory`最大的区别就是是否能够被原样重建。

[WWDC Session - iOS App Performance: Memory](https://developer.apple.com/videos/play/wwdc2012/242/)中些示例来具体的区别Memory是Clean||Dirty的：

```objc
- (void)displayWelcomeMessage {
	NSString *welcomeMessage = [NSString stringWithUTF8String:"Welcome to WWDC!"]; //Dirty 
	self.alertView.title = welcomeMessage;
	[self.alertView show];
}

- (void)displayWelcomeMessage {
	NSString *welcomeMessage = @"Welcome to WWDC!"; //Clean
	self.alertView.title = welcomeMessage;
	[self.alertView show];
}
```
```objc
- (void)allocateSomeMemory {
	void *buf = malloc(10 * 1024 * 1024); //Clean
	for (unsigned int i = 0; i < sizeof(buf), i++) {
		buf[i] = (char)random(); //Dirty
	} 
	…
}

其中void *buf = malloc(10 * 1024 * 1024);只是申请了VM地址空间，并没有修改(modify)
这些地址内的数据，所以是Clean的；
但是buf[i] = (char)random();是写入了应用的产生的数据，
修改了这些地址内的数据，所以变为了Dirty的；
```
实际上，在我们的应用的内存分类中，大部分都是`Dirty Memory`。而正是`Dirty Memory`的大小以及变化是我们应该关心和控制的。

在我们的应用（前台应用）运行的不同阶段，内存`Clean Memory`&&`Dirty Memory`的大小会动态地变化：

<div align='center'>
<img 
src="/images/app_memory_uasage_change.png" 
width="600" 
title = "应用运行的不同阶段内存变化"
alt = "应用运行的不同阶段内存变化"
align = center
/>
</div>

- `Start State`是没有开始运行我们的应用的状态下可能的初始`Clean Memory`和`Dirty Memory`的一种分布。
- 当我们启动应用，应用开始工作，开始生成和使用较大量的`Dirty Memory`。一部分`Clean Memory`被回收，腾出的空间给持续增长的`dirty memory`;另外，一小部分的`free memory`也直接被利用。
- 当正在运行的应用的`Dirty Memory`过大使得系统产生内存压力`memory pressure`的时候，系统会选择杀死一些`background apps`来直接回收掉他们的`Dirty Memory`；同时在可能条件下，也继续回收`clean memory`。
- 由于杀掉了一些后台应用，使得`free memory`增加。在继续使用应用的的时候，`clean memory`被回收的必要性因为足够的`free memory`而降低，因而出现了`clean memory`的增长。

## <span id="Reference">参考资料</span>
- [MIT open course - computation structures](https://6004.mit.edu/)
- [Virtual Memory](https://www.youtube.com/watch?v=qcBIvnQt0Bw&list=PLiwt1iVUib9s2Uo5BeYmwkDFUh70fJPxX)
- [Fixing Memory Issues](https://developer.apple.com/videos/play/wwdc2013/410/)
- [iOS App Performance: Memory](https://developer.apple.com/videos/play/wwdc2012/242/)
- [Memory Usage Performance Guidelines](https://developer.apple.com/library/content/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)

