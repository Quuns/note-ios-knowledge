# 【BibiGPT】AI 一键总结：[27_KVO，KVC底层原理.mp4](https://bibigpt.co/new/video?url=https%3A%2F%2Fbibigpt-files.s3.oss-cn-beijing.aliyuncs.com%2F1736405a-8ac6-4e3d-86b6-73ebab08d5e3%2FMjdfS1ZP77yMS1ZD5bqV5bGC5Y6f55CGLW1wNC1lbmNvZGVk.mp3%3FX-Amz-Algorithm%3DAWS4-HMAC-SHA256%26X-Amz-Content-Sha256%3DUNSIGNED-PAYLOAD%26X-Amz-Credential%3DLTAI5t7SY7MqvSzvdCxPSrEj%252F20260505%252Foss-cn-beijing%252Fs3%252Faws4_request%26X-Amz-Date%3D20260505T045547Z%26X-Amz-Expires%3D7200%26X-Amz-Signature%3D5bb220fd1f0d95e4f1763f7f0704ae9fb5272dd625ab414a902b0931d4c03fa7%26X-Amz-SignedHeaders%3Dhost%26x-amz-checksum-mode%3DENABLED%26x-id%3DGetObject)


## 摘要
本节课系统讲解了 iOS 中 KVC 与 KVO 的底层原理、调用流程、常见崩溃原因以及自定义实现思路。课程先从 KVC 的赋值和取值查找顺序入手，分析 `setValue:forKey:` 与 `valueForKey:` 的内部逻辑；随后重点讲解 KVO 如何通过运行时动态生成子类、修改 `isa` 指针、重写 setter 方法来实现属性监听，并延伸到 FBKVOController 的封装思路与自定义 KVO 的实现方式。


### 亮点
- 🔑 KVC 的核心思想是通过字符串形式的 Key 访问或修改对象属性，而不是直接调用属性访问器。
- 🧭 KVC 赋值时会优先查找 `set<Key>:`、`_set<Key>:` 等 setter 方法，找到后会直接调用对应方法完成赋值。
- 🧱 如果 KVC 没有找到 setter 方法，并且 `accessInstanceVariablesDirectly` 返回 `YES`，系统会继续按照 `_key`、`_isKey`、`key`、`isKey` 的顺序查找成员变量。
- ⚠️ 当 KVC 既找不到访问器方法，也找不到对应成员变量时，会调用 `setValue:forUndefinedKey:`，默认情况下会导致程序崩溃。
- 🔍 KVC 取值流程会优先查找 `get<Key>`、`<key>`、`is<Key>`、`_<key>` 等方法，找到后根据方法返回值取值。
- 🧩 KVC 的 getter 方法匹配规则比 setter 更宽松，取值时只要方法名匹配即可，而赋值时必须是真正的 setter 形式。
- 🛠️ 自定义 KVC 的实现可以基于运行时 API，按系统查找顺序依次查找方法和成员变量，并在失败时抛出未定义 Key 的异常。
- 👀 KVO 是观察者模式的一种实现，用于监听对象某个属性值的变化，并在变化时自动回调观察者。
- 🚨 使用 KVO 时必须在合适时机移除监听，否则被观察对象生命周期长于观察者时，可能向已释放对象发送消息而崩溃。
- 🧬 KVO 的底层原理是在添加观察者后，运行时动态生成一个 `NSKVONotifying_` 开头的子类，并让被观察对象的 `isa` 指针指向该子类。
- 🧠 动态生成的 KVO 子类会重写被观察属性的 setter 方法，并在 setter 内部调用 `willChangeValueForKey:` 和 `didChangeValueForKey:`。
- 🚫 KVO 默认不能直接监听普通成员变量，因为成员变量没有 setter 方法，无法触发动态子类中的 setter 逻辑。
- ✅ 如果通过 KVC 修改成员变量，仍有可能触发 KVO，因为 KVC 内部会配合 `willChangeValueForKey:` 和 `didChangeValueForKey:` 机制。
- 🔄 当移除 KVO 监听时，被观察对象的 `isa` 指针会重新指回原始类，但动态生成的 KVO 子类通常不会立即销毁。
- 🧰 FBKVOController 通过中介者封装系统 KVO，减少手动 `removeObserver` 的负担，并用 block 方式让回调逻辑更清晰。
- 🧪 自定义 KVO 的关键步骤包括动态创建派生类、添加 setter 方法、修改 `isa` 指针、保存观察者信息、在 setter 中触发回调。
- 🧹 更完整的自定义 KVO 还需要考虑自动移除监听，可以在动态派生类中添加 `dealloc` 方法来完成清理和 `isa` 指针恢复。
- 📌 理解 KVC 与 KVO 的底层机制不仅有助于日常开发排查问题，也能更从容地应对 iOS 底层原理相关面试。


