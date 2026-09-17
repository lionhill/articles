# SwiftUI 导航动画排查：一个约 1pt 的 `UINavigationBar` 尾部移动是怎么定位的

## 从一个很小的视觉问题，追到 Navigation Lifecycle 和 Presentation Layer

有些 UI Bug 很小，但很值得追到底。

这次遇到的问题只有大约 1pt。

在 iPhone 的 compact navigation 环境下，SwiftUI `NavigationStack` 执行 push 后，目标页面主体看起来已经停止运动，但系统原生的 inline navigation title 和 Back button 还会继续向左移动一点点，持续几个 frame 才最终停下。

最终位置其实是正确的。

问题出在“它是怎么到达最终位置的”。

一开始，这很像 SwiftUI 的 Layout 或 Safe Area 问题。但最后发现，真正值得关注的是底层 `UINavigationBar` 的生命周期：root 页面隐藏了 navigation bar，而 push 后的 destination 又让它重新显示。

整个排查过程最终涉及三个层次：

```text
SwiftUI Navigation State
        ↓
UIKit UINavigationBar Lifecycle
        ↓
Core Animation Presentation State
```

真正有价值的不是最后改了几行代码，而是怎样一步一步把这三个层次分开。

---

## 一、问题是什么

正常情况下，一次 push 应该类似：

```text
开始 Push
    ↓
目标页面进入
    ↓
Navigation Title / Back Button 同步进入
    ↓
全部一起停止
```

实际看到的却是：

```text
开始 Push
    ↓
目标页面进入
    ↓
页面主体已经停止
    ↓
Title + Back Button 继续向左移动一点
    ↓
Navigation Bar 最终停止
```

这个额外移动很小，大约在 1pt 左右。

在测试环境中，可以在以下条件下复现：

- iOS 26.5 Simulator
- iOS 27 Simulator
- 实体 iPhone

而在 regular-width iPad navigation 中没有观察到相同现象。

这个差异后来成为一个重要线索。

---

## 二、为什么它一开始非常像 Layout Bug

一个控件在动画最后阶段又“补走”一点，很容易让人首先怀疑布局。

例如：

- Safe Area 又算了一次；
- SwiftUI 发生了第二次 Layout；
- Inline Title 宽度发生变化；
- Back Button 宽度重新计算；
- Toolbar Item identity 变化；
- `NavigationStack` 转场时序有问题；
- `UIHostingController` frame 改变；
- 外层 Container Geometry 发生变化。

这些怀疑其实都合理。

但这里真正应该先回答的问题不是：

> 最终位置为什么算错了？

因为最终位置本身没有错。

更准确的问题应该是：

> **现在看到的移动，到底来自 Layout 变化，还是来自一个还没有结束的 Animation？**

这两个方向完全不同。

---

## 三、Model Layer 和 Presentation Layer

UIKit 最终通过 Core Animation Layer 显示内容。

而在动画执行期间，有两个状态特别值得区分。

### Model Layer

Model Layer 表示程序已经设定好的最终状态。

例如：

```swift
view.layer.position
```

以及 UIView 的最终 frame。

### Presentation Layer

Presentation Layer 表示动画当前这一刻，屏幕上真正显示的位置。

概念上可以通过：

```swift
view.layer.presentation()?.position
```

观察。

因此，如果：

```text
Model Geometry 已经稳定
```

但是：

```text
Presentation Geometry 仍然变化
```

那就意味着：

**不是 Layout 又执行了一次，而是 Core Animation 还没有结束。**

这成为整个问题的第一个真正突破点。

---

## 四、Instrumentation 得到了什么结果

在问题发生的最后几个 frame 中，Navigation Title 的 Model X 坐标一直保持不变。

例如：

```text
Model X:

143.67
143.67
143.67
143.67
143.67
```

但是 Presentation X 还在继续向最终位置收敛：

```text
Presentation X:

144.81
144.51
144.11
143.99
143.91
143.67
```

Back control 也出现了类似行为。

与此同时，下列数据都保持稳定：

- `UIHostingController` geometry；
- `UINavigationController` geometry；
- `UINavigationBar` model geometry；
- safe-area insets；
- navigation title model frame；
- Back control width。

没有发现第二次 Layout。

没有发现 Safe Area 又发生变化。

也没有发现 Back button 宽度变化。

这时问题的性质已经明显发生变化。

它不再像：

```text
SwiftUI 又重新排版了一次
```

而更像：

```text
页面主体已经完成转场，
但 UINavigationBar 自己的 Presentation Animation
还没有执行完。
```

---

## 五、为什么没有直接用 offset 修掉

一个 1pt 的问题，很容易让人写出一个 1pt 的修复：

```swift
.offset(x: -1)
```

或者干脆自己实现一套 Navigation Title。

但这里这样做并不合理。

因为我们已经确认：

> **最终 Model Geometry 本来就是正确的。**

如果为了补偿一段暂时存在的 Presentation Animation，而故意去修改正确的 Model Geometry，本质上是在制造第二个问题。

因此这里有一个很重要的原则：

