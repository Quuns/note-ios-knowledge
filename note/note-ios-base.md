# iOS 基础知识
strong/copy/assign/weak 属性 的区别

## 一、先总览一句话区别
- **strong**：强引用，持有对象，不让销毁
- **copy**：拷贝一份新对象，杜绝外部篡改
- **weak**：弱引用，不持有，对象销毁自动置 `nil`
- **assign**：纯赋值，不引用、不管理内存，对象销毁会**野指针**

---

# 二、逐个详细拆解

## 1. strong（强引用）
### 原理
- 引用计数 +1
- **持有对象**，只要有 `strong` 指向，对象就不会被释放

### 适用类型
- 所有 OC 对象：`UIView`、`NSString`、数组、模型、自定义对象

### 特点
- 正常持有，生命周期跟着持有者走
- 容易**循环引用**（两个类互相 strong 引用）

### 示例
```objc
@property (nonatomic, strong) UIButton *btn;
@property (nonatomic, strong) NSArray *arr;
```

---

## 2. copy（拷贝）
### 原理
- 赋值时**重新拷贝一份内存**，不是指向同一块地址
- 分 **浅拷贝 / 深拷贝**
- 原对象变了，当前属性**不会跟着变**

### 适用类型
- `NSString`、`NSArray`、`NSDictionary` 这类**可变/不可变**都用 copy
- 防止外部传入 `NSMutableString` 后篡改内部值

### 为什么字符串必须用 copy？
如果用 `strong`：
外部传一个 `NSMutableString`，外部修改，你的内部属性**也跟着变**，Bug 爆炸。
用 `copy`：直接复制一份，外部怎么改跟我无关。

### 示例
```objc
@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSArray *list;
```

---

## 3. weak（弱引用）
### 原理
- 引用计数**不+1**
- 不持有对象
- 对象销毁时，**自动置为 nil**，不会野指针崩溃

### 适用类型
- UI 控件 `delegate`
- 避免循环引用：代理、Block 里引用自己
- Storyboard/Xib 拖线控件

### 典型场景
```objc
@property (nonatomic, weak) id<UITableViewDelegate> delegate;
@property (nonatomic, weak) UIView *subView;
```

### 关键优势
**不会野指针，对象没了自动变 nil，发消息不崩溃**

---

## 4. assign（直接赋值）
### 原理
- 不做引用计数管理
- 只是**单纯地址赋值**
- 对象销毁后，指针**不置空，变成野指针**，访问直接崩溃

### 适用类型
- 基本数据类型：`int`、`float`、`BOOL`、`NSInteger`
- 枚举、结构体

### 绝对不能用 assign 修饰 OC 对象！
会出现：**对象已销毁，指针还在，访问直接 Crash**

### 示例
```objc
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, assign) BOOL isOpen;
```

---

# 三、四大属性横向对比表（直接收藏）
| 修饰符 | 引用计数 | 是否持有对象 | 对象销毁后 | 适用场景 |
|--------|----------|--------------|------------|----------|
| strong | +1 | 持有 | 指针还在，内存已释放 | 普通 OC 对象、UI控件、模型 |
| copy | 新对象拷贝 | 持有新副本 | 正常释放 | NSString、NSArray、字典 |
| weak | 不变 | 不持有 | **自动置 nil** | 代理、子视图、防循环引用 |
| assign | 不变 | 不持有 | **野指针、崩溃** | 基本数据类型、枚举、结构体 |

---

# 四、面试必背使用规范（直接背）
1. **普通 OC 对象、UI控件** → 用 **strong**
2. **NSString / NSArray / NSDictionary** → 必须用 **copy**
3. **Delegate、协议代理** → 必须用 **weak**
4. **基本数据类型 int/BOOL/NSInteger** → 用 **assign**
5. **Block 属性** → 用 **copy**

---

# 五、循环引用避坑口诀
- 互相 **strong** = 循环引用 = 内存泄漏
- 一方 strong、一方 weak 即可破解
- Block 内部用到 self 一定要：`__weak typeof(self) weakSelf = self;`

---

# 六、最简一句话记忆
- **strong 留住对象**
- **copy 复制防篡改**
- **weak 不持有、自动置空**
- **assign 只给基本类型用**

你需要我再给你补一份 **ARC 下内存管理底层原理 + MRC 区别** 吗？可以一起整理成一套面试全集。



# iOS Objective-C 核心面试知识点合集（Markdown 收藏版）
## 一、weak 底层原理
1. `weak` 引用**不改变对象引用计数**，仅做地址记录。
2. Runtime 维护全局 **SideTable 弱引用散列表**，所有 weak 指针地址都会存入表中。
3. 当对象引用计数为 0 准备销毁时：
   - Runtime 遍历弱引用表中所有指向该对象的指针
   - **全部自动置为 nil**
4. 特性：
   - 给 nil 发消息不崩溃
   - 只可修饰 OC 对象，不能修饰基本类型
   - 用于代理、子视图、打破循环引用

---

## 二、属性不写任何修饰符，系统默认规则
默认写法：
```objc
@property NSString *name;
```
**等价于**：
```objc
@property (atomic, readwrite, assign) NSString *name;
```

默认三合一：
- 原子性：**atomic**（加锁、线程安全、性能低）
- 读写：**readwrite**
- 内存语义：**assign**

