
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
```objc
dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
```

创建自定义并发队列
```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
```

## dispatch_once

是 GCD 提供的一次性线程安全执行 API，保证传入的 block 在整个应用生命周期中只执行一次，且天然线程安全。

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
>   - 防止编译器 / CPU 指令重排，保证 block 内的初始化操作先于 predicate 标记完成，避免 "半初始化" 状态被其他线程看到。
> - 轻量级等待（休眠 + 唤醒）
>   - 未抢到锁的线程休眠（不耗 CPU），等执行完成后被唤醒，比 @synchronized 或 pthread_mutex 更高效

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

## dispatch_barrier

是 GCD 中专为并发队列设计的线程同步工具，核心作用：像栅栏一样挡住前面的异步任务，等它们全部执行完，再执行自己，自己执行完后，后面的异步任务才继续执行。

- dispatch_barrier_async (建议使用)
  - 异步栅栏，确保在栅栏前的所有任务执行完成后，再执行栅栏内的任务。**不阻塞当前线程**。
- dispatch_barrier_sync (不建议使用)
  - 同步栅栏，确保在栅栏前的所有任务执行完成后，再执行栅栏内的任务。**阻塞当前线程**。

Q: 为什么不能用全局并发队列？

A: 系统全局队列（dispatch_get_global_queue）整个系统共用，你加栅栏会阻塞系统任务，所以 Apple 直接让栅栏失效，保证系统安全。

**注意**：栅栏任务不要做耗时操作，会卡住整个队列。

### 底层核心原理

1. 队列结构
   GCD 并发队列内部维护 2 个链表：
   - 普通任务链表
   - 栅栏任务链表

2. 调度规则
   - 只要栅栏未执行，后面的所有普通任务都不会被调度
   - 栅栏执行的条件：
     - 前面普通任务全部完成
     - 队列中无其他任务执行
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

```objc
- (void)test {
    dispatch_queue_t t = dispatch_queue_create("testQueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(t, ^{
        NSLog(@"1");
    });
    dispatch_async(t, ^{
        NSLog(@"2");
    });
    dispatch_barrier_async(_safeQueue, ^{
        NSLog(@"3");
    });
    NSLog(@"4");
    dispatch_async(t, ^{
        NSLog(@"5");
    });
    // 3 一定在 1、2 之后，且一定在 5 之前；
    // 4 可以出现在任何位置
    // 可能输出的答案是 [1,2,3,4,5]
}
```

## dispatch_group

监听一组异步任务全部完成，然后执行收尾操作。

```objc
dispatch_group_async(group, queue, ^{
    NSLog(@"开始任务 1");
    [NSThread sleepForTimeInterval:1]; // 模拟耗时
    NSLog(@"完成任务 1");
});

// 等价于

// 1. 手动 enter
dispatch_group_enter(group);

// 2. 普通异步执行
dispatch_async(queue, ^{
    NSLog(@"开始任务 1");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"完成任务 1");

    // 3. 任务结束后自动 leave
    dispatch_group_leave(group);
});
```

### 核心 API 速查表

| API | 作用 |
|-----|------|
| dispatch_group_create() | 创建分组 |
| dispatch_group_async() | 把同步任务加入分组（适合同步任务） |
| dispatch_group_enter() | 手动标记任务开始（计数 + 1） |
| dispatch_group_leave() | 手动标记任务结束（计数 - 1） |
| dispatch_group_notify() | 所有任务结束后执行回调（推荐） |
| dispatch_group_wait() | 阻塞等待任务结束（慎用） |

```objc
// 示例
// 1. 创建一个调度分组
dispatch_group_t group = dispatch_group_create();

// 2. 获取全局并发队列（并行执行任务）
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

// 任务 1
dispatch_group_async(group, queue, ^{
    NSLog(@"开始任务 1");
    [NSThread sleepForTimeInterval:1]; // 模拟耗时
    NSLog(@"完成任务 1");
});

// 任务 2
dispatch_group_async(group, queue, ^{
    NSLog(@"开始任务 2");
    [NSThread sleepForTimeInterval:2]; // 模拟耗时
    NSLog(@"完成任务 2");
});

// 任务 3
dispatch_group_async(group, queue, ^{
    NSLog(@"开始任务 3");
    [NSThread sleepForTimeInterval:1.5];
    NSLog(@"完成任务 3");
});

// 3. 监听：所有任务都完成后，执行这里
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"✅ 所有任务全部完成！刷新 UI/提示用户");
});

// 输出示例：
// 开始任务 1
// 开始任务 2
// 开始任务 3
// 完成任务 1
// 完成任务 3
// 完成任务 2
// ✅ 所有任务全部完成！刷新 UI/提示用户
```

