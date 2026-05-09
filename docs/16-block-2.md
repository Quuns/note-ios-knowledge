# 【BibiGPT】AI 一键总结：[387702301528577045_block（上）【一手优质IT资源微信x923713】(1).mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMzg3NzAyMzAxNTI4NTc3MDQ1X2Jsb2Nr77yI5LiK77yJ44CQ5LiA5omL5LyY6LSoSVTotYTmupDlvq7kv6F4OTIzNzEz44CRKDEpLW1wNC1lbmNvZGVk.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260509%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260509T145331Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D6bf02e83e21182a3fb1ffeddf73800710f1dd5d76168b67a202df9fbd280253a%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

![](https://bibigpt-apps.chatvid.ai/thumbnails/1736405a-8ac6-4e3d-86b6-73ebab08d5e3/Mzg3NzAyMzAxNTI4NTc3MDQ1X2Jsb2Nr5LiK5LiA5omL5LyY6LSoSVTotYTmupDlvq7kv6F4OTIzNzEzMQ==.jpg)


## 摘要
本节课主要围绕 Objective-C 中 Block 的底层原理、内存管理与循环引用展开。我先回顾了 Block 的类型、捕获变量与 copy 行为，然后重点讲解了 Block 捕获对象时为什么会产生循环引用，以及如何通过 weak-strong dance、__block、参数传递等方式解决。后半部分进一步分析了 __block 的底层结构、forwarding 指针、Block 的源码结构、invoke 函数指针、descriptor、copy/dispose 函数，以及多个面试题场景，帮助大家从源码和内存模型角度理解 Block 的真实行为。


### 亮点
- 🧱 Block 在 Objective-C 中本质上是一个对象，底层可以理解为包含 isa、flags、invoke、descriptor 等成员的结构体。
- 📦 Block 创建时通常先位于栈上，当被强引用或 copy 时，会从栈 Block 拷贝到堆 Block。
- 🔁 Block 捕获外部对象时通常是浅拷贝，本质上是保存对象指针，并增加对象的引用计数。
- ⚠️ 在 Block 内部无论使用 `self.property` 还是 `_ivar` 访问成员变量，都会捕获 `self`，因此都可能造成循环引用。
- 🧩 循环引用的根本原因是 `self` 强持有 Block，而 Block 内部又强持有 `self`，导致双方都无法释放。
- 🕊️ 使用 `__weak typeof(self) weakSelf = self;` 可以打破 Block 对 `self` 的强引用，从而避免循环引用。
- 💪 weak-strong dance 的作用是在 Block 执行期间临时强持有 `self`，既避免循环引用，又防止异步执行时对象提前释放。
- 🧹 使用 `__block` 修饰对象变量也可以解决循环引用，但使用后需要在 Block 内部手动将变量置为 `nil`，否则仍可能泄露。
- 🎯 将 `self` 作为 Block 参数传入，也可以避免 Block 捕获外部的 `self`，因为 Block 不会捕获自己的参数。
- 🧠 静态变量和全局变量可以在 Block 内部修改，因为全局变量不需要捕获，静态变量捕获的是地址。
- 🔧 普通局部变量默认不能在 Block 内部修改，因为 Block 捕获的是值拷贝，而不是原变量本身。
- 🧬 使用 `__block` 修饰局部变量后，编译器会将其包装成一个 `__Block_byref` 结构体，并通过 forwarding 指针统一访问。
- 🔀 当 `__block` 变量随 Block 从栈拷贝到堆时，栈上的 forwarding 指针会指向堆上的结构体，保证访问的是同一份逻辑数据。
- 🧾 Block 的 descriptor 中保存了 Block 的大小、签名、copy 函数、dispose 函数等信息。
- 🛠️ copy 和 dispose 函数主要用于管理 Block 捕获的对象；如果只捕获基本类型，通常不会生成这两个函数。
- 🧨 如果手动修改 Block 底层结构中的 invoke 函数指针为 `nil`，再调用 Block 就会崩溃，因为实现入口已经不存在。
- 🧪 栈 Block、堆 Block、全局 Block 的生命周期不同，面试题常通过 weak、copy、strong 的组合考察对生命周期的理解。
- 📚 本节课也穿插讨论了 GCD、信号量、栅栏函数、KVC、KVO、组件化、工程化和学习路线等扩展问题。


[#Block](https://bibigpt.co/search?q=Block) [#ObjectiveC](https://bibigpt.co/search?q=ObjectiveC) [#循环引用](https://bibigpt.co/search?q=%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8) [#内存管理](https://bibigpt.co/search?q=%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86) [#__block](https://bibigpt.co/search?q=__block) [#weakStrongDance](https://bibigpt.co/search?q=weakStrongDance) [#源码分析](https://bibigpt.co/search?q=%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90) [#iOS底层](https://bibigpt.co/search?q=iOS%E5%BA%95%E5%B1%82)


### 思考 
1. **[为什么在 Block 内部使用 `_name` 访问成员变量也会造成循环引用？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%E5%9C%A8%20Block%20%E5%86%85%E9%83%A8%E4%BD%BF%E7%94%A8%20%60_name%60%20%E8%AE%BF%E9%97%AE%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F%E4%B9%9F%E4%BC%9A%E9%80%A0%E6%88%90%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8%EF%BC%9F)**
  - 因为 `_name` 本质上仍然是通过当前对象的成员变量访问，Block 为了访问这个成员变量会捕获 `self`。只要 `self` 强持有 Block，而 Block 又强持有 `self`，就会形成循环引用。
- 2. **[weak-strong dance 为什么比单纯使用 weakSelf 更安全？](https://bibigpt.co/search?q=weak-strong%20dance%20%E4%B8%BA%E4%BB%80%E4%B9%88%E6%AF%94%E5%8D%95%E7%BA%AF%E4%BD%BF%E7%94%A8%20weakSelf%20%E6%9B%B4%E5%AE%89%E5%85%A8%EF%BC%9F)**
  - 单纯使用 weakSelf 可以避免循环引用，但如果 Block 是异步执行，对象可能在执行前已经释放，导致 weakSelf 为 nil。weak-strong dance 会在 Block 执行期间用 strongSelf 临时强持有对象，保证代码执行过程中对象不被提前释放。
- 3. **[`__block` 为什么能让局部变量在 Block 内部被修改？](https://bibigpt.co/search?q=%60__block%60%20%E4%B8%BA%E4%BB%80%E4%B9%88%E8%83%BD%E8%AE%A9%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F%E5%9C%A8%20Block%20%E5%86%85%E9%83%A8%E8%A2%AB%E4%BF%AE%E6%94%B9%EF%BC%9F)**
  - 因为编译器会把 `__block` 修饰的变量包装成一个 byref 结构体，并通过 forwarding 指针访问。当 Block 从栈拷贝到堆时，栈结构体和堆结构体会通过 forwarding 关联起来，所以 Block 内部修改的不是普通值拷贝，而是同一个逻辑变量。


### 术语解释
- **Block**: Objective-C 中的闭包结构，本质是一个对象，内部保存代码实现函数指针、捕获变量和描述信息，可用于回调、异步任务和函数式封装。
- **循环引用**: 两个或多个对象相互强持有，导致引用计数无法归零，从而无法释放内存，常见于 `self` 强持有 Block、Block 又捕获 `self` 的场景。
- **weak-strong dance**: 解决 Block 循环引用的常见写法，先用 weakSelf 避免强引用，再在 Block 内部用 strongSelf 临时强持有对象，兼顾安全性和内存释放。
- **`__block`**: Objective-C 中用于修饰变量的关键字，可让局部变量在 Block 内部被修改。底层会生成 byref 结构体，并借助 forwarding 指针统一访问。
- **Block descriptor**: Block 底层结构中的描述信息区域，用于存储 Block 大小、签名、copy/dispose 辅助函数等元数据。

---

## 视频章节总结 ｜ 深入剖析Block底层原理：内存管理与循环引用实战详解

本视频是逻辑教育底层项目进阶班第16天的课程回放，重点讲解了Block的底层内存管理与循环引用处理机制。讲师通过源码分析，详细解析了Block的类型、内存拷贝（Block Copy）原理，以及在面试中常见的循环引用问题。课程不仅深入探讨了如何通过weak-strong dance等技巧打破Block循环引用，还分析了Block底层的数据结构（如forwarding指针与内存布局），并解答了关于静态变量及局部变量捕获的区别。最后，讲师对iOS职业规划与求职心态给出了建议，强调了持续学习和多元化技能积累的重要性。

### [00:00](https://bibigpt.co/content/977d8afc-4526-4c2a-9e66-a056e06b1eaa?t=0.000) - 💡 Block 类型与内存拷贝机制
本章回顾了Block的分类，重点解释了Block创建时默认为栈Block，在被强引用赋值时会自动拷贝至堆上的机制。讲师通过源码分析指出，该拷贝过程本质上是浅拷贝，并详细说明了为什么Block在拷贝时会将捕获的成员变量引用计数加1，从而引发内存管理的复杂性。这一部分为后续处理循环引用提供了理论基础。

### [13:24](https://bibigpt.co/content/977d8afc-4526-4c2a-9e66-a056e06b1eaa?t=804.000) - 🔄 Block 循环引用与解决方案
课程深入讨论了Block导致内存泄漏的核心原因，即Block与self之间的相互强引用。讲师演示了即使使用下划线访问成员变量也会触发循环引用的实验，并引入了weak-strong dance（强弱共舞）作为解决方案。通过分析延时任务场景，解释了为何单纯使用weak引用在异步任务中会导致对象提前销毁，以及引入strong引用为何能确保Block执行期间的安全。

### [49:18](https://bibigpt.co/content/977d8afc-4526-4c2a-9e66-a056e06b1eaa?t=2958.000) - ⚙️ Block 变量捕获与 __block 原理
本章探讨了Block为何能够修改静态变量和全局变量，而无法直接修改局部变量的原因。讲师通过编译后的C++代码揭示了__block修饰符的底层实现：它将变量包装成了一个包含forwarding指针的结构体。该结构体通过forwarding指针将栈上和堆上的数据链接起来，从而确保了在Block内部能够正确地读写和同步变量值。

### [01:33:12](https://bibigpt.co/content/977d8afc-4526-4c2a-9e66-a056e06b1eaa?t=5592.000) - 🧬 Block 底层源码深度剖析
讲师详细拆解了Block在底层的结构体布局（Block Layout），包括isa指针、flags标志位以及invoke函数指针。通过调试指令和寄存器读取，展示了Block如何通过descript描述符来管理copy和dispose函数，以及它们在捕获对象时是如何进行内存管理的。此外，还探讨了Block签名（signature）的生成逻辑，帮助学员更直观地理解Block的本质。

### [01:51:36](https://bibigpt.co/content/977d8afc-4526-4c2a-9e66-a056e06b1eaa?t=6696.000) - 🚀 实战演练与职业发展建议
通过多个实战面试题（如野指针崩溃、变量修改等），巩固了学员对Block执行逻辑的理解。课程最后，讲师针对当前iOS求职环境进行了客观分析，鼓励学员不要过于焦虑市场寒冬，应注重多元化技术栈的积累，如学习跨平台框架等。讲师强调，技术是持续积累的过程，保持自信和竞争力是应对职场挑战的关键。