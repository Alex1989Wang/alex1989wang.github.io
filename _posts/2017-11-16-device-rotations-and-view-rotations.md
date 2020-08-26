---
layout: post
title: Device Rotations And View Rotations
date: 2017-11-16 10:48:19
tags:
- iOS
- View Rotations

categories:
- English
---

## App支持横屏

项目的需求需要在直播间，能够让设备支持`手动`的横屏：也就是需要点击某个`切换横屏`的按钮时候，触发横竖屏的切换。

`Apple`的文档对于这个部分比较的分散。这篇博客的初衷是为了整理这部分的官方文档，在完成这个部分之后也提供自己使用的解决方案的参考。

目前只是提供了整理的部分相关文档。会持续更新这部分内容。

<!-- more -->

## Documentation Snipets

### [`UIViewController Documentation` - Handling View Rotations](https://developer.apple.com/documentation/uikit/uiviewcontroller?language=objc)

As of iOS 8, all rotation-related methods are deprecated. Instead, rotations are treated as a change in the size of the view controller’s view and are therefore reported using the viewWillTransitionToSize:withTransitionCoordinator: method. When the interface orientation changes, UIKit calls this method on the window’s root view controller. That view controller then notifies its child view controllers, propagating the message throughout the view controller hierarchy.

In iOS 6 and iOS 7, your app supports the interface orientations defined in your app’s Info.plist file. A view controller can override the supportedInterfaceOrientations method to limit the list of supported orientations. Typically, the system calls this method only on the root view controller of the window or a view controller presented to fill the entire screen; child view controllers use the portion of the window provided for them by their parent view controller and no longer participate directly in decisions about what rotations are supported. The intersection of the app's orientation mask and the view controller's orientation mask is used to determine which orientations a view controller can be rotated into.

You can override the preferredInterfaceOrientationForPresentation for a view controller that is intended to be presented full screen in a specific orientation.
When a rotation occurs for a visible view controller, the willRotateToInterfaceOrientation:duration:, willAnimateRotationToInterfaceOrientation:duration:, and didRotateFromInterfaceOrientation: methods are called during the rotation. The viewWillLayoutSubviews method is also called after the view is resized and positioned by its parent. If a view controller is not visible when an orientation change occurs, then the rotation methods are never called. However, the viewWillLayoutSubviews method is called when the view becomes visible. Your implementation of this method can call the statusBarOrientation method to determine the device orientation.

Note
At launch time, apps should always set up their interface in a portrait orientation. After the application:didFinishLaunchingWithOptions: method returns, the app uses the view controller rotation mechanism described above to rotate the views to the appropriate orientation prior to showing the window.

### [Launching your iPhone Application in Landscape](https://developer.apple.com/library/content/technotes/tn2244/_index.html)

Allowing Your App to Rotate Into Portrait Orientation After Launch

Even if your app primarily displays its content in landscape, it may be advantageous or necessary to present certain view controllers in the portrait orientation. When UIKit detects a change in device orientation, it uses the UIApplication object and the root view controller to determine whether the new orientation is allowed. If both objects agree that the new orientation is supported, then auto rotation occurs. Otherwise the orientation change is ignored.

By default, the UIApplication object sources its supported interface orientations from the values specified for the UISupportedInterfaceOrientations key in the applications's Information Property List. You can override this behavior by implementing the application:supportedInterfaceOrientationsForWindow: method in your application's delegate. The supported orientation values returned by this method only take effect after the application has finished launching. You can therefore use this method to support a different set of orientations after launch.

Listing 3  Allowing your app to rotate into portrait after launch.

```objc
- (NSUInteger)application:(UIApplication *)application supportedInterfaceOrientationsForWindow:(UIWindow *)window {
    return UIInterfaceOrientationMaskAllButUpsideDown;
}
```
Recall that view controllers support portrait and both landscape orientations by default. Without the UIApplication object preventing rotation into portrait orientation at the app level, you will need to implement -supportedInterfaceOrientations as shown in Listing 4 for all view controllers that must only display in landscape.

Listing 4  Supporting only landscape orientations for a view controller.

```objc
- (NSUInteger)supportedInterfaceOrientations {
    return UIInterfaceOrientationMaskLandscape;
}
```
The system only queries the supported interface orientations of the app window's root view controller. You may discover that your view controller's supportedInterfaceOrientations method is not called if it is inside a container, such as a UINavigationController, which does not involve its child view controllers when deciding its supported interface orientations. In this case, the navigation controller's delegate should implement navigationControllerSupportedInterfaceOrientations: as shown in Listing 5. If your application must support iOS 6, you should instead subclass UINavigationController and override supportedInterfaceOrientations as shown in Listing 4.

Listing 5  Supporting only landscape orientations for a navigation controller.

```objc
-(NSUInteger)navigationControllerSupportedInterfaceOrientations:(UINavigationController *)navigationController {
    return UIInterfaceOrientationMaskLandscape;
}
```

## Related Methods

```objc
#if UIKIT_DEFINE_AS_PROPERTIES
@property(nonatomic, readonly) BOOL shouldAutorotate NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
@property(nonatomic, readonly) UIInterfaceOrientationMask supportedInterfaceOrientations NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
// Returns interface orientation masks.
@property(nonatomic, readonly) UIInterfaceOrientation preferredInterfaceOrientationForPresentation NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
#else
- (BOOL)shouldAutorotate NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
- (UIInterfaceOrientationMask)supportedInterfaceOrientations NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
// Returns interface orientation masks.
- (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
#endif
```
### `shouldAutorotate`
```ojbc
- (BOOL)shouldAutorotate NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
```
> Description	
> Returns a Boolean value indicating whether the view controller's contents should auto rotate.
> In iOS 5 and earlier, the default return value was NO.

### `supportedInterfaceOrientations`
```objc 
- (UIInterfaceOrientationMask)supportedInterfaceOrientations NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
```
> Description	

> Returns all of the interface orientations that the view controller supports.
> When the user changes the device orientation, the system calls this method on the root view controller or the topmost presented view controller that fills the window. If the view controller supports the new orientation, the window and view controller are rotated to the new orientation. This method is only called if the view controller's shouldAutorotate method returns YES.
> Override this method to report all of the orientations that the view controller supports. The default values for a view controller's supported interface orientations is set to UIInterfaceOrientationMaskAll for the iPad idiom and UIInterfaceOrientationMaskAllButUpsideDown for the iPhone idiom.
> The system intersects the view controller's supported orientations with the app's supported orientations (as determined by the Info.plist file or the app delegate's application:supportedInterfaceOrientationsForWindow: method) to determine whether to rotate.

> Returns

> A bit mask specifying which orientations are supported. See UIInterfaceOrientationMask for valid bit-mask values. The value returned by this method must not be 0.

### `preferredInterfaceOrientationForPresentation`
```objc
- (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation NS_AVAILABLE_IOS(6_0) __TVOS_PROHIBITED;
```
> Description	

> Returns the interface orientation to use when presenting the view controller.
> The system calls this method when presenting the view controller full screen. When your view controller supports two or more orientations but the content appears best in one of those orientations, override this method and return the preferred orientation.
> If your view controller implements this method, your view controller’s view is shown in the preferred orientation (although it can later be rotated to another supported rotation). If you do not implement this method, the system presents the view controller using the current orientation of the status bar.

> Returns	

> The interface orientation with which to present the view controller.

