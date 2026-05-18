# SDWebImage 源码深度解析 - iOS 底层架构师进阶指南

> 基于 SDWebImage 5.x 完整源码，面向 iOS Objective-C 高级开发者的系统学习文档。
> 全部内容围绕 Objective-C 实现，不涉及 Swift。

---

## 目录

1. [整体架构设计](#1-整体架构设计)
2. [多线程与并发模型](#2-多线程与并发模型)
3. [内存管理与缓存架构](#3-内存管理与缓存架构)
4. [iOS 底层知识点](#4-ios-底层知识点)
5. [网络层与任务管理](#5-网络层与任务管理)
6. [设计模式在 SDWebImage 中的真实应用](#6-设计模式在-sdwebimage-中的真实应用)
7. [性能优化实战](#7-性能优化实战)
8. [ObjC 高级编码规范](#8-objc-高级编码规范)
9. [源码精读路线图](#9-源码精读路线图)
10. [高频面试题](#10-高频面试题)

---

## 1. 整体架构设计

### 1.1 模块划分

SDWebImage 采用分层架构设计，核心模块如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    Category Layer (分类层)                    │
│     UIImageView+WebCache  UIButton+WebCache  UIView+WebCache │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Manager Layer (管理层)                     │
│                      SDWebImageManager                        │
│           (协调 Cache 和 Loader，对外统一接口)                  │
└─────────────────────────────────────────────────────────────┘
                    │                       │
          ┌─────────┘                       └─────────┐
          ▼                                           ▼
┌──────────────────────┐                ┌──────────────────────┐
│    Cache Layer       │                │    Loader Layer      │
│    (缓存层)          │                │    (加载层)          │
│                      │                │                      │
│  SDImageCache        │                │  SDWebImageDownloader│
│  SDMemoryCache       │                │  SDWebImageDownloader│
│  SDDiskCache         │                │     Operation        │
└──────────────────────┘                └──────────────────────┘
          │                                           │
          ▼                                           ▼
┌──────────────────────┐                ┌──────────────────────┐
│   Decoder Layer      │                │   Network Layer      │
│   (解码层)           │                │   (网络层)           │
│                      │                │                      │
│ SDImageCodersManager │                │    NSURLSession      │
│ SDImageIOCoder       │                │                      │
│ SDImageGIFCoder      │                │                      │
│ SDImageAPNGCoder     │                │                      │
└──────────────────────┘                └──────────────────────┘
```

### 1.2 核心类职责

| 类名 | 职责 | 源码位置 |
|------|------|----------|
| `SDWebImageManager` | 核心协调者，统筹缓存查询与网络加载 | `SDWebImage/Core/SDWebImageManager.h:101` |
| `SDImageCache` | 图片缓存管理，内存缓存+磁盘缓存 | `SDWebImage/Core/SDImageCache.h:79` |
| `SDMemoryCache` | 内存缓存实现，基于 NSCache 扩展 | `SDWebImage/Core/SDMemoryCache.h:74` |
| `SDDiskCache` | 磁盘缓存实现，文件系统存储 | `SDWebImage/Core/SDDiskCache.h:125` |
| `SDWebImageDownloader` | 下载器，管理下载队列和请求 | `SDWebImage/Core/SDWebImageDownloader.h:142` |
| `SDWebImageDownloaderOperation` | 单个下载任务，继承 NSOperation | `SDWebImage/Core/SDWebImageDownloaderOperation.h:57` |
| `SDImageCodersManager` | 解码器管理，责任链模式 | `SDWebImage/Core/SDImageCodersManager.h:32` |
| `SDWebImageCombinedOperation` | 组合操作，包装 cache + loader 操作 | `SDWebImageManager.h:25` |

### 1.3 依赖关系与单向依赖设计

**设计原则：单向依赖，降低耦合**

```
UIView+WebCache  ──依赖──▶  SDWebImageManager
                               │
                   ┌───────────┼───────────┐
                   ▼           ▼           ▼
            SDImageCache  SDImageLoader  SDImageTransformer
                   │           │
                   ▼           ▼
            SDDiskCache   SDWebImageDownloader
             SDMemoryCache      │
                                 ▼
                         SDWebImageDownloaderOperation
```

**源码体现：**

```objc
// SDWebImageManager.m:87-96
- (instancetype)initWithCache:(id<SDImageCache>)cache loader:(id<SDImageLoader>)loader {
    if ((self = [super init])) {
        _imageCache = cache;      // 依赖注入，不直接创建
        _imageLoader = loader;    // 依赖注入，支持替换
        _failedURLs = [NSMutableSet new];
        SD_LOCK_INIT(_failedURLsLock);
        _runningOperations = [NSMutableSet new];
        SD_LOCK_INIT(_runningOperationsLock);
    }
    return self;
}
```

**为什么这么设计？**

1. **依赖注入**：`SDWebImageManager` 通过 init 方法接收 Cache 和 Loader，允许外部注入自定义实现
2. **面向协议编程**：`SDImageCache` 和 `SDImageLoader` 都是协议，支持替换具体实现
3. **单一职责**：每个类只负责一件事，Manager 只负责协调

### 1.4 可扩展、可替换、可单元测试

**1. 可扩展性 - 通过协议和配置类**

```objc
// SDImageCacheConfig.h:143-151
// 自定义内存缓存类
@property (assign, nonatomic, nonnull) Class memoryCacheClass;
// 自定义磁盘缓存类  
@property (assign ,nonatomic, nonnull) Class diskCacheClass;
```

**2. 可替换性 - 通过协议约束**

```objc
// SDMemoryCache.h:15-69
@protocol SDMemoryCache <NSObject>
@required
- (instancetype)initWithConfig:(SDImageCacheConfig *)config;
- (id)objectForKey:(id)key;
- (void)setObject:(id)object forKey:(id)key cost:(NSUInteger)cost;
- (void)removeObjectForKey:(id)key;
- (void)removeAllObjects;
@end
```

**3. 可测试性 - 通过依赖注入**

```objc
// 测试时可以注入 Mock 对象
SDImageCache *mockCache = [[MockImageCache alloc] init];
SDWebImageManager *manager = [[SDWebImageManager alloc] initWithCache:mockCache 
                                                               loader:mockLoader];
```

---

## 2. 多线程与并发模型

### 2.1 NSOperation / NSOperationQueue 设计

**SDWebImageDownloader 的队列设计：**

```objc
// SDWebImageDownloader.m:117-119
_downloadQueue = [NSOperationQueue new];
_downloadQueue.maxConcurrentOperationCount = _config.maxConcurrentDownloads;
_downloadQueue.name = @"com.hackemist.SDWebImageDownloader.downloadQueue";
```

**并发数量控制：**

```objc
// SDWebImageDownloaderConfig.h:41
@property (nonatomic, assign) NSInteger maxConcurrentDownloads; // 默认 6

// 动态修改并发数 - 通过 KVO
// SDWebImageDownloader.m:116
[_config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxConcurrentDownloads)) 
           options:0 context:SDWebImageDownloaderContext];

// SDWebImageDownloader.m:440-443
- (void)observeValueForKeyPath:(NSString *)keyPath ... {
    if ([keyPath isEqualToString:NSStringFromSelector(@selector(maxConcurrentDownloads))]) {
        self.downloadQueue.maxConcurrentOperationCount = self.config.maxConcurrentDownloads;
    }
}
```

### 2.2 任务优先级设计

```objc
// SDWebImageDownloader.m:397-401
if (options & SDWebImageDownloaderHighPriority) {
    operation.queuePriority = NSOperationQueuePriorityHigh;
} else if (options & SDWebImageDownloaderLowPriority) {
    operation.queuePriority = NSOperationQueuePriorityLow;
}
```

### 2.3 LIFO 执行顺序实现

**巧妙的设计：通过依赖关系实现 LIFO**

```objc
// SDWebImageDownloader.m:403-410
if (self.config.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
    // 后来的 operation 让之前所有 pending 的 operation 依赖自己
    // 这样新 operation 会先执行，实现 LIFO
    for (NSOperation *pendingOperation in self.downloadQueue.operations) {
        [pendingOperation addDependency:operation];
    }
}
```

**原理图解：**

```
FIFO 模式：Operation1 -> Operation2 -> Operation3 (先入先出)

LIFO 模式：
  1. 添加 Operation1: queue = [Op1]
  2. 添加 Operation2: Op1 依赖 Op2 -> queue = [Op2, Op1]，Op2 先执行
  3. 添加 Operation3: Op1, Op2 都依赖 Op3 -> Op3 先执行
```

### 2.4 线程安全设计

#### 2.4.1 锁的选择与封装

**SDWebImage 内部锁封装（SDInternalMacros.h）：**

```objc
// SDInternalMacros.h:14-63

// 根据系统版本选择锁类型
#define SD_USE_OS_UNFAIR_LOCK TARGET_OS_MACCATALYST ||\
    (__IPHONE_OS_VERSION_MIN_REQUIRED >= __IPHONE_10_0) ||\
    (__MAC_OS_X_VERSION_MIN_REQUIRED >= __MAC_10_12)

// 锁声明
#if SD_USE_OS_UNFAIR_LOCK
#define SD_LOCK_DECLARE(lock) os_unfair_lock lock
#else
#define SD_LOCK_DECLARE(lock) os_unfair_lock lock API_AVAILABLE(...); \
OSSpinLock lock##_deprecated;  // 低版本回退到 OSSpinLock
#endif

// 锁操作
#define SD_LOCK(lock) os_unfair_lock_lock(&lock)
#define SD_UNLOCK(lock) os_unfair_lock_unlock(&lock)
```

**为什么选择 os_unfair_lock？**

1. **性能最优**：在 iOS 10+ 上，`os_unfair_lock` 替代了 `OSSpinLock`
2. **避免优先级反转**：`OSSpinLock` 存在优先级反转问题，`os_unfair_lock` 修复了这个问题
3. **轻量级**：适用于短时间的临界区保护

#### 2.4.2 实际应用场景

**场景1：保护 failedURLs 集合**

```objc
// SDWebImageManager.m:30-33
@interface SDWebImageManager () {
    SD_LOCK_DECLARE(_failedURLsLock);      // 声明锁
    SD_LOCK_DECLARE(_runningOperationsLock);
}

// 读操作
// SDWebImageManager.m:212-216
SD_LOCK(_failedURLsLock);
isFailedUrl = [self.failedURLs containsObject:url];
SD_UNLOCK(_failedURLsLock);

// 写操作
// SDWebImageManager.m:446-448
SD_LOCK(self->_failedURLsLock);
[self.failedURLs addObject:url];
SD_UNLOCK(self->_failedURLsLock);
```

**场景2：保护 HTTPHeaders 字典**

```objc
// SDWebImageDownloader.m:69-71
@implementation SDWebImageDownloader {
    SD_LOCK_DECLARE(_HTTPHeadersLock);
    SD_LOCK_DECLARE(_operationsLock);
}

// SDWebImageDownloader.m:185-188
- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field {
    SD_LOCK(_HTTPHeadersLock);
    [self.HTTPHeaders setValue:value forKey:field];
    SD_UNLOCK(_HTTPHeadersLock);
}
```

#### 2.4.3 @synchronized 的使用

**用于需要递归锁的场景：**

```objc
// SDWebImageDownloaderOperation.m:139-144
- (id)addHandlersForProgress:... completed:... decodeOptions:... {
    @synchronized (self) {
        [self.callbackTokens addObject:token];
    }
    return token;
}

// SDWebImageCombinedOperation.m:758-782
- (BOOL)isCancelled {
    @synchronized (self) {  // 递归锁，因为 cancel 方法可能被调用者检查
        return _cancelled;
    }
}
```

**为什么用 @synchronized？**

1. **递归锁需求**：`@synchronized` 是可重入的（递归锁）
2. **简洁性**：对于简单的对象级别同步，代码更简洁
3. **性能权衡**：虽然比 `os_unfair_lock` 慢，但在非热点路径可接受

### 2.5 dispatch_queue 的使用

**IO 串行队列设计：**

```objc
// SDImageCache.m:66
@property (nonatomic, strong, nonnull) dispatch_queue_t ioQueue;

// SDImageCache.m:121
_ioQueue = dispatch_queue_create("com.hackemist.SDImageCache.ioQueue", ioQueueAttributes);
```

**为什么磁盘 IO 用串行队列？**

1. **避免文件系统竞争**：多个线程同时读写文件可能导致数据损坏
2. **简化同步逻辑**：串行队列天然保证顺序执行
3. **可配置**：通过 `ioQueueAttributes` 可配置为并发队列（需配合原子写入）

**解码队列设计：**

```objc
// SDWebImageDownloaderOperation.m:115-117
_coderQueue = [[NSOperationQueue alloc] init];
_coderQueue.maxConcurrentOperationCount = 1;  // 串行队列
_coderQueue.name = @"com.hackemist.SDWebImageDownloaderOperation.coderQueue";
```

---

## 3. 内存管理与缓存架构

### 3.1 内存缓存 - SDMemoryCache 原理

**继承自 NSCache 并扩展弱引用缓存：**

```objc
// SDMemoryCache.h:74
@interface SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType> <SDMemoryCache>

// SDMemoryCache.m:16-26
@interface SDMemoryCache () {
    SD_LOCK_DECLARE(_weakCacheLock);
}
@property (nonatomic, strong, nonnull) NSMapTable<KeyType, ObjectType> *weakCache; // strong-weak cache
@end
```

**双层缓存设计：强引用 + 弱引用**

```objc
// SDMemoryCache.m:84-95 - 写入
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];  // NSCache 强引用
    if (!self.config.shouldUseWeakMemoryCache) return;
    if (key && obj) {
        SD_LOCK(_weakCacheLock);
        [self.weakCache setObject:obj forKey:key];  // NSMapTable 弱引用
        SD_UNLOCK(_weakCacheLock);
    }
}

// SDMemoryCache.m:97-117 - 读取
- (id)objectForKey:(id)key {
    id obj = [super objectForKey:key];  // 先查强引用
    if (!self.config.shouldUseWeakMemoryCache) return obj;
    if (key && !obj) {
        SD_LOCK(_weakCacheLock);
        obj = [self.weakCache objectForKey:key];  // 再查弱引用
        SD_UNLOCK(_weakCacheLock);
        if (obj) {
            // 命中弱引用缓存，回写强引用缓存
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}
```

**为什么需要弱引用缓存？**

1. **内存警告后恢复**：NSCache 在内存警告时会清理缓存，但 UIImageView 可能还持有图片
2. **避免重复加载**：弱引用缓存可以在强引用被清理后，从弱引用中恢复
3. **优化用户体验**：避免 cell 闪烁

**内存警告处理：**

```objc
// SDMemoryCache.m:78-81
- (void)didReceiveMemoryWarning:(NSNotification *)notification {
    [super removeAllObjects];  // 只清理强引用，保留弱引用
}
```

### 3.2 磁盘缓存 - SDDiskCache 实现

**文件存储路径设计：**

```objc
// SDDiskCache.m:258-261
- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path {
    NSString *filename = SDDiskCacheFileNameForKey(key);
    return [path stringByAppendingPathComponent:filename];
}
```

**文件名生成 - MD5 哈希：**

```objc
// SDDiskCache.m:306-323
static inline NSString * SDDiskCacheFileNameForKey(NSString *key) {
    const char *str = key.UTF8String;
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, (CC_LONG)strlen(str), r);  // MD5 哈希
    
    NSString *ext = keyURL.pathExtension;
    NSString *filename = [NSString stringWithFormat:@"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%@",
                          r[0], r[1], ... r[15], 
                          ext.length == 0 ? @"" : [NSString stringWithFormat:@".%@", ext]];
    return filename;
}
```

**为什么用 MD5？**

1. **固定长度**：URL 可能很长，MD5 生成固定 32 字符文件名
2. **避免特殊字符**：URL 中的特殊字符可能导致文件系统问题
3. **唯一性**：冲突概率极低

**过期策略实现：**

```objc
// SDDiskCache.m:140-231
- (void)removeExpiredData {
    // 支持多种过期类型
    NSURLResourceKey cacheContentDateKey;
    switch (self.config.diskCacheExpireType) {
        case SDImageCacheConfigExpireTypeAccessDate:
            cacheContentDateKey = NSURLContentAccessDateKey; break;
        case SDImageCacheConfigExpireTypeModificationDate:
            cacheContentDateKey = NSURLContentModificationDateKey; break;
        // ...
    }
    
    NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.config.maxDiskAge];
    
    // 遍历文件，删除过期文件
    for (NSURL *fileURL in fileEnumerator) {
        NSDate *modifiedDate = resourceValues[cacheContentDateKey];
        if ([[modifiedDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
            [urlsToDelete addObject:fileURL];
        }
    }
    
    // 如果超过最大大小，按时间排序删除最老的
    if (currentCacheSize > maxDiskSize) {
        NSArray *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                         usingComparator:^NSComparisonResult(id obj1, id obj2) {
            return [obj1[cacheContentDateKey] compare:obj2[cacheContentDateKey]];
        }];
        // 删除最老的文件直到满足大小限制
    }
}
```

### 3.3 图片解码时机

**为什么需要后台解码？**

Core Animation 在主线程渲染图片时会进行解码，这会：
1. 阻塞主线程
2. 导致列表滑动卡顿
3. UI 不响应

**解码入口 - SDImageCoderHelper：**

```objc
// UIImage+ForceDecode.m:42-47
+ (UIImage *)sd_decodedImageWithImage:(UIImage *)image {
    if (!image) return nil;
    return [SDImageCoderHelper decodedImageWithImage:image];
}
```

**解码时机选择：**

```objc
// SDImageCache.m:672-673 - 磁盘读取后解码
diskImage = [self diskImageForKey:key data:diskData options:options context:context];

// SDWebImageDownloaderOperation.m:593-596 - 下载完成后解码
image = SDImageLoaderDecodeProgressiveImageData(imageData, self.request.URL, YES, self, options, context);
```

### 3.4 避免 OOM 的设计思路

**1. 内存成本计算：**

```objc
// UIImage+MemoryCacheCost.m - 计算图片内存占用
- (NSUInteger)sd_memoryCost {
    CGImageRef cgImage = self.CGImage;
    if (!cgImage) return 0;
    
    size_t bytesPerRow = CGImageGetBytesPerRow(cgImage);
    size_t height = CGImageGetHeight(cgImage);
    return bytesPerRow * height;
}
```

**2. 内存限制配置：**

```objc
// SDImageCacheConfig.h:108-114
@property (assign, nonatomic) NSUInteger maxMemoryCost;   // 最大内存成本
@property (assign, nonatomic) NSUInteger maxMemoryCount;  // 最大图片数量
```

**3. 后台解码大图降采样：**

```objc
// SDImageCoderHelper 中的降采样解码
+ (UIImage *)decodedAndScaledDownImageWithImage:(UIImage *)image limitBytes:(NSUInteger)bytes {
    // 根据内存限制自动降采样
    // 避免解码超大图导致 OOM
}
```

---

## 4. iOS 底层知识点

### 4.1 RunLoop 应用

**SDWebImage 中没有直接使用 RunLoop，但有相关设计模式：**

**1. 常驻线程模式（可学习）：**

虽然 SDWebImage 使用 GCD 队列而非 RunLoop，但其设计思想是：
- 串行 IO 队列保证顺序执行
- 避免频繁创建销毁线程

**2. 异步回调到主线程：**

```objc
// SDCallbackQueue.m - 回调队列抽象
- (void)async:(dispatch_block_t)block {
    dispatch_async(self.queue, block);
}

// SDImageCache.m:702-710 - 回调到主线程
[(queue ?: SDCallbackQueue.mainQueue) async:^{
    doneBlock(diskImage, diskData, SDImageCacheTypeDisk);
}];
```

### 4.2 Runtime 应用

#### 4.2.1 关联对象（Associated Objects）

**添加属性的完整实现：**

```objc
// UIImage+ForceDecode.m:16-40
- (BOOL)sd_isDecoded {
    NSNumber *value = objc_getAssociatedObject(self, @selector(sd_isDecoded));
    if (value != nil) {
        return value.boolValue;
    }
    // 默认值计算逻辑
    CGImageRef cgImage = self.CGImage;
    if (cgImage) {
        CFStringRef uttype = CGImageGetUTType(self.CGImage);
        return uttype ? NO : YES;
    }
    return NO;
}

- (void)setSd_isDecoded:(BOOL)sd_isDecoded {
    objc_setAssociatedObject(self, @selector(sd_isDecoded), 
                            @(sd_isDecoded), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

**UIView 扩展属性：**

```objc
// UIView+WebCache.m:21-27
- (NSURL *)sd_imageURL {
    return objc_getAssociatedObject(self, @selector(sd_imageURL));
}

- (void)setSd_imageURL:(NSURL *)sd_imageURL {
    objc_setAssociatedObject(self, @selector(sd_imageURL), 
                            sd_imageURL, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

#### 4.2.2 方法交换（Method Swizzling）

**SDWebImage 没有使用方法交换，这是设计选择：**
- 侵入性低：只通过 Category 添加方法
- 明确性：调用者显式调用 sd_ 前缀方法

### 4.3 引用循环打破

#### 4.3.1 weak 代理模式 - SDWeakProxy

```objc
// SDWeakProxy.h:13-20
@interface SDWeakProxy : NSProxy
@property (nonatomic, weak, readonly, nullable) id target;
- (instancetype)initWithTarget:(id)target;
+ (instancetype)proxyWithTarget:(id)target;
@end

// SDWeakProxy.m:22-24
- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;  // 消息转发给弱引用的 target
}
```

**使用场景 - 打破 NSTimer 循环引用：**

```objc
// 典型使用模式
_timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                          target:[SDWeakProxy proxyWithTarget:self]
                                        selector:@selector(handleTimer)
                                        userInfo:nil
                                         repeats:YES];
```

#### 4.3.2 weakify/strongify 宏

```objc
// SDInternalMacros.h:82-106
#ifndef weakify
#define weakify(...) \
sd_keywordify \
metamacro_foreach_cxt(sd_weakify_,, __weak, __VA_ARGS__)
#endif

#ifndef strongify
#define strongify(...) \
sd_keywordify \
_Pragma("clang diagnostic push") \
_Pragma("clang diagnostic ignored \"-Wshadow\"") \
metamacro_foreach(sd_strongify_,, __VA_ARGS__) \
_Pragma("clang diagnostic pop")
#endif

// 展开后
#define sd_weakify_(INDEX, CONTEXT, VAR) \
CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);

#define sd_strongify_(INDEX, VAR) \
__strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
```

**实际使用：**

```objc
// SDWebImageManager.m:305-306
@weakify(operation);
operation.cacheOperation = [imageCache queryImageForKey:key ... completion:^(...) {
    @strongify(operation);
    if (!operation || operation.isCancelled) { ... }
}];
```

### 4.4 GCD 高级用法

#### 4.4.1 dispatch_once - 单例

```objc
// SDWebImageManager.m:66-73
+ (instancetype)sharedManager {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}
```

#### 4.4.2 dispatch_barrier - 并发队列写操作

```objc
// SDWebImageDownloaderOperation.m:619-625
// iOS 13+ 使用 barrier block
if (@available(iOS 13, tvOS 13, macOS 10.15, watchOS 6, *)) {
    [self.coderQueue addBarrierBlock:doneBlock];
} else {
    // 串行队列，等效于 barrier
    [self.coderQueue addOperationWithBlock:doneBlock];
}
```

#### 4.4.3 dispatch_sync 死锁规避

```objc
// SDImageCache.m:682-688 - 同步查询磁盘缓存
if (shouldQueryDiskSync) {
    __block NSData* diskData;
    __block UIImage* diskImage;
    dispatch_sync(self.ioQueue, ^{  // 在 ioQueue 上同步执行
        diskData = queryDiskDataBlock();
        diskImage = queryDiskImageBlock(diskData);
    });
}
```

**为什么不会死锁？**
- `ioQueue` 是串行队列
- 如果当前已在 `ioQueue`，说明是同步调用，不需要再 dispatch_sync
- 调用者保证不在 `ioQueue` 中调用同步方法

### 4.5 图片渲染与字节对齐

**CGImage 内存布局：**

```
┌─────────────────────────────────────────┐
│              CGImage 内存布局              │
├─────────────────────────────────────────┤
│  width: 图片宽度（像素）                    │
│  height: 图片高度（像素）                   │
│  bitsPerComponent: 每个颜色分量的位数        │
│  bitsPerPixel: 每个像素的总位数             │
│  bytesPerRow: 每行的字节数（有对齐！）       │
│  colorSpace: 颜色空间                      │
│  bitmapInfo: 位图信息（Alpha位置、字节序）   │
└─────────────────────────────────────────┘
```

**字节对齐的意义：**

```objc
// 内存对齐影响性能
// 未对齐：CPU 需要多次内存访问
// 对齐：CPU 单次访问效率更高

// bytesPerRow 计算（64字节对齐）
size_t bytesPerRow = (width * bitsPerPixel / 8 + 63) & ~63;
```

---

## 5. 网络层与任务管理

### 5.1 SDWebImageDownloader 并发设计

**URLSession 与 OperationQueue 协作：**

```objc
// SDWebImageDownloader.m:154-156
_session = [NSURLSession sessionWithConfiguration:sessionConfiguration
                                          delegate:self
                                     delegateQueue:nil];  // nil 让系统创建串行队列
```

**为什么 delegateQueue 传 nil？**
1. 系统自动创建串行队列处理代理回调
2. 避免多线程竞争问题
3. 代理方法顺序执行

### 5.2 请求合并机制

**同一 URL 复用 Operation：**

```objc
// SDWebImageDownloader.m:236-280
- (SDWebImageDownloadToken *)downloadImageWithURL:... {
    SD_LOCK(_operationsLock);
    NSOperation<SDWebImageDownloaderOperation> *operation = [self.URLOperations objectForKey:url];
    
    // 检查是否可以复用
    BOOL shouldNotReuseOperation;
    if (operation) {
        @synchronized (operation) {
            shouldNotReuseOperation = operation.isFinished || operation.isCancelled || 
                                      SDWebImageDownloaderOperationGetCompleted(operation);
        }
    } else {
        shouldNotReuseOperation = YES;
    }
    
    if (shouldNotReuseOperation) {
        // 创建新 operation
        operation = [self createDownloaderOperationWithUrl:url options:options context:context];
        [self.URLOperations setObject:operation forKey:url];
        [self.downloadQueue addOperation:operation];
    } else {
        // 复用现有 operation，只添加回调
        @synchronized (operation) {
            downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock 
                                                                    completed:completedBlock 
                                                                decodeOptions:decodeOptions];
        }
    }
    SD_UNLOCK(_operationsLock);
}
```

### 5.3 取消机制

**多层取消设计：**

```objc
// Token 层取消
// SDWebImageDownloadToken.m:601-610
- (void)cancel {
    @synchronized (self) {
        if (self.isCancelled) return;
        self.cancelled = YES;
        [self.downloadOperation cancel:self.downloadOperationCancelToken];
    }
}

// Operation 层取消
// SDWebImageDownloaderOperation.m:146-167
- (BOOL)cancel:(id)token {
    BOOL shouldCancel = NO;
    @synchronized (self) {
        if (tokens.count == 1 && [tokens indexOfObjectIdenticalTo:token] != NSNotFound) {
            shouldCancel = YES;  // 最后一个 token 取消，取消整个 operation
        }
    }
    if (shouldCancel) {
        [self cancel];
    } else {
        // 只移除该 token 的回调
        [self.callbackTokens removeObjectIdenticalTo:token];
        // 回调取消完成
        [self callCompletionBlockWithToken:token ... error:cancelError ...];
    }
}
```

### 5.4 Cookie 和 Header 管理

**自定义 Header：**

```objc
// SDWebImageDownloader.m:181-198
- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field {
    SD_LOCK(_HTTPHeadersLock);
    [self.HTTPHeaders setValue:value forKey:field];
    SD_UNLOCK(_HTTPHeadersLock);
}

// 创建请求时应用
// SDWebImageDownloader.m:316-317
SD_LOCK(_HTTPHeadersLock);
mutableRequest.allHTTPHeaderFields = self.HTTPHeaders;
SD_UNLOCK(_HTTPHeadersLock);
```

**Cookie 处理：**

```objc
// SDWebImageDownloader.m:313
mutableRequest.HTTPShouldHandleCookies = SD_OPTIONS_CONTAINS(options, SDWebImageDownloaderHandleCookies);
```

---

## 6. 设计模式在 SDWebImage 中的真实应用

### 6.1 单例模式

**正确实现方式：**

```objc
// SDImageCache.m:75-82
+ (instancetype)sharedImageCache {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}
```

**为什么不滥用单例？**

1. **全局状态**：难以测试
2. **生命周期**：无法控制创建和销毁时机
3. **依赖隐藏**：调用者不知道依赖了什么

**SDWebImage 的设计：单例 + 可实例化**

```objc
// 提供单例便利访问
+ (instancetype)sharedManager;

// 同时支持自定义实例化
- (instancetype)initWithCache:(id<SDImageCache>)cache loader:(id<SDImageLoader>)loader;
```

### 6.2 装饰者模式 / 分类扩展

**通过 Category 扩展功能：**

```objc
// UIImageView+WebCache.h - 扩展 UIImageView
@interface UIImageView (WebCache)
- (void)sd_setImageWithURL:(NSURL *)url;
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder;
// ... 多个便捷方法
@end

// UIButton+WebCache.h - 扩展 UIButton
@interface UIButton (WebCache)
- (void)sd_setImageWithURL:(NSURL *)url forState:(UIControlState)state;
@end
```

### 6.3 代理模式

**SDWebImageManagerDelegate：**

```objc
// SDWebImageManager.h:53-76
@protocol SDWebImageManagerDelegate <NSObject>
@optional
- (BOOL)imageManager:(SDWebImageManager *)imageManager 
 shouldDownloadImageForURL:(NSURL *)imageURL;  // 控制是否下载

- (BOOL)imageManager:(SDWebImageManager *)imageManager 
 shouldBlockFailedURL:(NSURL *)imageURL 
            withError:(NSError *)error;  // 控制失败 URL 黑名单
@end
```

### 6.4 策略模式

**通过协议定义策略：**

```objc
// SDImageCoder.h:136-202 - 解码策略
@protocol SDImageCoder <NSObject>
- (BOOL)canDecodeFromData:(NSData *)data;
- (UIImage *)decodedImageWithData:(NSData *)data options:(SDImageCoderOptions *)options;
- (BOOL)canEncodeToFormat:(SDImageFormat)format;
- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format options:(SDImageCoderOptions *)options;
@end
```

**多种策略实现：**

```objc
// SDImageIOCoder - JPEG/PNG/HEIC
// SDImageGIFCoder - GIF
// SDImageAPNGCoder - APNG
// SDImageHEICCoder - HEIF
```

### 6.5 责任链模式

**SDImageCodersManager - 解码器责任链：**

```objc
// SDImageCodersManager.h:32
@interface SDImageCodersManager : NSObject <SDImageCoder>

// SDImageCodersManager.m:101-115
- (UIImage *)decodedImageWithData:(NSData *)data options:(SDImageCoderOptions *)options {
    UIImage *image;
    NSArray<id<SDImageCoder>> *coders = self.coders;
    // 倒序遍历，后添加的优先级高
    for (id<SDImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canDecodeFromData:data]) {
            image = [coder decodedImageWithData:data options:options];
            break;  // 找到能处理的就停止
        }
    }
    return image;
}
```

**添加自定义解码器：**

```objc
[[SDImageCodersManager sharedManager] addCoder:[MyCustomCoder new]];
// 自定义解码器会被优先使用
```

---

## 7. 性能优化实战

### 7.1 卡顿优化

**1. 后台解码：**

```objc
// SDWebImageDownloaderOperation.m:569-596
[self.coderQueue addOperationWithBlock:^{
    // 在后台队列解码，不阻塞主线程
    UIImage *image = SDImageLoaderDecodeImageData(imageData, url, options, context);
}];
```

**2. 主线程安全调用：**

```objc
// UIView+WebCache.m:97
#define dispatch_main_async_safe(block)\
    if (dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL) == dispatch_queue_get_label(dispatch_get_main_queue())) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
```

### 7.2 内存峰值优化

**1. 渐进式解码：**

```objc
// SDWebImageDownloaderOperation.m:466-493
if (supportProgressive && !finished) {
    [self.coderQueue addOperationWithBlock:^{
        // 渐进式解码，先显示低质量图
        UIImage *image = SDImageLoaderDecodeProgressiveImageData(imageData, url, NO, self, options, context);
        [self callCompletionBlocksWithImage:image imageData:nil error:nil finished:NO];
    }];
}
```

**2. 图片降采样：**

```objc
// 通过 thumbnailPixelSize 选项控制
context[SDWebImageContextImageThumbnailPixelSize] = @(CGSizeMake(200, 200));
// 避免加载超大图
```

### 7.3 I/O 优化

**1. 异步磁盘写入：**

```objc
// SDImageCache.m:284-293
dispatch_async(self.ioQueue, ^{
    [self _storeImageDataToDisk:data forKey:key];
});
```

**2. 批量清理：**

```objc
// SDDiskCache.m:165-199
NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtURL:...];
// 一次性遍历，批量处理
```

### 7.4 高并发列表滑动优化

**1. 取消旧请求：**

```objc
// UIView+WebCache.m:72
[self sd_cancelImageLoadOperationWithKey:validOperationKey];
```

**2. 复用 Operation：**

```objc
// SDWebImageDownloader.m:247-256
// 同一 URL 的多个请求复用同一个 Operation
```

---

## 8. ObjC 高级编码规范

### 8.1 宏定义最佳实践

```objc
// SDInternalMacros.h

// 1. 编译时检查
#ifndef SD_LOCK_DECLARE
#define SD_LOCK_DECLARE(lock) os_unfair_lock lock
#endif

// 2. 类型安全
#define SD_OPTIONS_CONTAINS(options, value) (((options) & (value)) == (value))

// 3. 字符串化
#define SD_CSTRING(str) #str
#define SD_NSSTRING(str) @(SD_CSTRING(str))

// 4. 条件编译
#if SD_USE_OS_UNFAIR_LOCK
    // iOS 10+ 实现
#else
    // 低版本回退
#endif
```

### 8.2 const/static 用法

```objc
// SDWebImageDownloader.m:18-21
// 文件内常量
NSNotificationName const SDWebImageDownloadStartNotification = @"SDWebImageDownloadStartNotification";
NSNotificationName const SDWebImageDownloadStopNotification = @"SDWebImageDownloadStopNotification";

// 静态变量
static void * SDWebImageDownloaderContext = &SDWebImageDownloaderContext;
static NSString * _defaultDiskCacheDirectory;  // SDImageCache.m:57
```

### 8.3 泛型与 instancetype

```objc
// 泛型声明
// SDMemoryCache.h:74
@interface SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType> <SDMemoryCache>

// instancetype 返回类型
// SDImageCache.h:138
- (instancetype)initWithNamespace:(NSString *)ns;

// 协议中的泛型
// SDImageCoder.h:16
typedef NSDictionary<SDImageCoderOption, id> SDImageCoderOptions;
```

### 8.4 NS_OPTIONS / NS_ENUM

```objc
// SDWebImageDownloader.h:20-96
typedef NS_OPTIONS(NSUInteger, SDWebImageDownloaderOptions) {
    SDWebImageDownloaderLowPriority = 1 << 0,
    SDWebImageDownloaderProgressiveLoad = 1 << 1,
    SDWebImageDownloaderUseNSURLCache = 1 << 2,
    // ... 可组合的选项用 NS_OPTIONS
};

// SDImageCacheConfig.h:13-30
typedef NS_ENUM(NSUInteger, SDImageCacheConfigExpireType) {
    SDImageCacheConfigExpireTypeAccessDate,
    SDImageCacheConfigExpireTypeModificationDate,
    // ... 互斥的选项用 NS_ENUM
};
```

### 8.5 线程安全原子性写法

```objc
// 1. 使用锁保护
SD_LOCK_DECLARE(_lock);
SD_LOCK_INIT(_lock);

- (void)setValue:(id)value {
    SD_LOCK(_lock);
    _value = value;
    SD_UNLOCK(_lock);
}

// 2. 使用 @synchronized（需要递归锁时）
- (void)cancel {
    @synchronized (self) {
        if (_cancelled) return;
        _cancelled = YES;
    }
}

// 3. 使用 atomic 属性（简单场景）
@property (atomic, assign) BOOL cancelled;

// 4. 结合 getter/setter 自定义
- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:@"isFinished"];  // KVO 通知
    _finished = finished;
    [self didChangeValueForKey:@"isFinished"];
}
```

---

## 9. 源码精读路线图

### 阶段一：入口与流程（1-2 周）

**阅读顺序：**

1. **UIImageView+WebCache.h/m** - 入口 API
2. **UIView+WebCache.h/m** - 通用实现
3. **UIView+WebCacheOperation.h/m** - 操作管理

**学习重点：**
- API 设计思路
- Category 如何管理状态（关联对象）
- 取消机制如何工作

### 阶段二：核心管理器（2-3 周）

**阅读顺序：**

1. **SDWebImageManager.h/m** - 核心协调
2. **SDWebImageCombinedOperation** - 组合操作
3. **SDWebImageDefine.h** - 选项定义

**学习重点：**
- 缓存查询流程
- 下载触发条件
- 错误处理机制

### 阶段三：缓存系统（2-3 周）

**阅读顺序：**

1. **SDImageCache.h/m** - 缓存门面
2. **SDMemoryCache.h/m** - 内存缓存
3. **SDDiskCache.h/m** - 磁盘缓存
4. **SDImageCacheConfig.h/m** - 配置类

**学习重点：**
- NSCache 工作原理
- 文件系统操作
- 过期策略实现

### 阶段四：网络下载（2-3 周）

**阅读顺序：**

1. **SDWebImageDownloader.h/m** - 下载管理
2. **SDWebImageDownloaderOperation.h/m** - 下载操作
3. **SDWebImageDownloaderConfig.h/m** - 配置类

**学习重点：**
- NSURLSession 代理模式
- NSOperation 生命周期
- 并发控制实现

### 阶段五：解码系统（1-2 周）

**阅读顺序：**

1. **SDImageCoder.h** - 协议定义
2. **SDImageCodersManager.h/m** - 责任链
3. **SDImageIOCoder.h/m** - 基础解码
4. **SDImageGIFCoder.h/m** - 动图解码

**学习重点：**
- Image/IO 框架使用
- 渐进式解码原理
- 责任链模式实现

### 阶段六：内部工具（1 周）

**阅读顺序：**

1. **SDInternalMacros.h** - 宏定义
2. **SDWeakProxy.h/m** - 弱引用代理
3. **SDAsyncBlockOperation.h/m** - 异步操作

---

## 10. 高频面试题

### 10.1 底层原理题

**Q1: SDWebImage 如何保证线程安全？**

**答案：**
1. 使用 `os_unfair_lock`（iOS 10+）或 `OSSpinLock`（低版本）保护临界区
2. 磁盘 I/O 使用串行 GCD 队列保证顺序执行
3. `@synchronized` 用于需要递归锁的场景
4. 代理回调使用系统创建的串行队列

```objc
// SDWebImageManager.m:30-33
@interface SDWebImageManager () {
    SD_LOCK_DECLARE(_failedURLsLock);
    SD_LOCK_DECLARE(_runningOperationsLock);
}
```

**Q2: 为什么图片需要解码？解码时机是什么？**

**答案：**

**为什么需要解码：**
- 图片文件（JPEG/PNG）是压缩格式
- GPU 渲染需要原始位图数据（RGBA）
- Core Animation 在渲染时解码会阻塞主线程

**解码时机：**
1. 磁盘缓存读取后（后台线程）
2. 网络下载完成后（后台线程）
3. 渐进式下载过程中（后台线程）

```objc
// SDImageCache.m:672
diskImage = [self diskImageForKey:key data:diskData options:options context:context];
```

### 10.2 架构题

**Q3: SDWebImage 的架构设计有哪些亮点？**

**答案：**

1. **分层架构**：Category -> Manager -> Cache/Loader -> Decoder
2. **依赖注入**：通过 init 方法注入 Cache 和 Loader
3. **面向协议编程**：`SDImageCache`、`SDImageLoader`、`SDImageCoder` 都是协议
4. **责任链模式**：解码器可以动态添加
5. **配置分离**：Config 类管理可配置项

**Q4: 如何实现自定义缓存策略？**

**答案：**

```objc
// 1. 实现 SDMemoryCache 协议
@interface MyMemoryCache : NSObject <SDMemoryCache>
@end

