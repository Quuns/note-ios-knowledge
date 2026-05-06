# 【BibiGPT】AI 一键总结：[25_内存管理《weak》.mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMjVf5YaF5a2Y566h55CG44CKd2Vha-OAiy1tcDQtZW5jb2RlZA%253D%253D.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260505%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260505T045344Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D5d4eae1ddcec77da9b8db11493c709fd92378b3ceae3a4e031661aac5e33cba0%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

## 摘要
本节课继续讲解 iOS 内存管理，重点围绕 `weak` 的底层实现展开。课程从 `SideTable`、`weak_table`、`weak_entry_t` 的结构入手，分析弱引用指针如何被注册、移除，以及对象释放时如何清理弱引用。同时解释了为什么打印 `weak` 对象引用计数时可能显示为 2，并澄清 `weak` 不会真正增加对象引用计数。最后还初步引出了自动释放池 `autoreleasepool` 的作用、使用场景和内存峰值控制。


### 亮点
- 🧠 iOS 内存管理中常见的优化方案包括 Tagged Pointer、non-pointer isa 和 SideTable，它们分别解决小对象存储、对象元信息压缩和额外引用信息存储的问题。
- 🧩 Tagged Pointer 并不是真正指向堆内存的普通指针，而是直接把小对象的值编码进指针本身，从而节省内存。
- 🧱 non-pointer isa 会在 isa 指针中存储引用计数、是否正在释放、是否被弱引用等对象状态信息。
- 🗂️ 当 isa 中无法完整保存引用计数时，系统会把部分引用计数转移到 SideTable 的 `refcnts` 中维护。
- 🔒 SideTable 内部包含自旋锁，用于保证多线程环境下对引用计数表和弱引用表操作的线程安全。
- ⚖️ Apple 没有为所有对象只使用一张全局 SideTable 表，也没有为每个对象单独创dd建独立容器，而是通过 StripMap 将 SideTable 分散到固定数量的容器中，以平衡性能和内存消耗。
- 🧭 真机环境下 StripMap 通常有 8 个容器，模拟器环境下通常有 64 个容器，用于存放带锁的数据结构。
- 🪝 `weak_table` 是弱引用表，本质上是一个哈希结构，用对象地址作为 key，保存指向该对象的所有 weak 指针地址。
- 📦 一个对象可以被多个 weak 指针引用，因此 `weak_entry_t` 中保存的是 weak 指针地址数组，而不是单个 weak 指针。
- 🧮 当弱引用数量较少时，`weak_entry_t` 会使用内联数组保存 weak 指针地址；当数量超过阈值时，会切换到动态哈希数组，以兼顾性能和内存占用。
- 🔁 `storeWeak` 的核心流程是：如果 weak 指针之前指向过旧对象，就从旧对象的弱引用表中移除该 weak 指针地址；如果它现在指向新对象，就把该 weak 指针地址注册到新对象的弱引用表中。
- 🧹 `weak_unregister_no_lock` 负责从弱引用表中移除 weak 指针地址，如果某个 `weak_entry_t` 已经没有任何 weak 指针地址，就会进一步从 weak_table 中删除该 entry。
- ➕ `weak_register_no_lock` 负责向对象对应的 weak_table 中添加 weak 指针地址，如果找不到对应的 `weak_entry_t`，就会新建一个 entry。
- 🚫 `weak` 修饰的对象不会真正增加对象引用计数，它只是让 weak 指针地址被记录到对象对应的弱引用表中。
- 🔍 打印 weak 对象引用计数时出现 2，是因为底层调用 `objc_loadWeakRetained` 时会临时 retain 一次对象；该 retain 只存在于函数作用域内，函数结束后会释放，所以不会影响对象真实生命周期。
- 🧯 对象释放时会清理它对应的弱引用表，并将相关 weak 指针置空，从而避免野指针访问。
- 🪣 `autoreleasepool` 是 Objective-C 的一种延迟释放机制，手动创建的自动释放池会在离开作用域时释放池内对象。
- 🚀 在循环中使用 `@autoreleasepool` 可以及时释放临时对象，降低内存峰值，常见于图片处理等大量临时对象创建的场景。
- ❌ 课程中特别澄清：使用 `weak` 修饰对象并不意味着对象一定会被加入自动释放池，这种说法在当前语境下是不准确的。


