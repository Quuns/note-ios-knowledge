# iOS 开发高频场景面试题

---

## 一、内存管理场景

### 场景 1：Block 循环引用

**场景描述**：某个页面 pop 后迟迟不走 dealloc，页面每进一次出一次就多一个实例常驻内存。

**考点分析**：
- Block 会强引用捕获的外部对象
- `self` 持有 Block，Block 又持有 `self` → 循环引用
- 不仅仅是 `self`，任何被强引用的对象都可能形成"引用环"

**典型代码**：

```objc
// ❌ 循环引用 — self -> timer -> block -> self
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                              repeats:YES
                                                block:^(NSTimer *timer) {
    [self doSomething];  // self 被 block 强引用
}];

// ✅ 解环方式一：weak-strong dance
__weak typeof(self) weakSelf = self;
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                              repeats:YES
                                                block:^(NSTimer *timer) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf doSomething];
}];

// ✅ 解环方式二：不持有 timer 的一方用 weak
// 如果 block 只被 timer 持有，self 不持有 block 就没问题（但 NSTimer 的 block 方式 timer 被 RunLoop 持有，self 不持有 timer 时不会循环引用）
```

**延伸追问**：`__weak` 和 `__strong` 在 block 中为什么要配合使用（weak-strong dance）？

> 先用 `__weak` 断开强引用链。但在 block 执行期间如果 `weakSelf` 被释放，后续代码访问 `weakSelf` 会得到 nil，可能导致逻辑不完整。所以进入 block 后立即用 `__strong` 恢复强引用，保证 block 执行期间 self 不会被释放。block 执行完 `__strong` 变量销毁，引用解除。

---

### 场景 2：NSTimer 循环引用（经典陷阱）

**场景描述**：timer 加到 RunLoop 后，页面退出了 `-dealloc` 不调用，timer 也停不掉。

**分析**：

```objc
// ❌ 循环引用链：self -> timer -> self (target)
//               RunLoop -> timer (强引用)
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                              target:self
                                            selector:@selector(tick)
                                            userInfo:nil
                                             repeats:YES];

// 即使在 dealloc 中调用 invalidate 也永远不会执行 — 因为 self 永远不会 dealloc
```

**解决方式**：

```objc
// 方案 1：iOS 10+ 使用 block API + weakSelf
__weak typeof(self) weakSelf = self;
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer *t) {
    [weakSelf tick];
}];

// 方案 2：代理对象 — 创建一个中间对象持有 weak 引用指向 self
@interface WeakTimerTarget : NSObject
@property (nonatomic, weak) id target;
@property (nonatomic, assign) SEL selector;
@end

@implementation WeakTimerTarget
- (void)timerFired:(NSTimer *)timer {
    if (self.target) {
        #pragma clang diagnostic push
        #pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [self.target performSelector:self.selector withObject:timer];
        #pragma clang diagnostic pop
    } else {
        [timer invalidate];
    }
}
@end

// 方案 3：使用 GCD timer（dispatch_source），不依赖 RunLoop
```

**面试追问**：timer 的 `fireDate` 为什么在 ScrollView 滚动时不准？

> 默认 mode 是 `NSDefaultRunLoopMode`，滚动时 RunLoop 切换到 `UITrackingRunLoopMode`，timer 不会触发。需要在 `scheduledTimerWithTimeInterval` 后手动补一句 `[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes]`。

---

### 场景 3：代理用 weak 还是 strong？

**场景描述**：UIScrollView 的 delegate 用 weak，但自己写的代理属性有时候写成 strong 导致循环引用。

**考点分析**：

```objc
// ✅ 常规：delegate 用 weak
@property (nonatomic, weak) id<MyDelegate> delegate;

// 原因：A 持有 B，B.delegate = A → 如果 B 的 delegate 是 strong，
// 则 A -> B -> delegate(A) 形成循环引用
```

**例外情况**：
- `NSURLSession` 的 delegate 是 strong（因为 session 不持有 task，生命周期独立）
- `CAAnimation` 的 delegate 是 strong（animation 执行完被释放后 delegate 也需要被释放）

**面试话术**：绝大多数 delegate 用 weak，遵循"谁创建谁持有"的原则。只有少数 Apple API 因为特殊生命周期管理用了 strong。

---

### 场景 4：dealloc 中为什么要用 `__weak`？

**场景描述**：dealloc 中调用了某个方法，那个方法里又用了 self，会 crash 吗？

```objc
// ❌ 危险：dealloc 中 self 已经是"僵尸状态"（deallocating），
// 此时 isa 正在被改写，再用 self 可能 crash
- (void)dealloc {
    [self.someService unregister:self];  // 危险
}

// ✅ 安全做法
- (void)dealloc {
    __weak typeof(self) weakSelf = self;
    [self.someService unregister:weakSelf];
}
```

> 在 dealloc 中 self 处于析构过程中，虽然还没完全释放，但子类的 dealloc 已执行，对象状态不完整。用 weak 引用至少保证不会在 service 内部因为强引用 self 导致 use-after-free。

---

## 二、多线程场景

### 场景 5：并发写入导致 crash

**场景描述**：多个线程同时操作一个 NSMutableArray/NSMutableDictionary，偶现 crash，难以复现。

**考点分析**：