// 2. 配置到 CacheConfig
SDImageCacheConfig *config = [SDImageCacheConfig new];
config.memoryCacheClass = [MyMemoryCache class];

// 3. 创建缓存实例
SDImageCache *cache = [[SDImageCache alloc] initWithNamespace:@"my" config:config];
```

### 10.3 多线程题

**Q5: SDWebImage 如何控制并发下载数量？**

**答案：**

```objc
// SDWebImageDownloader.m:118
_downloadQueue.maxConcurrentOperationCount = _config.maxConcurrentDownloads; // 默认 6

// 动态修改
SDWebImageDownloaderConfig.defaultDownloaderConfig.maxConcurrentDownloads = 10;
```

**Q6: 如何实现 LIFO（后进先出）下载顺序？**

**答案：**

通过 NSOperation 的依赖关系实现：

```objc
// SDWebImageDownloader.m:403-410
if (self.config.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
    for (NSOperation *pendingOperation in self.downloadQueue.operations) {
        [pendingOperation addDependency:operation];
    }
}
```

### 10.4 内存题

**Q7: SDMemoryCache 为什么使用双层缓存（强引用+弱引用）？**

**答案：**

1. **强引用缓存（NSCache）**：正常缓存，会被内存警告清理
2. **弱引用缓存（NSMapTable）**：内存警告后可恢复

```objc
// SDMemoryCache.m:97-117
- (id)objectForKey:(id)key {
    id obj = [super objectForKey:key];  // 先查强引用
    if (key && !obj) {
        obj = [self.weakCache objectForKey:key];  // 再查弱引用
        if (obj) {
            [super setObject:obj forKey:key cost:cost];  // 回写强引用
        }
    }
    return obj;
}
```

**Q8: 如何避免图片加载导致的 OOM？**

**答案：**

1. **设置内存限制**：
```objc
SDImageCacheConfig *config = [SDImageCacheConfig new];
config.maxMemoryCost = 100 * 1024 * 1024; // 100MB
```

2. **使用降采样**：
```objc
context[SDWebImageContextImageThumbnailPixelSize] = @(CGSizeMake(200, 200));
```

3. **启用弱引用缓存**：
```objc
config.shouldUseWeakMemoryCache = YES;
```

### 10.5 RunLoop/Runtime 题

**Q9: SDWeakProxy 如何打破循环引用？**

**答案：**

```objc
// SDWeakProxy.m:22-24
- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;  // 消息转发给弱引用的 target
}

