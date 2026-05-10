# iOS 事件传递链
> “事件传递是从**视图家族树的根节点（Window）**开始找。虽然我眼睛看到按钮在最上面，但在程序的**视图树**里，Window才是最开始的那个老大哥。查找的顺序是**从视图树的根节点向叶子节点**一层层问下去（从上到下），最终找到了视觉上最表面的那个按钮（叶子节点）。”
## 一、响应者链条（Responder Chain）

### 1.1 什么是响应者链条

响应者链条是由多个响应者对象（UIResponder）链接起来的链条。在 iOS 中，**只有继承自 UIResponder 的类才能接收并处理事件**，包括 UIApplication、UIViewController、UIWindow 以及所有 UIView。

事件响应分为两个阶段：
- **事件传递**（找目标）：从 `UIWindow` 开始，自下而上（从父视图到子视图）通过 Hit-Testing 找到"最佳响应者"
- **事件响应**（处理事件）：从找到的最佳响应视图开始，自下而上（从子视图到父视图），沿着响应者链条逐级向上传递，直到有对象处理该事件

**响应链条的走向**：

```
被触摸的View → 父View → ... → 根View → 视图控制器 → UIWindow → UIApplication → 丢弃
```

具体来说：
- 如果 View 有 ViewController，则先传递给 ViewController
- 如果 ViewController 也不处理，则传递给 View 的父视图
- 依次向上，经过 UIWindow、UIApplication，最终被丢弃

### 1.2 一个 View 不响应的原因

**原因一：视图本身不允许交互**（hitTest 直接返回 nil）

| 条件 | 说明 |
|---|---|
| `userInteractionEnabled = NO` | 不允许用户交互。注意 UIImageView 和 UILabel 默认值为 NO |
| `hidden = YES` | 视图被隐藏 |
| `alpha <= 0.01` | 透明度极低，近似不可见 |

**原因二：触摸点不在视图范围内**（pointInside 返回 NO）

当 touch point 在视图的 bounds 之外时，pointInside 返回 NO，hitTest 返回 nil。

**原因三：触摸点虽然在范围内，但是被子视图拦截了**

Hit-Testing 会倒序遍历子视图（从后往前，即越后添加越先遍历），如果某个子视图的 hitTest 返回了非 nil，则事件被该子视图抢走。

**原因四：没有实现 touches 系列方法或未添加手势**

即使视图被 Hit-Testing 找到，如果它没有重写 `touchesBegan:withEvent:` 等方法，系统默认会将事件沿响应链向上传递给 `nextResponder`。

**底层原理**：hitTest 方法的第一步就是检查这三个前置条件，任一不满足直接返回 nil，该视图及其所有子视图都会被跳过，相当于"一票否决"。


## 二、hitTest 和 pointInside 的关系

### 2.1 函数签名

```objc
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
```

### 2.2 hitTest 的底层默认实现

```objc
// 模拟 hitTest:withEvent: 的底层实现（完整版）
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    
    // 第一步：前置过滤——检查能否响应事件
    if (self.userInteractionEnabled == NO || 
        self.hidden == YES || 
        self.alpha <= 0.01) {
        return nil;
    }
    
    // 第二步：调用 pointInside——判断触摸点是否在自己的范围内
    if (![self pointInside:point withEvent:event]) {
        return nil;
    }
    
    // 第三步：倒序遍历所有子视图（从后往前遍历）
    // 即最后添加的子视图最先被遍历
    NSInteger count = self.subviews.count;
    for (NSInteger i = count - 1; i >= 0; i--) {
        UIView *childView = self.subviews[i];
        
        // 坐标转换：将当前视图坐标系上的点转换到子视图的坐标系
        CGPoint childPoint = [self convertPoint:point toView:childView];
        
        // 递归调用子视图的 hitTest
        UIView *fitView = [childView hitTest:childPoint withEvent:event];
        
        // 如果找到合适的子视图，立即返回，不再继续遍历
        if (fitView) {
            return fitView;
        }
    }
    
    // 第四步：所有子视图都不合适，返回自己
    return self;
}
```

### 2.3 两者的关系总结

| 对比维度 | hitTest:withEvent: | pointInside:withEvent: |
|---|---|---|
| **职责** | 寻找最终响应事件的视图，递归遍历子视图 | 判断某个点是否在当前视图的 bounds 范围内 |
| **调用关系** | 方法体内部会调用 pointInside | 被 hitTest 内部调用 |
| **返回类型** | UIView（或 nil） | BOOL（YES/NO） |
| **是否可重写** | 可自定义事件分发逻辑 | 可自定义响应范围（如扩大按钮热区） |