```objc
// ❌ 多线程并发读写
dispatch_async(queue1, ^{ [self.mutableArray addObject:obj]; });
dispatch_async(queue2, ^{ [self.mutableArray addObject:obj]; });
// → EXC_BAD_ACCESS

// ✅ 方案 1：串行队列（读也可以同步返回，保证一致性）
@property (nonatomic, strong) dispatch_queue_t barrierQueue;
_barrierQueue = dispatch_queue_create("com.xxx.barrier", DISPATCH_QUEUE_CONCURRENT);

- (void)addObject:(id)obj {
    dispatch_barrier_async(self.barrierQueue, ^{
        [self.mutableArray addObject:obj];
    });
}

- (id)objectAtIndex:(NSUInteger)index {
    __block id obj = nil;
    dispatch_sync(self.barrierQueue, ^{
        if (index < self.mutableArray.count) {
            obj = self.mutableArray[index];
        }
    });
    return obj;
}

// ✅ 方案 2：@synchronized（简单但性能差）
@synchronized(self) { [self.mutableArray addObject:obj]; }

// ✅ 方案 3：NSLock / pthread_mutex（精细控制）
[self.lock lock];
[self.mutableArray addObject:obj];
[self.lock unlock];
```

**延伸追问**：`dispatch_barrier_async` 和普通 `dispatch_async` 的区别？

> barrier 在并发队列中起到"栅栏"作用：等队列中所有已排队的任务都执行完再执行 barrier block，barrier block 执行完后才执行后续任务。它不会阻塞当前线程，只是对队列中的任务做排序。

---

### 场景 6：大量图片下载的线程控制

**场景描述**：列表有 1000 个 cell，每个 cell 都要异步下载图片，如何管理？

**分析要点**：

1. **并发数控制** — 使用 NSOperationQueue 的 `maxConcurrentOperationCount`（如 4-6）限制同时下载数，避免 GCD 全局队列瞬间开启几百个线程
2. **复用问题** — cell 离开屏幕时取消对应下载任务（NSOperation 有 cancel，配合任务队列）
3. **内存控制** — 控制下载队列最大并发数同时，限制内存缓存大小（NSCache 或 LRU 算法）
4. **优先级** — 可视区域优先下载，离屏的降低优先级（`queuePriority`）

```objc
@interface ImageDownloadManager : NSObject
@property (nonatomic, strong) NSOperationQueue *downloadQueue;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSOperation *> *operations;
@property (nonatomic, strong) NSCache *imageCache;
@end

- (void)downloadImageWithURL:(NSString *)urlString
                  completion:(void(^)(UIImage *))completion {
    // 1. 查看缓存
    UIImage *cached = [self.imageCache objectForKey:urlString];
    if (cached) { completion(cached); return; }

    // 2. 取消已有任务（如果有）
    NSOperation *existing = self.operations[urlString];
    [existing cancel];

    // 3. 创建新任务
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:urlString]];
        UIImage *image = [UIImage imageWithData:data];
        if (image) {
            [self.imageCache setObject:image forKey:urlString];
        }
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            completion(image);
        }];
    }];
    self.operations[urlString] = op;
    [self.downloadQueue addOperation:op];
}

// cell 重用回调
- (void)cancelDownload:(NSString *)urlString {
    [self.operations[urlString] cancel];
    [self.operations removeObjectForKey:urlString];
}
```

---

### 场景 7：dispatch_once 为什么线程安全？

**场景描述**：单例创建用 `dispatch_once` 而不是 `@synchronized`，两者的区别？

```objc
// ✅ dispatch_once
+ (instancetype)sharedInstance {
    static MyClass *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[MyClass alloc] init];
    });
    return instance;
}

// ❌ @synchronized 版本（每次调用都加锁，性能差很多）
+ (instancetype)sharedInstance {
    static MyClass *instance = nil;
    @synchronized(self) {
        if (!instance) {
            instance = [[MyClass alloc] init];
        }
    }
    return instance;
}
```

**面试话术**：`dispatch_once` 底层使用原子操作（`__atomic_compare_exchange`）来保证 onceToken 状态只被修改一次，不是用锁实现的。它在首次执行后就不再有任何开销，而 `@synchronized` 每次都要获取锁。

---

### 场景 8：主线程卡顿排查

**场景描述**：用户反馈滑动卡顿，如何定位和优化？

**排查步骤**：

1. **Instruments - Time Profiler**：查看哪些方法在主线程耗时最多
2. **查看调用栈**：找到耗时操作（IO、网络、复杂计算）是否在主线程执行
3. **RunLoop Observer**：监听 RunLoop 的 `kCFRunLoopBeforeSources` 和 `kCFRunLoopAfterWaiting`，计算两次回调的时间差，超过 16ms 视为卡顿帧

```objc
// RunLoop 卡顿监控原理
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(
    kCFAllocatorDefault,
    kCFRunLoopAllActivities,
    YES,
    0,
    ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case kCFRunLoopAfterWaiting:
                // 记录开始时间
                self.startTime = CACurrentMediaTime();
                break;
            case kCFRunLoopBeforeSources:
            case kCFRunLoopBeforeTimers:
                // 计算耗时
                NSTimeInterval cost = CACurrentMediaTime() - self.startTime;
                if (cost > 0.016) { /* 卡顿前兆，收集堆栈 */ }
                break;
        }
    }
);
CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
```

**常见优化**：
- 高度计算提前缓存（不要在 `heightForRowAtIndexPath` 中做复杂计算）
- 图片解码移到子线程
- 复杂视图光栅化（`shouldRasterize`）但要慎用
- 减少透明视图 / 圆角 + masksToBounds

---

## 三、UI 性能场景