[#KVC](https://bibigpt.co/search?q=KVC) [#KVO](https://bibigpt.co/search?q=KVO) [#iOS底层原理](https://bibigpt.co/search?q=iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86) [#Runtime](https://bibigpt.co/search?q=Runtime) [#isa指针](https://bibigpt.co/search?q=isa%E6%8C%87%E9%92%88) [#观察者模式](https://bibigpt.co/search?q=%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F) [#FBKVOController](https://bibigpt.co/search?q=FBKVOController)


### 思考 
1. **[KVC 赋值时，如果没有 setter 方法会发生什么？](https://bibigpt.co/search?q=KVC%20%E8%B5%8B%E5%80%BC%E6%97%B6%EF%BC%8C%E5%A6%82%E6%9E%9C%E6%B2%A1%E6%9C%89%20setter%20%E6%96%B9%E6%B3%95%E4%BC%9A%E5%8F%91%E7%94%9F%E4%BB%80%E4%B9%88%EF%BC%9F)**
  - 如果没有找到 `set<Key>:` 或 `_set<Key>:` 等 setter 方法，并且 `accessInstanceVariablesDirectly` 返回 `YES`，KVC 会继续按照 `_key`、`_isKey`、`key`、`isKey` 的顺序查找成员变量并赋值。如果这些都找不到，就会调用 `setValue:forUndefinedKey:`，默认会崩溃。
- 2. **[为什么 KVO 监听后一定要在合适时机移除？](https://bibigpt.co/search?q=%E4%B8%BA%E4%BB%80%E4%B9%88%20KVO%20%E7%9B%91%E5%90%AC%E5%90%8E%E4%B8%80%E5%AE%9A%E8%A6%81%E5%9C%A8%E5%90%88%E9%80%82%E6%97%B6%E6%9C%BA%E7%A7%BB%E9%99%A4%EF%BC%9F)**
  - 因为被观察对象可能比观察者生命周期更长。如果观察者已经释放，而被观察对象仍然持有监听关系，属性变化时系统可能继续向已释放的观察者发送回调消息，从而导致崩溃。尤其是被观察对象为单例或全局对象时更容易出现问题。
- 3. **[KVO 为什么不能直接监听普通成员变量？](https://bibigpt.co/search?q=KVO%20%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E7%9B%B4%E6%8E%A5%E7%9B%91%E5%90%AC%E6%99%AE%E9%80%9A%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F%EF%BC%9F)**
  - KVO 的触发依赖被观察属性的 setter 方法。系统会动态生成子类并重写对应 setter，在 setter 内部调用变化通知方法。如果只是直接修改成员变量，没有经过 setter，就不会触发 KVO。若想监听类似变化，可以通过 KVC 或手动调用 `willChangeValueForKey:` 与 `didChangeValueForKey:`。


### 术语解释
- **KVC（Key-Value Coding）**: 键值编码，允许通过字符串 Key 间接访问或修改对象属性、成员变量，是 Objective-C 中非常重要的动态访问机制。
- **KVO（Key-Value Observing）**: 键值观察，用于监听对象属性变化。其底层依赖运行时动态生成子类、重写 setter 方法，并在属性变化时通知观察者。
- **`isa` 指针**: Objective-C 对象内部指向其类对象的指针。KVO 添加监听后，会将被观察对象的 `isa` 指向动态生成的 KVO 子类。
- **`NSKVONotifying_` 子类**: 系统在 KVO 监听时动态创建的派生类，类名通常以 `NSKVONotifying_` 开头，用于拦截 setter 方法并触发观察回调。
- **FBKVOController**: Facebook 开源的 KVO 封装框架，通过中介者和 block 回调简化 KVO 使用，降低忘记移除监听和回调逻辑混乱的问题。

---

## 视频章节总结 ｜ iOS底层进阶：深入剖析KVC与KVO实现原理及实战

本视频是逻辑教育iOS底层进阶班第27天的课程，详细讲解了KVC（键值编码）和KVO（键值观察）的底层原理。讲师首先介绍了KVC的赋值与取值流程，通过官方文档和代码演示，阐述了系统按特定顺序查找setter/getter方法、实例变量的过程，并演示了自定义KVC的实现。随后深入KVO部分，讲解了KVO的使用陷阱（如单例对象的观察者未移除导致崩溃）、手动触发与关闭KVO的方式。核心原理部分通过LLDB调试和运行时分析，揭示了调用addObserver后系统动态生成一个以NSKVONotifying_为前缀的子类，该子类重写了被观察属性的setter方法，并在其中插入willChangeValueForKey:和didChangeValueForKey:调用来实现通知。讲师还分析了Facebook的FBKVOController框架，利用中间者模式和自动移除观察者简化KVO使用。最后，带领学习者手写了一个支持block回调、自动移除的简易KVO框架，加深对运行时和KVO机制的理解。课程最后讲师与学员道别，整个课程理论与实践结合紧密，有助于深入理解iOS底层机制。

### [00:00](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=0.000) - 🔑 KVC 赋值与取值底层流程
本章从KVC的基本概念入手，通过官方文档和代码演示详细剖析了setValue:forKey:和valueForKey:的搜索模式。对于赋值操作，系统优先查找setKey:方法，若未找到则查找_setKey:方法；若仍无，则根据accessInstanceVariablesDirectly返回值决定是否按顺序访问带下划线的成员变量。取值流程类似，依次查找getKey、key、isKey等方法，未找到时同样按顺序查找成员变量。讲师通过重写方法、添加多个成员变量等案例，清晰展示了每一步的执行顺序和崩溃条件，并强调set方法必须是标准setter形式，而get方法只需前缀为get即可。最后通过图解总结了KVC的整套赋值和取值流程图。

### [40:00](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=2400.000) - 🛠️ 自定义 KVC 实现
在理解KVC底层搜索模式的基础上，讲师引导学员实现了一个简易的自定义KVC。首先定义了LGPSetValueForKey:和LGPValueForKey:两个方法，内部严格按照苹果官方文档的三个步骤编写逻辑：先查找setter/getter方法，若不存在则检查accessInstanceVariablesDirectly方法并遍历所有成员变量进行赋值或取值。代码演示中展示了如何使用runtime的class_getInstanceVariable获取成员变量并设置值。通过这个实践环节，学员不仅巩固了搜索模式，还学会了如何运用runtime模拟系统KVC行为，为后续自定义KVO打下基础。讲师指出自定义KVC主要是为了面试时展现对原理的理解，实际开发中直接使用系统API即可。

### [55:00](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=3300.000) - 📡 KVO 使用注意事项与触发条件
本章通过代码实例讲解了KVO的常见陷阱。首先演示了当观察者为局部变量且未移除时通常不会崩溃，但若被观察对象是单例，则会导致已释放的观察者继续接收消息而崩溃，解释了循环引用与观察者生命周期的关系。然后介绍了如何通过重写automaticallyNotifiesObserversForKey:返回NO来手动关闭KVO，以及如何使用willChangeValueForKey:和didChangeValueForKey:手动触发通知。重点分析了KVO的触发条件：只有通过setter方法修改属性值才能触发观察者回调，直接修改成员变量无效，而使用KVC的setValue:forKey:可以间接触发，因为KVC内部调用了willChange和didChange方法。这些细节为后续原理学习埋下伏笔。

### [01:08:20](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=4100.000) - ⚙️ KVO 底层原理：动态子类与 setter 重写
本章是KVO最核心的原理讲解。通过打印实例的class，发现调用addObserver后对象的类变成了NSKVONotifying_XXXX的子类，且该类是动态生成的。借助runtime函数打印方法列表，发现子类中重写了被观察属性的setter方法。进一步通过watchpoint指令监听属性变化，在汇编调用栈中捕捉到willChangeValueForKey和didChangeValueForKey的调用。由此总结出KVO的实现机制：系统动态创建中间子类并修改对象的isa指针指向该子类；子类重写setter，在其中调用两个will/didChange方法，最终触发observeValueForKeyPath回调。此外还验证了removeObserver时isa会指回原类，但子类不会被销毁。整个分析过程使用了LLDB、Objective-C运行时函数，逻辑严密。

### [01:26:40](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=5200.000) - 📦 FBKVOController 框架源码解读
讲师介绍了Facebook开源的FBKVOController框架，它解决了系统KVO需要手动移除观察者和回调函数容易臃肿的痛点。通过源码分析，该框架内部维护了一个单例（FBKVOSharedController）作为中介者，持有FBKVOInfo数组存储观察信息，并使用block作为回调。其自动移除观察者的原理在于：控制器在初始化时弱引用观察者，当观察者销毁时，控制器也会在dealloc中调用unobserveAll移除所有监听。讲师将框架比喻为“房产中介”，用户只需描述监听属性，其余都由框架处理。此章不仅展示了优秀代码的设计思想，也为自定义KVO提供了模板。

### [01:50:00](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=6600.000) - 🧪 手写自定义 KVO 并进行优化
结合前面的原理和框架思想，讲师带领学员动手实现了一个支持block回调、自动移除的自定义KVO。实现过程中，先判断属性是否有setter方法；若无则忽略。然后动态创建子类并重写setter，在setter中先调用父类的setter赋值，再触发block回调。为了避免手动移除，重写了子类的dealloc方法，在对象销毁时将isa指回原类并清理关联对象。针对多个属性监听的问题，将setter添加步骤抽离到循环外，并为每个属性动态添加独立的setter方法。最终实现了一个类似FBKVOController的轻量级KVO工具，仅需一行addObserver即可完成监听，无需手动remove。此章综合运用了runtime、关联对象、block等知识，实战性强。

### [02:30:00](https://bibigpt.co/content/6734a02d-59b6-4ecd-8928-26f222af846e?t=9000.000) - 🍵 课程回顾与讲师告别
在课程的最后，讲师简要回顾了KVC和KVO的全部知识点，强调经过深入讲解后，学员在面试和日常开发中应能从容应对这类问题。随后透露这是自己在逻辑教育的最后一节课，表达了对学员两个月包容和陪伴的感谢，并鼓励大家通过微信继续交流。整个告别氛围轻松温暖，讲师祝愿学员找到满意的工作，江湖再见。该章节虽非技术内容，但体现了课程的完整性和师生情谊。