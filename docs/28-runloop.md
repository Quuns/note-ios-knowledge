# 【BibiGPT】AI 一键总结：[28_RunLoop底层原理.mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMjhfUnVuTG9vcOW6leWxguWOn-eQhi1tcDQtZW5jb2RlZA%253D%253D.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260505%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260505T051339Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D496d78241ae44f50c1f5589317794ae193ebd6056bb673749ced32bc3c7dd6b2%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)

## 摘要
本节课我围绕 iOS 中 RunLoop 的底层原理展开讲解，从实际开发中常见的点击事件、通知、NSTimer、GCD Block 等入口切入，说明 RunLoop 为什么“无时无刻不在”。随后我通过调用栈、源码结构、数据结构、Mode 与 Source/Timer/Observer 的关系，逐步解释 RunLoop 如何运行、休眠、唤醒和退出，并结合 NSTimer 示例帮助大家建立完整理解。


### 亮点
- 🔁 RunLoop 本质上是一个运行循环，但它不是普通的死循环，而是能够在有任务时处理事件、无任务时休眠以节省 CPU 资源的机制。
- 🧵 每一个线程都可以对应一个 RunLoop，RunLoop 与线程是一一绑定关系，主线程的 RunLoop 默认存在并运行，而子线程的 RunLoop 默认不会自动启动。
- 🧪 通过 NSTimer 的调用栈可以看到，Timer 的回调最终会经过 Core Foundation 中的 CFRunLoop 相关函数完成调度。
- 👆 UI 点击事件、通知响应、Timer、Perform Selector、GCD Block 等都可以被纳入 RunLoop 的事件处理体系。
- 🧱 RunLoop 在底层是一个结构体，内部维护了线程、Mode 集合、当前 Mode、Timer、Source、Observer 等关键成员。
- 🧭 Mode 是 RunLoop 运行时的场景模型，一个 RunLoop 可以有多个 Mode，但同一时刻只能运行在一个具体 Mode 中。
- 📦 Timer、Source 和 Observer 可以理解为 RunLoop 中的事件项，它们需要依附于某个 Mode 才能被 RunLoop 调度。
- 🕹️ Common Modes 不是一个具体 Mode，而是一组被标记为 common 的 Mode 集合，用于让事件能在多个模式下生效。
- ⚙️ Source0 主要处理应用内部主动触发的事件，例如 UI 事件；Source1 则与 Mach Port、内核消息和线程唤醒机制有关。
- 😴 RunLoop 的运行流程包括进入循环、处理 Observer、处理 Timer、处理 Source0、等待 Source1 消息、进入休眠、被唤醒以及退出循环。
- ⏱️ NSTimer 被添加到 RunLoop 时，本质上是被加入到某个 Mode 的 Timer 集合中，等 RunLoop 运行到对应阶段时再触发回调函数。
- 🚦 子线程中添加 Timer 后如果不手动启动 RunLoop，Timer 不会正常触发，因为子线程的 RunLoop 默认没有运行。
- 🔍 阅读源码时不需要死记硬背每一行代码，而要抓住核心结构，例如 CFRunLoopRun、CFRunLoopRunSpecific、DoTimer、DoSource 等关键节点。
- 🧠 理解 RunLoop 的核心意义，不只是为了面试，更是为了掌握卡顿检测、线程保活、事件响应、性能优化等高级开发能力。
- 📚 学习底层原理后需要通过写博客、做笔记和复盘题目来巩固，因为真正能讲出来、写出来，才代表自己真的理解了。