### 场景 9：TableView 滑动掉帧

**场景描述**：列表复杂 cell 导致滑动 FPS 低于 60，如何优化？

**排查清单**：

1. **不要在 cellForRow 里做耗时操作**
  - 图片解码（ImageIO 下采样）
  - 日期格式化（NSDateFormatter 创建极贵，用静态变量复用）
  - 富文本计算

2. **离屏渲染检查**
  ```objc
  // ❌ 触发离屏渲染的组合：cornerRadius + masksToBounds
  cell.avatar.layer.cornerRadius = 20;
  cell.avatar.layer.masksToBounds = YES;

  // ✅ 用 Core Graphics 裁剪或 UIBezierPath + CAShapeLayer mask
  // ✅ 或直接用 image context 画圆角图片
  ```

3. **高度缓存** — 提前算好每条数据的高度，存到 model 中

4. **异步绘制** — 将文本/图片绘制放到后台线程（YYAsyncLayer / AsyncDisplayKit 的思路）

5. **预估高度** — `estimatedRowHeight` 减少 `tableView:heightForRowAtIndexPath:` 的调用量

---

### 场景 10：离屏渲染场景

**常见触发条件**：
- `cornerRadius + masksToBounds`（设置其中一个不会触发，两个同时才有）
- `shadow`（除非指定了 shadowPath）
- `mask`（layer.mask）
- `group opacity` + 多个子 layer
- `allowsEdgeAntialiasing`
- 自定义 drawRect 中使用 `CGContextDrawImage` 绘制大图

**面试建议**：每种场景都要能说出触发原因和替代方案。

---

### 场景 11：Auto Layout 性能

**场景描述**：复杂布局的 Auto Layout 导致 16ms 内算不完。

**分析**：Auto Layout 底层解线性方程组（Cassowary 算法），复杂度是 O(n²)，视图多时开销不可忽视。

**优化**：
- 复杂 cell 用 frame 手动布局
- `intrinsicContentSize` 计算缓存
- 减少约束嵌套层级
- `UIStackView` 优于手工约束链

---

### 场景 12：图片内存暴涨问题

**场景描述**：显示一张 6000×4000 的图片时内存暴涨。

**分析**：UIImage 解码后的内存占用 = 宽×高×4 (RGBA) 字节。6000×4000×4 = 96MB。但这只是解码后的占用，实际加载过程中还会有峰值。

**解决**：

```objc
// ImageIO 下采样（不把全分辨率图片加载到内存）
- (UIImage *)downsampleImageAt:(NSURL *)url toSize:(CGSize)size {
    NSDictionary *options = @{
        (__bridge id)kCGImageSourceShouldCacheImmediately: @YES,
        (__bridge id)kCGImageSourceCreateThumbnailFromImageAlways: @YES,
        (__bridge id)kCGImageSourceThumbnailMaxPixelSize: @(MAX(size.width, size.height)),
        (__bridge id)kCGImageSourceCreateThumbnailWithTransform: @YES,
    };
    CGImageSourceRef source = CGImageSourceCreateWithURL((__bridge CFURLRef)url, NULL);
    CGImageRef cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, (__bridge CFDictionaryRef)options);
    UIImage *image = [UIImage imageWithCGImage:cgImage];
    CGImageRelease(cgImage);
    CFRelease(source);
    return image;
}
```

---

## 四、Crash 分析场景

### 场景 13：unrecognized selector 排查

**场景描述**：线上 crash 日志显示 `-[XXX method]: unrecognized selector sent to instance`。

**排查思路**：

1. **对象提前释放** — 野指针，对象被释放后新对象分配在同一地址，向其发旧消息
2. **类型错误** — 服务器返回的类型和前端的 model 不匹配
3. **子类未实现协议方法** — 父类声明了 `@optional` 方法但没有实现
4. **热修复/Hook 问题** — swizzle 后方法签名不匹配

**线上防御**：

```objc
// 消息转发最后一步 — forwardInvocation: 可以做防 crash
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
    if (!sig) {
        // 返回一个通用签名防止 crash
        sig = [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return sig;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    // 吞掉 — 不 crash 但打日志
    NSLog(@"unrecognized selector: %@", NSStringFromSelector(anInvocation.selector));
}
```

---

### 场景 14：野指针 crash

**场景描述**：偶发 EXC_BAD_ACCESS，Xcode 开启 Zombie Objects 后复现。

**常见成因**：

- delegate 用 assign（MRC）/ weak 但对象提前释放后没有置 nil
- NSTimer target 释放后 RunLoop 仍然触发
- block 中访问已释放的对象

**排查工具**：
- Xcode Zombie Objects
- Address Sanitizer（推荐，能定位到具体行）
- Malloc Stack Logging

---

### 场景 15：多线程下 NSNotification crash

**场景描述**：某个页面收到通知后在非主线程更新 UI crash。

**分析**：`-[NSNotificationCenter addObserverForName:object:queue:usingBlock:]` 的 queue 参数指定了 block 执行线程。如果不指定（传 nil），block 在发通知的线程执行。

```objc
// ❌ 可能在子线程更新 UI
[[NSNotificationCenter defaultCenter] addObserverForName:NotificationName
                                                   object:nil
                                                    queue:nil
                                               usingBlock:^(NSNotification *note) {
    self.label.text = @"updated";  // 子线程 crash
}];

// ✅ 明确指定主线程
[[NSNotificationCenter defaultCenter] addObserverForName:NotificationName
                                                   object:nil
                                                    queue:[NSOperationQueue mainQueue]
                                               usingBlock:^(NSNotification *note) {
    self.label.text = @"updated";  // 安全
}];

// ✅ 或用 addObserver:selector: 方式，在 selector 内 dispatch 到主线程
```

