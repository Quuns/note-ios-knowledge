# iOS Objective-C 关联对象 面试完整版（用法+底层原理+面试必问）
## 一、先搞懂：关联对象是干嘛的
**核心作用**：给**已有的类（系统类/自己类）动态添加成员变量**，尤其是**分类（Category）不能直接加成员变量**，就用关联对象补。

### 前置知识点：为什么 Category 不能加成员变量？
1. 分类编译后是结构体，**没有 ivar 列表**；
2. 类的实例变量内存布局编译时就固定了，运行时不能直接扩容加 ivar；
3. 分类只能加方法、属性（只有setter/getter，没有底层ivar）。

👉 所以：**分类想存值 → 必须用关联对象**。

---

## 二、关联对象 4 种绑定策略（面试必背）
```objc
// 关键函数
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```

### 四种 policy 枚举
```objc
enum {
    OBJC_ASSOCIATION_ASSIGN = 0;        // 弱引用，不retain、不copy
    OBJC_ASSOCIATION_RETAIN_NONATOMIC;  // 强引用，非原子
    OBJC_ASSOCIATION_COPY_NONATOMIC;    // copy，非原子
    OBJC_ASSOCIATION_RETAIN;            // 强引用，原子
    OBJC_ASSOCIATION_COPY;              // copy，原子
};
```

### 记忆口诀
- **ASSIGN**：纯弱引用，野指针风险，一般用于基本数据类型、代理
- **RETAIN**：强引用，对象
- **COPY**：字符串/block 用，防止外部被修改
- 带 `NONATOMIC`：不加锁、效率高；不带：原子加锁、线程安全

---

## 三、实战用法（分类添加属性 标准写法）
### 示例：给 UIView 加一个 name 属性
```objc
// UIView+Name.h
#import <UIKit/UIKit.h>
@interface UIView (Name)
@property (nonatomic, copy) NSString *name;
@end
```

```objc
// UIView+Name.m
#import "UIView+Name.h"
#import <objc/runtime.h>

// 1. 定义唯一key，常用静态常量地址做key
static const void *kViewNameKey = &kViewNameKey;

@implementation UIView (Name)

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, kViewNameKey, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, kViewNameKey);
}

@end
```

### Key 的三种常用写法（面试会问）
1. **静态常量地址**（最标准，推荐）
`static const void *kKey = &kKey;`
2. **@selector 作为 key**
`objc_setAssociatedObject(self, @selector(name), name, policy);`
3. **字符串常量**（不推荐，有字符串复用冲突风险）

### 移除关联对象
```objc
// 移除当前对象所有关联对象
objc_removeAssociatedObjects(self);
```
**注意**：开发不建议随便全局移除，容易删掉别人绑定的关联对象；一般不用手动删，**对象销毁时系统自动清**。

---

## 四、底层原理（面试核心，必问深挖）
### 1. 整体架构
关联对象**不存在对象本身的内存里**，也不存在类里。
底层全局维护了一张 **哈希表 全局统一管理**：
- 全局有一个 **AssociationsManager** 管理者
- 内部维护 **AssociationsHashMap**
- 映射关系：**对象地址 → 关联属性字典**
- 每个对象对应一个小字典：**key → value+policy**

### 2. 底层数据结构层级
1. **AssociationsManager**
   全局单例，内部有自旋锁，保证线程安全。
2. **AssociationsHashMap**
   主哈希表，key：**对象的指针地址**，value：**ObjectAssociationMap**
3. **ObjectAssociationMap**
   单个对象的关联字典，key：**关联key**，value：**AssociationEntry**
4. **AssociationEntry**
   存储：关联的值 + 绑定策略 policy

### 3. 存流程 objc_setAssociatedObject
1. 加自旋锁，保证线程安全
2. 根据 object 地址，在全局大哈希表找对应的子字典
3. 不存在则创建子字典
4. 根据 key 存入 value 和 policy
5. 如果传 nil：**等同于移除该 key 的关联对象**

### 4. 取流程 objc_getAssociatedObject
1. 加锁
2. 通过 object 找全局哈希表中的子字典
3. 通过 key 查找对应的 AssociationEntry
4. 根据 policy 处理内存语义（retain/copy 等）
5. 返回值

### 5. 对象销毁时如何释放关联对象？
- 当对象执行 `dealloc` 时
- runtime 会检查该对象**是否有关联对象**
- 有则遍历全部关联对象，**按对应 policy 执行释放逻辑**（release、copy 释放等）
- 从全局哈希表中移除该对象的所有关联数据