**关键关系**（来自 Apple 官方文档）：hitTest 通过递归调用每个子视图的 pointInside:withEvent: 方法来遍历视图层次结构，确定应该将触摸事件发送给哪个子视图。

用更直观的话说：**hitTest 是做决策的总指挥，pointInside 是提供关键信息的情报员。** hitTest 先自检，然后调用 pointInside 问"点在我身上吗？"，如果在，就去遍历子视图；子视图也重复这一流程，最终找到最合适的视图。

**重要细节**：
- `hitTest` 的遍历是**倒序**的（从 subviews 数组末尾开始），即层级上越靠近用户的视图越先被测试
- `pointInside` 的参数 `point` 是**调用者自身坐标系**上的点，所以在递归时需要先做坐标转换

### 2.4 常见的自定义扩展场景

在实际开发中，经常会通过重写这两个方法来解决一些棘手需求：

**重写 pointInside 扩大按钮点击范围**：这是最常见的场景，按钮在界面上看着很小，但希望它更容易被点中。做法就是在 pointInside 中人为扩大判断区域，比如给按钮周围增加 20pt 的热区。

**重写 hitTest 实现事件穿透**：比如蒙层上有个透明的区域需要让下层视图响应。做法是在 hitTest 中判断，如果返回的是自己并且满足特定条件（比如点在了透明区域），就返回 nil 让事件穿透下去。

*关于这些自定义扩展的具体代码实现思路，在第三部分"两个 view 叠在一起"的场景中会有更详细的展开。*


## 三、两个 view 叠在一起时的响应问题

这是面试中被高频追问的实战场景，直接考验对事件分发机制的理解深度。

### 3.1 默认行为：哪个会响应？

**规则：层级更高的视图（subviews 数组更靠后的、视觉上离用户更近的）优先响应。**

由于 hitTest 会倒序遍历子视图数组（从末尾向前），也就是说最后被添加到父视图上的那个 view 会最先被测试。如果它满足响应条件，事件就被它"抢走"了，不会再传给后面的 view。

因此，在兄弟视图之间，"后来居上"是默认规则；在父子视图之间，子视图默认优先于父视图（因为遍历时子视图先被检测）。

### 3.2 如果不想让上面的 view 响应（让上面 view 穿透）

如果有两层叠放的视图，想点击上层却让下层响应（也就是事件穿透），有以下几种方案：

**方案一：重写 hitTest 返回 nil**

```objc
// 在顶层 View 中重写，使整个视图完全"穿透"
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    UIView *hitView = [super hitTest:point withEvent:event];
    
    // 如果最佳响应者是自己，返回 nil 使事件穿透
    if (hitView == self) {
        return nil;
    }
    return hitView;
}
```

**核心逻辑**：当 hitTest 返回 nil 时，表示该视图不处理事件，事件会自动穿透到下层的兄弟视图或父视图。

**方案二：重写 pointInside 返回 NO**

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    return NO;
}
```

适用于完全不需要让该视图接收触摸事件的场景。

**方案三：关闭 userInteractionEnabled**

```objc
topView.userInteractionEnabled = NO;
```

最简单的做法，但注意这会让该视图的**所有子视图也无法响应**事件，因为 hitTest 第一步检查的就是父视图的 userInteractionEnabled 状态，父视图不响应，子视图一票否决。

### 3.3 如果两个 view 都要响应（同时响应）

默认情况下同一层级的视图**不会同时响应**同一事件，hit-Testing 找到第一个合适的视图就结束了。要实现两个 view 都收到 touch，需要借助一些取巧的手段。

**方案一：在 hitTest 中做"额外处理"（Block 方案）**

```objc
// 在顶层 view 的 hitTest 中
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    UIView *hitView = [super hitTest:point withEvent:event];
    
    // 判断是否需要让下层 view 也处理
    if (yourBlock) {
        yourBlock();  // 通知下层 view
    }
    
    return hitView; // 正常返回，不影响默认的响应链
}
```

**方案二：上层 View 收到事件后主动调用下层 View 的 touches 方法**

```objc
// 在上层 view 中
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // 自己处理
    [self handleTouch];
    
    // 手动调起下层 view 的响应方法
    [self.bottomView touchesBegan:touches withEvent:event];
}
```

**方案三：用两个独立的 UIGestureRecognizer**

给两个 view 分别添加手势识别器，两者可以同时识别同一个触摸事件（需要设置手势代理，实现 `gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:` 返回 YES）。

**方案四：事件转发给 nextResponder**

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // 自己的处理后，再转发给下一个响应者
    [self.nextResponder touchesBegan:touches withEvent:event];
}
```

