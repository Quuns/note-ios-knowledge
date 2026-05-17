
---
# iOS ObjC NSNotificationCenter 底层原理+面试考点

## 一、基础概念速记

### NSNotificationCenter（通知中心）
**全称**：通知中心  
**作用**：**在对象之间进行一对多的通信，发送者和接收者之间不需要直接引用，实现松耦合的消息传递。**

**核心 API**
```objc
// 获取默认通知中心
[NSNotificationCenter defaultCenter];

// 添加观察者（selector 方式）
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(handleNotification:)
                                             name:@"NotificationName"
                                           object:nil];

// 添加观察者（block 方式，iOS 4.0+）
id observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"NotificationName"
                                                                object:nil
                                                                 queue:[NSOperationQueue mainQueue]
                                                            usingBlock:^(NSNotification *note) {
    // handle notification
}];

// 发送通知
[[NSNotificationCenter defaultCenter] postNotificationName:@"NotificationName"
                                                    object:nil
                                                  userInfo:@{@"key": @"value"}];

// 移除观察者
[[NSNotificationCenter defaultCenter] removeObserver:self];
[[NSNotificationCenter defaultCenter] removeObserver:self name:@"NotificationName" object:nil];
```

**特点**：
- 一对多通信：一个通知可以被多个观察者接收
- 发送者和接收者松耦合，不需要互相持有引用
- 同步发送：`postNotification` 会阻塞当前线程，直到所有观察者的回调执行完毕
- iOS 9.0+ 不需要手动 `removeObserver`，系统会自动移除（仅限 selector 方式，block 方式仍需手动移除）

---

## 二、底层原理（面试必考！）

### 1. 核心数据结构：哈希表 + 链表

NSNotificationCenter 内部维护一个**名为 `_namedTable` 的哈希表**，key 为通知名称，value 为观察者链表。

```
_namedTable (NSMapTable / CFMutableDictionaryRef)
┌─────────────────────────────────────────────────────┐
│  key: @"NotificationA"  →  ObservedObjectNode 链表   │
│                           [obs1, sel, obj] →         │
│                           [obs2, sel, obj] →         │
│                           [obs3, block, queue]       │
│                                                      │
│  key: @"NotificationB"  →  ObservedObjectNode 链表   │
│                           [obs4, sel, obj]           │
│                                                      │
│  key: nil (通配)        →  ObservedObjectNode 链表   │
│      (未命名通知，          [obs5, sel, obj]          │
│       存于_unnamedTable)                              │
└─────────────────────────────────────────────────────┘
```

**源码层面拆解**（基于 GNUStep / Swift 开源 Foundation）：

```objc
// 观察者节点结构（简化版）
typedef struct ObservedObjectNode {
    id observer;                    // 观察者对象（__weak 引用或不安全引用）
    SEL selector;                   // 回调方法
    id block;                       // block 回调（addObserverForName 方式）
    NSOperationQueue *queue;        // block 回调的执行队列
    id object;                      // 过滤的发送者（nil 表示不过滤）
    NSString *name;                 // 通知名称（nil 表示监听所有）
    struct ObservedObjectNode *next; // 链表指针
} ObservedObjectNode;
```

### 2. 添加观察者流程 (`addObserver:selector:name:object:`)

```
1. 根据 name 生成 key（name 为 nil 时 key 对应通配符，存入 _unnamedTable 或单独的通配槽位）
2. 创建 ObservedObjectNode，填充 observer、selector、object、name
3. 在哈希表中查找对应 key 的链表，将新节点插入链表头部
4. 对 observer 持有 __weak 引用（防止循环引用）
5. 如果 observer 为 nil，不添加
```

### 3. 发送通知流程 (`postNotificationName:object:userInfo:`)

这是面试最核心的考点，**务必记住执行顺序**：

```
1. 创建 NSNotification 实例（name + object + userInfo）
2. 在 _namedTable 中查找 name 对应的观察者链表
3. 同时查找 _unnamedTable（通配表，即 name=nil 注册的观察者）
4. 合并两个链表中匹配的观察者：
   - object 参数为 nil → 该 name 下所有观察者都匹配
   - object 不为 nil → 只匹配注册时 object 相同或为 nil 的观察者
5. 遍历匹配的观察者链表，逐个执行回调
6. 回调执行完毕后返回，post 方法结束
```

