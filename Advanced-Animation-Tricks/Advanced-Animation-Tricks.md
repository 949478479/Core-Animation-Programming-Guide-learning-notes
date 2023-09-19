# 动画技巧

- [定制动画的时间设置](#Customizing-the-Timing-of-an-Animation)
- [暂停与恢复动画](#Pausing-and-Resuming-Animations)
- [动画事务](#Explicit-Transactions-Let-You-Change-Animation-Parameters)
- [过渡动画](#Transition-Animations-Support-Changes-to-Layer-Visibility)
- [透视效果](#Adding-Perspective-to-Your-Animations)

<a name="Customizing-the-Timing-of-an-Animation"></a>
## 定制动画的时间设置

时间设置是动画的重要环节，可以利用 `CAMediaTiming` 协议中的方法和属性来指定精确的时间信息，`CAAnimation`（即动画对象的基类）和 `CALayer` 均采纳了该协议。在定制时间设置之前，需要理解图层与时间之间的关系。每个图层都拥有它自己的本地时间，并以此管理其动画时间。通常情况下，两个图层之间的本地时间非常接近，用户很难察觉其中的差异。然而，本地时间会受到父图层以及图层本身的时间设置的影响。例如，改变图层的 `speed` 属性将导致图层的动画持续时间根据一定比例发生改变。

为了方便在不同图层之间进行时间转换，`CALayer` 提供了 `convertTime(_:fromLayer:)` 和 `convertTime(_:toLayer:)` 方法，可以使用这些方法转换某个时间到图层的本地时间，或者将时间在不同图层间进行转换。使用 `CACurrentMediaTime()` 函数可以获取当前的绝对时间，通过如下方式即可转换为图层的本地时间（默认情况下这个转换后的值即是 `CACurrentMediaTime()` 的值，二者并无分别）：

```swift
let localLayerTime = layer.convertTime(CACurrentMediaTime(), fromLayer: nil)
```

获取到图层本地时间后，就可以据此设置和时间相关的动画属性或者图层属性，从而实现一些有趣的动画行为：

- 使用 `beginTime` 属性可以指定动画开始的延迟时间，通常情况下，动画会在下个运行循环开始。使用该属性可以延迟动画的开始，从而连接多个动画。延迟动画开始时间后，若需要图层在动画开始前就提前进入动画开始时的状态，只需设置动画的 `fillMode` 属性为 `kCAFillModeBackwards` 即可。如果设置图层的 `beginTime`，例如设置为 `CACurrentMediaTime() + 3.0`，图层被添加到视图层级上后会延迟 `3s` 才出现。
- 使用 `autoreverses` 属性可以在动画完成后再自动以动画形式恢复到最初状态，这意味着动画持续时间会变为设定时间的两倍。如果再配合 `repeatCount` 属性，就可以多次重复这一过程，该属性表示动画重复几个来回，因此设置为整数时，动画最终会以初始状态结束，设置为分数时，例如 `2.5`，动画最终会以结束状态结束。同样，该属性也会相应的增加动画实际的持续时间。
- 使用 `timeOffset` 属性可以控制动画的当前时间点。例如，将图层的 `speed` 设置为 `0`，再手动调整图层的 `timeOffset` 属性，使其在 `0s` 到动画时长这个区间内变化，就可手动控制动画了。

<a name="Pausing-and-Resuming-Animations"></a>
## 暂停与恢复动画

可以通过如下方式暂停图层的动画，同样可用于暂停 `UIView` 的动画，其本质依旧是核心动画：

```swift
extension CALayer {
    func pause() {
        let pausedTime = convertTime(CACurrentMediaTime(), fromLayer: nil)
        speed = 0.0
        timeOffset = pausedTime
    }

    func resume() {
        let pausedTime = timeOffset
        speed = 1.0
        timeOffset = 0.0
        beginTime = 0.0
        let timeSincePause = convertTime(CACurrentMediaTime(), fromLayer: nil) - pausedTime
        beginTime = timeSincePause
    }
}
```

<a name="Explicit-Transactions-Let-You-Change-Animation-Parameters"></a>
## 动画事务

对图层所做的改变都属于动画事务，`CATransaction` 类管理着动画的创建和组合，并在适当时间执行动画。大多数情况下，都不需要自己来创建事务，为图层添加显式或隐式动画时，核心动画会自动创建隐式事务。然而，也可以自己显式地创建事务，从而对动画进行更细微地控制。

创建事务需要使用 `CATransaction` 类，使用类方法 `begin()` 来开启一个新事务，使用类方法 `commit()` 来提交刚刚开启的事务，在这两个类方法之间都属于当前事务的范围。例如，如下代码开启并提交了一个简单的事务：

```swift
CATransaction.begin()
layer.opacity = 0.0
layer.zPosition = 200.0
CATransaction.commit()
```

使用显式事务可以对动画进行更细微地控制，例如持续时间、时间设置函数，以及其他参数。还可以指定一个闭包，在动画结束之后就会被调用。例如，如下代码通过事务将隐式动画时间设置为 `10s`（显式动画的 `duration` 的优先级更高，但如果没有设置，则会使用事务默认的时间，即 `0.25s`）：

```swift
CATransaction.begin()
CATransaction.setAnimationDuration(10.0)
// 修改图层属性。。。
CATransaction.commit()
```

事务还支持嵌套，在某个事务范围内可再次开启一个事务，注意 `begin()` 和 `commit()` 方法一定要成对调用，当最外层的 `commit()` 方法调用后，核心动画才会开始处理整个事务中涉及到的动画：

```swift
CATransaction.begin() // 外层事务开启
CATransaction.setAnimationDuration(2.0) // 外层事务的动画时间为 2.0 秒
layer.position = CGPoint()

CATransaction.begin() // 内层事务开启
CATransaction.setAnimationDuration(5.0) // 内层事务的动画时间为 5.0 秒
layer.opacity = 0.0
layer.zPosition = 200.0
CATransaction.commit() // 内层事务提交

CATransaction.commit() // 外层事务提交
```

<a name="Transition-Animations-Support-Changes-to-Layer-Visibility"></a>
## 过渡动画

`CATransition` 继承自动画基类 `CAAnimation` 而不是属性动画类 `CAPropertyAnimation`（即 `CABasicAnimation` 和 `CAKeyframeAnimation` 抽象类），因此，和属性动画不同，过渡动画无法从属性层面呈现动画，而是从图层层面呈现动画，例如添加、移除图层时执行动画。如下代码对 `UIImageView` 做动画，呈现旧图片会被新图片从左到右推走的动画效果：

```swift
let transition = CATransition()
transition.duration = 1.0
transition.type = kCATransitionPush
transition.subtype = kCATransitionFromLeft
imageView.layer.addAnimation(transition, forKey: nil)

imageView.image = UIImage(named: "newImage")
```

`iOS` 公开支持的过渡类型 `type` 有如下几种：

```swift
let kCATransitionFade: String   // 淡入淡出
let kCATransitionMoveIn: String // 新图层滑入，遮盖旧图层
let kCATransitionPush: String   // 将旧图层推走
let kCATransitionReveal: String // 旧图层滑走，揭露新图层
```

后三种类型还支持子类型 `subtype`：

```swift
let kCATransitionFromRight: String
let kCATransitionFromLeft: String
let kCATransitionFromTop: String
let kCATransitionFromBottom: String
```

除此之外还有些私有的过渡效果：

```swift
"cube"                  // 立体翻转效果（支持子类型）
"oglFlip"               // 平面翻转效果（支持子类型）
"suckEffect"            // 类似于桌布被抽走
"rippleEffect"          // 滴水效果
"pageCurl"              // 揭开书页效果（支持子类型）
"pageUnCurl"            // 盖上翻页效果（支持子类型）
"cameraIrisHollowOpen"  // 相机镜头打开效果
"cameraIrisHollowClose" // 相机镜头关上效果
```

根据文档说明，`CATransition` 还支持使用滤镜实现自定义的过渡效果，虽然从 `iOS 5` 就引入了滤镜类 `CIFilter`，但目前在 `iOS` 平台似乎依然无法使用滤镜来实现过渡动画。如果想通过滤镜来实现过渡动画，只能借助 `CADisplayLink` 来逐帧地使用滤镜生成中间图片，从而呈现动画效果。

<a name="Adding-Perspective-to-Your-Animations"></a>
## 透视效果

虽然图层支持三维效果，但是核心动画默认只呈现图层的二维效果。多个图层叠在一起时，它们之间在垂直于屏幕的方向上是有距离的概念的，通过图层的 `zPosition` 属性控制。若希望呈现图层的三维效果，可以修改图层的相应变换矩阵的值，即图层 `transfrom` 的 `m34` 的值。通常情况下，可以修改父图层的 `sublayerTransform` 属性，这样透视效果变换信息会应用于每个子图层。如下代码演示了如何修改 `m34`：

```swift
let perspective = CATransform3DIdentity
perspective.m34 = -1.0 / eyePosition
parentLayer.sublayerTransform = perspective
```

`eyePosition` 表示镜头距离，举个例子，站在一栋大厦不远处抬头观察大厦时，大厦是立体的，距离非常远观察时，大厦看起来就是平面的了。一般来说，这个值设置为几百到上千之间都是比较合适的。
