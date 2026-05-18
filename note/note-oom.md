# OOM (Out of Memory)

## 什么是 OOM

OOM 指应用因内存使用超过系统限制而被系统强制终止。iOS 上 OOM 分为两类：

- **前台 OOM (FOOM)**：应用在前台时因内存超限被 kill，用户体验最差（闪退）。
- **后台 OOM (BOOM)**：应用在后台时被系统回收，用户感知不到（下次冷启动）。

> 注意区分 Crash：OOM 不会产生 Crash 日志（系统直接发 SIGKILL，信号 9），也无法被 Mach Exception 或 NSException 捕获。

---

## iOS 内存管理底层原理

### 1. 虚拟内存 (Virtual Memory)

iOS 和 macOS 都使用虚拟内存系统，但 **iOS 没有交换分区 (Swap)**，这是理解 OOM 的关键前提。

- 每个进程拥有独立的 4GB（32 位）或 18EB（64 位实际用 64GB）虚拟地址空间。
- 虚拟内存按页（Page，iOS 上通常 16KB）管理。
- **Clean Memory**：可从磁盘重新加载的内存（代码段、系统 framework、mmap 映射的文件），系统可随时回收。
- **Dirty Memory**：已被写入修改的内存（堆、栈、全局变量等），无法回收，只能压缩或等进程主动释放。

```
物理内存紧张时：
  Clean Page → 直接丢弃，下次访问再从磁盘加载
  Dirty Page → 无法丢弃，只能压缩 (Compressed Memory) 或杀进程
```

> 因为没有 Swap，Dirty Memory 要么压缩、要么最终导致 OOM/Jetsam。

### 2. 内存页类型

| 类型 | 可回收 | 来源 |
|------|--------|------|
| Clean (file-backed) | 是 | 代码段、mmap 文件、framework __TEXT 段 |
| Dirty (anonymous) | 否 | malloc/new、栈、__DATA 段、copy-on-write 后的页 |
| Compressed | 否（已处理） | Dirty 页被压缩后存放 |
| Wired | 否 | 内核/驱动使用，永不换出 |

**Copy-on-Write (COW)**：fork 出的子进程共享父进程内存页，标记为只读。任一进程写入时，内核复制该页并将其标记为 Dirty。这就是 iOS 创建新进程时的优化手段。

### 3. Unified Memory Architecture (统一内存架构)

iOS 设备的 CPU 和 GPU 共享同一块物理内存（统一内存）：

- 传统 PC：CPU 和 GPU 各有独立显存，需要显式拷贝。
- iOS：两者共用 LPDDR，虽然省了拷贝但竞争也更激烈。
- 这意味着图片解码、渲染等 GPU 操作占用的内存也计入进程 Dirty Memory，直接推高 OOM 风险。

### 4. Memory Compressor (内存压缩)

iOS 7+ 引入，是 Swap 的替代方案：

1. 当内存压力升高，内核识别出最近最少使用的 Dirty Page。
2. 使用 WKdm 算法压缩（针对 4KB 块，iOS 解压极快），压缩比约 50%。
3. 压缩后的内存存入专门区域，原虚拟地址页标记为"已压缩"。
4. 进程访问该页时，内核解压并换入。

```
未压缩: 800MB   →   压缩后: 约 400-500MB（取决于数据可压缩性）
```

### 5. Jetsam 机制

Jetsam 是 iOS 的内存管理守护进程，负责在内存压力下选择进程杀掉。

**Jetsam 优先级（从低到高，越小越容易被杀）**：

| 优先级 | 角色 | 示例 |
|--------|------|------|
| Idle (0-1) | 后台、无焦点 | 后台大内存 App |
| Utility (3-4) | 系统工具 | 后台播放 |
| Accessory (5) | 外设配套 | MFi 设备应用 |
| Elevated (8) | 前台应用 | 用户正在使用的 App |
| Critical (10+) | 系统关键 | SpringBoard、phone |

触发条件：
1. 系统剩余物理内存低于阈值
2. 单个进程 Dirty Memory 超过"jetsam limit"（设备相关，如 1GB 内存设备前台约 50-65%）

---

## 内存分区（进程地址空间）

```
高地址
+------------------+
|    内核空间       |  (1GB, 32位)
+------------------+
|   栈 (Stack)     |  ↓ 向下增长，每个线程一个，默认 1MB (主线程) / 512KB (子线程)
+------------------+
|   mmap 区域      |  文件映射、图片、大内存分配（>128KB malloc 也可能走 mmap）
+------------------+
|   堆 (Heap)      |  ↑ 向上增长，malloc/new/OC 对象
+------------------+
|   BSS 段         |  未初始化的全局/静态变量
+------------------+
|   Data 段        |  已初始化的全局/静态变量
+------------------+
|   Text 段        |  代码（Clean Memory，只读）
+------------------+
低地址
```

