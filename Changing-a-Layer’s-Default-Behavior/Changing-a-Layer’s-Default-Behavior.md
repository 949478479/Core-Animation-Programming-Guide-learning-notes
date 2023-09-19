# 改变图层的默认行为

核心动画使用“行为对象”为图层实现了隐式动画，所谓的“行为对象”即是实现了 `CAAction` 协议的对象。所有的 `CAAnimation` 对象都实现了该协议，改变图层属性时触发的隐式动画就是依靠 `CAAnimation` 实现的。图层的可动画属性都会关联对应的 `CAAction` 对象，对于图层的自定义属性，可以另行提供 `CAAction` 对象，从而实现动画效果。

- [自定义 CAAction 对象](#Custom-Action-Objects-Adopt-the-CAAction-Protocol)
- [CAAction 对象需要安装到图层上才有效果](#Action-Objects-Must-Be-Installed-On-a-Layer-to-Have-an-Effect)
- [使用 CATransaction 临时禁用图层行为](#Disable-Actions-Temporarily-Using-the-CATransaction-Class)

<a name="Custom-Action-Objects-Adopt-the-CAAction-Protocol"></a>
## 自定义 CAAction 对象

要实现自定义的 `CAAction` 对象，需要实现 `CAAction` 协议，该协议只有一个方法 `runActionForKey(_:object:arguments:)`。在这个方法中，利用传入的可用信息来执行自定义的行为，可以在此方法中为图层添加动画，也可以执行其他任务，当然这样就没有动画效果了。

实现自己的 `CAAction` 对象时，需要明确何时应该触发相应的行为，可以通过相应的 `key` 来进行判断：

- 图层属性改变，属性可以是图层原有的属性，也可以是自己关联的属性，`key` 对应属性名称。
- 图层变为可见状态，或者添加到图层层级，`key` 为 `kCAOnOrderIn`。
- 图层变为隐藏状态，或者从图层层级移除，`key` 为 `kCAOnOrderOut`。
- 图层即将参与过渡动画，`key` 为 `kCATransition`。

<a name="Action-Objects-Must-Be-Installed-On-a-Layer-to-Have-an-Effect"></a>
## CAAction 对象需要安装到图层上才有效果

图层必须能够查找到相应的 `CAAction` 对象才能够执行它。当特定事件发生时，例如改变图层属性，图层会调用 `actionForKey(_:)` 方法来查找 `key` 对应的 `CAAction` 对象。可以通过好几种方式来提供 `CAAction` 对象，核心动画会按照如下顺序进行查找：

1. 如果图层拥有代理，并且代理实现了 `actionForLayer(_:forKey:)` 方法，图层会调用此方法。代理有三种选择：
    - 返回 `key` 对应的 `CAAction` 对象。
    - 返回 `nil`，表示代理放弃处理，搜索过程将继续向后进行。
    - 返回 `NSNull` 单例对象，搜索过程将立即结束。
2. 图层在自身的 `actions` 字典中根据 `key` 查找对应的 `CAAction` 对象。
3. 图层在自身的 `style` 字典中根据 `key` 查找包含该 `key` 的 `actions` 字典。就是说，在 `style` 字典中，`key` 对应的是一个 `actions` 字典，接着在这个 `actions` 字典中查找 `key` 对应的 `CAAction` 对象。
4. 图层调用自身的 `defaultActionForKey(_:)` 类方法。
5. 如果 `key` 对应图层原有的隐式动画，则执行隐式动画。

在以上过程中，一旦提供了 `CAAction` 对象，搜索过程就会立即结束。找到 `CAAction` 对象后，图层会调用 `CAAction` 对象的 `runActionForKey(_:object:arguments:)` 方法来执行相应的行为。如果自定义的 `CAAction` 对象是 `CAAnimation` 子类对象，那么不用自己实现 `runActionForKey(_:object:arguments:)` 方法，而如果是普通对象，则需要自己实现特定行为。

如下代码演示了如何使用图层代理提供 `CAAction` 对象，当图层的 `contents` 属性改变后，提供一个 `CATransition` 动画对象，其他情况下则沿用默认实现：

```swift
override func actionForLayer(layer: CALayer, forKey event: String) -> CAAction? {
    guard event == "contents" else {
        return super.actionForLayer(layer, forKey: event)
    }

    let anim = CATransition()
    anim.duration = 1.0
    anim.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseIn)
    anim.type = kCATransitionPush
    anim.subtype = kCATransitionFromRight
    return anim
}
```

<a name="Disable-Actions-Temporarily-Using-the-CATransaction-Class"></a>
## 使用 CATransaction 临时禁用图层行为

使用 `CATransaction` 的类方法可以临时禁用图层的行为，通常用于临时禁用图层的隐式动画：

```swift
CATransaction.begin()
CATransaction.setDisableActions(true)
layer.removeFromSuperlayer() // 不会触发隐式动画
CATransaction.commit()
```