// 使用场景
_timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                          target:[SDWeakProxy proxyWithTarget:self]  // 代理作为 target
                                        selector:@selector(handleTimer)
                                        userInfo:nil
                                         repeats:YES];
// self -> timer -> proxy -> (weak)self，打破循环
```

**Q10: weakify/strongify 宏的工作原理？**

**答案：**

```objc
// 展开前
@weakify(self);
[self doSomething:^{
    @strongify(self);
    [self method];
}];

// 展开后
__weak typeof(self) self_weak_ = self;
[self doSomething:^{
    __strong typeof(self) self = self_weak_;
    [self method];
}];
```

**为什么需要 strongify？**
- Block 捕获 weak 变量
- 执行时需要 strong 引用防止提前释放
- strongify 在 Block 开始时创建 strong 引用

---

## 附录：关键源码文件索引

| 文件 | 核心功能 | 重要程度 |
|------|----------|----------|
| `SDWebImageManager.m` | 核心协调逻辑 | ⭐⭐⭐⭐⭐ |
| `SDImageCache.m` | 缓存实现 | ⭐⭐⭐⭐⭐ |
| `SDWebImageDownloader.m` | 下载管理 | ⭐⭐⭐⭐⭐ |
| `SDWebImageDownloaderOperation.m` | 下载操作 | ⭐⭐⭐⭐ |
| `SDMemoryCache.m` | 内存缓存 | ⭐⭐⭐⭐ |
| `SDDiskCache.m` | 磁盘缓存 | ⭐⭐⭐⭐ |
| `SDImageCodersManager.m` | 解码管理 | ⭐⭐⭐ |
| `UIView+WebCache.m` | UI 扩展 | ⭐⭐⭐ |
| `SDInternalMacros.h` | 内部宏 | ⭐⭐⭐ |
| `SDWeakProxy.m` | 弱引用代理 | ⭐⭐⭐ |

---

> 文档版本：1.0
> 基于 SDWebImage 5.x 源码分析
> 适用于 iOS Objective-C 高级开发者