> 64 位后 ASLR 使实际布局更复杂，但逻辑结构类似。栈大小可通过 `-[NSThread stackSize]` 设置。

---

## 常见 OOM 场景（面试必考）

### 1. 大图片加载

```objc
// 错误示例：直接加载 12000x9000 图片
UIImage *image = [UIImage imageNamed:@"huge_photo.jpg"];
// 解码后占用内存 = 12000 * 9000 * 4 bytes = 432MB（RGBA 每像素 4 字节）
// 图片实际文件大小可能只有几 MB（压缩后的 JPEG），但解压到内存后是位图

// 正确做法：降采样
- (UIImage *)downsampledImageFromData:(NSData *)data toSize:(CGSize)size {
    CGImageSourceOptions options = @{
        (__bridge id)kCGImageSourceCreateThumbnailFromImageAlways: @YES,
        (__bridge id)kCGImageSourceCreateThumbnailWithTransform: @YES,
        (__bridge id)kCGImageSourceThumbnailMaxPixelSize: @(MAX(size.width, size.height)),
    };
    CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)data, NULL);
    CGImageRef imageRef = CGImageSourceCreateThumbnailAtIndex(source, 0, (__bridge CFDictionaryRef)options);
    UIImage *image = [UIImage imageWithCGImage:imageRef];
    CGImageRelease(imageRef);
    CFRelease(source);
    return image;
}
```

**面试关键数据**：
- 图片内存计算公式：`width × height × 4 bytes`（RGBA8，即 SRGB 格式）
- 带 Alpha 和不带的区别：不带 Alpha 也是 4 字节对齐（iOS 上层 API 统一用 32BGRA/CGBitmapInfo 不作区分）
- 图片解码在哪个线程：主线程同步解码（`imageNamed` / `imageWithContentsOfFile:` 返回后未完成），真正解码发生在主线程渲染时。**`imageNamed` 会对已解码图片做缓存**，`imageWithContentsOfFile:` 不会。

### 2. 循环引用导致内存泄漏

```objc
// 经典循环引用
@interface MyViewController : UIViewController
@property (nonatomic, copy) void(^completionBlock)(void);
@end

// Block 强引用 self，self 强引用 block → 循环引用
self.completionBlock = ^{
    [self doSomething];  // self 被 block 捕获
};

// 正确做法：使用 weakSelf
__weak typeof(self) weakSelf = self;
self.completionBlock = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf doSomething];
};
```

常见循环引用场景：
- **Block 捕获 self**：`self.block = ^{ [self x]; }`
- **NSTimer 强引用 target**：Timer 被 RunLoop 持有，target 又被 timer 持有 → 破环方法：使用 iOS 10+ 的 block API 或用 Proxy 中间对象
- **delegate 用 strong**：应该用 `weak`
- **Core Foundation 对象不手动释放**：`CGImageRef`、`CGContextRef` 等 Create/Copy 出来的对象

```objc
// NSTimer 不破环方案
// 方案1：iOS 10+ block API
[NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer *timer) {
    [weakSelf onTimerTick];
}];

// 方案2：中间 Proxy 对象
@interface WeakProxy : NSProxy
@property (nonatomic, weak) id target;
@end
@implementation WeakProxy
- (id)forwardingTargetForSelector:(SEL)aSelector { return self.target; }
@end
```

### 3. 并发 / 线程过多

- 每个线程栈（子线程 512KB，主线程 1MB）都是 Dirty Memory。
- 线程过多直接占用大量 Dirty Memory + 上下文切换开销。
- 应使用 GCD 的线程池管理，而不是直接 `pthread_create` 或 `NSThread` 大量创建。

### 4. 后台任务未释放

```objc
// 问题场景
- (void)applicationDidEnterBackground:(UIApplication *)application {
    // 没有主动释放大内存对象（如缓存的图片、数据模型）
    // 下次前台时可能已被 jetsam kill
}

// 正确做法
- (void)applicationDidEnterBackground:(UIApplication *)application {
    // 1. 清空图片缓存
    [[SDImageCache sharedImageCache] clearMemory];
    // 2. 关闭打开的数据库连接
    // 3. 暂停当前的解码/渲染操作
}
```

### 5. WebView / JS 内存泄漏

`WKWebView` 使用独立进程，但 `UIWebView`（已废弃）在主进程中运行。`WKWebView` 的 `configuration.userContentController` 有强引用问题：

```objc
// 问题：addScriptMessageHandler 会强引用 self
[self.webView.configuration.userContentController addScriptMessageHandler:self name:@"native"];
// → self → webView → configuration → userContentController → scriptMessageHandlers → self

// 正确：页面销毁时 remove
- (void)dealloc {
    [self.webView.configuration.userContentController removeScriptMessageHandlerForName:@"native"];
}
```

