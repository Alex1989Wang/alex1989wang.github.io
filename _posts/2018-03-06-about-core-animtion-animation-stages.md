---
layout: post
title: Core Animation动画的不同阶段
date: 2018-03-06 10:46:38
tags:
- Animation 

categories:
- Chinese
---

之前在阅读[Core Animation Programming Guide](http://www.awsomejiang.com/2017/09/09/about-core-animation/)的时候写过一篇很短的[总结](http://www.awsomejiang.com/2017/09/09/about-core-animation/)；内容不多，主要用来整理一些当时吸收到的知识。最近，又把文档看了一遍，而且开始逐步做了[翻译](https://alex1989wang.gitbooks.io/core-animation/content/)，有了更多新的体会。另外，WWDC这个关于[Optimizing 2D Graphics and Animation Performance](https://developer.apple.com/videos/play/wwdc2012/506/)中对于还介绍了`Core Animation`在动画之前的各个阶段准备。

动画不同的阶段和动画性能的调优密不可分。

这篇博客主要是对上篇博客的补充，内容基本都来自于上面提到的WWDC的视频前面部分；另外自己使用了一个简单的[Demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest)和对这个Demo的profile数据来验证视频内容。

<!-- more -->

## 动画的阶段

首先动画的整个过程可以分为大的三个阶段：

- 创建动画和更新视图层级(create animation and update view hierachy)
- 准备和提交动画(prepare and commit animation)
- 渲染每一帧(render each frame)

<div align='center'>
<img 
src="/images/core-animation-stages.png" 
width="470" 
title = "动画大致的三个阶段"
alt = "动画大致的三个阶段"
align = center
/>
</div>

实际上，对于开发者而言，这三个大的阶段中的后两个，我们都是无法直接干预的。阶段二，由`Core Animation`代劳；阶段三是通过进程通信`IPC - Inter-Process Communication`将图层数据给到了另外一个`render server`的线程去渲染绘制的。这个`render server`据说在iOS 6之前叫`SpringBoard`，之后叫`BackBoard`(参见iOS Core Animation: Advanced Techniques - chapter 12)。

对于上面提到的前面两个阶段，可以通过`Time Profiler`在动画过程中对`call stack`的采样，来将这个过程更加具体的细分为四个步骤。
WWDC[Optimizing 2D Graphics and Animation Performance](https://developer.apple.com/videos/play/wwdc2012/506/)使用的Demo动画为：

<div align='center'>
<img 
src="/images/core-animation-stage-code-list.png" 
width="390" 
title = "测试动画的代码样例"
alt = "测试动画的代码样例"
align = center
/>
</div>

我们将这段代码整理到[Demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest)中：

```objc
- (void)startTestAnimation {
    //used to profile core animation's possible call stack trace.
    NSBundle *mainBundle = [NSBundle mainBundle];
    UINib *viewNib = [UINib nibWithNibName:@"JWAnimationTestView" bundle:mainBundle];
    NSArray *views = [viewNib instantiateWithOwner:nil options:nil];
    JWAnimationTestView *animationView = [views firstObject];
    animationView.frame = CGRectMake(0, 0, 300, 500);
    animationView.backgroundColor = [UIColor brownColor];
    animationView.transform = CGAffineTransformMakeScale(0.01, 0.01);
    [UIView animateWithDuration:3
                     animations:
     ^{
         [self.view addSubview:animationView];
         animationView.transform = CGAffineTransformIdentity;
     }
                     completion:
     ^(BOOL finished) {
         [animationView removeFromSuperview];
     }];
}

```

需要特别指出的是`JWAnimationTestView`的视图结构中，包括了一个*有文字*的`UILabel`和*有图片*的`UIImageView`。因为文字和图片的存在，会调用相应的文字绘制和将图片处理设置给`layer`的`contents`属性这些操作，所以这样的设定是为了能够看到完整的`call stack`。

另外，我的Demo代码中使用了一个重复调用的timer以5.f秒的时间间隔调用`startTestAnimation`方法。这样做是为了让`time profiler`能够采样到足够多的`call stack`数据，我自己在验证的过程中，让动画运行了10次。

<div align='center'>
<img 
src="/images/core-animation-call-stack-mine.png" 
width="580" 
title = "自己测试得到的call-stack采样"
alt = "自己测试得到的call-stack采样"
align = center
/>
</div>

上面红框部分是时钟方法的重复调用：创建视图，开始动画。

下面部分是`Core Animation`在将图层数据和相关信息提交给`render server`之前所做的事情。你会发现`Core Animation`是注册了主线程`runloop`的观察者，在一次`runloop`的结束的时候，`runloop`会回调`Core Animation`的观察者，让它执行相应的layout和显示的方法；实际上，我们通常对某个视图的图层属性所做的修改，都是在当前`runloop`结束之前，`Core Animation`才将修改提交。（这是不是为什么不能在子线程上去做UI相关的工作的原因？因为`Core Animation`的提交是在主线程`runloop`上进行的）。

`time profiler`的[call statck以1ms左右频率采样本身可能会漏采样执行时间很短方法](https://developer.apple.com/videos/play/wwdc2016/418/)；另外，WWDC中用于动画的视图结构和我Demo中使用的也可能不同。因此我获得的`call stack`和WWDC中展示的有些不同。

<div align='center'>
<img 
src="/images/core-animation-call-stack-mine-2.png" 
width="580" 
title = "自己测试得到的call-stack采样"
alt = "自己测试得到的call-stack采样"
align = center
/>
</div>

WWDC中展示的call stack采样：

<div align='center'>
<img 
src="/images/core-animation-wwdc-call-stack.png" 
width="580" 
title = "wwdc展示的call-stack采样"
alt = "wwdc展示的call-stack采样"
align = center
/>
</div>

当中有一个很大的区别：

在wwdc展示的call stack中，在`CA::Context::commit_transaction(CA::Transaction*)`方法下级，有下列几个过程：

- `CA::Layer::layout_and_display_if_needed(CA::Transaction*)`代表layout和dispaly两个步骤：
    - `CA::Layer::layout_if_needed(CA::Transaction*)` layout过程
    - `CA::Layer::display_if_needed(CA::Transaction*)` dispaly过程
- `CA::Layer::prepare_commit(CA::Transaction*)` 提交准备阶段
- `CA::Layer::commit_if_needed(CA::Transaction*, void (*)(CA::Layer*, unsigned int, unsigned int, void*), void*)`提交阶段

在我们的call stack中是看不到`CA::Layer::layout_and_display_if_needed(CA::Transaction*)`，而是由`CA::Context::commit_transaction(CA::Transaction*)`直接调用的`CA::Layer::layout_if_needed(CA::Transaction*)`以及`[CALayer _display]`。按照`Time Profiler`的采样原理，如果能够采样到下级方法的调用，那么该方法的caller一定是也能够采样到的。在我们的call stack中，既然有`CA::Layer::layout_if_needed(CA::Transaction*)`，但是没有`CA::Layer::layout_and_display_if_needed(CA::Transaction*)`是不是说明`Core Animation`内部的代码结构已经不一样了？毕竟WWDC是2012年的。

不管怎样，从我们的call stack中也能够看到这四个完整的过程。

结合WWDC，以我们的call stack为例，来说明这四个过程分别大概都做了什么。

### layout过程

<div align='center'>
<img 
src="/images/core-animation-stage-layout.png" 
width="400" 
title = "layout的过程"
alt = "layout的过程"
align = center
/>
</div>

从上面layout的过程可以看出，其所做的主要任务就是将图层调用代理（也就是视图）实现整个视图层级的布局；比较有意思的是，`autolayout`的约束也是在这个时候更新和施加`apply`的（`-[UIView(Hierarchy) _updateConstraintsAsNecessaryAndApplyLayoutFromEngine]`）。

### display过程

<div align='center'>
<img 
src="/images/core-animation-stage-display.png" 
width="400" 
title = "display的过程"
alt = "display的过程"
align = center
/>
</div>

按照WWDC视频的说法，`display`这个阶段是负责draw图层的内容。如果你的某些视图实现了`drawRect:`方法或者其他UIKit提供的视图采用了图层的`drawInContext:`方法来提供内容的，比如这里的UILabel的实现，那么也会在这个阶段去调用。从这个阶段的工作来讲，其是`CPU bound`（受限于CPU性能）的。

### prepare过程 

<div align='center'>
<img 
src="/images/core-animation-stage-prepare.png" 
width="400" 
title = "prepare的过程"
alt = "prepare的过程"
align = center
/>
</div>

prepare过程也是`Core Animation`准备图层内容的环节，但是这个过程的准备是给那些没有实现`drawInContext:`(视频说的是`drawRect`，但实际上`drawRect`的底层是图层的`drawInContext`)而通过其他方式设置图层内容的视图准备的；比如，UIImageView，它的实现就是直接解码压缩格式的png图，设置给其layer的contents属性的。那么图片解码的过程就在这个prepare阶段。

### commit过程

commit阶段是讲所有的图层数据打包，通过进程间通信给到render server(另外一个进程)来绘制显示到屏幕上。通常情况下，CPU的工作在这个阶段是比较少的-从上面的10次重复动画来看，这个过程仅仅分担了1ms左右的cpu时间。但是WWDC指出，如果你的视图层级过于复杂，有非常多的图层，那这个commit的过程也会耗费比较长的时间。

<div align='center'>
<img 
src="/images/core-animation-stage-commit.png" 
width="400" 
title = "commit的过程"
alt = "commit的过程"
align = center
/>
</div>

## 参考资料

- [Core Animation Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html)
- [Optimizing 2D Graphics and Animation Performance](https://developer.apple.com/videos/play/wwdc2012/506/)
- [关于动画阶段的Demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest)

