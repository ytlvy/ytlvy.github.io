---
author: null
categories:
- IOS
date: '2015-07-13'
description: null
photos: null
tags:
- IOS
- Responder Chain
title: iOS Event Delivery/ The Responder Chain
---

## Event Delivery: The Responder Chain

当你设计应用的时候, 应用需要动态响应事件. 例如: 点击事件, (可以在显示屏上的不同对象上多次触发), 你需要决定那个对象来响应此事件, 并且需要了解这个对象是如何接收事件的.

当用户触发一个事件时, UIKit 创建一个` event object`, 此对象包含了需要处理的信息. 然后将此事件对象放入到当前 app 的事件队列中. 例如`touch`事件, 这种事件对象就是一个包含了一组` touch`的` UIEvent` 对象. `motion`事件, 此对象由你使用的` framework`和你感兴趣的` motion`类型来决定.

- Touch events. 当前 window 对象首先尝试将事件传送给 `touch` 触发的视图.这个视图被称为`hit-test` 视图. 寻找`hit-test`视图的过程被称为`hit-testing`, 将在下面描述.
- Motion and remote control events. 对于这些事件, 当前 window 对象会将 `shaking-motion` 或 `emote control event` 发送给`first responder`来响应.`first responder`也将在下面描述.

### Hit-Testing Returns the View Where a Touch Occurred
iOS 通过` hit-testing` 来寻找点击下方的视图. `Hit-Testing` 会查找任何包含该`touch`点坐标的视图对象, 规则为此坐标位于视图的` bounds`中, 如果包含, 会继续递归检测此视图的所有子视图. 在视图树中包含该点击的最下层视图, 即是` hit-test`视图. 当 iOS 决定了 ` hit-test` 视图后, 会将点击事件发送给此视图来处理.

说明如下, 假设用户点击了视图 E, iOS 查找` hit-test` 视图的逻辑如下:
1. 点击在视图 A 的范围内, 然后检测 子视图 B 和 C
2.  点击不再视图 B 范围内, 但是在视图 C 范围内, 继续检测 子视图 D 和 E
3.  点击不再视图 D 里, 但是在视图 E 中.
<!--more-->
视图 E 是最下层且包含 `touch`的视图树, 所以就是` hit-test`

![](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/hit_testing_2x.png)

`hitTest:withEvent:` 方法接受参数`CGPoint` 和 `UIEvent`, 返回` hit-test` 视图. `hitTest:withEvent:`方法首先`调用`在调用者自身使用` pointInside:withEvent:` 方法. 如果传入的参数` point ` 包含在视图的` bounds`中, 返回 YES , 然后继续递归调用所有子视图.

如果传入的参数点位置不在当前视图的范围内, 第一次调用` pointInside:withEvent:`返回 NO. 然后该点将被忽略, 然后` hitTest:withEvent:`将返回` nil`. 如意一个子视图返回 NO, 之后的视图树将被忽略, 因为该点如果不再这个子视图, 也必将不会在子视图的子视图. 这意味着*任何*超出父视图范围的子视图内的所有点, 将不能接受到`touch`事件,  因为点击的位置要同时位于父视图和子视图中. 在子视图的` clipsToBounds` 属性被设置为 NO, 可能发生.

> 即使点击之后超出了`hit-test`视图,  该点击对象还会和对应的` hit-test`视图一直绑定, 直到该点击结束.

`hit-test`视图拥有优先响应处理该点击事件的权利, 如果该视图不能处理, 则事件将向上沿着这个视图的反应链查找, 直到找到能处理为止.