**关键细节：遍历时拷贝链表，防止回调中增删观察者导致遍历异常。**

```objc
// 伪代码，核心流程
- (void)postNotificationName:(NSString *)name object:(id)object userInfo:(NSDictionary *)userInfo {
    NSNotification *notification = [NSNotification notificationWithName:name
                                                                 object:object
                                                               userInfo:userInfo];
    
    // 1. 从 _namedTable 取出 name 对应的链表
    // 2. 从 _unnamedTable 取出通配链表（注册时 name=nil）
    // 3. 合并并过滤（按 object 匹配）
    // 4. 拷贝链表快照（防止遍历时修改）
    NSArray *observers = [self observersForNotification:notification];
    
    for (ObservedObjectNode *node in observers) {
        if (node.block) {
            // block 方式：在指定 queue 执行
            [node.queue addOperationWithBlock:^{
                node.block(notification);
            }];
        } else {
            // selector 方式：同步调用
            [node.observer performSelector:node.selector withObject:notification];
        }
    }
}
```

### 4. 发送通知是同步的！

**这条面试极高频率考到。** 

`postNotification` 是同步方法，会**阻塞当前线程**，直到所有观察者回调执行完毕才返回。无论观察者注册在哪个线程，回调都在**发送通知的线程**上执行。

```objc
// 验证代码
NSLog(@"1 - before post");
[[NSNotificationCenter defaultCenter] postNotificationName:@"Test" object:nil];
NSLog(@"3 - after post");

// 回调中：
- (void)handleTest:(NSNotification *)note {
    sleep(3);  // 模拟耗时操作
    NSLog(@"2 - handler done");
}
// 输出顺序：1 → （等3秒）→ 2 → 3
```

**结论**：不要在 `postNotification` 中做耗时操作，否则会阻塞发送线程。如需异步，应该在回调中手动 dispatch 到后台队列。

### 5. block 方式的线程行为差异

```objc
// selector 方式：回调在 post 的线程同步执行
[center addObserver:self selector:@selector(handler:) name:@"Test" object:nil];

// block 方式：回调在指定的 queue 上执行（可异步）
[center addObserverForName:@"Test"
                    object:nil
                     queue:[NSOperationQueue mainQueue]  // 指定主队列
                usingBlock:^(NSNotification *note) {
    // 这个 block 在 mainQueue 上执行
}];
```

**block 方式的回调不阻塞 post 线程**，由指定的 queue 来调度。队列传 `nil` 时行为和 selector 方式一样——在 post 线程同步执行。

### 6. NSNotificationQueue —— 异步发送

```objc
// 使用通知队列实现异步合并发送
NSNotification *note = [NSNotification notificationWithName:@"Test" object:nil];
[[NSNotificationQueue defaultQueue] enqueueNotification:note
                                           postingStyle:NSPostWhenIdle]; // runloop idle 时发送
```

**三种发送时机（NSPostingStyle）**：
| 枚举值 | 说明 |
|--------|------|
| `NSPostNow` | 立即发送，等同于 `postNotification` |
| `NSPostASAP` | 当前 runloop 迭代结束时发送 |
| `NSPostWhenIdle` | runloop 空闲时发送 |

**合并策略（NSNotificationCoalescing）**：
| 枚举值 | 说明 |
|--------|------|
| `NSNotificationNoCoalescing` | 不合并 |
| `NSNotificationCoalescingOnName` | 按名称合并，同名通知只保留最后一个 |
| `NSNotificationCoalescingOnSender` | 按发送者合并 |

**底层原理**：NSNotificationQueue 依赖 runloop。enqueue 后并非立即 post，而是在指定时机由 runloop observer 回调触发实际的 `postNotification`。通知队列也用 runloop mode 过滤，只在匹配的 mode 下投递。

### 7. 内存管理 —— iOS 9 前后的区别