### 6. 大对象 / 快速分配

```objc
// 问题：循环中快速创建大量临时对象，触发 autorelease pool 未及时排空
for (int i = 0; i < 1000000; i++) {
    NSString *str = [NSString stringWithFormat:@"item_%d", i];
    // str 被加到 autorelease pool，在 pool drain 前不会释放
}

// 正确：使用 @autoreleasepool
for (int i = 0; i < 1000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"item_%d", i];
        // 每次迭代结束就释放
    }
}
```

### 7. 纹理 / 渲染内存

- OpenGL ES / Metal 纹理、FrameBuffer 占用 GPU 端统一内存。
- `UIGraphicsImageRenderer` 创建的大画布 `UIGraphicsBeginImageContextWithOptions` 也计入 Dirty Memory。
- 离屏渲染创建的临时缓冲区。

```objc
// 开启了 scale:0 (即设备 scale) 的 context
UIGraphicsBeginImageContextWithOptions(bigSize, YES, 0);
// bigSize = 2000x2000，scale = 3 → 实际像素缓冲区为 6000x6000x4 = 144MB
```

---

## 内存检测与排查工具

### 1. Xcode Memory Graph Debugger

- 可视化对象引用关系，找到循环引用。
- 在 Debug Navigator 中点击内存图标 → 查看所有存活对象和引用链。

### 2. Instruments - Allocations

- 查看进程内存增长趋势、分配热点。
- 按堆栈追踪每个 malloc 调用。
- 可录制 VM Tracker 查看各类内存（Dirty/Swapped/Compressed）占比。

### 3. Instruments - Leaks

- 检测泄漏对象（有分配但无任何引用指向的内存）。
- 注意：只检测纯泄漏（完全不可达），不检测逻辑泄漏（循环引用导致活着但不该活着的对象）。

### 4. Instruments - VM Tracker

- 直接看到 **Dirty Size** 和 **Resident Size**，这是判断 OOM 最重要的指标。
- 区分 Clean vs Dirty，定位哪些区域的 Dirty 增长最快。

### 5. MetricKit (iOS 13+)

```objc
// 系统提供的 OOM 诊断数据
- (void)didReceiveDiagnostic:(MXDiagnosticPayload *)payload {
    for (MXCrashDiagnostic *crashDiag in payload.crashDiagnostics) {
        // 包含 crash 前的内存快照、调用栈
    }
}
```

### 6. Jetsam 日志查看

设备 → 设置 → 隐私 → 分析与改进 → 分析数据 → 查找以 `JetsamEvent-` 开头的日志。

关键字段解读：

```
"uuid" = "xxx"                       // 进程 ID
"states" = ["frontmost", "resume"]    // 被杀时是前台
"priority" = 8                        // 前台高优先级但仍被杀
"rpages" = 98304                      // 占用物理内存页数，每页 16KB → 约 1.5GB
"reason" = "vnode-limit"             // 被杀原因（也可能是 "highwater"、"vm-pageshortage" 等）
```

**reason 含义**：
| Reason | 含义 |
|--------|------|
| `per-process-limit` | 超过单进程内存上限 |
| `vm-pageshortage` | 系统整体内存不足 |
| `vnode-limit` | 打开文件描述符过多 |
| `highwater` | 内存增长过快触发 |


### 7. 线上监控关键指标

```objc
// 获取进程当前物理内存（Dirty Memory + Compressed）
- (int64_t)memoryFootprint {
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    kern_return_t result = task_info(mach_task_self(), TASK_VM_INFO, (task_info_t)&vmInfo, &count);
    if (result != KERN_SUCCESS) return 0;
    return vmInfo.phys_footprint;  // 这是 Jetsam 判定的关键值，单位 byte
}
```

**面试重点**：`phys_footprint` 就是 Jetsam 判定 OOM 的核心指标，Android 的 PSS 与之类似但非同一个概念。

---

## ARC 与内存管理底层

### 引用计数存储

```objc
// 引用计数存储在 isa 指针或 SideTable 中
// isa 是 union 类型，部分位存 extra_rc (额外引用计数)
// 当 extra_rc 不够用时溢出到 SideTable

// 底层调用链：
// retain → _objc_rootRetain → rootRetain → sidetable_retain (溢出时)
// release → _objc_rootRelease → rootRelease → sidetable_release (溢出时)
```

### tagged pointer

```objc
// 小对象（如 NSNumber、小 NSString、NSDate）存储在指针本身的值中，不分配堆内存
// 64 位架构下，地址实际用不到 64 位，多余位用来存值
// 条件：值能用 60 位（或更少位）表示

NSNumber *num = @42;           // tagged pointer → 不占堆内存
NSNumber *bigNum = @(0xFFFFFFFFFFFFFFFF); // 超出范围 → 正常堆分配
```