> **如果 UI 最终停在正确的位置，但停止过程看起来不对，应该先检查 Animation State，而不是先修改 Layout State。**

---

## 六、一些合理的尝试为什么没有解决问题

在找到真正触发条件以前，也测试过很多常见方向，例如：

- 固定 root navigation title；
- 调整 toolbar visibility 的动画行为；
- 固定 Toolbar Item identity；
- 使用 principal title；
- 使用自定义 title view；
- 调整 safe-area ownership；
- 修改 navigation transition。

这些测试并不是没有价值。

它们帮助逐步排除了大量 SwiftUI 层面的可能性。

但那个最后约 1pt 的横向尾动仍然存在。

于是问题开始转向更底层的结构：

> Push 开始时，Navigation Controller 本身到底发生了什么变化？

---

## 七、最关键的结构差异

检查 compact navigation 的结构之后，发现 root 页面会隐藏系统原生 navigation bar。

概念上类似：

```swift
.toolbar(.hidden, for: .navigationBar)
```

而 push 到 destination 后，又让 navigation bar 显示：

```swift
.toolbar(.visible, for: .navigationBar)
```

这意味着一次看起来普通的 push，其实同时发生了两种状态变化。

第一种：

```text
Root → Destination
```

第二种：

```text
Navigation Bar Hidden → Visible
```

也就是说：

**Navigation Push Transition 和 Navigation Bar Visibility Transition 同时发生。**

这成为最值得单独验证的变量。

---

## 八、隔离实验一：让 Navigation Bar 永远不要真正隐藏

接下来做了一个非常有意的诊断实验。

这个实验不追求代码漂亮，也不追求 UI 可以正式发布。

只改变一个条件：

```text
禁止 iPhone compact navigation 中的
原生 UINavigationBar 真正进入 hidden 状态
```

Root 页面看起来是否完美并不重要。

只需要回答一个问题：

> **如果 Navigation Bar 从来没有被结构性隐藏，那个尾部横向移动还存在吗？**

结果非常明确：

**消失了。**

逐帧观察显示：

- Destination content；
- Navigation Title；
- Back Button；

现在是一起完成 push 的。

不再出现：

```text
页面主体已经停下
↓
Navigation Bar 自己还继续移动
```

这个实验第一次建立了比较强的因果关系。

问题不只是“和 `UINavigationBar` 有关”。

而是和 push 过程中：

```text
hidden → visible
```

这段生命周期变化有关。

---

## 九、隔离实验二：不用 UIKit Hack，只用正常 SwiftUI

第一个实验只是为了验证假设。

显然，不应该把 UIKit 层面的诊断性拦截带进生产代码。

下一步需要回答：

> 能不能既保持 Navigation Bar 的结构生命周期稳定，又让 Root 页面视觉上看不到它？

答案是可以。

思路从：

```text
把 Navigation Bar 本身隐藏
```

变成：

```text
Navigation Bar 一直存在，
只隐藏它的视觉内容
```

一个简化后的 SwiftUI 结构类似：

```swift
NavigationStack {
    RootView()
        .navigationTitle("")
        .navigationBarTitleDisplayMode(.inline)
        .toolbar(.visible, for: .navigationBar)
        .toolbarBackground(.hidden, for: .navigationBar)
}
```

这里真正重要的不是几个 modifier 本身。

而是 Navigation Bar 生命周期发生了变化。

### 修改前

```text
Root
Navigation Bar structurally hidden

Push

Destination
Navigation Bar becomes visible
```

### 修改后

```text
Root
Navigation Bar 一直存在
只是 Title / Background 不显示

Push

Destination
继续复用同一个 Navigation Bar
```

于是 push 时不再同时发生：

```text
hidden → visible
```

这种结构变化。

---

## 十、结果

正常 SwiftUI 版本复现了诊断实验的结果。

测试观察到：

- 原来的 navbar-only 横向尾动消失；
- 页面主体和 navigation chrome 同时结束动画；
- 系统原生 Back button 行为正常；
- Interactive Back 行为正常；
- Root 页面 geometry 基本保持不变；
- 不需要 negative padding；
- 不需要 offset；
- 不需要自定义 Back Button；
- 不需要人为修改 animation timing。

这个结果同时在 Simulator 和实体 iPhone 上得到验证。

---

## 十一、真正值得记住的是 Appearance 和 Structural Visibility 的区别

这次问题里，我认为最值得留下的是下面这个区别。

这两句话并不完全等价：

> 我不希望这个页面上“看见” Navigation Bar。

和：

> 我希望 Navigation Bar 在这个页面上“不参与 Navigation Hierarchy”。

在 SwiftUI 代码中：

```swift
.toolbar(.hidden, for: .navigationBar)
```

看起来很像一个单纯的视觉设置。

但底层 UIKit 行为可能会影响：

```text
UINavigationBar 是否参与当前状态
```

以及：

```text
它是否参与下一次 transition
```

当一个系统组件在 transition 的一端结构性不存在，而在另一端重新参与动画时，它的动画轨迹未必和页面主体完全相同。

---

## 十二、一个很实用的三层排查模型

这个案例最后可以抽象成一个比较通用的 UI 动画排查框架。

