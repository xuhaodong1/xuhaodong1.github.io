---
title: "iOS中的事件以及事件传递机制"
date: 2022-07-08 09:49:00
categories:
  - iOS
tags:
  - UI
---

## 前言

iOS 中有 8 种事件，本文重点介绍触摸事件的传递机制与响应流程。可以带着一下问题进行阅读：

* iOS 中有哪些事件类型，谁来进行事件的响应，怎么来响应呢？
* 触摸事件的传递与响应流程
* ```hitTest``` 方法的作用，它有什么实践场景？
* ```UIControl``` 与 ```UIGestureRecognizer``` 也能响应触摸事件，```UIResponder``` 的响应方式有什么不同？
<!-- more -->
## 响应者 & 响应者链

* 响应者即 ```UIResponder class``` 的一个实例；
* 响应者链为响应者组成的一个链式结构，不同的链式结构组合起来看起来像一个倒过来的树形结构。
* ```UIResponder``` 中包含了许多处理事件的方法，如果我们想在这个对象里响应事件，那么重写这个方法即可。

<img src="/images/blog/responderChain.png" alt="A flow diagram: On the left, a sample app contains a label (UILabel), a text field for the user to input text (UITextField), and a button (UIButton) to  press after entering text in the field. On the right, the flow diagram shows how, after the user pressed the button, the event moves through the responder chain—from UIView, to UIViewController, to UIWindow, UIApplication, and finally to UIApplicationDelegate." style="zoom:67%;" />

* **UIView**：如果 ```view``` 是 ```UIViewController``` 的 ```root view```，下一个响应者为 ```UIViewController```，否则下一个响应者为```superview```。
* **UIViewController**：如果 ```UIViewController``` 的 ```view``` 是 ```UIWindow``` 的 ```root view``` 下一个响应者对象是 ```window```；如果 当前 ```UIViewController``` 由另一个 ```UIViewController push``` 或者 ```presented```，则下一个响应者为 弹出该 ```vc``` 的 ```UIViewController```，例如 ```UINavigationController```、```UITableBarController```。
* **UIWindow**：下一个响应者为 ```UIApplication```
* **UIApplication**：下一个响应者为 ```UIApplicationDelegate```，前提是它不是 ```UIView```、```UIViewController```、以及不是 ```UIApplication``` 本身。一般来说，是指 ```AppDelegate```。

## 事件 & 谁是事件的第一响应者

| 事件类型                           | 第一响应者             |
| ---------------------------------- | ---------------------- |
| 触摸事件 touch events              | 发生触摸的视图         |
| 按压事件 press events              | 被聚焦的对象           |
| 摇动事件 shake-motion events       | 你(or UIKit)指定的对象 |
| 远程控制事件 remote-control event  | 你(or UIKit)指定的对象 |
| 编辑菜单消息 editing menu messages | 你(or UIKit)指定的对象 |
| 加速器 accelerometers              | 委任的对象             |
| 陀螺 gyroscopes                    | 委任的对象             |
| 磁力仪 magnetometer                | 委任的对象             |

在 iOS 中，有 8 种类型的事件，响应这些事件的对象被称为响应者，系统的一些常见的响应者为 ```UIView```、```UIViewController```、```UIWindow```、```UIAppllication```、```AppDelegate```，在找到最佳响应者后，如果事件没有被处理，事件会随着响应者链进行传递。不过有些事件在进行传递的时候，即使重写了响应事件的方法，特定对象不会进行响应，例如 ```shake-motion events``` 不会由 ```UIView```、```UIApplication```、```AppDelegate``` 进行响应。

* 触摸事件 ```touch events```，是 iOS 中最常见的事件，每一次触碰都会由 IOKit 通过 IPC 交给 SpringBoard，进而通过 ```mach port``` 传递给合适的进程进行响应，第一响应者是发生触碰的视图，后面会重点讲解。

```swift
open func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?)
@available(iOS 9.1, *)
open func touchesEstimatedPropertiesUpdated(_ touches: Set<UITouch>)
```

* 按压事件```press events```，表示如遥控器或者游戏手柄中进行按压触碰而产生的事件，由当前聚焦的对象进行响应。