这种方式不会跳过响应者链，而是沿着链条继续向上传递（注意 nextResponder 是该视图的父视图或视图控制器，不是下面的兄弟视图）。


## 四、进阶面试拓展

### 4.1 事件的完整生命周期

从手指触摸屏幕到最终响应，一个更完整的过程：

```
手指触摸屏幕
    ↓
IOKit 封装为 IOHIDEvent
    ↓
SpringBoard 接收 → 通过 Mach Port 传递给前台 App
    ↓
App RunLoop Source1 回调 → Source0 回调
    ↓
封装为 UIEvent，加入 UIApplication 事件队列
    ↓
UIApplication 取事件 → 派发给 KeyWindow
    ↓
UIWindow 调用 hitTest:withEvent:（Hit-Testing 寻找最佳响应者）
    ↓
找到 hit-test view → 调用其 touchesBegan/Moved/Ended 等
    ↓
如果该 view 不处理（未重写 touches），则沿响应链向上传递
    ↓
最终到 UIApplication 仍不处理 → 事件被丢弃
```

### 4.2 UIGestureRecognizer 在事件链中的特殊地位

手势识别器在实际开发中比 touches 方法更常用，它们在事件链中的行为也有特殊性：

**手势识别与 touches 方法的竞争**：当触摸发生时，系统会先将事件发送给手势识别器。一旦手势被识别（比如轻点手势的条件满足），系统会先调用 `touchesCancelled:withEvent:` 取消对视图的 touch 发送，然后再触发手势的 action。这就是为什么有时我们明明实现了 touches 方法却发现它们没有被调用的原因。

**UIControl 的子类有最高优先权**：UIButton、UISwitch 等 UIControl 子类会自动将触摸事件转换为特定的 action 消息，它们会阻止其父视图上的手势识别器识别触摸。比如 UIButton 添加到有 UITapGestureRecognizer 的父视图上，点击按钮只会触发按钮的 action，不会触发表面的手势。

### 4.3 一些可能追问的面试问题

**Q：穿透和扩大响应范围的根本区别在哪？**
两者都是基于 hitTest 的定制，但目的和实现完全相反：穿透是让某些区域不响应（hitTest 返回 nil），扩大响应范围是让某些原本不响应的区域能响应（重写 pointInside 将判断范围外扩）。

**Q：clipsToBounds 会影响事件响应吗？**
默认情况下 hitTest 不考虑视图的实际内容（如图片的透明区域），也不受 clipsToBounds 影响。但通过重写 pointInside 可以实现更精细的判断，例如判断像素透明度来决定是否响应。

**Q：如何找到当前响应链中某个 view 所在的 ViewController？**

```objc
// 沿响应链向上查找视图控制器
- (UIViewController *)viewController {
    UIResponder *responder = self;
    while (responder) {
        responder = [responder nextResponder];
        if ([responder isKindOfClass:[UIViewController class]]) {
            return (UIViewController *)responder;
        }
    }
    return nil;
}
```

### 4.4 面试总结——核心区别要能脱口而出

面试中除了说清原理，可以把这几个核心区别记牢，它们能体现你对机制的深层理解：

| 问题 | 核心答案 |
|---|---|
| 事件传递方向 | 上级 → 下级（父视图 → 子视图） |
| 事件响应方向 | 下级 → 上级（子视图 → 父视图） |
| hitTest 遍历顺序 | **倒序**：后添加的更靠下层的子视图先遍历 |
| pointInside 参数坐标系 | **调用者自身坐标系**，递归时需要坐标转换 |
| hitTest 返回 nil 意味着 | 当前视图不响应，事件穿透或被丢弃 |
| 视图不响应的前置条件 | userInteractionEnabled=NO / hidden=YES / alpha≤0.01 |
| 父视图关闭交互的后果 | **一票否决**：所有子视图都无法响应 |
| 同时响应多视图的本质 | iOS 机制本身不支持"同时"，需要手动桥接转发 |
| 事件穿透的本质 | 重写 hitTest 返回 nil，让事件落到下层视图 |

如果你能把这些面试题背后的核心概念——事件传递、事件响应、穿透与拦截的原理——说清楚，并且能灵活地举出实际开发中遇到的例子，基本上这类问题就稳了。