下面是[stackoverflow](http://stackoverflow.com/questions/4961386/event-handling-for-ios-how-hittestwithevent-and-pointinsidewithevent-are-r)的相关举例

#### 例子1
```
+--------------------------------------------+
|A                                                      |
| +--------- +          +-----------------+   |
| | B           |          |   C                  |   |
| |              |           |  +----------+    |    |  
| +--------- +          |   |  D         |     |   |
|                            |   |+---------+   |    |
|                           +------------------+   |
+--------------------------------------------+
```

Say you put your finger inside D. Here's what will happen:

1. `hitTest:withEvent: ` is called on `A`, the top-most view of the view hierarchy.

2. `pointInside:withEvent: ` is called recursively on each view.
    2. 1 `pointInside:withEvent:`  is called on `A`, and returns `YES`
    2.2 `pointInside:withEvent: `is called on `B`, and returns `NO`
    2.3 `pointInside:withEvent: i`s called on `C`, and returns `YES`
    2.4 `pointInside:withEvent: `is called on `D`, and returns `YES`

3. On the views that returned `YES`, it will look down on the hierarchy to see the subview where the touch took place. In this case, from A, C and D, it will be D.

4. D will be the hit-test view

#### 例子2
+----------------------------+
|A +--------+                  |
|  |B    +--------------+    |
|  |       |C            X  |    |
|  |      +---------------+   |
|  |            |                    |
|  +--------+                   |
|                                    |
+----------------------------+

Assume X - user's touch. pointInside:withEvent: on B returns NO, so hitTest:withEvent: returns A. I wrote category on UIView to handle issue when you need to receive touch on top most visible view.

```
- (UIView *)overlapHitTest:(CGPoint)point withEvent:(UIEvent *)event {
    // 1
    if (!self.userInteractionEnabled || [self isHidden] || self.alpha == 0)
        return nil;

    // 2
    UIView *hitView = self;
    if (![self pointInside:point withEvent:event]) {
        if (self.clipsToBounds) return nil;
        else hitView = nil;
    }

    // 3
    for (UIView *subview in [self.subviewsreverseObjectEnumerator]) {
        CGPoint insideSubview = [self convertPoint:point toView:subview];
        UIView *sview = [subview overlapHitTest:insideSubview withEvent:event];
        if (sview) return sview;
    }

    // 4
    return hitView;
}
```

1. We should not send touch events for hidden or transparent views, or views with userInteractionEnabled set to NO;
2. If touch is inside self, self will be considered as potential result.
3. Check recursively all subviews for hit. If any, return it.
4. Else return self or nil depending on result from step 2.

Note, `[self.subviewsreverseObjectEnumerator] `needed to follow view hierarchy from top most to bottom. And check for `clipsToBounds` to ensure not to test masked subviews.

Usage:

1. Import category in your subclassed view.
2. Replace hitTest:withEvent: with this
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    return [self overlapHitTest:point withEvent:event];
}
```


### The Responder Chain Is Made Up of Responder Objects
Many types of events rely on a responder chain for event delivery. The responder chain is a series of linked responder objects. It starts with the first responder and ***ends with the application object***. If the first responder cannot handle an event, it forwards the event to the next responder in the responder chain.

A responder object is an object that can respond to and handle events. The `UIResponder` class is the base class for all responder objects, and it defines the programmatic interface not only for event handling but also for common responder behavior. Instances of the `UIApplication`, `UIViewController`, and `UIView` classes are responders, which means that all views and most key controller objects are responders. Note that ***Core Animation layers are not responders***.

The first responder is designated to receive events first. Typically, the first responder is a view object. An object becomes the first responder by doing two things:

1. Overriding the `canBecomeFirstResponder` method to return YES.
2. Receiving a `becomeFirstResponder` message. If necessary, an object can send itself this message.

> Note: Make sure that your app has established its object graph before assigning an object to be the first responder. For example, you typically call the `becomeFirstResponder` method in an override of the `viewDidAppear: `method. If you try to assign the first responder in viewWillAppear:, your object graph is not yet established, so the `becomeFirstResponder` method returns NO.

Events are not the only objects that rely on the responder chain. The responder chain is used in all of the following:

- Touch events. If the hit-test view cannot handle a touch event, the event is passed up a chain of responders that starts with the hit-test view.

- Motion events. To handle shake-motion events with UIKit, the first responder must implement either the motionBegan:withEvent: or motionEnded:withEvent: method of the UIResponder class, as described in Detecting Shake-Motion Events with UIEvent.

- Remote control events. To handle remote control events, the first responder must implement the remoteControlReceivedWithEvent: method of the UIResponder class.

- Action messages. When the user manipulates a control, such as a button or switch, and the target for the action method is nil, the message is sent through a chain of responders starting with the control view.

- Editing-menu messages. When a user taps the commands of the editing menu, iOS uses a responder chain to find an object that implements the necessary methods (such as cut:, copy:, and paste:). For more information, see Displaying and Managing the Edit Menu and the sample code project, CopyPasteTile.

- Text editing. When a user taps a text field or a text view, that view automatically becomes the first responder. By default, the virtual keyboard appears and the text field or text view becomes the focus of editing. You can display a custom input view instead of the keyboard if it’s appropriate for your app. You can also add a custom input view to any responder object. For more information, see Custom Views for Data Input.
UIKit automatically sets the text field or text view that a user taps to be the first responder; Apps must explicitly set all other first responder objects with the becomeFirstResponder method.

UIKit automatically sets the text field or text view that a user taps to be the first responder; Apps must explicitly set all other first responder objects with the becomeFirstResponder method.

### The Responder Chain Follows a Specific Delivery Path
If the initial object—either the hit-test view or the first responder—doesn’t handle an event, UIKit passes the event to the next responder in the chain. Each responder decides whether it wants to handle the event or pass it along to its own next responder by calling the nextResponder method.This process continues until a responder object either handles the event or there are no more responders.

The responder chain sequence begins when iOS detects an event and passes it to an initial object, which is typically a view. The initial view has the first opportunity to handle an event. Figure 2-2 shows two different event delivery paths for two app configurations. An app’s event delivery path depends on its specific construction, but all event delivery paths adhere to the same heuristics.
![](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/iOS_responder_chain_2x.png)

For the app on the left, the event follows this path:

1. The initial view attempts to handle the event or message. If it can’t handle the event, it passes the event to its superview, because the initial view is not the top most view in its view controller’s view hierarchy.

2. The superview attempts to handle the event. If the superview can’t handle the event, it passes the event to its superview, because it is still not the top most view in the view hierarchy.

3. The topmost view in the view controller’s view hierarchy attempts to handle the event. If the topmost view can’t handle the event, it passes the event to its view controller.

4. The view controller attempts to handle the event, and if it can’t, passes the event to the window.

5. If the window object can’t handle the event, it passes the event to the singleton app object.

6. If the app object can’t handle the event, it discards the event.


The app on the right follows a slightly different path, but all event delivery paths follow these heuristics:

1. A view passes an event up its view controller’s view hierarchy until it reaches the topmost view.

2. The topmost view passes the event to its view controller.

3. The view controller passes the event to its topmost view’s superview.
Steps 1-3 repeat until the event reaches the root view controller.

4. The root view controller passes the event to the window object.

5. The window passes the event to the app object.

> Important: If you implement a custom view to handle remote control events, action messages, shake-motion events with UIKit, or editing-menu messages, don’t forward the event or message to nextResponder directly to send it up the responder chain. Instead, invoke the superclass implementation of the current event handling method and let UIKit handle the traversal of the responder chain for you.