---

## 五、架构设计场景

### 场景 16：首页接口过多导致加载慢

**场景描述**：App 首页依赖 5-6 个接口的数据，串行请求导致加载时间 = 所有接口耗时之和。

**分析**：

```objc
// ❌ 串行 — T = t1 + t2 + t3 + t4 + t5
[self requestModuleA:^{
    [self requestModuleB:^{
        [self requestModuleC:^{ ... }];
    }];
}];

// ✅ 并发 — T = max(t1, t2, t3, t4, t5)
dispatch_group_t group = dispatch_group_create();

dispatch_group_enter(group);
[self requestModuleA:^{ dispatch_group_leave(group); }];

dispatch_group_enter(group);
[self requestModuleB:^{ dispatch_group_leave(group); }];
// ... C, D, E

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    [self refreshAllModules];
});
```

**延伸**：如果有依赖关系怎么办？（如 D 依赖 A 和 B 的结果）

> 用 `dispatch_group_notify` 分批：第一波 A+B 并发，notify 回调中发起 C（分别依赖 A、B），再 notify 回调中发起 D。

---

### 场景 17：MVC 胖 Controller 如何瘦身

**场景描述**：ViewController 2000+ 行，什么都往里塞。

**拆分策略**：

1. **DataSource 分离** — TableView/CollectionView 的 DataSource 独立为一个对象
2. **网络层抽离** — ViewModel / Service 层处理数据获取和转换
3. **Helper 类** — 布局计算、日期格式化、字符串处理抽到工具类
4. **Category 拆分** — 按功能模块分 Category（如 `VC+TableView`, `VC+SearchBar`）
5. **MVVM** — Controller 只做绑定，ViewModel 处理业务逻辑

```objc
// 拆分示例
@interface HomeViewController : UIViewController
@property (nonatomic, strong) HomeDataSource *dataSource;    // 数据源管理
@property (nonatomic, strong) HomeViewModel *viewModel;       // 业务逻辑
@property (nonatomic, strong) HomeLayoutManager *layoutManager; // 布局计算
@end
```

---

### 场景 18：组件化如何实现页面跳转

**场景描述**：A 模块要跳转到 B 模块的页面，但模块之间没有代码依赖。

**方案对比**：

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| URL Router | `[Router openURL:@"myapp://user/detail?id=123"]` | 简单、通用 | 参数类型丢失、硬编码字符串 |
| Protocol + Mediator | `[Mediator userDetailVC:@{@"id": @123}]` | 类型安全、编译检查 | 需要维护协议和 Category |
| Target-Action (CTMediator) | `[Mediator performTarget:@"User" action:@"detailVC" params:@{...}]` | 解耦彻底 | Runtime 调用无编译检查 |

**面试话术**：推荐 Protocol+Mediator 方案：
- 每个模块暴露一个 Protocol 定义对外能力
- Mediator 负责协议与实现的绑定（类似依赖注入的容器）
- 跳转时通过 Mediator 获取实现，类型安全

```objc
// 协议定义
@protocol UserModuleProtocol <NSObject>
- (UIViewController *)userDetailVCWithUserID:(NSString *)userID;
@end

// 注册
[ModuleManager registerModule:@protocol(UserModuleProtocol) implClass:[UserModuleImpl class]];

// 使用
id<UserModuleProtocol> userModule = [ModuleManager moduleFor:@protocol(UserModuleProtocol)];
UIViewController *vc = [userModule userDetailVCWithUserID:@"123"];
[self.navigationController pushViewController:vc animated:YES];
```

---

## 六、网络与数据场景

### 场景 19：网络请求重复发送

**场景描述**：用户快速点击按钮，同一请求发出多次。

```objc
// ❌ 没有防重
- (IBAction)submitButtonTapped:(id)sender {
    [self submitRequest];  // 快速点 5 次发 5 次
}

// ✅ 方案 1：按钮短时间内不可用
- (IBAction)submitButtonTapped:(UIButton *)sender {
    sender.enabled = NO;
    [self submitRequestWithCompletion:^{
        sender.enabled = YES;
    }];
}

// ✅ 方案 2：检查是否有正在进行的请求
@property (nonatomic, assign) BOOL isSubmitting;
- (IBAction)submitButtonTapped:(id)sender {
    if (self.isSubmitting) return;
    self.isSubmitting = YES;
    [self submitRequestWithCompletion:^{
        self.isSubmitting = NO;
    }];
}

// ✅ 方案 3：取消上一个请求再发新的（搜索框场景）
- (void)searchBar:(UISearchBar *)searchBar textDidChange:(NSString *)searchText {
    [self.currentSearchTask cancel];
    self.currentSearchTask = [self searchWithKeyword:searchText completion:^(id result) {
        // handle result
    }];
}
```

---

### 场景 20：接口数据与 UI 不一致

**场景描述**：服务器返回的数据结构变了，客户端没更新导致展示异常。

**防御策略**：
1. 所有 JSON 解析加类型校验（`isKindOfClass:`）
2. Model 层提供默认值
3. 后台可动态下发 UI 结构（类似页面动态化），但需要降级兜底

