
# GCD

## 函数和队列

### 函数
- dispatch_async
  - 将任务异步提交到指定队列，立即返回，不阻塞当前线程。任务会在队列对应的线程（主队列→主线程，其他队列→后台线程）中稍后执行。
  - **不一定会创建子线程**。只是负责将任务提交到队列中，系统从线程池中复用线程在执行。如果队列是主队列，任务会在主线程中执行（依然异步）。**如果是全局并发队列或自定义串行/并发队列，才可能使用子线程**。
- dispatch_sync
  - 将任务同步提交到指定队列，**阻塞当前线程**，直到任务执行完成才继续。注意：若当前线程与目标队列是同一个串行队列，则发生死锁（如主线程往主队列同步提交任务）。
  

### 队列
- dispatch_get_global_queue
- dispatch_get_main_queue

创建自定义串行队列
-```objc
dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
```

创建自定义并发队列
-```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
```

### 其他函数

#### 1. dispatch_once 是GCD提供的一次性线程安全执行API，保证传入的block在整个应用生命周期中只执行一次，且天然线程安全。
- 确保任务只执行一次，无论在多少个线程中调用。
- 用于初始化全局变量、单例模式等。

> 底层原理：底层基于原子操作(CAS) + 轻量级锁/信号量 + 内存屏障实现。状态流转如下：
> 1. 状态定义（dispatch_once_t 的值）
> - 0（初始态）：未执行
> - ~0（全 1，完成态）：已执行完成
> - 中间值（锁状态）：执行中（加锁，其他线程等待）
> 2. 关键点：
> - 原子操作（CAS/OSAtomic）
>   - 保证对 predicate 的读 / 改 / 写原子性，避免多线程竞争导致的重复执行。
> - 内存屏障（Memory Barrier）
>   - 防止编译器 / CPU 指令重排，保证 block 内的初始化操作先于 predicate 标记完成，避免 “半初始化” 状态被其他线程看到。
> - 轻量级等待（休眠 + 唤醒）
>   - 未抢到锁的线程休眠（不耗 CPU），等执行完成后被唤醒，比 @synchronized 或 pthread_mutex 更高效
>

```objc
// 经典用法
+ (instancetype)sharedInstance {
    dispatch_once_t onceToken;
    static id instance = nil;
    dispatch_once(&onceToken, ^{
        // 代码只执行一次
        instance = [[self alloc] init];
    });
    return instance;
}
```

#### 2. dispatch_barrier 是 GCD 中专为并发队列设计的线程同步工具，核心作用：像栅栏一样挡住前面的异步任务，等它们全部执行完，再执行自己，自己执行完后，后面的异步任务才继续执行。

- dispatch_barrier_async (建议使用)
  - 异步栅栏，确保在栅栏前的所有任务执行完成后，再执行栅栏内的任务。**不阻塞当前线程**。
- dispatch_barrier_sync (不建议使用)
  - 同步栅栏，确保在栅栏前的所有任务执行完成后，再执行栅栏内的任务。**阻塞当前线程**。

Q: 为什么不能用全局并发队列？

A: 系统全局队列（dispatch_get_global_queue）整个系统共用，你加栅栏会阻塞系统任务，所以 Apple 直接让栅栏失效，保证系统安全。

**注意**：栅栏任务不要做耗时操作，会卡住整个队列

##### 底层核心原理（深度讲解）

1. 队列结构
GCD 并发队列内部维护 2 个链表：
- 普通任务链表
- 栅栏任务链表

2. 调度规则
- 只要栅栏未执行，后面的所有普通任务都不会被调度
- 栅栏执行的条件：
    - 前面普通任务 全部完成
    - 队列中 无其他任务执行
- 栅栏执行时：
    - 队列临时变成串行，只执行栅栏
- 栅栏完成后：
    - 恢复并发模式，调度后面的普通任务

```objc
// 最经典使用场景：线程安全的多读单写
// 需求
// 读操作：可以多线程并发执行（提高性能）
// 写操作：必须单独执行，不能和任何任务并发（防止数据错乱）
// 1. 创建【自定义并发队列】（关键！）
dispatch_queue_t _safeQueue = dispatch_queue_create("com.example.safeQueue", DISPATCH_QUEUE_CONCURRENT);

// 读操作：普通异步，并发执行
- (id)objectForKey:(NSString *)key {
    __block id obj;
    dispatch_sync(_safeQueue, ^{  // 同步等待结果
        obj = [self.dict objectForKey:key];
    });
    return obj;
}

// 写操作：栅栏函数，独占执行
- (void)setObject:(id)obj forKey:(NSString *)key {
    // 栅栏：写的时候，前面所有读都执行完，再写，写完再允许读
    dispatch_barrier_async(_safeQueue, ^{
        [self.dict setObject:obj forKey:key];
    });
}
```