进阶用法：手动控制任务进入/离开（最常用！）

因为 `dispatch_group_async` 只能包同步代码，**包异步代码会失效**。

**正确写法（网络请求必备）**
```objc
dispatch_group_t group = dispatch_group_create();

// -------- 任务 1：网络请求 --------
// 进入分组（计数 +1）
dispatch_group_enter(group);
[self requestApi1:^{
    NSLog(@"接口 1 完成");

    // 离开分组（计数 -1）
    dispatch_group_leave(group);
}];

// -------- 任务 2：网络请求 --------
dispatch_group_enter(group);
[self requestApi2:^{
    NSLog(@"接口 2 完成");
    dispatch_group_leave(group);
}];

// -------- 全部完成 --------
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"✅ 两个接口都请求完毕！");
});
```

**避坑指南**

1. **异步任务必须用 enter/leave** — 网络请求、文件下载、延时操作都要用。
2. **notify 队列建议用主队列** — 因为最后一般要刷新 UI。
3. **不要在主线程用 wait** — 会卡死界面。
4. **enter 和 leave 必须成对** — 否则要么不执行 notify，要么直接崩溃。

## dispatch_semaphore

`dispatch_semaphore`（GCD 信号量）是 GCD 中的线程同步工具，核心作用：

1. **控制并发数量**（限制同时执行的任务数）
2. **线程等待与唤醒**（让一个线程等待另一个线程执行完再继续）
3. **解决线程安全问题**（保护临界区资源）

它的原理非常简单：**信号量计数器 + 等待/发送信号**。

### 核心三个 API

信号量本质是一个**计数器**，只有 3 个方法：

| 函数 | 作用 |
|------|------|
| `dispatch_semaphore_create(long value)` | **创建信号量**，`value` 是初始计数值 |
| `dispatch_semaphore_wait(dispatch_semaphore_t, dispatch_time_t)` | **等待信号**（P操作）：计数值 -1，若 < 0，线程阻塞等待 |
| `dispatch_semaphore_signal(dispatch_semaphore_t)` | **发送信号**（V操作）：计数值 +1，若 ≤ 0，唤醒一个等待的线程 |

通俗理解：
- 信号量数值 = **剩余可执行名额**
- `wait`：抢占一个名额，没名额就**等着**
- `signal`：归还一个名额，有人等着就**唤醒**

### 基础使用步骤

```objc
// 1. 创建信号量（参数：初始计数器值）
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

// 2. 等待信号（占用名额）
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

// 3. 执行需要同步/保护的代码（临界区）
NSLog(@"执行任务");

// 4. 发送信号（释放名额）
dispatch_semaphore_signal(semaphore);
```

### 常用实战场景

#### 场景 1：线程同步（让异步任务变同步）

**需求**：等待网络请求/异步操作完成后，再执行后续代码。

```objc
// 创建信号量，初始值 0（必须等信号才能继续）
dispatch_semaphore_t sem = dispatch_semaphore_create(0);

// 模拟异步网络请求
NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:[NSURL URLWithString:@"https://www.baidu.com"] completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    NSLog(@"异步请求完成");

    // 发送信号：计数器 +1，唤醒等待的线程
    dispatch_semaphore_signal(sem);
}];
[task resume];

NSLog(@"等待请求完成...");
// 等待信号：计数器 -1（变为 -1），线程阻塞
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);

// 这行代码一定会在请求完成后执行
NSLog(@"请求已完成，继续执行后续代码");
```

✅ **关键点**：
- 初始值传 `0` → `wait` 会**立即阻塞**
- 异步任务完成后调用 `signal` → 唤醒线程