```objc
+ (instancetype)modelWithDictionary:(NSDictionary *)dict {
    UserModel *model = [[UserModel alloc] init];
    id name = dict[@"name"];
    model.name = [name isKindOfClass:[NSString class]] ? name : @"";
    id age = dict[@"age"];
    model.age = [age isKindOfClass:[NSNumber class]] ? [age integerValue] : 0;
    return model;
}
```

---

### 场景 21：图片缓存策略设计

**场景描述**：设计一个三级缓存系统用于图片加载。

**三级缓存架构**：

```
请求图片
  ├── 1. 内存缓存 (NSCache, ~50MB, 自动清理)
  │     └── 命中 → 直接返回
  │
  ├── 2. 磁盘缓存 (Library/Caches, ~200MB, LRU 淘汰)
  │     └── 命中 → 解码 → 放入内存缓存 → 返回
  │
  └── 3. 网络下载
        └── 下载 → 解码 → 存入磁盘 + 内存 → 返回
```

```objc
@interface ImageCache : NSObject
@property (nonatomic, strong) NSCache *memoryCache;

- (UIImage *)loadImageForURL:(NSString *)url {
    UIImage *image = [self.memoryCache objectForKey:url];
    if (image) return image;

    image = [self diskImageForURL:url];
    if (image) {
        [self.memoryCache setObject:image forKey:url cost:image.size.width * image.size.height * 4];
        return image;
    }
    return nil;
}
@end
```

**NSCache vs NSDictionary**：
- NSCache 在内存紧张时自动清理（线程安全）
- NSCache 不 copy key（NSDictionary 会 copy key）
- 可设置 `countLimit` / `totalCostLimit`

---

## 七、启动优化场景

### 场景 22：App 启动耗时优化

**场景描述**：冷启动到首页完全展示耗时 > 2 秒，需要优化。

**启动阶段分析**：

```
Pre-main (dyld 阶段)
  ├── 加载动态库 (dyld load)
  ├── rebase / binding (符号重定位)
  ├── ObjC runtime 初始化 (类注册、category、+load)
  └── 静态初始化器 (C++ static initializer)

Main 阶段
  └── main() → UIApplicationMain → AppDelegate didFinishLaunching
        └── → 首页 viewDidLoad → viewDidAppear → 数据加载完成
```

**优化手段**：

| 阶段 | 优化项 |
|------|--------|
| Pre-main | 减少动态库数量（< 6 个）、合并 Objective-C 类、移除无用 +load 方法 |
| Main | 延迟非首屏 init（SDK 初始化、数据库迁移等放到首屏显示后） |
| 首屏渲染 | 本地缓存首页数据、首屏接口拆分（先加载框架，后填充数据） |

---

## 八、Runtime 场景

### 场景 23：给所有 VC 的 viewDidAppear 添加日志

**场景描述**：需要统计每个页面的访问记录，但不能修改每个 VC。

```objc
// Method Swizzling
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [UIViewController class];
        SEL originalSel = @selector(viewDidAppear:);
        SEL swizzledSel = @selector(xx_viewDidAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSel);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSel);

        BOOL didAddMethod = class_addMethod(class, originalSel,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class, swizzledSel,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)xx_viewDidAppear:(BOOL)animated {
    [self xx_viewDidAppear:animated];  // 调用 original (已交换)
    NSLog(@"[PageTrack] 进入页面: %@", NSStringFromClass([self class]));
}
```

**面试追问**：为什么要在 `+load` 而不是 `+initialize` 中做？

> `+load` 在类加载时调用（早，只调一次），`+initialize` 在第一次使用时惰性调用（晚，可能被多次触发）。swizzle 应该在最早时机完成，保证所有代码路径都生效。

**再追问**：`class_addMethod` 那段 if/else 逻辑是做什么的？

> 防止 swizzle 的 selector 实现在父类中。如果子类没实现 originalSel，`class_addMethod` 会把 swizzled 实现加到子类上，然后 `replaceMethod` 把 swizzledSel 替换为父类的实现。这样不会影响父类行为。

---

### 场景 24：动态给 Category 添加属性

**场景描述**：给 UIButton 扩展一个 `touchEdgeInsets` 属性用于扩大点击区域。

```objc
@interface UIButton (EnlargeTouch)
@property (nonatomic, assign) UIEdgeInsets touchEdgeInsets;
@end

@implementation UIButton (EnlargeTouch)
- (UIEdgeInsets)touchEdgeInsets {
    return [objc_getAssociatedObject(self, _cmd) UIEdgeInsetsValue];
}
- (void)setTouchEdgeInsets:(UIEdgeInsets)touchEdgeInsets {
    objc_setAssociatedObject(self, @selector(touchEdgeInsets),
                             [NSValue valueWithUIEdgeInsets:touchEdgeInsets],
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    UIEdgeInsets insets = self.touchEdgeInsets;
    CGRect rect = CGRectMake(self.bounds.origin.x - insets.left,
                             self.bounds.origin.y - insets.top,
                             self.bounds.size.width + insets.left + insets.right,
                             self.bounds.size.height + insets.top + insets.bottom);
    return CGRectContainsPoint(rect, point);
}
@end
```

**面试追问**：`OBJC_ASSOCIATION_RETAIN_NONATOMIC` 和 `OBJC_ASSOCIATION_RETAIN` 有什么区别？

> 前者在存取时不加锁（非原子、快），后者加自旋锁（原子、安全但慢），对应 `@property` 的 `nonatomic` vs `atomic`。

---

## 九、KVO 场景