#### 待完善 3. dispatch_group 是GCD提供的一种同步提交任务的方式，确保在提交任务前，所有正在执行的任务都已完成。
```objc
// 示例
dispatch_group_t group = dispatch_group_create();
dispatch_group_enter(group);
dispatch_group_leave(group);
dispatch_group_wait(group, dispatch_get_main_queue());
NSLog(@"group");
```

#### 待完善 4. dispatch_semaphore 是GCD提供的一种同步提交任务的方式，确保在提交任务前，所有正在执行的任务都已完成。
```objc
// 示例
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
dispatch_semaphore_wait(semaphore, dispatch_get_main_queue());
NSLog(@"semaphore");
```

#### 待完善 5.dispatch_source 是 GCD 提供的事件驱动型内核事件监控工具，底层封装了 XNU 内核的 kqueue/Mach port，用于无轮询、低 CPU 占用地监听系统事件，事件触发后自动在指定队列异步回调。相比 NSTimer/RunLoop，它高精度、线程安全、不依赖 RunLoop、可后台运行


## 例题

### 1
```objc
@property (nonatomic, assign) int num;
- (void)test {
    dispatch_queue_t t = dispatch_queue_create("testQueue", DISPATCH_QUEUE_CONCURRENT);
    NSLog(@"1");
    dispatch_async(t, ^{
        NSLog(@"2");
        dispatch_sync(t, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
    dispatch_async(t, ^{
        NSLog(@"6");
    });

    // 5在6前面，1在234前面。 2在3前面。3在4前面。 1在5前面。6和234无关系。
    // 可能的答案 [1,5,2,3,4,6] [1,2,3,5,6,4]
}
```

### 1.1
```objc
- (void)test {
    dispatch_queue_t t = dispatch_queue_create("testQueue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"1");
    dispatch_sync(t, ^{
        NSLog(@"2");
    });
    dispatch_async(t, ^{
        NSLog(@"3");
        NSLog(@"4");
    });
    NSLog(@"5");
    dispatch_async(t, ^{
        NSLog(@"6");
    });

    // 1在2。 34在6前面。 5和34无关系。6一定在5后面
    // 可能的答案 [1,2,5,3,4,6]
}
```

### 1.2
```objc
- (void)test {
    dispatch_queue_t t = dispatch_queue_create("testQueue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"1");
    dispatch_sync(t, ^{
        NSLog(@"2");
    });
    dispatch_async(t, ^{
        NSLog(@"3");
        NSLog(@"4");
        dispatch_async(t, ^{
            NSLog(@"7");
        });
    });
    NSLog(@"5");
    // sleep(1); 加上就是答案2，不加就是答案1
    dispatch_async(t, ^{
        NSLog(@"6");
    });
    // 6和7没有相对关系 5和6之间如果有耗时操作,7就会比6先进队列
    // 可能的答案 [1,2,5,3,4,6,7]、[1,2,5,3,4,7,6]
}
```

### 2
```objc
@property (nonatomic, assign) int num;
- (void)test3 {
    self.num = 0;
    while (self.num < 100) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            self.num++;
        });
    } 
    NSLog(@"num = %d", self.num);
    // 最终输出的 num 值是不确定的，取决于运行时的线程调度、竞争严重程度和系统负载。通常情况下，最终结果会是一个 大于等于 100 的整数，且大多远大于 100（例如 200、500 甚至更多）
    // 即使把 nonatomic 改为atomic, 也不能保证最终结果是 100. 因为 atomic 只保证 单个 self.num 的 getter 和 setter 是原子的（即读取或写入时不会被打断），但 不保证 self.num++ 的整体原子性
}
```

### 3
```objc
@property (nonatomic, assign) int num;
- (void)test4 {
    self.num = 0;
    for (int i = 0; i < 100; ++i) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            self.num++;
        });
    }
    NSLog(@"num = %d", self.num);
    // 答案最终 <= 100， 这里是开启的子线程，不会阻塞主线程的结果输出

    // 若要最终输出的必须要刚好等于100，可以使用 dispatch_group + 递归锁 等待所有子线程执行完毕
    self.num = 0;
    dispatch_group_t group = dispatch_group_create();

    for (int i = 0; i < 100; ++i) {
        dispatch_group_enter(group);
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            @synchronized (self) {
                self.num++;
            }
            dispatch_group_leave(group);
        });
    }

    dispatch_group_wait(group, DISPATCH_TIME_FOREVER); // 作用是 阻塞当前线程（这里是主线程），直到 group 中的所有已加入任务全部执行完成。
    NSLog(@"num = %d", self.num);  // 输出 100
}
```