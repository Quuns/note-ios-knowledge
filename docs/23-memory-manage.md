# 【BibiGPT】AI 一键总结：[23_内存管理.mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMjNf5YaF5a2Y566h55CGLW1wNC1lbmNvZGVk.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260505%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260505T045136Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3Dc0cab71c5d92a95b78f10cb01d2c84452d85a385e359ca8438410602dbe8f7cf%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

## 摘要
本节课围绕 iOS 内存管理展开，重点讲解值类型与引用类型的存储差异、内存五大区、ARC 下引用计数的自动管理机制，以及 iOS 中常见的内存优化方案：Non-pointer isa、Tagged Pointer 和 SideTable。课程还深入分析了 `retain`、`release`、`dealloc` 的底层流程，并结合字符串、NSNumber、多线程崩溃案例说明 Tagged Pointer 在性能和内存占用上的优势。


### 亮点
- 🧠 iOS 中需要重点管理的是引用类型对象，而不是 `int`、`double` 等值类型。
- 🧱 值类型通常存储在栈区，由系统自动管理；引用类型对象通常存储在堆区，需要通过引用计数维护生命周期。
- 🗂️ 内存五大区包括堆区、栈区、全局区或静态区、常量区和代码区，其中主要需要程序员关注的是堆区。
- ⚡ 栈区遵循先进后出原则，访问速度较快，但空间有限，主线程栈空间通常约为 1MB，其他线程通常约为 512KB。
- 💥 如果在栈上声明过大的局部变量，可能导致栈溢出，从而引发程序崩溃。
- 🔁 ARC 本质上是编译器在合适的位置自动插入 `retain`、`release`、`autorelease` 等引用计数管理代码。
- 🧩 引用计数用于管理对象生命周期，当新创建的 Objective-C 对象引用计数为 1，引用计数减为 0 时对象会销毁并释放内存。
- 🪄 iOS 中的内存管理优化方案主要包括 Non-pointer isa、Tagged Pointer 和 SideTable。
- 🧬 Non-pointer isa 利用 `isa` 指针中未完全使用的位来存储引用计数、是否正在释放、是否有关联对象等额外信息。
- 🏷️ Tagged Pointer 是针对小对象的优化方案，它不再把指针仅作为地址，而是直接把小对象的真实值编码进指针中。
- 🚀 Tagged Pointer 可以减少堆内存分配和释放成本，因此在小字符串、NSNumber、NSDate 等对象上能提升空间效率和访问速度。
- 🔍 通过 `stringWithFormat:` 创建较短且不含中文或特殊字符的字符串时，可能生成 Tagged Pointer 类型的 `NSString`。
- 🧮 Tagged Pointer 在不同架构上的位布局不同，Intel 和 ARM64 的标记位位置并不完全一致，因此业务代码不应依赖内部位运算判断。
- 🔐 Tagged Pointer 会和进程启动时初始化的随机值进行混淆，直接观察内存地址时可能看到看似异常的指针值。
- 🧪 多线程频繁读写普通堆对象时，可能因为重复释放同一块内存而崩溃；但 Tagged Pointer 不需要 ARC 管理引用计数，因此不会触发同类释放问题。
- ➕ `retain` 的底层流程会先判断对象是否为 Tagged Pointer，如果是则直接返回；否则会根据引用计数存储位置更新 `isa` 或 SideTable。
- 📦 当 Non-pointer isa 中的 extra reference count 存不下引用计数时，系统会将一部分引用计数转移到 SideTable 中保存。
- ➖ `release` 与 `retain` 的逻辑相反，它会减少引用计数；当引用计数减为 0 时，会触发 `dealloc` 流程。
- 🧹 `dealloc` 会处理成员变量释放、关联对象移除、弱引用表清理以及 SideTable 中引用计数信息的清理。
- 📚 即使 ARC 已经自动处理大部分内存管理问题，理解引用计数、Tagged Pointer、SideTable 等机制仍然是 iOS 底层面试的重要内容。