[#iOS内存管理](https://bibigpt.co/search?q=iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86) [#weak底层原理](https://bibigpt.co/search?q=weak%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86) [#SideTable](https://bibigpt.co/search?q=SideTable) [#weak_table](https://bibigpt.co/search?q=weak_table) [#autoreleasepool](https://bibigpt.co/search?q=autoreleasepool)


### 思考 
1. **[为什么 `weak` 不增加引用计数，但打印 weak 对象引用计数时可能是 2？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%20%60weak%60%20%E4%B8%8D%E5%A2%9E%E5%8A%A0%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%EF%BC%8C%E4%BD%86%E6%89%93%E5%8D%B0%20weak%20%E5%AF%B9%E8%B1%A1%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%97%B6%E5%8F%AF%E8%83%BD%E6%98%AF%202%EF%BC%9F)**
  - 因为打印 weak 对象引用计数时，底层可能调用 `objc_loadWeakRetained`，该函数会临时 retain 对象以保证读取期间对象安全，因此看到的引用计数可能临时变成 2。但这个 retain 是局部性的，函数结束后会 release，所以 weak 本身并不会真正增加对象引用计数。
- 2. **[weak 指针为什么可以在对象释放后自动变成 nil？](https://bibigpt.co/search?q=weak%20%E6%8C%87%E9%92%88%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8F%AF%E4%BB%A5%E5%9C%A8%E5%AF%B9%E8%B1%A1%E9%87%8A%E6%94%BE%E5%90%8E%E8%87%AA%E5%8A%A8%E5%8F%98%E6%88%90%20nil%EF%BC%9F)**
  - 因为对象被 weak 引用时，系统会把 weak 指针的地址保存到对象对应的 `weak_table` 中。当对象释放时，运行时会根据弱引用表找到所有指向该对象的 weak 指针地址，并将这些指针清空，从而实现自动置 nil。
- 3. **[为什么系统需要多个 SideTable 容器，而不是只用一张全局表？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%E7%B3%BB%E7%BB%9F%E9%9C%80%E8%A6%81%E5%A4%9A%E4%B8%AA%20SideTable%20%E5%AE%B9%E5%99%A8%EF%BC%8C%E8%80%8C%E4%B8%8D%E6%98%AF%E5%8F%AA%E7%94%A8%E4%B8%80%E5%BC%A0%E5%85%A8%E5%B1%80%E8%A1%A8%EF%BC%9F)**
  - 如果只用一张全局表，多个对象同时操作引用计数或弱引用信息时会频繁竞争同一把锁，性能较低。如果为每个对象都创建独立表，又会浪费内存。使用固定数量的容器进行分散存储，可以在性能和内存之间取得平衡。


### 术语解释
- **SideTable**: Objective-C 运行时中的辅助数据结构，用于存储对象在 isa 中放不下的信息，例如额外引用计数和弱引用表等。
- **weak_table**: 弱引用表，用于记录某个对象被哪些 weak 指针引用。对象释放时，系统会通过它找到并清空相关 weak 指针。
- **weak_entry_t**: weak_table 中的条目，通常以对象地址为 key，内部保存所有指向该对象的 weak 指针地址。
- **storeWeak**: Runtime 中处理 weak 赋值的核心函数，负责旧弱引用关系的移除、新弱引用关系的注册，以及对象 weak 标记位的更新。
- **autoreleasepool**: 自动释放池，用于延迟释放对象。手动创建的 `@autoreleasepool` 会在作用域结束时释放池内对象，常用于控制循环中的临时对象内存峰值。

---

## 视频章节总结 ｜ iOS底层原理：Weak引用的底层实现与内存管理深度解析

本课程深入解析了 iOS 中 weak 引用的底层实现原理与内存管理机制。从 SideTable 的结构出发，介绍了其内部的锁、引用计数映射表和弱引用表。通过 StripMap 的分表缓存设计解决了全局表导致的锁竞争问题，从而在性能与内存间取得平衡。重点剖析了 weak 条目（weak_entry_t）的联合体数据结构，解释了当弱引用个数不超过 4 时使用内联数组、超过时则采用动态哈希数组的存储策略，以节省内存。详细梳理了 storeWeak 函数的完整流程：获取新旧 SideTable、移除旧弱引用、注册新弱引用并设置 isa 标志位。同时对比了 weak_register_no_lock 与 weak_unregister_no_lock 两个核心函数，展示了弱引用地址在哈希数组中的插入与移除逻辑。课程还通过源码和汇编调试，揭示了打印 weak 指针时引用计数临时为 2 的假象，强调其并不真正增加对象引用计数，仅因函数内临时 retain 所致。最后初步讲解了自动释放池的机制，区分了手动和自动添加场景，并通过示例演示了在循环中使用 @autoreleasepool 控制内存峰值的实践价值。

### [00:00](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=0.000) - 📝 课程开场与内存管理回顾
讲师开场后快速回顾了上节课关于 iOS 三种内存管理方案的内容：Tagged Pointer 用于小对象、Nonpointer isa 在指针中存储引用计数等元信息、以及 SideTable 结构。指出 SideTable 中除了引用计数表外还包含弱引用表，为本次 weak 专题做了铺垫。通过源码中的 rootRetain 调用引出当 isa 指针存不下引用计数时会转向 SideTable 存储一半计数的机制。

### [09:08](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=548.000) - 🔐 SideTable 结构与 StripMap 并发优化
深入分析 SideTable 的数据结构：一把 spinlock 锁、一个 RefcountMap 用于存放额外的引用计数、以及一个 WeakTable 弱引用表。指出每次操作 SideTable 都需要加锁解锁，若全局只有一张表则性能低下。苹果通过 StripMap 来管理带有锁的结构体，真机提供 8 张表、模拟器 64 张表，利用 indexOfPointer 函数将不同对象的 SideTable 均匀分配到各张表，既降低了加锁竞争，又避免了内存浪费，达到性能与内存的平衡。

### [19:04](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=1144.000) - ⚙️ storeWeak 函数全流程分析
当用 __weak 修饰对象时底层调用 objc_initWeak，进而调用 storeWeak 函数。storeWeak 首先通过新旧对象的地址获取对应的 oldSideTable 和 newSideTable，若该 weak 指针之前已指向其他对象，则调用 weak_unregister_no_lock 从旧对象的弱引用表中移除该指针地址；接着将新对象的弱引用表中注册该 weak 指针地址（weak_register_no_lock），最后调用 setWeaklyReferenced_nolock 将 isa 中的 weakly referenced 标志位置为 TRUE，表示该对象曾被弱引用。整个过程通过加锁保证线程安全，实现了 weak 指针引用的正确切换。

### [39:55](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=2395.000) - 📊 Weak 表的数据结构与存储策略
对象对应的弱引用表是一个哈希结构，以对象地址为 key，value 是一个 weak_entry_t 结构。weak_entry_t 采用联合体实现：当该对象被弱引用的个数不超过 4 时，使用固定大小的内联数组 inline_referrers 直接存储 weak 指针地址；一旦超过 4 个，则切换为动态分配的可扩容哈希数组 referent。这种设计避免了为大量只被少数弱引用指向的对象预分配大块内存，显著节省内存。课程还点明了 mask 和 max_hash_displacement 等字段的作用是解决哈希冲突。

### [47:58](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=2878.000) - 🔧 弱引用的注册与移除核心机制
weak_register_no_lock 负责向 weak_entry_t 中插入 weak 指针地址：先从对象对应的 SideTable 获取 weak_entry，若哈希数组中已存在则不重复添加，否则进行扩容并插入；若不存在该对象的 weak_entry 则新建一个。weak_unregister_no_lock 则负责移除：通过对象找到 weak_entry，在哈希数组中移除指定 weak 指针地址，移除后若 entry 内部数组为空，则将整个 weak_entry 从 SideTable 中删除，释放相应内存。这两个函数共同维护了对象弱引用关系的准确性，并被 dealloc 流程调用以清理所有弱引用。

### [01:22:43](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=4963.000) - ❓ Weak 指针引用计数为什么会是 2？
通过汇编追踪发现，直接打印 __weak 指针指向对象的引用计数时，底层调用的是 objc_loadWeakRetained 而非普通的 retainCount。该函数为了安全访问弱引用对象，会在其作用域内临时对对象执行 rootTryRetain 操作，使引用计数暂时加 1，因此打印出 2；当函数返回、临时变量出作用域后，会自动调用 release 将计数减回 1。这澄清了“weak 会增加引用计数”的误解，强调 weak 本身并不影响对象的真实引用计数，打印结果只是一个局部假象。

### [01:40:23](https://bibigpt.co/content/6e0b9e4f-19a8-4523-acb4-9b27e94eafcc?t=6023.000) - 🧪 自动释放池初探与内存峰值优化
课程初步引入自动释放池概念：通过 alloc/new/copy 等方法创建的对象并不会被自动加入自动释放池，除非显式添加 __autoreleasing 或使用 @autoreleasepool。在实际编码中，如果在循环中大量创建临时对象（如字符串拼接）而不使用 @autoreleasepool，内存会持续飙升；将循环体包裹在 @autoreleasepool 块中则能在每次迭代结束时及时释放临时对象，平滑内存峰值。此外还澄清了手动创建的 autoreleasepool 其释放时机仅取决于作用域结束，与 runloop 无关，而系统自动加入的则与 runloop 的迭代相关，这一点会在后续课程中详细展开。最后课程在答疑与轻松讨论中结束，鼓励学员保持自信、继续深耕底层技术。