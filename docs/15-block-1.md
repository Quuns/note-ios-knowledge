# 【BibiGPT】AI 一键总结：[387702301452511873_第十五节课- block（上）【一手优质IT资源微信x923713】(1).mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMzg3NzAyMzAxNDUyNTExODczX-esrOWNgeS6lOiKguivvi0gYmxvY2vvvIjkuIrvvInjgJDkuIDmiYvkvJjotKhJVOi1hOa6kOW-ruS_oXg5MjM3MTPjgJEoMSktbXA0LWVuY29kZWQ%253D.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260509%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260509T145226Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D31d8e12e5ddcc47f5570bbf4f7bac809ec06f8a240983ee8be33cf61b06fad28%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

![](https://bibigpt-apps.chatvid.ai/thumbnails/1736405a-8ac6-4e3d-86b6-73ebab08d5e3/Mzg3NzAyMzAxNDUyNTExODczX-esrOWNgeS6lOiKguivvi0gYmxvY2vkuIrkuIDmiYvkvJjotKhJVOi1hOa6kOW-ruS_oXg5MjM3MTMx.jpg)


## 摘要
本节课围绕 iOS 底层中的 `@synchronized` 和 Objective-C `block` 展开。前半部分通过编译重写和源码分析，讲解 `@synchronized` 底层会调用 `objc_sync_enter` 与 `objc_sync_exit`，并借助 `SyncData`、线程缓存、递归锁和全局表来实现多线程同步与递归调用。后半部分开始分析 block 的三种类型、copy 行为、引用计数变化以及循环引用问题，并强调课后需要通过源码、笔记和博客反复消化。


### 亮点
- 🔒 `@synchronized` 编译后核心会转换为 `objc_sync_enter` 和 `objc_sync_exit` 两个底层函数调用。
- 🧩 如果 `@synchronized(obj)` 中传入的对象为 `nil`，底层会直接返回，相当于这把锁没有任何效果。
- 🧠 `@synchronized` 的底层同步依赖 `SyncData` 结构，它内部保存了被锁对象、线程使用计数以及一把递归锁。
- 🔁 `@synchronized` 能支持递归调用，是因为底层使用了递归锁，并且会记录当前线程对同步代码块的加锁次数。
- 🧵 底层会利用 TLS（线程局部存储）缓存与当前线程相关的同步数据，从而减少每次都查全局表的成本。
- ⚡ `id2data` 的查找流程大致分为快速缓存、线程缓存、全局表查找以及首次创建 `SyncData` 四个阶段。
- 🗂️ `StripedMap` 是一个全局分片表结构，真机和模拟器上的分片数量不同，用于在性能和内存消耗之间做折中。
- 📊 `threadCount` 用来记录有多少线程正在使用某个同步对象对应的 `SyncData`，而 `lockCount` 用来记录当前线程递归加锁的次数。
- 🧱 block 按内存位置和捕获行为可以分为全局 block、栈 block 和堆 block 三种类型。
- 🌍 没有捕获外部局部变量，或者只使用静态变量、全局变量的 block，通常会是全局 block。
- 📦 捕获外部局部变量的 block 初始创建时通常在栈上，如果被强引用或 copy，就会被拷贝到堆上。
- 🧬 block 的本质是一个结构体，编译重写后可以看到它会保存函数指针、描述信息以及捕获的外部变量。
- 📈 栈 block 在捕获对象时会影响对象引用计数，拷贝到堆上时还会对捕获的对象再进行一次持有。
- 🧪 示例中对象引用计数出现 `1、3、4、5` 等变化，核心原因是 block 捕获对象和 block copy 时都会影响对象生命周期。
- ♻️ 对堆 block 再 copy 时，通常只是增加 block 自身的引用计数，不会再次增加其捕获对象的引用计数。
- ⚠️ block 即使没有被执行，只要在定义时捕获了 `self`，也可能形成 `self -> block -> self` 的循环引用。
- 🧭 使用 `_ivar` 访问成员变量时，block 底层仍然可能捕获 `self`，因此仍然需要警惕循环引用。
- 📝 课程最后强调，底层源码内容需要反复看录播、写博客和整理笔记，不能只听一遍就结束。


[#iOS底层](https://bibigpt.co/search?q=iOS%E5%BA%95%E5%B1%82) [#ObjectiveC](https://bibigpt.co/search?q=ObjectiveC) [#多线程锁](https://bibigpt.co/search?q=%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%94%81) [#Block](https://bibigpt.co/search?q=Block) [#内存管理](https://bibigpt.co/search?q=%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86) [#源码分析](https://bibigpt.co/search?q=%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)


### 思考 
1. **[`@synchronized` 为什么可以支持递归调用？](https://bibigpt.co/search?q=%60%40synchronized%60%20%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8F%AF%E4%BB%A5%E6%94%AF%E6%8C%81%E9%80%92%E5%BD%92%E8%B0%83%E7%94%A8%EF%BC%9F)**
  - 因为它底层并不是简单地使用一把普通互斥锁，而是通过 `SyncData` 中的递归锁来完成加锁和解锁，并通过线程相关的缓存结构记录当前线程的加锁次数。因此同一线程可以多次进入同一个同步代码块，而不会发生自锁死。
- 2. **[block 为什么通常建议用 `copy` 修饰？](https://bibigpt.co/search?q=block%20%E4%B8%BA%E4%BB%80%E4%B9%88%E9%80%9A%E5%B8%B8%E5%BB%BA%E8%AE%AE%E7%94%A8%20%60copy%60%20%E4%BF%AE%E9%A5%B0%EF%BC%9F)**
  - 因为捕获外部局部变量的 block 初始可能位于栈上，而栈上的内存生命周期不稳定，出了作用域后可能被系统回收。使用 `copy` 可以把栈 block 拷贝到堆上，延长其生命周期，避免后续调用时出现野指针或崩溃。在 ARC 下，很多情况下 `strong` 也会触发类似的拷贝效果，但语义上仍推荐使用 `copy`。
- 3. **[block 没有执行，为什么也可能产生循环引用？](https://bibigpt.co/search?q=block%20%E6%B2%A1%E6%9C%89%E6%89%A7%E8%A1%8C%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B9%9F%E5%8F%AF%E8%83%BD%E4%BA%A7%E7%94%9F%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8%EF%BC%9F)**
  - block 捕获外部变量是在定义或赋值阶段发生的，并不需要等到 block 被调用。如果 block 内部使用了 `self`，而 `self` 又强引用这个 block，就会形成 `self` 强持有 block、block 强持有 `self` 的闭环，从而导致对象无法释放。


### 术语解释
- **`@synchronized`**: Objective-C 提供的同步语法，用于对某个对象关联的同步代码块加锁。底层会调用 `objc_sync_enter` 和 `objc_sync_exit`。
- **`SyncData`**: `@synchronized` 底层使用的核心数据结构，保存被锁对象、递归锁、线程使用数量等信息，是理解其同步机制的关键。
- **TLS（Thread Local Storage）**: 线程局部存储，每条线程独有的一块存储空间，可用于缓存当前线程相关的数据，减少全局查找开销。
- **`StripedMap`**: 分片哈希表结构，用于把同步对象分散存储到多个表中，在查找效率和内存消耗之间取得平衡。
- **Block**: Objective-C 中的闭包结构，本质上可理解为一个结构体，能够保存代码逻辑和捕获的外部变量，常见类型包括全局 block、栈 block 和堆 block。

---

## 视频章节总结 ｜ iOS底层进阶：@synchronized底层锁机制与Block内存管理详解

本节课程主要深入解析了iOS底层技术中的两个核心主题：@synchronized锁的底层实现原理以及Block的类型与内存管理。讲师通过拆解源码，详细阐述了@synchronized如何通过哈希表管理数据、利用线程局部存储（TLS）实现多线程下的递归调用，并解释了为何真机性能优于模拟器。随后，课程转向Block的学习，分析了三种Block类型（栈、堆、全局），演示了如何通过编译观察其结构体本质，并深入讨论了循环引用的成因与解决方案。整节课通过丰富的案例与源码调试，帮助开发者打通底层知识链路，提升对内存管理与并发处理的认知。

### [00:00](https://bibigpt.co/content/aab59e9c-61ca-4378-ad02-6f798834a298?t=0.000) - 🔒 @synchronized锁的底层运行机制

![章节截图 00:00](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_9428177975420191464%5Cscreenshot_0.0s.jpg)

本章节重点分析了@synchronized的底层代码结构。讲师通过编译后的源码演示，指出其核心在于调用了objc_sync_enter和objc_sync_exit函数，并利用结构体构造与析构函数进行管理。重点解释了传入参数为空时的处理逻辑，以及依赖SyncData数据结构进行加解锁操作的过程，为理解后续的多线程递归机制奠定了基础。

### [35:12](https://bibigpt.co/content/aab59e9c-61ca-4378-ad02-6f798834a298?t=2112.000) - ⚙️ 多线程递归调用的底层逻辑

![章节截图 35:12](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_9428177975420191464%5Cscreenshot_2112.0s.jpg)

本章节探讨了@synchronized为何能实现多线程递归调用。通过分析SyncData单向链表和StripedMap哈希表，讲师解释了系统如何根据对象哈希值在多张表中分配管理，并利用TLS（线程局部存储）缓存每个线程的递归次数。这种设计确保了同一线程在递归调用时使用同一把递归锁，从而实现了并发安全下的递归功能。

### [01:42:54](https://bibigpt.co/content/aab59e9c-61ca-4378-ad02-6f798834a298?t=6174.000) - 🧱 Block的三种类型与本质

![章节截图 01:42:54](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_9428177975420191464%5Cscreenshot_6174.0s.jpg)

本章节从Block的分类入手，详细介绍了全局Block、堆Block和栈Block。讲师通过clang编译后的代码演示，清晰地揭示了Block的本质就是一个结构体。通过两条判定原则——是否捕获外部变量以及是否赋值给强/弱引用，指导学员快速判断Block的类型，并说明了为什么通常使用copy修饰符将栈上的Block复制到堆上以防止提前释放。

### [01:55:32](https://bibigpt.co/content/aab59e9c-61ca-4378-ad02-6f798834a298?t=6932.000) - 🔄 Block内存管理与循环引用探秘

![章节截图 01:55:32](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_9428177975420191464%5Cscreenshot_6932.0s.jpg)

本章节深入分析了Block在捕获外部变量时的内存变化。讲师通过具体的引用计数对比案例，演示了从栈拷贝到堆的过程如何导致引用计数增加。同时，针对面试中高频出现的循环引用问题，讲师通过实操演示了Block强引用self导致的内存泄漏，并讨论了如何利用弱引用等手段规避这一问题，深化了对内存管理周期的理解。

### [02:22:30](https://bibigpt.co/content/aab59e9c-61ca-4378-ad02-6f798834a298?t=8550.000) - 💡 课程答疑与内存管理实践

![章节截图 02:22:30](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_9428177975420191464%5Cscreenshot_8550.0s.jpg)

在课程的最后阶段，讲师针对学员提出的关于内存管理、栈区与堆区变量销毁机制进行了集中答疑。通过一段实操代码，演示了栈上变量在出作用域后内存地址的变化情况，并对比了堆区对象在ARC模式下的自动释放逻辑。讲师强调了阅读经典技术书籍《程序员的自我修养》以及通过手动编写博客来巩固学习成果的重要性。