[#iOS内存管理](https://bibigpt.co/search?q=iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86) [#ARC](https://bibigpt.co/search?q=ARC) [#TaggedPointer](https://bibigpt.co/search?q=TaggedPointer) [#NonPointerIsa](https://bibigpt.co/search?q=NonPointerIsa) [#SideTable](https://bibigpt.co/search?q=SideTable)


### 思考 
1. **[为什么 ARC 时代还要学习引用计数和内存管理底层原理？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%20ARC%20%E6%97%B6%E4%BB%A3%E8%BF%98%E8%A6%81%E5%AD%A6%E4%B9%A0%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E5%92%8C%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%EF%BC%9F)**
  - 因为 ARC 只是把 `retain`、`release` 等操作交给编译器自动插入，并不代表底层机制不存在。理解引用计数、对象释放流程和内存优化方案，可以帮助定位崩溃、解释面试题，并更好地理解 Objective-C Runtime。
- 2. **[Tagged Pointer 为什么不需要像普通对象一样维护引用计数？](https://bibigpt.co/search?q=Tagged%20Pointer%20%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E9%9C%80%E8%A6%81%E5%83%8F%E6%99%AE%E9%80%9A%E5%AF%B9%E8%B1%A1%E4%B8%80%E6%A0%B7%E7%BB%B4%E6%8A%A4%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%EF%BC%9F)**
  - Tagged Pointer 的值直接编码在指针内部，通常不需要在堆上分配对象内存，也就不需要通过引用计数管理堆对象生命周期。因此对 Tagged Pointer 调用 `retain` 或 `release` 时，底层通常会直接返回。
- 3. **[SideTable 什么时候会参与引用计数管理？](https://bibigpt.co/search?q=SideTable%20%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E4%BC%9A%E5%8F%82%E4%B8%8E%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E7%AE%A1%E7%90%86%EF%BC%9F)**
  - 当对象不是 Non-pointer isa，或者 Non-pointer isa 中用于存储引用计数的空间不够时，系统会借助 SideTable 保存额外的引用计数信息。SideTable 还与弱引用表等 Runtime 数据结构相关。


### 术语解释
- **ARC（Automatic Reference Counting）**: 自动引用计数机制，由编译器在合适位置自动插入 `retain`、`release`、`autorelease` 等代码来管理 Objective-C 对象生命周期。
- **Non-pointer isa**: 对 `isa` 指针的优化方案，不只存储类对象地址，还利用部分位存储引用计数、是否正在释放、是否有关联对象等状态信息。
- **Tagged Pointer**: 针对小对象的指针优化方案，把对象的真实值直接编码进指针中，减少堆内存分配，提高访问和创建销毁效率。
- **SideTable**: Runtime 中用于辅助管理对象引用计数、弱引用等信息的数据结构，当 `isa` 中存不下或不适合存储时会使用它。
- **dealloc**: 对象引用计数归零后触发的销毁流程，负责释放对象占用的内存，并清理成员变量、关联对象、弱引用表和 SideTable 信息。

---

## 视频章节总结 ｜ iOS内存管理深度剖析：从内存布局到Tagged Pointer与引用计数原理

本视频是iOS底层进阶班第23天课程，深入讲解了iOS内存管理的核心原理。课程从内存五大区（栈、堆、全局/静态区、常量区、代码区）入手，解释了值类型与引用类型在内存中的不同存储方式，明确了堆区需要程序员管理的原因。随后重点介绍了三种内存管理方案：non‑pointer isa、tagged pointer和sidetable。通过源码分析和WWDC视频，详细演示了tagged pointer如何将小对象的数据直接存入指针，节省内存并提升访问速度，以及在ARM和Intel架构下的不同标记位和类型标签。接着，逐行剖析了retain、release和dealloc的底层实现，包括extra_rc与sidetable之间的引用计数分半存储机制，旨在减少锁竞争优化性能。最后，通过面试题和答疑，强化了对内存管理方案的理解，并强调了实践总结的重要性。

### [00:00](https://bibigpt.co/content/dab6db07-ed3e-4351-9c5a-b9dc1174a05e?t=0.000) - 📚 内存管理基础与五大区

![章节截图 00:00](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_10996703883590016464%5Cscreenshot_0.0s.jpg)

课程首先明确了iOS中需要管理的对象类型：值类型（如int、char）存储在栈上，由系统自动管理；引用类型（继承自NSObject的类对象）分配在堆上，需程序员管理。接着详细讲解了内存五大区：栈区存放局部变量、方法参数等，空间有限（主线程1MB），超量会导致崩溃；堆区存放alloc/new创建的对象，易产生内存碎片；全局/静态区和常量区在编译时分配，存全局变量和常量；代码区存二进制代码。通过分析地址开头特征（如堆区0x6，栈区0x7）和代码示例，强调了堆区才是内存管理的主战场。最后引出了ARC和引用计数的基本概念。

### [40:27](https://bibigpt.co/content/dab6db07-ed3e-4351-9c5a-b9dc1174a05e?t=2427.070) - 🏷️ Tagged Pointer 技术详解

![章节截图 40:27](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_10996703883590016464%5Cscreenshot_2427.1s.jpg)

本章系统介绍了iOS的三种内存优化方案，并重点剖析了Tagged Pointer。它是一种针对小对象（如短NSString、NSNumber等）的优化技术，将对象的值直接存储在指针中，避免堆内存分配。通过WWDC视频和代码演示，展示了Tagged Pointer在Intel（最低位为1）和ARM（最高位为1）架构下的标志位、3位类型标签（如3代表NSNumber、6代表NSDate）以及后续位存储有效载荷的结构。讲解了苹果为防止指针伪造引入的混淆机制，并演示了如何通过解码函数还原原始指针。以字符串和数字为例，详细说明了长度位、ASCII码或数值的存储方式，并强调其长度限制（通常≤9个字符）。最后结合多线程面试题，说明了Tagged Pointer因无需引用计数管理而不会出现过度释放崩溃的关键优势。

### [01:46:31](https://bibigpt.co/content/dab6db07-ed3e-4351-9c5a-b9dc1174a05e?t=6391.380) - 🔁 retain、release 与 dealloc 底层探秘

![章节截图 01:46:31](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_10996703883590016464%5Cscreenshot_6391.4s.jpg)

本章深入分析了引用计数管理三大核心方法的源码实现。Retain首先排除Tagged Pointer；对于non‑pointer isa，尝试在isa的extra_rc字段中直接递增引用计数；若超额，则将一半计数转存到sidetable中，并设置标志位，以减少锁操作。Release流程相反，先递减extra_rc，归零后若标志位指示sidetable有值，则从中取出另一半计数减一再写回extra_rc，避免频繁访问散列表。Dealloc首先清理弱引用表、移除关联对象，调用C++析构函数释放成员变量，最后释放对象内存；若无这些附加数据，则直接free。通过源码逐行解读，揭示了苹果如何在保证效率的前提下精准管理对象生命周期。

### [02:30:25](https://bibigpt.co/content/dab6db07-ed3e-4351-9c5a-b9dc1174a05e?t=9025.530) - 💡 答疑总结与课程收尾

![章节截图 02:30:25](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_10996703883590016464%5Cscreenshot_9025.5s.jpg)

在课程的最后阶段，讲师针对学员的疑问进行了集中解答，重点澄清了non‑pointer isa与tagged pointer的区别：前者是isa指针的空间复用，存储引用计数和标志位；后者则把数据直接编码进指针，省去对象头。强调了extra_rc的容量（arm64下19位，约52万）足以覆盖绝大多数场景，sidetable仅作后备。同时，总结了内存管理作为面试重灾区的应对思路，鼓励学员通过写博客深度理解，并分享了学习路线和行业经验。课程在轻松的交流中收尾，并预告了后续对sidetable和weak引用的深入讲解。