### 致命坑
默认 `assign` 修饰 OC 对象，对象销毁后**不会置空**，直接变成野指针，访问必崩溃。
**开发严禁省略属性修饰符**。

### atomic / nonatomic 区别
- `atomic`：属性读写加自旋锁，线程安全，性能差
- `nonatomic`：不加锁，线程不安全，性能高
**日常开发全部用 nonatomic**

---

## 三、Category 分类 与 Extension 类扩展 区别
### 1. Extension（匿名扩展）
- 写在 `.m` 文件内部，私有不可见
- **可以添加成员变量 ivar、属性**
- 不能单独建文件，只能依附当前类
- 作用：定义**私有属性、私有方法**

### 2. Category 分类
- 可独立新建 `.h/.m` 文件
- **不能直接添加成员变量**（可用关联对象模拟属性）
- 可给系统类/自定义类扩展方法
- 可**覆盖原类同名方法**（分类方法优先级更高）

### 3. 对比汇总
| 对比项 | Extension | Category |
|--------|-----------|----------|
| 存放位置 | 本类 .m 内部 | 独立分类文件 |
| 成员变量 | 支持添加 ivar | 不支持直接加 ivar |
| 访问权限 | 私有 | 公开 |
| 覆盖方法 | 不支持 | 可覆盖原类方法 |
| 用途 | 私有属性、私有方法 | 扩展方法、系统类增强 |

### 4. Category 底层实现原理
1. 编译期：收集分类的**方法、协议、属性**信息。
2. Runtime 初始化时，通过 `class_addMethod` **动态把分类方法插入原类方法列表**。
3. 类的方法列表是链表结构，**分类方法插在原类方法前面**，所以同名会覆盖。
4. 分类没有额外内存布局，**不能自动合成 ivar**，只能用 `objc_setAssociatedObject` 关联对象间接实现属性。

---

## 四、ARC 与 MRC 理解
### MRC 手动引用计数
- 开发者手动管理内存：`alloc / retain / release / autorelease`
- 内存规则：`alloc/new/copy/mutableCopy` 创建的对象，必须手动 `release`
- 无 `weak` 关键字，只能用 `__unsafe_unretained`，对象销毁易野指针
- 开发繁琐、易内存泄漏、易崩溃

### ARC 自动引用计数
- 本质：**编译期特性**，不是运行时；编译器自动在合适位置插入 `retain/release/autorelease`
- 不用手动写内存管理代码
- 支持 `weak` 自动置空，杜绝野指针
- 自动推断 `strong/copy/weak/assign` 语义

### ARC & MRC 核心区别
1. MRC 手动写内存代码；ARC 编译器自动插入
2. MRC 允许手动调用 `retain/release`；ARC 禁止调用
3. ARC 有 `weak` 自动置空；MRC 无
4. ARC 编译期优化更强；MRC 全靠开发者规范

---

## 五、Block 全面理解 + ARC/MRC 区别 + 使用注意
### 1. Block 本质
- Block 是**封装代码逻辑+外部变量捕获的 OC 对象**
- 底层是 C 结构体 `__block_impl`
- 三种类型：
  1. `NSGlobalBlock`：全局块，无外部捕获变量
  2. `NSStackBlock`：栈上块，捕获外部变量，生命周期随栈
  3. `NSMallocBlock`：堆上块，被持有、生命周期可控

### 2. ARC 与 MRC 下 Block 区别
#### MRC
- Block 默认是 **NSStackBlock**（栈上）
- 出作用域立刻销毁，必须手动 `copy` 到堆
- Block 属性必须用 `copy`

#### ARC
- 编译器**自动将栈上 Block 拷贝到堆**
- 赋值给 `strong/copy` 变量自动升级为 `NSMallocBlock`
- 不用手动 copy，但**属性依旧必须用 copy**

### 3. Block 为什么属性必须用 copy
- Block 原始在栈上，生命周期极短
- `copy` 会把 Block 拷贝到**堆上**，由对象持有管理生命周期
- 规范写法：
```objc
@property (nonatomic, copy) void(^actionBlock)(void);
```

### 4. 变量捕获规则
- 普通局部变量：**值捕获**，只读不可改
- static / 全局变量：**指针捕获**，可修改
- `__block` 修饰变量：变量迁移到堆上，Block 内部可修改

### 5. Block 循环引用（ARC 标准写法）
```objc
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    // 内部业务逻辑，安全使用 strongSelf
};
```

### 6. Block 使用注意事项
1. Block 属性**永远用 copy**
2. 内部引用 self 必须用弱引用打破循环引用
3. 不要在 Block 内长期持有大对象，避免内存泄漏
4. 栈 Block 不能跨函数传递，必须拷贝到堆
5. 修改外部局部变量必须加 `__block`

---

## 六、速记口诀
1. weak：Runtime 弱引用表，不计数，销毁自动置 nil，防野指针。
2. 属性默认：不加修饰 = atomic+readwrite+assign，OC 对象必野指针。
3. Extension 私有可加成员变量；Category 可扩方法、运行时插入方法列表、不能直接加 ivar。
4. MRC 手动管内存；ARC 编译期自动插内存代码，自带 weak 容错。
5. Block 分全局/栈/堆，属性必写 copy；ARC 自动栈拷堆，用 weakSelf 解循环引用。

已纯 Markdown 结构化，直接复制存备忘录、笔记软件即可。