### 场景 25：KVO 忘记移除导致 crash

**场景描述**：页面退出后 KVO observer 没有移除，后续被观察对象变更 → crash。

```objc
// ❌ 如果 dealloc 前没 removeObserver，后续属性变化时会向已释放的 observer 发消息
[self.model addObserver:self
             forKeyPath:@"status"
                options:NSKeyValueObservingOptionNew
                context:nil];

// ✅ 保证 remove
- (void)dealloc {
    [self.model removeObserver:self forKeyPath:@"status"];
}
```

**iOS 11+ 改进**：iOS 11 起，即使忘记 remove，dealloc 时 KVO 也会自动解除部分观察（但不推荐依赖此行为）。

---

### 场景 26：KVO 手动触发 vs 自动触发

```objc
// 自动触发：直接设值
self.model.status = @"updated";  // 自动通知

// 手动触发：修改成员变量 + willChange + didChange
[self.model willChangeValueForKey:@"status"];
_model->_status = @"updated";
[self.model didChangeValueForKey:@"status"];

// 完全手动控制 KVO
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"status"]) {
        return NO;  // 手动触发
    }
    return [super automaticallyNotifiesObserversForKey:key];
}
```

---

## 十、RunLoop 场景

### 场景 27：延迟执行任务 — performSelector vs dispatch_after vs NSTimer

```objc
// performSelector — 依赖 RunLoop，精准度低
[self performSelector:@selector(task) withObject:nil afterDelay:1.0];

// dispatch_after — 不依赖 RunLoop，相对精准
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC),
               dispatch_get_main_queue(), ^{ [self task]; });

// NSTimer — 依赖 RunLoop，受 mode 影响
[NSTimer scheduledTimerWithTimeInterval:1.0 repeats:NO block:^(NSTimer *t) {
    [self task];
}];
```

**区别**：
- `performSelector:afterDelay:` 底层也是创建 NSTimer 加到 RunLoop
- `dispatch_after` 是 GCD 级别，RunLoop 休眠时也能触发
- `performSelector` 可以被 `cancelPreviousPerformRequestsWithTarget:` 取消

---

### 场景 28：常驻线程

**场景描述**：需要一个常驻线程来处理实时数据（如音视频采集），保持 RunLoop 运行。

```objc
@interface PermanentThread : NSObject
@property (nonatomic, strong) NSThread *thread;
@end

@implementation PermanentThread

- (instancetype)init {
    self = [super init];
    _thread = [[NSThread alloc] initWithTarget:self
                                      selector:@selector(runThread)
                                        object:nil];
    [_thread start];
    return self;
}

- (void)runThread {
    @autoreleasepool {
        // 添加 port 保证 RunLoop 有 source，不会立即退出
        [[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] run];  // 不停 run

        // 或者用 while 循环 + runMode:beforeDate: 可以控制退出条件
    }
}

- (void)executeTask:(void(^)(void))task {
    [self performSelector:@selector(internalTask:)
                 onThread:self.thread
               withObject:[task copy]
            waitUntilDone:NO];
}

- (void)stop {
    [self performSelector:@selector(stopThread)
                 onThread:self.thread
               withObject:nil
            waitUntilDone:YES];
}

- (void)stopThread {
    CFRunLoopStop(CFRunLoopGetCurrent());
}

@end
```

---

## 十一、实战综合场景

### 场景 29：日间/夜间模式切换

**考点**：大量 UI 实时刷新、主题管理架构

```objc
// 方案：Protocol + 通知
@protocol ThemeProtocol <NSObject>
- (void)applyTheme:(Theme *)theme;
@end

@interface ThemeManager : NSObject
+ (instancetype)shared;
@property (nonatomic, strong) Theme *currentTheme;
- (void)switchToTheme:(Theme *)theme;
@end

@implementation ThemeManager
- (void)switchToTheme:(Theme *)theme {
    self.currentTheme = theme;
    // 保存到 UserDefaults
    // 发通知，所有遵守协议的 View/VC 刷新
    [[NSNotificationCenter defaultCenter] postNotificationName:ThemeDidChangeNotification
                                                        object:theme];
}
@end
```

**每一页不需要遍历所有子视图**，而是每个 View / VC 在 `init` 或 `viewDidLoad` 时注册通知，收到通知后根据协议刷新自己的颜色/字体。

---

### 场景 30：大文件下载后 Crash（内存爆表）

**场景描述**：下载 500MB 的文件，直接 `[NSData dataWithContentsOfURL:]` crash。

**正确做法**：

```objc
NSURLSessionDownloadTask *task =
[[NSURLSession sharedSession] downloadTaskWithURL:url
                                completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
    // downloadTask 直接把文件写到磁盘（location 是临时文件路径）
    // 把它移到目标路径，全程不把整个文件放到 NSData 内存中
    NSString *destPath = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory,
                                                              NSUserDomainMask,
                                                              YES).firstObject
                          stringByAppendingPathComponent:@"largefile.zip"];
    [[NSFileManager defaultManager] moveItemAtURL:location
                                            toURL:[NSURL fileURLWithPath:destPath]
                                            error:nil];
}];
[task resume];
```

---

### 场景 31：线上问题热修复（JSPatch / Aspects / Method Swizzling）

**考点**：了解热修复的原理和风险

- JSPatch：通过 JS 调用 OC Runtime，下发 JS 脚本执行热修复（已被 App Store 限制）
- Aspects：AOP 面向切片编程，hook 方法前后插入逻辑
- Method Swizzling：直接交换 IMP，最高效但最危险

