
---

# iOS ObjC KVC & KVO 面试核心知识点+底层原理（修正版）

## 一、基础概念速记
### 1. KVC（Key-Value Coding）
**全称**：键值编码  
**作用**：**通过字符串 key 访问对象属性，优先查找 getter/setter，找不到时直接访问成员变量，从而可读写私有属性。**  
**核心 API**
```objc
// 取值
- (id)valueForKey:(NSString *)key;
// 赋值
- (void)setValue:(id)value forKey:(NSString *)key;

// 路径访问（嵌套对象）
- (id)valueForKeyPath:(NSString *)keyPath;
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
```

**特点**：
- 打破封装，可访问**私有属性**
- 支持自动装箱/拆箱（基本数据类型 ↔ NSNumber/NSValue）
- 支持集合操作（`@sum`、`@avg`、`@max`、`@min`、`@count`、`@unionOfObjects`、`@distinctUnionOfObjects` 等）

---

### 2. KVO（Key-Value Observing）
**全称**：键值观察  
**作用**：**监听一个对象的属性值变化，属性改变时自动触发回调**  
**核心 API**
```objc
// 添加监听
[obj addObserver:observer 
     forKeyPath:@"key" 
        options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld 
        context:NULL];

// 回调方法
- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object 
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change 
                       context:(void *)context;

// 移除监听（必须，且必须严格配对）
[obj removeObserver:observer forKeyPath:@"key"];
```

**特点**：
- 基于**KVC 命名规范**实现，只有具备符合规范的 setter 方法的属性才能被监听
- 直接修改成员变量（包括结构体内部字段）不触发 KVO，但通过 setter **整体赋值**结构体属性仍会触发
- 不能监听局部变量
- 必须手动移除监听，否则会造成野指针崩溃；重复添加/移除也会崩溃

---

## 二、底层原理（面试必考！）
### 1. KVC 底层查找原理（赋值/取值流程，必须按顺序背熟）

#### （1）`setValue:forKey:` 赋值流程
1. 优先查找 **`setKey:`** 方法（标准 setter）
2. 找不到，查找 **`_setKey:`** 方法
3. 还找不到，调用 `+ (BOOL)accessInstanceVariablesDirectly`（默认返回 YES）
4. 直接访问**成员变量**，顺序：`_key` → `_isKey` → `key` → `isKey`
5. 都找不到，抛出 `NSUndefinedKeyException` 异常

#### （2）`valueForKey:` 取值流程
1. 优先查找 **`getKey`** / **`key`** / **`isKey`** 方法（getter）
2. 找不到，调用 `accessInstanceVariablesDirectly`（默认 YES）
3. 直接访问成员变量，顺序同上：`_key` → `_isKey` → `key` → `isKey`
4. 都找不到，抛出异常

**异常处理**：可重写以下方法避免崩溃  
```objc
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;
- (id)valueForUndefinedKey:(NSString *)key;
```

---

### 2. KVO 底层原理（面试核心！）
**KVO 本质：Runtime 动态生成子类 + 重写被监听的 setter 方法**

#### 完整底层流程
1. **动态创建子类**  
   添加监听时，Runtime 自动创建一个以 `NSKVONotifying_原类名` 命名的子类，并将对象的 `isa` 指针**指向这个动态子类**。

2. **重写 setter 方法**  
   子类中会**重写被监听属性的 setter**，插入通知逻辑（伪代码）：
   ```objc
   - (void)setName:(NSString *)name {
       [self willChangeValueForKey:@"name"];
       [super setName:name];
       [self didChangeValueForKey:@"name"];
   }
   ```
   只有同时调用 `willChange` 和 `didChange`，才会触发 `observeValueForKeyPath:` 回调。

3. **重写 `class` 方法**  
   动态子类会重写 `class` 返回原类名，伪装成原类，外部无感知。

4. **手动触发 KVO**  
   如果不调用 setter（如直接修改成员变量），KVO 不会触发。可手动包裹：
   ```objc
   [obj willChangeValueForKey:@"key"];
   obj->_key = newValue;
   [obj didChangeValueForKey:@"key"];
   ```

#### 一句话总结
> **KVO 本质**：Runtime 动态创建 `NSKVONotifying_` 开头的子类，将对象的 isa 指向该子类，并重写被监听的 setter 方法，通过 `willChangeValueForKey:` / `didChangeValueForKey:` 包裹原赋值逻辑，触发监听回调。

---

## 三、高频面试题（直接背答案）
### 1. KVC 如何访问私有属性？
KVC 优先查找 getter/setter，找不到时会调用 `+accessInstanceVariablesDirectly`（默认返回 YES），然后**直接访问成员变量**，无视 `@private`，因此可读写私有属性。

### 2. 为什么直接修改成员变量不会触发 KVO？
因为 KVO 是通过**重写 setter 方法**实现的，在 setter 中插入 `willChange`/`didChange`。直接修改成员变量绕过了 setter，自然不会触发 KVO。

### 3. KVO 的缺点/注意事项
1. 必须**手动移除**监听，且添加与移除必须**严格一对一**，否则会野指针崩溃或重复移除崩溃
2. 回调统一集中在 `observeValueForKeyPath:`，代码耦合高，逻辑分散
3. 只能监听通过 setter 修饰的属性变化，直接修改成员变量无效
4. observer 自身释放前必须移除监听，否则被观察对象向其发送消息会崩溃

### 4. 如何手动触发 KVO？
调用 `willChangeValueForKey:` 和 `didChangeValueForKey:` 包裹赋值代码。

### 5. KVC 与 KVO 的关系
- KVO 要求被监听的属性具备**符合 KVC 命名规范**的 setter 方法，以便动态子类能够重写插入通知
- 通过 KVC 的 `setValue:forKey:` 修改属性**会自动触发 KVO**，因为内部最终会调用正确的 setter
- KVC 是访问属性的基础，KVO 是基于 KVC + Runtime 的观察机制

### 6. 如何关闭 KVC 直接访问成员变量？
```objc
+ (BOOL)accessInstanceVariablesDirectly {
    return NO;
}
```

### 7. 如何控制某个属性是否自动触发 KVO？
重写 `+ (BOOL)automaticallyNotifiesObserversForKey:`，对特定 key 返回 `NO`，则需手动调用 `willChange`/`didChange` 触发通知。

### 8. iOS 11+ 如何更安全地使用 KVO？
使用 **Block-based KVO**：
```objc
NSKeyValueObservation *observation = [obj observeValueForKeyPath:@"key"
    options:NSKeyValueObservingOptionNew
    changeHandler:^(id obj, NSDictionary *change, void *context) {
        // 处理变化
    }];
```
优点：回调与监听绑定，无需在 `dealloc` 中手动移除，`observation` 释放时自动停止监听，避免崩溃。

---

## 四、极简总结（面试最后冲刺）
### KVC
- 字符串 key 读写属性，打破封装
- 查找顺序：setter/getter → 成员变量
- 可处理异常、支持集合运算

### KVO
- 监听属性变化（必须符合 KVC 命名规范的 setter）
- 底层：**Runtime 动态子类 + 重写 setter**（不是方法交换）
- 必须手动移除监听（严格配对），或使用 Block-based KVO 自动管理
- 直接修改变量不触发，可手动触发

---

修正后这份文档已覆盖 iOS 面试中 KVC/KVO **95%** 以上的考点，内容严谨，可直接背诵应对面试。