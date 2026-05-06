# 【BibiGPT】AI 一键总结：[26_自动释放池.mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMjZf6Ieq5Yqo6YeK5pS-5rGgLW1wNC1lbmNvZGVk.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260505%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260505T045446Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D2f54e434f62f9baaa47ca250596f9d51d1ab0bc2060b10b1d2c2e5f26ba19e81%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

## 摘要
本节课主要讲解 Objective-C 中自动释放池（`@autoreleasepool`）的底层实现、数据结构、对象添加与释放流程，以及它和 RunLoop、线程、ARC/MRC 内存管理之间的关系。课程通过源码、编译结果、汇编调用和示例验证，解释了 `objc_autoreleasePoolPush`、`objc_autoreleasePoolPop`、`AutoreleasePoolPage`、哨兵对象等核心机制，并说明了在循环中手动使用自动释放池可以降低短时间内的内存峰值。


### 亮点
- 🧠 自动释放池是 Objective-C 中一种延迟释放对象的内存管理机制，它可以让对象不在作用域结束时立即释放，而是在池子被销毁时统一释放。
- 🔧 在 ARC 模式下，不能再直接使用 `NSAutoreleasePool`，通常需要通过 `@autoreleasepool {}` 来创建自动释放池。
- 🧩 `@autoreleasepool` 编译后本质上会转换为 `objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop` 两个底层函数调用。
- 🏗️ 自动释放池的底层结构由 `AutoreleasePoolPage` 组成，每个 Page 通过双向链表连接，按需创建和回收。
- 🧭 每个自动释放池都会有一个哨兵对象作为边界，释放时会从最新添加的对象开始，一直释放到哨兵对象为止。
- 🧵 自动释放池和线程强相关，每个线程都会维护自己的 autorelease 对象栈，而不是所有线程共享一个池。
- ⚡ 自动释放池会优先从线程本地存储 TLS 中查找当前可用的热页，以提升对象添加和释放的效率。
- 📦 一个 `AutoreleasePoolPage` 大小通常是 4KB，第一页扣除元数据和哨兵对象后大约可存放 504 个 autorelease 对象，后续页大约可存放 505 个对象。
- 🧪 通过 `__autoreleasing` 修饰符可以让对象显式进入自动释放池，而通过 `alloc/new/copy/mutableCopy` 创建的对象默认由 ARC 管理，不会自动进入池。
- 🔁 非 `alloc/new/copy/mutableCopy` 方式创建的某些 Objective-C 对象，可能会被系统自动加入自动释放池，例如部分便利构造方法创建的对象。
- 🔄 主线程默认开启 RunLoop，系统会在 RunLoop 即将进入时创建自动释放池，并在 RunLoop 即将休眠或退出前释放池中的对象。
- 🧵 子线程默认不一定开启 RunLoop，但如果子线程中产生了 autorelease 对象，也会维护自己的自动释放池，并通常在线程结束时清空。
- ❓ 面试中如果问 ARC 下 autorelease 对象何时释放，需要区分手动 `@autoreleasepool` 包裹的对象、主线程 RunLoop 管理的对象，以及子线程中的对象。
- 🚀 在大量循环创建临时 autorelease 对象时，手动包一层 `@autoreleasepool` 可以及时清理临时对象，避免内存峰值快速上涨。
- 🧯 自动释放池释放对象时，本质上仍然会向池中的对象发送 `release` 消息，因此它并不是脱离引用计数体系的独立机制。