### 6. 关键底层结论（面试必背）
1. 关联对象**不是存在对象自身内存**，是**全局哈希表统一托管**；
2. 对象本身内存大小**不会因为加了关联对象而变化**；
3. 关联对象生命周期**跟随原对象**，原对象销毁，关联对象自动销毁；
4. 底层用**自旋锁**保证哈希表操作线程安全；
5. 设置 value 为 nil 可以**单独移除某一个关联key**。

---

## 五、面试高频问答（直接背）
### Q1：分类为什么不能直接加成员变量？
类的实例变量布局编译时确定，分类不能修改原有内存布局；分类结构体本身也没有 ivar 列表，只能加方法，不能直接存变量，需用关联对象。

### Q2：关联对象的四种策略区别？
ASSIGN 弱引用；RETAIN 强引用；COPY 拷贝；带 NONATOMIC 非原子、无锁，性能更高。

### Q3：关联对象存在哪里？
不在对象、不在类，存在 **全局唯一的关联对象哈希表** 中，由 AssociationsManager 管理。

### Q4：关联对象会不会影响原对象内存大小？
**不会**，原对象内存布局不变，关联对象存在全局表。

### Q5：什么时候会释放关联对象？
原对象 dealloc 时，runtime 自动遍历并按策略释放，全局哈希表清除对应条目。

### Q6：objc_removeAssociatedObjects 慎用原因？
会**删除当前对象所有**关联对象，包括第三方框架绑定的，容易引发崩溃。

---

## 六、面试极简总结（背诵版）
关联对象用于给分类动态添加存储属性，四种内存策略：assign、retain、copy 区分原子/非原子。底层由全局 AssociationsManager 维护双层哈希表，以对象地址和关联 key 做映射，关联对象不占用对象自身内存，生命周期跟随原对象，dealloc 时 runtime 自动回收，底层自旋锁保证线程安全。

我可以给你整理一份**面试口述版逐字稿**，你直接背就能面试开口说，要不要我帮你压缩成1分钟面试背诵版本？


# 一句话直击核心
**OC 分类用 @property 声明属性，编译器绝对不会自动生成 `_xxx` 下划线成员变量，也不会自动生成 setter / getter 实现**。

我给你讲透原理、底层原因、和主类的区别，面试直接能说。

---

## 1. 先对比：主类 vs 分类
### 普通主类（.m 里写 @property）
```objc
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@end
```
在主类里：
编译器**自动帮你做三件事**：
1. 生成**下划线成员变量** `NSString *_name`
2. 自动生成 setter：`setName:`
3. 自动生成 getter：`name`

### 分类里写 @property
```objc
@interface NSObject (Test)
@property (nonatomic, copy) NSString *nickName;
@end
```
在**分类**里：
编译器**啥都不生成**：
❌ 没有 `_nickName` 成员变量
❌ 没有 setter 实现
❌ 没有 getter 实现

只**声明了方法签名**：
- `- (NSString *)nickName;`
- `- (void)setNickName:(NSString *)nickName;`

相当于只是**接口声明**，没有实现、没有存储变量。

---

## 2. 为什么分类不能生成下划线 ivar？
### 原因一：类的实例变量内存布局编译时固定
- 主类的成员变量 `_name` 是**编译时确定内存布局**，排布在对象的内存结构体里。
- 分类是**运行时动态追加**，不能修改原有类的内存布局，**不能往后追加 ivar**。
- 如果允许分类加 ivar，原有对象内存大小会变，会破坏内存对齐、野指针、布局错乱。

### 原因二：分类底层结构根本没有 ivar 列表
分类底层结构体：
```c
struct category_t {
    const char *name;
    struct class_t *cls;
    struct method_list_t *instance_methods;
    struct method_list_t *class_methods;
    struct protocol_list_t *protocols;
    // 没有 ivar_list_t！
};
```
分类结构体**根本没有存放成员变量的列表**，天生就不支持合成 ivar。

---

## 3. 那分类想存值怎么办？
两种方案：
1. **关联对象**（日常开发99%用这个）
   手动写 setter / getter，内部用 `objc_setAssociatedObject` 存值。
2. **静态全局变量**
   所有实例共享同一个变量，**不能每个对象独立存值**，基本不用。

---