#### 场景 2：限制最大并发数（控制线程数量）

**需求**：同时最多执行 3 个任务，防止线程爆炸/资源耗尽。

```objc
// 创建信号量，初始值 3 → 最多 3 个任务同时执行
dispatch_semaphore_t sem = dispatch_semaphore_create(3);
// 并发队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

// 模拟 10 个并发任务
for (int i = 0; i < 10; i++) {
    dispatch_async(queue, ^{
        // 抢占名额：计数器 -1
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);

        // 执行任务
        NSLog(@"执行任务 %d", i);
        sleep(1); // 模拟耗时操作

        // 释放名额：计数器 +1
        dispatch_semaphore_signal(sem);
    });
}
```

✅ **效果**：日志会**每 1 秒输出 3 条**，严格控制并发数。

#### 场景 3：线程安全（保护多线程访问的变量）

**需求**：多个线程同时修改一个变量，防止数据错乱。

```objc
// 初始化信号量为 1（互斥锁：同一时间只有 1 个线程访问）
dispatch_semaphore_t sem = dispatch_semaphore_create(1);
__block int count = 0;

dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
for (int i = 0; i < 1000; i++) {
    dispatch_async(queue, ^{
        // 加锁：进入临界区
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);

        // 安全操作共享变量
        count++;
        NSLog(@"当前计数：%d", count);

        // 解锁：离开临界区
        dispatch_semaphore_signal(sem);
    });
}
```

✅ **关键点**：
- 初始值 `1` = **互斥锁**（和 `@synchronized` 作用一样，但性能更高）

### 关键参数详解

#### `dispatch_semaphore_wait` 超时时间

```objc
// 永久等待（最常用）
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);

// 等待 3 秒，超时后不等待
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC);
dispatch_semaphore_wait(sem, time);
```

#### 返回值判断

`wait` 会返回一个 `long` 值：
- `0`：**成功获取信号**
- 非 0：**等待超时**

```objc
long result = dispatch_semaphore_wait(sem, time);
if (result == 0) {
    NSLog(@"获取信号成功");
} else {
    NSLog(@"等待超时");
}
```

### 使用注意事项

1. **必须成对使用** — 一个 `wait` 必须对应一个 `signal`，否则会永久死锁。
2. **不要在主线程无限等待** — `DISPATCH_TIME_FOREVER` 会卡住主线程，导致 APP 卡死。
3. **创建与使用要匹配** — 不要跨函数随意创建/释放信号量，统一管理。
4. **初始值不能为负数** — `dispatch_semaphore_create(x)` 中 `x` 必须 ≥ 0。

### 总结

**快速记忆口诀：**
```
创建信号量，数值定名额；
wait 减一抢，没值就等待；
signal 加一还，唤醒等待者；
成对不遗漏，线程更安全。
```

**核心要点：**
1. **核心 API**：`create`/`wait`/`signal`
2. **常用场景**：同步异步、限制并发、线程安全
3. **关键数值**：
   - `0` = 等待唤醒
   - `1` = 互斥锁
   - `N` = 最大并发数
4. **避坑**：必须成对调用，主线程慎用永久等待

## dispatch_source

`dispatch_source` 是 GCD 中用于处理事件源的工具。它不依赖 RunLoop，比 NSTimer 更精确，也支持自定义事件的合并。你提到的"add"和"timer"分别对应它的两种常见类型：

- `DISPATCH_SOURCE_TYPE_DATA_ADD`：**数据累加**，可将多次事件合并成一次回调。
- `DISPATCH_SOURCE_TYPE_TIMER`：**定时器**，可精确控制执行间隔。

### Add 例子（数据累加合并事件）

这个例子模拟多个线程并发触发事件，源会将累加值合并，最终只触发一次回调。