```swift
@available(iOS 9.0, *)
open func pressesBegan(_ presses: Set<UIPress>, with event: UIPressesEvent?)
@available(iOS 9.0, *)
open func pressesChanged(_ presses: Set<UIPress>, with event: UIPressesEvent?)
@available(iOS 9.0, *)
open func pressesEnded(_ presses: Set<UIPress>, with event: UIPressesEvent?)
@available(iOS 9.0, *)
open func pressesCancelled(_ presses: Set<UIPress>, with event: UIPressesEvent?)
```

* 摇动事件 ```shake-motion events```，晃动设备进行触发。

```swift
open func motionBegan(_ motion: UIEvent.EventSubtype, with event: UIEvent?)
open func motionEnded(_ motion: UIEvent.EventSubtype, with event: UIEvent?) 
open func motionCancelled(_ motion: UIEvent.EventSubtype, with event: UIEvent?)
```

* 远程控制事件 ```remote-control event```，在音视频播放时，锁屏界面或者控制中心中点击 “上一个”、“下一个”、“暂停”和“继续”等操作时触发的事件。

```swift
@available(iOS 4.0, *)
open func remoteControlReceived(with event: UIEvent?)
```

* 编辑菜单消息 ```editing menu messages```，编辑文本出现的菜单列表产生的事件。

```swift
open func buildMenu(with builder: UIMenuBuilder)
@available(iOS 13.0, *)
open func validate(_ command: UICommand)
@available(iOS 3.0, *)
open var undoManager: UndoManager? { get }
@available(iOS 13.0, *)
open var editingInteractionConfiguration: UIEditingInteractionConfiguration { get }
```

* 加速器事件、陀螺事件、磁力仪事件不跟随响应者链，```Core Motion``` 将这些事件直接传递给指定的委任对象。

## 触摸事件流程

<img src="/images/blog/image-20220620215940554.png" alt="image-20220620215940554" style="zoom:80%;" />

当触摸事件发生时，被用户面板即硬件由电信号采集到，之后再传递给 ```IOKit.framework```，并将事件封装为 ```IOHIDEvent```；之后通过 IPC 转发给 ```SpringBoard``` 进程；再由 ```SpringBoard``` 进程再次通过 ```IPC``` 将事件传递给合适的 APP 进程；由主线程 ```RunLoop``` 进行处理，先触发 ```source1``` 回调，后触发了 ```source0``` 回调，并将事件封装为 ```UIEvent```；然后将事件加入 ```UIApplication``` 对象的事件队列中，出队后，开始寻找最佳响应者 ```hit-Testing```，找到最佳响应者后。由 ```UIApplication``` 对象 从 ```sendEvent``` 方法将事件传递给 ```window``` 对象，再由 ```window``` 对象 ```sendEvent``` 到最佳响应者，随后进行事件响应以及传递。寻找最佳响应者以及事件响应后面会重点提及，这里先简单对 IOKit.framework、SpringBoard 以及 IPC 进行简单介绍：

* IOKit.framework：它为设备驱动程序(IOKit)的用户态组件，IOKit 来源于 NeXTSTEP 的 DriverKit，IOKit.framework 提供了内核态以及用户态双向通信的接口。
* SpringBoard：iOS 中的 SpringBoard 相当于 macOS 中的 Finder，它向用户提供了熟悉的图标界面，它记录了多触摸事件、加速器事件、按压事件等。
* IPC：macOS 和 iOS 中的进程间通信(InterProcess Communication) 是基于 mach，mach 是 iOS 和 macOS 中的核心，也是有别于其他操作系统的重点，mach 采用微内核的概念，即内核仅提供一些必要的功能，其他工作由用户态实现。mach 的 IPC 是通过在两个端口之间发送消息实现，具体可以参考 《深入解析Mac OS X & iOS 操作系统》。

### 寻找最佳响应者

1. 由 ```UIApplication``` 传递给 ```UIWindow```，如果有多个 ```UIWindow``` 对象，则按倒序进行查询。
2. 对于每一个 ```UIWindow```、```UIView``` 对象来说，也是倒叙查询其子视图和本视图能否响应。

如果从遍历方式来看，是一个反过来的 ```dfs```。倒叙是因为如果有视图重叠，在上方的是后加入的对象；具体来说都是通过 ```UIView``` 的 ```hitTest``` 方法进行判断是不是最佳响应者，如果存在则返回该 ```UIView```，不存在则返回 ```nil```。

* ```hitTest(_ point: CGPoint, with event: UIEvent?)``` **模拟代码**