## 4. 面试标准口述版（直接背）
分类中使用 @property 只会声明 setter 和 getter 方法签名，**不会自动生成下划线成员变量，也不会自动生成属性实现**。
因为类的实例变量内存布局编译时就固定了，分类是运行时动态加载，无法修改原有对象内存布局；同时分类底层结构体本身也没有 ivar 列表，天生不支持合成成员变量，所以分类要实现属性存储只能用关联对象。

---

我可以帮你把「分类 + @property + 关联对象」整理成**面试1分钟标准答案**，背下来面试官问就能脱口而出，要吗？



# 用大白话+生活化例子，给你彻底讲懂这四层结构
先记住一句总纲：
**关联对象根本不塞到你自己那个对象内存里，也不塞到类里，全部统一放到系统单独开的一个「全局大仓库」里管着。**

---

## 先把四层结构翻译成人话
1. **AssociationsManager**：全局唯一的**仓库管理员**，全程就一个，还带锁，保证线程安全。
2. **AssociationsHashMap**：管理员手里的**大账本（总哈希表）**
3. **对象地址 → 小字典**：账本每一行，对应**一个实例对象**
4. **小字典：key → value+policy**：每个对象自己专属的**小抽屉**，放自己所有关联对象

---

# 举个生活化例子
假设：
你有一个 `UIView *viewA`、还有一个 `UIView *viewB`
你分别给它们用关联对象绑定了名字、年龄。

## 1. 全局只有一个仓库管理员：AssociationsManager
整个APP运行期间，**就这一个管理员**，管所有类的所有关联对象，不分UIView、NSString、自定义类，全归他管。

## 2. 管理员手里有一本大账本：AssociationsHashMap
这本大账本的规则：
**Key：对象的内存地址**
**Value：这个对象专属的一个小字典（小抽屉）**

可以理解成：
账本每一行，写着「某一个对象的地址」，对应一个属于它自己的小抽屉。

## 3. 每个对象对应自己的小抽屉
- `viewA` 地址：0x1234
  对应一个**专属小字典（小抽屉A）**
- `viewB` 地址：0x5678
  对应一个**专属小字典（小抽屉B）**

> 重点：
> viewA 自己的内存里，**根本没装 name、age 这些关联值**
> 它只是一个空壳，真正的值全存在全局大账本里它对应的那个小抽屉里。

## 4. 小抽屉里面：key → value + 内存策略
viewA 的小抽屉里：
| 关联Key       | 存的值     | 策略                  |
|---------------|------------|-----------------------|
| kNameKey      | @"小明"    | COPY_NONATOMIC        |
| kAgeKey       | @20        | RETAIN_NONATOMIC      |

viewB 的小抽屉里：
| 关联Key       | 存的值     | 策略                  |
|---------------|------------|-----------------------|
| kNameKey      | @"小红"    | COPY_NONATOMIC        |

---

# 模拟底层存取过程（对应你写代码的逻辑）
### 你写代码：给 viewA 设置 name
```objc
objc_setAssociatedObject(viewA, kNameKey, @"小明", OBJC_ASSOCIATION_COPY_NONATOMIC);
```
底层干的事：
1. 找全局唯一管理员 `AssociationsManager`，上锁
2. 拿 `viewA` 的内存地址，去大账本 `AssociationsHashMap` 里查
3. 查到 viewA 对应的**小抽屉**
4. 往小抽屉里存入：`kNameKey = @"小明" + COPY策略`
5. 解锁完事

### 你取值：
```objc
NSString *name = objc_getAssociatedObject(viewA, kNameKey);
```
底层：
1. 找管理员、上锁
2. 用 viewA 地址找到专属小抽屉
3. 用 kNameKey 取出对应值，返回

---

# 对象销毁时怎么清？
当 `viewA` 要 dealloc 销毁：
1. Runtime 发现 viewA **在全局大账本里有绑定数据**
2. 找到它对应的小抽屉
3. 按照每个 key 的 policy，该 release 就 release、该清空就清空
4. 从全局大账本里**删掉 viewA 这一行**
5. 彻底释放，不留垃圾

---

# 帮你复盘你原来那段话，现在是不是完全懂了
> 关联对象不存在对象本身的内存里，也不存在类里。
✅ 翻译：值不放在view自己身上，也不放在UIView类里，全放全局仓库

> 底层全局维护了一张 哈希表 全局统一管理
✅ 翻译：系统一本大账本统一管所有关联对象

> • 全局有一个 AssociationsManager 管理者
✅ 唯一仓库管理员，带锁保安全