**iOS 9.0 之前**：
- 通知中心对 observer 是 **`__unsafe_unretained`** 引用（或 assign）
- 观察者释放时必须手动 `removeObserver`，否则通知中心持野指针，下一次发通知直接 **EXC_BAD_ACCESS 崩溃**

**iOS 9.0 之后**：
- selector 方式（`addObserver:selector:name:object:`）：通知中心对 observer 使用 **`__weak`** 引用。系统在 observer 释放时自动调用 `removeObserver`，**不需要手动移除**。
- block 方式（`addObserverForName:object:queue:usingBlock:`）：返回的匿名 observer 对象**仍需手动移除**，因为 block 本身被通知中心强持有，不会自动释放。

```objc
// iOS 9+ selector 方式：不需要 removeObserver
// 系统在 dealloc 时自动清理

// block 方式：必须手动移除！
// 返回值需要保存，在 dealloc 中移除
@interface MyClass ()
@property (nonatomic, strong) id<NSObject> notificationToken;
@end

- (instancetype)init {
    self = [super init];
    if (self) {
        __weak typeof(self) weakSelf = self;
        self.notificationToken = [[NSNotificationCenter defaultCenter]
            addObserverForName:@"Test"
                        object:nil
                         queue:nil
                    usingBlock:^(NSNotification *note) {
            // 使用 weakSelf 避免循环引用
            [weakSelf doSomething];
        }];
    }
    return self;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:_notificationToken];
}
```

### 8. NSNotification 本身

```objc
@interface NSNotification : NSObject <NSCopying, NSCoding>
@property (readonly, copy) NSString *name;     // 通知名称
@property (readonly, retain) id object;         // 发送者（通常传 self）
@property (readonly, copy) NSDictionary *userInfo; // 传递的数据
@end
```

- `object` 有双重作用：1）标识发送者，在回调中区分来源；2）作为注册时的过滤条件
- `userInfo` 传递额外数据，线程安全由使用者负责
- NSNotification 遵循 `NSCopying`，可以安全复制

---

## 三、面试高频考点

### 考点 1：postNotification 是同步还是异步？

**同步。** `postNotification` 会阻塞当前线程，逐一执行所有观察者的回调，全部执行完才返回。

追问：如果观察者在子线程注册，post 在主线程，回调在哪个线程？
**答：在主线程。** 回调的执行线程 = post 所在的线程，与注册线程无关。

### 考点 2：如何实现异步通知？

- 使用 `addObserverForName:object:queue:usingBlock:` 指定异步 queue
- 使用 `NSNotificationQueue` 的 `NSPostWhenIdle` 或 `NSPostASAP` 模式
- 在回调中手动 dispatch 到后台队列

### 考点 3：iOS 9 之后还需要 removeObserver 吗？

Selector 方式不需要，系统自动清理。Block 方式仍需要手动移除。

追问：为什么 block 方式不能自动移除？
**答**：`addObserverForName:object:queue:usingBlock:` 返回的是一个新的匿名 observer 对象，通知中心持有的是这个匿名对象而不是调用者 `self`。即使 `self` 被释放，这个匿名对象依然存活，block 仍然可被触发。只有手动 `removeObserver` 才能断开。

### 考点 4：通知中心的观察者链表数据结构是怎样的？

哈希表 + 单向链表。name（或 nil）作为 key，映射到该名称下的所有观察者节点组成的链表。链表节点记录 observer、selector/block、object 过滤条件等。

追问：为什么用链表而不是数组？
**答**：增删频繁的场景下，链表 O(1) 的插入和删除更高效（非尾部操作），数组需要 O(n) 的元素移动。

### 考点 5：postNotification 内部是如何保证遍历安全的？

在遍历观察者之前，**先对当前链表做一份快照拷贝**，然后遍历这份快照。这样即使某个观察者的回调中调用了 `addObserver` 或 `removeObserver`，修改的是原始链表，不影响正在遍历的快照。

### 考点 6：如何过滤通知？

通过 `object` 参数。注册时指定 `object`，则只有来自该对象的通知才会触发回调。注册时传 `nil`，则该名称下所有通知都会触发。