```swift
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard self.isUserInteractionEnabled && !self.isHidden && self.alpha > 0.01 else {
        return nil
    }
    if self.point(inside: point, with: event) {
        for subview in subviews.reversed() {
            let convertedPoint = subview.convert(point, from: self)
            let hitTestView = subview.hitTest(convertedPoint, with: event)
            if let hitTestView = hitTestView {
                return hitTestView
            }
        }
      return self
    }
    return nil
}
```

1. 需要 ```isUserInteractionEnabled``` 为 ```true```、```isHidden``` 为 ```fasle``` 且透明度 > 0.01

2. 如果命中点在视图内，尝试倒序遍历子视图，查找是否有更合适的点，若有则返回子视图的 ```hitTest()```，若无则返回本视图(```self```)。

3. 如果命中点不在视图内，则返回 ```nil```。

* ```point(inside point: CGPoint, with event: UIEvent?)``` **模拟代码**

```swift
override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
    return bounds.contains(point)
}
```

1. 判断当前 bounds 是否包含该点。

### 触摸事件的响应以及传递

找到最佳响应者后，```UIApplication``` 对象 ```sendEvent``` 到响应该视图的 ```UIWindow```，再有 ```UIWindow``` 对象 ```sendEvent``` 到最佳响应者，这一点可以通过查看调用栈帧看出：

<img src="/images/blog/image-20220704022615988.png" alt="image-20220704022615988" style="zoom:100%;" />

传递给最佳响应者后，便可以进行事件的响应了，对于触摸事件来说，调用上述提到的 5 个方法即代表响应。事件的拦截是通过 ```open func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?)``` 实现的，传递方式与规则见上文中 **响应者 & 响应者链**：

* 不重写，默认将事件交给响应者链传递
* 重写不掉用 ```super.touchesBegan(touches, with: event)```，事件由该响应者处理，不进行传递
* 重写并调用 ```super.touchesBegan(touches, with: event)```，将事件交给响应者链传递

采用 ```touchesBegan``` 等系列方法以响应算是比较底层的方式，为快速响应各种类型的触摸事件，Apple 提供了 ```UIGestureRecognizer``` 与 ```UIControl``` 这两种方式。

#### UIGestureRecognizer

UIGestureRecognizer 手势识别器是处理视图中的触摸和按压事件的最好方式，如果我们仅用触摸事件基本响应方式进行处理的话，难度较大且不现实。它是一个基类，Apple 提供了 8 种手势，同时也可以创建自定义手势。

* ```UITapGestureRecognizer```：轻点手势
* ```UIPinchGestureRecognizer```：捏合手势
* ```UIRotationGestureRecognizer```：旋转手势
* ```UISwipeGestureRecognizer```：滑动手势
* ```UIPanGestureRecognizer```：拖拽手势
* ```UIScreenEdgePanGestureRecognizer```：屏幕边缘拖拽手势
* ```UILongPressGestureRecognizer```：长按手势
* ```UIHoverGestureRecognizer```：指针悬停（macOS & iPadOS）

手势识别器分为离散型和持续性两种：

离散型手势在识别到手势后只调用一次 ```action``` 方法，其变化过程为：

* 识别成功：Possible —> Recognized

* 识别失败：Possible —> Failed

持续性手势在满足最初始识别条件后，会在手势信息变化中多次调用 action 方法，其变化过程为：

* 完整识别：Possible —> Began —> [Changed] —> Ended

* 不完整识别：Possible —> Began —> [Changed] —> Cancel

**对于 UIResponder 的触摸响应优先级来说，UIGestureRecognizer 的响应优先级会更高一点**；在 hit-Testing 过程中，就会判断当前 ```view``` 的手势识别器是否符合条件，符合条件的手势识别器对象会保存在 ```UIEvent``` 中，并在 ```sendEvent``` 时首先发送给它，如果手势识别器识别成功，则默认会取消剩余的触摸响应事件，表现为调用 ```touchesCancelled``` 方法。

三个重要的属性会改变上述过程：

* ```cancelsTouchesInView```：默认为 true，表示在识别手势成功后，是否取消剩余的触摸响应事件；
* ```delaysTouchesBegan```：默认为 false，表示是否在识别手势失败后，才将触摸事件传递给 ```hit-Tested view```；
* ```delaysTouchesEnded```：默认为 true，表示是否在识别手势失败后，才将 ```touchesEnded``` 事件发送给 ```hit-Tested view```；

手势冲突

