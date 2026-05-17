
---
# iOS 面经考点大全

> 来源：[iOS实习面经(字节美团阿里蘑菇街)](https://www.yuque.com/ioser/spb08a/nofps3)
> 涵盖公司：字节跳动、美团、阿里、蘑菇街
> 整理方式：按知识板块分类，重复考点合并，每题附答案（点击展开）

---

## 目录

- [一、Runtime & 消息机制](#一runtime--消息机制)
- [二、Block](#二block)
- [三、ARC & 内存管理](#三arc--内存管理)
- [四、Category](#四category)
- [五、KVC & KVO](#五kvc--kvo)
- [六、多线程](#六多线程)
- [七、网络](#七网络)
- [八、编译 & App 启动](#八编译--app-启动)
- [九、UI 相关](#九ui-相关)
- [十、操作系统基础](#十操作系统基础)
- [十一、算法 & 智力题](#十一算法--智力题)
- [十二、综合 & 软技能](#十二综合--软技能)

---

## 一、Runtime & 消息机制

### Q1. OC 中的方法调用过程（消息发送机制）

<details>
<summary>点击展开答案</summary>

OC 方法调用本质是 **消息发送**：`objc_msgSend(id self, SEL op, ...)`。

**完整流程：**

1. **快速查找**：通过 `isa` 指针找到类对象，在类的 `cache`（缓存表）中查找 SEL 对应的 IMP。找到则直接调用。

2. **慢速查找**：cache 未命中，在类的 `method_list`（方法列表）中二分查找 SEL 对应的 IMP。

3. **逐级查找**：类中未找到，通过 `superclass` 指针向父类递归查找（同样先查 cache，再查 method_list），直到 `NSObject`。

4. **动态方法解析**：以上都没找到，调用 `+resolveInstanceMethod:`（或 `+resolveClassMethod:`），允许运行时动态添加方法实现。返回 YES 则重新走一遍查找流程。

5. **消息转发**：
   - **快速转发**：`-forwardingTargetForSelector:` → 返回一个备选对象，方法会转发给这个对象执行
   - **完整转发**：`-methodSignatureForSelector:` + `-forwardInvocation:` → 获取方法签名，构造 NSInvocation 并手动处理
   - **兜底**：`-doesNotRecognizeSelector:` → 抛出 `unrecognized selector sent to instance` 异常崩溃

**关键数据结构**：
```
实例对象 → isa → 类对象（Class）
                  +-- cache: 方法缓存（哈希表，SEL -> IMP）
                  +-- method_list: 方法列表
                  +-- superclass -> 父类
                  +-- ...
```

**cache 加速机制**：每次方法调用后，IMP 会被缓存到当前类的 cache 中，下次相同 SEL 调用时直接命中。

</details>

### Q2. OC 比起 C 增加了什么？哪些依赖 Runtime？

<details>
<summary>点击展开答案</summary>

**OC 比 C 增加的核心能力：**

| 特性 | 说明 |
|------|------|
| 面向对象 | 类、对象、继承、多态 |
| 消息机制 | 动态方法派发 `objc_msgSend` |
| 动态类型 | `id` 类型、运行时类型检查 |
| Category | 运行时加载的分类 |
| Protocol | 协议（正式协议 + 可选方法） |
| Block | 闭包（基于 C 结构体实现） |
| Property | 属性自动合成 getter/setter/ivar |
| ARC | 自动引用计数，编译期插入 retain/release |
| KVC/KVO | 键值编码和观察 |
| NSNotification | 通知中心 |

**依赖 Runtime 实现的：**
- 消息发送（所有方法调用）
- 动态方法解析（`resolveInstanceMethod:`）
- 消息转发（`forwardingTargetForSelector:`、`forwardInvocation:`）
- Category 的方法/协议加载
- KVC / KVO 的实现
- Method Swizzling
- Associated Objects（关联对象）
- `isa-swizzling`（KVO 中动态生成子类）

**不依赖 Runtime 的：**
- C 函数调用
- 纯 C 结构体操作
- Block 的底层结构体定义（编译期确定，但 `__block` 变量的转发依赖 forwarding 指针）

</details>

### Q3. 面向对象三个特性是怎么通过 Runtime 构建的？

<details>
<summary>点击展开答案</summary>

**封装**：通过 `ivar`（实例变量）存储在对象内存中，默认是 `@protected` 权限。Property 通过自动合成 getter/setter 方法访问，编译器生成的 setter/getter 内部通过 `self` 偏移量访问 ivar。

**继承**：通过 `isa` + `superclass` 指针链实现。
```
实例对象 --isa--> 类对象 --superclass--> 父类对象 --> ... --> NSObject --> nil
                   |
                 元类 --superclass--> 父元类 --> ... --> NSObject 元类 --> NSObject
```
- 实例方法查找：沿类对象的 superclass 链向上
- 类方法查找：沿元类的 superclass 链向上

**多态**：通过消息发送机制实现。同一个 SEL 在不同类的 method_list 中映射到不同的 IMP（方法实现）。调用时根据接收者（receiver）的实际类型，在对应类的方法列表中查找实现，实现运行时动态派发。

**关键结构**：
```objc
// 类对象结构（简化）
struct objc_class {
    Class isa;                  // 指向元类
    Class superclass;           // 父类
    cache_t cache;              // 方法缓存
    class_data_bits_t bits;     // 类数据（methods, ivars, properties, protocols）
};

// 元类的特殊设计：
// - 元类的 isa 都指向根元类（NSObject 元类）
// - NSObject 元类的 isa 指向自己
// - NSObject 的 superclass 是 nil
// - NSObject 元类的 superclass 是 NSObject（保证类方法找不到时能查 NSObject 的实例方法）
```

</details>

### Q4. 对象的 ivar 存在哪？是根据什么生成的？类对象有 ivar 吗？

<details>
<summary>点击展开答案</summary>

**ivar 存储位置**：实例变量存储在**实例对象**的内存空间中。对象在堆上分配内存时，`isa` 指针之后紧跟着所有 ivar 的内存布局（按对齐规则排列）。

**ivar 的生成依据**：
- 在 `@interface` 的 `{}` 中直接声明的 ivar
- `@property` 声明的属性，编译器自动合成 `_propertyName` 的 ivar（除非手动 `@dynamic` 或同时手动实现 getter 和 setter）
- 类扩展（extension）中的 property 也会合成 ivar

**类对象有 ivar 吗？**
**没有。** 类对象只存储方法列表、协议列表、属性描述等元数据。ivar 是实例级别的数据，每个实例对象有一份独立的内存拷贝。类对象中存储的是 ivar 的**描述信息**（名称、类型、偏移量），供运行时在实例对象的内存布局中定位 ivar。

**ivar 布局**：编译期确定每个 ivar 在对象内存中的偏移量。编译器根据 property 的类型、顺序、内存对齐要求生成 ivar layout。`class_ro_t` 中的 `ivar_list` 记录了这些偏移信息。

</details>

---

## 二、Block

### Q5. Block 的实现原理、变量截获机制

<details>
<summary>点击展开答案</summary>

**Block 的本质**：Block 是一个 **Objective-C 对象**，底层是以下结构体：

```objc
struct __block_impl {
    void *isa;          // 指向 block 的类型（__NSGlobalBlock__, __NSStackBlock__, __NSMallocBlock__）
    int Flags;
    int Reserved;
    void *FuncPtr;      // 函数指针，指向 block 代码的实现
};

// 完整 Block 结构体（以截获一个 int 变量为例）
struct __MainBlock__impl_0 {
    struct __block_impl impl;
    struct __MainBlock_desc_0 *Desc;  // 描述信息（size, copy/dispose 函数）
    int capturedVar;                   // 截获的变量
};
```

**三种 Block 类型**：

| 类型 | 存储区域 | 条件 |
|------|----------|------|
| `__NSGlobalBlock__` | data 段 | 不截获任何变量（或只截获全局/静态变量） |
| `__NSStackBlock__` | 栈 | 截获局部变量，未执行 copy |
| `__NSMallocBlock__` | 堆 | 栈 block 执行 copy 后移到堆上 |

**变量截获规则**（核心考点）：

| 变量类型 | 截获方式 | 能否修改 |
|----------|---------|---------|
| 局部普通变量 | 值截获（深拷贝） | ❌ 不能 |
| 局部对象变量 | 值截获，同时 copy/dispose 管理内存 | ❌ 不能修改指向，但可调用对象方法 |
| `static` 局部变量 | 指针截获（地址引用） | ✅ 能 |
| 全局变量 | 不截获，直接访问 | ✅ 能 |
| `__block` 变量 | 指针截获（通过 `__Block_byref` 结构体转发） | ✅ 能 |

**`__block` 原理**：
```objc
// __block int num = 10; 被编译为：
struct __Block_byref_num_0 {
    void *__isa;
    struct __Block_byref_num_0 *__forwarding;  // 转发指针（关键！）
    int __flags;
    int __size;
    int num;   // 实际值
};
```
- `__forwarding` 指针：栈上的 `__Block_byref` 被 copy 到堆后，栈上原结构的 `__forwarding` 会指向堆上的新结构。这样无论访问栈还是堆上的结构，都能操作同一个值。
- 这保证了 `__block` 变量在 block copy 前后对值的修改是统一的。

**Block copy 的触发时机**：
- Block 作为返回值时
- 赋值给 `__strong` 修饰的变量时
- 作为 Cocoa API 的 usingBlock 参数时
- GCD 等需要将 block 存到堆上的 API

</details>

### Q6. Block 的 forwarding 指针是干什么的？

<details>
<summary>点击展开答案</summary>

`__forwarding` 是 `__Block_byref` 结构体中的一个指针，用于**保证 `__block` 变量在栈到堆迁移后，所有引用方访问同一份数据**。

**工作流程**：
1. `__block` 变量初始化时，`__Block_byref` 结构体在**栈**上，`__forwarding` 指向自己（栈地址）。
2. Block 执行 `copy` 时，`__Block_byref` 也被拷贝到**堆**上。
3. copy 完成后，**栈上的 `__forwarding` 被更新为指向堆上的新结构**；堆上的 `__forwarding` 指向自己（堆地址）。
4. Block 内部访问 `__block` 变量时，通过 `__forwarding-&gt;value` 访问，透明地定位到堆上的值。

```
copy 前：             copy 后：
栈: [value=10]       栈: [value=10, __forwarding ----+
     ^forwarding=self                              |
                                                   v
                       堆: [value=10, __forwarding=self]
```

**为什么需要这个设计？**
- Block 可能多次 copy，但 `__block` 变量只应有一份活跃的值
- 栈上的 block 销毁后，堆上的值仍然存活
- `__forwarding` 使得无论从栈还是堆访问，都能找到当前有效的堆内存

</details>

---

## 三、ARC & 内存管理

### Q7. ARC 是什么？原理是什么？都做了什么？

<details>
<summary>点击展开答案</summary>

**ARC（Automatic Reference Counting）**：编译器在**编译期**自动分析对象的引用关系，在适当位置插入 `retain`/`release`/`autorelease` 代码。

**核心原理**：
- 编译器分析每个对象的所有权修饰符（`__strong`、`__weak`、`__unsafe_unretained`、`__autoreleasing`）
- 根据引用计数的增/减规则，自动插入内存管理代码
- **不是 GC（垃圾回收）**，不会在运行时扫描和清理，是纯编译期行为

**ARC 做的事情**：
1. 自动插入 `retain`/`release`（局部变量离开作用域时 release）
2. 自动处理 property setter 中的内存管理（旧值 release，新值 retain）
3. 自动处理返回值的内存管理（`objc_autoreleaseReturnValue` + `objc_retainAutoreleasedReturnValue` 的 TLS 优化）
4. 管理 `__block` 变量的内存
5. dealloc 中自动调用 `[super dealloc]`，自动释放 ivar

**所有权修饰符总结**：
| 修饰符 | 行为 |
|--------|------|
| `__strong`（默认） | retain 对象，引用计数 +1 |
| `__weak` | 不 retain，对象释放时自动置 nil |
| `__unsafe_unretained` | 不 retain，不自动置 nil（野指针风险） |
| `__autoreleasing` | 将对象放入 autoreleasepool，延迟 release |

</details>

### Q8. weak 的实现原理？weak 表的 key-value 分别是什么？

<details>
<summary>点击展开答案</summary>

**weak 实现依赖 Runtime 中的全局 `weak_table`**（哈希表）。

**数据结构**：
```
weak_table（全局哈希表）
-----------------------------------------------------
  key: 对象地址（被弱引用的对象）
  value: weak_entry_t（含一个 weak 指针数组）
           +-- referent: 对象地址（回到 key）
           +-- inline_referrers[]: 所有指向该对象的
                  __weak 指针地址的数组
-----------------------------------------------------
```

**weak 的完整生命周期**：

1. **初始化 `__weak` 变量**：
   ```objc
   __weak NSObject *weakObj = obj;
   // 调用 objc_initWeak(&weakObj, obj)
   ```
   - 在 weak_table 中以 `obj` 地址为 key 查找/创建 `weak_entry_t`
   - 将 `&weakObj` 添加到 entry 的 `inline_referrers` 数组中
   - 返回 obj

2. **读取 `__weak` 变量**：
   ```objc
   NSLog(@"%@", weakObj);
   // 调用 objc_loadWeakRetained(&weakObj)
   ```
   - 从 weak_entry_t 中取出 referent
   - retain 后返回（保证对象在使用期间不被释放）
   - 调用方用完后再 release

3. **对象释放时**（dealloc 流程）：
   ```objc
   // dealloc → _objc_rootDealloc → object_dispose → objc_destructInstance
   // → objc_clear_deallocating
   ```
   - 以对象地址为 key 在 weak_table 中查找
   - 遍历 `inline_referrers` 数组，将所有 `__weak` 指针**置为 nil**
   - 从 weak_table 中删除该 entry

**weak 表的 key-value 总结**：
- **key**：被弱引用对象的**内存地址**（referent）
- **value**：一个包含所有指向该对象的 weak 指针地址列表的 `weak_entry_t`

**为什么 delegate 用 weak？**
防止循环引用。如果 delegate 用 strong，A 持有 B 且 B.delegate = A，两者互相持有导致无法释放。

</details>

### Q9. MRC 有 weak 吗？用什么代替？

<details>
<summary>点击展开答案</summary>

**MRC 没有 `__weak`。** MRC 时代只有以下选择：

| 修饰符 | 行为 |
|--------|------|
| `assign`（默认） | 简单赋值，不改变引用计数，对象释放后不置 nil |
| `retain` | 引用计数 +1 |
| `copy` | 复制对象 |

**MRC 中 delegate 用 `assign` 代替 weak**：

```objc
// MRC 时代
@property (nonatomic, assign) id<MyDelegate> delegate;
```

`assign` 不会自动置 nil，delegate 释放后变成**野指针**，这是 MRC 时代常见的 crash 来源。因此 MRC 下必须在 dealloc 中手动将 delegate 设为 nil。

**weak 的实现**是 Runtime 级别支持：通过全局 `weak_table`（哈希表）记录所有 weak 引用，对象释放时遍历将其置 nil。MRC 中没有这个机制。

</details>

### Q10. atomic 和 nonatomic 区别？锁原理？

<details>
<summary>点击展开答案</summary>

| 维度 | `atomic`（默认） | `nonatomic` |
|------|-----------------|-------------|
| 线程安全 | getter/setter 加自旋锁 | 无锁 |
| 性能 | 较慢 | 快 |
| 读写完整性 | 保证 getter/setter 原子性 | 不保证 |

**atomic 的锁原理**：
- 使用 **`spinlock_t`**（自旋锁）保护 setter 和 getter
- setter 时加锁 → 赋值 → 解锁
- getter 时加锁 → 取值 → 解锁
- 只是保证了 get/set 的原子性，**不是真正的线程安全**

**为什么 atomic 不等于线程安全？**
```objc
// atomic 只能保证 self.name 的单次读/写是原子的
// 但多步骤操作仍不安全：
if (self.name.length > 0) {       // 读操作1
    [self doSomething:self.name]; // 读操作2，期间可能被其他线程修改
}
```

**何时用 atomic？**
- 属性可能被多个线程同时读写，且不需要复杂逻辑操作
- 跨线程传递数据时的基本保证
- 实际开发中很少用，性能开销大且保护有限，通常用 nonatomic + 上层同步机制

</details>

### Q11. copy 的使用场景和条件？没重写 copyWithZone 会怎样？

<details>
<summary>点击展开答案</summary>

**使用场景**：当属性类型有**可变子类**时（`NSString`/`NSMutableString`、`NSArray`/`NSMutableArray`、`NSDictionary`/`NSMutableDictionary`）：

```objc
// 正确：防止外部传入 NSMutableString 后被修改
@property (nonatomic, copy) NSString *name;

// 错误：如果外部传入 NSMutableString，后续修改会影响属性值
@property (nonatomic, strong) NSString *name;
```

**使用条件**：类必须遵循 `NSCopying` 协议并实现 `-copyWithZone:`。
```objc
- (id)copyWithZone:(NSZone *)zone {
    MyClass *copy = [[MyClass allocWithZone:zone] init];
    copy.property1 = self.property1;
    copy.property2 = [self.property2 copy];
    return copy;
}
```

**没有重写 copyWithZone 的结果**：
- 调用 `[obj copy]` 时会抛异常：`*** -[MyClass copyWithZone:]: unrecognized selector sent to instance`
- 使用 `copy` 修饰该类型的 property 时，setter 中调用 `copy` 会崩溃
- 自定义类默认不支持 copy

**Block 属性为什么用 copy？**
- MRC 时代 block 在栈上，必须 copy 到堆才能安全持有（防止栈 block 被释放后成为野指针）
- ARC 下 `copy` 和 `strong` 对 block 效果相同（ARC 自动处理好）

</details>

---

## 四、Category

### Q12. Category 能添加什么？原理？为什么能添加 property 不能添加 ivar？

<details>
<summary>点击展开答案</summary>

**Category 能添加什么**：
- 实例方法、类方法
- Protocol
- Property（声明 + `@dynamic` 或关联对象实现）
- 通过 `Associated Objects` 给已有类关联值

**Category 不能添加什么**：
- **实例变量（ivar）** ❌ —— 核心限制
- 不能添加新的 `+load` 方法（可以定义但不会替换原有行为）

**原理**：
Category 在**运行时**被加载，通过 `objc_loadCategory` → `attachCategories` 将分类中的 method_list、protocol_list、property_list 合并到**主类**对应的列表中。合并后的数据存储在 `class_rw_t`（可读写类数据）中，而非编译期确定的 `class_ro_t`（只读类数据）。

```objc
// 加载顺序：
// 1. 编译期：生成 category_t 结构体，存储方法/属性/协议列表
struct category_t {
    const char *name;                    // 分类名
    classref_t cls;                      // 扩展的类
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};

// 2. 运行时：attachCategories 将列表追加到类的 class_rw_t 中
// 3. 后加载的 category 方法会覆盖先加载的（同名方法）
```

**为什么能添加 property 不能添加 ivar？**

| | Property | ivar |
|---|---------|------|
| 本质 | getter + setter 方法的声明 | 对象内存布局中的空间 |
| 影响范围 | 方法列表（运行时可修改） | 类结构体大小（编译期固定） |
| 添加时机 | 运行时动态追加到 method_list | 编译期确定，分配在 `class_ro_t` |

- Category 加载时，类的内存布局已经由编译期固定在 `class_ro_t` 中，不能改变
- 新增 ivar 意味着要改变对象的内存大小，这会导致已创建的实例对象内存错乱
- Property 只是 getter/setter 的声明（加到了 method_list），运行时可以追加

**怎么让 property 有值？**
```objc
// 在 .h 中声明 property
@interface NSObject (MyCategory)
@property (nonatomic, copy) NSString *myProp;
@end

// 在 .m 中用关联对象实现
#import <objc/runtime.h>
@implementation NSObject (MyCategory)
static const void *kMyPropKey = &kMyPropKey;

- (void)setMyProp:(NSString *)myProp {
    objc_setAssociatedObject(self, kMyPropKey, myProp, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)myProp {
    return objc_getAssociatedObject(self, kMyPropKey);
}
@end
```

</details>

### Q13. iOS 8 编译的 App 能在 iOS 10 上运行，如果 NSObject 修改了为什么还能正常运行？

<details>
<summary>点击展开答案</summary>

**关键原因：动态链接 + 运行时查找。**

1. **App 不内嵌系统库**：编译时链接的是 `UIKit.framework`、`Foundation.framework` 等系统库的**TBD（Text-Based Dylib Stub）**，只包含符号声明，不包含实际代码。运行时由 dyld 动态链接到当前 iOS 版本的实际系统库。

2. **方法通过 SEL 动态查找**：OC 方法调用是 `objc_msgSend(id, SEL, ...)`，SEL 在编译期确定（字符串哈希），IMP 在**运行时**通过 isa 链查找。iOS 10 上 NSObject 的方法列表可能增加了新方法或修改了实现，但已有的 SEL → IMP 映射仍然有效。

3. **二进制兼容性保证**：Apple 保证系统框架的 ABI 稳定性，公开 API 不会删除或修改签名（弃用的会用 `__attribute__((availability))` 标记，编译期产生警告但运行时仍可用）。

4. **Category 的加载机制**：系统框架的 Category 在主类初始化时被 `attachCategories` 合并。iOS 10 上的加载逻辑保证了兼容旧版编译的方法列表。

**可能的风险**：
- 使用私有 API（系统版本升级可能移除/改名）
- 依赖未定义行为（如 swizzle 系统方法的内部实现）
- 使用已废弃 API（iOS 10 上可能已移除）

</details>

---

## 五、KVC & KVO

### Q14. KVC 的使用？KVO 的使用和原理？

<details>
<summary>点击展开答案</summary>

**KVC（Key-Value Coding）**：通过字符串 key 访问对象属性。
```objc
[obj valueForKey:@"name"];
[obj setValue:@"Tom" forKey:@"name"];
[obj valueForKeyPath:@"person.address.city"];
```

**KVO（Key-Value Observing）**：监听对象属性变化。
```objc
[objA addObserver:objB forKeyPath:@"name"
          options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
          context:NULL];

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    // 处理变化
}
```

**KVO 底层实现原理**（核心）：

1. **动态创建子类**：`addObserver` 被调用时，Runtime 通过 `isa-swizzling` 技术，动态创建 `NSKVONotifying_OriginalClass` 子类。
2. **重写 setter**：子类重写被观察属性的 setter 方法，插入 `willChangeValueForKey:` 和 `didChangeValueForKey:` 调用。
3. **isa 指向**：将原对象的 `isa` 指针指向新的子类。
4. **触发通知**：setter 调用时，`didChangeValueForKey:` 内部遍历观察者列表，调用 `observeValueForKeyPath:...`。

```objc
// 重写后的 setter 伪代码
- (void)setName:(NSString *)name {
    _NSSetObjectValueAndNotify();  // 内部：
    // [self willChangeValueForKey:@"name"];
    // _name = name;
    // [self didChangeValueForKey:@"name"];
}
```

**手动触发 KVO**：
```objc
[self willChangeValueForKey:@"name"];
_name = @"new";
[self didChangeValueForKey:@"name"];
```

**注意事项**：
- 必须手动 `removeObserver`（iOS 9+ 的部分场景已自动清理，但不建议依赖）
- 只对遵循 KVC 命名规范的属性生效（有 setter，或直接修改 ivar + 手动通知）
- 直接修改 ivar 不触发 KVO（如 `_name = @"x"`）

</details>

---

## 六、多线程

### Q15. 进程和线程的区别？内存隔离吗？

<details>
<summary>点击展开答案</summary>

| 维度 | 进程（Process） | 线程（Thread） |
|------|----------------|---------------|
| 定义 | 资源分配的基本单位 | CPU 调度的基本单位 |
| 资源 | 独立的内存空间、文件描述符、全局变量 | 共享进程的内存空间 |
| 创建开销 | 大（复制父进程资源） | 小（共享进程资源） |
| 通信方式 | IPC（管道、共享内存、信号、socket） | 直接读写共享内存 |
| 隔离性 | 强，一个进程崩溃不影响其他 | 弱，一个线程崩溃可能导致整个进程崩溃 |
| 切换开销 | 大（需要切换内存映射） | 小（同一地址空间，只需切换寄存器） |

**内存隔离**：
- **进程之间**：**内存隔离**。每个进程有独立的虚拟地址空间，通过 MMU（内存管理单元）映射到不同的物理内存。进程 A 不能直接访问进程 B 的内存。
- **线程之间**：**不隔离**。同一进程内的线程共享堆、全局变量、静态变量、文件描述符。每个线程有独立的栈空间。

</details>

### Q16. GCD 怎么实现线程安全？barrier 原理？

<details>
<summary>点击展开答案</summary>

**GCD 实现线程安全的方式**：

1. **串行队列（Serial Queue）**：任务按 FIFO 顺序逐一执行，同一时间只有一个任务在执行：
```objc
dispatch_queue_t serialQueue = dispatch_queue_create("com.serial", DISPATCH_QUEUE_SERIAL);
dispatch_async(serialQueue, ^{ /* 线程安全写入 */ });
```

2. **并发队列 + Barrier**：读操作可并发，写操作用 barrier 独占：
```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.rw", DISPATCH_QUEUE_CONCURRENT);

// 读：可并发
- (id)read {
    __block id result;
    dispatch_sync(concurrentQueue, ^{
        result = self.data;
    });
    return result;
}

// 写：barrier 等待所有读完成后独占执行
- (void)write:(id)newData {
    dispatch_barrier_async(concurrentQueue, ^{
        self.data = newData;
    });
}
```

**Barrier 原理**：
- `dispatch_barrier_async` 将一个任务提交到并发队列
- 该任务**等待**队列中所有已提交任务执行完毕
- 该任务**独占**执行期间，队列不会启动其他任务
- 执行完毕后，队列恢复正常并发模式

**其他线程安全方案**：
- `@synchronized(obj)`：互斥锁，性能较差
- `NSLock`、`NSRecursiveLock`
- `pthread_mutex`：底层互斥锁
- `os_unfair_lock`：iOS 10+ 替代 OSSpinLock
- `dispatch_semaphore`：信号量控制并发数

</details>

### Q17. 进程间通信（IPC）的方式？

<details>
<summary>点击展开答案</summary>

| IPC 方式 | 说明 | iOS 使用情况 |
|-----------|------|-------------|
| **管道（Pipe）** | 单向数据流，父子进程间 | Foundation 提供 `NSPipe` |
| **共享内存** | 最快的 IPC，映射同一块物理内存 | iOS 沙盒限制，App 间不可用 |
| **Socket** | 网络通信，跨机器跨进程 | 可用，如 App 与 Extension 通信 |
| **Mach Port** | macOS/iOS 内核级 IPC | iOS 底层大量使用 |
| **XPC Service** | Apple 推荐的方式 | iOS 受限（App 与 Extension） |
| **信号（Signal）** | 异步事件通知 | 有使用限制 |
| **消息队列** | 异步消息传递 | 较少直接使用 |
| **文件** | 读写文件通信 | App Group 共享目录 |
| **URL Scheme** | App 间通信 | 打开/跳转到其他 App |
| **Keychain** | 同开发者 App 间共享数据 | 共享 credentials |
| **剪贴板** | App 间文本传输 | `UIPasteboard` |
| **App Group** | Extension 与宿主 App 共享 | `NSUserDefaults` + suiteName |

</details>

---

## 七、网络

### Q18. TCP 和 UDP 的区别？DNS 用 TCP 还是 UDP？

<details>
<summary>点击展开答案</summary>

| 维度 | TCP | UDP |
|------|-----|-----|
| 连接 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（确认、重传、排序） | 不可靠（尽力交付） |
| 顺序 | 保证有序 | 不保证有序 |
| 头部开销 | 20 字节 | 8 字节 |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有 | 无 |
| 适用场景 | HTTP、文件传输、邮件 | 视频直播、DNS、游戏、VoIP |
| 速度 | 较慢 | 快 |

**DNS 用 UDP 为主，TCP 为辅**：
- **默认 53 端口 UDP**：查询响应通常很小（&lt;512 字节），UDP 开销低、延迟小
- **区域传送（Zone Transfer）用 TCP**：数据量大，需要可靠传输
- **响应超过 512 字节时切 TCP**：DNS over TCP（RFC 7766）

**DNS 属于应用层协议**。

</details>

### Q19. 三次握手四次挥手过程？为什么握手三次挥手四次？

<details>
<summary>点击展开答案</summary>

**三次握手（建立连接）**：
```
客户端                         服务端
  |                              |
  |--- SYN, seq=x -------------->|
  |                              |
  |<-- SYN+ACK, seq=y, ack=x+1 --|
  |                              |
  |--- ACK, seq=x+1, ack=y+1 -->|
  |                              |
  |      ESTABLISHED              |
```

1. 客户端发送 `SYN=1, seq=x`，进入 `SYN_SENT`
2. 服务端回复 `SYN=1, ACK=1, seq=y, ack=x+1`，进入 `SYN_RCVD`
3. 客户端发送 `ACK=1, seq=x+1, ack=y+1`，双方进入 `ESTABLISHED`

**为什么握手三次？**
- 两次不够：防止已失效的连接请求报文突然传到服务端。若只用两次，服务端收到旧 SYN 后会直接建立连接，浪费资源。
- 三次刚好：第三次确认后，双方都确认自己的发送和接收能力正常。

**协商了什么？**：初始序列号（ISN）、MSS（最大报文段大小）、窗口大小、是否支持 SACK 等。

**四次挥手（断开连接）**：
```
客户端                         服务端
  |                              |
  |--- FIN, seq=u -------------->|   主动关闭
  |                              |
  |<-- ACK, ack=u+1 -------------|   确认收到
  |                              |
  |<-- FIN, seq=v, ack=u+1 ------|   被动关闭
  |                              |
  |--- ACK, ack=v+1 ------------>|   确认关闭
  |           TIME_WAIT          |
```

**为什么挥手四次？**
- TCP 是**全双工**通信，每个方向需要独立关闭
- 一方发 FIN 只表示该方不再发送数据，但仍可接收
- 另一方可能还有数据要发，先 ACK 再等数据发完才发自己的 FIN
- 所以是四次：FIN → ACK → FIN → ACK（ACK 和 FIN 不能合并）

</details>

### Q20. HTTPS 原理？

<details>
<summary>点击展开答案</summary>

**HTTPS = HTTP + TLS/SSL**，提供加密、身份认证、数据完整性。

**TLS 握手过程（简化）**：

1. **Client Hello**：客户端发送支持的 TLS 版本、加密套件列表、随机数 1
2. **Server Hello**：服务端选择 TLS 版本和加密套件，发送证书链（含公钥）、随机数 2
3. **证书验证**：客户端验证证书链（CA 签名、有效期、域名匹配）
4. **Pre-Master Secret**：客户端生成随机数 3，用服务端公钥加密发送
5. **生成会话密钥**：双方用随机数 1+2+3 通过 PRF 算法生成对称密钥
6. **通知对方**：双方发送加密的 Finished 消息，握手完成
7. **对称加密通信**：之后的数据用会话密钥对称加密

**混合加密**：非对称加密（RSA/ECDHE）用于交换对称密钥，对称加密（AES）用于数据传输。这样兼顾安全性和性能。

**Charles 抓包 HTTPS 的原理**：中间人攻击（MITM）。
- 客户端 → （信任 Charles 根证书）→ Charles 冒充服务端
- Charles → （真实 HTTPS）→ 目标服务端
- Charles 用自签证书与客户端通信，客户端安装 Charles 根证书后信任该证书
- 明文在 Charles 中可见

</details>

### Q21. HTTP/2.0 的主要特性

<details>
<summary>点击展开答案</summary>

| 特性 | 说明 |
|------|------|
| **二进制分帧** | 不再用文本协议，改为二进制帧传输，解析更高效 |
| **多路复用** | 一个 TCP 连接上并行传输多个请求/响应，消除队头阻塞 |
| **头部压缩** | HPACK 算法，减少重复头部的传输 |
| **服务端推送** | 服务器可主动推送客户端可能需要的资源 |
| **流优先级** | 可以指定请求的优先级，优化资源加载顺序 |
| **单连接** | 只需一个 TCP 连接即可 |

</details>

### Q22. 浏览器输入 URL 按下回车后的完整流程

<details>
<summary>点击展开答案</summary>

1. **URL 解析**：解析协议（https）、域名、端口、路径等
2. **DNS 解析**：域名 → IP 地址（浏览器缓存 → hosts 文件 → 本地 DNS → 根域名服务器 → 顶级域名服务器 → 权威 DNS）
3. **TCP 连接**：三次握手建立连接
4. **TLS 握手**（HTTPS）：证书验证、密钥协商
5. **发送 HTTP 请求**：构造请求报文，发送给服务端
6. **服务端处理**：后端处理请求，生成响应
7. **接收 HTTP 响应**：收到状态码、头部、body
8. **渲染页面**：
   - 解析 HTML → DOM 树
   - 解析 CSS → CSSOM 树
   - 合并成 Render 树
   - Layout（回流）：计算元素位置大小
   - Paint（重绘）：绘制像素
   - Composite：合成图层显示
9. **连接关闭或复用**：四次挥手或 HTTP keep-alive 复用

</details>

---

## 八、编译 & App 启动

### Q23. Xcode 按下运行按钮到 App 打开，整个编译过程？

<details>
<summary>点击展开答案</summary>

**编译阶段**：

| 步骤 | 产物 | 工具 | 说明 |
|------|------|------|------|
| 1. 预处理 | `.i` 文件（展开的源文件） | `clang -E` | 宏展开、头文件导入、条件编译 |
| 2. 编译 | `.s` 文件（汇编） | `clang -S` | 词法分析→语法分析→语义分析→IR 优化→生成汇编 |
| 3. 汇编 | `.o` 文件（目标文件） | `clang -c` / `as` | 汇编 → 机器码 |
| 4. 链接 | `Mach-O` 可执行文件 | `ld` | 合并 .o 文件和静态库，解析符号引用 |

**链接阶段详细**：
- 符号解析：将每个目标文件中的符号引用关联到确切的定义
- 重定位：修正代码和数据中的地址引用
- 合并段：将所有 .o 的 `__TEXT`、`__DATA` 等段合并

**启动阶段**：
1. **exec()**：内核加载 Mach-O，创建进程
2. **dyld（动态链接器）**：
   - 加载依赖的动态库（`UIKit`、`Foundation` 等）
   - Rebase（修正内部指针）
   - Bind（绑定外部符号）
   - ObjC Runtime 初始化（注册类、加载 Category）
   - 调用 `+load` 方法
3. **main() → UIApplicationMain()**
4. **AppDelegate 生命周期**

</details>

---

## 九、UI 相关

### Q24. Bounds 和 Frame 的区别？

<details>
<summary>点击展开答案</summary>

| 维度 | Frame | Bounds |
|------|-------|--------|
| 坐标系 | 以**父视图**左上角为原点 | 以**自身**左上角为原点 |
| 含义 | 视图在父视图中的**位置和大小** | 视图自身的**可见区域** |
| 原点 | `frame.origin` 相对于父视图 | `bounds.origin` 默认 (0, 0) |
| 影响 | 改变 frame 会改变位置 | 改变 bounds.origin 会影响子视图 |

**关键区别——滚动原理**：
- 修改 `scrollView.bounds.origin` 时，scrollView 自身不动，但其**子视图**的显示位置改变
- 本质上是改变了 scrollView 内部坐标系的映射关系

```objc
// 关键理解：
view.frame.origin = (50, 100)   // view 在父视图中的位置
view.bounds.origin = (0, 0)     // view 自身坐标原点（滚动后可非零）
```

</details>

### Q25. 点击事件的响应过程？

<details>
<summary>点击展开答案</summary>

iOS 事件响应分为两大阶段：**事件传递（Hit-Testing）** 和 **响应链（Responder Chain）**。

**事件传递（自上而下找目标）**：
1. 触摸发生后，`UIApplication` → `UIWindow`，调用 `-hitTest:withEvent:`
2. `hitTest` 内部调用 `-pointInside:withEvent:` 检查触点是否在自身范围内
3. 若在范围内，按**从后往前**的顺序遍历子视图（`subviews` 逆序），对每个子视图递归调用 `hitTest`
4. 找到最深层的、可响应的（`userInteractionEnabled=YES`, `hidden=NO`, `alpha&gt;0.01`）子视图
5. 这个视图就是 `First Responder`

**响应链（自下而上传递事件）**：
```
触摸视图
  → 父视图
    → 父视图的 ViewController
      → UIWindow
        → UIApplication
          → UIApplicationDelegate
```

- 如果 `First Responder` 不处理（或调用了 `[super touchesBegan:]`），事件沿响应链向上传递
- 任何 `UIResponder` 都可以重写 `touchesBegan:` 等方法来处理事件
- `UIGestureRecognizer` 在事件传递到 view 之前有机会识别手势

</details>

---

## 十、操作系统基础

### Q26. 为什么要有虚拟内存？分段和分页？

<details>
<summary>点击展开答案</summary>

**为什么要虚拟内存**：
1. **隔离性**：每个进程拥有独立的虚拟地址空间，进程间不能直接访问对方内存
2. **安全性**：进程无法越界访问，保护操作系统和其他进程
3. **内存扩展**：虚拟地址空间可以远大于物理内存（通过换页/swapping），让程序认为自己有更大内存
4. **简化编程**：程序员不需要关心物理内存的分配和碎片问题
5. **共享内存**：多个进程可以映射到同一块物理内存（如动态库）

**分段（Segmentation）**：
- 按逻辑单元划分：代码段、数据段、栈段、堆段
- 每段有独立的基址+长度，不同段可以有不同权限（r/w/x）
- 优点：符合程序逻辑结构，权限控制方便
- 缺点：段大小不一，产生外部碎片，内存利用率低

**分页（Paging）**：
- 将虚拟地址空间和物理内存都划分为**固定大小**的页（通常 4KB，iOS 16KB）
- 通过页表（Page Table）将虚拟页映射到物理页帧
- 优点：无外部碎片，换页灵活，支持 Copy-on-Write
- 缺点：页表本身占用内存（通过多级页表优化）

**现代 OS 用段页式**：先分段（逻辑划分），段内分页（物理管理）。macOS/iOS 使用分页机制。

</details>

### Q27. 进程的内存空间分别存什么？

<details>
<summary>点击展开答案</summary>

```
高地址
+---------------------------+
|   内核空间 (Kernel)         |  操作系统内核
+---------------------------+
|   栈 (Stack)               |  局部变量、函数调用帧、返回地址
|   v 向下增长                |
+---------------------------+
|   ...空闲空间...            |
+---------------------------+
|   ^ 向上增长                |
|   堆 (Heap)                |  malloc/new 分配的动态内存
+---------------------------+
|   未初始化数据 (.bss)       |  未初始化的全局/静态变量（默认 0）
+---------------------------+
|   已初始化数据 (.data)      |  已初始化的全局/静态变量
+---------------------------+
|   常量区 (.rodata)          |  字符串常量、const 常量
+---------------------------+
|   代码段 (.text)           |  机器指令（只读）
+---------------------------+
低地址
```

**函数调用时为什么压栈？**
- 保存调用者的**返回地址**（函数执行完回到哪）
- 保存调用者的**帧指针**（`rbp`/`fp`），恢复调用者的栈帧
- 为局部变量分配空间
- 保存被调用者需要保护的**寄存器**（callee-saved registers），如 `rbx`, `r12-r15`

</details>

### Q28. strlen() 和 sizeof() 作用于字符串时的区别？

<details>
<summary>点击展开答案</summary>

| 维度 | `strlen()` | `sizeof()` |
|------|-----------|-----------|
| 本质 | 函数（运行时） | 运算符（编译期） |
| 计算内容 | 字符串实际字符数（不含 `\0`） | 变量/类型占用的内存字节数 |
| 对字符数组 | 运行时扫描到 `\0` | 编译期数组大小（含 `\0`） |
| 对指针 | 运行时扫描指针指向的内容 | 指针本身的大小（8 字节 64 位） |

```c
char str1[] = "hello";     // 数组
char *str2 = "hello";      // 指针

strlen(str1);  // 5
sizeof(str1);  // 6（数组大小，含 '\0'）

strlen(str2);  // 5
sizeof(str2);  // 8（指针大小，64 位系统）
```

**面试陷阱**：当数组作为函数参数传递时会退化为指针：
```c
void func(char arr[]) {
    sizeof(arr); // 8（是指针大小！不是数组大小）
}
```

</details>

---

## 十一、算法 & 智力题

### Q29. O(1) 复杂度删除链表节点（给定节点地址）

<details>
<summary>点击展开答案</summary>

**思路**：不遍历找前驱节点，而是**把下一个节点的值复制到当前节点，然后删除下一个节点**。

```c
void deleteNode(ListNode *node) {
    if (node == NULL || node->next == NULL) return; // 尾节点无法这样删
    
    ListNode *next = node->next;
    node->val = next->val;        // 复制下一个节点的值
    node->next = next->next;      // 跳过下一个节点
    free(next);                   // 释放下一个节点
}
```

**边界条件**：
- 节点为尾节点时，此方法无效（因为没有下一个节点可替换），需要遍历到前驱
- 节点为空时返回
- 这是 O(1) 的关键：利用**给定节点地址**直接操作，不是从头找

</details>

### Q30. 两个栈实现队列

<details>
<summary>点击展开答案</summary>

**思路**：入队栈（inStack）+ 出队栈（outStack）。

```objc
@interface QueueWithTwoStacks : NSObject
@property (nonatomic, strong) NSMutableArray *inStack;
@property (nonatomic, strong) NSMutableArray *outStack;
@end

@implementation QueueWithTwoStacks

- (void)enqueue:(id)obj {
    [self.inStack addObject:obj];  // O(1)
}

- (id)dequeue {
    if (self.outStack.count == 0) {
        // 出队栈为空时，将入队栈全部倒入出队栈
        while (self.inStack.count > 0) {
            [self.outStack addObject:[self.inStack lastObject]];
            [self.inStack removeLastObject];
        }
    }
    id obj = [self.outStack lastObject];
    [self.outStack removeLastObject];
    return obj;  // 均摊 O(1)
}

@end
```

**复杂度分析**：
- enqueue：O(1)
- dequeue：均摊 O(1)（每个元素最多入栈两次、出栈两次）
- 空间：O(n)

</details>

### Q31. 字符串全排列

<details>
<summary>点击展开答案</summary>

**回溯算法**：

```objc
- (NSArray<NSString *> *)permute:(NSString *)str {
    NSMutableArray *result = [NSMutableArray array];
    NSMutableString *track = [NSMutableString string];
    NSMutableArray *used = [NSMutableArray array];
    for (int i = 0; i < str.length; i++) {
        [used addObject:@(NO)];
    }
    [self backtrack:str result:result track:track used:used];
    return result;
}

- (void)backtrack:(NSString *)str result:(NSMutableArray *)result
            track:(NSMutableString *)track used:(NSMutableArray *)used {
    if (track.length == str.length) {
        [result addObject:[track copy]];
        return;
    }
    for (int i = 0; i < str.length; i++) {
        if ([used[i] boolValue]) continue;
        [used[i] = @(YES)];
        [track appendFormat:@"%C", [str characterAtIndex:i]];
        [self backtrack:str result:result track:track used:used];
        [track deleteCharactersInRange:NSMakeRange(track.length - 1, 1)];
        [used[i] = @(NO)];
    }
}
```

时间复杂度：O(n!)，空间复杂度：O(n)（递归深度）。

</details>

### Q32. 二叉树的镜像

<details>
<summary>点击展开答案</summary>

```objc
- (TreeNode *)mirrorTree:(TreeNode *)root {
    if (root == nil) return nil;
    // 交换左右子节点
    TreeNode *tmp = root.left;
    root.left = [self mirrorTree:root.right];
    root.right = [self mirrorTree:tmp];
    return root;
}
```

时间 O(n)，空间 O(h)（递归栈深度）。

</details>

### Q33. 连续子数组的最大和（DP）

<details>
<summary>点击展开答案</summary>

**Kadane 算法**：

```objc
- (int)maxSubArray:(NSArray<NSNumber *> *)nums {
    int maxSum = [nums[0] intValue];
    int curSum = [nums[0] intValue];
    for (int i = 1; i < nums.count; i++) {
        int num = [nums[i] intValue];
        // dp[i] = max(dp[i-1] + num, num)
        curSum = MAX(curSum + num, num);
        maxSum = MAX(maxSum, curSum);
    }
    return maxSum;
}
```

时间 O(n)，空间 O(1)。

</details>

### Q34. 天平称重问题：10 筐物品，9 筐 100g，1 筐 90g，找最少次数

<details>
<summary>点击展开答案</summary>

**不用砝码（只用天平比大小）**：
- 二分法：5 vs 5 → 轻的在轻的一侧 → 2 vs 2 → 1 vs 1
- 最少需要 **3 次**

**有任意质量砝码时（用秤称重量）**：
- **1 次！** 从第 i 筐取出 i 个物品放一起称重：
  - 第 1 筐取 1 个，第 2 筐取 2 个……第 10 筐取 10 个
  - 理论总重（若全是 100g）：`(1+2+...+10) × 100 = 5500g`
  - 实际称得重量，差值 `(5500 - 实际) / 10` = 哪一筐是 90g
  - 例如差 30g → 第 3 筐（30/10=3）

</details>

### Q35. 遍历子 View，奇偶层分别染色（递归+迭代）

<details>
<summary>点击展开答案</summary>

```objc
// 递归方案 — 无额外参数：用 BOOL 判断
- (void)colorViewsRecursive:(UIView *)root level:(NSInteger)level {
    if (level % 2 == 0) {
        root.backgroundColor = [UIColor redColor];   // 偶数层
    } else {
        root.backgroundColor = [UIColor blueColor];  // 奇数层
    }
    for (UIView *subview in root.subviews) {
        [self colorViewsRecursive:subview level:level + 1];
    }
}

// 不用 level 参数的优化：用 root 的 tag 或利用栈传层号
// 实际上 level 参数是必要的，优化掉的说法不太合理

// 迭代方案（BFS）
- (void)colorViewsIterative:(UIView *)root {
    NSMutableArray *queue = [NSMutableArray arrayWithObject:root];
    NSInteger level = 0;
    while (queue.count > 0) {
        NSInteger size = queue.count;
        UIColor *color = (level % 2 == 0) ? [UIColor redColor] : [UIColor blueColor];
        for (NSInteger i = 0; i < size; i++) {
            UIView *node = queue.firstObject;
            [queue removeObjectAtIndex:0];
            node.backgroundColor = color;
            [queue addObjectsFromArray:node.subviews];
        }
        level++;
    }
}
```

</details>

### Q36. 概率题：F() 返回 0 概率 0.3，返回 1 概率 0.7，设计 G() 等概率返回 0/1

<details>
<summary>点击展开答案</summary>

**思路**：调用 F() 两次，利用结果不对称的对称性。

```
调用两次 F():
  00: P = 0.3 × 0.3 = 0.09
  01: P = 0.3 × 0.7 = 0.21  ← 用这个 → 返回 0
  10: P = 0.7 × 0.3 = 0.21  ← 用这个 → 返回 1  
  11: P = 0.7 × 0.7 = 0.49  → 重新来
```

```objc
- (int)G {
    while (YES) {
        int a = F();
        int b = F();
        if (a == 0 && b == 1) return 0;
        if (a == 1 && b == 0) return 1;
        // 00 或 11 则重新来
    }
}
```

**等概率返回 [0, 1000]**：`G()` 等概率返回 0/1，调用 10 次组成一个 10 位二进制数（0~1023），若结果在 0~1000 内则返回，否则重来。

</details>

---

## 十二、综合 & 软技能

### Q37. OC 的垃圾回收 vs Java GC？

<details>
<summary>点击展开答案</summary>

| 维度 | OC (ARC) | Java (GC) |
|------|----------|-----------|
| 机制 | 引用计数（编译期插入 retain/release） | 可达性分析（运行时 GC 线程） |
| 时机 | 引用计数归零立即释放 | GC 触发时才回收（不确定） |
| 暂停 | 无 STW（Stop The World） | 有 STW（GC 暂停应用线程） |
| 循环引用 | 需要手动处理（weak/unsafe_unretained） | GC 自动检测并回收 |
| 性能 | 确定性高、延迟低 | 吞吐量高、分配快 |
| 内存碎片 | 有 | 通过压缩整理消除 |

OC 历史上曾有过 GC（macOS 10.5-10.8），但已在 macOS 10.8 废弃，iOS 从未支持 GC。

</details>

### Q38. Cookie 和 Session 的区别？

<details>
<summary>点击展开答案</summary>

| 维度 | Cookie | Session |
|------|--------|---------|
| 存储位置 | 客户端浏览器 | 服务端 |
| 安全性 | 较低（可被篡改、窃取） | 较高（数据在服务端） |
| 容量 | 小（~4KB） | 大（取决于服务端内存） |
| 生命周期 | 可设置过期时间 | 默认会话结束时失效 |
| 实现 | HTTP 头部字段 | 依赖 Cookie 传递 Session ID |

**工作流程**：
1. 用户登录，服务端创建 Session，返回 Session ID 通过 Set-Cookie
2. 浏览器后续请求自动携带 Cookie（含 Session ID）
3. 服务端根据 Session ID 验证用户身份

</details>

### Q39. iOS 和 Android App 为什么不能通用？

<details>
<summary>点击展开答案</summary>

| 维度 | iOS | Android |
|------|-----|---------|
| 开发语言 | Swift/ObjC | Kotlin/Java |
| 系统框架 | UIKit/SwiftUI | Android SDK/Jetpack |
| 编译目标 | ARM64 Mach-O | DEX/ART |
| UI 渲染 | Core Animation + 离屏合成 | Skia + SurfaceFlinger |
| 沙盒机制 | 严格沙盒，App 间隔离 | 较宽松，有文件系统访问 |
| 生命周期 | UIApplication + AppDelegate | Activity/Fragment/Service |
| 后台限制 | 严格限制 | 相对宽松 |
| 通知系统 | APNs | FCM + 自启动 |

**跨平台方案**：React Native、Flutter、UniApp（使用统一的 JS/Dart 层，调用各平台原生能力）。

</details>

### Q40. 路由器和交换机的区别？

<details>
<summary>点击展开答案</summary>

| 维度 | 路由器 | 交换机 |
|------|--------|--------|
| 工作层级 | 网络层（L3） | 数据链路层（L2） |
| 转发依据 | IP 地址 | MAC 地址 |
| 功能 | 连接不同网络（WAN↔LAN） | 连接同一网络内设备 |
| 家用场景 | 光猫/路由器：连接外网 | 交换机：扩展局域网口 |

**面试官想听的家用场景**：
- 路由器：连接光猫，提供 WiFi，负责拨号上网、NAT 转换、DHCP 分配 IP
- 交换机：路由器端口不够用时接交换机扩展有线口，或者组建局域网

</details>

---

*持续更新中——基于真实面经整理，覆盖字节、美团、阿里、蘑菇街等大厂 iOS 面试考点。*