**风险**：
- App Store 审核风险（JSPatch 已被明确禁止）
- Hook 顺序不确定，可能与其他库的 swizzle 冲突
- 线上 bug 更推荐用配置开关/降级策略解决

---

### 场景 32：设计一个埋点系统

**要求**：无侵入、可配置、支持页面进入/退出/点击/曝光

```objc
// 方案 1：Method Swizzling VC 生命周期
// +load 中 swizzle viewDidAppear / viewDidDisappear

// 方案 2：Aspects hook 关键方法
[UIViewController aspect_hookSelector:@selector(viewDidAppear:)
                          withOptions:AspectPositionAfter
                           usingBlock:^(id<AspectInfo> aspectInfo) {
    [Tracker trackPageEnter:NSStringFromClass([aspectInfo.instance class])];
} error:nil];

// 方案 3：UIAlertController 等系统控件通过 hook 特定方法追踪

// 方案 4：TableView/CollectionView 曝光通过
// didEndDisplayingCell / willDisplayCell 回调追踪
```

**配置化**：埋点规则通过 JSON 配置文件下发，支持动态增删：

```json
{
  "events": [
    {"type": "page_enter", "class": "HomeViewController", "eventId": "home_show"},
    {"type": "click", "class": "BuyButton", "selector": "buyButtonTapped:", "eventId": "buy_click"}
  ]
}
```

---

### 场景 33：下载 1000 张图片的设计方案

**场景描述**：需要一次性下载 1000 张图片（如离线缓存包），要求高效、可控、不 OOM、支持进度回调。

**考点分析**：综合性设计题，覆盖并发控制、内存管理、缓存策略、错误处理、进度追踪。

---

**1. 整体架构**

```
调用方
  │
  ▼
ImageBatchDownloader（对外接口）
  ├── 接收 URL 列表 + 配置（并发数、重试次数、超时）
  ├── 统一回调：单张进度 / 总进度 / 全部完成
  │
  ▼
DownloadCoordinator（调度层）
  ├── NSOperationQueue（maxConcurrent = 4~6）
  ├── 维护待下载 / 下载中 / 已完成 / 失败四个集合
  ├── 失败重试逻辑（最多 2 次）
  │
  ▼
ImageDownloadOperation（单张下载）
  ├── NSURLSession dataTask（可 cancel）
  ├── 下载到内存 → 解码 → 写入磁盘
  └── 回调 coordinator 更新状态
```

---

**2. 并发控制 — 为什么是 4~6？**

```objc
// 并发数不能太高也不能太低
// 太高：带宽竞争、内存峰值大、HTTP/2 单连接并发有限
// 太低：带宽利用率不足、总耗时过长
// 4~6 是经验值，兼顾带宽利用与内存控制

@property (nonatomic, strong) NSOperationQueue *downloadQueue;

- (instancetype)init {
    self = [super init];
    _downloadQueue = [[NSOperationQueue alloc] init];
    _downloadQueue.maxConcurrentOperationCount = 5; // 关键
    return self;
}
```

**追问：为什么用 NSOperationQueue 而不是 GCD？**

> NSOperation 支持 cancel、依赖关系、优先级，方便单独控制每个下载任务。GCD 一旦 dispatch 就无法取消（除非自己实现 flag 判断）。

---

**3. 内存控制 — 边下边存，不堆积**

核心思路：**下载完成一张 → 解码 → 写入磁盘 → 释放内存**，而不是等 1000 张全下完再处理。

```objc
@interface ImageDownloadOperation : NSOperation

@property (nonatomic, copy) NSString *url;
@property (nonatomic, copy) NSString *diskPath;  // 目标存储路径
@property (nonatomic, copy) void(^progressBlock)(NSUInteger completed, NSUInteger total);

@end

@implementation ImageDownloadOperation

- (void)main {
    if (self.isCancelled) return;

    // 1. 网络下载
    NSData *data = [self downloadDataSync]; // 用信号量同步等待
    if (!data || self.isCancelled) return;

    // 2. 解码（这里会占用内存，但只针对单张图）
    UIImage *image = [UIImage imageWithData:data];
    if (!image) return;

    // 3. 写入磁盘（JPEG 压缩可进一步控制体积）
    NSData *jpegData = UIImageJPEGRepresentation(image, 0.85);
    [jpegData writeToFile:self.diskPath atomically:YES];

    // 4. data、image、jpegData 在 block 结束后自动释放
    // 下一张图片开始前，这一张的内存已回收
}

@end
```

**追问：如果图片是 PNG 且超大（如单张 20MB），怎么优化？**

> 不在内存中持有完整 UIImage，直接用 ImageIO 渐进式解码 + 逐块写入磁盘，避免峰值内存。或者不转为 UIImage，直接把下载的 `NSData` 落盘（缺点是无法验证图片合法性）。

---

**4. 进度回调 — 线程安全计数**

```objc
@interface DownloadCoordinator : NSObject

@property (nonatomic, assign) NSUInteger totalCount;
@property (nonatomic, assign) NSUInteger completedCount; // 注意线程安全
@property (nonatomic, strong) dispatch_queue_t callbackQueue;

@end

// 每个 Operation 完成后回调
- (void)onImageDownloaded:(NSString *)url success:(BOOL)success {
    @synchronized(self) {
        self.completedCount++;
        NSUInteger done = self.completedCount;

        dispatch_async(self.callbackQueue ?: dispatch_get_main_queue(), ^{
            if (self.progressBlock) {
                self.progressBlock(done, self.totalCount);
            }

            if (done == self.totalCount) {
                if (self.completionBlock) {
                    self.completionBlock(self.failedURLs);
                }
            }
        });
    }
}
```

