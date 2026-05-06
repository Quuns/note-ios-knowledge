# LLVM 编译器

## 编译流程

- 预处理
    - 替换宏 定义宏  #define  #include  #endif
- 编译前端
    - 语法分析-检查语法错误
    - 语义分析-检查语义错误
- 代码生成
    - 生成中间代码IR (Intermediate Representation)
    - 优化IR
- 编译后端
    - 生成汇编代码
- 汇编-生成可执行文件
- 链接-合并多个目标文件和库文件，生成最终的可执行文件


## 启动优化

- 二进制重排优化
    - 核心目的是为了减少缺页终中断的次数，让app在启动时能够更快地加载到内存中，提高启动速度
    - 优化原理：将app的代码和数据段重新排列，使代码段和数据段在内存中更连续，减少缺页终中断的次数
- 代码优化
    - 删除未使用的代码
    - 


## 界面卡顿原理&优化

### 术语解释
- 垂直同步信号（VSync）: 屏幕按固定刷新频率发出的同步信号，系统需要在下一次信号到来前准备好新一帧画面，否则可能掉帧。

- 帧缓冲区（Frame Buffer）: 存放最终渲染结果的内存区域，显示硬件会从这里读取每个像素的颜色信息并显示到屏幕上。
- CADisplayLink: iOS 中与屏幕刷新同步的定时器，常用于 FPS 统计、动画驱动和卡顿检测。
- RunLoop: 线程的事件循环机制，负责处理定时器、输入源、事件和休眠唤醒状态；监听主 RunLoop 状态可以辅助判断主线程卡顿。
- 下采样（Downsampling）: 在加载图片时按目标显示尺寸生成较小版本的图片，减少解码成本和内存占用。

### 前置知识

1. 界面卡顿的本质通常是掉帧，而掉帧来自 CPU 和 GPU 在下一次垂直同步信号Vsync到来前没有完成计算与渲染。

2. 在 60Hz 屏幕下，每一帧大约只有 16.67 毫秒的处理时间，超过这个时间就可能产生肉眼可见的卡顿。

3. 双缓冲可以通过两个帧缓冲区交替读写来减少等待，但如果渲染仍然赶不上刷新节奏，依旧会发生掉帧。（三缓冲区可以解决）
    - 三缓冲区：在双缓冲的基础上，再增加一个缓冲区，用于存储下一次渲染的帧。
    - 优化原理：在渲染时，先将当前帧渲染到缓冲区中，而不是直接渲染到屏幕上。
    - 优化效果：可以减少掉帧，提高界面的流畅度。
    - 注意：三缓冲区的实现需要在渲染时添加额外的逻辑，来切换缓冲区。

### 判断卡顿的方法

- CADisplayLink 可以绑定主 RunLoop，用垂直同步信号触发次数来估算 FPS，从而判断主线程是否出现卡顿。
- 可以在主线程中添加一个定时器，定时检查当前帧的渲染时间，如果超过 16.67 毫秒，就认为当前帧卡顿了。


### 优化方法


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
    dispatch_async(t, ^{
        NSLog(@"6");
    });

    // 可能的答案 [1,2,5,3,4,6,7] ? 有问题
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