[#RunLoop](https://bibigpt.co/search?q=RunLoop) [#iOS底层原理](https://bibigpt.co/search?q=iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86) [#CFRunLoop](https://bibigpt.co/search?q=CFRunLoop) [#NSTimer](https://bibigpt.co/search?q=NSTimer) [#Source0Source1](https://bibigpt.co/search?q=Source0Source1) [#线程与事件循环](https://bibigpt.co/search?q=%E7%BA%BF%E7%A8%8B%E4%B8%8E%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF) [#性能优化](https://bibigpt.co/search?q=%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)


### 思考 
1. **[RunLoop 和普通 while 死循环有什么区别？](https://bibigpt.co/search?q=RunLoop%20%E5%92%8C%E6%99%AE%E9%80%9A%20while%20%E6%AD%BB%E5%BE%AA%E7%8E%AF%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F)**
  - 普通 while 死循环会持续占用 CPU，而 RunLoop 会在没有事件时进入休眠，等待事件、消息或超时唤醒后再继续处理任务，因此它既能保持线程存活，又能节省系统资源。
- 2. **[为什么子线程中的 NSTimer 有时不会执行？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%E5%AD%90%E7%BA%BF%E7%A8%8B%E4%B8%AD%E7%9A%84%20NSTimer%20%E6%9C%89%E6%97%B6%E4%B8%8D%E4%BC%9A%E6%89%A7%E8%A1%8C%EF%BC%9F)**
  - 因为子线程的 RunLoop 默认不会自动启动。NSTimer 被加入 RunLoop 后，需要手动调用 RunLoop 的运行方法，让 RunLoop 跑起来，Timer 才能在对应 Mode 中被调度执行。
- 3. **[Source0 和 Source1 的区别是什么？](https://bibigpt.co/search?q=Source0%20%E5%92%8C%20Source1%20%E7%9A%84%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F)**
  - Source0 通常处理 App 内部主动触发的事件，需要手动唤醒 RunLoop；Source1 基于 Mach Port 和内核消息机制，能够接收系统或其他线程发送的消息，并具备唤醒 RunLoop 的能力。


### 术语解释
- **RunLoop**: 运行循环机制，用于在线程中循环等待并处理事件。它可以让线程在有任务时工作、无任务时休眠，是 iOS 事件响应、Timer、线程保活和性能优化的重要基础。
- **CFRunLoop**: Core Foundation 层面的 RunLoop 实现，是 RunLoop 的底层 C 语言结构。NSRunLoop 可以理解为对 CFRunLoop 的 Objective-C 封装。
- **RunLoop Mode**: RunLoop 的运行模式，用来区分不同场景下要处理的事件集合。一个 RunLoop 可以包含多个 Mode，但一次只能运行在一个 Mode 中。
- **Source0 / Source1**: RunLoop 中的事件源类型。Source0 主要处理应用内部事件，Source1 主要处理基于端口和内核消息的事件，并常用于唤醒 RunLoop。
- **NSTimer**: 定时器对象，必须被添加到 RunLoop 的某个 Mode 中才能被调度执行。它的回调并不是凭空触发，而是由 RunLoop 在运行过程中检查并调用。

---

## 视频章节总结 ｜ iOS底层原理：RunLoop源码深度剖析与实战

本视频是iOS底层原理课程的RunLoop专题，讲师通过实战演示与源码分析，深入讲解了RunLoop的运行机制、数据结构和底层原理。课程从RunLoop无处不在的现象入手，通过堆栈分析证明其存在，并引入官方文档。接着剖析RunLoop的本质——一个特殊的do-while循环，详细解读其结构体、与线程的一一绑定关系、Mode与Item（Source、Timer、Observer）的对应关系。通过Timer实例，展示了任务如何加入Mode并在RunLoop中执行回调。随后深入RunLoop源码，完整呈现其运行循环的10个步骤：通知进入/退出、处理Source0、Source1唤醒、Timer回调、休眠等待消息等。最后总结了RunLoop的核心考点，强调线程、Mode与事件处理的关系，并给出面试题与学习建议，帮助学员系统掌握RunLoop底层原理。

### [00:00](https://bibigpt.co/content/f20a652f-9a5a-4490-bd34-a168b0fda2fa?t=0.000) - 🚀 RunLoop初探：无处不在的运行循环

![章节截图 00:00](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_6723009258709292188%5Cscreenshot_0.0s.jpg)

本章从实际开发中的现象出发，演示Timer、通知、UI事件等最终都会经过RunLoop，通过堆栈分析（bt命令）在源码中定位到CFRunLoopRun，证明RunLoop是应用事件处理的枢纽。随后引导学员查阅Apple官方文档（Threading Programming Guide），理解RunLoop在系统中的地位。同时提出Source0与Source1的区别等经典面试问题，为后续深入讲解做铺垫。讲师通过对比普通死循环导致CPU高占用，突出RunLoop的特殊性：它能节省CPU资源，在无事时休眠。

### [51:29](https://bibigpt.co/content/f20a652f-9a5a-4490-bd34-a168b0fda2fa?t=3089.000) - 🔍 RunLoop数据结构与线程关系解析

![章节截图 51:29](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_6723009258709292188%5Cscreenshot_3089.0s.jpg)

本章聚焦RunLoop的源码结构，通过分析CFRunLoopGetCurrent函数的创建过程，揭示RunLoop与线程（pthread）的一一绑定关系，即每个线程都有一个对应的RunLoop，但默认不启动（主线程除外）。深入CFRunLoop结构体，讲解其核心成员：线程（_pthread）、当前Mode（_currentMode）、Mode集合（_modes）以及CommonMode等。强调Mode是一种集合类型，用于存放Source、Timer、Observer等事件源，RunLoop通过切换Mode来处理不同优先级的事件。同时澄清Source0（处理App内部事件如UI）与Source1（基于Mach Port，具备唤醒能力）的区别。

### [01:24:44](https://bibigpt.co/content/f20a652f-9a5a-4490-bd34-a168b0fda2fa?t=5084.000) - ⏱️ 实例剖析：Timer如何融入RunLoop

![章节截图 01:24:44](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_6723009258709292188%5Cscreenshot_5084.0s.jpg)

以Timer为例，详细演示一个事件如何加入RunLoop并最终得到执行。首先通过子线程Timer不run就不work的例子，说明RunLoop必须运行才能处理事件。然后从API层（addTimer:forMode:）追踪到CFRunLoopAddTimer源码，揭示Timer会根据Mode类型被存储到对应的Mode集合中。随后在RunLoop运行时，通过CFRunLoopDoTimer遍历所有Timer，检查触发时间并调用其回调block。整个过程展示了“存储-轮询-回调”的事件处理模型，加深对RunLoop运行机制的理解。

### [01:58:15](https://bibigpt.co/content/f20a652f-9a5a-4490-bd34-a168b0fda2fa?t=7095.000) - ⚙️ RunLoop底层运行原理全揭秘

![章节截图 01:58:15](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_6723009258709292188%5Cscreenshot_7095.0s.jpg)

本章完整拆解RunLoop运行循环的核心逻辑。从CFRunLoopRun函数入手，剖析其外层do-while循环如何通过结果码控制退出。重点讲解内层循环的经典10步：通知Observer即将进入Loop；将要处理Timer/Source0；处理Source0事件；如果有Source1则通过mach_msg唤醒并跳转处理；否则通知Observer即将休眠；进行休眠等待mach_port消息或超时；被唤醒后通知Observer；处理唤醒事件（如Timer、Source1、GCD等）；判断是否退出。还分析了RunLoop使用GCD定时器作为超时计时，而非自身Timer，避免递归依赖。最后通过状态机（Activity）说明RunLoop的有状态性，为卡顿监测等应用奠定基础。

### [02:17:22](https://bibigpt.co/content/f20a652f-9a5a-4490-bd34-a168b0fda2fa?t=8242.000) - 📚 课程总结与学习路径

![章节截图 02:17:22](https://asset.localhost/C%3A%5CUsers%5CADMIN%5CAppData%5CLocal%5Cco.bibigpt.desktop%5Cscreenshots%5Cvideo_6723009258709292188%5Cscreenshot_8242.0s.jpg)

讲师总结RunLoop的核心知识点：线程一一对应、Mode与Item的一对多关系、事件处理流程、休眠唤醒机制及状态回调。重申RunLoop在界面优化（卡顿检测）、线程保活等场景的应用价值。随后介绍课程结束后的考试题目，鼓励学员通过写博客、做总结巩固知识，并给出后续学习建议（如音视频、架构等方向）。最后在轻松互动中解答学员关于面试、技术方向等问题，结束整个大师班底层课程。