### 第一层：Layout State

先问：

- Frame 有没有变？
- Safe Area 有没有变？
- Intrinsic Size 有没有变？
- 有没有第二次 Layout Pass？

常见观察点包括：

```text
UIView.frame
safeAreaInsets
layoutSubviews
viewDidLayoutSubviews
GeometryReader
```

---

### 第二层：Hierarchy / Lifecycle State

再问：

- 这个系统 View 在 transition 两端都存在吗？
- Navigation Bar 是 hidden 还是 visible？
- Toolbar Item 是被隐藏，还是被移除？
- Hosting Controller 有没有变化？
- 同一个原生 View 是否持续参与整个 Transition？

这一层在 SwiftUI 中特别容易被忽略。

因为 declarative API 隐藏了很多 UIKit hierarchy 的细节。

---

### 第三层：Presentation State

最后问：

- Model Layer 是否已经到达最终位置？
- `CALayer.presentation()` 是否还在变化？
- Core Animation 是否仍然在完成一个尚未结束的动画？

本案例最终可以概括成：

```text
Layout State:
Stable

Navigation-Bar Lifecycle:
Hidden → Visible

Presentation State:
Still Animating
```

当这三个状态被区分开以后，真正的问题就容易定位得多。

---

## 十三、为什么“单变量实验”特别重要

整个排查过程中，最有价值的一步其实不是最漂亮的一步。

反而是那个很粗暴的诊断实验：

```text
Navigation Bar 能不能永远不要被隐藏？
```

它一次只改变了一个变量。

没有同时改 Layout。

没有同时改 Title。

没有同时改 Safe Area。

没有同时改 Animation Duration。

结果一旦问题消失，搜索范围就从：

```text
SwiftUI
Layout
Safe Area
Title Measurement
Back Button
Toolbar
Hosting Controller
UIKit
Core Animation
```

迅速缩小到：

```text
Navigation-Bar Lifecycle
```

之后再把结论转换成正式 SwiftUI 架构，就简单很多。

一个比较有效的排障流程可以是：

```text
Observe
观察现象

↓

Instrument
测量状态

↓

Hypothesis
形成单一假设

↓

Isolation Experiment
只改变一个变量

↓

Causality
建立因果关系

↓

Production Design
设计正式方案
```

这通常比连续尝试多个“看起来可能有效”的 UI tweak 更高效。

---

## 十四、对 SwiftUI Navigation 的一个实际建议

如果一个 SwiftUI 应用的结构是：

```text
Root 页面
不希望看到系统 Navigation Chrome

↓

Push 后的页面
又希望使用原生 Navigation Bar
```

并且问题恰好发生在 Root → Destination 的 push 过程中，那么可以考虑测试：

**让原生 Navigation Bar 一直参与 Navigation Hierarchy，只改变它的视觉表现。**

例如：

```swift
NavigationStack {
    RootView()
        .navigationTitle("")
        .navigationBarTitleDisplayMode(.inline)
        .toolbar(.visible, for: .navigationBar)
        .toolbarBackground(.hidden, for: .navigationBar)
}
```

实际项目当然会根据 UI 设计做不同处理。

这里更重要的是结构原则：

> **如果同一个原生 Navigation Bar 还要参与后续 Push Transition，可以优先考虑改变它的 Appearance，而不是改变它的 Structural Visibility。**

这并不是一个普遍规则。

`.toolbar(.hidden)` 本身当然有完全合理的使用场景。

但如果一个 Animation Artifact 恰好出现在：

```text
bar-hidden root
        ↓
bar-visible destination
```

这样的结构里，那么 Navigation Bar Lifecycle 很值得尽早检查。

---

## 十五、最终总结

这个 Bug 表面上只有大约 1pt。

但它很好地展示了现代 iOS UI 开发中三个不同层次之间的区别：

```text
SwiftUI 声明状态
        ↓
UIKit View Lifecycle
        ↓
Core Animation Presentation State
```

最关键的发现不是：

> Navigation Title 最后多移动了约 1pt。

而是：

> **Model Geometry 从头到尾都是正确的。**

真正还在变化的是 Presentation Layer。

之后，通过让原生 `UINavigationBar` 在整个 Navigation Lifecycle 中持续存在，避免 push 同时触发：

```text
hidden → visible
```

最终消除了这段额外的尾部动画。

如果只留下一个结论，我会选这一句：

> **在修改一个 View 最终应该停在哪里之前，先确认它的最终 Geometry 是否真的错了。你看到的，也可能只是一个还没有结束的动画。**

---

## 测试范围说明

本文描述的是一个特定 SwiftUI Navigation 架构下观察并验证到的行为。

它并不意味着所有 `NavigationStack` 应用，或者所有：

```swift
.toolbar(.hidden, for: .navigationBar)
```

的使用方式都会产生相同问题。

更有普遍价值的是：

- 如何区分 Layout 和 Presentation Animation；
- 如何观察底层 UIKit 生命周期；
- 如何通过单变量实验建立因果关系；
- 如何避免在最终 Geometry 正确时使用位置补偿去掩盖动画问题。