**优化效果**：减少堆内存分配，减少 retain/release 开销，极端情况下对 OOM 有一定缓解。

### SideTable

```objc
// SideTable 结构：每个 SideTable 对应一段地址范围的对象的引用计数
struct SideTable {
    spinlock_t slock;         // 自旋锁（已优化为 os_unfair_lock）
    RefcountMap refcnts;      // 溢出引用计数表
    weak_table_t weak_table;  // weak 引用表
};

// 全局有多张 SideTable 分散存储，通过对象地址 hash 找到对应表，减少竞争
```

---

## 面试高频考点总结

### 考点 1：图片为什么会占那么大内存？

图片文件（JPEG/PNG）是压缩格式，显示前必须解码为位图（Bitmap）。位图内存 = 宽 × 高 × 每像素字节数。一张 100KB 的 JPEG，解压后可能占几十 MB。

### 考点 2：怎么计算 App 实际使用的物理内存？

用 `task_vm_info.phys_footprint`，不是 `resident_size`。`phys_footprint` 是 Jetsam 做 OOM 判定时使用的指标，包含了 Dirty Memory + Compressed Memory。

### 考点 3：ARC 和 MRC 下的内存管理区别，哪些对象需要手动释放？

- Core Foundation 对象（Create/Copy 规则的函数返回的）：`CFRelease()`
- `CGImageRef`、`CGContextRef` 等 Core Graphics 对象
- MRC 下所有 NSObject 子类需要手动 retain/release
- ARC 下可以用 `__bridge_transfer` / `CFBridgingRelease` 在 CF 和 NS 间转移所有权

### 考点 4：`@autoreleasepool` 的使用场景和原理

- 原理：每个 RunLoop 迭代开始创建一个 autorelease pool，结束时 drain。
- 需要手动添加的场景：循环中创建大量临时对象、子线程（没有 RunLoop 时 autorelease 不会及时释放）。
- 底层：AutoreleasePoolPage 是双向链表，每个 Page 约 4KB，通过 `push`/`pop` 管理对象的 autorelease 标记。

### 考点 5：weak 指针的实现原理

1. `__weak` 修饰的对象，编译器生成 `objc_initWeak` / `objc_destroyWeak` 调用。
2. 运行时维护一个全局 weak 表（SideTable 中），key 是对象地址，value 是所有指向该对象的 weak 指针地址数组。
3. 对象 dealloc 时，遍历 weak 表找到所有指向它的 weak 指针，全部置为 nil。

### 考点 6：OOM 和 Crash 的区别

| | OOM | Crash |
|---|---|---|
| 信号 | SIGKILL (9) | SIGABRT/SIGSEGV/SIGBUS 等 |
| 可捕获 | 否 | 是（NSException / signal handler） |
| 是否有 crash log | 否 | 是（.crash / .ips 文件） |
| 原因 | 系统 Jetsam 判定内存超限 | 代码逻辑错误/内存访问越界等 |
| 线上监控 | 通过启动时长/前后台变化推断 | 直接捕获上传 |

### 考点 7：线上如何判断 OOM？

- 应用启动时记录上次退出是否正常（设置 flag）。
- App 被 kill 时没有机会执行 `applicationWillTerminate`（前台 kill）或正常退出标记。
- 判断流程：上次没有正常退出 + 不是 crash + 不是系统升级 + 不是用户主动杀掉的 → 大概率是 OOM。
- 配合 `phys_footprint` 在关键路径（大图加载、进入主要页面后）上报。

### 考点 8：为什么 iOS 没有 Android 那样的 GC 但还是有 OOM？

iOS 使用 ARC（编译期插入 retain/release，确定性内存释放），理论上可以做到"引用计数归零立即释放"。但 OOM 来源不止泄漏：
- 瞬时峰值：大图解码、大画布渲染
- 逻辑上的"活"对象过多（全量缓存、一次加载大量数据）
- retain cycle 导致逻辑泄漏
- 后台线程 autorelease pool 未及时 drain
- 系统整体内存不足（Jetsam 从全局考虑）

---

## 总结：OOM 防护最佳实践 Cheatsheet

1. **图片降采样**：不要直接加载超大图，用 `CGImageSourceCreateThumbnailAtIndex`
2. **及时释放**：大对象用完置 nil，用 `@autoreleasepool` 包裹临时对象
3. **断循环引用**：block 用 weak-strong dance，NSTimer 用 block API，delegate 用 weak
4. **监控 phys_footprint**：关键路径打点上报，接近设备阈值时降级（释放缓存）
5. **后台清缓存**：`applicationDidEnterBackground:` 清空非必要内存
6. **避免线程爆炸**：用 GCD 线程池而非大量创建 NSThread
7. **CF 对象配对释放**：Create/Copy 系列必须 CFRelease
8. **线上 OOM 率监控**：通过前后台和退出标识推断 + MetricKit 诊断
