### 1. 在 FlutterBoost 下如何管理 Flutter 页面的生命周期？原生的 Flutter 的 AppLifecycleState 事件会不一致，比如 ViewAppear 会导致 app 状态suspending 或者 paused。混合栈怎么处理？
回答：在混合栈下，页面事件基于以下自定义的事件：
```dart
enum ContainerLifeCycle {
  Init,
  Appear,
  WillDisappear,
  Disappear,
  Destroy,
  Background,
  Foreground
}
```
对于页面事件重复，请参考下面的FAQ。
### 2. 如何判断 flutter 的 widget 或者 container 是当前可见的？
回答：有个api可以判断当前页面是否可见：
```dart
bool isTopContainer = FlutterBoost.BoostContainer.of(context).onstage
```
传入你widget的context，就能判断你的widget是否是可见的
基于这个API，可以判断你的widget是否可见，从而避免接收一些重复的生命周期消息。参考这个issue:https://github.com/alibaba/flutter_boost/issues/498

### 3. 您好，我想请教一下 flutter_boost 有关的问题：ABC 三个都是 flutter 页面，从 A页面 -> B页面 -> C页面，当打开C页面时希望自动关掉B页面，当从C页面返回时直接返回A页面，可有什么方法？
回答：你只需要操作 Native 层的 UINavigationController 里的 vc 数组就可以了。就如同平时你操作普通的 UIViewController 一样。因为 FlutterBoost 对 Native 层的FlutterViewController 和 Dart 层的 flutter page 的生命周期管理是一致的，当 FlutterViewController 被销毁，其在 dart 层管理的 flutter page 也会自动被销毁。

### 4. 在 ios 中 voice over 打开，demo 在点击交互会 crash;
回答：无障碍模式下目前 Flutter Engine 有bug，已经提交 issue 和 PR 给 flutter 啦。请参考这个 issue：https://github.com/alibaba/flutter_boost/issues/488及其分析。提交给 flutter 的 PR 见这里：https://github.com/flutter/engine/pull/14155

### 5. 在 ios 模拟器下运行最新的 flutter boost 会闪退
回答：如上面第4条所说的，最新的 flutter engine 在 voice over 下有 bug，会导致c rash。因为模拟器下 flutter 默认会将 voice over 模式打开，所以其实就是辅助模式，这会触发上面的 bug：“在 ios 中 voice over 打开，demo 在点击交互会 crash”。
可参考 Engine 的代码注释：
```c++
#if TARGET_OS_SIMULATOR
  // There doesn't appear to be any way to determine whether the accessibility
  // inspector is enabled on the simulator. We conservatively always turn on the
  // accessibility bridge in the simulator, but never assistive technology.
  platformView->SetSemanticsEnabled(true);
  platformView->SetAccessibilityFeatures(flags);
```

### 6. 似乎官方已经提供了混合栈的功能，参考这里：https://flutter.dev/docs/development/add-to-app; FlutterBoost 是否有存在的必要？
回答：官方的解决方案仅仅是在 native 侧对 FlutterViewController 和 Flutterengine 进行解耦，如此可以一个 FlutterEngine 切换不同的 FlutterViewController 或者 Activity 进行渲染。但其并未解决 Native 和 Flutter 页面混合的问题，无法保证两侧的页面生命周期一致。即使是 Flutter 官方针对这个问题也是建议使用 FlutterBoost。
其差别主要有：

|*|FlutterBoost2.0	|Flutter官方方案	|其他框架|
|----|----|----|----|
|是否支持混合页面之间随意跳转	|Y	|N	|Y|
|一致的页面生命周期管理(多Flutter页面)	|Y	|N	|?|
|是否支持页面间数据传递(回传等)	|Y	|N	|N|
|是否支持测滑手势	|Y	|Y	|Y|
|是否支持跨页的hero动画	|Y	|Y	|N|
|内存等资源占用是否可控	|Y	|Y	|Y|
|是否提供一致的页面route方案	|Y	|Y	|N|
|iOS和Android能力及接口是否一致	|Y	|N	|N|
|框架是否稳定，支持Flutter1.9	|Y	|N	|?|
|是否已经支持到View级别混合	|N	|N	|N|

同时 FlutterBoost 也提供了一次性创建混合工程的命令：flutterboot。代码参考：https://github.com/alibaba-flutter/flutter-boot

### 7. 如果我需要通过 FlutterViewController 再弹出一个新的但 frame 比较小的 FlutterViewController，应该怎么实现？
回答：如果不加处理会遇到 window 大小变化的问题，但可以解决。具体可以参考这个issue：https://github.com/alibaba/flutter_boost/issues/435

### 8. Flutter ViewController 如何设置横屏
VC 设置横屏依赖于 NavigationController 或者 rootVC。可以通过一下方式来设置：
1. dart 层的 SystemChrome.setPreferredOrientations 函数并非直接设置转向，而是设置页面优先使用的转向(preferred)
2. app 的转向控制除了 info.plist 的设置外，主要受 UIWindow.rootViewController 控制。大概过程是这样的：硬件检测到转向，就会调用 UIWindow 的转向函数，然后调用其 rootViewController 的 shouldAutorotate 判断是否需要自动转，然后取 supportedInterfaceOrientations 和 info.plist 中设置的交集来判断可否转
3. 对于 UIViewController 中的转向，也只在 rootviewcontroller 中才有效

举例如下，实现步骤可以这样：
1. 重写NavigationController：
```objc
-(BOOL)shouldAutorotate
{
//    id currentViewController = self.topViewController;
//
//
//     if ([currentViewController isKindOfClass:[FlutterViewController class]])
//        return [currentViewController shouldAutorotate];

    return YES;
}

-(UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    id currentViewController = self.topViewController;
    if ([currentViewController isKindOfClass:[FlutterViewController class]]){
        NSLog(@"[XDEBUG]----fvc supported:%ld\n",[currentViewController supportedInterfaceOrientations]);
         return [currentViewController supportedInterfaceOrientations];
    }
    return UIInterfaceOrientationMaskAll;
}
```
2. 改 dart 层：因为 SystemChrome.setPreferredOrientations 的设置是全局的，但混合栈是多页面，所以在 main 函数中设置，后面在新建一个 FlutterViewController 时会被冲掉。为了解决这个问题，需要在每个 dart 页面的 build 处都加上这语句来设置每个页面能支持哪些转向类型

### 9. FlutterBoost for flutter1.12出现和surface相关的crash。可以参考这个issue：https://github.com/flutter/flutter/issues/52455
可能 flutter engine的 bug 引起
