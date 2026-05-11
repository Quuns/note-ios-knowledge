根据你提供的《Objective-C高级编程》第2章内容，以下是为面试准备的Blocks知识点整理：

### 一、Blocks 核心概念与基础
1.  **定义**：Blocks 是 C 语言的扩充功能，本质是**带有自动变量（局部变量）的匿名函数**。
2.  **与其他语言的对应**：在计算机科学中，此概念被称为**闭包**或 **lambda 计算**等。
3.  **优点**：简洁地实现回调，特别适用于异步操作。

### 二、Block 语法与使用
1.  **完整语法**：
    `^ 返回值类型 (参数列表) { 表达式 }`
2.  **省略形式**：
    - 无返回值、无参数时，可省略返回值和参数列表。
    - 有返回值时，编译器能根据return语句推断返回类型。
3.  **Block 类型变量声明**：
    `返回值类型 (^变量名)(参数列表) = Block;`
    - **使用 typedef 简化声明**：`typedef 返回值类型 (^Block类型名)(参数列表);`

### 三、截获自动变量值（核心机制）
1.  **截获行为**：Block 会在执行时**截获（保存）** 其内部使用的自动变量的**瞬间值**。
    - 即使在 Block 调用前修改了外部自动变量的值，Block 内部仍使用其定义时的值。
2.  **不可修改**：默认情况下，**不能修改**截获的自动变量的值，否则编译报错。
3.  **特殊处理**：
    - 对于 `NSMutableArray` 等对象，可以修改变量值（如调用 `addObject:`），但**不能对该变量本身赋值**。
    - **C 语言数组**不能被 Block 截获，需要使用指针替代。

### 四、__block 说明符
1.  **作用**：允许在 Block 内部**修改**外部的自动变量。
2.  **实现原理**：带有 `__block` 的变量会被编译器转换为一个**结构体实例**。Block 会持有指向该结构体的指针，通过指针间接访问和修改变量值。
3.  **`__forwarding` 指针**：结构体中有一个 `__forwarding` 指针，无论 `__block` 变量在栈上还是被复制到堆上，都能通过它正确访问到同一个变量。

### 五、Block 的实质
通过 `clang -rewrite-objc` 可以看到，Block 的实质是一个 **Objective-C 对象**。
- Block 结构体包含：
    - `isa` 指针：指向 Block 的类，决定 Block 的存储域。
    - `FuncPtr`：指向 Block 逻辑对应 C 函数的函数指针。
    - 被截获的自动变量：作为结构体的成员变量。

### 六、Block 的存储域与内存管理（面试重点）
根据存储位置不同，Block 分为三类：

| Block 类 | 存储域 | 说明 |
| :--- | :--- | :--- |
| **`_NSConcreteStackBlock`** | **栈** | 默认情况下的 Block 都是栈上的。 |
| **`_NSConcreteGlobalBlock`** | **数据区域 (.data)** | 在以下情况产生：① 定义在全局区域；② 不截获任何自动变量。 |
| **`_NSConcreteMallocBlock`** | **堆** | 当对栈上的 Block 执行 `copy` 操作时，会复制到堆上。 |

1.  **从栈到堆的复制**：
    - **必须复制**的原因：栈上的 Block 在其作用域结束后会被释放，超出作用域调用会导致崩溃。
    - **何时自动复制**：在 **ARC 环境**下，以下情况编译器会自动将栈 Block 复制到堆上：
        - Block 作为**函数返回值**。
        - 将 Block 赋值给 `__strong` 修饰的成员变量或变量。
        - 向包含 `usingBlock` 参数的 Cocoa 或 GCD API 传递 Block。
    - **手动复制**：在其他情况下（如 `[NSArray initWithObjects:]` 传递 Block），需要手动调用 `copy` 方法。

2.  **copy/release 操作效果**：

| Block 类 | 调用 `copy` 的效果 |
| :--- | :--- |
| `_NSConcreteStackBlock` | **从栈复制到堆** |
| `_NSConcreteGlobalBlock` | **什么也不做** (引用计数不变) |
| `_NSConcreteMallocBlock` | **增加引用计数** (retain count +1) |

### 七、Block 与对象
1.  **`__strong` 对象截获**：Block 会持有（retain）截获的 `__strong` 对象。当 Block 从栈复制到堆时，该对象也被堆上的 Block 持有。
2.  **`__weak` 对象截获**：Block 弱引用该对象，修改变量指向的值，且 nil 会被复制给变量，可避免循环引用。
3.  **`copy/dispose` 函数**：编译器为 Block 生成 `_copy` 和 `_dispose` 函数，分别在 Block 复制到堆时持有对象，在 Block 从堆释放时释放对象。

### 八、Block 循环引用（面试重中之重）
1.  **问题根源**：对象强引用 Block，Block 强引用对象（如 `self`），导致互相持有，无法释放。
    ```objc
    // 典型循环引用示例
    @interface MyObject : NSObject
    @property (nonatomic, copy) void (^blk)(void);
    @end

    // ...
    self.blk = ^{
        NSLog(@"%@", self); // self -> blk, blk -> self
    };
    ```
2.  **解决方案**：
    - **方案一：`__weak`（推荐）**：在 Block 外部声明一个 `__weak` 指向 `self` 的变量，在 Block 内部使用。
        ```objc
        __weak typeof(self) weakSelf = self;
        self.blk = ^{
            NSLog(@"%@", weakSelf);
        };
        ```
    - **方案二：`__block`（需执行Block）**：声明一个 `__block` 变量指向 `self`，在 Block 内部使用后将其置为 `nil`，**必须确保 Block 被执行**。
    - **方案三：`__unsafe_unretained`**：功能类似 `__weak`，但指向对象释放后不会自动置为 nil，可能导致悬垂指针，不推荐。

这些是书中关于 Blocks 的全部核心知识点，掌握了这些在面试中就足以应对 Block 相关的大部分问题。祝你面试顺利！