```objc
// 1. 创建一个累加类型的 source，绑定到全局队列
dispatch_source_t addSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0,
                                                      dispatch_get_global_queue(0, 0));

// 2. 设置事件处理器：当累加值有变化时调用
dispatch_source_set_event_handler(addSource, ^{
    // 读取当前累加的总值
    unsigned long value = dispatch_source_get_data(addSource);
    NSLog(@"🔥 累加值更新：%lu", value);
});

// 3. 启动 source（必须调用，否则不会工作）
dispatch_resume(addSource);

// 4. 模拟多个线程并发 merge 数据
dispatch_apply(10, dispatch_get_global_queue(0, 0), ^(size_t i) {
    // 每次 merge 将累加值增加 1，source 内部会自动求和
    dispatch_source_merge_data(addSource, 1);
    NSLog(@"线程 %zu merge 了 1", i);
});

// 保持运行一会儿，让回调执行
sleep(1);
```

**运行现象**：你会看到 10 条 merge 日志，但**累加回调只触发了一次**，打印出 `累加值更新：10`。这就是 Add 类型的核心能力：**将高频事件合并，避免频繁回调**。

### Timer 例子（定时器）

```objc
// 1. 创建一个定时器 source，绑定到主队列（可任意队列）
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0,
                                                 dispatch_get_main_queue());

// 2. 设置定时器：何时开始、间隔、容差
dispatch_source_set_timer(timer,
                          dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC), // 2秒后启动
                          1 * NSEC_PER_SEC,  // 每1秒触发一次
                          0.1 * NSEC_PER_SEC); // 允许100毫秒的误差（提高能效）

// 3. 设置事件处理器
dispatch_source_set_event_handler(timer, ^{
    NSLog(@"⏰ 定时器触发：%@", [NSDate date]);
});

// 4. 设置取消处理器（可选）
dispatch_source_set_cancel_handler(timer, ^{
    NSLog(@"计时器已取消");
});

// 5. 启动定时器
dispatch_resume(timer);

// 6. 10秒后取消定时器（实际使用时可以按需取消）
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 10 * NSEC_PER_SEC),
               dispatch_get_main_queue(), ^{
    dispatch_source_cancel(timer);
});
```

**关键说明**：
- `dispatch_source_set_timer` 的**容差值**可以让系统合并相近的定时任务以节省电量，设为 0 可获得最精确的时间，但更耗电。
- 定时器在后台线程或主线程都行，取决于你创建时指定的队列。
- **一定要 `dispatch_resume`**，否则 source 不会执行。
- 使用 `dispatch_source_cancel` 来停止，并在 cancel handler 中做清理。

### 核心 API 速查

| 函数 / 常量 | 作用 |
|------------|------|
| `dispatch_source_create(TYPE, ...)` | 创建 source，TYPE 可选 `DATA_ADD` 或 `TIMER` |
| `dispatch_source_set_event_handler(source, block)` | 设置事件回调 |
| `dispatch_source_merge_data(addSource, val)` | 对 ADD 源累加数据 |
| `dispatch_source_get_data(addSource)` | 获取 ADD 源当前累加值 |
| `dispatch_source_set_timer(timer, start, interval, leeway)` | 设置定时器参数 |
| `dispatch_resume(source)` | **必须调用**以启动 source |
| `dispatch_source_cancel(source)` | 取消 source |

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

    // 5在6前面，1在234前面。2在3前面。3在4前面。1在5前面。6和234无关系。
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

    // 1在2前。34在6前面。5和34无关系。6一定在5后面
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
    // 最终输出的 num 值是不确定的，取决于运行时的线程调度、竞争严重程度和系统负载。通常情况下，最终结果会是一个大于等于 100 的整数，且大多远大于 100（例如 200、500 甚至更多）
    // 即使把 nonatomic 改为 atomic，也不能保证最终结果是 100。因为 atomic 只保证单个 self.num 的 getter 和 setter 是原子的（即读取或写入时不会被打断），但不保证 self.num++ 的整体原子性
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
    // 答案最终 <= 100，这里是开启的子线程，不会阻塞主线程的结果输出

    // 若要最终输出的必须刚好等于100，可以使用 dispatch_group + 递归锁等待所有子线程执行完毕
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

    dispatch_group_wait(group, DISPATCH_TIME_FOREVER); // 作用是阻塞当前线程（这里是主线程），直到 group 中的所有已加入任务全部执行完成。
    NSLog(@"num = %d", self.num);  // 输出 100
}
```