```objc
// 只接收 senderA 发的 "DataChanged" 通知
[center addObserver:self selector:@selector(handler:) name:@"DataChanged" object:senderA];

// 接收所有发送者的 "DataChanged" 通知
[center addObserver:self selector:@selector(handler:) name:@"DataChanged" object:nil];

// 接收所有通知（name 和 object 都是 nil，通配）
[center addObserver:self selector:@selector(handler:) name:nil object:nil];
```

### 考点 7：通知和委托（delegate）的区别

| 对比维度 | NSNotification | Delegate |
|----------|---------------|----------|
| 通信模式 | 一对多 | 一对一 |
| 耦合度 | 松耦合（无需持有引用） | 紧耦合（需持有 delegate 引用）|
| 双向通信 | 单向（发送→接收） | 可双向（返回值） |
| 性能 | 较低（查表 + 遍历链表） | 较高（直接方法调用） |
| 编译检查 | 无（字符串通知名，可能拼错） | 有（方法签名由协议约束） |
| 适用场景 | 跨模块、全局事件、一对多 | 同模块、一对一、带返回值回调 |

### 考点 8：通知和 KVO 的区别

| 对比维度 | NSNotification | KVO |
|----------|---------------|-----|
| 触发方式 | 主动 post | 属性值变化时自动触发 |
| 使用场景 | 任意事件通知 | 监控对象属性变化 |
| 观察对象 | 任意 NSNotification | 指定对象的属性 |
| 代码侵入 | 需要主动发通知 | setter 调用自动触发 |
| 一对多 | 天然支持 | 支持 |

### 考点 9：NSNotificationCenter 是线程安全的吗？

**是。** NSNotificationCenter 内部对 `_namedTable` 的读写使用**读写锁（pthread_rwlock）**或**互斥锁**保护。添加和移除观察者、发送通知都是线程安全的。

**但是**，回调的执行在 post 的线程上，如果回调中操作共享数据，仍需使用者自己加锁。

### 考点 10：通知名为什么用 NSString，可能存在什么问题？

字符串没有编译检查，容易拼写错误。且如果多个模块使用相同的通知名，可能意外串扰。

**最佳实践**：
```objc
// 方案1：使用 extern NSString *const 常量
// .h
extern NSString * const kMyNotificationName;
// .m
NSString * const kMyNotificationName = @"MyNotificationName";

// 方案2：使用 NSNotificationName 类型（推荐）
// .h
extern NSNotificationName const MyModuleDataChangedNotification;
// .m  
NSNotificationName const MyModuleDataChangedNotification = @"MyModuleDataChangedNotification";
```

---

## 四、常见崩溃场景

### 1. 野指针崩溃（iOS 9 之前）

```objc
// Observer dealloc 后没有 removeObserver
// 下次 post 时访问已释放的 observer → EXC_BAD_ACCESS
```

**修复**：在 `dealloc` 中调用 `removeObserver:`。

### 2. block 方式的内存泄漏

```objc
// ❌ 循环引用
self.token = [[NSNotificationCenter defaultCenter] addObserverForName:@"Test"
                                                               object:nil
                                                                queue:nil
                                                           usingBlock:^(NSNotification *note) {
    [self doSomething]; // self → token → block → self
}];

// ✅ 正确写法：weak-strong dance
__weak typeof(self) weakSelf = self;
self.token = [[NSNotificationCenter defaultCenter] addObserverForName:@"Test"
                                                               object:nil
                                                                queue:nil
                                                           usingBlock:^(NSNotification *note) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    [strongSelf doSomething];
}];
```

### 3. 移除时机不当

```objc
// ❌ 不要用 nil 作为 name 去 remove（会移除所有，包括系统注册的）
[[NSNotificationCenter defaultCenter] removeObserver:self];

// ✅ 精确移除
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self name:kMyNotification object:nil];
}
```

---

## 五、一句话总结

> NSNotificationCenter 是**同步、一对多、基于哈希表+链表的观察者模式实现**。核心是 `_namedTable`（按通知名索引的哈希表）存储观察者链表。`postNotification` 同步遍历链表快照执行回调，iOS 9+ selector 方式自动清理，block 方式需手动移除。block 方式通过 queue 参数可实现异步回调。适合跨模块松耦合通信，但要注意线程安全和内存管理。
