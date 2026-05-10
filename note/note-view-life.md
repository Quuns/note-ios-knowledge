# 视图生命周期
在 iOS 开发中，`UIViewController` 的生命周期管理着视图的加载、显示、布局和销毁过程。以下是 Objective-C 环境下完整的生命周期方法调用顺序及说明，适用于面试回答。

---

## 1. 初始化阶段

根据创建方式不同，初始化入口有所区别：

- **纯代码创建**  
  `init` → `initWithNibName:bundle:`（内部调用，nibName 为 nil）

- **通过 Storyboard 创建**  
  `initWithCoder:`（从归档恢复）  
  → `awakeFromNib`

- **通过 XIB 创建**  
  `initWithNibName:bundle:`

> 此阶段 **视图尚未加载**，`self.view` 为 nil。

---

## 2. 视图加载阶段

- **`loadView`**  
  当控制器的视图被首次访问时调用。**一般不要重写**（除非完全自定义视图）。默认实现会从 nib/storyboard 加载或创建空 UIView。

- **`viewDidLoad`**  
  视图加载完成，但**尚未添加到窗口**。适合做初始化操作（设置子视图、请求数据、添加通知等）。**只调用一次**。

---

## 3. 视图将要/已经出现

- **`viewWillAppear:`**  
  视图即将添加到窗口层级，**尚未显示**。此时视图已存在，但可能还没有进行最终布局。适合做一些进入前的准备工作（如隐藏导航栏、刷新界面）。

- **`viewWillLayoutSubviews`**  
  即将对子视图进行布局（旋转、尺寸改变等时也会触发）。可在此调整子视图 frame。

- **`viewDidLayoutSubviews`**  
  子视图布局完成后调用。适合做布局完成后的收尾工作。

- **`viewDidAppear:`**  
  视图已经完整显示在屏幕上。适合启动动画、定时器、开始数据刷新等。

---

## 4. 视图将要/已经消失

- **`viewWillDisappear:`**  
  视图即将从窗口移除（但还在视图层级中）。适合保存状态、停止动画、释放资源。

- **`viewDidDisappear:`**  
  视图已经从窗口移除。适合停止网络请求、移除观察者等清理工作。

---

## 5. 销毁阶段

- **`dealloc`**  
  控制器对象被释放时调用。移除通知、KVO、代理等，避免野指针。

> ⚠️ 注意：ARC 下通常不需要手动释放属性，但需要移除对单例、通知的监听。

---

## 6. 内存警告处理（已废弃的方法）

- **`didReceiveMemoryWarning`**  
  收到内存警告时调用。iOS 6 以后，系统不再自动调用 `viewDidUnload`（已废弃），建议在此释放非必需持有的资源（如缓存、不显示的图片等）。

---

## 完整典型顺序（首次呈现并返回后释放）

```
initWithCoder: / initWithNibName: / init
loadView
viewDidLoad
viewWillAppear
viewWillLayoutSubviews
viewDidLayoutSubviews
viewDidAppear
—— 切换/返回/关闭 ——
viewWillDisappear
viewDidDisappear
dealloc
```

---

## 常见面试追问点

### Q：`loadView` 何时会被手动调用？
一般不要直接调用，系统会在访问 `self.view` 且视图为空时自动触发。如果重写，必须自己创建 `self.view` 并赋值。

### Q：`init` 方法中能获取 `self.view.frame` 吗？
不能，此时视图尚未加载，`self.view` 为 nil。

### Q：`viewDidLayoutSubviews` 和 `viewWillLayoutSubviews` 的区别？
前者是布局完成后的钩子，后者是布局即将开始的钩子。两者在旋转、滚动（如 `UIScrollView`）时都会多次触发。

### Q：容器控制器（如 `UINavigationController`）会对生命周期造成影响吗？
基本顺序一致，但 `viewWillAppear` / `viewDidAppear` 会随着 push/pop 自动调用。TabBar 切换也会触发，但只针对当前选中的控制器。

### Q：`viewDidUnload` 为什么被废弃？
iOS 6 以后，系统不会在内存警告时自动释放控制器的视图，因为视图可以被安全地引用（比如被其他视图持有）。开发者应在 `didReceiveMemoryWarning` 中手动释放可重建的资源。

---

## 回答建议（面试口语化）

> “UIViewController 的生命周期从初始化开始，根据是否使用 storyboard 或 xib 分别调用 `initWithCoder:` 或 `initWithNibName:`。当视图第一次被访问时，系统调用 `loadView` 加载视图，然后 `viewDidLoad` 对视图做初始配置。  
> 在视图即将显示前调用 `viewWillAppear:`，随后进行子视图布局（`viewWillLayoutSubviews`、`viewDidLayoutSubviews`），最后 `viewDidAppear:` 表示视图已完全可见。  
> 消失过程对称：`viewWillDisappear:`、`viewDidDisappear:`。当控制器被释放会调用 `dealloc`。  
> 另外，收到内存警告会触发 `didReceiveMemoryWarning`，应释放非必要资源。iOS 6 以后不再有 `viewDidUnload`，因为系统不再自动释放视图。”

这样回答既完整又精炼，足以应对大多数 iOS 面试场景。