---

**5. 取消 & 暂停 & 恢复**

```objc
// 取消全部 — NSOperationQueue 原生支持
- (void)cancelAll {
    [self.downloadQueue cancelAllOperations];
}

// 暂停 / 恢复 — 利用 suspended 属性
- (void)pause {
    self.downloadQueue.suspended = YES;
}

- (void)resume {
    self.downloadQueue.suspended = NO;
}
```

注意：`cancelAllOperations` 只是将队列中的 Operation 标记为 cancel，已经在执行的 Operation 需要在 `-main` 里自行检查 `self.isCancelled`。

---

**6. 失败重试 + 超时**

```objc
@interface ImageDownloadOperation : NSOperation
@property (nonatomic, assign) NSInteger maxRetryCount;  // 默认 2
@property (nonatomic, assign) NSTimeInterval timeout;     // 默认 30s
@end

@implementation ImageDownloadOperation

- (void)main {
    NSInteger retryLeft = self.maxRetryCount;

    while (retryLeft >= 0) {
        if (self.isCancelled) return;

        NSData *data = [self downloadWithTimeout:self.timeout];
        if (data) {
            [self processAndSave:data];
            return; // 成功
        }

        retryLeft--;
        if (retryLeft >= 0) {
            // 指数退避：第 1 次等 1s，第 2 次等 2s
            sleep((int)(self.maxRetryCount - retryLeft + 1));
        }
    }

    // 重试耗尽，标记失败
    [self.coordinator onImageDownloaded:self.url success:NO];
}

- (NSData *)downloadWithTimeout:(NSTimeInterval)timeout {
    __block NSData *result = nil;
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);

    NSURLSessionDataTask *task = [[NSURLSession sharedSession]
        dataTaskWithURL:[NSURL URLWithString:self.url]
        completionHandler:^(NSData *data, NSURLResponse *resp, NSError *err) {
            result = data;
            dispatch_semaphore_signal(sem);
        }];
    [task resume];

    // 超时控制
    dispatch_semaphore_wait(sem, dispatch_time(DISPATCH_TIME_NOW, timeout * NSEC_PER_SEC));
    [task cancel]; // 超时取消
    return result;
}

@end
```

---

**7. 面试追问汇总**

| 追问 | 回答要点 |
|------|----------|
| 并发数为什么是 4~6？ | HTTP/2 单连接并发限制、带宽竞争、内存峰值三者平衡 |
| NSOperation vs GCD？ | cancel / 依赖 / 优先级，GCD 不具备 |
| 1000 张下完内存峰值多少？ | 并发数 × 单张解码内存（约 5 × 8MB = 40MB） |
| 如何避免重复下载？ | 下载前检查磁盘文件是否已存在（通过 URL hash 映射文件路径） |
| 后台下载怎么做？ | NSURLSession background configuration，系统级调度，进程退出也继续 |
| 下载中途杀进程重启怎么办？ | 持久化下载列表 + 已完成的 URL 集合，重启后跳过已完成的 |

---

**8. 完整对外接口**

```objc
@interface ImageBatchDownloader : NSObject

/// 批量下载图片
/// @param urls 图片 URL 数组
/// @param destDir 目标存储目录（如 Library/Caches/offline_images）
/// @param progress 总进度回调 (已完成数, 总数)
/// @param completion 完成回调 (失败 URL 数组)
- (void)downloadImages:(NSArray<NSString *> *)urls
               destDir:(NSString *)destDir
              progress:(void(^)(NSUInteger completed, NSUInteger total))progress
            completion:(void(^)(NSArray<NSString *> *failedURLs))completion;

/// 取消全部
- (void)cancelAll;

/// 暂停 / 恢复
@property (nonatomic, assign, getter=isSuspended) BOOL suspended;

/// 并发数（默认 5）
@property (nonatomic, assign) NSInteger maxConcurrentDownloads;

@end
```

---

## 面试速查表

| 场景编号 | 关键词 | 核心考点 |
|----------|--------|----------|
| 1 | Block 循环引用 | weak-strong dance |
| 2 | NSTimer | RunLoop、中间代理对象 |
| 3 | delegate weak/strong | 循环引用判断 |
| 5 | 多线程写入 | barrier / lock / @synchronized |
| 8 | 主线程卡顿 | RunLoop Observer 监控 |
| 9 | TableView 掉帧 | cellForRow 耗时 + 离屏渲染 |
| 13 | unrecognized selector | 消息转发链、防 crash |
| 17 | 胖 Controller | MVVM / DataSource 分离 |
| 18 | 组件化跳转 | Router / Mediator / Target-Action |
| 19 | 请求重复 | 防重点击 / NSOperation cancel |
| 21 | 图片缓存 | 三级缓存 / NSCache LRU |
| 22 | 启动优化 | Pre-main + Main 阶段分析 |
| 23 | Method Swizzling | +load 时机 + class_addMethod |
| 24 | Category 加属性 | Associated Object |
| 28 | 常驻线程 | RunLoop source / mode |
| 33 | 批量下载 1000 张图片 | 并发控制 / 内存峰值 / NSOperation vs GCD / 失败重试 / 进度回调 |