手势默认是互斥的，但可以利用 ```UIGestureRecognizerDelegate``` 进行手势优先级处理。

#### UIControl

UIControl 是响应特定动作或意图的视觉元素的控件基类，它是 ```UIView``` 的子类，因此它也是响应者对象；```UIButton```、```UISwitch```、```UISlider``` 等都是它的子类，也可以自定义 ```UIControl```。通过  ```addTarget(_:action:for:)``` 指定响应事件和对象和方法，如果 ```target``` 为 ```nil```，则按照响应链传递该事件。

```swift
open func beginTracking(_ touch: UITouch, with event: UIEvent?) -> Bool
open func continueTracking(_ touch: UITouch, with event: UIEvent?) -> Bool
open func endTracking(_ touch: UITouch?, with event: UIEvent?) // touch is sometimes nil if cancelTracking calls through to this.
open func cancelTracking(with event: UIEvent?) // event may be nil if cancelled for non-event reasons, e.g. removed from window
```

与 ```UIResponder``` 类似，```UIControl``` 有 4 种跟踪触摸事件的方法，分别与 ```UIResponder``` 的 ```began```、```moved```、```ended```、```cancelled``` 相对应。如果查看其调用栈，可以发现在 ```UIResponder``` 方法内部调用了 ```UIControl``` 的跟踪方法。

![image-20220707113956921](/images/blog/image-20220707113956921.png)

如果在响应事件的方法打断点，查看调用栈帧，会发现 ```UIControl``` 会首先将事件通过 ```sendAction:to:forEvent:``` 发送给 ```UIApplication```，再通过 ```sendAction``` 转发给发送的对象的对象。

![image-20220707114606464](/images/blog/image-20220707114606464.png)

与 ```UIGestureRecognizer``` 相比，事件仍会优先传递到 ```UIGestureRecognizer```，这一点可以重写 ```UIGestureRecognizer``` 的 4 个响应方法验证。

如果 ```UIControl``` 是其子视图，会判断其是否为系统默认控件，系统默认控件则优先响应 ```UIControl``` 的 ```action``` 方法，如果为自定义控件，则默认优先响应 ```UIGestureRecognizer``` 的 ```action```。值得注意的是，如果将 ```UIGestureRecognizer``` 的 ```cancelsTouchesInView``` 改为``` false```(默认为 ```true```)，则发现 ```UIGestureRecognizer``` 也会进行响应，个人理解为 ```cancelsTouchesInView``` 改变了响应互斥的特性，因此本身也会响应。 

如果 ```UIControl``` 为父视图或平级视图，由于仍会优先将事件传递到 ```UIGestureRecognizer```， 则可以根据其 ```cancelsTouchesInView```、```delaysTouchesBegan```、```delaysTouchesEnded``` 判断事件能否传递到 ```UIControl```，这一点 ```UIControl``` 与 ```UIResponder``` 一致。

### 应用实践

#### 扩大响应区域

1. 重写本视图的 ```func point(inside point: CGPoint, with event: UIEvent?) -> Bool``` 

```swift
override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
    // 将响应区域扩大 30
    return self.bounds.inset(by: .init(top: -30, left: -30, bottom: -30, right: -30)).contains(point)
}
```

2. 重写父视图的 ```func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView?```

```swift
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    // 将响应区域扩大 30
    // subView 为应扩大响应区域的视图
    if subView.frame.inset(by: .init(top: -30, left: -30, bottom: -30, right: -30)).contains(point) {
        return subView
    }
    return super.hitTest(point, with: event)
}
```

#### 根据触摸实时修改 view 位置

```swift
override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
    let touch = touches.randomElement()
    let prePoint = touch?.precisePreviousLocation(in: self)
    let currPoint = touch?.location(in: self)
    if let prePoint = prePoint, let currPoint = currPoint {
        let offsetX = currPoint.x - prePoint.x
        let offsetY = currPoint.y - prePoint.y
        self.transform = self.transform.translatedBy(x: offsetX, y: offsetY)
    }
}
```

## 🔗

[Using Responders and the Responder Chain to Handle Events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events)

[iOS 触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

[Apple - UIGestureRecognizer](https://developer.apple.com/documentation/uikit/uigesturerecognizer?language=objc)

[UIKit: UIControl](http://southpeak.github.io/2015/12/13/cocoa-uikit-uicontrol/)

《深入解析Mac OS X & iOS 操作系统》