> • 内部维护 AssociationsHashMap
✅ 管理员手里的大账本

> • 映射关系：对象地址 → 关联属性字典
✅ 账本：对象地址 对应 它自己的专属小抽屉

> • 每个对象对应一个小字典：key → value+policy
✅ 小抽屉里，用唯一key存好值和内存管理策略

---

# 面试极简口述版（你可以直接背）
关联对象不存到对象自身内存，也不存到类中。底层由全局唯一的AssociationsManager管理者维护一张全局哈希表AssociationsHashMap，映射关系是以**对象内存地址**作为Key，Value是该对象专属的子字典；子字典内部再通过关联Key存取对应的数值和内存管理策略，相当于每个对象都有自己独立的关联对象小仓库，统一由系统全局托管。

需要我给你画一张**简易层级结构图（文字版）**，方便你脑子里固化这个模型吗？


# 关联对象完整 Demo（可直接复制运行）
功能：**给 NSObject 加分类，动态新增 nickname、age 两个属性**，用关联对象存储，演示存取、互不干扰、生命周期。

## 1. 新建分类 NSObject+Associated.h
```objc
#import <Foundation/Foundation.h>

@interface NSObject (Associated)

// 分类属性，不会自动生成ivar、不会自动实现set/get
@property (nonatomic, copy) NSString *nickname;
@property (nonatomic, assign) NSInteger age;

@end
```

## 2. 分类实现 NSObject+Associated.m
```objc
#import "NSObject+Associated.h"
#import <objc/runtime.h>

// 定义唯一Key（标准写法：静态常量地址）
static const void *kNicknameKey = &kNicknameKey;
static const void *kAgeKey = &kAgeKey;

@implementation NSObject (Associated)

- (void)setNickname:(NSString *)nickname {
    // COPY 字符串用copy策略
    objc_setAssociatedObject(self, kNicknameKey, nickname, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)nickname {
    return objc_getAssociatedObject(self, kNicknameKey);
}

- (void)setAge:(NSInteger)age {
    // 基本数值用ASSIGN
    objc_setAssociatedObject(self, kAgeKey, @(age), OBJC_ASSOCIATION_ASSIGN);
}

- (NSInteger)age {
    return [objc_getAssociatedObject(self, kAgeKey) integerValue];
}

@end
```

## 3. 主函数测试 main.m
```objc
#import <Foundation/Foundation.h>
#import "NSObject+Associated.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // 两个独立对象
        NSObject *obj1 = [[NSObject alloc] init];
        NSObject *obj2 = [[NSObject alloc] init];
        
        // 给obj1设置关联对象
        obj1.nickname = @"张三";
        obj1.age = 25;
        
        // 给obj2设置关联对象
        obj2.nickname = @"李四";
        obj2.age = 30;
        
        // 取值测试：互不干扰
        NSLog(@"obj1：%@，%ld岁", obj1.nickname, (long)obj1.age);
        NSLog(@"obj2：%@，%ld岁", obj2.nickname, (long)obj2.age);
        
        // 置空测试
        obj1.nickname = nil;
        NSLog(@"obj1置空后昵称：%@", obj1.nickname);
        
    }
    return 0;
}
```

## 4. 输出结果
```
obj1：张三，25岁
obj2：李四，30岁
obj1置空后昵称：(null)
```

---

# 关键演示点（对应底层原理）
1. **分类 @property 没有生成 _nickname、_age 下划线成员变量**
   全靠关联对象存在**全局哈希表**，不在对象本身内存里。

2. **不同对象关联数据互相隔离**
   obj1、obj2 是两个不同内存地址，在全局哈希表里各对应一个「子字典」，数据互不污染。

3. **策略使用规范**
   - 字符串：用 `OBJC_ASSOCIATION_COPY_NONATOMIC`
   - 基本数值包装成 NSNumber：用 `OBJC_ASSOCIATION_ASSIGN`

4. **设为 nil 等价于移除单个关联对象**
   `obj1.nickname = nil` 内部底层就删掉了这个 key 对应的条目。

---

# 面试可以补充的点（结合这个Demo）
- 分类不能自动生成 ivar，所以手动用关联对象补存储；
- 每个对象的关联数据隔离，底层靠「对象地址 -> 子字典」映射；
- 对象销毁时，关联对象**自动跟着释放**，不用手动 `objc_removeAssociatedObjects`。

我再给你写一个**用 @selector 当 Key 的简化版 Demo**，要不要？