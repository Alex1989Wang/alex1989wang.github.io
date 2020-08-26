---
layout: post
title: 视图控制器转换(一) View Controller Transistion - No.1
subtitle: 自定义Presentation转换动画 (customizing presentation transition)
date: 2017-12-25 21:26:32

tags:
- iOS Programming
- Animation

categories:
- Chinese
---

自定义视图控制器的转换动画在[2013年的一次WWDC session: Custom Transitions Using View Controllers](https://developer.apple.com/videos/play/wwdc2013/218/)就已经被提出了。不论是系统的`presentation`、导航控制器的`push||pop`以及`UITabBarController`在转换控制器的时候，都可以自定义转换动画。这个好处是，Apple只是向开发者开放了自定义控制器转换时候的动画的接口，而不会破坏转换之后的`控制器层级关系`；因而，开发者可以只是关注如何实现一个自定义的动画。

计划用三篇博文来分别示例介绍如何自定义系统的`presentation`、导航控制器的`push||pop`以及`UITabBarController`的点击转换动画。实际上，Apple给的自定义流程对于这三种都是大同小异。从`transition`的开始到`transition`的结束，是一次`view controller heirachy(视图控制器层级)`和`view heirachy(视图层级)`从稳定的初始状态变化到`动画中间态`,然后再次回到另外的一种稳定状态的过程。

<!-- more -->

[Custom Transitions Using View Controllers](https://developer.apple.com/videos/play/wwdc2013/218/)有详细的剖析这个过程。具体有如下图片：

<div align='center'>
<img 
src="/images/view_controller_state_transition.png" 
width="600" 
title = "视图控制器转换开始和结束状态"
alt = "视图控制器转换开始和结束状态"
align = center
/>
<br />
<br />
</div>

图中是`当前正在显示`的视图控制器A的视图层级被`Transition`出来的视图控制器B的视图层级替换的始终过程；结束时，视图控制器A的所有视图和子视图将不再出现在当前的`window`上。如果来研究下面的中间状态的话，有一个很重要的状态发生地：`containerView`，是自定义动画发生的地方。Apple的设计十分巧妙，给开发者开放一个只在`transition`过程中才存在的动画`容器视图`可以让动画和`transition`过程中的其他视图完全没有耦合；另外，`containerView`只是在`transition`过程中才存在，而当开发者在完成自己的自定义动画之后，会调用`completeTransition:`(待会儿解释这个方法)告知系统动画完成，系统会拆除`containerView`，将视图层级回归到新的`consistent`的状态。

<div align='center'>
<img 
src="/images/view_controller_transition_intermediate.png" 
width="600" 
title = "视图控制器中间状态&&containerView"
alt = "视图控制器中间状态&&containerView"
align = center
/>
<br />
</div>

# 实现步骤

以presentation为例：也就是对系统方法`- (void)presentViewController:animated:completion: NS_AVAILABLE_IOS(5_0)`的弹出动画如何做自定义。

[示例代码](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWViewControllerTransitions)

效果如下:
<div align='center'>
<img 
src="/images/view_controller_transition.gif" 
width="375" 
width="667" 
title = "效果图"
alt = "效果图"
align = center
/>
<br />
</div>

## 实现转换代理(The Transitioning Delegate)
实现转换代理(The Transitioning Delegate)。`transitioning delegate`这个名字就揭示它的作用是作为`被弹出`的视图控制器的转换代理；`被弹出`的控制器可以在`transition的不同时机`询问必要的信息来动画地展示（presentation）或者移除（dismissal）自己。这些必要信息包括：
- 动画对象（Animator objects）。动画对象是遵从于`UIViewControllerAnimatedTransitioning`协议的对象（该协议规定了转换动画必须的实现的方法，因而遵从该协议的对象被称为动画对象）。动画对象是后面示例代码的重点，它负责将视图以动画的方式去呈现`presentation`或者去掉`dismissal`。
- 交互动画对象。用来和手势结合，实现随手势进行的过程而渐变的动画，类似于导航控制器的`edge pan`来pop掉处于顶部的控制器。
- 展示控制器（Presentation controller）。`presentation controller`是用来控制`presentation style`的，也就是控制了当`被弹出`的控制器在屏幕上时是以什么style来呈现的。
	
展示控制器（Presentation controller）将不再这里做介绍；在这篇博文中，也不对交互动画对象(Intractive Animator)做给出示例，而将在下一篇中介绍它。

谁作为`被present出`的控制器的`transitioning delegate`会比较合适呢？这个问题需要根据App本身的逻辑来确定，可以将`transitioning delegate`独立出来，也可以将这个责任给`presenting view controller`(也就是调用`- (void)presentViewController:animated:completion:`方法的控制器)。在示例代码中，使用了后者机制(JWVCPViewController.m 100 - 115)。

```objc
//JWVCPViewController.m 100 - 115
- (void)collectionView:(UICollectionView *)collectionView
didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    if (indexPath.item >= 0 && indexPath.item < self.pictureNames.count) {
        self.selectedCell = [collectionView cellForItemAtIndexPath:indexPath];
        NSString *bookName = self.pictureNames[indexPath.item];
        UIStoryboard *mainStoryboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
        NSString *detailVCId = NSStringFromClass([JWVCPBookDetailViewController class]);
        JWVCPBookDetailViewController *bookDetailVC =
        [mainStoryboard instantiateViewControllerWithIdentifier:detailVCId];
        bookDetailVC.bookName = bookName;
        bookDetailVC.transitioningDelegate = self;
        [self presentViewController:bookDetailVC
                           animated:YES
                         completion:nil];
    }
}
```

如果需要某对象作为`被present出`的控制器的转换代理的话，需要改对象遵从于`UIViewControllerTransitioningDelegate`协议（这跟成为任何delegate都要遵从于对应的delegate协议一致）。`UIViewControllerTransitioningDelegate`只有为数不多的几个方法，所有都是`optional`的，也就是不实现的话，系统将会使用系统自带的`presentation`效果。

```objc
//分别为presentation和dismissal提供动画对象
- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source;
- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed;

//分别为presentation和dismissal提供交互动画对象
- (nullable id <UIViewControllerInteractiveTransitioning>)interactionControllerForPresentation:(id <UIViewControllerAnimatedTransitioning>)animator;
- (nullable id <UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:(id <UIViewControllerAnimatedTransitioning>)animator;

//提供展示控制器对象
- (nullable UIPresentationController *)presentationControllerForPresentedViewController:(UIViewController *)presented presentingViewController:(nullable UIViewController *)presenting sourceViewController:(UIViewController *)source NS_AVAILABLE_IOS(8_0);
```

在示例代码中，只是对弹出和收回做了自定义的动画`非交互动画`，因而只需要实现前面两个方法返回动画对象。
```objc
//JWVCPViewController.m 117 - 135
#pragma mark - UIViewControllerTransitioningDelegate
- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source {
    self.presentationAnimator.presenting = YES;
    CGRect cellViewRect = [self.selectedCell convertRect:self.selectedCell.bounds toView:self.view];
    self.presentationAnimator.originRect = cellViewRect;
    self.presentationAnimator.originCornerRadius = self.selectedCell.layer.cornerRadius;
    self.selectedCell.hidden = YES;
    return self.presentationAnimator;
}

- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed {
    self.presentationAnimator.presenting = NO;
    __weak typeof(self) weakSelf = self;
    self.presentationAnimator.aniCompletion = ^{
        __strong typeof(weakSelf) strSelf = weakSelf;
        strSelf.selectedCell.hidden = NO;
    };
    return self.presentationAnimator;
}
```

## 自定义Presentaion||Dismissal的系统调用顺序

- 当我们在调用系统的`- (void)presentViewController:animated:completion: - 弹出`或者`- (void)dismissViewControllerAnimated:completion: - 收起`时，系统或先调用`transitioning delegate`的对应`- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:presentingController:sourceController:`或者`- (nullable id <UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:`方法来获取动画对象。
- 调用动画对象的`transitionDuration:`来获得动画的时长。
- 执行动画的对象的`animateTransition:`方法来执行开发者自定义的动画。
- 系统等待动画对象完成动画后（或者在合适的时机）来调用系统在`animateTransition:`等方法中传入的`context transitioning object`的`completeTransition:`方法来告知动画完成。这之后就是前面提到的系统拆除`containerView`，构建新的稳定的`consistent`视图控制器和视图层级，回调`presentViewController:animated:completion:`的completion block。

```objc
- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext {
    UIView *bookDetailView = (self.isPresenting) ?
    [transitionContext viewForKey:UITransitionContextToViewKey] :
    [transitionContext viewForKey:UITransitionContextFromViewKey];
    CGRect smallOriginViewRect = self.originRect;
    UIView *containerView = transitionContext.containerView;
    CGRect toViewFrame = [transitionContext viewForKey:UITransitionContextToViewKey].frame;

    CGFloat xScale = fabs(smallOriginViewRect.size.width /
                         bookDetailView.bounds.size.width);
    CGFloat yScale = fabs(smallOriginViewRect.size.height/
                         bookDetailView.bounds.size.height);
    CGAffineTransform scaleTrans = CGAffineTransformMakeScale(xScale, yScale);
    if (self.presenting) {
        bookDetailView.center = (CGPoint){CGRectGetMidX(smallOriginViewRect),
            CGRectGetMidY(smallOriginViewRect)};
        bookDetailView.transform = scaleTrans;
        bookDetailView.layer.cornerRadius = self.originCornerRadius/xScale;
        bookDetailView.layer.masksToBounds = YES;
    }

    [containerView addSubview:bookDetailView];
    [containerView insertSubview:[transitionContext viewForKey:UITransitionContextToViewKey]
                    belowSubview:bookDetailView];
    [UIView animateWithDuration:kAnimationDuration
                          delay:0
         usingSpringWithDamping:0.5
          initialSpringVelocity:0
                        options:0
                     animations:
     ^{
         bookDetailView.center = (self.presenting) ?
         (CGPoint){CGRectGetMidX(toViewFrame), CGRectGetMidY(toViewFrame)} :
         (CGPoint){CGRectGetMidX(self.originRect), CGRectGetMidY(self.originRect)};
         bookDetailView.transform = (self.presenting) ?
         CGAffineTransformIdentity : scaleTrans;
         bookDetailView.layer.cornerRadius = (self.presenting) ? 0 :
         self.originCornerRadius/xScale;
     }
                     completion:
     ^(BOOL finished) {
         if (self.aniCompletion) {
             self.aniCompletion();
         }
         [transitionContext completeTransition:YES];
     }];
}
```

以上是动画对象中`animateTransition:`方法中的关键代码。

# 参考资料

- [Custom Transitions Using View Controllers](https://developer.apple.com/videos/play/wwdc2013/218/)
- [View Controller Programming Guide for iOS](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/PresentingaViewController.html#//apple_ref/doc/uid/TP40007457-CH14-SW1)
- [Advances in UIKit Animations and Transitions](https://developer.apple.com/videos/play/wwdc2016/216/)
    - [sample code](https://developer.apple.com/library/content/samplecode/PhotoTransitioning/Introduction/Intro.html) 
    - [sample code](https://github.com/DenHeadless/ZoomInteractiveTransition)
- [View Controller Advancements in iOS 8](https://developer.apple.com/videos/play/wwdc2014/214/)


