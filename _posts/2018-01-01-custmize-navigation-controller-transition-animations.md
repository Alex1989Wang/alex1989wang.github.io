---
layout: post
title: 视图控制器转换(二) View Controller Transistion - No.2
subtitle: 自定义Navigation转换动画 (customizing navigation transition)
date: 2018-01-01 22:07:03

tags:
- iOS Programming
- Animation

categories:
- Chinese
---

在[上一篇](http://www.awsomejiang.com/2017/12/25/View-Controller-Transistion-customizing-presentation-md/)中，讨论了基本的自定义视图控制器转场动画的步骤；同时，也以自定义系统的`presentation`动画为例，用[demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWViewControllerTransitions)中的代码示例了实现过程。

在这一篇中，将示例如何实现手势交互的`Interative Transition`的交互动画。另外我们自定义的系统转场种类是导航控制器的`navigation transition`。

`Interactive Transition`的实现和[上一篇](http://www.awsomejiang.com/2017/12/25/View-Controller-Transistion-customizing-presentation-md/)中非交互性的转场动画的实现基本步骤是一致的。除此之外，需要我们告知系统交互动画在手势的作用下，已经完成了多少；当手势结束之后，需要根据目前的手势的状态来选择告知交互动画已经完成还是需要取消。

<!-- more -->

下图大致说明了这个过程的不同阶段：

<div align='center'>
<img 
src="/images/view_controller_navigation.png" 
width="800" 
title = "导航控制器转场过程不同阶段"
alt = "导航控制器转场过程不同阶段"
align = center
/>
</div>

- 若`view controller 1`的`navigation controller`要将`view controller 2`push出来；而我们想要去自定义这样的一个`push transition`，那需要成为该`navigation controller`的代理。
- 在`vc 1`转场至`vc 2`的开始时（`begin state`），导航控制器的代理方法：`- (id<UIViewControllerAnimatedTransitioning>)navigationController:animationControllerForOperation:fromViewController:toViewController:`会被调用，请求代理返回一个遵从于`UIViewControllerAnimatedTransitioning`协议的动画对象。（这和[上一篇](http://www.awsomejiang.com/2017/12/25/View-Controller-Transistion-customizing-presentation-md/)中成为`transition delegate`提供`presenation transition`需要的动画对象完全一致）。如果导航控制器的代理返回nil值，那么系统的导航转场动画将被使用。
- 除了返回动画对象来描述转场即将使用的动画以外，`navigation controller`的代理还将被询问:`- (id<UIViewControllerInteractiveTransitioning>)navigationController:interactionControllerForAnimationController:`返回一个遵从于`UIViewControllerInteractiveTransitioning`协议的交互转场对象。若在该方法中返回nil值，那么自定义的`transition`将不具有交互特性。
- 随后根据使用的交互手势和该转场动画的定义来更新转场完成的比例：`- (void)updateInteractiveTransition:`；在手势结束或者取消的时候，根据动画的定义，相应的判断该交互转场是结束还是取消了：`- (void)cancelInteractiveTransition || - (void)finishInteractiveTransition`。

[示例项目](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWViewControllerTransitions)所使用的动画效果比较简单。下图是该效果的展示：

<div align='center'>
<img 
src="/images/view_controller_navigation_480.gif" 
width="275" 
height="480" 
title = "示例效果"
alt = "示例效果"
align = center
/>
</div>

# 具体实现

## 设置代理

上面的示例效果展示了从一个`JWNTViewController`的实例跳转到一个`JWNTSecondViewController`的实例的过程。我们新建了`JWNTNavigationDelegate`类来处理`UINavigationController`的代理方法。在`JWNTViewController`中添加一个属性作为当前导航控制的代理：

```objc
#pragma mark - Lazy Loading
- (JWNTNavigationDelegate *)localNaviDelegate {
    if (!_localNaviDelegate) {
        _localNaviDelegate = [[JWNTNavigationDelegate alloc] init];
    }
    return _localNaviDelegate;
}
```

在调用系统的`- (void)pushViewController:animated:`方法之前设置好该代理：

```objc
self.navigationController.delegate = self.localNaviDelegate;
[self.navigationController pushViewController:secondVC animated:YES];
```

在实际设计自定义的导航转场动画过程中，还应该考虑到导航控制器之前是否有代理存在；如果存在，那么需要暂存带代理，在调用完`- (void)pushViewController:animated:`之后将初始的代理设置回去。

## 返回动画对象

当开始执行转场之后，也就是调用完`- (void)pushViewController:animated:`之后；`UIKit`将询问代理返回动画对象 - 在示例中，返回一个`JWNTNavigationSlideTransition`的实例：

```objc
- (id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController
                                  animationControllerForOperation:(UINavigationControllerOperation)operation
                                               fromViewController:(UIViewController *)fromVC
                                                 toViewController:(UIViewController *)toVC {
    //slide slideTransition animator
    JWNTNavigationSlideTransition *slideTransitionAnimator = self.slideTransition;
    slideTransitionAnimator.naviOperation = operation;
    return slideTransitionAnimator;
}
```
同时在需要手势交互的时候（也就是pop操作时，屏幕左边的`UIScreenEdgePanGestureRecognizer`能够被成功识别时）也返回一个交互动画对象：

```objc
- (id<UIViewControllerInteractiveTransitioning>)navigationController:(UINavigationController *)navigationController
                         interactionControllerForAnimationController:(id<UIViewControllerAnimatedTransitioning>)animationController {
    if ([animationController isKindOfClass:[JWNTNavigationSlideTransition class]]) {
        return [(JWNTNavigationSlideTransition *)animationController isInteractive] ?
        (JWNTNavigationSlideTransition *)animationController : nil;
    }
    return nil;
}
```

在该示例中，我们返回的也是这个动画对象。能够这样做是因为动画对象类`JWNTNavigationSlideTransition`是`UIPercentDrivenInteractiveTransition`的子类，`UIPercentDrivenInteractiveTransition`是遵从于`UIViewControllerInteractiveTransitioning`（视图控制器交互转场）协议的`concrete class`；也就是`JWNTNavigationSlideTransition`继承于`UIPercentDrivenInteractiveTransition`的时候本身遵从于`UIViewControllerInteractiveTransitioning`协议了，只需要让它遵从于`UIViewControllerAnimatedTransitioning`协议，实现必要方法，就能既用它来作为动画控制器（animation controller）也可以用来做交互控制器（interaction controller）。

[上一篇](http://www.awsomejiang.com/2017/12/25/View-Controller-Transistion-customizing-presentation-md/)中，我们已经介绍过，实现`UIViewControllerAnimatedTransitioning`必要的两个方法：

```objc
#pragma mark - UIViewControllerAnimatedTransitioning
- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext {
    return kNavigatioinSlideDuration;
}

- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext {
    if (self.naviOperation == UINavigationControllerOperationNone) {
        return;
    }

    BOOL isPushOperation = (self.naviOperation == UINavigationControllerOperationPush);
    UIView *fromView = [transitionContext viewForKey:UITransitionContextFromViewKey];
    UIView *toView = [transitionContext viewForKey:UITransitionContextToViewKey];
    UIView *containerView = [transitionContext containerView];

    CGRect containerViewBounds = containerView.bounds;
    CGRect leftRect = (CGRect){-CGRectGetWidth(containerViewBounds), 0,
        CGRectGetWidth(containerViewBounds),
        CGRectGetHeight(containerViewBounds)};
    CGRect rightRect = (CGRect){CGRectGetWidth(containerViewBounds), 0,
        CGRectGetWidth(containerViewBounds),
        CGRectGetHeight(containerViewBounds)};
    fromView.frame = containerView.bounds;
    toView.frame = (isPushOperation) ? rightRect : leftRect;

    [containerView addSubview:fromView];
    [containerView addSubview:toView];
    if (isPushOperation) {
        self.transitionContext = transitionContext;
        UIScreenEdgePanGestureRecognizer *edgePanGest =
        [[UIScreenEdgePanGestureRecognizer alloc]
         initWithTarget:self
         action:@selector(didPanContainerViewEdge:)];
        edgePanGest.maximumNumberOfTouches = 1;
        edgePanGest.edges = UIRectEdgeLeft;
        [toView addGestureRecognizer:edgePanGest];
        CGFloat containerHalfWidth = 0.5 * CGRectGetWidth(containerViewBounds);
        self.panHorizontalTreshhold = (fpclassify(containerHalfWidth) == FP_ZERO) ?
        0.5 * [UIScreen mainScreen].bounds.size.width : containerHalfWidth;
    }

    //animation
    [UIView animateWithDuration:kNavigatioinSlideDuration
                     animations:
     ^{
         fromView.frame = (isPushOperation) ? leftRect : rightRect;
         toView.frame = containerViewBounds;
     }
                     completion:
     ^(BOOL finished) {
         BOOL transitionCancelled = transitionContext.transitionWasCancelled;
         [transitionContext completeTransition:!(transitionCancelled)];
     }];
}
```

它们整体描述了动画的过程。需要注意的是，在`- (void)animateTransition:`方法中，如果是`push`操作，则在`toView`上添加了一个屏幕边缘`pan`的手势。而该手势用来更新使用屏幕边缘触碰`pop`的时候动画的完成度：

```objc
#pragma mark - Interactive Transitioning
- (void)didPanContainerViewEdge:(UIScreenEdgePanGestureRecognizer *)contaierEdgePan {
    CGPoint transition = [contaierEdgePan translationInView:contaierEdgePan.view];
    CGFloat horizontalCompletionPercent =
    MIN(1.0, MAX(0, fabs(transition.x)/ self.panHorizontalTreshhold));
    switch (contaierEdgePan.state) {
        case UIGestureRecognizerStateBegan: {
            self.interactive = YES;
            [contaierEdgePan setTranslation:CGPointZero
                                     inView:contaierEdgePan.view];
            UIViewController *toVC =
            [self.transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
            [toVC.navigationController popViewControllerAnimated:YES];
            break;
        }
        case UIGestureRecognizerStateChanged: {
            [self updateInteractiveTransition:horizontalCompletionPercent];
            break;
        }
        case UIGestureRecognizerStateFailed:
        case UIGestureRecognizerStateCancelled:
        default: {
            self.interactive = NO;
            if (horizontalCompletionPercent >= kNavigationPopThreshhold) {
                [self finishInteractiveTransition];
            } else {
                [self cancelInteractiveTransition];
            }
            break;
        }
    }
}
```

对于，刚接触iOS动画的开发者来讲，可能有一个疑问：`- (void)animateTransition:`这个方法是在这个过程中只调用一次的。在不断更新动画的完成度时：`- (void)updateInteractiveTransition:`，`UIKit`是如果做到让动画停留在对应的某一帧；然后根据手势`pan`的幅度，又去到其他对应的帧的？

这个问题需要了解基本的动画原理：当我们在`- (void)animateTransition:`方法中使用`block-based`的动画接口：

```objc
//animation
[UIView animateWithDuration:kNavigatioinSlideDuration
                 animations:
 ^{
     fromView.frame = (isPushOperation) ? leftRect : rightRect;
     toView.frame = containerViewBounds;
 }
                 completion:
 ^(BOOL finished) {
     BOOL transitionCancelled = transitionContext.transitionWasCancelled;
     [transitionContext completeTransition:!(transitionCancelled)];
 }];
```
改变了视图的可动画属性`frame`的值的时候，这个时候该视图`layer`的`model layer`的值已经被改变，但是`presentation layer`的值是没有改变的；如果你在这个时候打印这两个`layer`中`posititon`的值，就能立马发现区别。动画的过程就是根据动画的其他参数`timing function (时间函数)、是否additive`等差值出`presentation layer`值渐变到`model layer`值的过程。如果是这样`UIKit`自然同样能够差值出任意时刻的`presentation layer`被改变的可动画属性的具体值是什么，从而绘在屏幕上绘制出相应的状态。可以参考[Core Animation Essentials](https://developer.apple.com/videos/play/wwdc2011/421/)。


# 参考资料

## 转场动画部分
- [Custom Transitions Using View Controllers](https://developer.apple.com/videos/play/wwdc2013/218/)
- [View Controller Programming Guide for iOS](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/PresentingaViewController.html#//apple_ref/doc/uid/TP40007457-CH14-SW1)
- [Advances in UIKit Animations and Transitions](https://developer.apple.com/videos/play/wwdc2016/216/)
    - [sample code](https://developer.apple.com/library/content/samplecode/PhotoTransitioning/Introduction/Intro.html) 
    - [sample code](https://github.com/DenHeadless/ZoomInteractiveTransition)
- [View Controller Advancements in iOS 8](https://developer.apple.com/videos/play/wwdc2014/214/)

## UIKit动画原理部分
- [Core Animation Essentials](https://developer.apple.com/videos/play/wwdc2011/421/)
- [Best Practices for Cocoa Animation](https://developer.apple.com/videos/play/wwdc2013/213/)