[#ObjectiveC](https://bibigpt.co/search?q=ObjectiveC) [#自动释放池](https://bibigpt.co/search?q=%E8%87%AA%E5%8A%A8%E9%87%8A%E6%94%BE%E6%B1%A0) [#RunLoop](https://bibigpt.co/search?q=RunLoop) [#ARC](https://bibigpt.co/search?q=ARC) [#内存管理](https://bibigpt.co/search?q=%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)


### 思考 
1. **[ARC 模式下，autorelease 对象到底什么时候释放？](https://bibigpt.co/search?q=ARC%20%E6%A8%A1%E5%BC%8F%E4%B8%8B%EF%BC%8Cautorelease%20%E5%AF%B9%E8%B1%A1%E5%88%B0%E5%BA%95%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E9%87%8A%E6%94%BE%EF%BC%9F)**
  - 如果对象被手动放进 `@autoreleasepool` 中，它会在超出该自动释放池作用域时释放。如果是系统自动加入自动释放池的对象，在主线程中通常会随着 RunLoop 即将休眠或退出时释放；如果在未开启 RunLoop 的子线程中，则通常在线程结束时释放。
- 2. **[为什么在循环中经常要手动写 `@autoreleasepool`？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%E5%9C%A8%E5%BE%AA%E7%8E%AF%E4%B8%AD%E7%BB%8F%E5%B8%B8%E8%A6%81%E6%89%8B%E5%8A%A8%E5%86%99%20%60%40autoreleasepool%60%EF%BC%9F)**
  - 因为循环中可能会频繁创建大量临时 autorelease 对象，如果只依赖外层 RunLoop 或线程结束时统一释放，短时间内内存峰值可能迅速升高。手动在循环内部创建自动释放池，可以让每轮循环结束后及时释放临时对象。
- 3. **[`alloc` 创建的对象会自动进入自动释放池吗？](https://bibigpt.co/search?q=%60alloc%60%20%E5%88%9B%E5%BB%BA%E7%9A%84%E5%AF%B9%E8%B1%A1%E4%BC%9A%E8%87%AA%E5%8A%A8%E8%BF%9B%E5%85%A5%E8%87%AA%E5%8A%A8%E9%87%8A%E6%94%BE%E6%B1%A0%E5%90%97%EF%BC%9F)**
  - 默认不会。通过 `alloc/new/copy/mutableCopy` 创建的对象通常由 ARC 的强引用和引用计数规则管理。如果希望它进入自动释放池，需要使用类似 `__autoreleasing` 的方式，或通过会返回 autorelease 对象的便利构造方法创建。


### 术语解释
- **自动释放池（Autorelease Pool）**: Objective-C 中用于延迟释放对象的一种内存管理机制，池子销毁时会统一向其中的对象发送 `release` 消息。
- **AutoreleasePoolPage**: 自动释放池底层用于存储 autorelease 对象的页结构，每页约 4KB，并通过双向链表连接。
- **哨兵对象（Pool Boundary）**: 自动释放池中的边界标记，用来区分当前池的释放范围；执行 `pop` 时会释放到该哨兵对象为止。
- **RunLoop**: 线程中的事件循环机制，主线程默认开启，并会在循环进入和休眠等阶段自动创建和释放自动释放池。
- **TLS（Thread Local Storage）**: 线程本地存储，用于保存当前线程相关的数据，例如自动释放池中的热页信息，以便快速查找和操作。

---

## 视频章节总结 ｜ iOS底层原理：自动释放池源码解析与性能优化实战

本视频是iOS底层系列课程的第26节，深入讲解了自动释放池（Auto Release Pool）的底层原理与应用。课程从自动释放池的基本概念入手，通过Clang编译后的C++代码揭示了@autoreleasepool块本质上是对objc_autoreleasePoolPush和objc_autoreleasePoolPop的调用。随后，结合objc4源码详细分析了核心数据结构AutoreleasePoolPage，包括哨兵对象（POOL_BOUNDARY）、双向链表、线程本地存储（TLS）以及冷热页的概念，并演示了push操作中对象的添加流程和新页的创建。接着，重点讨论了哪些对象会被添加到自动释放池、__autoreleasing修饰符的作用，以及与RunLoop的紧密关系：主线程的RunLoop通过注册两个Observer在即将进入循环时创建自动释放池，在即将休眠时销毁池并释放对象，从而解释了自动释放对象的释放时机。课程还计算了每个Page可存储504/505个对象，并通过pop操作演示了如何从热页向冷页遍历并发送release消息。最后，通过内存暴涨的Demo对比，展示了手动添加@autoreleasepool在循环中缓解内存峰值的实战技巧，并总结了相关面试题的回答要点。

### [00:00](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=0.000) - 📖 课程引入与自动释放池概念
课程开始，讲师介绍了自动释放池是OC中一种自动回收内存的机制，它可以延迟变量的释放时机。通过第一个Demo，展示了@autoreleasepool代码块，并指出在ARC模式下只能使用@autoreleasepool而不能使用NSAutoreleasePool。随后，将源码转为C++代码，发现@autoreleasepool被转换为构造函数和析构函数的调用，分别对应push和pop操作。这为后续深入源码分析奠定了理论基础。

### [32:51](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=1971.290) - 📦 核心数据结构与Push入池机制
进入objc4源码，逐行分析AutoreleasePoolPage的实现。首先解读了源码注释，明确自动释放池是一个堆栈指针，每个指针要么是待释放对象，要么是哨兵对象（POOL_BOUNDARY）。哨兵标志着池的边界，当pop时，所有比哨兵“热”的对象都会被释放。AutoreleasePoolPage继承自AutoreleasePoolPageData，内部包含线程、父节点、子节点、深度、next指针等成员，构成双向链表。Push操作首先通过TLS获取hotPage，若不存在则调用autoreleaseNoPage创建首页；若当前Page已满，则通过do-while循环查找未满的子Page或创建新Page。构造函数中通过begin()计算数据存储起始地址，Page首部占用56字节，后续对象地址依次排列。

### [01:11:47](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=4307.180) - ⏳ 对象添加规则与RunLoop释放时机
本章详细说明了哪些对象会被自动加入自动释放池：通过__autoreleasing修饰的变量，或者使用非alloc/new/copy/mutableCopy创建的对象（如stringWithFormat:长字符串）会被系统自动添加；而alloc/new/copy等方法创建的对象由ARC管理引用计数，不会入池。通过_objc_autoreleasePoolPrint函数可以打印池内对象，验证了添加行为。紧接着深入讲解了自动释放池与RunLoop的关系：App启动时主线程RunLoop自动开启，注册了两个Observer——一个在kCFRunLoopEntry（最高优先级）创建自动释放池，另一个在kCFRunLoopBeforeWaiting（最低优先级）销毁池并释放所有对象。因此，未手动添加的autorelease对象在主线程会随RunLoop休眠时释放；子线程若未开启RunLoop，则在线程销毁时释放，若开启了RunLoop则与主线程行为一致。这部分也构成了面试高频问题的标准答案。

### [01:59:10](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=7150.400) - 📊 Page容量与Pop释放原理
讲解了AutoreleasePoolPage的大小固定为4KB（4096字节），除去56字节的自身数据和一个哨兵占用的8字节，实际可存储504个对象地址；第二页起可存505个。通过循环创建505个对象验证了新Page的生成，并直观展示了冷热页的切换。Pop操作接收哨兵对象作为参数，从当前hotPage向前遍历，依次对每个对象调用objc_release，直到遇见哨兵对象停止。整个过程会逆向遍历多个Page，确保当前池中所有对象都被释放，实现内存的自动回收。

### [02:19:18](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=8358.340) - ⚡ 实战：防止内存暴涨
通过一个循环创建大量UIImage（或自定义对象）的Demo，对比显示了不使用@autoreleasepool时内存持续攀高、甚至导致崩溃的现象，而使用@autoreleasepool包裹循环体后内存保持平稳。分析其原因在于：频繁创建大量autorelease对象时，RunLoop未及时回收会导致内存堆积；手动添加的@autoreleasepool可以立即在块结束时强制释放临时对象，避免峰值过高。许多第三方框架在循环内部也采用此技巧优化内存。但需注意，即使不添加@autoreleasepool，ARC也会在合适时机插入release，只是释放速度可能赶不上创建速度，因此在热路径中仍推荐手动管理。

### [02:34:57](https://bibigpt.co/content/d36767ba-110a-49ec-a0fd-0e9a62b44aaa?t=9297.820) - 🔚 课程总结与答疑
课程最后，讲师总结了自动释放池的核心原理：它是一种自动内存回收机制，配合ARC和RunLoop共同管理OC对象生命周期。针对学员提出的关于冷热页、释放顺序、ARC对比、子线程池、嵌套池等疑问进行了解答，并提醒大家课后自行总结以加深理解。预告了下一节课程将讲解KVC和KVO，随后下课。