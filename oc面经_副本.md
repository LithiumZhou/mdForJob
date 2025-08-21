# 面经

## 聊聊property

一种语法糖

**`@property` 本质上是一个编译器指令，它会自动为你生成一个用于存储数据的私有实例变量（`ivar`），以及一套标准的、遵循你指定内存管理规则（如`strong`或`copy`）的 `getter` 和 `setter` 方法，从而让你能以“点语法”（`object.property`）安全地访问这个变量。**

## nonnull

## assign, weak, strong, copy,atomic,nonatomic

* **assign和weak:**给对象使用这个关键字，他们两个同样都会给指针赋值，不增加引用计数，但是核心区别在于，**weak指向的对象销毁后，weak会自动指向nil**，assign也不会增加引用计数，但是会导致**野指针，即对象被销毁，但是指针还指向那块区域**。并且，由于oc的msg_send机制，oc中不存在空指针异常，会先判断指针是不是为空。
* **strong和copy：** strong赋值时会直接复制指针地址，并且会引用计数加一，修改赋值和背赋值的指针里面的对象，都会互相影响。但是copy指针赋值时，相当于复制出一个新的对象在堆中，然后把被赋值的指针指向新对象，互不干扰。
* **atomic 和 nonatomic**:atomic使得setter和getter方法变成原子性（原理就是在方法上加上互斥锁），nonatomic则不是原子性的。
* **unsafe_unretained**:类似weak，不会增加引用计数，但是对象销毁不会给指针置为nil

## atomic是如何实现原子性的

### 1. 自动加锁的存取方法

当你将一个属性声明为 `atomic`（这也是默认行为），编译器会自动合成对应的getter和setter方法。在这些自动生成的方法内部，包含了一个锁，以保护对底层实例变量的访问。

*   **Setter方法**: 当一个线程调用setter方法来修改属性值时，该方法会先获取一个锁。在持有锁的期间，它会完成整个赋值操作（对于对象，这可能包括释放旧值并保留新值）。操作完成后，释放锁。 任何其他尝试访问该属性的线程都必须等待，直到该锁被释放。

*   **Getter方法**: 同样，当一个线程调用getter方法读取属性值时，它也会先获取同一个锁。在持有锁的情况下读取值，然后释放锁。 这确保了在读取期间，没有其他线程可以修改该值，从而防止读取到不完整或损坏的数据。

### 2. 使用的锁类型

Objective-C的运行时（runtime）使用了一个内部锁池来管理这些锁。具体来说，它通常采用的是一种自旋锁（Spin Lock）。

*   **自旋锁（Spin Lock）**: 当一个线程尝试获取一个已经被持有的自旋锁时，该线程不会立即进入睡眠状态，而是在一个循环中“旋转”，反复检查锁是否被释放。这种方式在锁被持有的时间非常短的情况下效率很高，因为它避免了线程上下文切换带来的开销。

`atomic`属性的setter方法大致可以被理解为以下伪代码：

```objectivec
// 伪代码示例
- (void)setValue:(id)newValue {
    // 获取与属性关联的锁
    spin_lock(&property_lock);

    if (_value != newValue) {
        [_value release]; // for MRC
        _value = [newValue retain]; // for MRC
        // 在ARC下，则是简单的赋值
    }

    // 释放锁
    spin_unlock(&property_lock);
}
```

而getter方法则类似：

```objectivec
// 伪代码示例
- (id)value {
    id result;

    // 获取与属性关联的锁
    spin_lock(&property_lock);

    result = _value; // 在MRC下，通常还会autorelease
    [result retain]; // 确保返回时对象有效
    [result autorelease];

    // 释放锁
    spin_unlock(&property_lock);

    return result;
}
```
虽然具体的实现由Objective-C运行时处理，并且可能使用`objc_setProperty`和`objc_getProperty`等底层函数，但核心思想就是通过加锁来保证访问的独占性。

### `atomic`的保证与局限

*   **保证**: `atomic`确保了单次存取操作的完整性。 例如，如果一个线程正在写入一个新值，另一个线程在读取时，不会读到只写了一半的“垃圾”数据。它要么读到旧的值，要么读到完整的新值。

*   **局限**: `atomic`本身并不能保证线程安全。它只保护了单个的getter和setter操作。 如果你的逻辑涉及多个步骤（比如“先读取，再基于读取的值进行计算，最后写入新值”），`atomic`无法保护这整个序列。在读取和写入之间，其他线程仍然可能修改这个属性的值。

**示例**:
假设有一个`atomic`的`count`属性。

```objectivec
// 这种操作不是线程安全的
self.count = self.count + 1;
```

这个操作实际上分为三步：
1.  调用getter读取`self.count`的当前值。
2.  将该值加1。
3.  调用setter将新值写回`self.count`。

`atomic`只保证了第1步和第3步各自的原子性，但无法保证在第1步和第3步之间没有其他线程修改`self.count`。因此，这可能会导致更新丢失的问题。要实现这种复合操作的线程安全，需要使用更明确的锁机制，如`@synchronized`或GCD。

## ios中有几种复制

### **两大核心复制类型：`copy` vs. `mutableCopy`**

这是由 `NSObject` 的一个非正式协议 `NSCopying` 和一个正式协议 `NSMutableCopying` 所定义的两种最基本的复制行为。

#### **1. `copy` -> 产生不可变副本**

*   **调用方法**: `[someObject copy]`
*   **需要遵守**: `NSCopying` 协议，并实现 `- (id)copyWithZone:(NSZone *)zone;` 方法。 **重要！！！！！**
*   **核心目的**: 无论原始对象是可变的还是不可变的，调用 `copy` 方法后，你**总是**会得到一个**不可变 (Immutable)** 的副本。
*   **具体行为**:
    *   **对不可变对象 (`NSString`, `NSArray`) 调用 `copy`**:
        *   这通常不会创建一个全新的对象。出于性能优化，它会执行一次**“浅拷贝”**，但实际上更像是**指针拷贝并增加引用计数**。因为原始对象和副本都是不可变的，内容完全一样，没有必要再浪费内存去创建一个一模一样的新对象。所以，返回的只是指向原始对象的另一个指针（引用计数+1）。
    *   **对可变对象 (`NSMutableString`, `NSMutableArray`) 调用 `copy`**:
        *   这**会创建一个全新的、不可变的对象** (`NSString`, `NSArray`)。
        *   新对象的内容与原始可变对象在拷贝那一刻的内容完全相同。
        *   这是一个非常常见的用法，用于获取一个可变对象的“快照”，确保这个快照在未来不会再被意外修改。

#### **2. `mutableCopy` -> 产生可变副本**

*   **调用方法**: `[someObject mutableCopy]`
*   **需要遵守**: `NSMutableCopying` 协议，并实现 `- (id)mutableCopyWithZone:(NSZone *)zone;` 方法。**重要！！！！**
*   **核心目的**: 无论原始对象是可变的还是不可变的，调用 `mutableCopy` 方法后，你**总是**会得到一个**可变 (Mutable)** 的副本。
*   **具体行为**:
    *   **对不可变对象 (`NSString`, `NSArray`) 调用 `mutableCopy`**:
        *   **会创建一个全新的、可变的对象** (`NSMutableString`, `NSMutableArray`)。
        *   新对象的内容与原始不可变对象的内容相同。
    *   **对可变对象 (`NSMutableString`, `NSMutableArray`) 调用 `mutableCopy`**:
        *   **也会创建一个全新的、可变的对象** (`NSMutableString`, `NSMutableArray`)。
        *   新对象的内容与原始可变对象的内容相同。

---

### **两大复制深度：浅拷贝 vs. 深拷贝**

`copy` 和 `mutableCopy` 的默认行为，特别是对于**容器类**（如 `NSArray`, `NSDictionary`），是**浅拷贝**。这是一个极其重要的概念。

#### **3. 浅拷贝 (Shallow Copy)**

*   **定义**: **只复制容器对象本身，而不复制容器内部所引用的元素对象。**
*   **比喻**: 你复印了一份通讯录。你得到了一本**新的通讯录本子**（新的容器对象），但这本新通讯录上记录的所有人的**电话号码**（指向元素的指针），和旧本子上是**一模一样**的。
*   **行为**:
    *   当你执行 `NSMutableArray *newArray = [oldArray mutableCopy];` 时：
        *   `newArray` 是一个全新的、独立的 `NSMutableArray` 实例。你可以向 `newArray` 中添加或删除元素，而**不会影响** `oldArray`。
        *   但是，`newArray` 和 `oldArray` 内部的元素，依然是指向**同一批原始对象**的指针。
*   **后果**: 如果你从 `newArray` 中取出一个 `Person` 对象，并修改了它的属性（`person.name = @"New Name";`），那么当你从 `oldArray` 中取出“同一个” `Person` 对象时，会发现它的 `name` 也被改变了。因为它们本来就是同一个对象。

#### **4. 深拷贝 (Deep Copy)**

* **定义**: **不仅复制容器对象本身，还会递归地、逐一地复制容器内部所引用的所有元素对象。**

* **比喻**: 你不仅复印了通讯录，还为通讯录上的每一个人都**克隆**了一个一模一样的“克隆人”。你的新通讯录上，记录的都是这些“克隆人”的电话号码。

* **行为**:

  *   执行深拷贝后，你会得到一个全新的容器，并且容器里的所有元素也都是全新的、独立的副本。
  *   修改新容器中任何一个元素的状态，**绝对不会**影响到原始容器中的任何元素。

* **如何实现**:

  * Objective-C **没有一个通用的、系统级的 `deepCopy` 方法**。因为框架不知道你的自定义对象应该如何进行深拷贝。

  * 你必须**自己实现**深拷贝逻辑。

  * **对于容器类**: Foundation 提供了一个便利的初始化方法来实现深拷贝：

    ```objc
    // `copyItems:YES` 会遍历 `originalArray`，并对其中的每一个元素调用 `copy` 方法。
    // 这要求数组中的所有元素都必须遵守 `NSCopying` 协议。
    NSArray *deepCopiedArray = [[NSArray alloc] initWithArray:originalArray copyItems:YES];
    ```

    **注意**: 这依然是一个“一层”的深拷贝。如果数组元素 `Person` 内部还有一个 `Car` 对象，这个 `Car` 对象是不会被拷贝的，除非你在 `Person` 的 `copyWithZone:` 方法里自己去实现对 `Car` 的拷贝。

  * **对于自定义对象**: 你需要在你自己的 `copyWithZone:` 实现中，对需要深拷贝的属性，也调用 `copy` 或 `mutableCopy`。

    ```objc
    - (id)copyWithZone:(NSZone *)zone {
        MyObject *copy = [[MyObject allocWithZone:zone] init];
        // 对 name 属性进行 copy，确保它是一个新的字符串实例
        copy.name = [self.name copy]; 
        // 对 array 属性进行深拷贝
        copy.myArray = [[NSMutableArray alloc] initWithArray:self.myArray copyItems:YES];
        return copy;
    }
    ```



## 用copy修饰符还需要做什么，如何实现深拷贝和浅拷贝

* 假如是官方类，该类需要遵循`NSCopying`协议

* 假如自定义类，则需要重写，以下是**深拷贝**，完全copy了新的一个name属性

```objc
// Person.m
#import "Person.h"

@implementation Person

// 重写这个方法，告诉系统具体怎么做拷贝操作
- (id)copyWithZone:(NSZone *)zone {
    // 1. 创建一个新的Person对象实例
    Person *personCopy = [[[self class] allocWithZone:zone] init];
    
    // 2. 将当前对象的属性值，赋值给新创建的副本对象
    //    这里定义了深拷贝还是浅拷贝。
    //    对于对象类型，通常也使用copy来防止共享实例。
    personCopy.name = [self.name copy];
    //    对于基本数据类型，直接赋值即可。
    personCopy.age = self.age;
    
    // 3. 返回创建好的副本
    return personCopy;
}

@end
```

* 以下是浅拷贝

```objc

@implementation Person

// 3. 实现 copyWithZone: 方法
- (id)copyWithZone:(NSZone *)zone {
    // a. 创建一个新的 Person 对象实例
    //    使用 [[[self class] allocWithZone:zone] init] 是标准写法，能确保子类也能正确拷贝
    Person *personCopy = [[[self class] allocWith-Zone:zone] init];
    
    // b. 将当前对象的 name 属性值赋给副本的 name 属性
    //    因为 personCopy.name 的 setter 是 copy 属性，这里会自动执行 [self.name copy]
    //    对于 NSString 来说，这就是我们想要的浅拷贝行为
    personCopy.name = self.name; 
    
    // c. 返回创建好的副本对象
    return personCopy;
}

@end
```

## ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些？

* 对象类型`@property (strong, atomic, readwrite) NSString *name;`

* 非对象类型`@property (assign, atomic, readwrite) NSInteger score`

## `ARC` 在运行时做了哪些工作？

主要就是维护weak指针。

## autoreleasepool

1. **主线程的事件循环（系统层面，我们无感知）**：iOS应用的主线程RunLoop在每次事件循环（如处理一次点击、一次屏幕刷新）的开始和结束，系统都会自动地为我们创建和销毁一个自动释放池。这就是为什么我们在viewDidLoad或者按钮点击方法里创建的临时对象能够被自动回收的原因。
2. **循环中产生大量临时对象（开发者必须手动干预）**：这是我们需要手动使用它的最经典场景。比如在一个for循环中，如果需要反复地读取文件、处理图片、解析数据，会产生大量临时对象。
   - **如果不加池子**：这些临时对象会全部堆积在当前RunLoop的那个“大池子”里，直到事件循环结束才被释放，这会导致**内存峰值瞬间暴涨**，甚至可能因为内存不足而导致程序崩溃。
   - **正确做法**：在循环内部用@autoreleasepool把代码包起来。这样，每次循环结束时，池子就会被销毁，本次循环产生的临时对象被**立即回收**，使得内存使用非常平稳，大大优化了性能。
3. **自己创建的子线程**：如果我们自己通过NSThread等方式创建并启动一个子线程，这个线程默认是没有自动释放池的。我们必须在该线程的执行方法内部，手动用@autoreleasepool包裹代码，否则所有在这个线程中产生的autoreleased对象都将无法被释放，造成严重的内存泄漏。

好的，这是一个非常深入的问题，能问到这里，说明您已经对OC的内存管理有了相当的理解。自动释放池的底层原理是Objective-C运行时（Runtime）一个非常精巧的设计。

我们可以从**数据结构**、**关键函数**和**与线程的关系**三个层面来彻底解析它。

---

### 一、核心数据结构：`AutoreleasePoolPage`

自动释放池并不是一个简单的数组或栈，它的底层实现依赖于一个C++类——`AutoreleasePoolPage`。

你可以把整个自动释放池系统想象成一个**以线程为单位的双向链表**，链表中的每一个节点就是一个 `AutoreleasePoolPage` 对象。

**`AutoreleasePoolPage` 的特点：**

1.  **固定大小**：每个 `AutoreleasePoolPage` 对象都是一个固定大小（通常是4096字节，即一页内存）的内存区域。
2.  **内部结构**：它内部除了存储一些管理信息（如指向父、子页面的指针）外，主要是一个**栈结构**，用来存放被`autorelease`的对象的**指针**。它有一个`next`指针，指向栈顶，每次有新对象加入，`next`就向上移动。
3.  **双向链表**：当一个`AutoreleasePoolPage`满了之后，它会自动创建一个新的页面（`child`），并与自己连接起来，形成一个双向链表。这样就解决了单个池子容量有限的问题。

**简单来说：** 自动释放池在底层是由一连串的 `AutoreleasePoolPage` 页面组成的，这些页面构成了一个链表，而每个页面内部又是一个后进先出（LIFO）的栈，用来存放对象指针。

---

### 二、关键函数：`push` 和 `pop`

我们写的 `@autoreleasepool` 只是一个语法糖。在编译时，编译器会把它转换成对两个核心函数的调用：`objc_autoreleasePoolPush()` 和 `objc_autoreleasePoolPop()`。

```objectivec
// 你写的代码：
@autoreleasepool {
    // ... do something ...
}

// 编译器转换后的伪代码：
void *pool = objc_autoreleasePoolPush();
// ... do something ...
objc_autoreleasePoolPop(pool);
```

#### 1. `objc_autoreleasePoolPush()`

当这个函数被调用时，它并**不是**真的创建了一个全新的 `AutoreleasePoolPage`。它做了一个非常巧妙的操作：

它向当前线程的 `AutoreleasePoolPage` 的栈顶压入一个**“哨兵”对象（Sentinel Object）**。

*   **哨兵**：你可以把它理解为一个特殊的`nil`或者一个边界标记。它不代表任何真实的对象，它的唯一作用就是**标记一个自动释放池的边界**。
*   `objc_autoreleasePoolPush()` 会返回这个哨兵的地址，也就是上面伪代码中的`pool`变量。

#### 2. `[obj autorelease]` (ARC自动插入)

当一个对象需要被放入池中时（在ARC下由编译器自动处理），会调用 `objc_autorelease(obj)` 函数。这个函数会找到当前线程正在使用的 `AutoreleasePoolPage`，然后把这个对象的指针压入到页面的栈顶。

#### 3. `objc_autoreleasePoolPop(pool)`

当代码执行到 `}` 时，`objc_autoreleasePoolPop()` 函数被调用，并把之前 `push` 操作返回的那个“哨兵”地址作为参数传进去。

这个函数的工作流程是：

1.  从当前 `AutoreleasePoolPage` 的栈顶开始。
2.  依次取出栈中的每一个对象指针。
3.  向取出的对象发送一条 `release` 消息。
4.  一直重复这个过程，直到遇到那个作为边界的**哨兵对象**为止。

这就完美地解释了池子的嵌套关系：内层池子`pop`的时候，只会释放到内层池子的哨兵那里，绝不会影响到外层池子的对象。

---

### 三、与线程的关系：线程局部存储 (Thread-Local Storage)

自动释放池的一个极其重要的特性是：**池是与线程严格绑定的**。

*   每个线程都维护着自己独立的**自动释放池栈**（`AutoreleasePoolPage` 的链表）。
*   一个线程的操作完全不会影响到另一个线程的自动释放池。

这是通过**线程局部存储（TLS）**技术实现的。运行时系统会在一个专门的数据结构中，以当前线程ID为key，存储该线程的`AutoreleasePoolPage`链表信息。当调用`push`或`pop`时，系统会先获取当前线程ID，然后找到专属于这个线程的池子进行操作。

这就是为什么我们常说：“主线程有自己的RunLoop来管理池子，而子线程需要你手动创建池子。”

这是一个非常关键且精准的问题！您的理解是完全正确的。

**出栈操作本身，并不等同于“释放对象内存”。**

我们可以把“出栈”这个操作，更精确地理解为：**自动释放池放弃了对池内对象的“延迟释放”的责任，并将这个决定权交还给了对象的引用计数系统。**

让我们来深入解析这个过程。

## autoReleasePool在什么情况下使用

* **在循环中创建大量临时对象** 

* #### 在主线程之外，创建了一个需要长期运行的子线程

  - **问题描述**：
    - 主线程有系统自动管理的 Autorelease Pool。
    - 但是，如果你自己创建了一个子线程（比如使用 NSThread），并且这个线程需要**长期存在**（而不是执行完一个 Block 就退出），那么这个子线程**默认是没有** Autorelease Pool 的。
    - 如果你在这个子线程里，创建了任何 autorelease 的对象，它们将**无处可去**，永远不会被添加到任何 Pool 中，从而**导致内存泄漏**。
  - **解决方案：为子线程手动创建自己的“顶层” Pool**

  ```objc
  // 【好例子】
  + (void)myThreadMain {
      // 为整个线程的生命周期，创建一个顶层的 Autorelease Pool
      @autoreleasepool {
          
          NSLog(@"子线程开始运行...");
          
          // 你可以在这里设置一个 RunLoop，让线程持续运行
          NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
          [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
          
          // 启动 RunLoop
          while (![NSThread currentThread].isCancelled) {
              // 在 RunLoop 的每一次循环内部，也创建一个小 Pool
              // 这和主线程的机制是一样的
              @autoreleasepool {
                  [runLoop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
              }
          }
          
          NSLog(@"子线程即将退出...");
      } // 顶层 Pool 在线程退出时被销毁
  }
  ```

## 什么对象会加入autoReleasePool

用类方法**的便利构造器**创造的对象都会加入自动释放池，因为默认要遵守 谁创造谁释放的原则，所以这些对象离开方法之后要延迟释放，只能用pool

## autoReleasePool在什么情况下释放

## CGfloat 和 float说说 有什么区别

* 一种根据cpu架构自适应的float，CGfloat在32位cpu下就是float，在64位cpu下会变成double
* 具体底层实现 是通过条件编译

## category能不能添加property

这是一个非常经典且重要的 Objective-C 面试题，答案是“**可以，但又不完全可以**”，需要分两个层面来精确回答。

**简短的答案是：**

> 你**可以**在 `Category` 的 `@interface` 中声明 `@property`，但这只会为你生成 **getter 和 setter 方法的声明**。它**不会**自动为你生成实例变量（ivar），也**不会**自动为你合成 getter/setter 方法的实现。你需要通过**关联对象（Associated Objects）**来手动实现这些方法的背后存储。

---

### **详细的、分层次的回答**

#### **第一步：回答“可不可以”**

“是的，我们**可以在 `Category` 中声明 `@property`**。从语法上讲，这是完全允许的。”

```objc
// UIView+MyAdditions.h
#import <UIKit/UIKit.h>

@interface UIView (MyAdditions)

// 语法上完全正确
@property (nonatomic, strong) NSString *myCustomID;

@end
```
---

### **第二步：解释“为什么行为不同”，即 `@dynamic` 的隐式行为**

“在 `Category` 中声明一个 `@property`，编译器只会为我们做一件事：**自动生成 `setMyCustomID:` 和 `myCustomID` 这两个方法的声明**。”

“它**不会**做以下两件在常规类中会自动做的事：”
1.  **不会自动生成实例变量**: “它不会为我们生成一个 `_myCustomID` 这样的下划线实例变量。这是因为 `Category` 的底层结构 `category_t` 中，根本就没有空间来存储实例变量的信息。一个类的内存布局在编译时就已经确定了，`Category` 是在运行时才被附加到类上的，它不能在运行时去改变一个类已经固化的内存大小和布局。”
2.  **不会自动合成存取方法**: “由于没有实例变量可以去存取，编译器也**不会**为我们自动合成 `setMyCustomID:` 和 `myCustomID` 这两个方法的**实现**。”

“实际上，在 `Category` 中声明 `@property`，就等价于你在这个 `Category` 的 `@implementation` 中隐式地写下了 **`@dynamic myCustomID;`**。它是在告诉编译器：‘这两个方法的实现，我会在运行时通过其他方式提供，你不用管了。’”



## catgory和extension

### 一、Category (分类)

**核心定义**：Category 是一种**在不修改原始类源代码的情况下，为这个类添加新方法**的机制。

你可以为任何你知道其接口的类（无论是系统类如 `NSString`，还是你自己写的类）添加分类。

**主要用途和特点：**

1.  **为现有类添加新功能**：这是最核心的用途。比如，我们可以为 `NSString` 增加一个计算MD5值的方法，或者为 `UIColor` 增加一个通过十六进制字符串创建颜色的方法。
2.  **模块化代码**：如果一个类的实现文件（`.m`）过于庞大，可以根据功能逻辑创建不同的 Category，将方法的实现分散到不同的文件中，使代码结构更清晰。
    *   `MyViewController+Networking.m`
    *   `MyViewController+Animations.m`
3.  **【重要限制】无法添加新的实例变量**：Category **不能**直接为类添加实例变量。这是它与类扩展最根本的区别。虽然可以通过“关联对象”（Associated Objects）这种运行时技巧间接实现，但这并非其设计初衷。

**语法：**

**.h 文件 (接口声明)**
```objectivec
#import "OriginalClass.h"

// 接口文件名通常是 OriginalClass+CategoryName.h
@interface OriginalClass (CategoryName)

// 在这里声明你想要添加的新方法
- (void)aNewMethod;

@end
```

**.m 文件 (实现)**
```objectivec
#import "OriginalClass+CategoryName.h"

// 实现文件名通常是 OriginalClass+CategoryName.m
@implementation OriginalClass (CategoryName)

- (void)aNewMethod {
    // 方法的具体实现
    NSLog(@"This is a new method from a category!");
}

@end
```

---

### 二、Class Extension (类扩展)

**核心定义**：Class Extension 通常被称为**“匿名分类”（Anonymous Category）**。它的主要作用是**为一个类添加私有的属性和私有的方法声明**，这些内容只在类的内部可见和使用。

类扩展必须在**主类的实现文件（`.m`）中**声明，它声明的所有内容，也必须在同一个 `.m` 文件中实现。

**主要用途和特点：**

1.  **声明私有属性 (Private Properties)**：这是最核心的用途。我们经常在 `.h` 文件中将属性声明为 `readonly`（只读），然后在 `.m` 文件的类扩展中将其重新声明为 `readwrite`（可读写），这样就实现了“对外只读，对内可写”的封装。

2.  **声明私有方法 (Private Methods)**：在类扩展中声明的方法，相当于这个类的私有方法。它们无需对外暴露接口，但可以在 `.m` 文件内部被调用，并能享受到编译器的类型检查。

3.  **【重要】可以添加新的实例变量**：因为类扩展是主类实现的一部分，在编译时，它里面的所有信息都会被整合到主类中。因此，它**可以**像主接口一样声明 `@property`，编译器会自动为其生成对应的实例变量。

4.  **与主类紧密耦合**：类扩展中声明的一切，都必须在主类的 `@implementation` 块中实现。它不是一个独立的模块，而是主类实现的一个“补充部分”。

**语法：**

**通常只存在于 .m 文件中**
```objectivec
#import "MyClass.h"

// MyClass.m

// 类扩展，写在 @implementation 之前
@interface MyClass () // 注意，括号里是空的

// 在这里声明私有属性和私有方法
@property (nonatomic, strong) NSString *privateData;
- (void)aPrivateHelperMethod;

@end


@implementation MyClass

// 在这里实现所有的方法，包括私有方法
- (void)aPrivateHelperMethod {
    // ...
}

// ... 其他方法的实现 ...

@end
```



**一句话总结：**

*   想给一个**没有源码**的类（如`NSString`）加点新功能，用 **Category**。
*   想给**你自己的类**加点**私有**的变量和方法，不让外面知道，用 **Class Extension**。

## 内存管理，arc，mrc和引用计数和autoreleasepool

## runloop的本质，observer，timer，source

好的，我们完全抛开比喻，只用最直接、最基础的计算机语言来解释 `RunLoop`。

---

### 第一部分：RunLoop 的核心目的

**一个程序（或者一个线程），如果没有一个循环，它的代码就会从头执行到尾，然后终止。**

例如，一个简单的 C 程序：
```c
int main() {
    // 任务1
    // 任务2
    // 任务3
    return 0; // 执行完毕，程序/进程退出
}
```
但是，一个图形界面的应用程序，比如 iOS App，它启动后不能马上退出。它必须**持续地运行**，等待用户的输入（比如触摸屏幕），或者等待网络数据的到达。

为了让一个线程能够**持续存在并等待事件**，最简单的方式就是使用一个无限循环：
```c
while (true) {
    // 检查有没有新任务？
}
```
这种方式的问题是，即使没有任何新任务，这个循环也会一直空转，导致 CPU 占用率 100%，这会大量消耗电量和计算资源。

**RunLoop 的核心目的，就是为了解决这个问题。** 它提供了一种**高效的事件等待和处理机制**：
*   当**有事件**需要处理时，线程就处于**运行**状态。
*   当**没有事件**需要处理时，线程就进入**休眠**状态，不占用 CPU 资源。

**所以，RunLoop 是一个管理线程生命周期和事件处理的对象。**

---

### 第二部分：RunLoop 的工作机制

`RunLoop` 的工作机制，就是一个**循环**。我们可以把它的伪代码想象成这样：

```c
void RunLoop::run() {
    do {
        // 1. 检查有没有“输入源 (Input Sources)”或“定时器 (Timers)”事件需要立即处理。
        //    “输入源”就是事件的来源，比如用户的触摸、网络数据的到达等。
        
        // 2. 如果有，就去处理这些事件。
        //    比如，调用你为按钮点击写的回调函数。
        
        // 3. 如果处理完了，或者一开始就没有任何事件，就准备让线程休眠。
        
        // 4. 调用操作系统内核的函数，让当前线程进入休眠状态。
        //    这个休眠是真正的休眠，CPU 会把时间片分配给其他需要运行的线程。
        
        // 5. 线程会一直在这里“暂停”，直到以下情况之一发生：
        //    a. 一个新的“输入源”事件到达。
        //    b. 一个“定时器”的时间到了。
        //    c. 你从其他线程手动唤醒了这个 RunLoop。
        
        // 6. 当线程被唤醒后，它会从休眠中返回，跳回到循环的开始（第1步），去处理那个唤醒它的事件。
        
    } while (shouldKeepRunning);
}
```

**简单来说，RunLoop 就是一个循环，它不断地：检查事件 -> 处理事件 -> 如果没事干就休眠 -> 被唤醒 -> 检查事件...**

---

### 第三部分：那些你不懂的名词解释

`RunLoop` 的世界里，有几个关键的“名词”，它们是构成整个机制的零件。

#### **1. `Input Sources` (输入源)**

* **是什么？** 就是**产生事件的东西**，是 `RunLoop` 需要监听和处理的对象。

  在 **Apple RunLoop** 中，**Source** 代表输入源（Input Source），用于处理事件。它分为两类：

  - **Source0**：非基于端口的输入源（手动唤醒）
  - **Source1**：基于端口的输入源（内核可唤醒）

  ------

  ## ✅ **Source0**

  ### **特点**

  - **非端口驱动**，不依赖内核，完全在用户态。
  - 不会主动唤醒 RunLoop，**必须手动唤醒**（`CFRunLoopWakeUp()`）。
  - 用于处理**自定义事件**，比如：
    - `performSelector:onThread:`
    - 手动触发的任务（如 GCD 的 `dispatch_async` 到主线程）

  ------

  ## ✅ **Source1**

  ### **特点**

  - **基于端口（Port-based）**，底层是 **Mach port**。
  - 内核事件可以直接唤醒 RunLoop（系统自动）。
  - 用于处理**系统事件、跨线程通信、进程通信**。

  ### **典型场景**

  - **系统级事件**
    - 触摸、键盘、屏幕刷新，最终通过 Mach port 发送到 RunLoop。
  - **CFMachPort**
    - 用于和内核或其他进程通信。
  - **GCD 主队列任务**
    - 底层通过 Mach port 将 block 投递到主线程。

  ------

  ## ✅ **二者区别总结**

  | 特性         | Source0                       | Source1                            |
  | ------------ | ----------------------------- | ---------------------------------- |
  | **基于端口** | 否                            | 是（Mach port）                    |
  | **唤醒机制** | 手动唤醒（`CFRunLoopWakeUp`） | 内核事件自动唤醒                   |
  | **事件来源** | 自定义任务、`performSelector` | 系统事件、GCD、IPC                 |
  | **常见用途** | 线程间通信、自定义输入源      | 系统输入源（触摸、定时器、信号等） |

  ------

  ✅ **RunLoop 处理 Source 的顺序**

  1. 先处理 **Source0**（非端口事件）
  2. 再处理 **Source1**（端口事件）

#### **2. `Timers` (定时器)**

*   **是什么？** 就是你熟悉的 `NSTimer`。
*   **作用**：它允许你在未来的某个特定时间点，或者周期性地执行一段代码。
*   **与 RunLoop 的关系**：`NSTimer` **必须**被添加到一个 `RunLoop` 中才能生效。当定时器的时间到了，它会**唤醒**休眠的 `RunLoop`，然后 `RunLoop` 会执行你为定时器指定的回调方法。

#### **3. `Observers` (观察者)**

*   **是什么？** 它可以让你**监视 `RunLoop` 自身的状态变化**。
*   **作用**：你可以创建观察者，来监听 `RunLoop` 的某几个关键“时间点”，并在这些时间点执行你自己的代码。
*   **关键时间点包括**：
    *   `RunLoop` 即将开始处理事件。
    *   `RunLoop` 即将开始处理 `Timers`。
    *   `RunLoop` 即将开始处理 `Sources`。
    *   `RunLoop` 即将进入休眠状态。
    *   `RunLoop` 刚刚从休眠中被唤醒。
*   **实际应用**：iOS 系统内部就用观察者做了很多事。比如，**自动释放池 (`@autoreleasepool`)** 的创建和销毁，就是通过监听 `RunLoop` 的“即将休眠”和“循环开始”等状态来实现的。UI 的刷新也是在 RunLoop 的某个固定阶段统一进行的。

#### **4. `RunLoop Modes` (运行模式)**

*   **是什么？** 一个 `RunLoop Mode` 就是一个**事件处理方案的“集合”**。一个 `RunLoop` 在运行时，**必须**指定一个 Mode。
*   **作用**：它决定了在**当前这个时刻**，`RunLoop` **只关心**哪些 `Input Sources` 和 `Timers`。
*   **举例说明**：
    *   **`NSDefaultRunLoopMode` (默认模式)**：当你的 App 处于静止状态时，主线程的 `RunLoop` 就运行在这个模式下。
    *   **`UITrackingRunLoopMode` (滚动模式)**：当你的手指在 `UITableView` 上快速滑动时，为了保证滑动的流畅，系统会把主线程的 `RunLoop` **临时切换到**这个模式。
    *   **为什么切换？** 因为在 `UITrackingRunLoopMode` 这个“集合”里，可能**只注册了**与滑动和UI刷新相关的事件源，而**没有注册**其他的事件源（比如你的 `NSTimer`）。
*   **解决了什么问题？**
    *   这就解释了为什么当你在 `UITableView` 上滑动时，一个默认的 `NSTimer` 会停止触发。因为 `RunLoop` 切换到了一个“**不关心**”你那个 `Timer` 的模式。
    *   **解决方案**：`NSRunLoopCommonModes`。这是一个特殊的“组合模式”，它包含了 `NSDefaultRunLoopMode` 和 `UITrackingRunLoopMode`。如果你把你的 `Timer` 注册到 `NSRunLoopCommonModes` 下，那么无论 `RunLoop` 处于默认状态还是滚动状态，你的 `Timer` 都会被处理。

## runloop的几种模式

---

### 1. 什么是 RunLoop Mode？

一个 `RunLoop Mode` 是一个**`Input Sources` (输入源)** 和 **`Timers` (定时器)** 的集合。

可以把它想象成一个**“工作场景”**或**“任务清单”**。

*   一个 `RunLoop` 对象，可以包含多个不同的 `Mode`（多个不同的任务清单）。
*   但是，在**任何一个特定的时间点**，`RunLoop` 只能选择**其中一个 `Mode`** 来运行。
*   当 `RunLoop` 在某个 `Mode` 下运行时，它就只会处理和监听那些被**注册到这个 `Mode` 下**的 `Sources` 和 `Timers`，而会**完全忽略**其他 `Mode` 下的所有事件源。

**比喻**：
你是一个多才多艺的员工，你有两份工作职责（两个 `Mode`）：
*   **“日常工作”模式 (`Default Mode`)**：你的任务清单上写着“回复邮件”、“写代码”、“开会”。
*   **“紧急救火”模式 (`Tracking Mode`)**：你的任务清单上只写着一件事——“处理线上Bug”。

当你在“日常工作”模式下时，你只会处理邮件、代码和会议。即使有线上Bug的警报响了（另一个`Mode`下的事件），你也会暂时忽略。只有当你的老板命令你切换到“紧急救火”模式时，你才会放下手头所有的日常工作，专心致志地去处理Bug。

---

### 2. iOS 中常见的几种 RunLoop Mode

Cocoa Touch 框架为我们预定义了几种常见的 `Mode`：

*   **`NSDefaultRunLoopMode` (或 `kCFRunLoopDefaultMode`)**
    *   **含义**：**默认模式**。
    *   **使用场景**：这是 App 的**主线程**在绝大多数**空闲时间**所处的 `Mode`。几乎所有非 UI 追踪的事件，比如 `NSTimer` 的默认触发、网络请求的回调等，都在这个模式下处理。
*   **`UITrackingRunLoopMode`**
    *   **含义**：**界面追踪模式**。
    *   **使用场景**：当用户的**手指正在与 `UIScrollView`（包括其子类 `UITableView`, `UICollectionView`）进行交互**时，比如**拖动、滑动**，主线程的 `RunLoop` 就会**自动切换**到这个 `Mode`。
    *   **目的**：为了保证**滑动的极致流畅**。在这个模式下，`RunLoop` 会优先处理所有与滑动和界面刷新相关的事件，可能会**暂停**或**延迟**处理其他不紧急的事件（比如 `Default Mode` 下的 `Timer` 和网络回调），从而把所有 CPU 资源都让给界面的渲染。
*   **`NSRunLoopCommonModes` (或 `kCFRunLoopCommonModes`)**
    *   **含义**：**通用模式集合**。
    *   **这不是一个真正的模式**，而是一个**“占位符”**或**“标签”**。
    *   它代表了一个**`Mode` 的集合**，默认情况下，这个集合包含了 `NSDefaultRunLoopMode` 和 `UITrackingRunLoopMode`。
    *   **核心作用**：当你把一个事件源（比如 `NSTimer`）注册到 `NSRunLoopCommonModes` 下时，系统实际上会**自动**把它同时注册到这个集合所包含的所有 `Mode` 中（也就是同时注册到 `Default` 和 `Tracking` 模式下）。

---

### 3. RunLoop Mode 的实际应用与问题解决

理解 `Mode` 的核心，就是为了解决实际开发中遇到的问题。

#### **经典问题：`NSTimer` 在 `UITableView` 滚动时停止工作**

*   **问题现象**：你在 `viewDidLoad` 里创建了一个每秒触发一次的 `NSTimer` 用来更新一个倒计时标签。一切正常，但当你用手指按住 `UITableView` 并上下滑动时，倒计时就“冻结”了，停止更新。手指一离开，倒计时又恢复了。

*   **原因分析**：
    1.  你创建 `Timer` 时，如果用 `[NSTimer scheduledTimerWithTimeInterval:...]`，它会**默认**被添加到**当前 `RunLoop` 的当前 `Mode`** 下，也就是 `NSDefaultRunLoopMode`。
    2.  当你的手指开始滑动 `TableView` 时，主线程的 `RunLoop` 为了保证滑动流畅，**自动从 `NSDefaultRunLoopMode` 切换到了 `UITrackingRunLoopMode`**。
    3.  此时，`RunLoop` 戴上了“`Tracking` 模式”的过滤器，它只会处理注册到这个 `Mode` 下的事件。
    4.  由于你的 `Timer` 只注册在 `Default` 模式下，所以当 `RunLoop` 运行在 `Tracking` 模式时，你的 `Timer` 就被**完全忽略**了，即使它的时间到了，也不会被触发。
    5.  当你手指离开屏幕，滑动结束，`RunLoop` 又**自动切换回** `NSDefaultRunLoopMode`，于是它又开始关心你的 `Timer` 了，倒计时恢复。

*   **解决方案**：
    *   在创建 `Timer` 后，手动将它添加到 **`NSRunLoopCommonModes`** 下。
        ```objectivec
        // 1. 先创建 Timer，但不启动它
        NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 
                                                 target:self 
                                               selector:@selector(updateCountdown) 
                                               userInfo:nil 
                                                repeats:YES];
        
        // 2. 手动将 Timer 添加到 Common Modes
        //    这会把它同时添加到 Default 和 Tracking 两个 Mode 中
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        ```
    *   **效果**：现在，无论 `RunLoop` 是在 `Default` 模式下（App静止时），还是在 `Tracking` 模式下（用户滑动时），它都会处理这个 `Timer` 的事件，倒计时就能持续不断地更新了。

## runloop和线程的关系

好的，我们来详细、清晰地阐述 `RunLoop` 和**线程 (Thread)** 之间的关系。

**一句话概括：`RunLoop` 并不等于线程，但它与线程是一种“一对一”的、紧密的“伴生”关系。`RunLoop` 是用来管理线程的事件循环，让线程能够“活”下去并响应事件的工具。**

我们可以从以下几个关键点来理解它们的具体关系：

---

### 1. 关系的核心：一一对应

*   **一个线程，最多只能有一个 `RunLoop`**。
*   **一个 `RunLoop`，也只服务于某一个特定的线程**。

它们之间的关系是**一一对应**的，并且存储在一个全局的字典里，以 `pthread_t` (线程ID) 为 Key，`RunLoop` 对象为 Value。你不能把线程 A 的 `RunLoop` 交给线程 B 去用。

当你调用 `[NSRunLoop currentRunLoop]` 时，系统就是用当前线程的 ID，去这个全局字典里查找并返回对应的 `RunLoop`。

---

### 2. RunLoop 的创建：懒加载

*   **线程本身并不天生就有 `RunLoop`**。一个标准的 `NSThread` 被创建出来后，它的任务就是执行你指定的那个方法，执行完后，线程的生命周期就结束了，然后被销毁。
*   `RunLoop` 是**懒加载 (Lazy Loading)** 的。只有当一个线程**第一次**尝试获取它的 `RunLoop` 时（即第一次调用 `[NSRunLoop currentRunLoop]`），系统才会检查这个线程有没有对应的 `RunLoop`。
    *   **如果没有**，系统就会为这个线程**创建一个新的 `RunLoop`**，并把它和这个线程关联起来（存入那个全局字典）。
    *   **如果已经有了**，就直接返回已存在的那个。

---

### 3. 主线程与子线程的区别

这里就体现出了它们关系中最重要的一个区别：

*   **主线程 (Main Thread)**：
    *   **`RunLoop` 是自动创建并运行的**。
    *   当你的 iOS App 启动时，系统就已经为主线程创建好了 `RunLoop`，并且在 `UIApplicationMain` 函数内部，启动了一个**永不停止**的 `run` 调用。
    *   **这就是为什么你的 App 能够一直存活，并响应用户触摸、定时器、网络回调等所有事件的根本原因**。主线程的 `RunLoop` 是 App 的“心脏”。

*   **子线程 (Background Thread)**：
    *   **`RunLoop` 默认是不存在的，更不会自动运行**。
    *   如果你只是用 `dispatch_async` 往一个后台队列里派发一个任务，这个任务执行完后，它所在的线程可能就会被系统回收。这个过程通常不需要 `RunLoop`。
    *   **只有当你需要在一个子线程里，实现类似主线程那样的“持续等待事件”的功能时，你才需要手动去管理它的 `RunLoop`**。

---

### 4. 什么时候需要在子线程手动管理 RunLoop？

当你需要在子线程里做以下这些**需要“持续等待”**的事情时，你就必须手动获取并启动它的 `RunLoop`：

1.  **使用 `NSTimer`**：
    
    *   `NSTimer` 的触发，依赖于被添加到 `RunLoop` 中。如果你在一个子线程里创建了一个 `Timer`，但不启动这个线程的 `RunLoop`，那么这个 `Timer` **永远不会被触发**。
    
2.  **使用 `performSelector...afterDelay:`**：
    
    *   这个方法的延迟执行，也是通过 `RunLoop` 的 `Timer` 机制来实现的。没有 `RunLoop`，它就不会执行。
    
4.  **保持线程存活**：
    *   有时候，你可能希望创建一个**常驻线程 (Resident Thread)**，让它一直存在，随时准备接收任务，而不是每次有任务都去创建一个新线程（以节省创建线程的开销）。
    *   要实现这一点，唯一的办法就是获取这个线程的 `RunLoop`，并让它**进入一个永不退出的循环**，比如：
        ```objectivec
        @autoreleasepool {
            // 获取当前子线程的 RunLoop（如果不存在，这里会自动创建）
            NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
            
            // 【关键】添加一个 Port 作为 Source1。
            // 如果一个 RunLoop 里没有任何 Source 或 Timer，
            // 调用 run 方法会立刻退出。
            // 添加一个 Port 可以防止它立刻退出。
            [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
            
            // 【启动 RunLoop！】
            // 这行代码会让当前线程进入一个类似 while(true) 的循环，
            // 开始监听事件。当没有事件时，线程会休眠。
            [runLoop run];
            
            // (通常，这里的代码永远不会被执行到，除非 RunLoop 被明确停止)
        }
        ```

## runloop的几种状态

以下是 `RunLoop` Observer 可以观察到的八种主要状态：

1. **`kCFRunLoopEntry`：**

   - **含义：** RunLoop 即将进入循环。
   - **时机：** RunLoop 运行的第一个事件，表示 RunLoop 刚刚开始执行。

2. **`kCFRunLoopBeforeTimers`：**

   - **含义：** RunLoop 即将处理定时器事件。
   - **时机：** RunLoop 准备好处理 `CFRunLoopTimerRef` 事件之前。

3. **`kCFRunLoopBeforeSources`：**

   - **含义：** RunLoop 即将处理输入源事件。
   - **时机：** RunLoop 准备好处理 `CFRunLoopSourceRef` 事件之前。

4. **`kCFRunLoopBeforeWaiting`：**

   - **含义：** RunLoop 即将进入休眠（等待事件）。
   - **时机：** RunLoop 已经处理完所有即时事件，将要进入内核休眠状态，等待事件发生以唤醒。这是 RunLoop 最空闲的时候。

5. **`kCFRunLoopAfterWaiting`：**

   - **含义：** RunLoop 已经从休眠中被唤醒。
   - **时机：** RunLoop 接收到事件并从内核唤醒之后，准备处理事件之前。

6. **`kCFRunLoopExit`：**

   - **含义：** RunLoop 即将退出循环。
   - **时机：** RunLoop 即将停止运行（例如，当 RunLoop 被显式停止或达到其运行条件）。

   

## 怎么给子线程保活

在 iOS 或 macOS 上，如果你想**让子线程保活**，通常用的是 **RunLoop**，因为普通子线程在任务执行完后会直接退出。

------

### ✅ **为什么子线程会退出？**

- 子线程默认没有 RunLoop。
- 如果线程任务执行完，没有继续的事情要做，它就会直接结束。

------

### ✅ **保活的核心思路**

- 为子线程开启一个 **RunLoop**，让它一直处于运行状态，不退出。
- 不过 RunLoop 没有事件会自动退出，所以需要 **添加一个事件源（Source）或 Timer** 来让它保持运行。

------

### ✅ **常用实现方式**

#### **方法 1：添加一个 `Port` 保活**

```objc
NSThread *thread = [[NSThread alloc] initWithBlock:^{
    NSLog(@"子线程开始运行");
    [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run]; // 启动 RunLoop
    NSLog(@"子线程结束"); // 理论上不会走到这里，除非手动停止 RunLoop
}];
[thread start];
```

- `addPort:` 是给 RunLoop 添加一个输入源（Source1），否则 `run` 会立即退出。

------

#### **方法 2：添加一个虚拟的 Timer**

```objc
NSThread *thread = [[NSThread alloc] initWithBlock:^{
    NSLog(@"子线程开始运行");
    NSTimer *timer = [NSTimer timerWithTimeInterval:10 target:self selector:@selector(dummy) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
}];
[thread start];
```

- Timer 会定期触发事件，保证 RunLoop 活着。



------

### ✅ **注意点**

1. **RunLoop 不会无缘无故保持运行**，必须有事件源（Port、Timer、Source）。
2. **GCD** 线程池里的线程不能这么保活，因为它们是系统管理的。
3. 子线程保活后，要注意避免内存泄漏或死循环，确保退出机制。

## runtime

### 面试官：“同学，讲一下你对 Runtime 的理解吧”

你可以这样分三步来回答，由浅入深，层层递进：

#### 第一步：一句话定义【我是谁？】

**面试官，您好。我对 Runtime 的理解是，它更像是一套处于 Objective-C 底层的、用 C 语言实现的 API 库。它的核心作用，就是把 C 语言这种静态语言的很多决策，比如调用哪个函数，都从“编译时”推迟到了“运行时”来做，这就赋予了 Objective-C 极致的动态性。**

> **解析：**
> *   **是什么**：一套 C 语言 API 库。（定位准确，不是什么虚无缥缈的东西）
> *   **核心思想**：决策从**编译时**推迟到**运行时**。（一针见血，点出本质）
> *   **带来了什么**：动态性。（直接说出结果）
>
> 这一句话就给面试官一个清晰的第一印象：你抓住了重点。

---

#### 第二步：解释核心机制【我怎么工作的？】

**这种动态性的实现，主要依赖于它的“消息发送机制”。我们平时写的 `[anObject aMethod]`，在编译后其实并不会直接调用方法，而是会被转化成一个 C 函数调用，也就是 `objc_msgSend(anObject, @selector(aMethod))`。**

**`objc_msgSend` 函数在运行时会做几件事：**
1.  首先，通过对象的 `isa` 指针找到它的类。
2.  然后在类的缓存和方法列表里，根据方法名 `@selector(aMethod)` 去查找对应的函数指针（IMP）。
3.  如果找到了，就执行这个函数。
4.  如果在本类找不到，就会沿着继承链去父类里找。
5.  如果一直到根类都找不到，Runtime 也不会马上崩溃，而是会启动一套**消息转发**流程，给我们机会去补救。

**所以，调用哪个方法，是在运行时才通过这套查找机制动态确定的。**

> **解析：**
> *   **引出关键**：消息发送机制 `objc_msgSend`。
> *   **描述流程**：清晰地描述了 `isa` -> `Class` -> `Method List` -> `IMP` 这个查找过程。这展示了你对底层细节的了解。
> *   **体现深度**：提到了“消息转发”，说明你的知识体系比较完整，知道有补救措施。
>
> 这一步展示了你的技术深度，证明你不是只会背概念。

---

#### 第三步：举出实际应用【我能用来做什么？】

**正是因为 Runtime 的这套动态机制，我们才能在实际开发中实现很多强大的功能。我举几个最典型的例子：**

1.  **方法交换 (Method Swizzling)**：我们可以在运行时动态地交换两个方法的实现。最经典的应用就是做“埋点”，比如在不修改原有代码的情况下，我们可以 hook 掉所有 `UIViewController` 的 `viewWillAppear:` 方法，在里面加入我们自己的统计代码，实现对所有页面展示的全局监控。这是一种 AOP（面向切面编程）思想的体现。

2.  **动态添加属性 (Associated Objects)**：我们都知道 Category 不能直接添加实例变量，但借助 Runtime 的关联对象技术，我们就可以在运行时为一个已有的类“附加”上新的属性，极大地扩展了 Category 的能力。

3.  **字典转模型**：像 `YYModel`、`Mantle` 这些第三方库，它们之所以能自动把 JSON 字典转换成数据模型，核心原理就是利用 Runtime 遍历一个模型类的所有属性名，然后根据属性名作为 Key 去字典里取值，再动态地赋给模型的属性。

好的，我们来把“方法交换”（Method Swizzling）这个概念彻底拆解清楚。这绝对是面试中考察 Runtime 理解度的头号问题。

我会用一个非常形象的比喻，然后一步步带你完成代码实现，让你不仅“懂”，还能“会用”。

---

### 二、技术原理：操纵方法列表

我们再回顾一下 Runtime 的核心：一个类（`Class`）内部有一张“方法列表”，记录着这个类能响应的所有方法。

这个列表里的每一项（`Method`）都包含两个关键信息：
1.  **方法名 (SEL)**：方法的选择器，比如 `viewWillAppear:`。
2.  **方法实现 (IMP)**：一个函数指针，指向真正要执行的代码。

`SEL` 和 `IMP` 像钥匙和门一样配对。`[vc viewWillAppear:YES]` 就是用 `viewWillAppear:` 这把钥匙（SEL），去打开它对应的那扇门（IMP）。

**方法交换，就是把两把钥匙对应的门（IMP）给互换了。**

*   原来：`viewWillAppear:` -> 指向 -> **原始的实现代码**
*   我们新增一个方法：`my_viewWillAppear:` -> 指向 -> **我们新增的带统计功能的代码**

**交换后：**
*   `viewWillAppear:` -> 指向 -> **我们新增的带统计功能的代码**
*   `my_viewWillAppear:` -> 指向 -> **原始的实现代码**

这样，当系统再调用 `[vc viewWillAppear:YES]` 时，它实际上执行的是我们写的代码。而在我们自己的代码里，再调用一下 `[self my_viewWillAppear:YES]`，就等于调用了原始的实现，保证了原有功能不丢失。

---

### 三、实战演练：Hook `UIViewController` 的 `viewWillAppear:`

这是一个最经典、最实用的例子：我们想在每个视图控制器出现时都打印一条日志，但又不想去修改每一个 `UIViewController` 子类。

#### 步骤1：创建 Category

这是做方法交换最规范的方式。为 `UIViewController` 创建一个 Category，比如叫 `UIViewController+Logging`。

#### 步骤2：在 `+load` 方法中进行交换

`+load` 方法非常特殊，它在类被加载到内存时就会自动调用，而且只调用一次，是执行方法交换最安全的地方。

```objectivec
// UIViewController+Logging.m
#import "UIViewController+Logging.h"
#import <objc/runtime.h> // 必须引入 Runtime 头文件

@implementation UIViewController (Logging)

// +load 方法会在类加载时自动调用，且只会调用一次
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        // 拿到 UIViewController 类
        Class class = [self class];

        // 1. 获取原始方法
        SEL originalSelector = @selector(viewWillAppear:);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);

        // 2. 获取我们自己实现的方法
        SEL swizzledSelector = @selector(my_viewWillAppear:);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // 3. 交换！
        // 这一步是核心，直接交换两个方法的 IMP
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}

// 4. 我们自己的方法实现
- (void)my_viewWillAppear:(BOOL)animated {
    // 在这里，加入我们想注入的代码
    NSLog(@"页面即将出现: %@", NSStringFromClass([self class]));

    // 关键！调用自己，但实际上因为 IMP 已经交换，
    // 这里调用的是系统原始的 viewWillAppear: 实现
    [self my_viewWillAppear:animated];
}

@end
```

**代码讲解：**

1.  **`+load` 和 `dispatch_once`**：`+load`保证了交换操作在程序启动时就完成。`dispatch_once` 是一个额外的保险，确保即使在复杂的环境下，交换代码也绝对只执行一次。
2.  **获取 `Method`**：使用 `class_getInstanceMethod` 来获取类中特定 `SEL` 对应的 `Method` 对象。这个对象里包含了 IMP 的信息。
3.  **`method_exchangeImplementations`**：这是 Runtime 提供的一个最直接的交换函数。它接收两个 `Method` 对象，然后把它们的 IMP 互相替换掉。简单粗暴，非常有效。
4.  **实现自定义方法**：我们的 `my_viewWillAppear:` 方法是新的入口。
    *   第一行，执行我们自己的逻辑（打印日志）。
    *   第二行，`[self my_viewWillAppear:animated]`，**这是最容易让人困惑的地方！** 你可能会想，这不是在调用自己，导致无限循环吗？
        *   **绝对不会！** 因为在 `+load` 里，`my_viewWillAppear:` 这个 `SEL` 指向的 `IMP` 已经被换成了系统**原始**的 `viewWillAppear:` 的实现。所以这里虽然方法名叫 `my_viewWillAppear:`，但它执行的却是系统原始的功能。

现在，你项目里任何一个 `UIViewController`（或其子类）在即将显示时，控制台都会自动打印出一条日志，而你没有动过任何业务代码！

## kvo和kvc

### KVO 原理深入解析：动态子类与 isa-swizzling

KVO（Key-Value Observing），即键值观察，是 Objective-C 中一种强大的设计模式，允许一个对象监听另一个对象特定属性的变更。其实现核心依赖于 Objective-C 的动态特性，特别是 `isa-swizzling` 和动态子类创建。

#### KVO 的核心实现原理

当您为一个对象的属性首次添加观察者时，系统会在运行时执行一系列操作，从而实现 KVO：

1.  **动态创建子类**：系统会动态地创建一个被观察对象所属类的一个子类。这个新的子类名通常以 `NSKVONotifying_` 作为前缀，例如，如果您观察 `Person` 类的对象，系统会创建一个名为 `NSKVONotifying_Person` 的新类。
2.  **`isa` 指针交换（isa-swizzling）**：被观察对象的 `isa` 指针，原本指向其原始类（如 `Person`），现在会被修改为指向新创建的动态子类 (`NSKVONotifying_Person`)。这个过程被称为 `isa-swizzling`。因此，尽管开发者在代码中看到的仍然是原始类的实例，但其在运行时的实际类型已经变为动态生成的子类。
3.  **重写属性的 setter 方法**：动态创建的子类会重写被观察属性的 `setter` 方法。例如，如果您观察 `age` 属性，`setAge:` 方法就会被重写。在这个重写的方法中，系统会执行以下操作：
    *   在调用原始的 `setter` 方法之前，先调用 `willChangeValueForKey:` 方法，通知系统该属性即将发生变化。
    *   调用父类（即原始类）的 `setter` 方法，实际地改变属性值。
    *   在调用原始的 `setter` 方法之后，再调用 `didChangeValueForKey:` 方法，通知系统该属性已经发生变化。
4.  **触发观察者方法**：`didChangeValueForKey:` 方法的内部实现会负责调用观察者的 `observeValueForKeyPath:ofObject:change:context:` 方法，从而将属性变更的通知传递给观察者。

#### KVO 与 KVC 的关系

KVO 与 KVC (Key-Value Coding) 紧密相关。为了触发 KVO，必须通过 KVC 的方式来修改属性值，即使用 `setValue:forKey:` 方法或者直接调用属性的 `setter` 方法。如果直接修改实例变量（例如 `_age = 25;`），将不会触发 KVO。

#### 使用注意事项

*   **只能观察对象属性**：KVO 只能用于监听对象属性的变化，而不能监听成员变量或方法。
*   **移除观察者**：必须在观察者被销毁前，调用 `removeObserver:forKeyPath:` 方法移除观察。否则，当被观察对象继续发送通知时，程序会因为向一个不存在的对象发送消息而崩溃。

### KVC 原理

KVC（Key-Value Coding），即键值编码，是 Objective-C 提供的一种机制，允许开发者通过字符串（键）来间接访问对象的属性，而无需调用明确的存取方法（getter/setter）。这种动态的访问能力是构建在 Objective-C 的运行时特性之上的。

KVC 的核心在于其遵循一套明确的查找规则来定位并操作对象的属性。这套规则主要体现在 `valueForKey:` (获取值) 和 `setValue:forKey:` (设置值) 这两个核心方法中。

#### `setValue:forKey:` 的工作原理

当你调用一个对象的 `setValue:forKey:` 方法时，KVC 会按照以下顺序进行查找并尝试设置属性值：

1.  **查找存取方法 (Setter)**：首先，KVC 会在目标对象中查找一个名为 `set<Key>:` 的方法（其中 `<Key>` 是键名，首字母大写）。如果找到了这个方法，KVC 会调用它并将传入的值作为参数，完成设置。例如，对于 `[person setValue:@30 forKey:@"age"]`，系统会查找 `setAge:` 方法。

2.  **查找下划线前缀的存取方法**：如果第一步没有找到 `set<Key>:` 方法，KVC 会继续查找一个名为 `_set<Key>:` 的方法。如果找到，便会调用该方法。

3.  **检查 `accessInstanceVariablesDirectly`**：如果以上两种 setter 方法都没有找到，KVC 会调用类的 `accessInstanceVariablesDirectly` 类方法。这是一个可以被重写的类方法，用于控制 KVC 是否应该直接访问实例变量。如果该方法返回 `NO`，则 KVC 的查找过程会停止，并抛出一个 `NSUnknownKeyException` 异常。

4.  **直接访问实例变量**：如果 `accessInstanceVariablesDirectly` 返回 `YES` (这是默认行为)，KVC 会按照以下顺序查找与键名匹配的实例变量：
    *   `_<key>` (例如 `_age`)
    *   `_is<Key>` (例如 `_isRetired`)
    *   `<key>` (例如 `age`)
    *   `is<Key>` (例如 `isRetired`)
    一旦找到匹配的实例变量，KVC 就会直接为其赋值。

5.  **抛出异常**：如果经过以上所有步骤仍然没有找到任何可以设置的方式，系统会调用 `setValue:forUndefinedKey:` 方法。该方法的默认实现会抛出 `NSUnknownKeyException` 异常，但你可以重写这个方法来自定义错误处理逻辑。

#### `valueForKey:` 的工作原理

当你调用 `valueForKey:` 来获取属性值时，其查找顺序与设置值类似，但专注于获取：

1.  **查找取值方法 (Getter)**：KVC 会按照以下顺序查找取值方法：
    *   `get<Key>`
    *   `<key>`
    *   `is<Key>`
    只要找到其中任何一个，KVC 就会调用该方法并返回结果。
2.  **检查 `accessInstanceVariablesDirectly`**：同样地，如果前面几步都没有成功，KVC 会检查 `accessInstanceVariablesDirectly` 的返回值。如果为 `NO`，则查找停止并抛出异常。
3.  **直接访问实例变量**：如果 `accessInstanceVariablesDirectly` 返回 `YES`，KVC 会按照 `_<key>`、`_is<Key>`、`<key>`、`is<Key>` 的顺序查找实例变量，并直接返回其值。
5.  **抛出异常**：如果所有查找都失败了，系统会调用 `valueForUndefinedKey:` 方法，其默认实现同样是抛出 `NSUnknownKeyException` 异常。

#### KVC 的应用场景

*   **动态绑定**：允许你在运行时根据字符串来决定访问哪个属性，这在编写通用框架或动态界面时非常有用。
*   **简化代码**：可以用来操作集合，例如 `[transactions valueForKeyPath:@"payee.name"]` 可以获取一个数组中所有 `transaction` 对象的 `payee` 对象的 `name` 属性，形成一个新的数组。
*   **KVO 的基础**：KVO 严重依赖 KVC。当你通过 KVC 兼容的方式（即调用 setter 或 `setValue:forKey:`）修改属性值时，才能触发 KVO 通知。

## instance对象、类对象、元对象、isa指针

* instance对象中实际只存储属性，不存储方法，所以调用方法实际上是通过isa指针找到class对象调用方法，方法都是存在class对象的。
* instance对象的isa指针指向class对象，class对象的isa指针指向meta-class
* instance对象： 成员变量的具体值；class对象： 对象方法、属性、成员变量描述信息、协议信息；meta-class对象： 类方法





## oc内存区域

好的，这是一个关于 Objective-C 程序运行基础的经典问题。一个 Objective-C 程序在运行时，其内存空间主要被划分为五个核心区域。了解这些区域的用途和特性，对于理解变量生命周期、内存管理（ARC/MRC）以及程序性能至关重要。

这五个内存区域通常被称为“五大区”：

1.  **栈区 (Stack)**
2.  **堆区 (Heap)**
3.  **全局/静态存储区 (Static/Global Storage Area)**
4.  **常量存储区 (Constant Storage Area)**
5.  **代码区 (Code/Text Section)**

下面我们来详细解析每一个区域。

---

### 1. 栈区 (Stack)

栈区就像一个自动管理的“临时工作台”，遵循“后进先出”（LIFO, Last-In, First-Out）的原则。

*   **存放内容**：
    *   **局部变量**：在函数或方法内部定义的非静态变量，例如 `int a;`、`NSString *str;`（这里存放的是 `str` 这个指针本身，而不是 `NSString` 对象实体）。
    *   **函数/方法的参数**：调用一个方法时，传递给它的参数值或指针。
    *   **函数/方法的返回地址**：当方法执行完毕后，程序需要知道从哪里继续执行。
*   **管理方式**：
    *   **全自动管理**。 当一个方法被调用时，系统会自动在栈顶为它分配一块内存，称为“栈帧”（Stack Frame），用于存放上述内容。当该方法执行完毕返回时，这个栈帧会被自动销毁。
    *   程序员完全不需要关心这部分内存的分配和释放。。

### 2. 堆区 (Heap)

堆区则像一个需要手动管理的“大型仓库”，用于存放生命周期更长、尺寸更大的数据。

*   **存放内容**：
    *   所有通过 `alloc`/`new`/`copy`/`mutableCopy` 等方法创建的 **Objective-C 对象**。例如，当你写 `NSString *str = [[NSString alloc] initWithFormat:@"..."];` 时，`str` 指针存放在栈上，但它指向的那个 `NSString` 对象实体则被分配在堆上。
*   **特点**：
    *   **空间巨大**：堆的可用空间远大于栈，理论上受限于设备的物理内存和虚拟内存。
    *   **分配速度较慢**：在堆上分配和回收内存比在栈上要慢，因为系统需要查找合适的空闲内存块，并处理内存碎片等问题。
    *   **生命周期灵活**：堆上对象的生命周期与引用计数绑定，只要还有强引用指向它，它就会一直存在，可以跨越多个方法和作用域。

### 3. 全局/静态存储区 (Static/Global Storage Area)

这个区域用来存放那些在程序整个运行期间都存在的变量。它内部又可以细分为两个部分：

*   **数据段 (Data Segment)**：
    *   **存放内容**：**已初始化**的全局变量和静态变量。 例如 `static int i = 10;` 或 `NSString *globalString = @"Hello";`。

### 4. 常量存储区 (Constant Storage Area)

这个区域用来存放常量，通常是只读的。

*   **存放内容**：
    *   **字符串常量**。例如，你写的 `@"Hello, World!"`，这个字符串本身就存放在这里。
    *   其他一些编译时就能确定的常量。
*   **特点**：
    *   **只读**：这块内存区域通常是受保护的，尝试修改它会导致程序崩溃。
    *   **共享**：相同的字符串常量在内存中通常只有一份拷贝，以节约空间。

### 5. 代码区 (Code/Text Section)

这个区域存放程序要执行的二进制代码。

*   **存放内容**：
    
    *   编译后的**函数和方法的机器指令**。
    
    

### 总结与示例

```objc
NSString *globalString; // 未初始化，存放在 BSS 段

NSString *initializedGlobalString = @"Global"; // 已初始化，存放在数据段

- (void)memoryExample {
    // 'a' 是局部变量，存放在栈区
    int a = 10;

    // 'str' 这个指针本身存放在栈区
    // @"Hello" 这个字符串常量存放在常量区
    // str 指向常量区
    NSString *str = @"Hello";

    // 'array' 这个指针存放在栈区
    // [[NSMutableArray alloc] init] 创建的 NSMutableArray 对象实体存放在堆区
    // array 指向堆区
    NSMutableArray *array = [[NSMutableArray alloc] init];

    // 'staticVar' 是静态局部变量，存放在数据段 (因为它被初始化了)
    static int staticVar = 100;
    
    // ... 当方法执行结束时 ...
    // 'a', 'str', 'array' 这三个栈上的指针被销毁
    // 'array' 指向的堆上对象因为没有其他强引用，其引用计数变为0，被ARC回收
}
```

## 一个class不想他的某个属性被观察怎么办？

### `automaticallyNotifiesObserversForKey:` 阻止 KVO 的原理：中断“自动通知”的入口

要理解为什么重写 `automaticallyNotifiesObserversForKey:` 能阻止 KVO，我们需要深入到 KVO 实现的核心环节。其原理在于，这个方法是 KVO **自动通知机制的“总开关”或“决策点”**。

我们回顾一下 KVO 的标准流程：

1.  **动态子类创建**：当你为一个对象（如 `BankAccount` 实例）的属性（如 `balance`）添加第一个观察者时，运行时系统会动态创建一个 `BankAccount` 的子类，通常命名为 `NSKVONotifying_BankAccount`。

2.  **ISA Swizzling**：系统将被观察对象的 `isa` 指针从指向 `BankAccount` 修改为指向这个新的 `NSKVONotifying_BankAccount` 子类。

3.  **重写 Setter 方法**：这是最关键的一步。这个动态子类会重写被观察属性的 setter 方法。例如，`setBalance:` 方法会被重写。

现在，我们来看这个被重写的 `setBalance:` 方法**内部的伪代码**是什么样的：

```objc
// 这是在动态子类 NSKVONotifying_BankAccount 中，系统为你生成的 setBalance: 方法的“想象中”的实现

- (void)setBalance:(double)newBalance {
    // 关键决策点：在执行任何操作前，先询问“是否应该自动通知？”
    BOOL shouldAutomaticallyNotify = [[self class] automaticallyNotifiesObserversForKey:@"balance"];

    if (shouldAutomaticallyNotify) {
        // 如果答案是 YES (默认行为)
        [self willChangeValueForKey:@"balance"]; // 1. 发出“将要改变”的通知
    }

    // 调用原始类的实现，真正去改变值
    // [super setBalance:newBalance]; 会调用 BankAccount 类中原始的 setBalance: 实现
    struct objc_super superInfo = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self)) // 获取父类，即 BankAccount
    };
    void (*objc_msgSendSuper_typed)(struct objc_super *, SEL, double) = (void *)objc_msgSendSuper;
    objc_msgSendSuper_typed(&superInfo, _cmd, newBalance); // 调用父类的 setter

    if (shouldAutomaticallyNotify) {
        // 如果答案是 YES
        [self didChangeValueForKey:@"balance"]; // 2. 发出“已经改变”的通知，并触发观察者
    }
}
```

#### 原理解析

从上面的伪代码中可以清晰地看到：

1.  **前置检查**：在动态生成的 setter 方法内部，系统做的第一件事**不是**立即调用 `willChangeValueForKey:`，而是先调用 `[self class] automaticallyNotifiesObserversForKey:key]` 来进行一次询问。

2.  **条件分支**：
    *   **如果返回 `YES`** (默认情况): 程序会继续执行 `if` 语句块中的代码，即依次调用 `willChangeValueForKey:`、父类的原始 setter 和 `didChangeValueForKey:`。这构成了完整的 KVO 通知流程，观察者最终会收到通知。
    *   **如果返回 `NO`** (我们重写后的情况): `if` 条件不满足，`willChangeValueForKey:` 和 `didChangeValueForKey:` 这两个关键的通知方法将**完全被跳过**。程序只会执行中间调用父类 setter 的那部分代码，也就是只完成了属性值的修改，但没有任何通知机制被触发。

3.  **谁在调用这个方法**：调用者是**动态生成的 setter 方法**。因此，`automaticallyNotifiesObserversForKey:` 成为了这个动态 setter 内部逻辑的一个关键控制阀。

## gcd和nsthread

* 一个死锁场景

```objective-c
dispatch_queue_t queue = dispatch_queue_create("test.queue", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{    // 异步执行 + 串行队列
    dispatch_sync(queue, ^{  // 同步执行 + 当前串行队列
        // 追加任务 1
        [NSThread sleepForTimeInterval:2];              // 模拟耗时操作
        NSLog(@"1---%@",[NSThread currentThread]);      // 打印当前线程
    });
});
```

1. queue 是一个串行队列，它按照 FIFO (先进先出) 的原则开始执行 **Task A**。
2. 在 **Task A** 的执行过程中，遇到了 dispatch_sync。
3. dispatch_sync 会将内层 block 任务（我们称之为 **Task B**）添加到 queue 中，并且**阻塞** **Task A** 所在的线程，直到 **Task B** 执行完毕。
4. 现在问题来了：queue 是一个**串行队列**，它当前正在执行 **Task A**。根据串行队列的规则，它必须等 **Task A** 执行完毕后，才能去执行下一个任务，也就是 **Task B**。
5. 这就导致了：**Task A** 在等待 **Task B** 完成，而 queue 在等待 **Task A** 完成才能去执行 **Task B**。彼此互相等待，形成死锁。

* 这个一样会死锁哈，跟上面同理。

```objective-c
- (void)syncMain {
    
    NSLog(@"currentThread---%@",[NSThread currentThread]);  // 打印当前线程
    NSLog(@"syncMain---begin");
    
    dispatch_queue_t queue = dispatch_get_main_queue();
    
    dispatch_sync(queue, ^{
        // 追加任务 1
        [NSThread sleepForTimeInterval:2];              // 模拟耗时操作
        NSLog(@"1---%@",[NSThread currentThread]);      // 打印当前线程
    });

}
```

**永远不要向当前所在的同一个串行队列中，使用** **dispatch_sync** **同步派发任务。**

## GCD、NSThread、NSOperation性能上有何区别

## NSNotificationCenter通知中心的实现原理

```objective-c
    // a. Observer: 谁来收听 (通常是 self)
    // b. Selector: 收到广播后，要调用哪个方法来处理 (比如 aMethodToHandleNotification:)
    // c. Name:     要收听哪个频道 (NSNotificationName)
    // d. Object:   [可选] 只收听某个特定主持人（发送者）的广播。如果传 nil，就收听这个频道上所有人的广播。
    [center addObserver:self 
               selector:@selector(aMethodToHandleNotification:) 
                   name:@"UserDidLoginNotification" 
                 object:nil];
	 // 发通知 
   [center postNotificationName:@"UserDidLoginNotification" 
                          object:self 
                        userInfo:userInfo];
```

**目录一：按“通知名”索引 (最常用)**

*   这是一个字典 (`nameTable`)。
*   **Key**: `NSNotificationName` (通知名)。
*   **Value**: **又是一个字典！** 我们称之为 `objectTable`。

    *   在这个 `objectTable` 字典里：
        *   **Key**: `object` (发送者实例的指针)。
        *   **Value**: 一个**数组或链表**。这个列表里存放的才是最终的“订阅者信息”结构体。
            *   **订阅者信息结构体**：包含 `observer` (监听者) 和 `selector` (响应方法)。

---

### 我们用您的描述，结合这个更精确的模型，来重演一次“发送通知”的过程

假设我们执行了这行代码：
`[center postNotificationName:@"UserDidLogin" object:someLoginManager userInfo:nil];`

`NSNotificationCenter` 会执行以下一系列查找和操作：

1.  **第一步：查找“精确匹配”的订阅者**
    *   `NSNotificationCenter` 首先来到**目录一 (`nameTable`)**。
    *   它拿着 `Key = "UserDidLogin"`，在这个字典里查找。
    *   **找到了！** 它得到了一个 `objectTable`。它存的是订阅了UserDidLogin的但是 对实例要求不同的对象，要求同一个实例的作为一个列表存在objectTable中。
    *   接着，它拿着 `Key = someLoginManager` (发送者实例)，在这个 `objectTable` 里查找。
    *   **找到了！** 它得到了一个订阅者列表（比如 `listA`）。这个列表里要求的都是someLoginManager这个实例的对象。
    *   然后，它会**遍历 `listA`**，向里面的每一个 `observer` 调用它对应的 `selector`。

2.  **第二步：查找“频道匹配，不限主持人”的订阅者**
    *   `NSNotificationCenter` 再次来到**目录一 (`nameTable`)**。
    *   它还是用 `Key = "UserDidLogin"` 找到了那个 `objectTable`。
    *   这一次，它在这个 `objectTable` 里查找一个**特殊的 Key：`nil`**（代表不指定发送者）。
    *   **找到了！** 它得到了另一个订阅者列表（比如 `listB`），这个列表里是所有只关心 "UserDidLogin" 这个频道，不关心是谁发送的订阅者。
    *   然后，它会**遍历 `listB`**，一一调用他们的方法。

## SEL和imp的原理和使用

---

### SEL 的本质：一个 C 字符串（方法名）

正如上一个回答中提到的，`SEL` 的本质是一个在编译时被注册的、唯一的 C 字符串，它代表了一个方法的名字。

*   **类型**: `typedef struct objc_selector *SEL;`
*   **核心**: 它只是一个 **名字**。在整个程序中，所有同名的方法（例如，不同类中都有的 `viewDidLoad` 方法）都对应着同一个 `SEL`。编译器会维护一个全局的方法名表，`SEL` 就是指向这个表中某个名字的指针。

---

### IMP 的本质：一个函数指针

这是问题的关键。**`IMP` 的本质是一个指向方法具体实现的函数指针 (Function Pointer)**。

*   **类型**: `typedef id (*IMP)(id, SEL, ...);`

让我们来拆解这个函数指针的定义，这至关重要：

```c
id (*IMP)(id self, SEL _cmd, ...);
```

这个定义告诉我们，`IMP` 指向的任何一个函数（也就是任何一个 OC 方法的底层实现）都必须遵循这个格式：

1.  **返回值 (`id`)**: 函数的返回值类型是 `id`（一个通用的对象指针）。当然，如果方法的实际返回值是 `void`、`int` 等，这里会做相应的类型转换。

2.  **第一个参数 (`id self`)**: 这是每个方法实现都必须有的 **第一个隐藏参数**。它指向 **调用该方法的对象实例**。这就是为什么在方法内部，你可以直接使用 `self` 来引用当前对象。`objc_msgSend` 在调用 `IMP` 时，会自动把消息的接收者传给这个参数。

3.  **第二个参数 (`SEL _cmd`)**: 这是 **第二个隐藏参数**。它指向 **该方法的 `SEL`**。也就是说，在方法内部，你可以知道自己是被哪个选择器调用的。`objc_msgSend` 也会自动把调用的 `SEL` 传给这个参数。`_cmd` 并不常用，但在某些高级场景（如日志记录或基于方法名做不同处理）下很有用。

4.  **可变参数 (`...`)**: 代表方法的其他显式参数。比如 `[myObject setValue:@"newValue" forKey:@"aKey"];` 这个方法，`@"newValue"` 和 `@"aKey"` 就会作为第三、第四个参数传递给 `IMP` 指向的函数。

**所以，`IMP` 的本质就是：一个标准的 C 函数的入口地址，这个函数负责执行方法的具体逻辑。**

### 它们如何协同工作：再看 `objc_msgSend`

当我们执行 `[receiver message];` 时，完整的流程是：

1.  **编译**: 编译器将这行代码转换为 `objc_msgSend(receiver, @selector(message));`。
2.  **查找**: `objc_msgSend` 函数拿到 `receiver` 和 `SEL` (`@selector(message)`)。
3.  **定位**: 它在 `receiver` 的类（或父类）的方法列表（`method list`）中进行查找，匹配 `SEL`。
4.  **获取 IMP**: 找到与 `SEL` 绑定的那个 `Method` 结构体，并从中提取出 `IMP`（那个函数指针）。
5.  **调用**: **直接像调用一个 C 函数一样调用这个 `IMP`**，并把 `receiver` 作为第一个参数 (`self`)，把 `@selector(message)` 作为第二个参数 (`_cmd`) 传进去。
    ```c
    // 伪代码
    imp(receiver, @selector(message));
    ```

### 为什么这个分离很重要？（高级应用）

`SEL` 和 `IMP` 的分离是 Objective-C 动态性的核心。因为它允许我们在运行时去改变 `SEL` 和 `IMP` 之间的映射关系。最经典的应用就是 **Method Swizzling (方法调配)**。

Method Swizzling 的本质就是：

1.  获取两个 `SEL`：比如 `originalSEL` 和 `swizzledSEL`。
2.  通过 `SEL` 找到它们各自对应的 `originalIMP` 和 `swizzledIMP`。
3.  **交换这两个 `IMP`**。
4.  这样，当外界调用 `originalSEL` 时，实际执行的是 `swizzledIMP`；调用 `swizzledSEL` 时，执行的却是 `originalIMP`。

这就像把字典里“苹果”和“香蕉”两个词条的解释内容互换了一下。查的是“苹果”，读到的却是香蕉的解释。这种强大的能力完全建立在 `SEL` 和 `IMP` 的分离之上。

## 为什么必须在主线程中操作UI

## nsurlsession

---

### **1. `NSData * _Nullable data`**

*   **这是什么？**: 这是最核心的返回内容，代表了**服务器响应的主体（Response Body）**。
*   **数据类型**: `NSData`。这是一个通用的、用于存储任意二进制数据的“容器”对象。它可以是任何东西：
    *   **JSON 数据**: 如果你请求的是一个 API 接口，`data` 通常是一段 UTF-8 编码的 JSON 字符串的二进制表示。
    *   **图片数据**: 如果你下载的是一张图片，`data` 就是这张图片的二进制数据（如 JPEG, PNG 格式）。
    *   **HTML 数据**: 如果你请求的是一个网页，`data` 就是这个网页的 HTML 文本的二进制表示。
    *   **文件数据**: 如果下载的是一个文件（如 PDF, ZIP），`data` 就是这个文件的原始二进制数据。
*   **为什么是 `_Nullable` (可选的)？**:
    *   当网络请求**失败**时（比如网络中断、服务器错误），`data` 将会是 `nil`。
    *   当请求成功，但服务器返回的响应体就是**空**的时候，`data` 的长度可能是 0，但它本身不是 `nil`。
*   **如何使用？**:
    *   对于 **JSON**: 你需要使用 `NSJSONSerialization` 将 `NSData` 解析成 `NSDictionary` 或 `NSArray`。
        ```objc
        NSError *jsonError;
        NSDictionary *jsonDict = [NSJSONSerialization JSONObjectWithData:data options:0 error:&jsonError];
        ```
    *   对于 **图片**: 你可以直接用 `data` 创建一个 `UIImage` 对象。
        ```objc
        UIImage *image = [UIImage imageWithData:data];
        ```
    *   对于 **文本/HTML**: 你可以将其转换成 `NSString`。
        ```objc
        NSString *htmlString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        ```

---

### **2. `NSURLResponse * _Nullable response`**

*   **这是什么？**: 它包含了关于服务器响应的**元数据 (Metadata)**，但不包含响应的主体内容。可以把它想象成快递包裹上的**运单信息**，而不是包裹里的货物。
*   **数据类型**: `NSURLResponse`。在实际的网络请求中，你收到的通常是它的子类 `NSHTTPURLResponse`，这个子类包含了更多与 HTTP 协议相关的详细信息。
*   **为什么是 `_Nullable` (可选的)？**: 当请求因为客户端自身的原因失败（比如 URL 格式错误、网络连接在发送前就已断开）而根本没有收到任何服务器响应时，`response` 会是 `nil`。
*   **包含哪些重要信息？**: 在你把它转换成 `NSHTTPURLResponse` 后，可以获取到：
    *   **`statusCode`**: **HTTP 状态码**（如 200, 404, 500）。这是判断请求是否成功的**最重要依据**。
        ```objc
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        if (httpResponse.statusCode == 200) {
            // 请求成功
        }
        ```
    *   **`allHeaderFields`**: 一个包含了所有 **HTTP 响应头**的 `NSDictionary`。你可以从中获取 `Content-Type`, `Content-Length`, `Set-Cookie` 等信息。
    *   **`MIMEType`**: 服务器告知的内容类型，如 `"application/json"`, `"image/jpeg"`。
    *   **`expectedContentLength`**: 服务器告知的响应体大小。
    *   **`URL`**: 最终响应的 URL（如果发生了重定向，这可能会和你的原始请求 URL 不同）。

---

### **3. `NSError * _Nullable error`**

*   **这是什么？**: 它代表了**网络请求过程中发生的错误**。
*   **数据类型**: `NSError`。
*   **为什么是 `_Nullable` (可选的)？**: 这是判断请求是否成功的**第一个信号**。
    *   如果**网络请求成功完成**（注意，“成功完成”不代表 HTTP 状态码是 200，只代表网络层面没有错误），`error` 对象将会是 `nil`。
    *   如果请求在任何阶段**失败**，`error` 就会被填充一个描述错误信息的对象。
*   **包含哪些重要信息？**:
    *   **`domain`**: 错误的域，指明了错误的来源（如 `NSURLErrorDomain` 表示是 URL 加载系统出的错）。
    *   **`code`**: 错误码。你可以根据这个码来判断具体的错误类型，比如 `NSURLErrorNotConnectedToInternet` (没有网络连接), `NSURLErrorTimedOut` (请求超时)。
    *   **`localizedDescription`**: 一段可读的、本地化了的错误描述字符串，可以直接展示给用户。

---

### **总结：如何处理这三个参数**

一个健壮的网络请求回调处理逻辑，应该遵循以下顺序：

```objc
NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url 
                                                        completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
    
    // 1. 首先，检查 error 对象
    // 如果 error 不是 nil，说明在网络层面就出错了（如断网、超时）
    if (error) {
        NSLog(@"网络请求失败: %@", error.localizedDescription);
        // 在这里处理网络错误，比如弹窗提示用户
        return;
    }
    
    // 2. 检查 response，特别是 HTTP 状态码
    // 确保 response 是 NSHTTPURLResponse 类型
    if ([response isKindOfClass:[NSHTTPURLResponse class]]) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        
        // 如果状态码不是 2xx，说明是服务器端的错误（如 404 Not Found, 500 Internal Server Error）
        if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
            NSLog(@"HTTP 请求失败，状态码: %ld", (long)httpResponse.statusCode);
            // 在这里处理服务器错误
            return;
        }
    }
    
    // 3. 最后，处理 data
    // 走到这里，说明网络请求成功，并且服务器返回了 2xx 状态码
    if (data) {
        // 在这里进行 JSON 解析、图片转换等操作
        NSLog(@"成功接收到数据，长度: %lu", (unsigned long)data.length);
        // ... a lot of work with data
    }
    
}];

[task resume];
```
所以，`NSURLSession` 的返回数据结构，并不是一个单一的对象，而是通过**回调闭包提供的 `(data, response, error)` 这三个可选参数的组合**，它们共同为你描绘了一次网络请求从发起、响应到完成的全貌。

## 可变数组原理

类似java的arraylist，动态数组底层，满了就扩容两倍--》创建一个新的两倍容量的数组 把内容拷贝过去

## block和delegate及如何使用

## 循环引用

## 任务A、B、C，怎么让C一定在A、B后发生， A、B顺序无所谓，三个任务并列不嵌套，说出所有方法

好的，这是一个非常经典的并发编程问题，在 Objective-C 中有多种优雅且高效的解决方案。问题核心是**任务依赖**：任务 C 依赖于任务 A 和 B 的完成。

下面我将列出所有在 OC 中实现此需求的常用方法，从最推荐到最不推荐的顺序列出，并附上代码示例和优缺点分析。

### 核心思路

所有方法的共同思路都是建立一个机制，让系统知道 A 和 B 这两个任务已经完成，只有当这个“完成”信号被确认后，才开始执行 C。

---

### 方法一：使用 Grand Central Dispatch (GCD) - `DispatchGroup` (最推荐)

这是最现代、最灵活、也是最常用的 GCD 解决方案。`DispatchGroup` 就像一个任务计数器，你告诉它要追踪几个任务，每完成一个就减一，当计数器归零时，就执行你指定好的最终任务。

**原理：**
1.  创建一个 `dispatch_group_t`。
2.  在将任务 A 和 B 加入队列之前，分别调用 `dispatch_group_enter()`，使 group 的计数加一。
3.  在任务 A 和 B 的代码块执行完毕后，分别调用 `dispatch_group_leave()`，使 group 的计数减一。
4.  使用 `dispatch_group_notify()` 注册一个 block，这个 block 会在 group 计数器变为 0 时自动执行。我们将任务 C 放在这个 block 中。

**代码示例：**

```objectivec
- (void)runTasksWithDispatchGroup {
    NSLog(@"开始执行任务...");

    dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();

    // 任务A
    dispatch_group_enter(group);
    dispatch_async(concurrentQueue, ^{
        NSLog(@"任务 A 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0]; // 模拟耗时操作
        NSLog(@"任务 A 执行完毕");
        dispatch_group_leave(group);
    });

    // 任务B
    dispatch_group_enter(group);
    dispatch_async(concurrentQueue, ^{
        NSLog(@"任务 B 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0]; // 模拟耗时操作
        NSLog(@"任务 B 执行完毕");
        dispatch_group_leave(group);
    });

    // 当 Group 中所有任务都执行完毕后，会收到通知，然后执行任务C
    dispatch_group_notify(group, concurrentQueue, ^{
        // 也可以指定在主线程执行任务 C
        // dispatch_group_notify(group, dispatch_get_main_queue(), ^{ ... });
        NSLog(@"任务 A 和 B 都已完成");
        NSLog(@"任务 C 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"任务 C 执行完毕");
    });
    
    NSLog(@"主线程继续执行其他事情，不会被阻塞...");
}```

**优缺点：**
*   **优点：** 非常灵活，A 和 B 可以在不同的队列上执行，甚至可以是网络请求等异步回调。代码逻辑清晰，是处理任务组的最佳实践。完全非阻塞。
*   **缺点：** 需要手动管理 `enter` 和 `leave` 的配对，如果忘记调用 `leave` 会导致 `notify` 永远不执行。

---

### 方法二：使用 `NSOperation` 和 `NSOperationQueue` (同样非常推荐)

这是更高层级的面向对象的解决方案。`NSOperation` 天然支持任务依赖，代码非常直观易读。

**原理：**
1.  为 A、B、C 分别创建 `NSOperation` 对象（通常使用 `NSBlockOperation`）。
2.  为 C 操作添加依赖：`[operationC addDependency:operationA]` 和 `[operationC addDependency:operationB]`。
3.  将这三个操作都添加到 `NSOperationQueue` 中。队列会自动处理依赖关系，只有当一个操作的所有依赖都完成后，它才会被执行。

**代码示例：**

```objectivec
- (void)runTasksWithOperationQueue {
    NSLog(@"开始执行任务...");

    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 任务A
    NSBlockOperation *operationA = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"任务 A 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"任务 A 执行完毕");
    }];

    // 任务B
    NSBlockOperation *operationB = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"任务 B 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"任务 B 执行完毕");
    }];

    // 任务C
    NSBlockOperation *operationC = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"任务 A 和 B 都已完成");
        NSLog(@"任务 C 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"任务 C 执行完毕");
    }];

    // 关键：为C添加依赖A和B
    [operationC addDependency:operationA];
    [operationC addDependency:operationB];

    // 将所有操作添加到队列中，队列会自动处理执行顺序
    [queue addOperation:operationA];
    [queue addOperation:operationB];
    [queue addOperation:operationC];
    
    NSLog(@"主线程继续执行其他事情，不会被阻塞...");
}
```

**优缺点：**
*   **优点：** 面向对象，代码意图极其清晰，完美表达了“依赖”关系。支持更复杂的操作，如取消操作、设置优先级、KVO 监控状态等。
*   **缺点：** 相较于 GCD 有一定的性能开销（虽然在大多数场景下可以忽略不计）。

---

### 方法三：使用 GCD - `DispatchBarrier` (特定场景适用)

`DispatchBarrier`（栅栏函数）提供了一种在并发队列中创建同步点的机制。它会等待队列中在它之前提交的所有任务执行完毕，然后它自己再执行，在它执行期间，队列不会执行其他任务。等它执行完毕后，队列恢复正常。

**原理：**
1.  创建一个**并发队列** (`DISPATCH_QUEUE_CONCURRENT`)。
2.  将任务 A 和 B `async`（异步）地添加到这个队列中。
3.  将任务 C 用 `dispatch_barrier_async`（异步栅栏）的方式添加到同一个队列中。
4.  系统保证，任务 C 会在任务 A 和 B 完成后才开始执行。

**代码示例：**

```objectivec
- (void)runTasksWithDispatchBarrier {
    NSLog(@"开始执行任务...");

    // 必须是自己创建的并发队列
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.example.myqueue", DISPATCH_QUEUE_CONCURRENT);

    // 任务A
    dispatch_async(concurrentQueue, ^{
        NSLog(@"任务 A 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];
        NSLog(@"任务 A 执行完毕");
    });

    // 任务B
    dispatch_async(concurrentQueue, ^{
        NSLog(@"任务 B 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"任务 B 执行完毕");
    });

    // 栅栏任务C
    dispatch_barrier_async(concurrentQueue, ^{
        NSLog(@"任务 A 和 B 都已完成");
        NSLog(@"任务 C 开始执行 (线程: %@)", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
        NSLog(@"任务 C 执行完毕");
    });

    // 可以在栅栏后继续添加任务，这些任务会在C之后执行
    dispatch_async(concurrentQueue, ^{
        NSLog(@"任务 D 在 C 之后执行");
    });
}
```

**优缺点：**
*   **优点：** 语法简单，适用于所有任务都在同一个可控的并发队列中的场景。
*   **缺点：** **不灵活**。所有任务（A, B, C）必须在同一个你自己创建的并发队列中。如果使用全局队列 (`dispatch_get_global_queue`)，栅栏函数的效果会退化成普通的 `dispatch_async`，无法保证顺序。

## **Method Swizzling**

好的，`Method Swizzling`（方法交换）是 Objective-C Runtime 中一个非常强大但也极具风险的“黑魔法”。它允许你在运行时**交换两个方法的实现**，从而达到“钩子”（Hook）的效果，在不修改原始代码的情况下，改变或扩展原有方法的行为。

掌握它的正确使用姿势，特别是在 `+load` 方法中、使用 `dispatch_once`、以及调用原始实现，是编写健壮 Swizzling 代码的关键。

---

### **一、核心思想：交换 IMP**

在 Objective-C 中，一个“方法”在运行时的表现形式，主要由两部分构成：
*   **Selector (SEL)**: 方法的选择子，可以理解为方法的“名字”或 ID，比如 `@selector(viewWillAppear:)`。
*   **Implementation (IMP)**: 指向方法具体实现的 C 函数指针。

`Method Swizzling` 的本质，就是找到了代表两个方法的 `Method` 结构体，然后**交换了它们内部的 IMP 指针**。

**比喻一下**:
想象有两个政府官员的办公室门牌：**“市长办公室” (SEL A)** 和 **“秘书办公室” (SEL B)**。

*   通常情况下，“市长办公室”门牌后面坐的是**市长本人 (IMP A)**。
*   “秘书办公室”门牌后面坐的是**秘书本人 (IMP B)**。

`Method Swizzling` 就相当于，你偷偷地把这两个门牌摘下来，**互相换了一下**。
*   现在，你再去敲“市长办公室”的门，开门的却是**秘书**。
*   而去敲“秘书办公室”的门，开门的却是**市长**。

你并没有改变人（IMP），只是改变了名字（SEL）与人的映射关系。

---

### **二、最佳实践：一个标准的 Swizzling 实现步骤**

我们以一个最经典的例子——**为所有 `UIViewController` 的 `viewWillAppear:` 方法添加页面统计功能**——来展示标准的实现步骤。

#### **第 1 步：创建一个 `Category`**

为你想 Hook 的类创建一个 `Category`。这是组织 Swizzling 代码的最佳位置。

**UIViewController+Swizzling.h**
```objc
#import <UIKit/UIKit.h>

@interface UIViewController (Swizzling)
@end
```

#### **第 2 步：在 `+load` 方法中执行交换**

`+load` 方法是执行 Swizzling 的**唯一、绝对正确的时机**。
*   **为什么是 `+load`?**:
    1.  **调用时机早且唯一**: `+load` 方法在 App 启动时，当类被加载到内存后就会被调用，并且每个类的 `+load`（包括其所有 `Category` 的 `+load`）都只会被调用一次。这保证了我们的方法交换操作能尽早地、且仅一次地完成。
    2.  **线程安全**: 运行时在调用 `+load` 方法时，内部是加锁的，所以它是线程安全的。

**UIViewController+Swizzling.m**
```objc
#import "UIViewController+Swizzling.h"
#import <objc/runtime.h> // 引入 Runtime 头文件

@implementation UIViewController (Swizzling)

+ (void)load {
    // dispatch_once 保证了即使 +load 被意外多次调用（虽然不太可能），
    // 交换的代码也只执行一次。这是一个非常安全的编程习惯。
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        // 1. 获取原始方法和我们自定义方法的 Selector
        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(my_viewWillAppear:);
        
        // 2. 获取 Class 对象
        Class class = [self class];
        
        // 3. 获取原始方法和自定义方法的 Method 结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // 4. 【核心】尝试添加方法，并进行交换
        // class_addMethod 的作用是：如果本类中没有实现 swizzledSelector，
        // 就动态添加一个。这主要为了处理被 Hook 的方法是在父类中实现的情况。
        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {
            // 如果成功添加了方法，说明我们是在给父类的方法“打补丁”。
            // 此时，原始的 selector (viewWillAppear:) 已经指向了我们的新实现。
            // 我们需要把我们自己的 selector (my_viewWillAppear:) 的实现，
            // 替换成父类的原始实现。
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            // 如果本类已经有 viewWillAppear: 的实现，那么直接交换两个方法的 IMP 即可。
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Swizzled Method Implementation

// 这是我们自定义的、用来替换原始方法的方法
- (void)my_viewWillAppear:(BOOL)animated {
    // 【关键】调用原始实现
    // 由于方法已经被交换，此时调用 my_viewWillAppear: 实际上会执行
    // 系统原始的 viewWillAppear: 的代码。
    // 这一步是必须的，否则原始功能就丢失了。
    [self my_viewWillAppear:animated];
    
    // 在这里，添加我们自己的新功能，比如页面统计
    NSLog(@"页面即将出现: %@", NSStringFromClass([self class]));
    // [Analytics logPageView:NSStringFromClass([self class])];
}

@end
```

---

### **三、代码详解与注意事项**

1.  **为什么用 `dispatch_once`?**
    *   虽然 `+load` 本身只会被调用一次，但 `dispatch_once` 是一个万无一失的保险。它可以防止因为某些复杂的继承关系或手动调用 `+load`（错误的做法）而导致交换被执行多次。多次交换会把方法的实现换回去，导致 Swizzling 失效。

2.  **为什么不直接 `method_exchangeImplementations`?**
    *   `method_exchangeImplementations` 的一个前提是，**两个方法都必须在当前类中有实现**。
    *   考虑一个场景：`MyViewController` 继承自 `UIViewController`，但它**没有**重写 `viewWillAppear:` 方法。此时，`viewWillAppear:` 的实现是在父类 `UIViewController` 中的。
    *   如果你直接在 `MyViewController` 上交换 `viewWillAppear:` 和 `my_viewWillAppear:`，`method_exchangeImplementations` 会失败，因为它在 `MyViewController` 里找不到 `viewWillAppear:` 的实现。
    *   所以，更健壮的做法是先用 `class_addMethod` **尝试添加**。
        *   如果**成功** (`didAddMethod == YES`)，说明 `MyViewController` 本身没有实现 `viewWillAppear:`。`class_addMethod` 已经把 `viewWillAppear:` 这个 SEL 指向了我们 `my_viewWillAppear:` 的 IMP。我们现在只需要把 `my_viewWillAppear:` 这个 SEL 指向父类的原始 IMP 就行了，这就是 `class_replaceMethod` 在做的事。
        *   如果**失败**，说明 `MyViewController` 自身已经重写了 `viewWillAppear:`。这时，两个方法都在本类中有实现，我们就可以安全地使用 `method_exchangeImplementations` 来直接交换它们。

3.  **如何调用原始实现？**
    *   在 `my_viewWillAppear:` 中，当你写下 `[self my_viewWillAppear:animated];` 这行代码时，**不要**被它的名字迷惑！
    *   因为在 `+load` 中，`my_viewWillAppear:` 这个 **SEL** 对应的 **IMP** 已经被换成了**系统原始 `viewWillAppear:` 的 IMP**。
    *   所以，这行代码实际上是在**调用系统原始的实现**。这是保证原有功能不被破坏的关键一步。

---

### **总结：使用 `Method Swizzling` 的黄金法则**

1.  **时机**: 永远在 `+load` 方法中执行 Swizzling。
2.  **安全**: 永远使用 `dispatch_once` 来保证交换代码只执行一次。
3.  **健壮**: 优先使用 `class_addMethod` 和 `class_replaceMethod` 的组合来处理，以应对方法实现在父类中的情况。
4.  **责任**: 在你的自定义方法中，**必须**调用原始的实现（通过调用你自己的、已经被交换了的 selector），否则你会破坏整个调用链。
5.  **命名**: 自定义方法的命名要有清晰的前缀（如 `my_`, `swizzled_`），以避免与原始方法或第三方库的方法名冲突。
6.  **谨慎**: Swizzling 是一个全局性的、侵入性的操作。一定要清楚你这么做的目的和可能带来的副作用，避免滥用。

## runtime有哪些具体的用处

### **一、消息发送 (Message Sending)**

这是 Runtime **最核心、最基础**的功能。在 Objective-C 中，你写的每一个方法调用，比如 `[myObject doSomething];`，在编译时并不会直接转换成对某个函数地址的调用。

*   **编译时**: 它被编译器转换成了一个 C 函数调用：**`objc_msgSend(myObject, @selector(doSomething))`**。
*   **运行时**: 当程序执行到这里时，`objc_msgSend` 函数会：
    1.  通过 `myObject` 的 `isa` 指针，找到它的**类 (Class)**。
    2.  在类的**方法缓存 (`cache_t`)** 中查找是否有 `doSomething` 这个方法的实现。如果找到了，就直接跳转去执行，这是最快的情况。
    3.  如果缓存中没有，它就会去类的**方法列表 (`method_list_t`)** 中查找。
    4.  如果在本类的方法列表中找不到，它会沿着**继承链向上**，去父类的方法列表中继续查找。
    5.  一旦找到了方法的实现（一个 C 函数指针，IMP），就执行它。
    6.  如果一直到根类（`NSObject`）都找不到，就会启动下面的**消息转发**机制。

**这个动态查找的过程，就是“消息发送”。它带来了极大的灵活性，是所有其他动态特性的基础。**

---

### **二、动态方法解析与消息转发 (Dynamic Method Resolution & Message Forwarding)**

当 `objc_msgSend` 找不到一个方法的实现时，Runtime 并不会立即崩溃，而是给了你三次“拯救”的机会。

1.  **动态方法解析 (`resolveInstanceMethod:`)**:
    *   **功能**: Runtime 会先调用这个类方法，问：“我找不到 `doSomething` 的实现，你能不能现在**动态地给它添加一个**？”
    *   **应用**: 可以用来实现类似 `@dynamic` 属性的存取方法。当一个属性被声明为 `@dynamic`，编译器不会生成 getter/setter，你就可以在这个方法里，用 `class_addMethod` 动态地为这个属性添加存取方法的实现。**Core Data 的 `NSManagedObject` 就是这么做的。**

2.  **快速转发 (`forwardingTargetForSelector:`)**:
    *   **功能**: 如果上一步没有处理，Runtime 会调用这个实例方法，问：“既然你自己处理不了，那有没有**另一个对象**可以替你处理 `doSomething` 这个消息？”
    *   **应用**: 可以轻松地实现**“代理”**或**“聚合”**的效果。比如，一个对象内部持有了多个其他对象，它可以把收到的消息，根据需要转发给内部的某个对象去处理，让自己看起来像拥有了多个对象的功能。

3.  **完整转发 (`forwardInvocation:`)**:
    *   **功能**: 这是最后的机会。Runtime 会把这个消息的全部细节（包括`target`, `selector`, 所有参数）打包成一个 `NSInvocation` 对象，然后调用这个方法，说：“好了，所有信息都在这里了，你自己看着办吧！”
    *   **应用**: 这是最灵活但也最复杂的方式。你可以修改 `invocation` 的任何部分，比如改变目标、改变方法、改变参数，甚至可以不把它发出去，直接在这里设置一个返回值。**iOS 中的 `AOP`（面向切面编程）库、以及一些模拟多重继承的方案，都依赖于这个机制。**

---

### **三、关联对象 (Associated Objects)**

*   **功能**: 允许你在**运行时**，为一个**已存在的对象**（甚至是你没有源码的系统类对象），**动态地添加“实例变量”**。
*   **核心函数**: `objc_setAssociatedObject` 和 `objc_getAssociatedObject`。
*   **原理**: 它通过一个全局的哈希表（`AssociationsManager`），以“被关联的对象”为 Key，来存储你附加给它的任何其他对象。
*   **应用**:
    *   **为 `Category` 添加属性**: 这是关联对象最最经典的应用。因为 `Category` 不能直接添加实例变量，所以我们通过关联对象，来为 `Category` 添加的属性提供一个“存储空间”。比如，给 `UIView` 添加一个 `tapActionBlock` 属性。
    *   **动态地为对象附加一些上下文信息**。

---

### **四、方法交换 (Method Swizzling)**

*   **功能**: 允许你在运行时，**交换两个方法的实现 (IMP)**。
*   **核心函数**: `method_exchangeImplementations`。
*   **原理**: 它直接操作类的方法列表，找到两个 `Method` 对象，然后交换它们内部的 `IMP`（函数指针）。
*   **应用**: 这是进行 **AOP (Aspect-Oriented Programming)** 的主要手段，应用极其广泛。
    *   **全局功能注入**: 比如，为所有 `UIViewController` 的 `viewWillAppear:` 方法，都注入一段用于“页面统计”的代码。我们可以创建一个 `UIViewController` 的 `Category`，在 `+load` 方法里，将 `viewWillAppear:` 和我们自己写的 `my_viewWillAppear:` 进行交换。这样，每次系统调用 `viewWillAppear:` 时，实际上会执行到我们的代码，我们在我们的代码里先执行统计，然后再调用原始的实现。
    *   **防止数组越界 Crash**: 我们可以交换 `NSArray` 的 `objectAtIndex:` 方法和我们自己的安全版本，在我们的版本里先检查索引是否越界，如果不越界再调用原始实现。
    *   **调试和日志**: 交换关键方法，在方法调用前后打印日志。



## 消息发送机制

### 序幕：从代码到C函数

当你写下这行我们熟悉的代码时：

```objc
[receiver message];
```

编译器并不会把它翻译成一个简单的函数调用。相反，它会将其转换为对一个C函数的调用，这个函数就是大名鼎鼎的 `objc_msgSend`：

```c
objc_msgSend(receiver, selector, arg1, arg2, ...);
```

*   **`receiver`**: 消息的接收者，一个指向对象的指针 (`id`)。
*   **`selector`**: 消息的选择器 (`SEL`)，它本质上是一个C字符串，是方法名 (`message`) 的唯一标识符。在编译时，所有同名的方法都会对应同一个`SEL`。
*   **`arg1, arg2, ...`**: 传递给方法的参数。

现在，所有的魔法都发生在 `objc_msgSend` 函数的内部。这个函数通常是用汇编语言编写的，因为它需要处理可变参数、快速跳转，并且对性能要求极高。

### 正式流程：`objc_msgSend` 的寻址之旅

`objc_msgSend` 的核心任务是：**根据 `receiver` 和 `selector`，找到对应的函数实现（IMP），然后跳转去执行它。**

以下是它按优先级和顺序执行的详细步骤：

#### 第1步：空值检查（Nil Check）

这是整个流程的第一道关卡，也是 Objective-C 的一个重要安全特性。

*   **检查 `receiver` 是否为 `nil`？**
    *   **是**: 如果接收者是 `nil`，`objc_msgSend` 会直接“静默”地返回一个零值（`0`, `0.0`, `NO`, 或者 `nil`），不会执行任何后续操作，也不会报错。这就是为什么给`nil`发送消息是安全的。
    *   **否**: 如果接收者不为`nil`，则进入下一步。

#### 第2步：寻找类定义（Find the Class）

要找一个方法，首先得知道这个对象属于哪个类。

*   **通过 `receiver` 的 `isa` 指针找到它的类（Class）。**
    *   每个 Objective-C 对象（在堆上分配）的第一个成员变量就是一个 `isa` 指针。
    *   这个 `isa` 指针指向定义该对象的类。例如，一个 `NSString` 对象的 `isa` 指针就指向 `NSString` 这个类本身。`Class` 本身也是一个对象，它包含了类的所有元信息。

#### 第3步：快速路径 - 查找方法缓存（The Fast Path: Cache Lookup）

为了极致的性能，运行时系统设计了一个高效的缓存机制。因为一个方法被调用后，很可能在短时间内再次被调用。

*   **在当前类的 `cache_t` 中查找 `selector`。**
    *   每个 `Class` 结构体中都包含一个名为 `cache` 的成员，它是一个散列表（Hash Table）。
    *   这个缓存表以 `SEL` 作为 `key`，以方法的具体实现地址（`IMP`）作为 `value`。`IMP` 本质上就是一个函数指针，指向实现了该方法逻辑的C函数。
    *   查找过程非常快，通常只需要几次内存读取和比较。

*   **查找结果：**
    *   **命中（Cache Hit）**: 这是最理想的情况。如果缓存中找到了 `selector` 对应的 `IMP`，`objc_msgSend` 会立即获得这个函数指针，然后直接跳转（`GOTO`）到该地址去执行代码，并将 `receiver` 和其他参数传递过去。流程到此结束，这是消息发送的最快路径。
    *   **未命中（Cache Miss）**: 如果缓存中没有找到，说明这是该方法第一次被调用，或者缓存已被清空。此时，进入下一步“慢速路径”。

#### 第4步：慢速路径 - 遍历方法列表（The Slow Path: Method List Search）

如果在缓存中找不到，系统就必须去类的“花名册”里按名字查找了。

*   **在当前类的 `method_list_t` 中查找 `selector`。**
    *   `Class` 结构体中还包含一个方法列表，它是一个数组，存储了该类所有自己实现的实例方法（`method_t`）。
    *   `method_t` 结构体包含了方法的 `SEL` 和 `IMP`。
    *   系统会遍历这个列表，将每个方法的 `SEL` 与要查找的 `selector` 进行比较。

*   **查找结果：**
    *   **找到**: 如果在当前类的方法列表中找到了匹配的 `selector`，系统就拿到了对应的 `IMP`。
    *   **关键一步：** 在跳转去执行这个 `IMP` **之前**，系统会**将这个 `(SEL, IMP)` 对添加到第3步的缓存中**。这样，下一次再调用同一个方法时，就能直接在缓存中命中，走快速路径了。
    *   然后，跳转去执行 `IMP`。流程结束。
    *   **未找到**: 如果遍历完当前类所有的方法，还是没找到，说明这个方法不是由当前类自己实现的。此时，进入下一步。

#### 第5步：攀爬继承链 - 查找父类（Climbing the Inheritance Chain）

子类继承了父类的方法，所以如果在子类中找不到，就要去父类中继续找。

*   **通过当前类的 `superclass` 指针找到它的父类。**
    *   `Class` 结构体中有一个指向其直接父类的指针。
*   **在父类中重复查找过程。**
    *   系统会跳转到父类，然后**从第3步（查找父类的缓存）开始**，重复整个查找过程（先查父类缓存，再查父类方法列表）。
*   **循环往复**: 如果在父类中也没找到，就继续通过父类的 `superclass` 指针找到爷爷类，再重复查找... 这个过程会一直持续，直到继承链的顶端——根类（通常是 `NSObject`）。

#### 第6步：最终结局

*   **在某一级父类中找到**: 如果在向上攀爬的过程中，在某一个父类中找到了 `IMP`（无论是从它的缓存还是方法列表），系统同样会**将这个 `(SEL, IMP)` 对缓存到消息最初接收者 `receiver` 的类的缓存中**（注意：是缓存到原始类的 `cache` 里，而不是父类的），然后跳转执行。
*   **抵达根类仍未找到**: 如果一直找到了根类 (`NSObject`)，并且把根类的方法列表都查完了，还是没有找到对应的 `selector`。

    **至此，消息发送的流程宣告失败。**



## oc的消息转发机制

### 为什么需要消息转发？

想象一下，你给一个只会说中文的人（一个对象）下达了一个英文指令（发送了一个他无法响应的消息）。

*   **静态语言 (如 C++)**：编译器在编译时就会发现这个问题，直接报错：“他听不懂英文！”，程序无法通过编译。
*   **Objective-C (动态语言)**：编译器并不严格检查。程序在运行时，当这个指令真的被下达时，这个人（对象）才会发现自己听不懂。

此时，如果直接让他“宕机”（程序崩溃），未免太不灵活。于是 Objective-C 的设计者给了他一个“应急预案”，这个预案分为三步。

---

### 消息转发的三大阶段

当 `objc_msgSend` 在一个对象及其所有父类的**方法列表和缓存**中，都找不到某个 `SEL` 对应的 `IMP` 时，消息转发流程启动。

#### **第一阶段：动态方法解析 (Dynamic Method Resolution)**

*   **比喻**：“**临时学一句**”
*   **触发的方法**：Runtime 会调用该类的类方法 `+ (BOOL)resolveInstanceMethod:(SEL)sel` (对于实例方法) 或 `+ (BOOL)resolveClassMethod:(SEL)sel` (对于类方法)。
*   **你的机会**：这是你的**第一次**机会。Runtime 在问你：“嘿，我找不到 `SEL` 这个方法的实现，你能不能现在**动态地**给我创建一个，然后告诉我你创建好了？”
*   **如何实现**：你可以在这个方法里，使用 `class_addMethod` 函数，为一个“不存在”的选择器，动态地添加一个 C 函数的实现。
    ```objectivec
    #import <objc/runtime.h>
    
    // 这是一个 C 函数，它的签名必须符合 Objective-C 方法的规范
    // 前两个参数是固定的：id self, SEL _cmd
    void dynamicMethodIMP(id self, SEL _cmd) {
        NSLog(@"消息被动态解析了！我在处理这个未知的方法。");
    }
    
    + (BOOL)resolveInstanceMethod:(SEL)sel {
        // 判断是不是我们想要动态处理的那个未知方法
        if (sel == @selector(aMissingMethod)) {
            // 使用 class_addMethod 动态地将 aMissingMethod 这个 SEL
            // 指向 dynamicMethodIMP 这个 C 函数的实现
            class_addMethod([self class], sel, (IMP)dynamicMethodIMP, "v@:");
            // "v@:" 是类型编码，表示 "返回值void，参数id, SEL"
            
            // 告诉 Runtime：“我已经处理好了！”
            return YES;
        }
        // 如果不是我们想处理的，就交给父类
        return [super resolveInstanceMethod:sel];
    }
    ```
*   **结果**：如果你在这个方法里添加了实现并返回 `YES`，那么 Runtime 就会认为方法已经找到了，它会**重新发起一次消息发送**，这次就会直接调用你刚刚添加的那个 C 函数实现，**整个消息转发流程到此结束**。如果返回 `NO`，则进入第二阶段。

---

#### **第二阶段：快速转发路径 (Fast Forwarding Path)**

*   **比喻**：“**找个同事替我**”
*   **触发的方法**：`- (id)forwardingTargetForSelector:(SEL)sel`
*   **你的机会**：这是你的**第二次**机会。Runtime 知道你不会说英文，于是问你：“好吧，既然你学不会，那你们办公室里有没有**另外一个人**会说英文，我可以把这个指令直接转交给他？”
*   **如何实现**：你可以在这个方法里，返回一个**备用的、能够响应这个 `SEL` 的对象**。
    ```objectivec
    @interface BackupObject : NSObject
    - (void)aMissingMethod; // BackupObject 实现了这个方法
    @end
    
    // 在你的原始类中
    - (id)forwardingTargetForSelector:(SEL)sel {
        if (sel == @selector(aMissingMethod)) {
            // 告诉 Runtime：“别问我了，去问我的同事 aBackupObject 吧！”
            return [[BackupObject alloc] init]; 
        }
        return [super forwardingTargetForSelector:sel];
    }
    ```
*   **结果**：如果你返回了一个非 `nil` 的对象，Runtime 就会把这个消息**直接、原封不动地**转发给你返回的那个对象。就好像 `[aBackupObject aMissingMethod]` 被调用了一样。这个过程非常快，因为它只涉及一次消息的重定向。**整个消息转发流程到此结束**。如果返回 `nil`，则进入最后、最重量级的第三阶段。

---

#### **第三阶段：标准转发路径 (Normal Forwarding Path)**

*   **比喻**：“**开个会，正式讨论一下这个指令怎么办**”
*   **触发的方法**：这个阶段分为两步。
    1.  **`- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel`**
        *   **Runtime 的提问**：“好吧，你也不会，你同事也不会。那这个英文指令到底是个什么东西？它的语法结构是怎样的？它需要什么参数？返回什么结果？请你给我一份详细的**‘指令说明书’ (`NSMethodSignature`)**。”
        *   **你的职责**：你必须返回一个 `NSMethodSignature` 对象，来描述这个未知 `SEL` 的**方法签名**（参数类型、返回值类型等）。你可以为它伪造一个，或者从其他对象那里获取一个。
    2.  **`- (void)forwardInvocation:(NSInvocation *)anInvocation`**
        *   **Runtime 的指令**：只有在上一步你成功提供了“指令说明书”后，这一步才会被调用。Runtime 会把原始的消息，连同它的所有参数，完整地打包成一个 `NSInvocation` 对象，然后交给你，并对你说：“这是完整的‘任务简报’ (`anInvocation`)，包含了指令本身、所有附件材料（参数）。现在，**你自己看着办吧！**”

*   **如何实现**：
    ```objectivec
    // 步骤 1: 提供“指令说明书”
    - (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
        if (sel == @selector(aMissingMethodWithParam:)) {
            // 我们可以从另一个能响应类似方法的对象那里“借”一个方法签名
            // 或者自己手动创建一个
            return [BackupObject instanceMethodSignatureForSelector:@selector(anotherMethodWithParam:)];
        }
        return [super methodSignatureForSelector:sel];
    }
    
    // 步骤 2: 自由处理“任务简报”
    - (void)forwardInvocation:(NSInvocation *)anInvocation {
        SEL sel = anInvocation.selector;
        
        BackupObject *backup = [[BackupObject alloc] init];
    
        if ([backup respondsToSelector:sel]) {
            // 你可以选择把这个完整的任务，转发给另一个对象去执行
            [anInvocation invokeWithTarget:backup];
            
        } else {
            // 你也可以选择完全不执行，或者做一些其他的事情，比如记录日志
            NSLog(@"最终还是无法处理这个消息：%@", NSStringFromSelector(sel));
            // 如果连这里都不处理，最终就要调用父类的实现，
            // 而 NSObject 的默认实现是抛出 an unrecognized selector exception
            [super forwardInvocation:anInvocation]; 
        }
    }
    ```
*   **结果**：这是最灵活、最强大的一个阶段。`NSInvocation` 就像一个**完整的消息对象**，你可以读取它的所有信息（target, selector, arguments），也可以修改它们，甚至可以把它转发给多个不同的对象。**无论你在这里做什么，只要你不调用 `[super forwardInvocation:anInvocation]`，程序就不会崩溃。整个消息转发流程到此彻底结束。**

## 离屏渲染是什么，怎么避免？

## viewcontroller的生命周期

好的，这是一个 iOS 开发面试中的“必考题”。理解 `UIViewController` 的生命周期，是进行 iOS App 开发的基础。

一个 `UIViewController` 的生命周期，指的是从它被创建、显示在屏幕上，直到它从屏幕上消失、被销lehui的整个过程。在这个过程中，系统会在特定的时间点，自动调用一系列预先定义好的方法，让开发者有机会在这些时间点执行自己的代码。

---

### 核心生命周期方法（按典型顺序）

以下是当一个视图控制器被加载并显示出来的过程中，最核心、最常用的生命周期方法的调用顺序。

#### 1. 初始化阶段

*   **`init` / `initWithNibName:bundle:` / `initWithCoder:`**
    *   **时机**: 当你通过代码 (`[[MyViewController alloc] init]`)、从 Storyboard 或 XIB 文件创建 `UIViewController` 的实例时被调用。
    *   **职责**:
        *   这是对象的**第一次**初始化机会。
        *   **只应该**在这里做与视图（`view`）无关的、最基础的数据初始化工作。比如，初始化一些属性的默认值。
        *   **严禁**在这里访问 `self.view` 或操作任何 UI 控件，因为此时 `view` 还没有被创建，访问它会导致 `loadView` 被提前调用，引发问题。

#### 2. 视图加载阶段

*   **`loadView`**
    *   **时机**: 当 `self.view` **第一次被访问**，并且它恰好是 `nil` 的时候，这个方法会被自动调用。
    *   **职责**:
        *   它的**唯一职责**就是**创建**视图控制器的根视图（`self.view`）。
        *   如果你是纯代码开发，不使用 Storyboard 或 XIB，那么你通常会重写这个方法，在这里创建一个 `UIView`（或者 `UIScrollView` 等）并赋值给 `self.view`。
        *   **如果你重写了 `loadView`，你必须自己给 `self.view` 赋值，并且绝对不能调用 `[super loadView]`。**
        *   如果你使用 Storyboard 或 XIB，系统会自动帮你完成 `loadView` 的工作，你**永远不应该**主动调用或重写它。

*   **`viewDidLoad`**
    *   **时机**: 在 `loadView` 成功创建并加载了视图层级（`view hierarchy`）到内存后被调用。此时，`self.view` 已经有值，并且 `outlet` 已经被连接。
    *   **职责**:
        *   这是生命周期中**最常用**的方法之一。
        *   在这里进行视图的**一次性设置**。因为 `viewDidLoad` 在整个生命周期中通常只会被调用一次。
        *   **适合做**：
            *   添加子视图 (`[self.view addSubview:...]`)。
            *   设置约束 (Auto Layout)。
            *   为 UI 控件赋值初始数据。
            *   发起网络请求，加载页面所需的数据。

#### 3. 视图显示/消失阶段 (可能会被多次调用)

这一组方法会在视图控制器每次出现或消失时被成对调用，比如 Tab 切换、Push/Pop、Present/Dismiss。

*   **`viewWillAppear:`**
    *   **时机**: 在视图即将被添加到视图层级中，显示在屏幕上**之前**被调用。
    *   **职责**:
        *   做一些视图显示前的准备工作。
        *   **适合做**：
            *   更新 UI 状态，确保界面显示的是最新数据。
            *   设置 `navigationBar` 的样式。
            *   注册通知或 KVO。
            *   启动一些动画。

*   **`viewDidAppear:`**
    *   **时机**: 在视图已经完全显示在屏幕上，动画执行完毕**之后**被调用。
    *   **职责**:
        *   处理视图显示之后才能执行的任务。
        *   **适合做**：
            *   处理一些比较耗时的数据加载或计算（避免阻塞 `viewWillAppear` 影响过渡动画）。
            *   启动复杂的动画或视频播放。
            *   弹出提示框。

*   **`viewWillDisappear:`**
    *   **时机**: 在视图即将从视图层级中移除，从屏幕上消失**之前**被调用。
    *   **职责**:
        *   执行清理工作或保存状态。
        *   **适合做**：
            *   保存用户输入的数据。
            *   取消网络请求。
            *   移除通知或 KVO。
            *   将 `navigationBar` 恢复原状。

*   **`viewDidDisappear:`**
    *   **时机**: 在视图已经完全从屏幕上消失，动画执行完毕**之后**被调用。
    *   **职责**:
        *   执行最终的清理工作。
        *   **适合做**：
            *   停止动画、计时器或视频播放。
            *   释放一些大的、只在页面显示时才需要的资源。

#### 4. 内存与销毁阶段

*   **`dealloc`**
    *   **时机**: 当视图控制器的引用计数为0，即将被系统销毁时调用。
    *   **职责**:
        *   执行最终的、彻底的清理工作。
        *   **必须做**：
            *   移除所有未移除的通知 (`[[NSNotificationCenter defaultCenter] removeObserver:self]`)。
            *   将 `delegate` 指针置为 `nil` (在非 ARC 或需要手动管理的情况下)。
            *   停止所有 `NSTimer` 并将其置为 `nil`。
            *   取消 KVO 监听。
        *   **严禁**在这里调用 `[super dealloc]` (在 ARC 环境下)。

---

### 视图布局相关的生命周期方法

除了上述核心方法，还有一组与 Auto Layout 相关的生命周期方法。

*   **`viewWillLayoutSubviews`**
    *   **时机**: 在视图控制器的 `view` 即将布局其子视图**之前**被调用。
    *   **职责**: 在这里可以修改约束，为即将到来的布局做准备。

*   **`viewDidLayoutSubviews`**
    *   **时机**: 在视图控制器的 `view` 已经布局完其子视图**之后**被调用。
    *   **职责**: 在这里可以获取到子视图**最终**的、准确的 `frame`。如果你的某些操作依赖于 UI 控件的最终尺寸和位置，应该在这里执行，而不是在 `viewDidLoad` 里（因为那时的 `frame` 可能是不准确的）。

---

### 总结图

```
 [init] -> [loadView] -> [viewDidLoad]
     |                               ^
     |                               | (内存警告时，view被卸载，再次访问时重新加载)
     |                               |
     +-------------------------------+
     |
     V
 [viewWillAppear:] -> [viewWillLayoutSubviews] -> [viewDidLayoutSubviews] -> [viewDidAppear:]
     ^                                                                            |
     | (页面再次出现)                                                            | (页面消失)
     +--------------------------------------------------------------------------+
     |
     V
 [viewWillDisappear:] -> [viewDidDisappear:]
     |
     | (VC 被销毁)
     V
 [dealloc]
```

**面试时的回答策略**:
1.  **先说核心流程**: 按照 `init` -> `loadView` -> `viewDidLoad` -> `viewWillAppear` -> `viewDidAppear` -> `viewWillDisappear` -> `viewDidDisappear` -> `dealloc` 的顺序清晰地讲一遍。
2.  **解释每个方法的核心职责**: 清楚地说明在每个方法里“应该做什么”和“不应该做什么”。
3.  **强调 `viewDidLoad` 与 `viewWillAppear` 的区别**: `viewDidLoad` 只调用一次，适合做一次性初始化；`viewWillAppear` 会被多次调用，适合做每次页面出现都需要更新的操作。
4.  **(加分项) 提及布局相关的方法**: 主动说出 `viewWillLayoutSubviews` 和 `viewDidLayoutSubviews`，并解释它们与 `viewDidLoad` 在获取 `frame` 上的区别，能体现你对 Auto Layout 流程的理解。
5.  **(加分项) 提及内存警告**: 说明在旧的系统版本中，`view` 可能会在内存警告时被卸载，之后再次触发 `loadView` 和 `viewDidLoad` 的流程。

## 字典的key一般是什么？

一般是一些不可变对象，比如**NSString**和**NSNumber**。

* 如果一定要自定义对象作为key的话，必须实现`copyWithZone`方法和`hash`方法和`isEqual`，而且要确保计算hash值的那些属性不可变。

## iOS中，自动布局和基本布局的区别

* 基本布局就是frame，里面origin属性说明了你在父视图的一个绝对位置。
* 自动布局就是和其他视图的一个相对关系，一个约束

## OC中的死循环有什么？可能引起什么crash

## Block的内存管理

### 一、 Block的三种类型及其存储位置

一个Block在创建时，根据其内部实现和上下文，它可能是以下三种类型之一：

1.  **`__NSGlobalBlock__` (全局Block)**
    *   **存储位置**: **数据段 (Data Segment)**，和全局变量、静态变量存放在一起。
    *   **产生条件**: 当一个Block的实现体中，**没有使用任何外部的、需要捕获的局部变量**时，它就是一个全局Block。它就像一个普通的C函数，本身不依赖任何外部状态。
    *   **内存管理**: 它的生命周期和整个App一样长，**不需要进行任何内存管理**。对它进行`copy`或`retain`操作，什么都不会发生。
    *   **示例**:
        ```objective-c
        void (^myGlobalBlock)(void) = ^{
            NSLog(@"Hello, World!");
        };
        NSLog(@"Block type: %@", [myGlobalBlock class]); // 输出: __NSGlobalBlock__
        ```

2.  **`__NSStackBlock__` (栈Block)**
    *   **存储位置**: **栈 (Stack)**。
    *   **产生条件**: 当一个Block**使用了外部的局部变量**，但还没有被`copy`操作时，它就是一个栈Block。它的生命周期和它所在的**作用域（scope）**绑定。
    *   **内存管理**: 这是**最危险**的一种Block。它存放在栈上，一旦其所在的方法或函数执行完毕，栈帧被销毁，这个Block的内存就会被**立刻回收**。如果在其作用域之外调用这个Block，就会导致**野指针和程序崩溃**。
    *   **示例 (MRC环境下)**: 在ARC下，编译器通常会自动优化，很难直接观察到栈Block，但在MRC或特定场景下它会出现。
        ```objective-c
        // 假设在MRC环境下
        int age = 25;
        void (^myStackBlock)(void) = ^{
            NSLog(@"Age is %d", age);
        };
        NSLog(@"Block type: %@", [myStackBlock class]); // 输出: __NSStackBlock__
        ```

3.  **`__NSMallocBlock__` (堆Block)**
    *   **存储位置**: **堆 (Heap)**。
    *   **产生条件**: **一个栈Block被`copy`操作之后，就会变成一个堆Block。** 这是Block能跨作用域传递和使用的关键。
    *   **内存管理**: 堆Block是一个真正的Objective-C对象，它的生命周期由**引用计数**来管理。
        *   当被`copy`时，引用计数+1。
        *   当被其他`strong`指针持有时，引用计数+1。
        *   当持有它的`strong`指针消失时，引用计数-1。
        *   当引用计数为0时，它和它所捕获的所有对象会被销毁。
    *   **示例**:
        ```objective-c
        int age = 25;
        void (^myBlock)(void) = ^{
            NSLog(@"Age is %d", age);
        };
        // 执行copy操作，将其从栈复制到堆
        id myHeapBlock = [myBlock copy];
        NSLog(@"Block type: %@", [myHeapBlock class]); // 输出: __NSMallocBlock__
        ```

---

### 二、 ARC下的自动`copy`

在**ARC（自动引用计数）**环境下，为了避免栈Block带来的危险，编译器为我们做了很多自动优化。以下情况，编译器会自动将一个栈Block执行`copy`操作，将其转化为堆Block：

1.  **当Block作为方法的返回值时。**
2.  **当Block被赋值给一个`strong`修饰的Block类型属性时。**
3.  **当Block作为参数，传递给某些系统方法时**（比如`dispatch_async`, `UIView`的动画Block等）。
4.  **当Block被显式调用 `copy` 方法时。**

**结论**：在ARC时代，你基本上可以认为，只要你将一个Block赋值给一个变量或属性，或者传递它，它就已经被自动地、安全地转换成了**堆Block**。你不再需要手动`copy`。

---

### 三、 变量捕获机制 (The Magic)

这是Block内存管理最核心、也最容易出问题的地方。当一个Block使用了外部的变量时，它会“捕获”这个变量。捕获的方式取决于变量的类型和修饰符。

#### 1. 捕获局部基本数据类型 (如`int`, `float`)

*   **方式**: **值捕获 (Capture by Value)**。
*   **行为**: Block在创建时，会**复制**一份该变量的**当前值**，并存储在Block自己的结构体内部。之后，外部变量的任何改变，都**不会**影响Block内部的那个副本。
*   **示例**:
    ```objective-c
    int age = 25;
    void (^myBlock)(void) = ^{
        NSLog(@"Age inside block: %d", age); // age = 25
    };
    age = 30;
    myBlock(); // 输出: "Age inside block: 25"
    ```

#### 2. 捕获局部对象类型 (如`NSString *`)

*   **方式**: **指针捕获，并根据所有权修饰符决定强弱关系**。
*   **默认行为 (`strong`)**:
    *   默认情况下，Block会**强引用（`retain`）**它所捕获的对象。
    *   这个强引用会一直持续到**堆Block本身被销毁**为止。
    *   **这是导致循环引用的根源！**
*   **示例**:
    ```objective-c
    NSMutableArray *array = [NSMutableArray arrayWithObject:@"A"];
    void (^myBlock)(void) = ^{
        [array addObject:@"B"];
        NSLog(@"Array: %@", array);
    };
    [array addObject:@"C"];
    myBlock(); // 输出: "Array: (A, C, B)"
    // Block捕获的是array这个指针，所以外部对array指向的对象的修改是可见的。
    ```

#### 3. 使用 `__block` 修饰符

如果你希望在Block内部能够**修改**一个外部的局部变量，你必须用 `__block` 来修饰它。

*   **行为**: 当一个变量被 `__block` 修饰后，它不再是简单的值捕获或指针捕获。编译器会把它包装成一个**特殊的对象**。Block捕获的是这个**包装对象的指针**。
*   **效果**:
    *   Block内部和外部操作的，都是这同一个包装对象里的值。
    *   因此，在Block内部对`__block`变量的修改，在外部是可见的，反之亦然。
    *   **注意**: 对于对象类型的`__block`变量，在ARC下，Block同样会对其产生**强引用**，这也会导致循环引用！
*   **示例**:
    ```objective-c
    __block int age = 25;
    void (^myBlock)(void) = ^{
        age = 30; // 可以修改了
        NSLog(@"Age inside block: %d", age); // age = 30
    };
    myBlock();
    NSLog(@"Age outside block: %d", age); // 输出: "Age outside block: 30"
    ```

---

### 四、 Block与循环引用 (The Ultimate Trap)

这是Block内存管理的终极话题。

*   **发生条件**:
    1.  一个对象 `obj` 持有了一个Block（通常是`strong`属性）。
    2.  这个Block的实现体内部，又强引用了 `obj`（通常是直接使用了`self`或其成员变量）。

*   **图示**:
    `self` -> (strong) -> `block` -> (strong) -> `self`

*   **解决方案：打破循环！**
    使用 `__weak` 修饰符来告诉Block，不要强引用`self`。

*   **标准解法**:
    ```objective-c
    // 创建一个self的弱引用副本
    __weak typeof(self) weakSelf = self;
    
    self.myBlock = ^{
        // 在Block内部，为了防止weakSelf在执行过程中被释放，
        // 再创建一个strong引用来临时持有它。
        __strong typeof(weakSelf) strongSelf = weakSelf;
        
        // 确保strongSelf不是nil才执行后续操作
        if (!strongSelf) {
            return;
        }
    
        // 现在可以安全地使用 strongSelf 了
        [strongSelf doSomething];
        NSLog(@"Name: %@", strongSelf.name);
    };
    ```
    *   `__weak typeof(self) weakSelf = self;`: Block捕获的是`weakSelf`，它对`self`是一个弱引用，不会增加`self`的引用计数，**循环被打破**。
    *   `__strong typeof(weakSelf) strongSelf = weakSelf;`: 这是一个**作用域内的临时强引用**。它的作用是防止在Block执行期间（特别是多线程异步执行时），`self`恰好被释放了，导致`weakSelf`变为`nil`，从而引发问题。这个`strongSelf`会在Block执行完毕后自动释放。

## ios怎么实现单例模式

```objc
// MyManager.m
// 用NS_UNAVAILABLE将 init, new, copy, mutableCopy这四个方法全部私有
#import "MyManager.h"

@implementation MyManager

// 核心的全局访问方法
+ (instancetype)sharedManager {
    // 1. 声明一个静态的、指向唯一实例的指针。
    //    `static` 保证了这个变量在整个文件范围内是唯一的，并且在App生命周期内只被初始化一次。
    static MyManager *sharedInstance = nil;
    
    // 2. 声明一个 dispatch_once_t 令牌。
    //    这是一个必须的、用于 dispatch_once 的“锁”，它也必须是静态的。
    static dispatch_once_t onceToken;
    
    // 3. 使用 dispatch_once 来执行初始化代码。
    //    这是整个实现中最关键的部分。
    dispatch_once(&onceToken, ^{
        // 这个 Block 里的代码，在整个App的生命周期中，
        // 无论从哪个线程、被调用多少次，都只会被执行“有且仅有”的一次。
        // GCD 底层保证了其原子性和线程安全性。
        sharedInstance = [[self alloc] initInstance];
    });
    
    // 4. 返回那个唯一的、已经被创建好的实例。
    return sharedInstance;
}

// 一个私有的、真正的初始化方法
- (instancetype)initInstance {
    self = [super init];
    if (self) {
        NSLog(@"MyManager instance has been initialized.");
    }
    return self;
}

// 覆盖 allocWithZone: 来防止通过 [[MyManager alloc] init] 创建新实例 (更彻底的保护)
+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    // 理论上，有了 dispatch_once 和 NS_UNAVAILABLE 已经足够安全。
    // 但覆盖 allocWithZone 是一个更“偏执”、更经典的保护方式，
    // 确保任何形式的 alloc 调用都返回同一个实例。
    return [self sharedManager];
}


@end
```

## weak在所在对象释放后 是如何置空的

* 系统会维护一个弱引用的map，key是对象的地址，value是弱引用指针的地址数组，当一个对象被销毁时，会去查这个对象在不在表里，如果在，就遍历value，将弱引用指针全部置为nil。
* 所以当你用weak指针指向一个对象的时候，系统也会自动的将你这个weak指针加到这个表里。

## category的原理

好的，`Category` (分类) 是 Objective-C 语言中一个极其强大、灵活且独特的特性。理解它的原理，对于深入掌握 Objective-C 的动态特性和许多常见框架（如 Foundation 中的 `NSObject+AssociatedObject`）的设计思想至关重要。

简单来说，`Category` 的核心作用是：**在不修改原始类源代码、也不知道其内部实现的情况下，为这个类动态地添加新的方法。**

---

### 一、`Category` 能做什么，不能做什么？

在深入原理之前，我们先明确它的能力边界：

**能做到的：**
1.  **添加实例方法**: 这是最主要、最常见的用途。可以为一个已存在的类（甚至是你没有源码的系统类，如 `NSString`）添加新的功能。
2.  **添加类方法**: 同样可以为类本身添加新的方法。
3.  **实现协议**: 可以让一个类遵循某个新的协议，并实现协议中定义的方法。
4.  **拆分实现**: 将一个庞大的类的实现，拆分到多个不同的 `.m` 文件中，以提高代码的组织性和可维护性。

---

### 二、`Category` 的编译与加载原理

`Category` 的魔法，完全是在**运行时 (Runtime)** 才真正展现的。

#### **1. 编译时**

*   当你编译一个 `Category` 文件时（比如 `NSString+MyAdditions.m`），编译器会把它编译成一个独立的二进制文件。
*   在编译产物中，`Category` 被表示为一个名为 `category_t` 的 C 结构体。这个结构体里包含了关于这个 `Category` 的所有信息：
    ```c
    struct category_t {
        const char *name; // Category 所属的原始类的名字，如 "NSString"
        classref_t cls;   // 指向原始类对象的指针 (编译时通常是 nil)
    
        // 【核心】方法列表
        struct method_list_t *instanceMethods; // 添加的实例方法列表
        struct method_list_t *classMethods;    // 添加的类方法列表
    
        // 协议列表
        struct protocol_list_t *protocols; // 实现的协议列表
    
        // 属性列表 (注意：这里只记录了 @property 的声明，不包含实例变量)
        struct property_list_t *instanceProperties; 
        
        // ... 其他字段
    };
    ```
*   **关键点**: 在编译时，原始类（`NSString`）和 `Category`（`NSString+MyAdditions`）是**完全分离**的两个东西。

#### **2. 运行时（App 启动时）**

当你的 App 启动时，Objective-C 的运行时系统会执行一系列的加载和初始化工作。其中一个关键步骤就是**处理所有的 `Category`**。

这个过程大致如下：

1.  **加载镜像**: 运行时会加载程序的所有二进制镜像（包括主程序、动态库、以及所有 `Category` 的编译产物）。

2.  **发现 `Category`**: 运行时会扫描这些镜像，找到所有 `category_t` 结构体。

3.  **附加 `Category` (Attaching Categories)**: 这是最核心的步骤。运行时会遍历所有找到的 `category_t`，然后执行一个类似“合并”的操作。
    *   **定位原始类**: 运行时根据 `category_t` 中的 `name` 字段，找到它要扩展的那个原始类（比如 `NSString` 类对象）。
    *   **合并方法**: 运行时会将 `Category` 的**方法列表、协议列表和属性列表**，**“附加”**到原始类的总列表中。
        *   **方法列表的合并**: 这不是简单的替换，而是将 `Category` 的方法列表**插入**到原始类的方法列表的**最前面**。
        *   **这就是为什么 `Category` 的方法会“覆盖”原始类方法**：当进行方法查找（消息发送）时，运行时会遍历一个类的方法列表。由于 `Category` 的方法被加在了最前面，所以它会**先被找到并执行**。

**一个重要的细节**:
*   这个“附加”的过程，是在 App 启动时，`main()` 函数执行之前，由 `dyld`（动态链接器）和运行时协作完成的。
*   这个过程是**一次性的**。一旦附加完成，`Category` 中的方法就成了这个类“固有”的一部分，从外部看，无法区分一个方法是来自原始类还是来自 `Category`。

---

### 三、`Category` 与 `+load` 和 `+initialize` 的关系

*   **`+load` 方法**:
    *   `+load` 方法是在**运行时加载类和 `Category` 时**被调用的，它在 `main()` 函数之前执行。
    *   **原始类和所有它的 `Category`，只要定义了 `+load` 方法，就都会被调用一次**。
    *   调用的顺序是：先调用原始类的 `+load`，然后按照编译链接的顺序，依次调用每个 `Category` 的 `+load`。
    *   `+load` 非常适合用来执行一些“一次性”的、依赖于运行时但又需要在所有代码执行前的设置，比如**方法交换 (Method Swizzling)**。

*   **`+initialize` 方法**:
    *   `+initialize` 是在类**第一次接收到消息时**被调用的（懒加载）。
    *   如果一个类和它的 `Category` 都实现了 `+initialize`，那么**只有最后一个被编译器链接的 `Category` 的 `+initialize` 方法会生效**，它会“覆盖”掉原始类和其他 `Category` 的实现。这是因为 `+initialize` 本身就是一个普通的方法调用，遵循 `Category` 的方法覆盖规则。

## 子线程中如何管理对象的生命周期

## GCD

### Dispatch Queues

#### 1. 核心概念：任务、同步与异步、串行与并发

在使用 Dispatch Queues 之前，需要先理解几个核心概念：

*   **任务 (Task):** 你希望执行的代码块。在 Objective-C 中，这通常是一个 Block。
*   **同步 (Synchronous) vs. 异步 (Asynchronous):**
    *   **同步 (`dispatch_sync`):** 会阻塞当前线程，直到添加到队列的任务执行完毕。 调用 `dispatch_sync` 并将目标设置为当前队列会导致死锁。
    *   **异步 (`dispatch_async`):** 不会阻塞当前线程，会立即返回。任务会在稍后的某个时间点在另一个线程上执行。
*   **串行 (Serial) vs. 并发 (Concurrent):**
    *   **串行队列 (Serial Queue):** 任务按照先进先出 (FIFO) 的顺序，一次只执行一个。 这对于保护共享资源或需要按特定顺序执行任务的场景非常有用。
    *   **并发队列 (Concurrent Queue):** 同样遵循 FIFO 的顺序开始执行任务，但可以同时执行多个任务，无需等待前一个任务完成。 这使得并发队列非常适合执行可以独立且同时进行的任务。

#### 2. Dispatch Queues 的类型

GCD 提供了几种不同类型的队列：

*   **主队列 (Main Dispatch Queue):**
    *   一个全局可用的串行队列。
    *   所有提交到主队列的任务都在应用程序的主线程上执行。
    *   主要用于更新 UI，因为所有 UI 操作都必须在主线程上进行。
    *   通过 `dispatch_get_main_queue()` 获取。

*   **全局并发队列 (Global Concurrent Queues):**
    *   由系统提供的一组并发队列。
    *   这些队列具有不同的服务质量 (Quality of Service, QoS) 或优先级，以告知系统任务的重要性。
    *   通过 `dispatch_get_global_queue()` 获取，可以指定优先级。

*   **自定义队列 (Custom Queues):**
    *   使用 `dispatch_queue_create()` 函数创建你自己的串行或并发队列。
    *   创建时需要提供一个唯一的标签（通常是反向 DNS 命名），这对于调试非常有帮助。
    *   **创建串行队列:**
        ```objectivec
        dispatch_queue_t mySerialQueue = dispatch_queue_create("com.example.mySerialQueue", DISPATCH_QUEUE_SERIAL);
        ```
        或者，第二个参数传 `NULL` 也是默认创建串行队列。
    *   **创建并发队列:**
        ```objectivec
        dispatch_queue_t myConcurrentQueue = dispatch_queue_create("com.example.myConcurrentQueue", DISPATCH_QUEUE_CONCURRENT);
        ```

#### 3. 如何使用 Dispatch Queues

将任务提交到队列的基本语法如下：

*   **异步提交任务到全局队列:**
    ```objectivec
    // 获取一个全局并发队列
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    // 异步执行任务
    dispatch_async(globalQueue, ^{
        // 在后台执行耗时操作，例如网络请求
        NSLog(@"后台任务正在执行...");
    
        // 操作完成后，回到主线程更新UI
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"在主线程更新UI");
            // self.myLabel.text = @"更新完成";
        });
    });
    ```

*   **同步提交任务到自定义串行队列:**
    ```objectivec
    dispatch_queue_t mySerialQueue = dispatch_queue_create("com.example.mySerialQueue", NULL);
    
    dispatch_sync(mySerialQueue, ^{
        // 这个代码块会阻塞当前线程，直到执行完毕
        NSLog(@"同步任务执行");
    });
    ```

### **Dispatch Group**

好的，我们来着重深入地介绍 **GCD 的任务组 (Dispatch Group)**。这是 GCD 中一个极其强大且实用的工具，专门用于解决“**等待多个异步任务全部执行完毕后，再去做某件事**”的场景。

**核心思想：一个智能的“任务计数器”**

你可以把 `DispatchGroup` 想象成一个非常聪明的计数器，它的工作逻辑如下：

1.  **初始化**：创建一个任务组，此时计数器为 0。
2.  **任务进组 (`enter`)**：每当你要开始一个需要追踪的异步任务时，你手动调用 `enter` 方法。这会使任务组的内部计数器 **加 1**。
3.  **任务出组 (`leave`)**：当你追踪的那个异步任务执行完毕时（无论是在它的回调 block 中，还是在其他地方），你再手动调用 `leave` 方法。这会使任务组的内部计数器 **减 1**。
4.  **监听完成 (`notify`)**：你可以预先注册一个“通知任务”。当任务组的计数器从 1 变为 0 时（也就是所有“进组”的任务都已经“出组”了），系统会自动执行你注册的这个通知任务。

这个机制的美妙之处在于，它完全不关心你追踪的任务是什么、在哪个线程上执行、需要多长时间。它只关心 `enter` 和 `leave` 是否平衡。

---

**核心 API 详解**

理解任务组，关键是掌握下面这几个 API：

1.  **`dispatch_group_create()`**
    *   **作用**：创建一个新的、空的 Dispatch Group。
    *   **返回值**：`dispatch_group_t`，你后续操作所依赖的句柄。
2.  **`dispatch_group_enter(dispatch_group_t group)`**
    *   **作用**：手动通知 `group`，一个任务已经加入到组中。
    *   **调用时机**：在你发起一个需要追踪的异步操作**之前**调用。
    *   **效果**：`group` 的内部未完成任务数 +1。
3.  **`dispatch_group_leave(dispatch_group_t group)`**
    *   **作用**：手动通知 `group`，一个之前加入的任务已经执行完毕。
    *   **调用时机**：在异步操作完成的回调中调用。
    *   **效果**：`group` 的内部未完成任务数 -1。
    *   **重要**：`enter` 和 `leave` **必须严格配对**。多一个 `enter` 会导致 `notify` 永远不执行；多一个 `leave` 会导致程序崩溃。
4.  **`dispatch_group_notify(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block)`**
    *   **作用**：注册一个回调 Block，当 `group` 里的任务全部完成（计数器归零）时，这个 Block 会被**异步**地提交到你指定的 `queue` 中执行。
    *   **特性**：**非阻塞**。调用 `dispatch_group_notify` 会立即返回，当前线程会继续执行下面的代码，不会被卡住。这是它最大的优点。
    *   `queue`: 你希望最终的“收尾任务”在哪个队列上执行。最常见的用法是传入 `dispatch_get_main_queue()`，以便在所有后台任务完成后更新 UI。
    *   `block`: 所有任务完成后要执行的代码。

---

**经典应用场景：并发网络请求**

假设你需要从两个不同的服务器 API 获取数据（用户信息和产品列表），并且必须在这两个请求都成功返回后，才能刷新界面。

**使用 Dispatch Group 的完美实现：**

```objectivec
- (void)fetchAllDataAndRefreshUI {
    NSLog(@"开始加载所有数据...");
    
    // 1. 创建一个任务组
    dispatch_group_t group = dispatch_group_create();
    
    // 2. 创建一个用于执行网络请求的队列
    dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    __block id userInfo;       // 用于存储请求结果
    __block NSArray *productList; // 用于存储请求结果

    // --- 任务 A: 获取用户信息 ---
    // 2.1 手动让任务进组
    dispatch_group_enter(group); 
    dispatch_async(concurrentQueue, ^{
        NSLog(@"开始请求用户信息...");
        // 模拟网络请求
        [self fetchFromServer:@"/api/user" completion:^(id result) {
            NSLog(@"获取用户信息成功!");
            userInfo = result;
            // 2.2 任务完成，手动让任务出组
            dispatch_group_leave(group); 
        }];
    });

    // --- 任务 B: 获取产品列表 ---
    // 3.1 手动让任务进组
    dispatch_group_enter(group);
    dispatch_async(concurrentQueue, ^{
        NSLog(@"开始请求产品列表...");
        // 模拟网络请求
        [self fetchFromServer:@"/api/products" completion:^(id result) {
            NSLog(@"获取产品列表成功!");
            productList = result;
            // 3.2 任务完成，手动让任务出组
            dispatch_group_leave(group);
        }];
    });

    NSLog(@"两个请求已发出，主线程继续执行其他事情...");

    // 4. 注册一个通知：当 group 中所有任务都完成后，在主线程执行
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"所有网络请求均已完成！");
        
        // 在这里，我们可以安全地认为 userInfo 和 productList 都有值了
        // 可以在主线程刷新 UI
        [self.nameLabel setText:userInfo[@"name"]];
        [self.tableView reloadData];
        
        NSLog(@"UI 刷新完毕！");
    });
}

// 模拟一个网络请求方法
- (void)fetchFromServer:(NSString *)api completion:(void (^)(id result))completion {
    // 模拟随机耗时
    double delay = arc4random_uniform(3) + 1.0; 
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        if ([api isEqualToString:@"/api/user"]) {
            completion(@{@"name": @"张三", @"uid": @"1001"});
        } else {
            completion(@[@"iPhone 20", @"MacBook Pro M5"]);
        }
    });
}
```

### Dispatch Barrier

好的，我们来详细深入地探讨 Objective-C 中的 Dispatch Barriers（调度栅栏）。

#### 1. 什么是 Dispatch Barrier？

Dispatch Barrier（调度栅栏）是 Grand Central Dispatch (GCD) 提供的一种强大的同步机制。它的核心功能是在一个**自定义的并发队列 (Custom Concurrent Queue)** 中创建一个“瓶颈”或“同步点”。

想象一下一个多车道的收费站：

*   **普通任务 (`dispatch_async`)**：就像多条车道上的汽车，可以并行通过收费站。
*   **栅栏任务 (`dispatch_barrier_async`)**：就像一个需要临时关闭所有车道进行维护的指令。在维护期间（即栅栏任务执行期间），任何新的汽车（新任务）都不能进入收费站，而已经在收费站内的汽车（已开始执行的任务）会继续通过。维护完成后，收费站重新开放，汽车又可以并行通过了。

具体来说，当您将一个栅栏任务提交到一个并发队列时，会发生以下情况：

1.  队列会等待所有在栅栏任务**之前**提交的普通任务执行完毕。
2.  在等待期间，队列不会开始执行任何在栅栏任务**之后**提交的新任务。
3.  一旦所有先前的任务都完成了，队列就会开始**单独执行**这个栅栏任务。在栅栏任务执行期间，队列中不会有其他任何任务在执行。
4.  当栅栏任务执行完毕后，队列会恢复其正常的并发行为，继续并行执行在栅栏任务之后提交的任务。

这个特性使得 Dispatch Barrier 成为实现**“读者-写者”模式 (Readers-Writer Pattern)** 的完美工具。

#### 2. 为什么需要 Dispatch Barrier？—— 经典的“读者-写者”问题

在并发编程中，一个常见的问题是如何安全地访问一个可变的数据集合（例如 `NSMutableArray` 或 `NSMutableDictionary`）。

*   **读取 (Reading)** 操作通常是线程安全的，只要在读取期间没有写入操作即可。多个线程可以同时读取数据而不会产生问题。
*   **写入 (Writing)** 操作（包括添加、修改、删除）则不是线程安全的。如果一个线程正在写入，而另一个线程试图读取或写入，就可能导致数据损坏或应用崩溃。

**解决方案：**

*   允许**并发读取**，因为这不会引起冲突。
*   确保**写入是独占的**，即当一个线程正在写入时，其他任何线程（无论是读还是写）都必须等待。

Dispatch Barrier 完美地契合了这一模型。我们可以将所有的读操作作为普通任务提交，而将所有的写操作作为栅栏任务提交。

#### 3. Barrier 的核心函数

GCD 提供了两个栅栏函数，它们都将一个 block 作为栅栏任务提交到队列中。

#### `dispatch_barrier_async`

这是最常用的栅栏函数。它**异步**地提交栅栏任务，并**立即返回**，不会阻塞当前线程。

```objectivec
void dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t barrier_block);
```

*   **`queue`**: 目标并发队列。
*   **`barrier_block`**: 要作为栅栏执行的代码块。

这是实现线程安全读写的首选，因为它不会因为等待写操作完成而阻塞调用线程（例如主线程），从而保持了应用的响应性。

#### `dispatch_barrier_sync`

这个函数**同步**地提交栅栏任务。它会**阻塞当前线程**，直到栅栏任务执行完毕并返回。

```objectivec
void dispatch_barrier_sync(dispatch_queue_t queue, dispatch_block_t barrier_block);
```

它的行为与 `dispatch_barrier_async` 类似，都保证了栅栏任务的独占执行。关键区别在于 `dispatch_barrier_sync` 会等待栅栏任务完成。这在某些场景下可能有用，例如，您需要立即知道写操作的结果才能继续执行后续代码，但它会牺牲调用线程的并发性。

#### 4. 代码实战：创建一个线程安全的 `NSMutableArray`

下面是一个经典的例子，演示如何使用 `dispatch_barrier_async` 来封装一个 `NSMutableArray`，使其可以被多个线程安全地读写。

```objectivec
#import <Foundation/Foundation.h>

@interface ThreadSafeArray : NSObject

// 公开的、不可变的数组属性，用于外部安全地读取数据快照
@property (nonatomic, strong, readonly) NSArray *array;
@property (nonatomic, readonly) NSUInteger count;

- (void)addObject:(id)object;
- (id)objectAtIndex:(NSUInteger)index;

@end

@implementation ThreadSafeArray {
    // 1. 一个自定义的并发队列
    dispatch_queue_t _concurrentQueue;
    // 2. 内部的可变数组，作为实际的数据存储
    NSMutableArray *_backingArray;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        // 创建一个并发队列，并使用反向DNS命名法进行标记，便于调试
        _concurrentQueue = dispatch_queue_create("com.example.threadSafeArray.queue", DISPATCH_QUEUE_CONCURRENT);
        _backingArray = [NSMutableArray array];
    }
    return self;
}

// 写操作：使用栅栏来确保独占访问
- (void)addObject:(id)object {
    // 使用异步栅栏，不会阻塞调用线程
    dispatch_barrier_async(_concurrentQueue, ^{
        // 这个 block 会在所有先前的读操作完成后执行
        // 在它执行期间，不会有任何其他 block (读或写) 执行
        [_backingArray addObject:object];
        NSLog(@"添加对象 '%@' 到线程 %@", object, [NSThread currentThread]);
    });
}

// 读操作：并发执行
- (id)objectAtIndex:(NSUInteger)index {
    __block id result;
    // 使用同步方式从队列中读取，以确保能立即返回结果
    dispatch_sync(_concurrentQueue, ^{
        // 这个 block 可以与其他读操作并发执行
        if (index < _backingArray.count) {
            result = _backingArray[index];
             NSLog(@"读取对象在线程 %@", [NSThread currentThread]);
        }
    });
    return result;
}

// 读操作：并发执行
- (NSUInteger)count {
    __block NSUInteger count;
    dispatch_sync(_concurrentQueue, ^{
        count = _backingArray.count;
    });
    return count;
}

// 读操作：并发执行
- (NSArray *)array {
    __block NSArray *snapshot;
    dispatch_sync(_concurrentQueue, ^{
        snapshot = [_backingArray copy]; // 返回一个不可变副本，防止外部修改
    });
    return snapshot;
}

@end
```

**如何使用这个类：**

```objectivec
// 在某个方法中
ThreadSafeArray *safeArray = [[ThreadSafeArray alloc] init];

// 在多个线程中同时操作
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

// 启动多个写入者
for (int i = 0; i < 5; i++) {
    dispatch_async(globalQueue, ^{
        [safeArray addObject:@(i)];
    });
}

// 启动多个读者
for (int i = 0; i < 5; i++) {
    dispatch_async(globalQueue, ^{
        NSUInteger count = safeArray.count;
        NSLog(@"当前数组数量: %lu", (unsigned long)count);
    });
}
```

即使在多线程环境下，由于栅栏的存在，对 `_backingArray` 的访问也是完全安全的。

## NSOperation

### 1. 什么是 `NSOperation`？

`NSOperation` 是一个**抽象类**，它代表一个独立的、可执行的工作单元。它本身不能直接使用，必须使用它的子类。`NSOperation` 的强大之处在于它将一个操作（任务）封装成了一个对象。

与直接使用 GCD 的 block 相比，将操作封装成对象带来了许多好处：

*   **依赖管理:** 可以轻松地在操作之间建立依赖关系，例如“操作B必须在操作A完成后才能开始”。
*   **状态监控:** 操作具有内置的状态机（如 `isReady`, `isExecuting`, `isFinished`, `isCancelled`），可以通过 KVO (Key-Value Observing) 来监控这些状态。
*   **取消操作:** 可以取消一个已经提交但尚未开始执行，或者正在执行的操作。
*   **优先级设置:** 可以为操作设置不同的执行优先级。
*   **可重用性:** 将复杂的业务逻辑封装在一个 `NSOperation` 子类中，可以在多处复用。

`NSOperation` 通常与 `NSOperationQueue`（操作队列）结合使用。`NSOperationQueue` 负责接收 `NSOperation` 对象，并根据队列的设置和操作的依赖关系，在适当的时机并发地执行它们。

### 2. `NSOperation` vs. GCD：如何选择？

| 特性           | Grand Central Dispatch (GCD)                                 | NSOperation / NSOperationQueue                               |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **抽象层级**   | 更轻量，更接近底层 C 语言 API                                | 更高级，面向对象的 Objective-C 接口                          |
| **核心**       | 提交代码块 (block) 到调度队列                                | 将操作 (operation) 添加到操作队列                            |
| **依赖关系**   | 难以直接实现，需要借助 `DispatchGroup` 或 `DispatchBarrier`  | **非常简单**，通过 `addDependency:` 方法直接设置             |
| **取消操作**   | 困难，只能通过 `DispatchWorkItem` 实现，且只能取消未开始的任务 | **简单**，可以直接调用 `cancel` 方法，并可在操作内部检查 `isCancelled` 状态 |
| **状态监控**   | 不支持 KVO                                                   | **完全支持 KVO**，可以监控 `isExecuting`, `isFinished` 等状态 |
| **最大并发数** | 不直接支持（需要通过信号量等方式间接实现）                   | **非常简单**，通过 `maxConcurrentOperationCount` 属性设置    |
| **性能**       | 通常更快，开销更小                                           | 因为是更高层级的抽象，会有一点额外的开销，但通常可忽略不计   |
| **适用场景**   | 简单的、一次性的后台任务，如更新 UI、简单的异步调用          | 复杂的、需要依赖关系、可取消、可重用的任务，或需要精确控制并发数量的场景 |

**简单总结：** 如果你只需要简单地将一个任务扔到后台异步执行，GCD 是最简单直接的选择。如果你需要对任务有更多的控制，比如建立复杂的执行顺序、随时取消任务、或者限制同时执行的任务数量，那么 `NSOperation` 是更好的选择。

### 3. `NSOperation` 的子类

苹果提供了两个可以直接使用的具体子类：

#### `NSBlockOperation`

这是最常用、最灵活的子类。它允许你使用一个或多个 block 来创建一个操作。

*   **单一 Block:**
    ```objectivec
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"这是一个NSBlockOperation任务，在线程 %@ 上执行", [NSThread currentThread]);
    }];
    
    // 启动操作 (可以直接启动，或添加到队列)
    // [operation start]; // 注意：直接调用 start 会在当前线程同步执行，除非是自定义的异步操作
    ```

*   **多个 Blocks:**
    `NSBlockOperation` 可以包含多个 block。当操作执行时，它会并发地执行所有的 block。只有当所有 block 都执行完毕后，整个 `NSBlockOperation` 才算完成。

    ```objectivec
    NSBlockOperation *multiBlockOp = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"这是初始的 block");
    }];
    
    // 添加额外的执行 block
    [multiBlockOp addExecutionBlock:^{
        NSLog(@"额外的 block 1");
    }];
    [multiBlockOp addExecutionBlock:^{
        NSLog(@"额外的 block 2");
    }];
    
    // 当 multiBlockOp 执行时，这3个block会并发执行
    ```

#### `NSInvocationOperation`

这个子类用于封装一个对目标对象的方法调用（invocation）。在现代 Objective-C 开发中，由于 block 的普及，它已经不那么常用了，但了解一下还是有必要的。

```objectivec
// 假设有一个方法 -(void)myTaskMethod:(id)data;
NSInvocationOperation *invocationOp = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(myTaskMethod:) object:data];
```

### 4. `NSOperationQueue`：操作的执行者

`NSOperationQueue` 是一个管理和执行 `NSOperation` 对象的队列。

*   **创建队列:**
    ```objectivec
    NSOperationQueue *myQueue = [[NSOperationQueue alloc] init];
    ```

*   **添加操作:**
    ```objectivec
    // 创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{ /* ... */ }];
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{ /* ... */ }];
    
    // 添加到队列
    [myQueue addOperation:op1];
    [myQueue addOperation:op2];
    
    // 也可以直接添加一个 block，队列会自动将其封装成 NSOperation
    [myQueue addOperationWithBlock:^{
        NSLog(@"这是一个直接添加的 block");
    }];
    ```

*   **主队列:**
    与 GCD 的主队列类似，`NSOperationQueue` 也有一个主队列，所有添加到主队列的操作都会在**主线程**上执行，非常适合用于更新 UI。
    ```objectivec
    NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];
    [mainQueue addOperationWithBlock:^{
        // 更新 UI
    }];
    ```

### 5. `NSOperation` 的核心特性详解

#### 依赖管理 (Dependencies)

这是 `NSOperation` 最强大的功能之一。你可以指定一个操作必须在另一个或多个操作完成后才能开始执行。

```objectivec
NSBlockOperation *opA = [NSBlockOperation blockOperationWithBlock:^{ NSLog(@"A 执行"); sleep(2); }];
NSBlockOperation *opB = [NSBlockOperation blockOperationWithBlock:^{ NSLog(@"B 执行"); }];
NSBlockOperation *opC = [NSBlockOperation blockOperationWithBlock:^{ NSLog(@"C 执行，在A和B之后"); }];

// 设置依赖：C 依赖于 A 和 B
[opC addDependency:opA];
[opC addDependency:opB];

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
// 将所有操作添加到队列中，顺序无所谓
// 队列会自动根据依赖关系来决定执行顺序
[queue addOperations:@[opA, opB, opC] waitUntilFinished:NO];

// 输出会是：
// A 执行
// B 执行 (A和B会并发执行)
// (等待A和B都完成后)
// C 执行
```
**注意：** 依赖关系不应形成循环（例如 A 依赖 B，B 依赖 A），否则会导致死锁。

#### 控制并发数

你可以非常简单地通过 `maxConcurrentOperationCount` 属性来控制一个队列能同时执行多少个操作。

```objectivec
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// 设置最大并发数为 1，使其变成一个串行队列
queue.maxConcurrentOperationCount = 1;

// 设置为 5，表示最多同时执行 5 个操作
// queue.maxConcurrentOperationCount = 5;

// 默认值是 NSOperationQueueDefaultMaxConcurrentOperationCount，表示由系统决定最佳并发数
```

#### 取消操作

你可以取消队列中的所有操作，或者取消单个操作。

```objectivec
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
NSBlockOperation *longRunningOp = [NSBlockOperation blockOperationWithBlock:^{
    for (int i = 0; i < 1000; i++) {
        // 在耗时操作内部，必须定期检查 isCancelled 状态
        if ([NSThread currentThread].isCancelled) { // 或者 self.isCancelled 在 NSOperation 子类中
            NSLog(@"操作被取消，退出循环");
            return;
        }
        NSLog(@"循环 %d", i);
        usleep(10000); // 模拟耗时
    }
}];

[queue addOperation:longRunningOp];

// 在某个时刻，决定取消
// [longRunningOp cancel]; // 取消单个操作
[queue cancelAllOperations]; // 取消队列里的所有操作
```
**重点:** 调用 `cancel` 只是将 `isCancelled` 属性设置为 `YES`。你必须在自己的代码中主动检查这个属性，并中断任务的执行，才能真正实现取消。

## `dispatch_semaphore_t`

好的，我们来非常详细地介绍一下 `dispatch_semaphore_t`（信号量）。它是 Grand Central Dispatch (GCD) 中一个极其强大且高效的同步工具。

理解信号量的最好方式是通过一个生动的比喻：**停车场车位管理系统**。

---

### 一、核心概念：什么是信号量？

`dispatch_semaphore_t` 本质上是一个**计数器**，用来控制对有限资源的访问。它遵循一个简单的原则：

*   当计数值 **大于 0** 时，允许访问资源。
*   当计数值 **等于 0** 时，任何试图访问资源的新请求都必须等待。

**停车场比喻：**

*   **信号量**：就是停车场入口处的电子显示牌。
*   **信号量的计数值**：就是显示牌上显示的“**剩余车位数**”。
*   **线程**：就是一辆辆想要进入停车场的汽车。

---

### 二、三大核心函数

信号量的所有操作都围绕三个核心函数展开：

#### 1. `dispatch_semaphore_create(long value)`

*   **作用**：创建一个新的信号量，并设置其**初始计数值**。
*   **参数 `value`**：初始计数值。这个值必须大于或等于0。
*   **比喻**：建造一个新停车场，并规定它总共有 `value` 个车位。电子显示牌的初始数字就是 `value`。

```objc
// 创建一个初始计数值为 5 的信号量
// 相当于停车场有 5 个空车位
dispatch_semaphore_t semaphore = dispatch_semaphore_create(5);
```

#### 2. `dispatch_semaphore_wait(semaphore, timeout)`

*   **作用**：尝试“获取”一个资源，可以理解为“请求进入”。这个函数会检查信号量的计数值。
    *   **如果计数值 > 0**：函数会立刻将计数值减 1，然后立即返回，表示“获取成功”。
    *   **如果计数值 == 0**：函数会**阻塞**当前线程，让线程进入休眠状态，直到计数值再次大于0，或者等待超时。
*   **参数 `semaphore`**：要操作的信号量。
*   **参数 `timeout`**：超时时间。通常使用：
    *   `DISPATCH_TIME_NOW`：不等待，如果计数值为0，立刻返回。
    *   `DISPATCH_TIME_FOREVER`：永远等待，直到计数值大于0。
*   **比喻**：一辆车开到停车场入口。
    *   **如果“剩余车位” > 0**：显示牌数字减 1，道闸打开，汽车进入。
    *   **如果“剩余车位” == 0**：汽车必须在入口外排队等待。司机（线程）在车里睡觉（阻塞）。

#### 3. `dispatch_semaphore_signal(semaphore)`

*   **作用**：释放一个资源，可以理解为“宣布离开”。这个函数会将信号量的计数值加 1。
*   **如果之前有线程因为计数值为0而被阻塞**，`signal` 会唤醒其中一个正在等待的线程。
*   **比喻**：一辆车从停车场驶出。
    *   显示牌上的“剩余车位”数字加 1。
    *   如果入口外有排队的汽车，停车场管理员会通知排在第一位的汽车，现在可以进入了。

---

### 三、三大经典使用场景

信号量的不同用法取决于你为它设定的初始值。

#### 场景一：用作互斥锁（保证线程安全）

这是最常见的用法之一，可以实现一个性能非常高的锁。

*   **如何实现**：创建一个初始计数值为 **1** 的信号量。
*   **原理**：
    1.  第一个线程调用 `wait`，计数值从 1 变为 0，线程继续执行。
    2.  此时若有第二个线程调用 `wait`，由于计数值为 0，第二个线程将被阻塞。
    3.  直到第一个线程执行完毕，调用 `signal`，计数值从 0 恢复为 1。
    4.  第二个线程被唤醒，`wait` 函数返回，计数值再次变为 0，第二个线程开始执行。
*   **比喻**：一个只有一个车位的 **VIP 停车场**。一次只能进一辆车。

*   **代码示例**：
    ```objc
    // 创建一个初始值为 1 的信号量，作为锁
    dispatch_semaphore_t lockSemaphore = dispatch_semaphore_create(1);
    NSMutableArray *sharedArray = [NSMutableArray array];
    
    // 在并发队列中模拟多线程写操作
    for (int i = 0; i < 100; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            // 等待信号量，相当于加锁
            // 如果计数值>0，则-1并继续；否则等待
            dispatch_semaphore_wait(lockSemaphore, DISPATCH_TIME_FOREVER);
            
            // --- 临界区代码开始 ---
            [sharedArray addObject:@(i)];
            NSLog(@"Added %d, Current Thread: %@", i, [NSThread currentThread]);
            // --- 临界区代码结束 ---
            
            // 释放信号量，相当于解锁
            // 计数值+1，若有等待的线程则唤醒一个
            dispatch_semaphore_signal(lockSemaphore);
        });
    }
    ```

#### 场景二：控制最大并发数

这是信号量非常强大的一个功能，可以精确控制同时执行任务的数量。

*   **如何实现**：创建一个初始计数值为 **N** 的信号量，N 就是你想要的最大并发数。
*   **原理**：循环异步派发大量任务到一个并发队列中，但在每个任务的开头调用 `wait`。这样，只有前 N 个任务可以成功通过 `wait`（将计数值从 N 减到 0），后续的任务都会被阻塞，直到有一个任务执行完毕并调用 `signal`，释放出一个“名额”。
*   **比喻**：一个有 **N 个车位**的公共停车场。最多只能同时停 N 辆车。

*   **代码示例**：
    ```objc
    // 创建一个初始值为 3 的信号量，表示最多允许 3 个任务并发执行
    dispatch_semaphore_t concurrencySemaphore = dispatch_semaphore_create(3);
    dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    // 模拟派发 10 个耗时任务
    for (int i = 0; i < 10; i++) {
        dispatch_async(concurrentQueue, ^{
            // 在任务开始时，请求一个“并发名额”
            // 如果并发数已满（计数值为0），则在此等待
            dispatch_semaphore_wait(concurrencySemaphore, DISPATCH_TIME_FOREVER);
            
            NSLog(@"Task %d is starting. Thread: %@", i, [NSThread currentThread]);
            sleep(2); // 模拟耗时操作
            NSLog(@"Task %d is finished.", i);
            
            // 任务结束，释放“并发名额”
            dispatch_semaphore_signal(concurrencySemaphore);
        });
    }
    ```
    你会发现，日志总是3个任务一组地开始和结束。

## 引用计数放在内存的哪个地方

这是一个非常深入且极好的问题，它触及了 Objective-C 运行时 (Runtime) 的核心实现。答案比“在某个整数里”要复杂得多，并且随着苹果的优化，其存储方式也一直在演进。

简单来说，一个对象的引用计数**存储在两个主要地方**：

1.  **直接在对象内部（通过 `isa` 指针）**：这是为了速度而优化的“快速路径”(Fast Path)。
2.  **在一个全局的、独立的“边表” (Side Table) 中**：当对象内部存不下时，使用的“慢速路径”(Slow Path)。

下面我们来详细拆解这个过程。

---

### 背景：对象内存布局

一个标准的 Objective-C 对象（在堆上分配的）在内存中的起始位置是一个名为 `isa` 的指针。

```c
struct objc_object {
    isa_t isa;
    // ... 其他成员变量 ...
};
```

在早期的 Objective-C 中，`isa` 就是一个单纯的指针，指向该对象所属的类 (`Class`)。但在 64 位架构下，一个指针有 64 位，而内存地址远用不了这么多位，这为苹果的优化留下了巨大空间。

于是，苹果引入了 **"non-pointer isa"**（非指针 `isa`）的概念。`isa` 不再是一个简单的指针，而是一个包含了多种信息的**位域 (bitfield)** 集合。

---

### 1. 快速路径：存储在 `isa` 指针中 (`extra_rc`)

在现代 64 位运行时中，`isa_t` 是一个联合体 (union)，它既可以是一个指向类的指针，也可以是一个包含多个标志位的结构体。

```c
union isa_t {
    Class cls; // 指向类的指针
    uintptr_t bits; // 64位整数
    struct { // 这是一个简化的位域结构，以 arm64 为例
        uintptr_t nonpointer        : 1;  // 0: 是纯指针; 1: 是包含额外信息的位域
        uintptr_t has_assoc         : 1;  // 是否有关联对象
        uintptr_t has_cxx_dtor      : 1;  // 是否有 C++ 析构函数
        uintptr_t shiftcls          : 33; // 【核心】类的地址 (Class address)
        uintptr_t magic             : 6;  // 用于调试，验证是否是 isa_t
        uintptr_t weakly_referenced : 1;  // 是否被弱引用指向
        uintptr_t deallocating      : 1;  // 是否正在 dealloc
        uintptr_t has_sidetable_rc  : 1;  // 【关键】引用计数是否已存入 Side Table
        uintptr_t extra_rc          : 19; // 【关键】额外的引用计数值 (reference count)
    }
};
```

**关键点：**

*   **`nonpointer`**: 这个位决定了 `isa` 的类型。对于绝大多数对象，它为 1。
*   **`extra_rc`**: 这就是**引用计数的直接存储区**。在 arm64 架构下，它有 19 位。
*   **`has_sidetable_rc`**: 这个标志位用来指示引用计数是否已经“溢出”并存储到了 Side Table 中。

**`retain` 操作的流程（快速路径）：**

1.  当一个对象被 `retain` 时，运行时首先检查其 `isa`。
2.  如果 `has_sidetable_rc` 位是 `0`，说明引用计数还在 `isa` 内部。
3.  运行时会原子地 (atomically) 将 `extra_rc` 的值加 1。
4.  因为这是直接在对象内存上进行的操作，没有额外的函数调用和查表，所以速度极快。

**`extra_rc` 的局限性：** 19 位所能表示的最大数字是 `2^19 - 1 = 524287`。虽然对于绝大多数对象来说已经足够，但如果一个对象的引用计数超过了这个值，`isa` 就存不下了。这时就需要“慢速路径”。

---

### 2. 慢速路径：存储在边表 (Side Table) 中

当 `extra_rc` 存满时，或者当一个对象第一次被弱引用 (`weak`) 指向时，运行时会求助于一个全局的数据结构——**Side Tables**。

`Side Tables` 是一个复杂的哈希表结构，它将对象的地址与其额外的状态信息（包括完整的引用计数和弱引用列表）关联起来。

```c
// SideTable 的简化结构
struct SideTable {
    spinlock_t slock; // 用于保证线程安全的自旋锁
    RefcountMap refcnts; // 存储完整引用计数的哈希表
    weak_table_t weak_table; // 存储弱引用信息的表
};

// 全局有一个 SideTable 的数组，以减少多线程下的锁竞争
static StripedMap<SideTable> SideTables;
```

**`retain` 操作的流程（慢速路径）：**

1.  当 `extra_rc` 即将溢出时（达到最大值），运行时会执行以下操作：
    a. 从 `extra_rc` 中取出当前的计数值（例如 524287）。
    b. 找到对象地址对应的 `SideTable`。
    c. 在 `SideTable` 的 `refcnts` 哈希表中，以**对象地址为 key**，存储一个**完整的引用计数值** (e.g., `524287 + 1`)。
    d. 将对象 `isa` 中的 `extra_rc` 清零，并**将 `has_sidetable_rc` 标志位置为 `1`**。

2.  此后，每一次对该对象进行 `retain` 或 `release` 操作：
    a. 运行时检查到 `has_sidetable_rc` 为 `1`。
    b. 它会直接忽略 `extra_rc`，转而去查找全局的 `SideTable`。
    c. 在 `SideTable` 中找到该对象的引用计数值，并对其进行增减。
    d. 这个过程涉及查哈希表和加锁，因此比直接操作 `isa` 要慢。

#### Side Table 与弱引用 (`weak`)

`SideTable` 的另一个重要职责是管理弱引用。

当一个对象被 `weak` 指针指向时，运行时会在 `weak_table` 中记录下“哪个对象 (`key`) 被哪些 `weak` 指针 (`value`, 指针的地址列表) 指向了”。

当这个对象的引用计数变为 0 并开始 `dealloc` 时，运行时会：
1.  根据对象地址去 `SideTable` 的 `weak_table` 中查找。
2.  遍历所有指向它的 `weak` 指针。
3.  将这些 `weak` 指针的值自动设置为 `nil`。

这就是 `weak` 能在对象释放后自动变 `nil` 的底层原理。

---

### 特殊情况：Tagged Pointer

对于一些小对象，比如特定数值的 `NSNumber`、短的 `NSString` 等，苹果使用了 **Tagged Pointer** 技术。

*   **原理：** 不在堆上分配内存，而是将对象的数据**直接编码存储在指针自身的值里**。
*   **判断方法：** 对于 64 位架构，如果一个指针的最低有效位是 1 (macOS) 或最高有效位是 1 (iOS)，那么它就是一个 Tagged Pointer。
*   **引用计数：** **Tagged Pointer 对象没有引用计数**。它们不存储在堆上，其生命周期管理方式更像栈上的基本数据类型。对它进行 `retain` 和 `release` 操作实际上是空操作 (no-op)。当你把它赋值给另一个变量时，仅仅是指针值（也就是数据本身）的拷贝。

## 从32b到64b的指针，苹果做了什么

这是一个绝佳的问题，它揭示了苹果在软件工程和系统优化方面的深厚功力。从 32 位到 64 位的迁移，苹果所做的**远不止是把指针从 4 字节扩展到 8 字节**那么简单。他们将这次架构升级视为一个巨大的契机，对 Objective-C 的底层运行时（Runtime）进行了根本性的重塑，从而带来了惊人的性能提升和内存效率优化。

核心思想是：**一个 64 位的指针拥有巨大的地址空间（2^64），但实际上当前硬件和操作系统远用不了这么多地址位。那么，这些“多余”的、未被使用的比特位就是宝贵的资源，可以用来存储额外的信息。**

苹果主要做了两大革命性的创新：

1.  **Tagged Pointer（标签指针）**
2.  **Non-Pointer ISA（非指针 ISA）**

---

### 1. 创新一：Tagged Pointer —— 指针即数据

**【问题】**
在 32 位时代，即使是存储一个很小的整数，比如 `[NSNumber numberWithInt:5]`，系统也需要在堆（Heap）上分配一块内存来创建一个完整的 `NSNumber` 对象，然后返回一个指向这块内存的指针。这个过程涉及内存分配、引用计数管理（`retain`/`release`），开销相对较大，并且会产生大量的小内存碎片。

**【64位解决方案：Tagged Pointer】**
苹果工程师意识到，对于像 `NSNumber`、`NSDate`、短 `NSString` 这样的小对象，它们的数据本身可能比一个 64 位指针还要小。何必再费力去分配内存呢？

**做法是：**

1.  **标记 (Tagging):** 他们设定了一个规则，用指针值的某一个（或几个）比特位作为“标签”，来表明“我不是一个指向内存地址的指针，我本身就包含了数据”。（例如，在 iOS arm64 架构中，如果一个指针的最高位是 1，它就是一个 Tagged Pointer）。
2.  **载荷 (Payload):** 指针中除了标签位之外的剩余比特位，就用来直接存储对象的数据。比如 `NSNumber` 的数值，或者 `NSString` 的字符编码。

**带来的巨大优势：**

*   **无需内存分配：** 创建对象时，直接将数据编码成一个指针格式的整数即可，速度极快。
*   **无引用计数：** Tagged Pointer 对象不参与 ARC 的引用计数。对它发送 `retain` 或 `release` 消息，实际上是空操作。它的生命周期由其值本身决定，而不是由堆内存管理。
*   **内存效率高：** 大大减少了堆上的小对象碎片，提升了内存使用效率和CPU缓存命中率。
*   **速度更快：** 读取数据时，也无需解引用指针去访问内存，直接从指针值中解码即可。

**举例：**
`NSNumber *num1 = @1;` 在 64 位系统上，`num1` 这个“指针”的值本身就包含了数字 1 的信息，它并不指向任何堆上的内存块。

---

### 2. 创新二：Non-Pointer ISA —— 指针即仪表盘

**【问题】**
在 32 位时代，每个对象的第一个成员变量 `isa` 是一个纯粹的指针，仅仅指向该对象所属的 `Class`（类对象）。而对象的其他状态，比如引用计数，则需要额外的存储空间（或者像之前讨论的，存储在全局的 Side Table 中）。

**【64位解决方案：Non-Pointer ISA】**
苹果工程师再次利用了 64 位指针的“奢侈空间”。他们将 `isa` 指针从一个单纯的“地址指针”变成了一个包含丰富信息的“位域集合体 (Bitfield)”。

**做法是：**

`isa` 指针的 64 个比特位被精细地划分，分别用来存储不同的信息，就像一个飞机的仪表盘。

```c
// 简化的 arm64 架构下的 isa 位域结构
struct {
    uintptr_t nonpointer        : 1;  // 标志位：表明这是个 Non-Pointer ISA
    uintptr_t has_assoc         : 1;  // 是否有关联对象
    uintptr_t has_cxx_dtor      : 1;  // 是否有 C++ 析构函数
    uintptr_t shiftcls          : 33; // 【核心】类的地址 (被压缩存储)
    uintptr_t magic             : 6;  // 调试用的魔法数
    uintptr_t weakly_referenced : 1;  // 是否被弱引用指向
    uintptr_t deallocating      : 1;  // 是否正在 dealloc
    uintptr_t has_sidetable_rc  : 1;  // 引用计数是否已“溢出”到 Side Table
    uintptr_t extra_rc          : 19; // 【核心】直接存储的引用计数值
};
```

**带来的巨大优势：**

*   **极速的引用计数：**
    *   **快速路径：** 对于绝大多数对象，其引用计数（`extra_rc`，19位足够表示50多万次引用）可以直接在 `isa` 内部通过原子操作进行增减。这比去查找并锁定一个全局的哈希表（Side Table）要快几个数量级。
    *   **慢速路径：** 只有当 `extra_rc` 溢出，或者对象有了弱引用时，才会启用 `has_sidetable_rc` 标志，并将完整的引用计数存入 Side Table。这保证了常见情况下的极致性能。
*   **对象状态信息聚合：** 无需额外查询，通过 `isa` 就能快速判断一个对象是否有弱引用、是否有关联对象等，加快了运行时的判断速度。
*   **节省内存：** 对于没有溢出的对象，它的引用计数信息是“免费”存储在 `isa` 指针里的，不需要额外的内存。

## nsstring生命周期的管理

---

### 策略一：编译时常量 (`__NSCFConstantString`)

这种字符串的生命周期管理最为简单。

*   **它们是什么？**
    所有使用 `@"..."` 字面量语法在代码中直接写死的字符串。
    ```objectivec
    NSString *myString = @"Hello, World!";
    ```

*   **存储在哪里？**
    它们不存储在堆（Heap）上，也不在栈（Stack）上。它们被存储在应用程序二进制文件的**数据段 (`__TEXT,__cstring` 或 `__DATA,__cfstring`)** 中。这意味着，当你的 App 启动时，这个字符串的内存就已经被加载并存在了，直到 App 终止。

*   **生命周期管理：**
    *   **无引用计数：** 对这种字符串发送 `retain`, `release`, `autorelease` 消息，实际上都是**空操作 (no-op)**。它们不会被释放。
    *   **生命周期：** 与应用程序的生命周期完全相同。它们是“永生”的。
    *   **内存地址：** 每次运行程序，这个字符串对象的内存地址都是固定的（在 ASLR 机制下，基地址会变，但相对位置不变）。

---

### 策略二：标签指针 (`NSTaggedPointerString`)

这是苹果在 64 位架构下引入的重大优化。

*   **它们是什么？**
    在运行时动态创建的、**内容较短**的字符串。其内容可以被直接编码进 64 位的指针自身。

*   **存储在哪里？**
    **无处存储！** 这就是关键。它**不占用任何堆内存**。指针本身就是数据。运行时通过检查指针值的特定标记位（例如，最高位是否为1）来识别它是一个 Tagged Pointer。

*   **生命周期管理：**
    *   **无引用计数：** 和常量字符串一样，它们也不参与引用计数。`retain` 和 `release` 都是空操作。
    *   **生命周期：** 它的生命周期由持有该“指针值”的变量决定。当变量超出作用域时，这个“值”就消失了。没有 `dealloc` 的过程。
    *   **本质：** 它更像一个 `int` 或 `double` 这样的值类型，而不是一个对象。

*   **如何验证？**
    ```objectivec
    NSString *dynamicShortString = [NSString stringWithFormat:@"abc%d", 123]; // 创建一个短字符串
    NSLog(@"Class: %@", [dynamicShortString class]); // 输出: NSTaggedPointerString
    ```
    **注意：** 只有当字符串足够短，并且由特定API（如 `stringWithFormat:`）创建时，才有可能成为 Tagged Pointer。如果字符串很长，系统会自动切换到下一个策略。

## uiview怎么渲染到屏幕上

好的，这个问题问得非常棒！理解`UIView`的渲染流程是掌握iOS图形和性能优化的基石。这趟旅程涉及多个系统框架，从您写的代码一直到屏幕上发光的物理像素。

我们把它想象成一个“**从设计蓝图到建成大楼**”的过程。

*   **设计蓝图**：您在代码中创建和布局的`UIView`对象。
*   **大楼**：您在屏幕上最终看到的像素。

下面就是这个过程的详细步骤，我们会从最基础的层面讲起。

---

### 第一步：蓝图的绘制 (`UIView` 和 `CALayer`)

首先，您需要知道一个最重要的概念：**`UIView`本身不直接负责在屏幕上画任何东西。**

`UIView`更像是一个“项目经理”。它负责：
*   **管理内容**：知道自己的背景色、尺寸、位置。
*   **处理事件**：响应用户的点击、拖拽等手势。
*   **维护层级关系**：知道自己有哪些子视图（subviews）。

真正负责“视觉表现”的是`UIView`背后一个叫做**`CALayer`**（核心动画层）的对象。每个`UIView`都拥有一个`CALayer`。您可以把它看作是`UIView`的“视觉内容板”。

当你写下这行代码时：
```objectivec
myView.backgroundColor = [UIColor redColor];
myView.frame = CGRectMake(10, 10, 100, 100);
```
`UIView`实际上是在对它自己的`layer`说：“嘿，`CALayer`，把你的背景色设置为红色，并且你的尺寸和位置是(10, 10, 100, 100)。”

所以，**App的UI实际上是一个由`CALayer`构成的层级树（Layer Tree）**，而不是`UIView`树。`UIView`只是这个树的友好接口和管理者。

### 第二步：提交蓝图 (Commit Transaction)

您的App通过代码对`UIView`（也就是`CALayer`）进行了很多修改，比如改变颜色、移动位置、添加子视图等。但这些改动不会立刻导致屏幕刷新。如果每改一个属性就刷新一次屏幕，效率会极其低下。

取而代之，系统采用了一个**“批量处理”**的机制。

1.  **RunLoop（运行循环）**：iOS的App主线程一直处在一个叫做`RunLoop`的循环中。它不断地等待事件（如触摸、定时器），处理事件，然后再次等待。
2.  **标记为“脏”**：当你修改一个`CALayer`的属性时，它会被系统内部标记为“脏”（dirty），意思就是“这个图层需要被重绘”。
3.  **提交事务**：在一个`RunLoop`周期的末尾，系统会检查所有被标记为“脏”的图层。它会把所有这些改动（比如新的背景色、新的位置等）打包成一个“事务（Transaction）”，然后通过IPC（进程间通信）发送给一个独立的进程。

这个独立的进程叫做**Render Server (渲染服务)**，在iOS中它通常是`backboardd`。

**关键点**：从这一刻起，渲染的工作就**脱离了你的App进程**，交给了系统级的渲染服务。这就是为什么即使你的App主线程偶尔卡顿一下，系统动画（比如切换App）有时仍然能保持流畅的原因。

### 第三步：渲染服务解析蓝图 (Decode & Draw)

Render Server收到了你App发来的“图层树”和所有改动信息。现在它需要把这些抽象的描述信息，变成GPU能理解的**位图（Bitmap）**。

这里有两种情况：

*   **情况A：简单的图层内容**
    *   如果一个`CALayer`的内容很简单，比如只是一个背景色，或者它的`contents`属性被直接设置为一张图片（`CGImageRef`）。
    *   这种情况非常高效。Render Server可以直接把颜色信息或者已经解码的图片数据准备好，等待交给GPU。

*   **情况B：需要自定义绘制的图层 (CPU绘图)**
    *   如果你创建了一个`UIView`的子类，并重写了它的`draw(_:)`方法（在Objective-C中是`drawRect:`）。
    *   当这个图层需要更新它的内容时，Render Server会要求你的App执行`drawRect:`方法。
    *   在这个方法里，你通常会使用**Core Graphics**框架（比如`UIGraphicsGetCurrentContext()`）来在CPU上进行绘制，画线、画圆、填充颜色等。
    *   你画完后，结果会被存入一块内存中的位图。然后这个位图会被传递给Render Server。
    *   **这是一个非常昂贵的操作！** 因为它涉及CPU的计算，并且需要在App进程和Render Server进程之间传递一块很大的位图数据。**这也是为什么苹果官方强烈建议我们尽量避免重写`drawRect:`的原因。**

### 第四步：GPU合成大楼 (Compositing)

现在，Render Server已经把所有图层都准备成了位图（在GPU中称为**纹理 Texture**）。

接下来是**GPU（图形处理单元）**大显身手的时候了。GPU最擅长的工作就是**合成（Compositing）**。

你可以把每一个`CALayer`的位图想象成一张**透明的玻璃板**，上面画着内容。GPU的工作就像把这些玻璃板按照正确的顺序、位置、透明度、旋转角度等属性，一层一层地叠起来，最终“压”成一张完整的图片。

这个过程在GPU上是并行处理的，速度极快。

### 第五步：显示在屏幕上 (Display)

1.  **帧缓冲区 (Framebuffer)**：GPU将合成好的最终图片，放入一块名为**帧缓冲区**的特殊内存区域。这块内存里的数据，直接对应着屏幕上即将显示的每一个像素的颜色。

2.  **双缓冲机制 (Double Buffering)**：为了防止用户看到正在绘制中的、不完整的画面（即“屏幕撕裂”），系统使用了双缓冲技术。
    *   屏幕当前正在显示的是**前台缓冲区（Front Buffer）**。
    *   GPU在后台默默地将新的一帧画面绘制到**后台缓冲区（Back Buffer）**。
    *   当GPU画完一整帧后，系统会等待一个时机。

3.  **垂直同步 (VSync)**：这个时机就是屏幕的**垂直同步信号**。你的iPhone屏幕以固定的频率刷新（比如60Hz或120Hz）。每当屏幕准备好刷新下一帧时，就会发出一个VSync信号。
    *   收到信号后，系统会**瞬间交换**前后台缓冲区的指针。后台缓冲区变成新的前台缓冲区，开始显示在屏幕上；而旧的前台缓冲区则变成后台缓冲区，等待GPU绘制下一帧。

至此，从`UIView`的属性修改，到最终像素显示在屏幕上的完整旅程就结束了。

## app从点击屏幕到完成渲染，中间发生了什么

好的，这是一个非常宏大且深入的问题，它几乎串联起了iOS系统中从用户交互到图形渲染的整条核心链路。我们来把这个复杂的过程分解成一条清晰、连贯的故事线。

这个故事可以分为两大幕：

*   **第一幕：响应与决策 (在你的App进程内，CPU主导)**
    *   从手指触摸屏幕，到你的App决定需要更新UI。
*   **第二幕：渲染与显示 (系统服务与硬件主导)**
    *   从App提交UI更新请求，到像素最终在屏幕上发光。

---

### **第一幕：响应与决策 (App进程内)**

这个阶段的目标是：**将物理触摸，转换为UI更新的“意图”**。

#### **1. 硬件事件的捕获**

1.  **触摸发生**：你的手指接触到屏幕的电容触摸屏。
2.  **硬件上报**：触摸屏固件检测到位置、压力等信息，将其转换为硬件信号。
3.  **I/O Kit驱动**：操作系统内核的I/O Kit驱动程序接收到这些硬件信号，并将其打包成一个**IOHIDEvent (I/O硬件事件)**。

#### **2. 系统进程的分发**

1.  **SpringBoard接收**：这个`IOHIDEvent`首先被发送到系统的桌面管理器进程——**SpringBoard**。SpringBoard是所有事件的第一个接收者。
2.  **判断事件归属**：SpringBoard会判断这个触摸事件发生在哪个App的窗口区域内。它确定是你的App后，会将这个事件通过**IPC (进程间通信，底层是Mach端口)** 转发给你的App进程。

#### **3. App内部的事件响应**

1.  **主线程的RunLoop**：你的App主线程有一个**RunLoop (运行循环)**，它一直在等待新的事件源。当收到SpringBoard转发来的触摸事件后，RunLoop被唤醒。
2.  **事件队列与`UIApplication`**：`UIApplication`的单例对象接收到事件，并将其放入一个事件队列中。
3.  **寻找第一响应者 (Hit-Testing)**：这是关键的一步。`UIWindow`会从自己开始，**从后往前**遍历它的子视图层级，询问每个视图：“这个触摸点(`CGPoint`)在你的边界(`bounds`)之内吗？”
    *   这个过程是**递归**的。如果一个父视图包含该点，它会继续询问它的子视图。
    *   最终，它会找到层级最深、又能响应事件的那个视图（比如一个`UIButton`），这个视图就是这次触摸事件的**“第一响应者”(First Responder)**。
4.  **响应者链 (Responder Chain)**：事件被传递给第一响应者。如果这个视图自己不处理这个事件（或者处理完后想让父级也知道），它可以将事件沿着**响应者链**向上传递（从子视图 -> 父视图 -> ... -> UIViewController -> UIWindow -> UIApplication）。

#### **4. 业务逻辑与UI更新**

1.  **执行代码**：假设第一响应者是一个按钮，它的`touchUpInside`事件被触发了。你在这个按钮关联的`@IBAction`方法里的代码现在开始执行。
2.  **修改模型/视图**：在你的代码里，你可能会做一些事情，比如：
    ```objectivec
    // 假设这是按钮的点击方法
    - (void)onMyButtonClick:(UIButton *)sender {
        // 1. 改变一个UIView的背景色
        self.myTargetView.backgroundColor = [UIColor blueColor];
        
        // 2. 改变一个CALayer的边框宽度
        self.myTargetView.layer.borderWidth = 2.0;
        
        // 3. 让一个视图执行动画
        [UIView animateWithDuration:0.3 animations:^{
            self.anotherView.alpha = 0.0;
        }];
    }
    ```3.  **标记为“脏”**：当你修改`myTargetView`的背景色或`layer`的属性时，`Core Animation`框架并不会立刻去重绘。它只是在内部将`myTargetView`的`CALayer`标记为**“脏”(dirty)**，意思是“这个图层需要被更新”。

#### **5. 提交“设计蓝图”**

1.  **RunLoop即将休眠**：当你的点击事件处理代码执行完毕后，RunLoop的一个周期也即将结束。
2.  **`CATransaction`提交**：在RunLoop进入休眠状态之前，`Core Animation`会执行一个非常重要的操作：它会收集所有被标记为“脏”的`CALayer`以及它们所有的属性变更，将这些信息打包成一个**事务（CATransaction）**。
3.  **发送给Render Server**：这个事务被打包后，通过IPC（进程间通信）发送给系统的**渲染服务进程 (Render Server)**。

**至此，第一幕结束。你的App已经将“需要把A视图背景改成蓝色，B视图透明度变成0”这个明确的“设计蓝图”交给了系统。**

---

### **第二幕：渲染与显示 (系统服务与硬件)**

这个阶段的目标是：**将“设计蓝图”变成屏幕上发光的像素**。

#### **6. Render Server的准备工作**

1.  **接收并解码**：Render Server进程接收到来自你App的图层树和属性变更信息。如果某个图层的内容是一张被压缩的图片（如JPEG），它需要先**解码(Decode)**成原始的**位图(Bitmap)**数据。
2.  **（可选）CPU重绘**：如果某个`CALayer`需要通过`drawRect:`方法来更新内容，Render Server会通过IPC回调你的App，让App在CPU上执行`drawRect:`，然后把生成的位图传回来。这是一个昂贵的往返操作。
3.  **上传GPU**：Render Server将所有图层的内容（现在都是位图格式了）和绘制指令，上传到**GPU（图形处理器）**的显存中。在GPU里，位图被称为**纹理(Texture)**。

#### **7. GPU的魔法时刻**

1.  **顶点着色器 (Vertex Shader)**：GPU首先会根据`CALayer`的位置、大小、形变等信息，计算出每个图层矩形的顶点在屏幕上的最终坐标。
2.  **片元/像素着色器 (Fragment/Pixel Shader)**：然后，GPU会对图层内的每一个像素进行计算，决定这个像素最终的颜色（比如应用透明度、混合颜色等）。
3.  **合成 (Compositing)**：GPU将所有图层的纹理，按照`CALayer`树定义的正确顺序、位置和透明度，一层层地叠加在一起，形成一幅完整的屏幕画面。这个过程在GPU上是高度并行的，速度极快。
4.  **渲染到后台缓冲区**：合成好的最终画面，被绘制到**帧缓冲区(Framebuffer)**的一个**后台缓冲区(Back Buffer)**中。

#### **8. 最终的显示**

1.  **等待垂直同步 (VSync)**：GPU画完一整帧后，系统会等待屏幕硬件发出的**垂直同步信号**（通常每秒60次或120次）。
2.  **缓冲区交换 (Buffer Swap)**：收到VSync信号后，系统会瞬间交换前后台缓冲区的角色。后台缓冲区变成新的**前台缓冲区(Front Buffer)**，它的内容被视频控制器读取，并最终显示在屏幕上。
3.  **点亮像素**：屏幕上的物理像素点，根据前台缓冲区的数据，发出对应颜色的光。

**至此，第二幕结束。从你手指的一次点击，到屏幕上颜色的变化，一次完整的渲染周期宣告完成。**

## layoutSubViews什么时候会调用，他是干什么的

好的，这是一个在 iOS 开发中至关重要的方法。我们来详细拆解一下 `layoutSubviews` 是做什么的，以及它会在什么时候被调用。

### 他是干什么的？（What it does?）

`layoutSubviews` 是 `UIView` 类中的一个核心方法。它的**根本目的**是：**为当前视图（`self`）的子视图（`subviews`）计算并设置正确的尺寸（size）和位置（position）。**

简单来说，`layoutSubviews` 就是一个专为你编写**手动布局代码**而准备的“地方”。

当你创建一个自定义 `UIView`，并且不使用或部分不使用 Auto Layout，而是想通过代码精确控制其内部子视图的 `frame`（或 `bounds` 和 `center`）时，你就应该重写（override）这个方法。

**核心职责：**

1.  **布局计算**：在该方法内部，你可以获取当前视图的 `bounds`（这是进行布局计算最可靠的尺寸依据），然后根据你的设计逻辑，计算出每个子视图应该有的 `frame`。
2.  **应用布局**：将计算出的 `frame` 赋值给相应的子视图。

**一个至关重要的规则：**

**你几乎永远不应该直接调用 `[self layoutSubviews];`。** 这是一个由系统在特定时机调用的“回调”方法。如果你需要触发布局更新，应该使用下面会提到的 `setNeedsLayout` 或 `layoutIfNeeded`。

---

### 什么时候会调用？（When is it called?）

系统会在认为一个视图的布局“失效”或“需要更新”时，自动调用 `layoutSubviews` 方法。以下是触发调用的最常见时机：

1.  **初始化与添加**：
    *   当一个视图第一次被添加到父视图上时（例如调用 `[parentView addSubview:childView];`），它的 `layoutSubviews` 会被触发。

2.  **视图尺寸改变**：
    *   直接修改一个视图的 `frame` 或 `bounds` 的 `size` 部分时，该视图的 `layoutSubviews` 会被调用。这也是最常见的触发方式。

3.  **设备旋转**：
    *   当设备发生旋转时，屏幕主窗口（`UIWindow`）的尺寸会发生变化，这个变化会沿着视图层级树向下传递，导致各级视图的 `layoutSubviews` 被触发以适应新尺寸。

4.  **滚动 `UIScrollView`**：
    *   当用户滚动一个 `UIScrollView`（或其子类如 `UITableView`, `UICollectionView`）时，`UIScrollView` 本身的 `bounds` 的原点（`origin`）会发生变化。这会触发 `UIScrollView` 自己的 `layoutSubviews` 方法，以便它可以重新排列其内部的瓦片视图（tiled views）等内容。

5.  **Auto Layout 约束变化**：
    *   当你使用 Auto Layout 时，如果一个约束被更新、激活或停用，导致视图的尺寸或位置可能发生变化，Auto Layout 引擎会在计算完成后，最终通过调用 `layoutSubviews` 来应用这些变化。在这种情况下，你通常不需要重写它，除非你要混合使用 Auto Layout 和手动布局。

6.  **手动触发**：
    这也是我们作为开发者最关心的部分。我们有两种方式可以“请求”系统调用 `layoutSubviews`：

    *   **`[view setNeedsLayout];`**
        *   **作用**：给视图打上一个“需要重新布局”的标记（sets a flag）。它告诉系统：“这个视图的布局已经失效了，请在下一个 UI 更新周期（update cycle）中方便的时候调用我的 `layoutSubviews`”。
        *   **特点**：**异步调用**。该方法会立即返回，不会马上执行布局。系统会将多个布局请求合并，在一次更新周期中统一处理，效率很高。这是**最常用、最推荐**的触发方式。

    *   **`[view layoutIfNeeded];`**
        *   **作用**：如果视图已经被标记为“需要重新布局”，则立即强制系统执行布局过程，即**同步调用** `layoutSubviews`。
        *   **特点**：**同步调用**。该方法会阻塞当前线程，直到布局完成才返回。
        *   **使用场景**：通常用在需要立即获取布局更新后视图 `frame` 的地方。一个经典的例子是：你改变了一个约束，然后想基于这个变化后的新 `frame` 执行一个动画。你需要在动画 block 之前调用 `layoutIfNeeded`，以确保动画从正确的起始 `frame` 开始。

### 示例代码与最佳实践

假设我们有一个自定义的 `ProfileView`，里面包含一个头像 `UIImageView` 和一个名字 `UILabel`。

```objc
#import "ProfileView.h"

@interface ProfileView ()
@property (nonatomic, strong) UIImageView *avatarImageView;
@property (nonatomic, strong) UILabel *nameLabel;
@end

@implementation ProfileView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        // 1. 创建并添加子视图
        _avatarImageView = [[UIImageView alloc] init];
        _avatarImageView.backgroundColor = [UIColor lightGrayColor]; // 占位颜色
        [self addSubview:_avatarImageView];
        
        _nameLabel = [[UILabel alloc] init];
        _nameLabel.text = @"Username";
        _nameLabel.textAlignment = NSTextAlignmentCenter;
        [self addSubview:_nameLabel];
    }
    return self;
}

// 2. 这是实现布局逻辑的地方！
- (void)layoutSubviews {
    // 最佳实践：总是先调用父类的实现
    [super layoutSubviews];
    
    // 使用当前视图的 bounds 来进行计算，而不是 frame
    CGRect bounds = self.bounds;
    
    CGFloat avatarSize = bounds.size.height * 0.7;
    CGFloat padding = 10.0;
    
    // 计算头像的位置和尺寸
    self.avatarImageView.frame = CGRectMake(
        (bounds.size.width - avatarSize) / 2.0, // 水平居中
        0,
        avatarSize,
        avatarSize
    );
    
    // 计算标签的位置和尺寸
    self.nameLabel.frame = CGRectMake(
        0,
        CGRectGetMaxY(self.avatarImageView.frame) + padding, // 在头像下方
        bounds.size.width,
        20
    );
}

// 如果你想在某个时刻主动更新布局
- (void)updateName:(NSString *)newName {
    self.nameLabel.text = newName;
    
    // 假设更新名字后需要调整布局（虽然这里不需要，但作为例子）
    // 你会调用 setNeedsLayout, 而不是直接调用 layoutSubviews
    [self setNeedsLayout];
}

@end
```

## 那么masonry的约束应该写在哪里比较合适

## iOS的各种锁

好的，iOS 中的锁是多线程编程中保证数据安全、避免竞态条件的核心工具。在 Objective-C 环境下，我们有多种锁可供选择，它们在性能、使用场景和实现原理上各有不同。

下面我将详细介绍 iOS 开发中常见的各种锁，并分析它们的原理、使用场景和优缺点。

### 锁的分类

从大方向上，我们可以将锁分为两类：

1.  **互斥锁 (Mutex)**：同一时间只允许一个线程访问被保护的资源。当一个线程持有锁时，其他试图获取该锁的线程会被阻塞，直到锁被释放。这是最常见的锁类型。
2.  **自旋锁 (Spinlock)**：当一个线程试图获取锁但锁已被占用时，该线程不会被阻塞（即进入休眠状态），而是在一个循环中“自旋”，不断地检查锁是否已释放。

此外，还有一些更广义的同步机制，如信号量，也能起到锁的作用。

---

### iOS 中常见的锁 (Objective-C)

#### 1. `@synchronized`

这是 Objective-C 提供的一个便捷的互斥锁指令，也是最容易使用的锁。

*   **原理**：`@synchronized(obj)` 内部实际上是创建并管理一个哈希表，`obj` 作为 `key`，每个 `key` 对应一个递归互斥锁（Recursive Mutex）。当你使用相同的 `obj` 对象作为锁时，就能保证代码块的互斥执行。编译器会自动在代码块的开始处添加加锁逻辑，在结束（或异常抛出）时添加解锁逻辑。
*   **特点**：
    
    *   **方便易用**：语法简单，无需手动创建和销毁锁对象。
    *   **自动处理异常**：即使在 `@synchronized` 块中发生异常，也能保证锁会被正确释放。相当于语法糖，try，catch，finally，在finally里会释放锁。
    *   **递归锁**：允许同一线程在未释放锁的情况下，重复获取该锁，而不会造成死锁。
*   **使用场景**：对代码性能要求不那么高，希望快速实现线程安全的场景。适合保护小段的代码逻辑。
*   **缺点**：性能是所有锁中相对较低的，因为它需要维护一个全局的锁缓存池。
*   **代码示例**：
    ```objc
    - (void)myMethod {
        // 使用 self 对象作为锁，保证该方法在多线程环境下是安全的
        @synchronized (self) {
            // 需要保护的代码
            // ...
        }
    }
    ```

#### 2. `NSLock`

`NSLock` 是 Foundation 框架中对 POSIX 互斥锁 `pthread_mutex` 的一个基本封装，遵循 `NSLocking` 协议。

*   **原理**：它是一个标准的互斥锁。当一个线程调用 `lock` 方法获取锁后，其他线程再次调用 `lock` 会被阻塞。
*   **特点**：
    *   **纯粹的互斥锁**：非递归。如果同一线程尝试两次调用 `lock` 而不解锁，会造成死锁。
    *   **API 简单**：只有 `lock`, `unlock`, `tryLock` (尝试获取锁，不阻塞), `lockBeforeDate:` (在指定日期前阻塞等待)。
*   **使用场景**：需要保护一段代码，且能保证在同一线程中不会重复加锁的场景。
*   **代码示例**：
    ```objc
    NSLock *_lock = [[NSLock alloc] init];
    
    - (void)myMethod {
        [_lock lock];
        // 需要保护的代码
        // ...
        [_lock unlock];
    }
    ```

#### 3. `NSRecursiveLock`

`NSRecursiveLock` 是递归锁，同样是对 `pthread_mutex` 的封装，但设置了递归属性。

*   **原理**：它允许同一个线程在释放锁之前多次获取该锁，而不会导致死锁。系统会记录该线程获取锁的次数，每次调用 `unlock` 都会使计数减一，直到计数归零，锁才会被真正释放，其他线程此时才能获取该锁。
*   **特点**：
    *   **递归性**：解决了在递归调用或循环中需要重复加锁的问题。
*   **使用场景**：当一个方法内部可能会再次直接或间接调用到自身（或其他需要同一个锁的方法）时，必须使用递归锁。
*   **代码示例**：
    
    ```objc
    NSRecursiveLock *_recursiveLock = [[NSRecursiveLock alloc] init];
    
    - (void)recursiveMethod:(int)count {
        [_recursiveLock lock];
        if (count > 0) {
            NSLog(@"Count: %d", count);
            [self recursiveMethod:count - 1];
        }
        [_recursiveLock unlock];
    }
    ```

#### 4. `NSConditionLock`

`NSConditionLock` 是一种条件锁，它可以让你根据特定的条件来控制线程的加锁和等待。

*   **原理**：它在 `NSLock` 的基础上增加了一个“条件”整数值。一个线程可以请求获取一个持有特定条件值的锁。如果锁未被占用或已被占用但条件值不匹配，线程会阻塞。当另一个线程释放锁并设置了匹配的条件值时，等待的线程才会被唤醒。
*   **特点**：
    
    *   **条件驱动**：可以实现更复杂的线程同步逻辑，让线程按预定的顺序执行。
*   **使用场景**：经典的“生产者-消费者”模型。当任务队列中有任务时，消费者线程才能获取锁并执行任务。
*   **代码示例**：
    ```objc
    // 初始化时，锁未被占用，条件为 NO_DATA
    NSConditionLock *_conditionLock = [[NSConditionLock alloc] initWithCondition:NO_DATA];
    
    // 消费者线程
    - (void)consumer {
        // 等待直到锁的条件变为 HAS_DATA
        [_conditionLock lockWhenCondition:HAS_DATA];
        
        // 处理数据...
        
        // 释放锁，并将条件设置为 NO_DATA
        [_conditionLock unlockWithCondition:NO_DATA];
    }
    
    // 生产者线程
    - (void)producer {
        // 获取锁（不关心条件）
        [_conditionLock lock];
        
        // 生产数据...
        
        // 释放锁，并将条件设置为 HAS_DATA，以唤醒消费者
        [_conditionLock unlockWithCondition:HAS_DATA];
    }
    ```

#### 5. `dispatch_semaphore_t` (信号量)

信号量是 GCD 提供的一种同步机制，虽然不是传统意义上的“锁”，但可以完美地实现锁的功能，并且性能很高。

*   **原理**：信号量维护一个计数值。当调用 `dispatch_semaphore_wait` 时，如果计数值大于0，则将其减1并立即返回；如果计数值为0，则线程被阻塞。当调用 `dispatch_semaphore_signal` 时，会将计数值加1。
*   **用法当锁**：将信号量的初始值设为 **1**，就实现了一个标准的互斥锁。`wait` 相当于加锁，`signal` 相当于解锁。
*   **特点**：
    *   **高性能**：在等待锁时，会使线程进入休眠，不会消耗CPU时间，性能优于自旋锁。
    *   **控制并发数**：可以将初始值设为 N (N > 1)，从而允许最多有 N 个线程同时访问某个资源。
*   **使用场景**：既可以作为高效的互斥锁使用，也可以用于控制最大并发任务数。
*   **代码示例**：
    ```objc
    // 创建信号量，初始值为1
    dispatch_semaphore_t _semaphore = dispatch_semaphore_create(1);
    
    - (void)myMethod {
        // 等待信号量，相当于加锁 (如果计数值>0，则-1并继续；否则等待)
        dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
        
        // 需要保护的代码
        // ...
        
        // 释放信号量，相当于解锁 (计数值+1)
        dispatch_semaphore_signal(_semaphore);
    }
    ```

#### 6. `pthread_mutex`

这是基于 C 语言的 POSIX 标准线程锁，`NSLock` 等都是对它的封装。它提供了更高的可定制性。

*   **原理**：标准的互斥锁实现。
*   **特点**：
    *   **跨平台**：遵循 POSIX 标准。
    *   **类型可配置**：在初始化时可以将其配置为普通锁、递归锁等多种类型。
    *   **性能较高**：比 `NSLock` 等OC封装层更接近底层，性能更好。
    *   **使用复杂**：需要手动处理初始化 (`pthread_mutex_init`) 和销毁 (`pthread_mutex_destroy`)。
*   **代码示例**：
    ```objc
    #import <pthread.h>
    pthread_mutex_t _mutex;
    
    // 在 init 方法中初始化
    - (instancetype)init {
        self = [super init];
        if (self) {
            // 初始化互斥锁
            pthread_mutex_init(&_mutex, NULL);
        }
        return self;
    }
    
    - (void)myMethod {
        pthread_mutex_lock(&_mutex);
        // 需要保护的代码
        pthread_mutex_unlock(&_mutex);
    }
    
    // 在 dealloc 方法中销毁
    - (void)dealloc {
        pthread_mutex_destroy(&_mutex);
    }
    ```

**选择建议**：

*   **首选推荐**：在不考虑并发数控制的情况下，`dispatch_semaphore_t` (信号量设为1) 和 `pthread_mutex` 是兼顾性能和功能的优秀选择。如果对性能要求极致，并且App最低版本支持 iOS 10，则 `os_unfair_lock` 是最佳选择。
*   **简单场景**：如果只是保护一小段代码，不希望处理复杂的API，那么 `NSLock` 已经足够好。
*   **懒人/安全模式**：如果不确定用什么锁，或者代码逻辑比较复杂可能抛出异常，`@synchronized` 是最安全的选择，尽管性能稍差。
*   **特殊场景**：遇到递归调用用 `NSRecursiveLock`，遇到复杂的线程依赖关系用 `NSConditionLock`。

## 如何使线程保活

## TableView优化，怎么减少卡顿

### 优化清单总结

| 优先级   | 问题领域        | 优化策略                                            |
| -------- | --------------- | --------------------------------------------------- |
| **最高** | **Cell 高度**   | **预计算并缓存高度到数据模型**                      |
| **最高** | **图片处理**    | **异步加载 + 后台解码/缩放 (使用 SDWebImage)**      |
| **高**   | **Cell 创建**   | 保证 dequeueReusableCell，保持视图层级扁平          |
| **高**   | **图形渲染**    | 避免离屏渲染（圆角、阴影），设置视图为 opaque       |
| **中**   | **数据处理**    | 网络请求、JSON 解析等全部放到后台线程               |
| **中**   | **Auto Layout** | 如果使用自适应高度，**必须**提供 estimatedRowHeight |

## NSTimer造成的循环引用

---

### 一、问题场景：控制器与 Timer 的“死亡拥抱”

最常见的 `NSTimer` 内存泄漏发生在**一个 `UIViewController` 中**。

假设我们有一个 `MyViewController`，它需要在界面上显示一个倒计时。很自然地，我们会这样做：

1.  `MyViewController` 需要一个 `NSTimer` 属性来持有和管理这个定时器。
2.  `NSTimer` 在创建时，需要指定一个 `target`（目标对象）和一个 `selector`（要调用的方法）。通常，我们会把 `MyViewController` 自身作为 `target`。

**示例代码 (有问题的版本):**

```objc
#import "MyViewController.h"

@interface MyViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation MyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 创建并启动一个 NSTimer
    // self 作为 target，意味着 timer 会强引用 self
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                  target:self // 问题所在！
                                                selector:@selector(updateCountdown)
                                                userInfo:nil
                                                 repeats:YES];
}

- (void)updateCountdown {
    NSLog(@"Updating countdown...");
}

- (void)dealloc {
    NSLog(@"MyViewController is being deallocated.");
    // 我们期望在这里停止 timer
    [self.timer invalidate];
    self.timer = nil;
}

@end
```

### 二、循环引用是如何形成的？

这段看似正常的代码，却隐藏着一个致命的循环引用，我们可以称之为“死亡拥抱”：

1.  **控制器强引用 Timer**：
    `MyViewController` 有一个 `strong` 属性 `timer`。所以，`self` -> `timer` 是一条**强引用**。

2.  **Timer 强引用控制器**：
    这是最关键的一点。当你调用 `[NSTimer scheduledTimerWithTimeInterval:target:selector:...]` 时，`NSTimer` 对象为了能够在指定时间间隔后调用 `target` 的 `selector` 方法，它必须**持有**这个 `target`。因此，`NSTimer` 会**强引用**它的 `target`。所以，`timer` -> `self` 也是一条**强引用**。

**结果就形成了一个无法打破的闭环：**

`self (Controller) ----strong----> timer ----strong----> self (Controller)`



### 三、内存泄漏的后果

当用户离开这个 `MyViewController` 界面时（例如，通过 `navigationController` pop 返回上一级），会发生什么？

1.  `UINavigationController` 会释放它对 `MyViewController` 实例的强引用。
2.  此时，我们期望 `MyViewController` 的引用计数会变为 0，从而触发 `dealloc` 方法被调用。
3.  **但是，由于 `timer` 还强引用着 `MyViewController`，所以 `MyViewController` 的引用计数实际上是 1，而不是 0！**
4.  因此，`MyViewController` 的 `dealloc` 方法**永远不会被调用**。
5.  既然 `dealloc` 不被调用，那么 `[self.timer invalidate];` 这行关键代码也**永远不会被执行**。
6.  因为 `timer` 没有被 `invalidate`（停止），它会继续持有对 `MyViewController` 的强引用，并且会永远在后台触发 `updateCountdown` 方法。

**最终导致：**

*   `MyViewController` 对象和 `NSTimer` 对象都无法被释放，造成了**内存泄漏**。
*   定时器会一直在后台运行，持续消耗 CPU 资源。如果 `updateCountdown` 方法里有复杂逻辑，还会导致额外的性能问题。

---

### 四、解决方案

要解决这个问题，核心思想就是**打破这个循环引用**。我们有多种方法可以实现。

#### 方案一：在合适的时机手动停止 Timer

最简单直接的方法是，在视图控制器即将消失的时候，手动将 `timer` 停掉。`viewWillDisappear:` 或 `viewDidDisappear:` 是很好的时机。

```objc
// 在 MyViewController.m 中添加
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    // 在视图即将消失时，主动停止 timer
    // 这会解除 timer 对 self 的强引用
    [self.timer invalidate];
    self.timer = nil;
}
```

*   **优点**：简单易懂，改动最小。
*   **缺点**：不够完美。如果 `timer` 是需要跨多个页面后台运行的，这种方法就不适用了。它只适用于 `timer` 的生命周期与视图控制器完全一致的场景。

## 有哪些crash的情况

好的，在 Objective-C 开发中，Crash（崩溃）是每个开发者都必须面对和解决的问题。理解 Crash 的种类和原因，是提升应用稳定性的关键。

iOS 中的 Crash 大致可以分为两大类：**Mach 异常（底层内核级异常）** 和 **未被捕获的 NSException（上层应用级异常）**。此外，还有一些非典型的“崩溃”，如被系统强杀。

---

### 一、未被捕获的 NSException (Uncaught NSException)

这是最常见、也最容易理解的一类 Crash。它发生在 Objective-C/Cocoa Touch 框架层面，通常是因为代码逻辑错误或不当的 API 使用。当一个 `NSException` 对象被抛出 (`@throw`)，但没有任何 `@try-@catch` 块来捕获它时，应用就会崩溃。

#### **常见的 NSException 类型与场景：**

1.  **`NSRangeException` (数组越界)**
    *   **场景**: 试图访问一个数组、字符串等集合类型中不存在的索引。
    *   **示例**:
        ```objc
        NSArray *array = @[@"a", @"b"];
        NSString *value = array[2]; // Crash! 数组只有 0 和 1 两个索引
        ```

2.  **`NSInvalidArgumentException` (无效参数)**
    *   **场景**: 向方法传递了不合法或 `nil` 的参数。
    *   **示例**:
        ```objc
        NSDictionary *dict = @{@"key": @"value"};
        [dict setValue:nil forKey:@"anotherKey"]; // Crash! 字典的 value 不能为 nil
        
        NSString *str = nil;
        NSArray *array = @[@"a", str, @"c"]; // Crash! NSArray 的元素不能为 nil
        ```

3.  **`NSUnknownSelectorException` (找不到方法)**
    *   **场景**: 调用了一个对象上不存在的方法。
    *   **示例**:
        ```objc
        NSNumber *number = @(123);
        [number length]; // Crash! NSNumber 没有 length 方法，这是 NSString 的方法
        ```

4.  **`NSInternalInconsistencyException` (内部不一致)**
    *   **场景**: 通常发生在 API 的使用违反了其内部状态或预期时。
    *   **示例**:
        ```objc
        // 在没有 beginUpdates 的情况下调用 endUpdates
        [self.tableView endUpdates]; // Crash!
        
        // 在 UITableView 的数据源方法中，返回的 cell 数量和实际更新的数据不一致
        ```

5.  **`NSFileHandleOperationException` (文件操作错误)**
    *   **场景**: 对文件进行读写操作时发生错误。

6.  **KVC/KVO 相关的 Crash**
    *   **KVC**: `setValue:forUndefinedKey:` (为一个不存在的 key 赋值) 或 `valueForUndefinedKey:` (取一个不存在的 key 的值)。
    *   **KVO**:
        *   对一个没有被注册为观察者的对象，移除了观察。
        *   重复添加或移除观察者。
        *   被观察的对象在 dealloc 时，仍然有观察者存在（观察者没有被移除）。

---

### 二、Mach 异常 (Mach Exception) / 信号 (Signal)

这类 Crash 发生在更底层的操作系统内核层面，通常与内存管理、硬件指令错误等相关。当这些底层异常发生时，内核会向应用进程发送一个对应的 **Unix 信号 (Signal)** 来通知它。如果应用没有处理这个信号，默认的动作就是终止进程，即 Crash。

这些异常的错误信息通常以 `EXC_` 开头。

#### **常见的信号与场景：**

1.  **`EXC_BAD_ACCESS` (Signal: `SIGSEGV` / `SIGBUS`) - 坏内存访问**
    这是最常见的底层 Crash，通常被称为“野指针”或“悬垂指针”问题。
    
    *   **`SIGSEGV` (Segmentation Fault)**: 访问了一段不属于你的、非法的内存地址。
        *   **场景**:
            *   访问一个已经释放（dealloc）的对象。ARC 环境下，对象被释放后，指向它的指针没有被置为 `nil`，后续再通过这个指针访问对象就会 Crash。
            *   访问一个空指针（`NULL`）指向的地址。
    *   **`SIGBUS` (Bus Error)**: 访问了一个有效的内存地址，但访问方式是错误的（比如地址未对齐）。在 iOS 中比较少见，但在某些架构或特定操作下可能发生。
    *   **示例**:
        ```objc
        NSObject *obj = [[NSObject alloc] init];
        __unsafe_unretained NSObject *pointer = obj; // 使用 __unsafe_unretained 来模拟非 ARC 的情况
        
        // 假设 obj 在某个时刻被释放了
        // [obj release]; 
        
        // 此时 pointer 变成了野指针，再访问就会 EXC_BAD_ACCESS
        [pointer description]; // Crash!
        ```
    
2.  **`EXC_BAD_INSTRUCTION` (Signal: `SIGILL`) - 非法指令**
    *   **场景**: CPU 试图执行一个它不认识或非法的指令。
        *   最常见的原因是调用了一个不存在的函数指针。
        *   在 Swift 中，一些强制解包 `!` 失败时，会以这种形式 Crash。
    *   **示例**:
        ```objc
        typedef void (*FuncPtr)(void);
        FuncPtr func = (FuncPtr)0x0; // 一个指向地址0的函数指针
        func(); // Crash! CPU 试图执行地址0处的指令，这通常是非法的
        ```

3.  **`EXC_ARITHMETIC` (Signal: `SIGFPE`) - 算术异常**
    *   **场景**: 发生了数学运算错误，最常见的是**除以零**。
    *   **示例**:
        ```objc
        int a = 10;
        int b = 0;
        int c = a / b; // Crash!
        ```

---

### 三、被系统强杀 (Killed by OS)

这类情况不完全算是传统意义上的“Crash”，因为没有明确的异常或信号。应用是被操作系统主动终止的，通常是由于违反了某些系统规则。在 Crash 日志中，它们通常没有异常类型，但会有一个终止原因代码。

#### **常见的强杀原因：**

1.  **Watchdog Timeout (看门狗超时)**
    *   **场景**: 应用的主线程被**长时间阻塞**，无法响应系统事件（如启动、恢复、挂起等）。操作系统有一个“看门狗”机制，如果应用在规定时间内没有完成这些任务，就会被认为已经无响应，从而被强杀。
    *   **错误码**: `0x8badf00d` (谐音 "ate bad food")。
    *   **原因**: 在主线程执行了耗时的同步网络请求、大量的文件 I/O、复杂的计算等。

2.  **内存占用过高 (Out of Memory, OOM)**
    *   **场景**: 应用占用的内存超出了系统为其分配的额度。当系统内存紧张时，会优先终止内存占用最高的进程。
    *   **错误码**: `0xc00010ff` (谐音 "cool off")。
    *   **原因**: 内存泄漏（循环引用）、加载了过大的图片或数据而没有及时释放。

3.  **后台任务超时**
    *   **场景**: 应用在进入后台时，通过 `beginBackgroundTaskWithExpirationHandler:` 申请了一段额外的执行时间来完成某个任务，但在这个时间耗尽前任务仍未完成。

4.  **CPU 占用过高**
    *   **场景**: 应用在后台持续高强度地使用 CPU，导致设备发热、耗电过快，系统会为了保护硬件而终止该进程。

## 不使用官方和第三方的动画 如何实现头像旋转动画

```objc
@interface ViewController ()
@property (nonatomic, strong) UIImageView *avatarView;
@property (nonatomic, strong) CADisplayLink *displayLink;
@property (nonatomic, assign) CGFloat currentAngle;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.avatarView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"avatar"]];
    self.avatarView.frame = CGRectMake(100, 100, 100, 100);
    self.avatarView.layer.cornerRadius = 50;
    self.avatarView.clipsToBounds = YES;
    [self.view addSubview:self.avatarView];
    
    // 开始旋转
    [self startRotate];
}

- (void)startRotate {
    self.currentAngle = 0;
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(updateRotation)];
    [self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)updateRotation {
    self.currentAngle += M_PI / 180; // 每次加 1 度
    if (self.currentAngle >= M_PI * 2) {
        self.currentAngle = 0;
    }
    self.avatarView.transform = CGAffineTransformMakeRotation(self.currentAngle);
}

- (void)stopRotate {
    [self.displayLink invalidate];
    self.displayLink = nil;
}

@end

```

## ios中native和h5页面的区别

好的，这是一个在移动开发中非常核心且常见的话题。Native App 和 H5 页面代表了两种截然不同的技术实现和用户体验路径。

我将从**定义、用户体验、性能、功能、开发维护成本**等多个维度，为你详细对比它们的区别。

---

### **核心定义**

*   **Native App (原生应用)**:
    *   **定义**: 使用平台官方推荐的开发语言和工具链，直接为特定操作系统（如 iOS 或 Android）编写的应用。
    *   **iOS 技术栈**: 主要使用 **Objective-C** 或 **Swift** 语言，配合苹果的 **UIKit** (或 SwiftUI) 框架，通过 Xcode 进行开发。
    *   **比喻**: 像是在一个国家（操作系统）里，用**官方语言（Swift/OC）**和**官方建筑规范（UIKit）**建造的一座**本地建筑**。它和这个国家的环境、设施完美融合。

*   **H5 页面 (Web App / 移动网页)**:
    *   **定义**: 本质上就是一个**移动设备上优化的网页**。它使用标准的 Web 技术编写，并在一个浏览器内核中运行。
    *   **技术栈**: 主要使用 **HTML5, CSS3, JavaScript**，通常会配合一些前端框架（如 Vue.js, React）。
    *   **在 iOS 中的运行环境**: H5 页面是在一个 `WKWebView`（一个内嵌在 App 中的浏览器组件）里加载和渲染的。
    *   **比喻**: 像是一个标准化的**集装箱房屋**。它可以被放置在任何国家（任何支持浏览器的平台），但它与当地的建筑风格和设施可能不是原生集成的，需要通过一个“吊车”（`WKWebView`）来安放。

---

### **详细对比**

| 对比维度          | Native App (原生应用)                                        | H5 页面 (Web App)                                            |
| :---------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **性能体验**      | **极高**                                                     | **较低**                                                     |
|                   | **流畅度**: 直接利用 GPU 进行渲染，动画、手势响应极其流畅丝滑。 | **流畅度**: 依赖浏览器内核的渲染，复杂动画和手势可能会有延迟和卡顿感。 |
|                   | **加载速度**: 代码和资源大部分在本地，启动和加载速度快。     | **加载速度**: 需要通过网络加载 HTML/CSS/JS 和资源，受网络状况影响大，首次加载慢。 |
|                   | **内存占用**: 经过高度优化，内存管理更高效。                 | **内存占用**: `WKWebView` 本身是一个较重的进程，内存开销相对较大。 |
| **功能调用**      | **无限制**                                                   | **非常受限**                                                 |
|                   | 可以直接、无缝地调用**所有系统级 API**，如相机、相册、GPS、蓝牙、联系人、推送通知、Touch ID/Face ID 等。 | 只能通过浏览器内核提供的**有限的、标准化的接口**来访问硬件。很多高级功能（如蓝牙、后台任务）无法直接调用，需要原生代码“搭桥”才能实现。 |
| **离线能力**      | **强**                                                       | **弱**                                                       |
|                   | 核心功能和数据可以完全在本地运行和存储，不受网络限制。       | 严重依赖网络。虽然可以通过一些技术（如 Service Worker, LocalStorage）实现简单的离线缓存，但复杂逻辑的离线运行非常困难。 |
| **用户体验 (UX)** | **最佳**                                                     | **一般**                                                     |
|                   | **UI/UX 一致性**: 完全遵循苹果的**人机交互指南 (HIG)**，外观和交互与系统原生应用（如设置、邮件）保持一致，用户学习成本低。 | **UI/UX 一致性**: 通常是自定义的 Web 风格，与系统风格有差异，可能给用户带来割裂感。 |
|                   | **交互反馈**: 手势识别精准、响应迅速，提供丰富的触感反馈。   | **交互反馈**: 手势处理能力有限，反馈通常不如原生灵敏。       |
| **开发与维护**    | **成本高，周期长**                                           | **成本低，周期短**                                           |
|                   | **平台特定**: 需要为 iOS 和 Android 分别开发和维护两套代码。 | **跨平台**: “一次编写，到处运行”，一套代码可以同时在 iOS, Android 和 Web 上运行。 |
|                   | **开发语言**: 需要掌握 Swift/Objective-C 等原生语言。        | **开发语言**: 使用前端 Web 技术，人才储备更广泛。            |
| **更新与发布**    | **慢，受限**                                                 | **快，灵活**                                                 |
|                   | 任何微小的改动都需要重新**打包、提交到 App Store 审核、等待用户更新**。整个流程周期长，无法快速修复 bug 或上线新功能。 | **动态更新**: 内容和逻辑都托管在服务器上。只需在服务器上修改代码，所有用户**打开 App 时就能立即看到最新的内容**，无需审核和更新 App。这对于需要频繁迭代的业务（如电商活动页）是巨大优势。 |
| **安装成本**      | **高**                                                       | **低**                                                       |
|                   | 用户需要从 App Store 下载并安装，占用手机存储空间。          | 用户无需安装，可以通过一个链接、二维码或在 App 内直接打开。  |

---

### **混合开发 (Hybrid App)：两者的结合**

在实际开发中，很少有应用是 100% 的 Native 或 100% 的 H5。绝大多数主流 App（如微信、淘宝、支付宝）都采用**混合开发的模式**，取长补短。

*   **框架/外壳是 Native**: App 的主框架、导航栏、标签栏，以及对性能和体验要求极高的核心功能（如微信的聊天、朋友圈）是**原生开发**的。这保证了应用的稳定性和基础体验。
*   **内容/业务是 H5**: 一些变化频繁、非核心的功能页面（如电商的活动页、新闻文章详情页、小程序、内置的帮助文档）则使用 **H5 页面内嵌**的方式。这利用了 H5 跨平台和动态更新的优势。

**这种模式的优势**:
*   **兼顾了性能与灵活性**: 核心体验用 Native 保证，业务迭代用 H5 加速。
*   **降低了开发成本**: 大量非核心业务可以用一套 H5 代码实现。
*   **实现了动态化运营**: 可以不通过 App 更新，就上线新的活动和功能。

## `@dynamic` 关键字？

## autolayout实现原理

## 网络缓存中涉及到什么算法思想有研究过吗

### **第一部分：讲解最经典的 LRU (Least Recently Used) 算法**

“首先，最著名也是最常用的算法之一，是 **LRU，即‘最近最少使用’算法**。”

*   **核心思想**: “LRU 的核心思想非常直观，它基于一个假设：**如果一个数据最近被访问过，那么它在将来也很有可能被再次访问。反之，如果一个数据已经很久没有被访问了，那么它在将来被访问的可能性就很小。**”
*   **如何实现**: “因此，当缓存满了需要淘汰数据时，LRU 会选择**淘汰掉那个最久没有被访问过的数据**。”
    *   “要实现 LRU，我们需要一个数据结构，它不仅能存储键值对，还要能记录每个数据的访问顺序。最经典的的实现方式是使用一个**哈希表（`NSDictionary` / `HashMap`）**和**一个双向链表（`Doubly Linked List`）**。”
    *   **哈希表**: 负责提供 `O(1)` 时间复杂度的快速查找。Key 是缓存的键，Value 是指向链表节点的指针。
    *   **双向链表**: 负责维护数据的访问顺序。
        *   当一个数据被**访问（命中）**时，我们会把它从链表的当前位置移动到**链表的头部**。
        *   当需要**插入**一个新数据时，我们把它放在**链表的头部**。
        *   当需要**淘汰**数据时，我们直接淘汰掉**链表的尾部**的那个节点，因为它就是最久没有被访问过的。
*   **优点**: “LRU 算法实现相对简单，性能高效（查找、插入、删除都是 O(1)），并且在大多数场景下，它的预测都相当准确，能够提供很高的缓存命中率。”
*   **缺点**: “但 LRU 也有一个明显的缺点，就是它无法应对**‘偶然的、周期性的批量访问’**。比如，一个程序在启动时需要进行一次全量的数据扫描，这会导致大量不常用的老数据被突然访问一次，从而把缓存中真正有用的热点数据全部给‘冲刷’出去，造成‘缓存污染’。”

> **面试官视角**: 候选人清晰地解释了 LRU 的思想、实现方式（哈希表+双向链表）、以及它的优缺点。特别是能指出其在“批量扫描”场景下的不足，显示了其思考的深度。

---

### **第二部分：讲解 LFU (Least Frequently Used) 算法作为对比**

“为了解决 LRU 害怕‘偶发性访问’的问题，就有了另一种经典的算法——**LFU，即‘最不经常使用’算法**。”

*   **核心思想**: “LFU 的思想是：**如果一个数据在过去一段时间内被访问的次数很多，那么它就是热点数据，应该被保留。反之，一个数据被访问的次数很少，即使它刚刚被访问过，也应该被优先淘汰。**”
*   **如何实现**: “实现 LFU 需要为每个缓存项额外维护一个**访问频率计数器**。当需要淘汰数据时，LFU 会选择淘汰掉那个**访问次数最少**的数据。如果多个数据的访问次数相同，通常会再结合 LRU 的思想，淘汰掉其中最久没有被访问的那个。”
    *   “LFU 的实现比 LRU 要复杂得多。一个经典的实现是使用**一个哈希表**来存储键值对，再用**另一个哈希表**来存储频率和对应的数据链表。即 `Freq -> DoublyLinkedList`。这使得在访问一个数据导致其频率变化时，需要在不同的频率链表之间移动节点。”
*   **优点**: “LFU 能够很好地识别出那些长期、持续被访问的‘热点数据’，不容易被偶发的、批量的访问所干扰，长期来看缓存命中率可能更高。”
*   **缺点**:
    *   **实现复杂**: 数据结构的维护和操作都比 LRU 复杂。
    *   **频率更新开销大**: 每次访问都需要更新频率计数器，开销比 LRU 大。
    *   **历史数据问题**: 一个数据在早期被大量访问，获得了很高的频率，但之后再也不被访问了。它可能会因为这个“历史包袱”而在缓存中“赖”很久，无法被淘汰，这被称为**“缓存抖动”**问题。为了解决这个问题，LFU 的实现通常还需要引入**时间衰减**机制，让频率计数随时间推移而降低。

> **面试官视角**: 候选人能清晰地对比 LFU 和 LRU，并准确地指出了 LFU 的优缺点及其核心挑战（实现复杂、历史数据问题），并能提到“时间衰减”这样的优化思路，知识体系非常完整。

---

### **第三部分：结合实际应用场景（体现工程思维）**

“在实际的网络缓存应用中，比如我们常用的 `SDWebImage` 或 `YYCache`，它们通常不会只使用一种单一的、纯粹的算法，而是采用**混合策略或者对经典算法进行优化**。”

*   **`YYCache` 的实现**: “我了解过 `YYCache` 的设计，它的内存缓存部分就实现了一个非常高效的、近似 LRU 的淘汰算法。它没有使用标准的双向链表，而是通过一个字典来维护所有节点，并通过 `_lru` 这个内部 `dispatch_semaphore_t` 信号量和异步队列来定期、批量地淘汰尾部的对象，以平衡性能和实时性。”
*   **实际的权衡**: “在设计一个网络缓存系统时，我们需要根据业务场景来做权衡：”
    *   **对于图片缓存**: “像用户头像这种会反复加载的，就很适合 LRU/LFU。但对于新闻客户端里只看一次的列表图片，可能用一个更简单的 **FIFO（先进先出）**或者基于**缓存大小（Cost）**和**存活时间（TTL）**的策略就足够了。”
    *   **对于 API 数据缓存**: “可能更看重数据的**时效性**，所以基于 TTL（Time-To-Live）的过期策略会和 LRU/LFU 结合使用。一个数据即使再热门，只要超过了指定的缓存时间，也必须被淘汰或重新验证。”

## Masonry有什么亮点

好的，`Masonry` 是 iOS 和 macOS 开发中一个极其流行且经典的第三方**自动布局（Auto Layout）框架**。它的出现，极大地改变了开发者用代码编写自动布局的方式。

要说 `Masonry` 的亮点，可以总结为以下几个方面，它不仅解决了原生 API 的痛，还提供了一套极具表现力的编程范式。

---

### **亮点一：链式语法 (Chaining Syntax)，极大地提高了代码的可读性和内聚性**

这是 `Masonry` **最核心、最耀眼**的亮点。它引入了一种非常流畅的、类似自然语言的链式调用语法。

**我们先来看一下原生 Auto Layout API 的痛点：**
```objc
// 原生 API：创建一个约束，需要 5 个参数，代码冗长且分散
[NSLayoutConstraint constraintWithItem:view1
                             attribute:NSLayoutAttributeLeading
                             relatedBy:NSLayoutRelationEqual
                                toItem:superview
                             attribute:NSLayoutAttributeLeading
                            multiplier:1.0
                              constant:20.0];```
要为一个视图设置 `top`, `left`, `bottom`, `right` 四个约束，你需要写四遍这样又长又臭的代码，非常痛苦，而且难以阅读。

**再来看 `Masonry` 的解决方案：**
```objc
// Masonry：使用链式语法，将所有约束内聚在一起
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(20);
    make.left.equalTo(superview.mas_left).with.offset(20);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-20);
    make.right.equalTo(superview.mas_right).with.offset(-20);
}];
```
或者更简洁的写法：
```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(superview).with.insets(UIEdgeInsetsMake(20, 20, 20, 20));
}];
```
**这种语法的亮点在于：**
1.  **可读性极强**: `make.top.equalTo(superview.mas_top)` 读起来就像一句英文：“让它的顶部等于父视图的顶部”。
2.  **代码内聚**: 一个视图的所有布局约束，都被集中在一个 `Block` 中定义，逻辑清晰，一目了然。
3.  **简洁**: 大量使用了点语法（`.`）来连接属性，并提供了如 `edges`, `size`, `center` 这样的复合属性，极大地减少了代码量。

---

### **亮点二：自动装箱 (Auto-Boxing)，简化了常量和乘数的处理**

`Masonry` 巧妙地封装了对常量的处理，你不需要去区分是在处理 `constant` 还是 `multiplier`。

*   **原生 API**: 你需要明确地传入 `constant` 和 `multiplier` 两个参数。
*   **`Masonry`**:
    *   **常量 (Offset)**: 通过 `.offset()` 方法来设置。
        ```objc
        make.left.equalTo(superview).offset(20);
        ```
    *   **乘数 (Multiplier)**: 通过 `.multipliedBy()` 方法来设置。
        ```objc
        make.width.equalTo(superview).multipliedBy(0.5); // 宽度是父视图的一半
        ```
    *   **值类型的自动装箱**: 你可以直接传入一个**数字**，而不需要手动包装成 `NSNumber`。
        ```objc
        make.height.equalTo(@44); // Masonry 会自动处理
        ```

---

### **亮点三：强大的约束管理能力（更新、替换、卸载）**

`Masonry` 不仅让创建约束变得简单，管理已存在的约束也同样优雅。

1.  **更新约束 (`mas_updateConstraints`)**:
    *   **场景**: 你只想修改某个约束的**常量值**，比如一个视图的高度需要动态变化。
    *   **做法**:
        ```objc
        [myView mas_updateConstraints:^(MASConstraintMaker *make) {
            make.height.equalTo(@(self.dynamicHeight)); // 只会更新高度约束的 constant
        }];
        ```
    *   **亮点**: 它非常智能。它会去查找之前已经存在的、与 `height` 相关的约束，并只更新它的 `constant` 值。**如果找不到对应的约束，它会自动为你创建一个新的。** 这避免了先移除旧约束再添加新约束的繁琐操作。

2.  **重新创建约束 (`mas_remakeConstraints`)**:
    *   **场景**: 你需要**完全丢弃**之前对一个视图设置的所有约束，然后应用一套全新的约束。比如，一个视图需要从屏幕左边移动到右边，它依赖的参照物都变了。
    *   **做法**:
        ```objc
        [myView mas_remakeConstraints:^(MASConstraintMaker *make) {
            // 这里会先删除掉 myView 之前所有的约束
            // 然后再添加下面的新约束
            make.top.equalTo(anotherView);
            make.right.equalTo(superview);
        }];
        ```

3.  **对约束本身的引用**:
    *   `Masonry` 允许你把创建的约束保存到一个变量中，以便后续单独对它进行修改或移除。
    ```objc
    // 保存对顶部约束的引用
    MASConstraint *topConstraint = [myView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(superview.mas_top).with.offset(20);
        // ...
    }];
    
    // 在之后的某个时刻
    topConstraint.offset = 50; // 直接修改约束的 offset
    [topConstraint uninstall]; // 单独移除这条约束
    ```

## MVMM

好的，这是一个非常核心的架构设计问题。我们不仅要理解 MVC 和 MVVM 的理论区别，更重要的是能通过代码来直观地感受它们在**职责划分、代码组织和可测试性**上的巨大差异。

我们将以一个非常简单的“登录”场景为例，分别用 MVC 和 MVVM 来实现，让你能清晰地看到对比。

**场景需求**:
*   界面上有两个输入框：用户名（`username`）和密码（`password`）。
*   一个登录按钮（`loginButton`）。
*   当用户名和密码都输入内容后，登录按钮才可用。
*   点击登录按钮，模拟一个网络请求，然后根据结果显示成功或失败的提示。

---

### **MVC (Model-View-Controller) 模式实现**

在经典的 MVC 模式中，`Controller` 是一个“大总管”，它承担了几乎所有的“胶水代码”和业务逻辑。

#### **1. Model (`LoginModel.h/m`)**
模型非常简单，只负责定义数据结构。
```objc
// LoginModel.h
@interface LoginModel : NSObject
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;
@end
```

#### **2. View (`LoginViewController` 的 `view`)**
视图层就是 `LoginViewController` 的 `view` 以及其上的 `UITextField` 和 `UIButton`。它本身没有逻辑。

#### **3. Controller (`LoginViewController.h/m`)**
`Controller` 是所有逻辑的中心。

```objc
// LoginViewController.m
#import "LoginViewController.h"
#import "LoginModel.h"

@interface LoginViewController () <UITextFieldDelegate>
@property (nonatomic, strong) UITextField *usernameField;
@property (nonatomic, strong) UITextField *passwordField;
@property (nonatomic, strong) UIButton *loginButton;
@property (nonatomic, strong) LoginModel *model;
@end

@implementation LoginViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.model = [[LoginModel alloc] init];
    
    // --- View 的创建和布局 ---
    [self setupViews];
    
    // --- Controller 承担了所有的事件处理和逻辑 ---
    [self.usernameField addTarget:self action:@selector(textFieldDidChange:) forControlEvents:UIControlEventEditingChanged];
    [self.passwordField addTarget:self action:@selector(textFieldDidChange:) forControlEvents:UIControlEventEditingChanged];
    [self.loginButton addTarget:self action:@selector(loginButtonTapped:) forControlEvents:UIControlEventTouchUpInside];
    
    // --- Controller 承担了初始状态的设置 ---
    [self updateLoginButtonState];
}

// 1. 事件处理：监听 TextField 的变化
- (void)textFieldDidChange:(UITextField *)textField {
    if (textField == self.usernameField) {
        self.model.username = textField.text;
    } else if (textField == self.passwordField) {
        self.model.password = textField.text;
    }
    
    // 2. 业务逻辑：根据 Model 的状态更新 View
    [self updateLoginButtonState];
}

// 3. 业务逻辑：更新登录按钮的可用状态
- (void)updateLoginButtonState {
    BOOL isUsernameValid = self.model.username.length > 0;
    BOOL isPasswordValid = self.model.password.length > 0;
    self.loginButton.enabled = isUsernameValid && isPasswordValid;
    self.loginButton.backgroundColor = self.loginButton.enabled ? [UIColor blueColor] : [UIColor grayColor];
}

// 4. 事件处理 & 网络逻辑：处理登录按钮点击
- (void)loginButtonTapped:(UIButton *)sender {
    NSLog(@"开始登录，用户名: %@, 密码: %@", self.model.username, self.model.password);
    
    // 模拟网络请求
    [self showLoadingIndicator];
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self hideLoadingIndicator];
        // 模拟成功
        if ([self.model.username isEqualToString:@"admin"] && [self.model.password isEqualToString:@"123"]) {
            [self showAlertWithTitle:@"成功" message:@"登录成功！"];
        } else {
            [self showAlertWithTitle:@"失败" message:@"用户名或密码错误"];
        }
    });
}
// ... 省略 setupViews, showAlertWithTitle, showLoadingIndicator 等 UI 代码 ...
@end
```

**MVC 的问题在这里体现得淋漓尽致：**
*   **Massive View Controller (臃肿的视图控制器)**: `LoginViewController` 做了太多事！它既要负责创建和布局 `View`，又要监听 `View` 的事件，还要更新 `Model`，同时处理业务逻辑（按钮是否可用），甚至还包含了网络请求和弹窗提示。
*   **耦合度高**: `View` (TextField, Button) 和 `Controller` 紧密耦合。`Model` 和 `Controller` 也紧密耦合。
*   **可测试性差**: 几乎所有的逻辑都在 `ViewController` 里。你怎么去测试“按钮可用性”这个逻辑？你必须创建一个完整的 `LoginViewController` 实例，还要给它添加 `View`，然后模拟文本输入... 这非常困难。

---

### **MVVM (Model-View-ViewModel) 模式实现**

MVVM 的核心是引入了一个新的角色：`ViewModel`。它的任务是**从 `Controller` 中抽离出“展示逻辑”和“状态”**。

#### **1. Model (`LoginModel.h/m`)**
保持不变。

#### **2. View (`LoginViewController` 的 `view`)**
保持不变，但它的**数据源和事件处理者**将变成 `ViewModel`。

#### **3. ViewModel (`LoginViewModel.h/m`)**
这是 MVVM 的灵魂。它是一个纯粹的 `NSObject`，不包含任何 `UIKit` 的东西。

```objc
// LoginViewModel.h
#import "LoginModel.h"

@interface LoginViewModel : NSObject

// --- 属性：暴露给 View 的状态 ---
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;

// --- 属性：经过处理后，可以直接被 View 绑定的状态 ---
@property (nonatomic, assign, readonly) BOOL isLoginButtonEnabled;
@property (nonatomic, strong, readonly) UIColor *loginButtonColor;
@property (nonatomic, copy, readonly) NSString *statusText;

// --- Block 回调：用于通知 View 发生了变化 ---
@property (nonatomic, copy) void (^loginStateDidChange)(LoginViewModel *viewModel);

// --- 方法：暴露给 View 的行为 ---
- (void)login;

@end
```

```objc
// LoginViewModel.m
@implementation LoginViewModel

- (instancetype)init {
    self = [super init];
    if (self) {
        // 初始化状态
        [self updateLoginState];
    }
    return self;
}

// 1. 当输入变化时，更新内部状态
- (void)setUsername:(NSString *)username {
    _username = username;
    [self updateLoginState];
}

- (void)setPassword:(NSString *)password {
    _password = password;
    [self updateLoginState];
}

// 2. 核心的“展示逻辑”，更新可绑定的属性
- (void)updateLoginState {
    BOOL isUsernameValid = self.username.length > 0;
    BOOL isPasswordValid = self.password.length > 0;
    
    // 更新可以直接被 View 使用的状态
    _isLoginButtonEnabled = isUsernameValid && isPasswordValid;
    _loginButtonColor = _isLoginButtonEnabled ? [UIColor blueColor] : [UIColor grayColor];
    
    // 通过回调通知 View 刷新
    if (self.loginStateDidChange) {
        self.loginStateDidChange(self);
    }
}

// 3. 核心的“业务逻辑”
- (void)login {
    NSLog(@"ViewModel 开始登录，用户名: %@, 密码: %@", self.username, self.password);
    
    _statusText = @"正在登录...";
    if (self.loginStateDidChange) { self.loginStateDidChange(self); }

    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        if ([self.username isEqualToString:@"admin"] && [self.password isEqualToString:@"123"]) {
            self->_statusText = @"登录成功！";
        } else {
            self->_statusText = @"用户名或密码错误";
        }
        if (self.loginStateDidChange) { self.loginStateDidChange(self); }
    });
}
@end
```

#### **4. Controller (`LoginViewController.h/m`)**
`Controller` 现在变得极其“苗条”。它的唯一职责就是**创建并持有 `View` 和 `ViewModel`，并把它们“绑定”在一起**。

```objc
// LoginViewController.m (MVVM version)
#import "LoginViewController.h"
#import "LoginViewModel.h"

@interface LoginViewController ()
@property (nonatomic, strong) UITextField *usernameField;
@property (nonatomic, strong) UITextField *passwordField;
@property (nonatomic, strong) UIButton *loginButton;
@property (nonatomic, strong) UILabel *statusLabel; // 用于显示状态
@property (nonatomic, strong) LoginViewModel *viewModel;
@end

@implementation LoginViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.viewModel = [[LoginViewModel alloc] init];
    
    [self setupViews];
    
    // --- Controller 唯一的核心职责：数据绑定 (Data Binding) ---
    [self bindViewModel];
}

- (void)bindViewModel {
    // 1. View -> ViewModel 的绑定
    [self.usernameField addTarget:self action:@selector(usernameDidChange:) forControlEvents:UIControlEventEditingChanged];
    [self.passwordField addTarget:self action:@selector(passwordDidChange:) forControlEvents:UIControlEventEditingChanged];
    [self.loginButton addTarget:self.viewModel action:@selector(login) forControlEvents:UIControlEventTouchUpInside];
    
    // 2. ViewModel -> View 的绑定 (通过 Block 回调)
    __weak typeof(self) weakSelf = self;
    self.viewModel.loginStateDidChange = ^(LoginViewModel *viewModel) {
        // 当 ViewModel 的状态改变时，这个 block 会被调用
        // Controller 在这里把 ViewModel 的新状态，应用到 View 上
        weakSelf.loginButton.enabled = viewModel.isLoginButtonEnabled;
        weakSelf.loginButton.backgroundColor = viewModel.loginButtonColor;
        weakSelf.statusLabel.text = viewModel.statusText;
    };
    
    // 3. 触发一次初始绑定
    self.viewModel.loginStateDidChange(self.viewModel);
}

// 事件处理，只负责将 View 的变化传递给 ViewModel
- (void)usernameDidChange:(UITextField *)textField {
    self.viewModel.username = textField.text;
}

- (void)passwordDidChange:(UITextField *)textField {
    self.viewModel.password = textField.text;
}
// ... 省略 setupViews ...
@end
```
*（在实际项目中，我们通常会使用 **KVO** 或响应式框架如 **RAC/RxSwift** 来实现更优雅的数据绑定，从而省去 `loginStateDidChange` 这个 block 和 `usernameDidChange:` 这些方法。）*

### **对比总结**

| 对比项       | MVC                                                          | MVVM                                                         |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **核心职责** | `Controller` 做了所有事：UI事件、业务逻辑、网络、导航...     | `ViewModel` 抽取了**状态**和**展示逻辑**。<br>`Controller` 只负责**绑定**。 |
| **代码量**   | `Controller` 极度**臃肿**                                    | `Controller` 变得非常**轻薄**。<br>`ViewModel` 包含了大部分逻辑。 |
| **耦合关系** | `View <-> Controller <-> Model`                              | `View <-> Controller <-> ViewModel -> Model`<br>`View` 与 `ViewModel` 通过绑定**解耦**。 |
| **可测试性** | **差**。要测试业务逻辑，必须实例化一个完整的 `ViewController`。 | **极高**。`ViewModel` 是一个**纯粹的 `NSObject`**，不依赖任何 `UIKit`。你可以非常轻松地为它编写单元测试，输入 `username` 和 `password`，然后断言 `isLoginButtonEnabled` 的值是否正确。 |
| **数据流**   | 比较混乱，可以是双向的。                                     | **单向数据流**非常清晰：`View` 产生事件 -> `ViewModel` 处理并更新状态 -> `View` 响应状态变化。 |

通过这个代码对比，你可以非常直观地看到，MVVM 模式通过引入 `ViewModel`，成功地将 `ViewController` 从繁重的业务逻辑中解放出来，使其回归到了一个更纯粹的“视图管理者”的角色，从而极大地提升了代码的**可测试性、可维护性和模块化程度**。

## 软件设计原则

### 1. SOLID 原则

SOLID 是面向对象设计中五个基本原则的首字母缩写，由罗伯特·C·马丁（Robert C. Martin）提出。它们是现代软件工程的基石。

*   **S - 单一职责原则 (Single Responsibility Principle, SRP)**
    *   **含义**：一个类或模块应该有且只有一个引起它变化的原因。 这意味着一个类应该只负责一项功能。
    *   **目的**：提高内聚性，降低耦合度。 当你需要修改一项功能时，只需修改对应的类，不会影响到其他不相关的功能。
*   **O - 开放/封闭原则 (Open/Closed Principle, OCP)**
    *   **含义**：软件实体（如类、模块、函数等）应该对扩展开放，对修改封闭。
    *   **目的**：允许在不修改现有代码的情况下增加新功能。这通常通过继承和多态，或者使用插件式架构来实现，从而提高了系统的稳定性和灵活性。
*   **L - 里氏替换原则 (Liskov Substitution Principle, LSP)**
    *   **含义**：所有引用基类的地方必须能透明地使用其子类的对象，而程序行为不发生改变。简单来说，子类对象可以替换掉所有父类对象出现的地方，而不会引起任何错误或不符合预期的行为。
    *   **目的**：确保继承体系的正确性。子类不应该重写父类的方法以使其执行完全不同或不兼容的操作，这保证了代码的可靠性和可预测性。
*   **D - 依赖倒置原则 (Dependency Inversion Principle, DIP)**
    *   **含义**：高层模块不应该依赖于低层模块，两者都应该依赖于抽象。抽象不应该依赖于细节，细节应该依赖于抽象。
    *   **目的**：实现模块间的解耦。通过面向接口编程，而不是面向实现编程，可以轻松地替换底层实现细节，而无需修改高层模块。遵循这些设计原则，可以显著提升软件质量，使软件产品在整个生命周期中都易于管理和演进。

## 事件传递

`hitTest:`的本质是一个递归函数，它的返回值决定了整个查找链的走向。**一旦在递归的任何一层中，有一个子视图的`hitTest:`调用返回了一个非`nil`的对象，那么这个对象就会被逐层向上传递，并成为整个查找过程的最终结果。**

让我们用一个更精确的、伪代码式的算法来描述任何一个`UIView`的`hitTest:withEvent:`方法的内部逻辑：

```objectivec
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    
    // 步骤 1: 检查自身先决条件
    // 如果视图隐藏、不允许交互或完全透明，那么它和它的所有子视图都无法响应事件。
    if (self.hidden || !self.userInteractionEnabled || self.alpha < 0.01) {
        return nil; // 直接返回 nil，终止这条分支的查找
    }
    
    // 步骤 2: 检查触摸点是否在自己内部
    // 如果点不在视图的边界内，那么它和它的所有子视图也肯定无法响应。
    if (![self pointInside:point withEvent:event]) {
        return nil; // 直接返回 nil，终止这条分支的查找
    }
    
    // 步骤 3: 从后往前（从最上层）遍历子视图，进行递归查找
    // 这是最关键的一步
    for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
        
        // 坐标转换：将触摸点从当前视图的坐标系转换到子视图的坐标系
        CGPoint subviewPoint = [self convertPoint:point toView:subview];
        
        // 递归调用子视图的 hitTest: 方法
        UIView *hitTestView = [subview hitTest:subviewPoint withEvent:event];
        
        // ----> 核心判断在此 <----
        // 如果子视图的 hitTest: 方法返回了一个非 nil 的对象，
        // 这意味着在子视图的层级中已经找到了最合适的响应者。
        if (hitTestView != nil) {
            // 那么当前视图的查找任务就完成了，它不需要再检查其他子视图，
            // 也不需要检查自己。它要做的就是立刻将这个结果向上传递。
            return hitTestView; 
        }
    }
    
    // 步骤 4: 如果所有子视图都没有返回结果，那么自己就是响应者
    // 能执行到这里，说明：
    // 1. 触摸点在当前视图内部（通过了步骤2）。
    // 2. 遍历了所有子视图，但它们的 hitTest: 都返回了 nil。
    // 因此，当前视图就是这次查找的最佳响应者。
    return self; 
}
```

### 结合一个具体例子来走一遍流程

假设我们有这样的视图层级：
`UIWindow`
└── `View A` (绿色背景)
    └── `View B` (蓝色背景，在A的内部)
        └── `Button C` (红色按钮，在B的内部)

用户触摸了 **红色按钮 C**。

1.  **`UIWindow` 的 `hitTest:` 被调用。**
    *   步骤 1 & 2 通过。
    *   步骤 3: `Window` 只有一个子视图 `View A`。它调用 `[View A hitTest:...]`。

2.  **`View A` 的 `hitTest:` 被调用。**
    *   步骤 1 & 2 通过。
    *   步骤 3: `View A` 只有一个子视图 `View B`。它调用 `[View B hitTest:...]`。

3.  **`View B` 的 `hitTest:` 被调用。**
    *   步骤 1 & 2 通过。
    *   步骤 3: `View B` 只有一个子视图 `Button C`。它调用 `[Button C hitTest:...]`。

4.  **`Button C` 的 `hitTest:` 被调用。**
    *   步骤 1 & 2 通过（触摸点在按钮内）。
    *   步骤 3: `Button C` 没有子视图，`for` 循环直接结束。
    *   步骤 4: 因为没有子视图返回结果，并且点在自己内部，**`Button C` 的 `hitTest:` 方法返回 `self` (也就是 `Button C` 对象)**。这是整个递归链中**第一个返回的非 `nil` 对象**。

5.  **返回链开始回溯**
    *   现在回到 `View B` 的 `hitTest:` 方法中。它收到了 `[Button C hitTest:...]` 的返回值，这个值是 `Button C` 对象，**不是 `nil`**。
    *   `View B` 的代码执行到 `if (hitTestView != nil)`，判断为真。
    *   **于是 `View B` 的 `hitTest:` 立刻 `return hitTestView;`**，也就是把 `Button C` 对象原封不动地返回了。它不会再检查其他子视图，也不会返回自己。

6.  **继续回溯**
    *   现在回到 `View A` 的 `hitTest:` 方法中。它收到了 `[View B hitTest:...]` 的返回值，这个值是 `Button C` 对象，**不是 `nil`**。
    *   `View A` 的代码执行到 `if (hitTestView != nil)`，判断为真。
    *   **于是 `View A` 的 `hitTest:` 立刻 `return hitTestView;`**，把 `Button C` 对象继续向上传递。

7.  **最终结果**
    *   最后，`UIWindow` 的 `hitTest:` 方法收到了 `[View A hitTest:...]` 的返回值，即 `Button C` 对象。
    *   `UIWindow` 也将这个对象作为最终结果返回。

至此，整个 Hit-Testing 过程结束。系统确定了**红色按钮 C**是这次触摸事件的最佳响应者（Hit-Test View）。

好的，iOS的事件触摸与传递过程是面试中的一个核心考点。它分为两个主要阶段：

1.  **Hit-Testing (寻找过程)**：系统通过这个阶段来找到最适合响应触摸事件的那个视图（View）。这是一个**从父视图到子视图**的递归查找过程。
2.  **Responder Chain (响应过程)**：在找到最适合的视图后，事件会沿着**响应者链**传递，让链上的各个响应者对象都有机会处理这个事件。这是一个**从子视图到父视图**的传递过程。

下面我们详细分解这两个过程。

---

### 第一阶段：Hit-Testing (寻找最佳响应者)

当用户触摸屏幕时，系统会产生一个触摸事件（`UIEvent`），并将其加入到当前应用的事件队列中。`UIApplication`会从队列中取出该事件，然后开始寻找处理它的最佳视图。

这个过程的核心方法是 `hitTest:withEvent:`。

#### Hit-Testing 算法流程

1.  **起点**：`UIApplication`将事件传递给`UIWindow`。
2.  **调用 `hitTest:`**：`UIWindow`调用自己的`hitTest:withEvent:`方法。

`hitTest:withEvent:`方法的内部逻辑大致如下：

**Step A: 检查自身能否响应事件**

*   首先，它会调用`pointInside:withEvent:`方法判断触摸点是否在当前视图的坐标范围内。
*   如果`pointInside:`返回`NO`（触摸点不在视图内），那么`hitTest:`直接返回`nil`，表示自己和所有子视图都无法响应。
*   如果`pointInside:`返回`YES`，则继续下一步。

**Step B: 检查子视图**

*   它会**从后往前**（也就是从最上层的子视图到最下层的子视图）遍历自己的所有子视图。
*   对于每一个子视图，它会先将触摸点的坐标从当前视图转换到子视图的坐标系中，然后以新的坐标点调用该子视图的`hitTest:withEvent:`方法。
*   **递归**：这个过程会一直递归下去，直到最深层的视图。

**Step C: 返回结果**

*   如果在遍历子视图的过程中，有一个子视图的`hitTest:`方法返回了一个非`nil`的视图对象，那么当前视图的`hitTest:`方法就会**立即停止遍历**，并直接将这个对象返回。这个返回的视图就是所谓的“最佳响应者”（Hit-Test View）。
*   如果遍历完所有子视图，它们的`hitTest:`方法都返回`nil`，那么就意味着没有子视图能响应这个事件。由于在Step A中已经确认了触摸点在当前视图内，所以当前视图就成为最佳响应者，`hitTest:`方法会返回`self`。

#### 哪些视图会被忽略？

在Hit-Testing过程中，如果一个视图满足以下任何一个条件，它和它的所有子视图都会被直接忽略：

*   `userInteractionEnabled = NO` (用户交互关闭)
*   `hidden = YES` (视图隐藏)
*   `alpha < 0.01` (视图基本全透明)

**总结：Hit-Testing就是一个深度优先的递归搜索，目的是找到包含触摸点且在视图层级中最深、最上层的那个视图。**

---

### 第二阶段：Responder Chain (响应者链的事件传递)

一旦Hit-Testing过程找到了最佳响应者（Hit-Test View），事件就会被传递给它，然后事件就进入了响应者链的传递阶段。

#### 什么是响应者链 (Responder Chain)？

响应者链是由一系列能响应事件的`UIResponder`对象组成的链条。`UIView`、`UIViewController`、`UIWindow`、`UIApplication`都是`UIResponder`的子类。

每个`UIResponder`对象都有一个`nextResponder`属性，指向链中的下一个响应者。

#### 事件传递流程

1.  **首个响应者处理**：系统首先将事件（通过`touchesBegan:withEvent:`等方法）传递给Hit-Testing找到的那个**最佳响应者（First Responder）**。

2.  **响应或传递**：
    *   如果这个First Responder实现了相应的触摸事件处理方法（如`touchesBegan:withEvent:`），它就可以处理这个事件。处理完后，事件传递就此结束。
    *   如果它想让其他响应者也处理，可以调用`[super touchesBegan:withEvent:]`。
    *   如果它**没有实现**相应的处理方法，或者在实现中明确调用了`super`的同名方法，那么事件就会沿着`nextResponder`属性，传递给链中的**下一个响应者**。

#### 响应者链的默认传递路径

事件的传递路径通常如下：

1.  **UIView**: 如果一个视图是First Responder，它的`nextResponder`是：
    *   如果它是某个`UIViewController`的根视图（`self.view`），那么下一个响应者就是这个`UIViewController`。
    *   否则，它的下一个响应者就是它的**父视图（superview）**。

2.  **UIViewController**: 它的`nextResponder`是它的根视图的父视图（`self.view.superview`）。

3.  **UIWindow**: 它的`nextResponder`是`UIApplication`。

4.  **UIApplication**: 它的`nextResponder`是`AppDelegate`（如果`AppDelegate`继承自`UIResponder`且不是视图、视图控制器或应用本身）。

5.  **最终**：如果事件一路传递到`UIApplication`都无人处理，那么这个事件就会被系统丢弃。

### 图解总结

```
触摸发生
   │
   ▼
┌─────────────────────────┐
│   Phase 1: Hit-Testing (从上到下)   │
│   UIWindow -> Superview -> Subview  │
│   (调用 hitTest:withEvent:)       │
└─────────────────────────┘
   │
   ▼
 [ 找到了最合适的视图 (Hit-Test View) ]
   │
   ▼
┌─────────────────────────┐
│  Phase 2: Responder Chain (从下到上)  │
│ Hit-Test View -> Superview -> UIViewController -> UIWindow -> UIApplication
│   (调用 touchesBegan: 等方法, 沿 nextResponder 传递) │
└─────────────────────────┘
   │
   ▼
 事件被处理或被丢弃
```



---

### 事件响应的三种“拦截”方式

当一个触摸事件（`UIEvent`）通过 **Hit-Testing** 过程找到了最合适的响应视图（我们称之为 Hit-Test View）后，系统会把这个事件交给这个视图，然后就开始了**响应链（Responder Chain）**的旅程。

在这个旅程中，有三道主要的“关卡”可以拦截并处理这个事件。它们的**优先级**通常是：

**第一优先级：手势识别器 (UIGestureRecognizer)**
**第二优先级：`touchesBegan:` / `touchesMoved:` / `touchesEnded:` 系列方法**
**第三优先级：`addTarget:` 方式 (只适用于 `UIControl` 及其子类)**

让我们来详细分析每一道关卡。

---

### 第一道关卡 (最高优先级)：手势识别器

在系统准备调用 Hit-Test View 的 `touchesBegan:` 方法**之前**，它会先做一个非常重要的检查：**这个视图以及它的父视图上，有没有附加手势识别器？**

1.  **手势的“偷窥权”**:
    *   `UIWindow` 在分发触摸事件时，会首先将事件“喂”给所有附加在视图层级中的手势识别器。这给了手势一个“**优先窥探**”事件的机会。
    *   手势识别器会分析这个触摸事件序列（Began, Moved, Ended），判断它是否符合自己定义的模式（比如单击、双击、长按、拖动）。

2.  **手势成功识别 (状态变为 `Recognized` / `Ended`)**:
    *   一旦某个手势（比如 `UITapGestureRecognizer`）成功识别出了一个单击手势。
    *   **默认行为**：
        1.  它会立即向它的 `target` 发送 `action` 消息。
        2.  为了不让事件“一事二主”，手势会“**吞掉**”这个事件。它会向视图发送 `touchesCancelled:` 消息，**阻止**视图的 `touchesBegan:` / `touchesEnded:` 方法被调用。
        3.  因为 `touches` 系列方法没有被调用，所以事件自然**不会**再沿着响应者链向上传递。

3.  **手势识别失败**:
    *   如果手势分析了触摸事件后，认为不符合它的模式（比如用户是拖动而不是单击），它的状态会变为 `Failed`。
    *   此时，手势会“放开”这个事件，触摸事件会继续它正常的旅程，去调用视图的 `touchesBegan:` 等方法。

**结论**: **只要一个手势成功识别了事件，默认情况下，`touchesBegan:` 就不会被调用，响应者链的传递也就被“截胡”了。**

*   **可以改变这个默认行为吗？** 可以。`UIGestureRecognizer` 有一个 `cancelsTouchesInView` 属性，如果设置为 `NO`，那么即使手势成功识别，`touches` 系列方法也依然会被调用。

---

### 第二道关卡：重写 `touches` 系列方法

如果一个视图上没有手势，或者手势识别失败了，那么事件就会来到第二道关卡。

1.  Hit-Test View 的 `touchesBegan:withEvent:` 方法会被调用。
2.  **默认行为**:
    *   如果你**重写**了这个方法，并在里面处理了事件逻辑，但**没有**调用 `[super touchesBegan:withEvent:]`。
    *   那么，你就“**消费**”了这个事件。
    *   事件的传递就此**终止**，它不会再被传递给视图的 `nextResponder`（比如它的父视图或视图控制器）。

3.  **传递事件**:
    *   如果你在方法的最后调用了 `[super touchesBegan:withEvent:]`。
    *   那么你就明确地告诉系统：“我的事办完了，现在把这个事件交给我的上级（`nextResponder`）去处理吧。”
    *   事件就会继续沿着响应者链向上传递。

**结论**: **重写 `touchesBegan:` 并处理事件（且不调用 `super`），确实会阻止事件向父视图传递。**

---

### 第三道关卡：`UIControl` 的 `addTarget:` 机制

这道关卡比较特殊，它只适用于 `UIButton`, `UISlider`, `UITextField` 等继承自 `UIControl` 的控件。

`UIControl` 内部封装了对 `touches` 系列方法的处理，它自己实现了一套完整的触摸跟踪和事件分发机制。

1.  **内部实现**:
    *   当一个 `UIButton` 收到 `touchesEnded:` 事件时，它会在内部判断这次触摸是否是一次有效的“点击”（比如手指是否在按钮范围内抬起）。
    *   如果是有效点击，它就会触发 `UIControlEvents.touchUpInside` 事件。
    *   然后，它会查找所有通过 `addTarget:action:forControlEvents:` 添加的 `target-action` 对，并向它们发送消息。

2.  **与响应者链的关系**:
    *   `UIControl` 在处理完自己的 `target-action` 逻辑后，**默认不会再将事件传递给它的 `super`**。它认为自己作为专门的交互控件，已经完成了事件的最终处理。
    *   因此，当你点击一个按钮时，它的 `target-action` 会被触发，但通常情况下，这个按钮的父视图的 `touchesBegan:` 是不会被调用的。

**结论**: **是的，`addTarget:` 响应了事件后，默认也不会再将触摸事件往父视图抛。**

## `nil`、`NIL`、`NSNULL` 有什么区别？

好的，这是一个在 Objective-C 面试中几乎必考的基础题，它能很好地考察面试者对C语言指针、Objective-C对象以及Foundation框架细节的理解。

这三者的核心区别在于它们的**类型**和**用途**。

---

### 1. `nil`

*   **类型**: `(id)0`
*   **本质**: 一个指向 **Objective-C 对象** 的**空指针**。它是一个对象指针类型的字面量 `0`。
*   **用途**:
    *   主要用于表示一个 **Objective-C 对象的指针不指向任何对象**。
    *   在 Objective-C 中，向 `nil` 发送任何消息都是**完全合法**的，并且不会产生任何效果，也不会导致程序崩溃。返回值根据方法的返回类型而定（对象类型返回 `nil`，数值类型返回 `0`，`BOOL` 返回 `NO`，结构体返回全 `0` 的结构体）。这是一个非常重要的特性，可以大大简化代码，避免大量的 `if (obj != nil)` 判断。

**示例**:
```objectivec
NSString *myString = nil;
if (myString) {
    // 这段代码不会执行
}

// 向 nil 发送消息是安全的
NSUInteger length = [myString length]; // length 的值会是 0
NSLog(@"Length is: %lu", (unsigned long)length);
```

**记忆点**: **`nil` 是给 Objective-C 对象指针用的“空”。**

---

### 2. `NULL`

*   **类型**: `(void *)0`
*   **本质**: 一个通用的**C语言空指针**。它是一个泛型指针，可以被转换成任何类型的指针。
*   **用途**:
    *   主要用于 C 语言环境下的各种指针类型，包括但不限于 `char *`, `int *`, 以及 Core Foundation 和 Core Graphics 中的 C 语言风格对象指针（如 `CGContextRef`, `CFArrayRef`）。

**示例**:
```c
// C 语言字符串
char *myCString = NULL;

// Core Graphics 上下文
CGContextRef context = NULL;

// 如果尝试把 NULL 赋值给 OC 对象指针，虽然编译器可能不报错，
// 但在语义上是不规范的，应该使用 nil。
// 不推荐的写法: NSString *str = NULL;
// 推荐的写法: NSString *str = nil;
```

**记忆点**: **`NULL` 是给 C 语言指针用的“空”。**

---

### 3. `NSNull`

*   **类型**: `NSNull *` (一个 Objective-C 类)
*   **本质**: **一个真正的 Objective-C 对象**，它是一个单例类 `NSNull` 的唯一实例。
*   **用途**:
    *   它的存在是为了解决一个特殊的问题：像 `NSArray`、`NSDictionary` 这样的 **Foundation 集合类不能包含 `nil` 值**。如果你尝试向一个 `NSMutableArray` 中添加 `nil`，或者用一个 `nil` 值创建一个 `NSDictionary`，程序会立即崩溃。
    *   但是，在很多业务场景中，我们需要在集合中表示一个“空”或“缺失”的值（比如服务器返回的JSON中某个字段的值是 `null`）。
    *   `NSNull` 就是用来扮演这个**占位符 (placeholder)** 角色的。它是一个合法的、非 `nil` 的对象，可以被安全地添加到集合中，用来在语义上表示“这里应该有一个值，但现在是空的”。

**示例**:
```objectivec
// 错误的做法，会导致崩溃！
// NSArray *myArray = @[@"Apple", nil, @"Banana"]; 
// [mutableDict setObject:nil forKey:@"middleName"];

// 正确的做法
// 获取 NSNull 的单例实例
NSNull *nullObject = [NSNull null];

// 创建一个包含“空值”的数组和字典
NSArray *myArray = @[@"Apple", nullObject, @"Banana"];
NSDictionary *myDict = @{
    @"firstName": @"John",
    @"middleName": nullObject, // 使用 NSNull 作为占位符
    @"lastName": @"Doe"
};

// 从集合中取值时，需要判断取出的对象是否是 NSNull
id middleName = myDict[@"middleName"];
if (middleName == [NSNull null]) {
    NSLog(@"Middle name is missing.");
}
```

**记忆点**: **`NSNull` 是一个表示“空值”的 Objective-C 对象，专门用于填充集合类。**

---

### `PerformSelector` 的使用和实现原理

# 计网

## ipv4和ipv6有什么区别

位数不同，ipv4是32个bit，ipv6是128个bit

## session和cookie的区别

## 微信语音通话，有一端网络不好时，是怎么确定哪一端的

## dns劫持

### 二、DNS 劫持的技术实现方式

DNS 劫持可以发生在从你的电脑到根域名服务器的任何一个环节，但最常见的劫持发生在以下几个地方：

#### 1. 本地劫持

*   **恶意软件/病毒**: 电脑中毒后，恶意软件可以直接修改你电脑的 `hosts` 文件。这个文件的优先级高于 DNS 查询，可以强制将某个域名指向一个恶意 IP。
*   **修改 DNS 设置**: 恶意软件也可以直接修改你操作系统的网络设置，将你的首选 DNS 服务器地址改成一个由黑客控制的恶意 DNS 服务器地址。

#### 2. 路由器劫持

*   这是非常常见的一种家庭网络攻击。黑客通过破解你家用路由器的管理密码，登录到路由器后台。
*   然后，他修改路由器 DHCP 服务配置中的 DNS 服务器地址。
*   这样一来，所有连接到这个路由器的设备（你的手机、电脑、iPad），在自动获取网络配置时，都会被**强制分配一个恶意的 DNS 服务器**。你访问任何网站，都会先去问这个恶意服务器，导致大范围的劫持。

#### 3. DNS 缓存投毒 (DNS Cache Poisoning)

*   这是一个更高级的攻击方式，目标是**运营商的 Local DNS 服务器**。
*   当一个 Local DNS 服务器收到一个它不知道的域名查询时（比如 `www.mybank.com`），它会去问根服务器、顶级域服务器等，进行递归查询。
*   黑客会利用 DNS 协议的一些漏洞，抢在真正的应答回来之前，**伪造一个错误的应答**发给这个 Local DNS 服务器。
*   如果这个 Local DNS 服务器“中毒”了，它就会把这个错误的“域名-IP”映射关系**缓存**起来。
*   其后果是，所有使用这个运营商 Local DNS 的用户（可能是一个城市、一个省的用户），在缓存过期前，访问 `www.mybank.com` 都会被导向那个恶意的 IP 地址，造成大规模影响。

---

### 三、DNS 劫持的危害

1.  **网页钓鱼 (Phishing)**:
    *   将你访问的银行、电商网站（如 `www.icbc.com.cn`）劫持到一个**一模一样的假网站**上。
    *   你在这个假网站上输入的用户名、密码、银行卡号等敏感信息，会全部被黑客窃取。

2.  **弹窗广告 / 流量劫持**:
    *   把你访问的任何网页都劫持到一个中间服务器上。
    *   这个服务器在返回给你正常的网页内容之前，会**强行插入一段广告代码**，导致你的浏览器右下角疯狂弹窗。或者将网页上的正常链接替换成他们自己的推广链接，赚取佣金。

3.  **无法访问特定网站**:
    *   攻击者可以将一个域名（比如 `www.google.com`）直接解析到一个无效的、无法访问的 IP 地址上，导致你无法打开这个网站。

4.  **中间人攻击 (Man-in-the-Middle)**:
    *   这是最危险的情况。流量被导向一个黑客控制的代理服务器。这个服务器可以解密你的非 HTTPS 流量，或者像我们之前讨论的那样，通过伪造证书来尝试解密 HTTPS 流量，窃取所有通信内容。

---

### 四、如何防范 DNS 劫持

作为普通用户，我们可以采取以下措施：

1.  **使用公共 DNS / HTTPDNS**:
    *   **修改 DNS 服务器地址**: 不使用运营商默认分配的 DNS，手动将你的电脑或路由器的 DNS 地址修改为更可靠的公共 DNS，比如：
        *   **Cloudflare DNS**: `1.1.1.1` 和 `1.0.0.1`
        *   **Google DNS**: `8.8.8.8` 和 `8.8.4.4`
        *   **国内推荐**: `119.29.29.29` (DNSPod) 或 `223.5.5.5` (AliDNS)
    *   **使用 HTTPDNS (DNS over HTTPS - DoH)**: 这是更先进的防范手段。它将 DNS 查询本身通过 HTTPS 加密通道进行，让中间的任何环节都无法窃听或篡改查询内容。现代的浏览器（如 Chrome, Firefox）和操作系统（Windows 11, iOS 14+）都已支持 DoH。

2.  **加强路由器安全**:
    *   **修改默认密码**: 一定要修改路由器的管理员登录密码，不要使用 `admin/admin` 这样的弱密码。
    *   **关闭远程管理**: 关闭从外网访问路由器后台的功能。
    *   **及时更新固件**: 保持路由器固件是最新版本，修复已知的安全漏洞。

3.  **注意 HTTPS**:
    *   访问重要网站时，**务必确认网址是以 `https://` 开头，并且浏览器地址栏显示了安全锁标志**。虽然 DNS 劫持可以把你导向一个假的 HTTP 网站，但对于 HTTPS 网站，由于证书的存在，劫持会变得更加困难，浏览器通常会给出非常明显的安全警告。

4.  **安装安全软件**:
    *   保持杀毒软件是最新状态，可以有效防止恶意软件对本地 `hosts` 文件和 DNS 设置的篡改。

通过这些方法，可以极大地降低你被 DNS 劫持的风险。

## 5层网络分别是什么，讲讲每层什么协议，有什么东西

## 子网掩码的作用

## 当客户端宕机了，服务端如何判断是否要断开这个连接

1. **首先，也是最常用的，是应用层超时。** 服务器程序会自己维护一个计时器，比如设置一个60秒的空闲超时。如果在60秒内没有收到客户端的任何数据，就主动调用close()来关闭连接，回收资源。这种方式非常灵活和及时。
2. **其次，还可以依赖TCP协议本身的Keep-Alive机制。** 我们可以开启这个选项，当连接长时间（默认2小时）没有数据时，TCP层会自动发送探测包。如果多次探测都失败，内核就会自动判定连接失效并关闭它。但因为默认时间太长，它通常只作为一种兜底的保险措施。”

## 内存对齐解释下

## https详细解释

## get和post区别

* get请求资源，post发送资源，一个有请求体一个没有
* get天然幂等，post不是幂等的
* get结果可以被缓存，post一般不缓存

## ***应用层有哪些协议

* HTTP、HTTPS：网页浏览
* FTP：文件传输
* SMTP：邮件传输
* DNS：解析域名的ip地址
* CDN：

## ***HTTP 状态码

* HTTP 状态码分为五个类别，分别以 1、2、3、4、5 开头，代表信息性、成功、重定向、客户端错误和服务器错误。

* 1xx 状态码表示请求已被接收，客户端可以继续请求。比如：100 Continue。在 2xx 成功状态码中，最常见的是 **200 OK**，表示请求成功并返回数据；**201 Created** 表示服务器成功创建了新的资源。在 3xx 重定向中，**301 Moved Permanently 用于永久重定向**，**302 Found** 用于临时重定向。**4xx** 是客户端错误，比如 400 Bad Request 表示请求格式错误，**401 Unauthorized 表示未授权**，**403 Forbidden 表示禁止访问**，最常见的 **404 Not Found** 表示资源不存在。5xx 是服务器错误，比如 500 Internal Server Error 表示服务器内部错误，503 Service Unavailable 表示服务暂时不可用。

## HTTP的长连接是什么？

## HTTP为什么不安全？

## ***tcp/ip网络模型

* **应用层**:提供用户直接使用的网络服务，如网页浏览、文件传输等。（HTTP（网页）、FTP（文件传输）、DNS（域名解析）

* **传输层**：端到端的数据传输服务。TCP 协议提供可靠的、面向连接的服务，而 UDP 协议提供不可靠的、无连接的服务。
* **网络层**：主要处理 IP 协议，负责数据包的路由和寻址，实现不同网络之间的数据包转发。
* **网络接口层**：负责在同一局域网内传输数据，比如网络包的封帧、 MAC 寻址。

## tcp头部有哪些字段

---

### 字段详解

#### 1. 源端口 (Source Port) - 16位
*   **作用**: 标识发送方应用程序的端口号。
*   **好比**: 运单上的“发件人地址门牌号”。
*   **例子**: 当你的浏览器访问网页时，操作系统会为浏览器分配一个临时的、通常大于1023的端口号（如 54321），这就是源端口。

#### 2. 目的端口 (Destination Port) - 16位
*   **作用**: 标识接收方应用程序的端口号。
*   **好比**: 运单上的“收件人地址门牌号”。
*   **例子**: 浏览器访问一个 HTTP 网站，目的端口就是 80；访问一个 HTTPS 网站，目的端口就是 443。

#### 3. 序列号 (Sequence Number) - 32位
*   **作用**: 这是TCP可靠性的基石之一。它为发送的每一个字节都编一个号。这个字段的值是当前数据包中**第一个字节的编号**。
*   **好比**: 一本书的“页码”。它告诉接收方，这份数据是书里的第几页，以便接收方能按正确顺序重组。
*   **例子**: 如果一个数据包的序列号是 300，它携带了100字节的数据，那么下一个数据包的序列号就应该是 400。

#### 4. 确认号 (Acknowledgment Number) - 32位
*   **作用**: 这是TCP可靠性的另一个基石。它告诉发送方，我**期望接收的下一个字节的序列号是多少**。这隐含地确认了在此之前的所有字节都已成功接收。
*   **好比**: 你读完书的第399页后，告诉作者：“我已读完399页，请给我第400页。”
*   **注意**: 这个字段只有在下面的 `ACK` 标志位被设置为1时才有效。

#### 5. 数据偏移 (Data Offset) - 4位
*   **作用**: 指明 TCP 头部有多长，也就是数据部分从哪里开始。
*   **好比**: 运单上注明：“运单本身有2页，货物清单从第3页开始。”
*   **单位**: 它的单位是 **32位字（4字节）**。所以一个值为 `5` 的数据偏移意味着头部长度是 `5 * 4 = 20` 字节（这是最小的头部长度）。最大值为 `15`，意味着头部最长为 `15 * 4 = 60` 字节。

#### 6. 保留 (Reserved) - 3位
*   **作用**: 为未来使用而保留的字段，目前必须全部设置为0。

#### 7. 标志位 (Flags) - 9位
*   **作用**: 这是一些开关，用来控制TCP连接的状态。
*   **好比**: 运单上的一排复选框，如“加急”、“需要签收”、“易碎品”等。
    *   **NS (Nonce Sum)**: ECN（显式拥塞通知）的扩展，用于防止意外或恶意的拥塞反馈。
    *   **CWR (Congestion Window Reduced)**: 发送方在降低其发送速率后，用此标志通知对方。
    *   **ECE (ECN-Echo)**: 表示收到了一个拥塞信号。
    *   **URG (Urgent)**: 表示“紧急指针”字段有效，有紧急数据需要优先处理（现代应用中很少使用）。
    *   **ACK (Acknowledgment)**: 表示“确认号”字段有效。在连接建立后，几乎所有的数据包都会置1。
    *   **PSH (Push)**: 催促接收方尽快将数据交给应用程序，不要为了提高网络效率而等待填满缓冲区（例如，在SSH这种实时交互应用中）。
    *   **RST (Reset)**: 异常关闭连接。用于响应无效的数据包或重置一个混乱的连接。
    *   **SYN (Synchronize)**: 在建立连接的“三次握手”过程中，第一个数据包会置1，用于同步序列号。
    *   **FIN (Finish)**: 正常关闭连接。表示发送方已经没有数据要发送了。

#### 8. 窗口大小 (Window Size) - 16位
*   **作用**: 用于流量控制。它告诉对方，我本地的接收缓冲区还剩下多大空间，你这次最多可以发这么多数据给我，再多我就处理不过来了。
*   **好比**: 你告诉快递员：“我的储物柜还剩2立方米的空间，你下次送货别超过这个体积。”

#### 9. 校验和 (Checksum) - 16位
*   **作用**: 用于检查数据在传输过程中是否出错。发送方会根据头部和数据内容计算一个值；接收方用同样的方法再算一遍，如果两个值不匹配，说明数据已损坏，这个数据包会被丢弃。
*   **好比**: 运单上的“物品数量和总重量”，收件人可以核对一下，看看中途有没有丢失或损坏。

#### 10. 紧急指针 (Urgent Pointer) - 16位
*   **作用**: 只有当 `URG` 标志位置1时才有效。它是一个偏移量，和序列号相加，指向紧急数据的最后一个字节。

#### 11. 选项 (Options) - 可变长度 (0-40字节)
*   **作用**: 提供了对TCP协议的扩展能力。
*   **好比**: 运单上的“备注”栏。
*   **常见选项**:
    *   **Maximum Segment Size (MSS)**: 在建立连接时，双方协商每个TCP数据包能携带的最大数据量。
    *   **Window Scale**: 用于扩大“窗口大小”字段的表示范围，以适应高速网络。
    *   **Timestamps**: 用于更精确地计算往返时间（RTT）和防止序列号回绕问题。

#### 12. 填充 (Padding)
*   **作用**: 因为“选项”字段的长度是可变的，所以需要用0来填充，以确保整个TCP头部的总长度是32位（4字节）的整数倍。

通过这些精心设计的字段，TCP才得以实现其著名的可靠数据传输、流量控制和拥塞控制等高级功能。

## IP报文的头部有哪些

---

### **IPv4 头部结构 (The Classic)**

IPv4的头部是一个**20到60字节**的变长结构。可以把它想象成快递包裹上的“面单”，上面写满了包裹的路由信息。这个面单分为**固定部分（20字节）**和**可选部分（最多40字节）**。固定部分 (前20字节，必须存在)**

1.  **版本 (Version, 4位)**:
    *   **作用**: 指明IP协议的版本。对于IPv4，这个值永远是**4** (二进制`0100`)。
    *   **意义**: 让路由器知道该如何解析这个报文的其余部分。
2.  **首部长度 (IHL - Internet Header Length, 4位)**:
    *   **作用**: 表示整个IP头部的长度。
    *   **单位**: 4字节。它的值乘以4，就是头部真正的字节数。
    *   **范围**: 最小值为**5** (二进制`0101`)，代表 `5 * 4 = 20` 字节（即没有可选部分）。最大值为**15** (二进制`1111`)，代表 `15 * 4 = 60` 字节。
4.  **总长度 (Total Length, 16位)**:
    *   **作用**: 表示整个IP数据包的**总长度**，包括头部和数据部分。
    *   **单位**: 字节。最大长度为 2^16 - 1 = 65535字节。
5.  **标识 (Identification, 16位)**:
    *   **作用**: 用于**分片与重组**。当一个IP包因为太大而需要被分割成多个小片段（分片）时，所有这些小片段都拥有**相同**的标识号。
    *   **意义**: 帮助目标主机识别哪些小片段属于同一个原始数据包，以便将它们重新组合起来。
8.  **生存时间 (Time to Live - TTL, 8位)**:
    *   **作用**: 防止数据包在网络中**无限循环**。
    *   **工作方式**: 数据包每经过一个路由器，该路由器就会将TTL的值**减1**。当TTL变为**0**时，路由器会丢弃这个包。
    *   **意义**: 这是网络诊断工具 `traceroute` 的工作原理基础。
9.  **协议 (Protocol, 8位)**:
    *   **作用**: 指明IP包的**数据部分**承载的是哪种**上层协议**。
    *   **常见值**: **6** 代表 **TCP**，**17** 代表 **UDP**，**1** 代表 **ICMP** (Ping命令用的协议)。
    *   **意义**: 告诉目标主机的操作系统，应该把这个数据包交给哪个程序来处理（比如，是交给处理TCP的模块还是处理UDP的模块）。
10. **首部校验和 (Header Checksum, 16位)**:
    *   **作用**: 用于检验IP**头部**在传输过程中是否发生了损坏。
    *   **工作方式**: 发送方计算头部校验和并填充，路由器每修改TTL后都需要**重新计算**并填充，接收方再次校验。如果不匹配，则丢弃该包。**注意：它只校验头部，不校验数据部分。**
11. **源IP地址 (Source Address, 32位)**:
    *   **作用**: 发送方的IP地址，比如 `192.168.1.100`。
12. **目的IP地址 (Destination Address, 32位)**:
    *   **作用**: 接收方的IP地址，比如 `8.8.8.8`。



## ？解释 DNS 的作用和工作原理。

## tcp的滑动窗口和拥塞控制

## 用户态和内核态是如何切换的？用户态和内核态的区别？

好的，这是一个操作系统核心中的核心概念。用户态和内核态的切换，是现代操作系统为了**保护系统安全和稳定**而设计的基石。

我将用一个银行办理业务的比喻来帮助你直观理解，然后深入到技术细节，解释三种主要的切换方式。

### 一、核心思想的比喻：客户与银行柜员

想象一下，你（**用户程序**）要去银行办理一项业务，比如取一大笔现金。

*   **用户态 (你在大厅)**: 你在银行的大厅里，这里是客户区域。你可以自由地做很多事情，比如填表、排队、看宣传册。这些操作都是**安全**的，不会影响到银行金库的核心安全。**这是用户程序在自己的内存空间里运行，执行普通指令的状态。**

*   **内核态 (柜员在柜台内)**: 当你需要办理取款业务时，你不能自己冲进柜台后面的工作区，更不能自己去打开金库。你必须通过一个**指定的窗口**，把你的申请单（比如身份证和取款单）递给银行柜员（**操作系统内核**）。
    *   柜员在柜台后面的工作区，拥有**极高的权限**。他可以操作点钞机、访问储户数据库、甚至进入金库。
    *   他会严格地**审查**你的请求是否合法（身份是否正确，余额是否足够）。
    *   如果合法，他会帮你完成操作，然后把现金通过窗口递还给你。
    *   操作完成后，柜员就回到他的工作状态，而你拿着钱回到了大厅。

**这个从“递交申请”到“拿回结果”的过程，就是一次从用户态到内核态，再回到用户态的切换。**

**关键点：**
1.  **权限隔离**: 你（用户程序）永远无法直接操作核心资源（硬件、内核数据）。
2.  **受控入口**: 你只能通过操作系统提供的**唯一、合法的入口**（系统调用）来请求服务。
3.  **模式切换**: 当内核开始为你服务时，CPU 的“身份”就从“客户代表”切换成了“银行职员”，权限等级提升。

---

### 二、技术上的实现：CPU 的特权级别

现代 CPU 自身就提供了硬件级别的支持，通常被称为**特权级别 (Privilege Levels)** 或**环 (Rings)**。

*   **Ring 0 (内核态)**: 权限最高。操作系统内核就运行在这一层。它可以执行所有 CPU 指令，访问所有内存和硬件设备。
*   **Ring 3 (用户态)**: 权限最低。普通的用户应用程序运行在这一层。它只能执行一部分“安全”的指令，只能访问被操作系统分配给它自己的虚拟内存空间。

**用户态到内核态的切换，本质上就是 CPU 的执行状态从 Ring 3 切换到 Ring 0 的过程。** 这个切换不是通过一个简单的函数调用完成的，它必须通过一个特殊的、由硬件支持的指令，这个指令就像是银行柜台上的那个“窗口”。

---

### 三、三种主要的切换方式

从用户态切换到内核态，主要有以下三种途径。这些途径都是由**用户程序无法预测或控制的事件**触发的。

#### 1. 系统调用 (System Call) - 最主要、最主动的方式

这是用户程序为了获得内核的服务而**主动**发起的切换。

*   **触发场景**: 当你的程序需要执行一些它没有权限做的事情时，就必须请求内核帮忙。比如：
    *   **文件操作**: 打开、读取、写入文件 (`open`, `read`, `write`)。用户程序不能直接操作硬盘。
    *   **网络通信**: 创建套接字、发送、接收数据 (`socket`, `send`, `recv`)。用户程序不能直接控制网卡。
    *   **内存管理**: 申请更多内存 (`mmap`, `brk`)。
    *   **进程控制**: 创建新进程 (`fork`)、结束进程 (`exit`)。

*   **切换过程**:
    1.  **用户态**: 应用程序调用一个库函数（如 C 库中的 `printf`）。
    2.  **陷入 (Trap)**: 这个库函数内部会准备好系统调用所需的参数（比如要写入的数据、文件描述符等），然后执行一条特殊的 CPU 指令，比如 `int 0x80` (在 x86 早期) 或 `syscall` (在 x86-64)。
    3.  这条指令会触发一个**软件中断**，使 CPU **立即暂停**当前的用户代码执行。
    4.  CPU 将当前的执行状态（如程序计数器、栈指针等）保存起来，然后根据中断向量表，跳转到操作系统内核中预设好的**中断处理程序**。同时，CPU 的特权级别从 Ring 3 **切换到 Ring 0**。
    5.  **内核态**: 内核的中断处理程序接管控制权。它会检查参数的合法性，然后执行相应的内核代码来完成用户的请求（比如把数据写入到与显示器相关的内核缓冲区）。
    6.  **返回**: 内核服务完成后，会执行另一条特殊的指令（如 `iret` 或 `sysret`），将之前保存的用户态上下文恢复，并将 CPU 的特权级别**从 Ring 0 切换回 Ring 3**。
    7.  **用户态**: 用户程序从刚才被中断的地方继续执行，就好像什么都没发生过一样，只是它请求的操作已经完成了。

#### 2. 中断 (Interrupt) - 来自外部硬件的“敲门”

当中断发生时，CPU 必须立即停止当前的工作（无论是在用户态还是内核态），去处理这个中断。如果当时正在用户态，这就导致了一次被动的切换。

*   **触发场景**: 来自外部硬件设备的信号，需要操作系统立即处理。
    *   **键盘输入**: 你在键盘上按下一个键。
    *   **鼠标移动**: 你移动或点击鼠标。
    *   **网卡收到数据**: 网络上传来一个数据包。
    *   **定时器到期**: 操作系统的调度器需要用定时器中断来切换不同的进程。

*   **切换过程**: 和系统调用非常相似。
    1.  **用户态**: 你的程序正在愉快地运行。
    2.  **中断信号**: 硬件（如网卡）向 CPU 发送一个中断信号。
    3.  CPU 收到信号后，**立即暂停**当前用户代码，保存现场，然后根据中断号跳转到内核中对应的**中断服务程序**，并将特权级别切换到 Ring 0。
    4.  **内核态**: 内核处理这个硬件事件（比如把网卡收到的数据拷贝到内核缓冲区）。
    5.  **返回**: 处理完毕后，恢复现场，切换回 Ring 3，用户程序继续执行。

#### 3. 异常 (Exception) - 程序自己“犯错”了

当用户程序在执行过程中，发生了 CPU 无法处理的错误或特殊情况时，会触发异常。

*   **触发场景**:
    *   **缺页异常 (Page Fault)**: 程序试图访问一个有效的虚拟地址，但该地址对应的数据目前不在物理内存中（可能被换到磁盘上了）。这是一个非常常见且正常的异常。
    *   **除以零**: 执行了数学上非法的操作。
    *   **非法指令**: 试图执行一个 CPU 不认识的指令。
    *   **段错误 (Segmentation Fault)**: 试图访问一个它无权访问的内存地址（比如访问内核空间或空指针）。

*   **切换过程**:
    1.  **用户态**: 程序执行到一条错误指令（如 `mov [0x0], eax`）。
    2.  CPU 检测到这个错误，无法继续执行，于是触发一个内部的**异常**。
    3.  和中断类似，CPU 暂停程序，保存现场，跳转到内核中对应的**异常处理程序**，切换到 Ring 0。
    4.  **内核态**: 内核分析异常的类型。
        *   如果是**缺页异常**，内核会去磁盘把数据加载到内存，然后返回用户态让程序重试这条指令。
        *   如果是**严重的错误**（如段错误），内核会认为这个程序已经不可救药了，就会发送一个信号（如 `SIGSEGV`）来**终止**这个程序。
    5.  **返回/终止**: 根据处理结果，要么切换回用户态继续执行，要么直接终止用户进程。



## ping 的工作原理

好的，`ping` 是网络诊断中最基础、最常用的工具之一。它的工作原理非常简单、直接，但背后却完美地展示了网络分层模型的协作。

我将用一个非常生动的比喻，然后深入技术细节来为你彻底解释 `ping` 的工作原理。

### 一、核心思想的比喻：声纳探测

你可以把 `ping` 命令想象成潜水艇的**声纳探测**。

1.  **发出脉冲**: 你的电脑（潜水艇）向一个目标（比如另一艘潜艇 `google.com`）发出一个特定频率的声纳脉冲，并同时看一眼手表，记下当前时间。这个脉冲上写着：“喂，听得到吗？我是A，这是我的第1号脉冲。”
2.  **等待回声**: 你安静地等待。
3.  **收到回声**: 目标潜艇听到了你的脉冲，它并不会做什么复杂的分析，只是立刻、原封不动地产生一个“回声”，发回给你。这个回声上同样写着：“听到了，我是B，这是对你第1号脉冲的回应。”
4.  **计算结果**: 当你收到这个回声时，你再看一眼手表，用当前时间减去你发出脉冲的时间，就得到了**往返时间（Round-Trip Time, RTT）**。你就知道了：
    *   **目标是存在的**: 既然有回声，说明对方确实在那里。
    *   **距离有多远**: 往返时间越长，说明“距离”越远（网络延迟越高）。
    *   **信号是否丢失**: 如果你发出了10个脉冲，只收到了8个回声，你就知道了有20%的丢包率。

`ping` 的工作原理，就是这个过程的计算机网络版本。

---

### 二、技术核心：ICMP 协议

`ping` 的实现，完全依赖于一个叫做 **ICMP (Internet Control Message Protocol，互联网控制报文协议)** 的协议。

*   **ICMP 的角色**: ICMP 并不是用来传输用户数据的（不像 TCP 或 UDP），它是在 IP 网络中用于发送**控制消息**和**错误报告**的“信使”。它就像是网络的“勤务兵”或“诊断医生”。
*   **工作在哪一层**: ICMP 属于**网络层**协议，和 IP 协议是平级的“兄弟”关系。ICMP 报文是直接封装在 IP 数据包中进行传输的。

---

### 三、`ping` 的详细工作步骤

当你在命令行输入 `ping google.com` 时，背后会发生以下一系列精确的步骤：

#### **第1步：构造 ICMP 回显请求 (Echo Request)**

1.  `ping` 程序首先会通过 DNS 查询，将 `google.com` 这个域名解析成一个 IP 地址（比如 `172.217.160.78`）。
2.  然后，`ping` 程序会创建一个 **ICMP 回显请求（Echo Request）** 报文。这是一个非常简单的数据包，主要包含：
    *   **类型 (Type)**: 值为 **8**，代表这是一个 Echo Request。
    *   **代码 (Code)**: 值为 **0**。
    *   **校验和 (Checksum)**: 用于检查报文在传输中是否出错。
    *   **标识符 (Identifier)**: 用于区分同一台机器上可能同时进行的多个 `ping` 任务。
    *   **序列号 (Sequence Number)**: 从 0 或 1 开始，每发一个包就加 1。这就像声纳脉冲的编号，用于将请求和响应一一对应。
    *   **数据 (Data)**: 一段可选的、可以填充任意内容的数据，用于计算往返时间。

#### **第2步：封装成 IP 数据包**

这个构造好的 ICMP 报文，会被交给网络层的 IP 协议，像装进一个信封一样，被封装成一个 **IP 数据包**。
*   **IP 头部信息**:
    *   **源 IP 地址**: 你的电脑的 IP 地址。
    *   **目的 IP 地址**: `172.217.160.78` (google.com)。
    *   **协议号**: 值为 **1**，表示 IP 包里装的是 ICMP 报文。
    *   **TTL (Time To Live)**: 生存时间。一个整数，比如 64 或 128。每经过一个路由器，这个值就会减 1。

#### **第3步：发送与路由**

你的电脑将这个 IP 数据包发送出去。它会经过你家里的路由器、运营商的交换机、以及互联网上的多个路由器，一步步地向目标 IP 地址前进。每经过一个路由器，IP 包头里的 TTL 值都会被减 1。

#### **第4步：目标主机接收与处理**

1.  当 `google.com` 的服务器收到这个 IP 数据包后，它的网络协议栈会打开这个“信封”。
2.  它看到 IP 头里的协议号是 1，就知道里面装的是 ICMP 报文。
3.  它解析 ICMP 报文，发现类型是 8（Echo Request）。
4.  服务器的内核（操作系统）理解了这个请求的含义：“哦，有人在 ping 我，我需要回复一个回声。”

#### **第5步：构造 ICMP 回显应答 (Echo Reply) 并发回**

服务器会立刻构造一个 **ICMP 回显应答（Echo Reply）** 报文。
*   **类型 (Type)**: 值为 **0**，代表这是一个 Echo Reply。
*   **代码 (Code)**: 值为 **0**。
*   **标识符和序列号**: **完全复制**自收到的 Echo Request 报文。这是确保应答能与请求匹配上的关键。
*   **数据**: 也完全复制自收到的请求报文。

这个应答报文同样会被封装在一个新的 IP 包里，只不过这次源 IP 和目的 IP 地址反了过来，然后被发回你的电脑。

#### **第6步：你的电脑接收应答并计算结果**

1.  你的电脑收到这个返回的 IP 数据包。
2.  `ping` 程序解析出里面的 ICMP Echo Reply 报文。
3.  它检查**标识符和序列号**，找到与之匹配的那个发出去的请求。
4.  它用当前时间减去当时发送请求时记录的时间，计算出 **RTT**。
5.  它还会记录下返回的 IP 包头里**剩余的 TTL 值**。
6.  最后，`ping` 程序将这些信息格式化后，打印在你的屏幕上，就是你看到的那一行：
    `Reply from 172.217.160.78: bytes=32 time=15ms TTL=116`

这个过程会默认重复数次（在 Windows 上是 4 次，在 Linux/macOS 上会一直持续直到你按 `Ctrl+C`），最后给出一个汇总统计。

---

### **为什么 `ping` 不使用 TCP 或 UDP？**

*   **目的不同**: TCP/UDP 用于传输应用程序数据，需要端口号来区分不同的应用。而 `ping` 的目的是**测试网络连通性**，这是一个非常底层的网络诊断功能，与任何具体应用无关。
*   **足够简单**: ICMP 非常轻量级，它不需要像 TCP 那样建立复杂的连接（三次握手），也不需要端口号。它直接利用 IP 协议进行传输，完美地满足了“发一个包，收一个回应”的简单需求。

所以，`ping` 通过使用 ICMP 的 Echo Request/Reply 机制，实现了一个简单、高效、可靠的网络连通性和延迟探测工具。

## 哪些应用是udp，哪些用是tcp

* **TCP:**HTTP / HTTPS,FTP（文件传输协议）,SMTP（简单邮件传输协议）
* **UDP**：DNS（域名系统），实时视频和语音传输，在线游戏

## ？进程通信的方式,线程之间的通信方式

### 1. Mach端口 (Mach Ports)

*   **核心思想**：这是iOS/macOS操作系统内核（XNU内核）最基础、最核心的IPC机制。它就像是系统内核管理的**一个高效的、单向的“信箱系统”**。
*   **工作原理**：
    1.  一个进程（比如渲染服务）可以创建一个“端口”（信箱），并在内核中注册它。
    2.  另一个进程（比如你的App）可以获取到这个端口的“发送权”（信箱地址）。
    3.  你的App进程可以将一封“信”（Mach消息，包含数据）通过内核发送到这个端口。
    4.  内核负责将这封信安全地从你的App进程内存，拷贝到渲染服务进程的内存，并通知它有新信件。
*   **优点**：极其快速、高效，是系统服务间通信的基石，由操作系统内核直接支持。
*   **缺点**：API非常底层和复杂，直接使用容易出错。
*   **使用场景**：
    *   **`UIView`渲染过程**：`Core Animation`框架与`Render Server(backboardd)`之间的通信，就是基于Mach端口的。你的App将图层树的改动打包成Mach消息，发送给Render Server。
    *   几乎所有的系统高级框架，底层都可能用到了Mach端口。

---

### 2. XPC (Cross-Process Communication)

*   **核心思想**：这是苹果官方**现代且推荐**的高级IPC框架，它构建在Mach端口之上，提供了更安全、更易用的接口。
*   **工作原理**：XPC隐藏了所有Mach端口的复杂细节。你只需要定义一个通信协议（protocol），然后创建一个连接，就可以像调用本地对象的方法一样，异步地调用另一个进程的方法。XPC会自动处理消息的序列化、发送、接收和反序列化。
*   **优点**：
    *   **安全**：连接是私有的，权限控制严格。
    *   **稳定**：如果通信的另一端服务崩溃了，你的App不会崩溃，只会在回调中收到一个错误。
    *   **异步**：不会阻塞主线程。
*   **缺点**：相比直接使用底层API，会有一点点性能开销（但在绝大多数情况下可以忽略不计）。
*   **使用场景**：
    *   **App和App扩展（App Extension）之间**的通信。例如，你的主App需要和你的“今日小组件(Today Widget)”或“自定义键盘”通信。
    *   在macOS上，用于将App拆分成多个小的、单一职责的服务进程，以提高稳定性和安全性。

---

### 3. Socket (套接字)

*   **核心思想**：这是一种非常传统的IPC方式，源于Unix世界。它将进程间通信抽象成了网络通信的模型，即使两个进程在同一台机器上。
*   **工作原理**：一个进程创建一个“服务端Socket”并监听一个本地端口号（或一个本地文件路径）。另一个进程创建一个“客户端Socket”去连接这个服务端。一旦连接建立，双方就可以像读写文件一样互相发送数据流。
*   **优点**：跨平台，通用性强，不仅能用于本地IPC，也能直接用于网络通信。
*   **缺点**：相比Mach端口，性能稍差，API相对原始。
*   **使用场景**：一些需要跨平台的库或者有历史原因的项目可能会使用。在纯iOS开发中不常用作App内的IPC。

---

### 4. 共享内存 (Shared Memory)

*   **核心思想**：这是**速度最快**的IPC方式。操作系统会开辟一块特殊的内存区域，然后将这块内存同时映射到多个进程的地址空间里。
*   **工作原理**：进程A向这块共享内存写入数据，进程B能立刻看到这些数据的变化，因为它操作的其实是同一块物理内存。数据完全不需要在进程间进行拷贝。
*   **优点**：极快，尤其适合传输大量数据（如视频帧、音频数据）。
*   **缺点**：**极其危险**。因为共享的是同一块内存，必须由开发者自己处理复杂的**同步问题**（比如使用锁、信号量），否则极易导致数据错乱和崩溃。
*   **使用场景**：对性能有极致要求的场景，如图形处理、实时音视频流。`Core Animation`在某些情况下也会利用类似的机制（`IOSurface`）在GPU和不同进程间高效共享纹理数据。

---

### 5. 其他方式

*   **URL Scheme**：一个App可以通过一个特殊的URL（如 `weixin://`）来启动另一个App并传递少量参数。这是一种高层级的、用于**应用间跳转和调用**的IPC。
*   **App Groups & Shared Containers**：可以让同一个开发者的多个App（或App与其扩展）共享同一个文件目录。通过读写这个目录下的文件，可以实现数据交换。这是一种**基于文件系统**的IPC。
*   **剪贴板 (Pasteboard)**：用户通过“拷贝”和“粘贴”操作，实际上也是在进行一种最简单的、由用户驱动的进程间通信。

这是一个非常好的问题，因为它触及了面试中一个很重要的能力——**理解面试官的真实意图，并在合适的深度和广度上作答。**

对于“进程间通信方式”这个问题，面试官的期望通常是**一个分层的、从iOS具体应用到操作系统底层原理的“T”型回答**。

*   **“T”的横向部分**：代表广度。你需要展示你知道**iOS生态中存在的多种IPC方式**，并能说出它们各自的应用场景。
*   **“T”的纵向部分**：代表深度。你需要选择**一到两种**与iOS开发最相关、最重要的IPC方式，进行深入的原理剖析。

面试官**不希望**你只回答其中一个方面：
*   **只回答纯操作系统理论**（比如只讲信号量、管道、消息队列等教科书概念）：会让面试官觉得你脱离实际，可能只是在背书，不了解这些理论如何在iOS这个具体平台上落地。
*   **只回答纯iOS应用层**（比如只讲URL Scheme和App Groups）：会让面试官觉得你的知识深度不够，不了解这些高层封装背后的底层原理，对系统缺乏深入的理解。

---

### 一个理想的回答结构应该是这样的：

**第一步：总起句，直接点明核心，展示你的高度。**

> “关于iOS中的进程间通信，我认为可以从两个层面来看：一个是苹果为我们开发者封装好的上层API，另一个是支撑这些API的操作系统底层核心机制。在实际开发中，我们更多地是和上层API打交道。”

*(这个开场白，直接告诉面试官你知道这里面有层次，并且你的回答会覆盖这两个层次。)*

**第二步：展开“T”的横向部分，展示广度。**

> “在上层，我们常用的有几种方式：”
>
> *   “**URL Scheme**：主要用于App之间的跳转和简单的命令传递。”
> *   “**App Groups**：当同一个开发者的主App和App Extension（比如Widget）需要共享文件时，我们会用它来创建一个共享的沙盒目录，通过读写文件实现通信。”
> *   “**XPC (Cross-Process Communication)**：这是苹果官方最推荐的、现代化的IPC框架。它非常安全和稳定，是构建App Extension（如自定义键盘）和主App之间通信的首选。它的API设计得很好，可以让我们像调用本地方法一样进行异步通信。”
> *   “当然，还有一些比如**剪贴板**、**KeyChain共享**等，也可以看作是广义上的IPC。”

*(这部分展示了你对iOS开发生态的熟悉程度，知道在什么场景下该用什么工具。)*

**第三步：展开“T”的纵向部分，展示深度，并与iOS核心关联。**

> “而在这些上层API的背后，iOS/macOS的内核提供了更底层的通信机制。其中最核心、最重要的就是**Mach端口（Mach Ports）**。”
>
> *   “**Mach端口**是操作系统内核级的IPC，可以理解为非常高效的信箱系统，性能极高。它正是iOS整个UI渲染流程的基石。比如，我们的App在主线程对`CALayer`进行修改后，`Core Animation`框架就会在RunLoop的末尾，把所有图层树的改动打包成一个Mach消息，通过Mach端口发送给一个独立的**渲染服务进程（Render Server）**。”
> *   “这种设计使得渲染操作与App进程解耦，即使App主线程有短暂卡顿，也不会完全冻结整个系统的UI。XPC实际上也是构建在Mach端口之上的一个更安全的封装。”
> *   “除了Mach端口，还有像**Socket**和**共享内存**这样的传统机制。特别是**共享内存**，因为不需要数据拷贝，速度最快，所以在对性能要求极致的场景，比如GPU和多进程之间共享图形数据（`IOSurface`就是一种实现），就会用到它。”

*(这部分展示了你不仅知其然，还知其所以然。你能把IPC这个通用概念，和你刚才聊到的`UIView`渲染流程关联起来，形成知识体系的闭环，这是非常大的加分项。)*

**第四步：总结，回到实际。**

> “总的来说，虽然我们iOS开发者不常直接操作Mach端口，但理解它的存在和作用，对于我们理解系统性能、App生命周期以及像`Core Animation`这样的核心框架的工作原理，都有非常大的帮助。”



## ？HTTP 状态码有哪些？

## quic协议

好的，我们来深入探讨一下 **QUIC (Quick UDP Internet Connections)** 协议。它是互联网通信领域的一项重大革新，也是理解 HTTP/3 的关键。

简单来说，QUIC 是一个全新的、构建在 **UDP** 之上的**加密传输协议**。它被设计用来取代 TCP，旨在解决现代网络应用中的许多性能瓶颈。

可以把 QUIC 想象成一个“集大成者”，它将 **TCP 的可靠性**、**TLS 1.3 的加密安全性** 以及 **HTTP/2 的多路复用** 等特性融合并优化，全部打包在一个协议里。

---

### QUIC 的诞生动机：为什么我们需要它？

---

### QUIC 的核心特性与工作原理

#### 1. 彻底解决队头阻塞 (内置的多路复用)

这是 QUIC 最核心的优势。

*   **TCP 的问题**：所有 HTTP/2 的流（Streams）都混合在一个 TCP 连接中。一个 TCP 数据包的丢失，会导致整个 TCP 连接停顿，所有流都被阻塞。
*   **QUIC 的方案**：QUIC 将“流”作为一等公民。每个流在 QUIC 内部都是独立管理的。**一个流的数据包丢失，只会阻塞那一个流**，其他流的数据仍然可以被正常地处理和交付给上层应用。

**比喻**：如果说 HTTP/2 是把所有货物打散成小包裹放到一个传送带上（一个包裹丢失，整个传送带暂停），那么 QUIC 就像是为不同类型的货物开设了多个并行的传送带。一个传送带坏了，完全不影响其他的传送带继续运行。

#### 2. 大幅减少连接建立的延迟 (0-RTT & 1-RTT)

建立一个安全的 HTTPS 连接在 TCP 上通常很慢：
*   **TCP 握手**：需要1个RTT（往返时间）。
*   **TLS 握手**：需要1-2个RTT。
*   **总共**：需要2-3个RTT才能开始发送应用数据。

QUIC 将传输层握手和加密握手合并了：
*   **首次连接**：客户端和服务器通过 **1-RTT** 就可以完成握手并开始传输数据。
*   **后续连接**：如果客户端之前连接过该服务器，它可以利用缓存的密钥信息，实现 **0-RTT** 连接。这意味着客户端可以直接向服务器发送加密的应用数据，无需任何握手等待。这对移动网络等高延迟环境的体验提升是巨大的。

#### 3. 连接迁移 (Connection Migration)

这是一个对移动设备用户极其友好的特性。

*   **TCP 的问题**：一个 TCP 连接由一个四元组唯一标识：`(源IP, 源端口, 目的IP, 目的端口)`。当你手机的網絡从 Wi-Fi 切换到 4G 时，你的IP地址变了，这个四元组也就失效了，TCP 连接必须断开重连。视频通话会卡顿，文件下载会中断。
*   **QUIC 的方案**：QUIC 不使用IP地址来标识连接。它在连接建立之初，会生成一个64位的**连接ID (Connection ID)**。只要这个ID不变，无论你的IP地址和端口如何变化（从Wi-Fi切到4G，再切回来），底层的连接都**不会中断**。QUIC 会无缝地将数据包发送到新的IP地址，实现平滑的连接迁移。

#### 4. 内置的强制加密

*   **TCP 的世界**：加密是可选的（通过上层的TLS）。
*   **QUIC 的世界**：**加密是协议的内置部分，而且是强制的**。除了少数几个用于握手的包文，QUIC 的所有数据包都经过了认证加密（AEAD）。这大大提升了网络的安全性，并能有效防止中间人对协议进行篡改或干扰。

#### 5. 更灵活的拥塞控制

QUIC 的拥塞控制算法不再由操作系统内核写死。它在应用层实现，这意味着可以快速地部署和迭代新的、更先进的拥塞控制算法（如 Google 的 BBR），以适应不断变化的网络状况。

---

### QUIC 和 HTTP/3 的关系

这是一个非常重要的概念：

*   **QUIC** 是一个**传输层协议**，是 TCP 的替代品。
*   **HTTP/3** 是一个**应用层协议**，是 HTTP/2 的升级版。

它们的关系是：**HTTP/3 不再使用 TCP 作为其底层的传输协议，而是专门设计为在 QUIC 之上运行。**

所以，当你谈论 QUIC 时，你是在谈论一个新的“公路系统”（传输层）；当你谈论 HTTP/3 时，你是在谈论在这个新公路上跑的“汽车”（应用层）。

### 总结

| 特性         | TCP + TLS (用于 HTTP/2)         | QUIC (用于 HTTP/3)           | 优势                          |
| :----------- | :------------------------------ | :--------------------------- | :---------------------------- |
| **底层协议** | TCP                             | UDP                          | 绕过TCP的限制，自由实现新特性 |
| **连接建立** | 2-3 RTT                         | **0-1 RTT**                  | 连接延迟极低                  |
| **队头阻塞** | TCP层存在，一个包丢失阻塞所有流 | **已解决**，流之间完全独立   | 提升了多路复用下的传输效率    |
| **连接迁移** | 不支持，IP改变需重连            | **支持** (通过Connection ID) | 移动网络体验无缝切换          |
| **加密**     | 由上层TLS实现，可选             | **内置且强制** (基于TLS 1.3) | 安全性更高，防篡改            |
| **拥塞控制** | 在内核中实现，更新慢            | 在应用层实现，可快速迭代     | 更灵活、更先进                |

## HTTP1.1 -> HTTP2.0 -> HTTP3.0

HTTP/1.1 相比 HTTP/1.0 性能上的改进：

- **长连接**：使用长连接的方式改善了 **HTTP/1.0 短连接**造成的性能开销，不用每次请求都tcp**握手挥手**。
- **管道网络传输**：（HTTP1.0 :完全串行执行，后一个请求必须在前一个响应之后发送。）请求可以并行发出，但是响应必须串行返回。
- 但1.1仍然有其缺点**-队头阻塞**，虽然一次可以发送多个请求，但是服务端响应的顺序只能根据请求顺序响应，如果第一个请求的处理非常耗时，后续的响应即使已经准备好也无法发送。
- 假设客户端通过一个 HTTP/1.1 连接连续发送了请求 A、请求 B 和请求 C。

  1. 服务器接收到请求 A，开始处理。
  2. 服务器接收到请求 B，将其放入队列等待。
  3. 服务器接收到请求 C，将其放入队列等待。

  如果请求 A 的处理时间很长，或者服务器在处理请求 A 时遇到网络问题（比如服务器需要等待某个后端服务），那么**即使请求 B 和请求 C 已经准备好可以被处理和响应，它们也必须等待请求 A 完成并发送响应之后才能被处理**。这就好比一个队伍，第一个人堵住了，后面的人即使没事也无法前进。

HTTP/2 协议

* **头部压缩** ：如果你同时发出多个请求，他们的头是一样的或是相似的，协议会帮你**消除重复的部分**。
* HTTP/2 不再像 HTTP/1.1 里的纯文本形式的报文，而是全面采用了**二进制格式**.
* **多路复用**:HTTP/2 允许多个请求和响应在同一个 TCP 连接上**并发**地进行, 客户端可以同时发送多个请求的帧，服务器也可以同时发送多个响应的帧，而**不必按照先进先出的顺序**。接收端会根据帧中的流 ID 将属于同一个请求或响应的帧重新组装成完整的 HTTP 报文。

假设一个网页需要加载三个资源：HTML 文件 (Stream 1)、CSS 文件 (Stream 2) 和一张图片 (Stream 3)。

- **HTTP/1.1:** 浏览器通常会发起多个串行请求（受浏览器对同一域名并发连接数的限制）。如果 HTML 文件的响应很大或者服务器处理较慢，CSS 和图片即使服务器已经准备好也必须等待 HTML 文件完全下载完毕后才能开始传输（在同一个连接内）。
- **HTTP/2:** 浏览器会为这三个资源创建三个独立的流 (Stream 1, 2, 3)，并通过同一个 TCP 连接并发地发送请求帧。服务器可以并行地处理这三个请求，并将它们的响应分解成帧，并带有相应的流 ID，然后交错地在同一个 TCP 连接上发送这些帧。浏览器接收到这些帧后，会根据流 ID 将它们重新组装成完整的 HTML、CSS 和图片，而不会因为其中一个资源的传输缓慢而阻塞其他资源的传输。

* **服务器主动推送资源**：客户端在访问 HTML 时，服务器可以直接主动推送客户端可能需要的资源。

## HTTP和HTTPS的区别

## HTTPS加密,对称加密和非对称,为什么不只用其中一个呢?

https如何实现可靠传输

* 加密 
* **CA 签名验证:** 客户端会使用预装的 CA 的公钥来验证服务器证书上的签名，以确保证书是由可信的 CA 颁发的，没有被篡改。

## 什么是 TCP ？

## ？TCP的数据校验方式

## ？有哪些非对称加密算法， 有哪些对称加密算法

## ***为什么是三次握手

- 假设只有两次握手。客户端发送一个 SYN 包，服务器收到后回复 SYN+ACK，连接就建立了。如果一个延迟的、失效的 SYN 包（由于网络拥塞等原因滞留）到达服务器，服务器会认为这是一个新的连接请求，并发送 SYN+ACK，然后等待客户端发送数据。这就浪费了客户端的资源。**三次握手中的 ACK 就是为了确认服务器收到了客户端的 SYN，从而避免服务器为过时的 SYN 包建立连接。**
- 如果服务器发送的 SYN-ACK 包在网络中丢失，客户端并不知道连接已经建立成功。客户端可能会一直等待服务器的确认，而服务器却在等待客户端发送数据。这会导致客户端认为连接失败，而服务器却处于等待数据的状态，造成**连接状态的不一致**。

## 三次握手主要交换什么信息

这个问题问得非常好，直击了三次握手最核心的本质！很多人知道三次握手的过程，但对其“交换了什么信息”的理解不够深刻。

简单来说，三次握手主要交换了**两类核心信息**：

1.  **能力与状态的同步 (Synchronization of State)**: 双方确认彼此都“在线”且“有能力”进行通信。
2.  **初始序列号的交换 (Initial Sequence Number Exchange)**: 这是保证后续数据传输可靠性的基石。

我们来详细分解这“一手、二手、三手”分别交换了什么，以及这些信息为什么至关重要。

---

### **第一次握手 (客户端 -> 服务器)**

*   **发送方**: 客户端
*   **TCP 报文**: `SYN=1, seq=x`
*   **交换的信息**:
    1.  **连接意图**: 通过将 `SYN` 标志位置为1，客户端明确地告诉服务器：“**你好，我想和你建立一个连接。**” 这是最基本的意图表达。
    2.  **客户端的初始序列号 (ISN - Initial Sequence Number)**: 客户端随机选择一个初始序列号 `x`，并放在 `seq` 字段中。它告诉服务器：“**我这边的数据流，将从编号 `x` 开始计算。**”

*   **为什么需要交换这个信息？**
    *   `SYN` 是建立连接的“敲门砖”，没有它一切无从谈起。
    *   初始序列号 `x` 是后续所有数据可靠传输的起点。服务器需要知道这个起点，才能正确地确认收到了客户端的数据。

---

### **第二次握手 (服务器 -> 客户端)**

*   **发送方**: 服务器
*   **TCP 报文**: `SYN=1, ACK=1, seq=y, ack=x+1`
*   **交换的信息 (这是信息量最大的一次握手)**:
    1.  **同意连接的确认**: 通过将 `ACK` 标志位置为1，并设置 `ack=x+1`，服务器告诉客户端：“**我收到了你的连接请求（`SYN`），也收到了你的初始序列号 `x`。我期望你下一个发给我的数据，应该是从编号 `x+1` 开始。**” 这是对第一次握手的**确认**。
    2.  **服务器的连接意图**: 服务器也将 `SYN` 标志位置为1，这表示：“**我也同意建立连接，并且我也有话要说。**”
    3.  **服务器的初始序列号**: 服务器同样随机选择一个自己的初始序列号 `y`，放在 `seq` 字段中。它告诉客户端：“**我这边的数据流，将从编号 `y` 开始计算。**”

*   **为什么需要交换这个信息？**
    *   **确认客户端的发送能力**: `ack=x+1` 的发送，证明了服务器成功收到了客户端的 `SYN` 包。客户端收到这个 `ack` 后，就知道自己的发送通道是通畅的。
    *   **同步服务器的初始序列号**: 客户端必须知道服务器的初始序列号 `y`，这样它才能在后续通信中，正确地确认收到了来自服务器的数据。
    *   **确认服务器的接收能力**: `SYN=1` 的发送，本身也向客户端表明，服务器是“活着的”，并且有能力接收和处理请求。

---

### **第三次握手 (客户端 -> 服务器)**

*   **发送方**: 客户端
*   **TCP 报文**: `ACK=1, ack=y+1`
*   **交换的信息**:
    1.  **对服务器同意连接的最终确认**: 客户端将 `ACK` 标志位置为1，并设置 `ack=y+1`。它告诉服务器：“**我收到了你的同意连接信号（`SYN`），也收到了你的初始序列号 `y`。我确认收到了，我们现在可以正式开始通信了。**” 这是对第二次握手的**确认**。

*   **为什么需要交换这个信息？**
    *   **确认客户端的接收能力**: 客户端能发出这个包，说明它成功收到了服务器的 `SYN-ACK` 包。服务器收到这个最终的 `ACK` 后，就知道客户端的接收通道也是通畅的。
    *   **防止失效的连接请求 (核心)**: 这是三次握手最关键的功能。它向服务器证明，客户端**当前确实是处于想要建立连接的状态**。如果这是一个早已失效的、旧的连接请求导致的服务器响应，客户端会因为状态不匹配而不会发送这个最终的 `ACK`，从而避免了服务器空耗资源。



## 为什么是四次挥手，不是三次挥手

### 二、为什么挥手不能是三次？

现在我们能清晰地回答这个问题了。TCP挥手不能像握手那样将第二和第三步合并的原因是：

- **在握手时**，服务器收到客户端的 SYN 请求后，它自己也**没有数据要传输**，它的唯一任务就是同意连接。所以，它可以非常快地把表示“同意”的 ACK 和表示“我也要同步”的 SYN 捆绑在一个包里发出去。
- **在挥手时**，当服务器收到客户端的 FIN 请求时，它仅仅意味着客户端**不再发送数据了**。但服务器自己可能还有一大堆数据正在发送队列里，需要发给客户端。
  - TCP 协议规定，服务器必须**立刻**用一个 ACK 来确认收到了客户端的 FIN。
  - 但它**不能立刻**发送自己的 FIN，必须等到自己所有的数据都发送完毕后，才能发送 FIN。
  - 从发送 ACK 到发送 FIN 之间，可能有一段时间差。正是这个**不确定的时间差**，导致了 ACK 和 FIN 必须作为两个独立的步骤，从而构成了四次挥手。

##  ？HTTP协议流程

## TCP 四次挥手过程是怎样的？

## 为什么要等待2MSL？

如果服务端没有接收到客户端最后的ack，就会重发fin报文，这个时候为了正确关闭服务端的连接，所以客户端要等一下可能还存在的fin报文，2MSL是允许报文丢一次的时间。

## 为什么挥手要四次

你这个问题问得非常好！理解了三次握手，就必须理解为什么挥手需要四次，这能让你对 TCP 的**全双工（full-duplex）**特性有更深刻的认识。

简单来说，四次挥手的核心原因是：**TCP 连接是全双工的，一方关闭了发送通道，但可能仍然需要接收对方发来的数据。因此，服务器的“确认”（ACK）和“关闭”（FIN）是分开的两个步骤。**

---

### 一、一个生动的比喻：结束一场电话会议

想象你（客户端）和你的老板（服务器）正在进行一场重要的电话会议。

1.  **第一次挥手 (你 -> 老板):** 你汇报完了所有工作，于是你说：“老板，我这边的事情说完了。”
    *   **技术上**: 客户端发送一个 `FIN` 包，表示“我的数据已经发完了，我请求关闭我这一方的发送通道。” 此时，客户端进入 `FIN_WAIT_1` 状态。

2.  **第二次挥手 (老板 -> 你):** 老板听到了，但他可能还有话要对你说。他会先回应你一下：“好的，收到了，我知道你说完了。”
    *   **技术上**: 服务器收到 `FIN` 后，立刻发送一个 `ACK` 包作为回应，表示“我已收到你的关闭请求。” 此时，服务器进入 `CLOSE_WAIT` 状态，而客户端收到这个 `ACK` 后，进入 `FIN_WAIT_2` 状态。

**关键点来了：为什么不能在这里直接结束？**

> 因为虽然你已经说完了，但你的**老板可能还有最后的几点指示要交代**。你虽然不再说话（关闭了发送），但你的耳朵还得听着（接收通道仍然打开）。

3.  **第三次挥手 (老板 -> 你):** 老板交代完了所有事情，现在他也准备挂电话了。于是他说：“好了，我这边也说完了。”
    *   **技术上**: 服务器在处理完所有需要发送的数据后，也发送一个 `FIN` 包，表示“我这边的数据也发完了，我请求关闭我这一方的发送通道。” 此时，服务器进入 `LAST_ACK` 状态。

4.  **第四次挥手 (你 -> 老板):** 你听到老板也说完了，你最后回应一下：“好的，收到了，那我们挂电话吧。”
    *   **技术上**: 客户端收到服务器的 `FIN` 后，发送最后一个 `ACK` 包作为确认。发送完毕后，客户端进入 `TIME_WAIT` 状态（稍后解释），而服务器收到这个最终的 `ACK` 后，立即关闭连接。

---

### 二、为什么挥手不能是三次？

现在我们能清晰地回答这个问题了。TCP挥手不能像握手那样将第二和第三步合并的原因是：

*   **在握手时**，服务器收到客户端的 `SYN` 请求后，它自己也**没有数据要传输**，它的唯一任务就是同意连接。所以，它可以非常快地把表示“同意”的 `ACK` 和表示“我也要同步”的 `SYN` 捆绑在一个包里发出去。

*   **在挥手时**，当服务器收到客户端的 `FIN` 请求时，它仅仅意味着客户端**不再发送数据了**。但服务器自己可能还有一大堆数据正在发送队列里，需要发给客户端。
    *   TCP 协议规定，服务器必须**立刻**用一个 `ACK` 来确认收到了客户端的 `FIN`。
    *   但它**不能立刻**发送自己的 `FIN`，必须等到自己所有的数据都发送完毕后，才能发送 `FIN`。
    *   从发送 `ACK` 到发送 `FIN` 之间，可能有一段时间差。正是这个**不确定的时间差**，导致了 `ACK` 和 `FIN` 必须作为两个独立的步骤，从而构成了四次挥手。



## ？TCP关闭连接后操作系统需要释放哪些资源？

## ***滑动窗口、流量控制、拥塞控制

* **TCP 的流量控制** 是通过 **滑动窗口机制** 来实现的，接收方通过 TCP 头部的 **窗口大小字段**告知发送方自己剩余的缓冲空间，发送方根据这个信息调整发送速率，以避免接收方的缓冲区溢出。

* 发送方的滑动窗口大小等于：已发送未确认+未发送可发送的大小。

* 拥塞控制：拥塞窗口（cwnd）是在发送端定义的，一旦出现丢包，拥塞窗口就会减小，发送端的最大发送范围是拥塞窗口和滑动窗口中较小的一个
* **慢启动（Slow Start）**：
  - 初始 `cwnd` 小，每收到1个 ACK，**cwnd 的大小就会加 1，实现 指数增长**，直到达到 慢启动门限（ssthresh）
  - 拥塞避免算法：**每当收到一个 ACK 时，cwnd 增加 1/cwnd**，线性增加
  - 拥塞发生，启用快速重传，`cwnd = cwnd/2` ，也就是设置为原来的一半; `ssthresh = cwnd`;
  - 快速恢复

## 为什么tcp一开始要慢启动，不以最大能力传输

## ***TCP实现可靠传输的原理

* 三次握手
* 数据分块与序号标识
* 确认应答（ACK）与超时重传
* 流量控制
* 拥塞控制

## UDP如何实现可靠连接?

quic协议

## 粘包问题知道吗?TCP和UDP都会有粘包问题吗?

## OSI七层模型和tcp五层

* **应用层**：直接向用户应用程序提供网络服务。定义了用户程序发送请求的规则
* **表示层**：数据进行加密和解密的。
* **会话层**：会话层（第五层）主要负责管理**应用程序之间**的通信会话，或者说是建立、管理和终止**进程间的对话**。还要对话控制，全双工，半双工
* **传输层**：这一层主要是建立端到端的连接，要建立两个正确的端口号的连接，tcp
* **网络层**：主要是定义一个在整个互联网寻址的规则，ip寻址。
* **网络接口层**：主要解决在局域网怎么寻址路由封包的问题，mac地址来寻址
* **物理层**：主要解决0和1怎么转化为信号在物理链路上传输的问题

## Socket是什么

---

### 1. “Socket 就是一套 API”

您说的没错。Socket 的核心就是一套**操作系统提供给应用程序的编程接口（API）**。

它就像是应用程序和操作系统内核网络协议栈之间的一个“**约定**”或“**合同**”。这个合同规定了：

*   **应用程序需要做什么**：调用 `socket()`, `bind()`, `connect()`, `send()` 这些函数。
*   **操作系统承诺做什么**：当应用程序调用这些函数时，操作系统内核会负责完成所有底下肮脏、复杂的工作（比如打包、寻址、路由、保证可靠性等）。

这套API让程序员能够以一种标准化的方式来使用网络功能，而不用去关心底层用的是什么网卡、什么路由器，或者数据在光纤里到底是怎么传输的。

### 2. “`connect` 方法就是封装了三次握手”

这个比喻非常恰当。这完美地体现了“封装”的意义。

*   **从程序员的视角看**：
    我只需要调用一个简单的函数 `connect(socket_fd, &server_addr, sizeof(server_addr))`。这是一个**同步的、单一的动作**。在程序员看来，这个函数要么成功返回（连接建立），要么失败返回（连接失败）。

*   **在操作系统（TCP协议栈）的视角看**：
    这是一个**异步的、复杂的过程**：
    1.  **客户端**：创建一个TCP报文，将`SYN`标志位置1，并附上一个初始序列号（ISN），然后发送给服务器。进入`SYN_SENT`状态。
    2.  **服务器**：接收到SYN报文后，回复一个TCP报文，将`SYN`和`ACK`标志位都置1，并附上自己的ISN和对客户端ISN的确认号。进入`SYN_RCVD`状态。
    3.  **客户端**：接收到服务器的SYN-ACK报文后，再发送一个`ACK`报文给服务器进行最终确认。进入`ESTABLISHED`状态。
    4.  **服务器**：接收到最终的ACK后，也进入`ESTABLISHED`状态。

程序员调用`connect()`的那一刻，操作系统就在底层默默地完成了以上所有步骤，包括可能发生的**超时重传**。程序员完全不需要关心这些状态的转换和细节。

### 3. “`send` 方法只需要传数据，不关心报文格式的封装”

是的，这正是封装的魅力所在。

*   **从程序员的视角看**：
    我有一个字符串 "Hello, World!"。我把它看作一个**连续的字节流**，然后调用 `send(socket_fd, "Hello, World!", 13, 0)`。我只关心把这串字节“扔”进Socket里。

*   **在操作系统（TCP/IP协议栈）的视角看**：
    它接到这13个字节的数据后，会进行一系列复杂的“**打包**”工作：
    1.  **TCP层**：首先，TCP会根据MSS（最大报文段长度）把数据切割成合适的块。然后给每一块数据加上一个**TCP头部**，里面包含了源/目的端口号、序列号、确认号、校验和等信息。这个“TCP头 + 数据”被称为一个**TCP报文段（Segment）**。
    2.  **IP层**：TCP层把报文段交给IP层。IP层再给它加上一个**IP头部**，里面包含了源/目的IP地址等信息。这个“IP头 + TCP报文段”被称为一个**IP数据报（Datagram）**。
    3.  **数据链路层**：IP层再把数据报交给下一层，可能会被加上MAC地址等头部信息，成为一个**帧（Frame）**，最终在物理媒介上传输。

程序员通过一个简单的`send`调用，触发了底层协议栈精密的、层层的封装流程。我们完全不需要手动去拼接这些报文头部。

---

### 第一部分：对称加密 (Symmetric Encryption)

*   **是什么：** 加密和解密使用**同一个密钥**的加密方式。
*   **优点：** 速度极快，适合对大量数据进行加密。
*   **缺点：** 如何安全地把这个唯一的密钥告诉对方？如果密钥在传输过程中被窃听，那么所有加密内容都会被破解。

**HTTPS中的应用：** 在握手成功后，客户端和服务器之间传输的所有应用数据（如网页HTML、图片、API请求等）都是使用**对称加密**。常见的对称加密算法有AES、ChaCha20等。

---

### 第二部分：非对称加密 (Asymmetric Encryption)

*   **是什么：** 加密和解密使用**一对**不同的密钥：一个**公钥 (Public Key)** 和一个**私钥 (Private Key)**。
    *   **公钥**是公开的，任何人都可以获取。
    *   **私钥**是保密的，只有所有者持有。
    *   **核心规则：** 用公钥加密的数据，**只有对应的私钥**才能解开。
*   **优点：** 可以安全地交换信息，解决了对称加密的密钥分发难题。
*   **缺点：** 速度非常慢，比对称加密慢几个数量级，不适合加密大量数据。
*   **好比：** 银行有一把“公开的锁”（公钥）和一个只有银行自己有的“钥匙”（私钥）。
    1.  银行把这把“公开的锁”分发给所有人。
    2.  你想告诉银行你们的暗号是“芝麻开门”。
    3.  你把“芝麻开门”写在纸条上，放进一个盒子里，然后用银行给你的那把“公开的锁”锁上。
    4.  你把这个锁上的盒子寄给银行。
    5.  **即使中途有人截获了这个盒子，他也打不开，因为他没有那把唯一的“钥匙”。**
    6.  银行收到盒子后，用自己保管的“钥匙”（私钥）打开盒子，就安全地知道了你们的暗号是“芝oma开门”。

**HTTPS中的应用：** 非对称加密**不用于加密通信内容**，它的唯一目的就是在**TLS握手阶段**，安全地协商出后续通信要使用的**对称加密密钥**。这个过程也叫**密钥交换 (Key Exchange)**。常见的非对称加密算法有RSA、ECDHE等。

---

### 第三部分：数字证书 (Digital Certificate)

我们刚才解决了密钥交换的问题，但还有一个致命漏洞：**你怎么确定你拿到的那把“公开的锁”真的是银行给你的？**

如果一个骗子（**中间人**）在半路截胡：

1.  他假装成银行，给了你一把他自己的“公开的锁”。
2.  你用这把锁加密了“芝麻开门”并发送。
3.  骗子用他自己的“钥匙”打开盒子，知道了你们的暗号。
4.  然后骗子再用真的银行的“公开的锁”加密“芝麻开门”，发给银行。
5.  这样一来，骗子就处在你们中间，可以窃听和篡改所有信息，而你和银行却毫无察觉。这就是**中间人攻击 (Man-in-the-Middle Attack)**。

**数字证书就是为了解决这个问题而生的。**

*   **是什么：** 一个由权威的、受信任的第三方机构（**CA - Certificate Authority**，证书颁发机构，如Let's Encrypt, DigiCert）颁发的“数字身份证”。
*   **包含内容：**
    *   **服务器的公钥**（那把“公开的锁”）。
    *   **证书所有者的信息**（如网站域名）。
    *   **CA机构的数字签名**。
*   **工作原理：**
    1.  银行（服务器）向权威的CA机构申请证书。
    2.  CA机构经过严格验证，确认这个网站确实是该银行所有。
    3.  CA机构用它自己的**私钥**，对银行的公钥和信息进行“盖章”（数字签名）。
    4.  你的浏览器和操作系统里，**预装了所有主流CA机构的公钥**（它们的“根证书”）。
*   **验证过程：**
    1.  当你访问银行网站时，银行不再是直接给你一把“公开的锁”，而是给你这份盖了章的**数字证书**。
    2.  你的浏览器收到证书后，会用预装好的CA机构的公钥，去验证证书上的“数字签名”是否有效。
    3.  如果验证通过，就说明：
        *   这个证书确实是这个权威CA机构颁发的。
        *   证书里的公钥确实是属于这个网站的，没有被篡改。
    4.  浏览器这时才会放心地从证书里取出服务器的公钥，进行后续的密钥交换。如果验证失败，浏览器会弹出严重的安全警告。

---

### HTTPS完整加密流程（串联起来）

1.  **TCP三次握手**：客户端和服务器建立基本的TCP连接。

2.  **TLS握手开始**：
    a. **客户端发送 `Client Hello`**：告诉服务器我支持哪些加密算法。
    b. **服务器发送 `Server Hello` 和 `Certificate`**：服务器选定一套加密算法，并把自己的**数字证书**发给客户端。
    c. **客户端验证证书**：浏览器使用内置的CA根证书，验证服务器证书的真伪。**确认了对方的身份**。
    d. **密钥交换**：
        *   客户端生成一个随机的“会话密钥”的“种子”（预主密钥）。
            *   用从证书中取出的**服务器公钥（非对称加密）**，将这个“种子”加密后发送给服务器。
            e. **生成会话密钥**：客户端和服务器现在都拥有了相同的“种子”和之前交换的随机数，它们各自独立地计算出完全相同的**对称加密密钥（会话密钥）**。
            f. **握手结束**：双方互相发送一个用刚生成的会话密钥加密的`Finished`消息，验证对方是否计算出了正确的密钥。

3.  **加密通信**：

好的，这个问题问到了计算机网络的基石！TCP（Transmission Control Protocol，传输控制协议）之所以成为互联网上应用最广泛的协议（例如，你现在浏览网页用的HTTP就是基于TCP的），就是因为它提供了一种**可靠的、面向连接的**数据传输服务。

“可靠”这个词意味着，对于应用层来说，TCP创造了一个美好的“幻觉”：**数据就像在一个无差错、无丢失、无乱序的完美管道中流动。**

为了创造这个“幻觉”，TCP在底层实现了一套非常精巧和复杂的机制。下面我们来详细分解这些机制。

---

### TCP如何保证可靠传输？

我们可以把TCP传输比作一个**极其负责任的快递服务**。它要解决三个核心问题：
1.  **包裹（数据包）丢失了怎么办？**
2.  **包裹损坏了怎么办？**
3.  **包裹送达的顺序乱了怎么办？**

TCP通过以下几个核心机制协同工作来解决这些问题：

#### 1. 序列号 (Sequence Numbers)

*   **机制简介**: TCP将发送的数据看作是一个连续的字节流。在建立连接时，双方会各自确定一个初始序列号。之后，发送的**每一个字节**都会被编上一个唯一的序列号。
*   **解决什么问题**: **数据包乱序**。
*   **如何工作**:
    *   假设一个TCP报文段（数据包）包含了从序列号1001到2000的字节。那么这个报文段的序列号字段就是1001。
    *   接收方收到了序列号为1001、3001、2001的三个包。它就知道这三个包的顺序是错的。
    *   接收方会根据序列号对数据包进行重新排序，然后将排好序的数据交给上层应用。这样，无论数据包在网络中如何颠簸，应用层收到的永远是正确的顺序。

#### 2. 确认应答 (Acknowledgements, 简称ACK)

*   **机制简介**: 这是保证可靠性的核心反馈机制。接收方在成功收到数据后，会向发送方发送一个**确认报文（ACK）**。
*   **解决什么问题**: **确认数据是否被对方收到**，是超时重传机制的基础。
*   **如何工作**:
    *   ACK报文中包含一个“确认号”字段。这个确认号的规则是：**我期望收到的下一个字节的序列号**。
    *   例如，发送方发送了序列号为1001，长度为1000字节的数据包（即字节1001到2000）。
    *   接收方成功收到后，会回复一个ACK报文，其“确认号”字段的值就是 `1001 + 1000 = 2001`。
    *   这个ACK的意思是：“序列号在2001之前的所有字节我都已经安全收到了，现在请你从序列号2001开始发送。”

#### 3. 超时重传 (Timeout & Retransmission)

* **机制简介**: 发送方在发送一个数据包后，会启动一个**计时器**。如果在计时器超时之前没有收到对方对这个数据包的确认应答（ACK），发送方就会认为这个数据包在传输过程中丢失了。

* **解决什么问题**: **数据包丢失**。

*   **如何工作**:
    *   发送方发送序列号1001的数据包，并启动计时器。
    *   在网络中，这个数据包或者它对应的ACK确认包可能丢失了。
    *   计时器到时，发送方仍然没有收到确认号为2001的ACK。
    *   发送方就**重新发送**序列号1001的这个数据包，并重置计时器。
    *   这个超时时间的设置是动态调整的，TCP会根据网络延迟（RTT，往返时间）来智能计算一个合理的超时时长。
    
    

#### 5. 流量控制 (Flow Control)

*   **机制简介**: TCP提供了一种机制，让接收方可以控制发送方的发送速率，防止发送方发送得太快，导致接收方的缓冲区溢出。
*   **解决什么问题**: **防止接收方被“淹没”**。
*   **如何工作**:
    *   TCP头部的“窗口大小（Window Size）”字段就是用来做流量控制的。
    *   接收方在发送ACK确认包时，会同时告诉发送方自己当前的接收缓冲区还剩多少空间（即窗口大小）。
    *   发送方看到这个窗口大小后，就知道自己最多还能发送多少数据。如果窗口大小为0，发送方就会停止发送数据，直到收到一个窗口不为0的ACK。

#### 6. 拥塞控制 (Congestion Control)

* **机制简介**: 这是TCP最高级、最复杂的机制。流量控制是点对点地考虑接收方的能力，而拥塞控制则是全局性地考虑**整个网络**的承载能力。

* **解决什么问题**: **防止过多的数据注入网络，导致整个网络瘫痪（网络拥塞）**。

*   **如何工作**:
    *   TCP通过“慢启动（Slow Start）”、“拥塞避免（Congestion Avoidance）”、“快速重传（Fast Retransmit）”等一系列复杂算法来工作。
    *   核心思想是：发送方通过观察网络状况（比如是否发生丢包/超时）来判断网络是否拥堵。如果判断网络拥堵，就主动降低自己的发送速率；如果网络状况良好，就慢慢提高发送速率。
    

## CDN是什么

好的，这是一个在面试中出现频率极高的问题，因为它不仅考察你的基础知识，还能延伸到性能优化、网络架构等多个方面。

一个理想的回答应该像剥洋葱一样，从**核心定义**，到**工作原理**，再到**带来的好处**，最后能结合**实际场景**。

---

### 一个理想的面试回答结构

#### **第一部分：一句话定义 (告诉面试官你抓住了核心)**

> “CDN，全称是Content Delivery Network，内容分发网络。它本质上是一个**构建在现有互联网之上的、分布式的智能缓存系统**。它的核心目标是**解决用户访问延迟的问题**，让用户可以**就近获取**所需内容，从而提高访问速度和稳定性。”

*(这个定义包含了几个关键词：分布式、缓存系统、就近获取、提高速度和稳定性。)*

---

### **第二部分：工作原理 (用一个生动的比喻+技术解释)**

> “要理解CDN的工作原理，我们可以把它想象成一个**覆盖全国的连锁仓库系统**，比如京东物流。”
>
> **1. 没有CDN的情况：**
> “假设我们的网站服务器（源站）在北京，就像一个**总仓库**。那么一个深圳的用户下单买东西（发起HTTP请求），货物必须从北京的总仓库发出，经过漫长的公路运输（互联网链路），才能送到深圳用户手里。这个过程不仅慢，而且中间任何一个环节堵车（网络拥堵），都会导致更长的延迟。”
>
> **2. 有了CDN之后：**
> “有了CDN，就相当于京东在全国各地，包括深圳，都建立了很多**前置仓（这就是CDN节点）**。CDN会提前把北京总仓库里的热门商品（网站的热门静态资源，如图片、视频、JS文件等）**缓存**到这些前置仓里。”
>
> **3. 核心的“智能调度”——DNS：**
> “那么，当深圳用户再次下单（访问网站）时，神奇的事情发生了：”
>
> 1.  “用户的访问请求，第一步不是直接去北京，而是先去问一个叫**DNS（域名系统）**的‘智能客服’。这个DNS系统已经被CDN“改造”过了，它不仅仅是把域名翻译成IP地址。”
> 2.  “这个智能的DNS客服会分析这个请求来自哪里（比如通过用户的IP地址，知道他来自深圳），然后**不再返回北京总仓库的地址**。”
> 3.  “取而代之，它会返回离用户**最近、最快、最不拥挤**的那个**深圳前置仓（CDN节点）的IP地址**。”
> 4.  “于是，用户的浏览器就直接去访问这个深圳的CDN节点了。因为是同城访问，速度极快，用户几乎立刻就拿到了他想要的图片或视频。”
>
> **总结一下技术流程就是：**
> **用户请求 -> 智能DNS解析 -> 定位到最优的CDN边缘节点 -> 如果节点有缓存，直接返回给用户；如果没缓存，节点会去源站请求数据，缓存下来再返回给用户。**

*(这部分用比喻把复杂的技术讲得通俗易懂，然后又回归到技术流程，展示了你的理解深度和表达能力。)*

---

### **第三部分：CDN带来的核心价值 (展示你的业务思考)**

> “使用CDN能带来几个非常关键的好处：”
>
> 1.  **用户体验的极大提升**：这是最直接的价值。通过就近访问，网站的加载速度会得到质的飞跃，尤其对于图片、视频等大文件，效果极其明显。这直接关系到用户留存率。
>
> 2.  **减轻源站服务器的压力**：大部分的用户请求（通常能达到80%-95%）都被分布在全国各地的CDN节点给“挡”住了，只有少量请求（比如缓存未命中或动态内容）会真正到达我们的源站服务器。这大大降低了源站的带宽成本和服务器负载，使其能更专注于处理核心的业务逻辑。
>
> 3.  **提升网站的可用性和稳定性**：CDN节点是分布式的，并且有冗余。如果某个节点出现故障，DNS系统会自动将用户的请求调度到其他健康的节点上。甚至在源站出现短暂故障时，只要CDN节点有缓存，用户依然可以正常访问网站的部分内容，起到了一个“缓冲层”的作用。
>
> 4.  **一定的安全防护**：很多CDN服务商都集成了基础的DDoS攻击防护、WAF（Web应用防火墙）等功能，可以帮助抵御一些常见的网络攻击。

---

### **面试时如何回答这个问题？**

你可以这样来组织你的回答，体现出你对直播技术的深度理解。

**第一步：先给出明确的肯定答案，并点出核心区别。**

> “是的，直播服务不仅可以用CDN，而且CDN是支撑大规模、低延迟、高并发直播业务的**绝对核心基础设施**。不过，它和我们通常说的用于网站静态资源（如图片、视频点播文件）的CDN工作方式不太一样。静态资源CDN的核心是**‘缓存’**，而直播CDN的核心是**‘分发’和‘转发’**。”

*(这个开场白直接展现了你的洞察力，你知道关键区别在于“缓存” vs “转发”。)*

---

### **第二步：解释为什么直播不能简单地“缓存”。**

> “您刚才说的非常对，‘直播画面不都是一个个流吗？’，这正是问题的关键。直播的特点是**实时性**。数据流是持续不断产生的，而且用户希望看到的是**几乎没有延迟**的画面。如果我们像缓存图片一样，等一个文件（比如一段10秒的视频切片）完全下载到CDN节点再提供给用户，那延迟就太大了，直播就变成了‘慢播’。”

---

### **第三步：详细讲解直播CDN的工作原理。**

> “所以，直播CDN的工作流程更像一个**‘高效的接力转发系统’**，而不是一个‘仓库缓存系统’。整个流程是这样的：”
>
> **1. 推流 (Push)**
> > “首先，主播通过推流软件（比如OBS）或者我们的App，将摄像头采集到的音视频数据，使用**RTMP**或**SRT**等推流协议，**推送到离主播最近的一个CDN“收流”节点**。这个节点也叫**‘边缘接入点’**或**‘上行节点’**。”
> > “选择最近的节点是为了保证**推流的稳定性和低延迟**，这是保证直播质量的第一步。”
>
> **2. 内部传输与处理**
> > “这个CDN收流节点收到数据后，并不会等数据存满再动。它会**立刻**将这些音视频流，通过CDN厂商内部的**高速专线网络**，传输到他们的**中心处理集群**。”
> > “在中心集群，可能会进行一些处理，比如**转码**（生成不同清晰度的流，如1080P、720P、540P）、**录制**（用于后续回放）、**截图**（用于生成封面）、**内容审核**等。”
>
> **3. 全球分发 (Distribute)**
> > “处理完成后的多路清晰度流，会从中心集群被**实时分发**到CDN覆盖全球的**所有边缘节点**上。这个分发过程同样是走CDN内部的高速网络，而不是普通的公网，所以速度非常快。”
>
> **4. 拉流 (Pull)**
> > “最后，当一个**深圳的用户**打开我们的App想看这个直播时：”
> > 1.  “他的请求同样会经过**智能DNS**的调度。”
> > 2.  “DNS会返回离他最近、网络最好的那个**CDN深圳边缘节点的IP地址**。”
> > 3.  “用户的播放器就会向这个深圳节点发起请求，**‘拉取’**直播流。因为这个节点上已经有了从中心实时同步过来的直播数据，所以用户能很快地看到画面。”
> > 4.  “这个拉流过程通常使用**HTTP-FLV、HLS或CMAF**等更适合大规模分发的播放协议。”

*   **示意图**
    ```
      主播 (北京)
         |
      (RTMP推流)
         |
      CDN北京节点 (收流)
         |
      (CDN内部高速专线)
         |
      CDN中心服务器 (转码、录制)
         |
      (CDN内部高速分发)
      /      |      \
     /       |       \
    CDN上海节点  CDN广州节点  CDN深圳节点 ...
      |         |          |
    (拉流)    (拉流)     (拉流)
      |         |          |
    观众A     观众B      观众C
    ```

## 为什么说http是无状态的？

在 Web 交互中，这意味着：

1. **每个请求都是独立的**：当你在浏览器上发起一个 HTTP 请求（比如点击一个链接），服务器处理完这个请求并返回响应后，它就**完全忘记**了这次交互。
2. **服务器不保留客户端信息**：服务器不会主动记录是谁在访问它，也不会保留该客户端之前的操作历史。
3. **后续请求必须提供所有信息**：如果你需要进行一系列相关的操作（比如登录、添加到购物车、结算），那么你的**每一次**请求都必须重新告诉服务器“我是谁”以及所有必要的信息。服务器不会因为你上一步刚刚登录过，就“记住”你已经登录了。

### 三、为什么 HTTP 被设计成无状态的？

将核心协议设计成无状态，是出于非常重要的工程考量，带来了巨大的好处：

1. **简化服务器设计**：
   - 服务器不需要花费额外的资源（内存、CPU）来存储和维护海量的客户端状态信息。想象一下，如果一个像 Google 或 Facebook 这样的服务器需要记住全球数十亿用户每个人的当前操作状态，那将是一个巨大的负担。
   - 无状态使得服务器可以更专注于处理当前请求的业务逻辑，职责单一。
2. **增强可伸缩性（Scalability）**：
   - 这是最重要的优点。因为服务器不保存状态，所以任何一个客户端的请求都可以被**任意一台服务器**处理。
   - 这使得构建大型服务器集群和进行**负载均衡（Load Balancing）**变得非常容易。一个请求来了，负载均衡器可以随便把它分发给集群中任何一台空闲的服务器，而不用担心这台服务器是否“认识”这个客户端。这极大地提高了网站的并发处理能力和可靠性。
3. **提高可靠性（Robustness）**：
   - 如果一台服务器宕机了，客户端的下一个请求可以无缝地被另一台服务器接管，因为新的服务器不需要从宕机的服务器那里同步任何状态信息，它只需要根据请求本身包含的信息来处理即可。

### 四、既然无状态，那如何实现有状态的应用（如购物车、登录）？

这是一个关键问题。HTTP 协议本身是无状态的，但现实中的 Web 应用几乎全都是有状态的。我们是通过在**无状态的协议之上构建有状态的会话机制**来解决这个问题的。

这就好像我们虽然知道金鱼记不住事，但我们发明了一种方法来和它“有效”沟通：**每次都给它看一张写着我们身份和历史的卡片**。

常见的实现方式有：

1. **Cookie**：
   - 这是最经典、最常用的方式。
   - **流程**：
     1. 用户第一次登录成功后，服务器会生成一个唯一标识（Session ID），并通过响应头（Set-Cookie）把它发送给浏览器。
     2. 浏览器会自动保存这个 Cookie。
     3. 在后续的每一次请求中，浏览器都会**自动**地把这个 Cookie 放在请求头里一起发送给服务器。
   - 服务器每次收到请求，就检查请求头中的 Cookie，通过这个唯一标识就能从自己的数据库或缓存中查到该用户的会话信息（如登录状态、购物车内容等），从而实现了“记住”用户的效果。
2. **Session**：
   - Session 是一种服务器端的机制，它需要与 Cookie 配合。服务器为每个用户创建一个 Session 对象来存储状态信息，而 Cookie 中只存放这个 Session 的唯一ID。
3. **Token（如 JWT - JSON Web Tokens）**：
   - 在现代 Web 应用（特别是前后端分离的 SPA 和移动应用）中非常流行。
   - 服务器在用户认证成功后，会生成一个加密的 Token（令牌）返给客户端。这个 Token 本身就包含了用户的身份信息（以及可能的权限、过期时间等）。
   - 客户端在后续请求中，通常会在请求头（Authorization 字段）中携带这个 Token。服务器收到后，只需验证 Token 的签名是否有效，即可确认用户身份，无需再去查询数据库。

## 在浏览网页时，http如何升级成websocket的

---

### 二、技术核心：升级握手 (The Upgrade Handshake)

这个“升级”的过程，本质上是一个 cleverly-designed 的 HTTP 请求和响应。它利用了 HTTP 协议自身的一个特性，即协议升级机制。

#### 第1步：客户端发起“升级”请求

这一切始于一个由客户端（通常是浏览器中的 JavaScript）发起的、看起来很普通的 HTTP GET 请求。但它的“魔力”在于包含了几个特殊的 **HTTP 头信息 (Headers)**。

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
```

我们来解读这几个关键的“魔法”头信息：

*   `Upgrade: websocket`: 这是最明确的信号。客户端直截了当地告诉服务器：“我不想只做一次普通的HTTP通信，我想把这条连接升级成 WebSocket 协议。”
*   `Connection: Upgrade`: 这是一个标准的 HTTP/1.1 头，它告诉网络中的任何代理或服务器，这次连接的性质即将改变，不要在这次请求后就关闭它。
*   `Sec-WebSocket-Key`: 这是为了安全起见的一个**一次性随机密钥**。客户端会生成一个随机的、经过 Base64 编码的字符串。它的作用不是加密，而是为了确保服务器是一个真正懂 WebSocket 协议的服务器，而不是一个老的、不支持此协议的 HTTP 服务器。
*   `Sec-WebSocket-Version: 13`: 指定了客户端期望使用的 WebSocket 协议版本，目前 13 是最广泛使用的标准版本。

#### 第2步：服务器响应“升级”请求

如果服务器理解并同意这个升级请求，它会返回一个非常特殊的 HTTP 响应。这个响应的状态码不是我们常见的 `200 OK`。

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

解读关键的响应信息：

*   `HTTP/1.1 101 Switching Protocols`: 这个状态码是整个握手的核心。它明确地告诉客户端：“我同意你的请求，我们现在正式**切换协议**！”
*   `Upgrade: websocket` 和 `Connection: Upgrade`: 服务器会原样返回这两个头，确认它理解并同意了客户端的升级提议。
*   `Sec-WebSocket-Accept`: 这是服务器对客户端 `Sec-WebSocket-Key` 的一个**数学响应**，也是安全验证的关键。服务器会执行以下计算：
    1.  取客户端发来的 `Sec-WebSocket-Key` 的值。
    2.  将这个值与一个固定的、在 WebSocket 协议规范中定义的“魔法字符串” (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`) 拼接起来。
    3.  计算拼接后字符串的 **SHA-1 哈希值**。
    4.  将这个二进制的哈希值进行 **Base64 编码**。
    5.  将最终的结果作为 `Sec-WebSocket-Accept` 头的值返回。

客户端收到这个响应后，会用同样的方式进行一次计算，如果自己计算出的结果和服务器返回的 `Sec-WebSocket-Accept` 值完全一致，客户端就知道，对方确实是一个合法的、支持 WebSocket 的服务器。握手成功！

## 工厂模式

好的，工厂模式是面向对象设计中最基础、最重要，也是在 Objective-C 开发中应用最广泛的设计模式之一。理解它有助于编写出更灵活、更解耦、更易于维护的代码。

在 Objective-C 中，工厂模式主要有三种经典的变体：**简单工厂模式、工厂方法模式、以及抽象工厂模式**。

---

### **一、核心思想：将“创建对象”的责任封装起来**

无论哪种工厂模式，它们的核心思想都是一样的：

> **将对象的创建过程（`[[ClassName alloc] init]`）从使用它的地方（客户端代码）中隔离出来，封装到一个专门负责创建的“工厂”里。客户端不再直接依赖于具体的类，而是向工厂请求一个对象。**

**这样做的好处是什么？**
*   **解耦 (Decoupling)**: 客户端代码只知道它需要一个“产品”（比如一个 `UIView`），但它不需要知道这个产品具体是哪个子类（`RedButtonView`, `BlueButtonView`），也不需要知道创建它的复杂过程。
*   **灵活性和可扩展性**: 如果未来需要增加一个新的产品类型（比如 `GreenButtonView`），我们只需要修改工厂内部的逻辑，而**完全不需要改动任何使用这个工厂的客户端代码**。这非常符合**“对扩展开放，对修改关闭”**的开闭原则。

---

### **二、简单工厂模式 (Simple Factory Pattern)**

这是最常用、最直观的一种工厂模式，有时也被称为“静态工厂方法模式”。

*   **结构**: 通常由一个**单独的工厂类**，以及一个**静态的（类）方法**组成。这个方法根据传入的参数，来决定到底要创建并返回哪一个具体的子类实例。
*   **比喻**: 就像一个“汽水机”。你投币并按下一个按钮（**传入参数**，比如“可乐”或“雪碧”），汽水机（**工厂类**）内部就会根据你的选择，掉出一瓶对应的汽水（**具体产品实例**）。你只关心拿到汽水，不关心汽水机内部的机械构造。

**代码示例：**

假设我们要根据不同的数据类型，创建不同的 `UIView` 来展示。

1.  **定义一个产品基类 (或协议)**
    ```objc
    // BaseView.h
    @interface BaseView : UIView
    - (void)configureWithData:(NSDictionary *)data;
    @end
    ```

2.  **创建具体的产品子类**
    ```objc
    // ImageView.h, TextView.h ...
    @interface ImageView : BaseView @end
    @interface TextView : BaseView @end
    ```

3.  **创建工厂类**
    ```objc
    // ViewFactory.h
    typedef NS_ENUM(NSInteger, ViewType) {
        ViewTypeImage,
        ViewTypeTest
    };
    
    @interface ViewFactory : NSObject
    // 这就是工厂方法
    + (BaseView *)createViewWithType:(ViewType)type;
    @end
    
    // ViewFactory.m
    #import "ImageView.h"
    #import "TextView.h"
    
    @implementation ViewFactory
    + (BaseView *)createViewWithType:(ViewType)type {
        BaseView *view = nil;
        switch (type) {
            case ViewTypeImage:
                view = [[ImageView alloc] init];
                break;
            case ViewTypeTest:
                view = [[TextView alloc] init];
                break;
            // ... 如果未来有新的 View 类型，只需在这里加一个 case
        }
        return view;
    }
    @end
    ```

4.  **客户端使用**
    ```objc
    // Client code
    ViewType typeFromServer = ...; // 从服务器获取到要展示的类型
    BaseView *myView = [ViewFactory createViewWithType:typeFromServer];
    [self.view addSubview:myView];
    ```
    客户端代码完全不知道 `ImageView` 或 `TextView` 的存在。

**优点**: 简单、直观、有效。
**缺点**: 如果要增加新的产品类型，必须**修改工厂类的代码**，这在一定程度上违反了开闭原则。

---

### **三、工厂方法模式 (Factory Method Pattern)**

工厂方法模式是对简单工厂的“升级”，它更好地遵循了开闭原则。

*   **结构**: 它不再有一个全能的“中央工厂”。而是定义一个**创建对象的接口（一个方法）**，但将**具体的创建过程延迟到子类**中去实现。
*   **比喻**: 不再是一个万能的汽水机。而是有一个“饮料工厂”的**设计蓝图**（**抽象工厂类**），这个蓝图规定了所有工厂都必须有一个叫做 `produceDrink` 的车间（**工厂方法**）。
    *   “可口可乐工厂”（**具体工厂子类**）继承这个蓝图，并实现了 `produceDrink` 车间，专门用来生产可乐。
    *   “百事可乐工厂”（**另一个具体工厂子类**）也继承这个蓝图，实现了 `produceDrink` 车间，专门用来生产百事。

**代码示例：**

1.  **定义抽象工厂和工厂方法**
    ```objc
    // BaseViewFactory.h (抽象工厂)
    @interface BaseViewFactory : NSObject
    // 这就是工厂方法，由子类去实现
    - (BaseView *)createView; 
    @end
    ```

2.  **创建具体工厂子类**
    ```objc
    // ImageViewFactory.h
    @interface ImageViewFactory : BaseViewFactory
    @end
    
    // ImageViewFactory.m
    @implementation ImageViewFactory
    - (BaseView *)createView {
        return [[ImageView alloc] init]; // 只负责创建 ImageView
    }
    @end
    
    // TextViewFactory.h
    @interface TextViewFactory : BaseViewFactory
    @end
    
    // TextViewFactory.m
    @implementation TextViewFactory
    - (BaseView *)createView {
        return [[TextView alloc] init]; // 只负责创建 TextView
    }
    @end
    ```

3.  **客户端使用**
    ```objc
    // Client code
    BaseViewFactory *factory = nil;
    if (/* a condition */) {
        factory = [[ImageViewFactory alloc] init];
    } else {
        factory = [[TextViewFactory alloc] init];
    }
    
    BaseView *myView = [factory createView];
    [self.view addSubview:myView];
    ```

**优点**: **完全符合开闭原则**。当需要增加一个新产品 `VideoView` 时，我们只需要创建一个新的 `VideoViewFactory` 子类，而**完全不需要修改任何已有的工厂代码**。
**缺点**: 每增加一个产品，就需要增加一个对应的工厂类，导致类的数量成倍增加，系统结构更复杂。

---

### **四、抽象工厂模式 (Abstract Factory Pattern)**

这是最复杂、也是最强大的工厂模式。它用来处理**“产品族”**的创建。

*   **结构**: 提供一个接口，用于创建**一系列相关或相互依赖的对象**，而无需指定它们具体的类。
*   **比喻**: 你要去装修房子，有“中式风格”和“现代风格”两种选择。
    *   **抽象工厂**: `ThemeFactory` 协议，它定义了必须能生产 `createButton`, `createLabel`, `createWindow` 这些组件。
    *   **具体工厂1**: `ChineseThemeFactory` 类。它生产出来的按钮、标签、窗户都是**中式风格**的。
    *   **具体工厂2**: `ModernThemeFactory` 类。它生产出来的按钮、标签、窗户都是**现代风格**的。
    *   客户端只需要选择一个“风格工厂”，然后通过这个工厂创建出来的所有组件，就自动保证了风格的统一。

**代码示例（概念）：**
```objc
// 抽象工厂协议
@protocol ThemeFactoryProtocol <NSObject>
- (UIButton *)createButton;
- (UILabel *)createLabel;
@end

// 具体工厂
@interface DarkThemeFactory : NSObject <ThemeFactoryProtocol>
- (UIButton *)createButton { /* 返回一个黑色背景的按钮 */ }
- (UILabel *)createLabel { /* 返回一个白色字体的标签 */ }
@end

@interface LightThemeFactory : NSObject <ThemeFactoryProtocol>
- (UIButton *)createButton { /* 返回一个白色背景的按钮 */ }
- (UILabel *)createLabel { /* 返回一个黑色字体的标签 */ }
@end

// 客户端使用
id<ThemeFactoryProtocol> factory = [[DarkThemeFactory alloc] init]; // 选择一个主题
UIButton *myButton = [factory createButton];
UILabel *myLabel = [factory createLabel];
```

**优点**:
*   **保证产品族的兼容性**: 确保了由同一个工厂创建出来的所有产品，在风格或功能上是相互匹配的。
*   **隔离了具体实现**: 切换整个产品族变得非常容易，只需要更换具体工厂的实例即可。
**缺点**:
*   **最复杂**。
*   **扩展新产品困难**: 如果你想在 `ThemeFactory` 中增加一个 `createTextField` 的方法，那么所有的具体工厂子类都必须进行修改，这违反了开闭原则

## tcp通信用了哪些socket函数

`socket()`: 创建通信套接字。

`connect()`: 连接到服务器。

`send()`/`recv()`: 与服务器进行数据通信。

`close()`: 关闭通信套接字。

# 实习内容

## git

com.baidu.jjjjdn

新建本地分支第一件事就是这个 ，相当于给远程创建一个分支

git push --set-upstream origin dev/11.0.5/media/feed00-66115 

1.hash值改到1-100  解决

2.lower和upper没有校验是不是合法 解决

3.rankmode校验是不是合法 解决

4。新消息来了，要取消旧消息 解决

5.打散时长是否生效？

6.智能下切？

实际修改

1.范围改到1-100 解决

2新消息来了，要取消旧消息的延迟切换 解决 

3.rankmode校验是不是合法



git fetch origin 

git reset --hard origin/master

# uaproxy

## SO_ORIGINAL_DST 系统调用

一个tcp连接被 **iptables** 重定向之后，会在接字上设置一个特殊的选项 `SO_ORIGINAL_DST`

相当于暴露出一个特殊的套接字api，允许程序获取被 iptables 重定向前的原始目标地址和端口

## 什么是dial方法，什么是ipid检测，为什么可以解决这个反检测

```java
传统代理模式：
客户端 → [修改数据包] → 转发 → 目标服务器
        (保持原始 IP ID 序列，可能暴露不一致)

uaProxy 透明代理：
客户端 → [建立新连接] → 目标服务器
        (全新 IP ID 序列，完全独立)
```

P-ID（IP Identification）是IP协议头中的一个16位字段。它的**本来目的**是为了网络分片和重组。当一个tcp数据包太大超过mtu（**最大传输单元**），需要被拆分成多个小“碎片”在网络上传输时，这些碎片会有相同的IP-ID，以便接收方知道它们同属于一个原始数据包，并将它们重新拼装起来。

但是，它有一个对我们这个问题至关重要的**副作用**：

**为了保证唯一性，同一个操作系统内核在发送IP包时，会为每个包生成一个IP-ID。这个ID通常是全局、依次递增的。**

- 一台Windows电脑发出的包，其IP-ID可能是：1024, 1025, 1026, 1027...
- 一台Linux电脑发出的包，其IP-ID可能是：553, 554, 555, 556... (或者是随机的，但有规律)

路由器是 转发一个http请求，它不会重新建立 TCP 连接，只是根据路由表转发 IP 数据包

因为使用 dial 方法向真实的目标服务器建立 全新的 TCP 连接，然后逐个解析原本的tcp连接中的http请求，然后重新把这个http请求发出去。所以这样就会重新分包，分包出来都显示的是连续的ipid，不存在不连续的问题。

## iptables是什么，如何设置的规则、

* `iptables` 是 **Linux 系统内核 Netfilter 框架的命令行管理工具**，它主要负责**配置防火墙规则**。

  ------

  它的核心功能包括：

  - **数据包过滤 (Packet Filtering)：** 允许或阻止特定网络数据包的进出。
  - **网络地址转换 (NAT)：** 修改数据包的源或目标 IP 地址/端口。比如，SNAT 让内网多台设备共用一个公网 IP 上网；DNAT 则能将外部请求重定向到内部服务器，这是实现**透明代理**的关键。
  - **数据包修改 (Mangle)：** 给数据包打标记或修改其内容，用于更高级的流量控制。

创建一个 `uaProxy` 链来包含你的代理规则。

在 `uaProxy` 链中，首先放行**发往内网**的流量。

然后放行**已标记为代理自身流量**的包，防止回环。

**所有剩余的 TCP 流量**都会被重定向到你本地的 `12345` 端口，即你的 Go 代理程序。

最后，在 `PREROUTING` 链（用于转发其他设备的流量）和 `OUTPUT` 链（用于本机发出的流量）中，都设置跳转到 `uaProxy` 链，从而实现**对所有 TCP 流量的透明代理**。

# 操作系统

## 什么是内存碎片

## 虚拟内存是什么

## 进程调度算法有哪些？

---

### 1. 先来先服务 (First-Come, First-Served, FCFS)

这是最简单的一种调度算法。

*   **核心思想**：按照进程到达就绪队列的先后顺序进行调度。谁先来，谁就先被服务。
*   **生活中的例子**：在银行排队叫号，号码靠前的先办理业务。
*   **工作方式**：维护一个先进先出 (FIFO) 的队列。
*   **优点**：
    *   实现简单，容易理解。
    *   对所有进程都相对公平。
*   **缺点**：
    *   **效率低下**：平均等待时间可能很长。
    *   **护航效应 (Convoy Effect)**：如果一个耗时很长的进程先到达，它会一直占用 CPU，导致后面许多耗时很短的进程不得不长时间等待，就像一辆慢速卡车堵住了一整条高速公路。
*   **类型**：非抢占式 (Non-Preemptive)，即一个进程一旦获得 CPU，就会一直运行直到它完成或主动放弃（如等待 I/O）。

---

### 2. 最短作业优先 (Shortest Job First, SJF)

*   **核心思想**：优先选择**预计运行时间最短**的进程来执行。
*   **生活中的例子**：在超市结账时，你可能会选择排在只买了一两件商品的人后面，而不是排在购物车堆满的人后面。
*   **工作方式**：从就绪队列中找出预计运行时间最短的那个进程，并分配 CPU。
*   **优点**：
    *   **平均等待时间最短**：在所有算法中，SJF 被证明是能获得最低平均等待时间的。
*   **缺点**：
    *   **难以预测运行时间**：在实际系统中，很难精确地知道一个进程到底需要运行多久。通常只能靠历史数据进行估算。
    *   **对长作业不友好**：如果不断有新的短作业到来，长作业可能永远得不到执行，导致“饥饿”(Starvation) 现象。
*   **类型**：
    *   **非抢占式 SJF**：一旦开始，就必须运行到底。
    *   **抢占式 SJF** (也称为 **最短剩余时间优先, Shortest Remaining Time First, SRTF**)：如果一个新到达的进程比当前正在运行的进程的**剩余时间**还要短，则会中断当前进程，立即执行新来的短进程。

---

### 3. 优先级调度 (Priority Scheduling)

*   **核心思想**：为每个进程分配一个优先级，调度器总是选择优先级最高的进程来执行。
*   **生活中的例子**：飞机场的头等舱/VIP 乘客可以优先登机。
*   **工作方式**：就绪队列中的进程按照优先级排序。
*   **优点**：
    *   **灵活性高**：可以根据进程的重要性来决定其处理顺序，满足紧急任务的需求。
*   **缺点**：
    *   **可能导致饥饿**：低优先级的进程可能永远无法被调度。
    *   **解决方案**：可以采用**老化 (Aging)** 技术，即随着时间的推移，逐步提高那些长时间等待的进程的优先级。
*   **类型**：可以是抢占式的，也可以是非抢占式的。抢占式指当一个更高优先级的进程到来时，可以中断当前正在运行的低优先级进程。

---

### 4. 时间片轮转 (Round Robin, RR)

这是一种专门为分时系统设计的算法。

*   **核心思想**：将 CPU 的时间划分成一个个小的时间片 (Time Quantum/Slice)，公平地分配给每个进程。
*   **生活中的例子**：几个小朋友轮流玩一个玩具，每个人玩一分钟，时间到了就换下一个人。
*   **工作方式**：
    1.  所有就绪进程排成一个队列。
    2.  调度器选择队首进程，让它运行一个时间片。
    3.  如果进程在一个时间片内完成，就直接退出；如果没完成，它会被移到队尾，等待下一轮。
*   **优点**：
    *   **公平性好**：每个进程都有机会执行。
    *   **响应时间快**：对于交互式应用（如用户界面）非常友好，用户不会感觉程序被“卡死”。
*   **缺点**：
    *   **上下文切换开销**：时间片设置得太短，会导致频繁的进程上下文切换，消耗大量系统资源，降低效率。
    *   **性能权衡**：时间片设置得太长，算法就退化成了 FCFS，响应时间变差。
*   **类型**：抢占式。

---

### 5. 多级队列调度 (Multilevel Queue Scheduling)

* 好的，我们来详细地剖析一下**多级队列调度 (Multilevel Queue Scheduling)** 算法。

  ### 一、核心思想：分而治之，专窗专用

  想象一下一个大型银行，如果所有客户——无论是存100块钱的普通储户，还是办理上千万对公业务的企业家——都在同一个大厅里排同一条队，那效率肯定会非常低下。

  银行的聪明做法是开设**专窗**：
  *   **普通个人业务窗口**：处理速度快，业务简单，排队的人多。
  *   **VIP理财窗口**：服务要求高，业务复杂，排队的人少。
  *   **企业对公窗口**：业务流程长，需要大量审核。

  **多级队列调度的思想与此完全一样**。它不是把所有进程都放在一个大池子里，而是根据进程的**类型或特性**，将它们预先分类，放入不同的、独立的队列中。每个队列都可以有自己的一套“服务规则”。

  这种算法的目标是：**为不同类型的进程提供最适合它们的调度策略，从而优化整个系统的性能。**

  ---

  ### 二、工作机制详解

  多级队列调度的实现包含三个关键部分：

  **1. 队列的划分 (Partitioning the Queues)**

  首先，需要将就绪队列拆分成多个独立的队列。这种划分通常基于进程的特征，例如：
  *   **系统进程 (System Processes)**：如操作系统内核任务，优先级最高。
  *   **交互式进程 (Interactive Processes)**：如文本编辑器、图形界面应用。它们需要快速响应，对等待时间敏感。
  *   **批处理进程 (Batch Processes)**：如科学计算、数据备份。它们对响应时间不敏感，但需要长时间占用 CPU。
  *   **学生进程 (Student Processes)**：在教学系统中，可能会限制学生程序的资源。

  **2. 队列内部的调度 (Scheduling within a Queue)**

  每个队列都可以拥有**自己独立的调度算法**。这是该算法灵活性的体现。
  *   **交互式队列**：通常使用**时间片轮转 (Round Robin, RR)** 算法，以保证每个进程都能快速得到响应。
  *   **批处理队列**：通常使用**先来先服务 (FCFS)** 算法，因为响应时间不重要，简单处理即可。

  **3. 队列之间的调度 (Scheduling among the Queues)**

  当多个队列里都有进程在等待时，调度器必须决定先服务哪个**队列**。主要有两种方式：

  *   **固定优先级抢占式调度 (Fixed-Priority Preemptive Scheduling)**：这是**最常见**的方式。
      *   为每个队列分配一个固定的优先级（例如，交互式队列 > 批处理队列）。
      *   调度器**总是**先处理高优先级队列中的所有进程。
      *   只有当所有高优先级队列都为空时，调度器才会去处理低优先级队列。
      *   **抢占**：如果一个低优先级队列的进程正在运行时，一个更高优先级队列中突然来了一个新进程，那么低优先级的进程会立即被中断（被抢占），CPU 会被分配给那个新来的高优先级进程。

  *   **时间片划分 (Time Slicing)**：
      *   在队列之间也划分时间。例如，CPU 时间的 80% 分配给交互式队列，20% 分配给批处理队列。
      *   这种方式可以确保低优先级队列不会被完全“饿死”。

  ---

  ### 三、一个具体的例子

  假设一个系统设置了三个队列：

  1.  **队列1 (最高优先级)**：系统进程，使用 **FCFS** 算法。
  2.  **队列2 (中等优先级)**：交互式进程，使用 **RR** 算法（时间片=10ms）。
  3.  **队列3 (最低优先级)**：批处理进程，使用 **FCFS** 算法。

  **调度流程如下：**

  1.  调度器首先检查**队列1**。只要队列1中有进程，就一直按 FCFS 规则执行它们，直到队列1为空。
  2.  当队列1为空时，调度器接着检查**队列2**。它会按 RR 规则轮流执行队列2中的进程。
  3.  如果在执行队列2的进程时，队列1突然来了一个新的系统进程，那么队列2的当前进程会被**立即抢占**，CPU 转而去执行队列1的新进程。
  4.  只有当队列1和队列2都为空时，调度器才有机会去执行**队列3**中的批处理进程。
  5.  同理，如果一个批处理进程正在运行，此时任何一个交互式进程或系统进程就绪，这个批处理进程都会被立刻抢占。

  

  ---

  ### 四、优点和缺点

  **优点：**

  *   **针对性强，开销低**：可以为不同类型的进程量身定制调度策略，满足它们的需求（例如，交互式进程的快速响应）。
  *   **调度开销相对较小**：因为进程在创建时就已经被分配到固定队列，调度器只需在队列内部进行选择，而不需要在所有进程中进行搜索和比较。

  **缺点：**

  *   **缺乏灵活性 (Inflexible)**：这是它**最致命的缺陷**。一旦一个进程被分配到某个队列，它就**永远**待在那里了。如果系统错误地将一个 CPU 密集型进程（本应是批处理）识别为了交互式进程，它就会留在高优先级队列中，占用大量 CPU 时间，影响其他真正的交互式进程。
  *   **可能导致饥饿 (Starvation)**：在使用固定优先级调度时，如果高优先级队列（如交互式队列）一直有进程进入，那么低优先级队列中的进程可能永远也得不到执行的机会，从而被“饿死”。




| 算法名称               | 核心思想       | 抢占式？ | 优点               | 缺点                           |
| :--------------------- | :------------- | :------- | :----------------- | :----------------------------- |
| **先来先服务 (FCFS)**  | 按到达顺序     | 非抢占   | 简单、公平         | 效率低，有护航效应             |
| **最短作业优先 (SJF)** | 优先处理短作业 | 可选     | 平均等待时间最短   | 难预测运行时间，可能饿死长作业 |
| **优先级调度**         | 按优先级高低   | 可选     | 灵活，满足紧急需求 | 可能饿死低优先级进程           |
| **时间片轮转 (RR)**    | 公平分配时间片 | 抢占     | 响应快，公平       | 上下文切换有开销               |
| **多级队列**           | 按进程类型分组 | -        | 针对性强           | 不够灵活，进程无法移动         |
|                        |                |          |                    |                                |

## 有哪些页面置换算法

## 分页和分段有什么区别



## java里面的线程和操作系统的线程一样吗

## 进程切换和线程切换的区别？

* 进程切换：

## ***进程通信和线程通信区别是什么

## ***如何让两个不同进程的线程进行通信

## ***进程进行通信方式

* **共享内存和信号量**
* **管道**：通信效率相对较低，涉及到内核缓冲区的数据拷贝。顺序访问，不支持随机读取。
* **消息队列**：队列的长度和消息的大小可能受到系统限制。
* **Socket**：可以用于不同机器上的进程通过网络进行通信.从本质上讲，Socket 可以看作是**应用程序和网络协议栈之间的一个接口**。它隐藏了底层复杂的网络协议细节（如 TCP/IP 协议族），为应用程序提供了一组简单的 API（例如 `send`、`receive`、`connect`、`accept` 等），使得开发者可以方便地进行网络数据传输，而无需深入了解网络协议的实现。、

## select,poll, epoll 

好的，我们来详细地、深入地剖析 `select`。`select` 是 I/O 多路复用技术的“开山鼻祖”，虽然它在性能上有很多局限性，但理解它的工作原理，对于掌握整个 I/O 模型的发展脉络至关重要。

`select` 的核心思想是：**让程序能够同时监视多个文件描述符（`fd`），并在其中任何一个 `fd` 准备好进行 I/O 操作时，得到通知，从而避免了对每个 `fd` 进行单独的、阻塞式的轮询。**

---

### **一、`select` 的核心 API 组件**

`select` 的使用主要围绕一个核心函数和一套宏定义来操作其专用的数据结构 `fd_set`。

#### **1. 数据结构：`fd_set` (文件描述符集合)**

*   **本质**: `fd_set` 的底层实现就是一个**整型数组**，但我们通常把它想象成一个**位图 (Bitmap)**。
    ```c
    // 简化理解
    // 假设一个 long 是 32 位
    // FD_SETSIZE 通常是 1024
    long fds_bits[1024 / 32]; 
    ```
*   **作用**: 这个位图的**每一位（bit）**都代表一个文件描述符。比如，第 `i` 位就代表 `fd=i`。
*   **大小限制**: `fd_set` 的大小在编译时是固定的，由 `FD_SETSIZE` 这个宏定义决定，在大多数系统中，这个值是 **1024**。这意味着 `select` **最多只能同时监视 1024 个文件描述符**（从 0 到 1023）。这是它最著名的、也是最致命的缺陷之一。

#### **2. 操作 `fd_set` 的宏**

由于 `fd_set` 的具体实现可能因系统而异，我们不能直接去操作它的位。POSIX 标准提供了一套标准的宏来安全地操作它：

*   `FD_ZERO(fd_set *set)`: **清空**一个 `fd_set`。将位图的所有位都设置为 0。这是每次使用前的**初始化**步骤。
*   `FD_SET(int fd, fd_set *set)`: **添加**一个 `fd` 到集合中。将位图中代表 `fd` 的那一位设置为 1。
*   `FD_CLR(int fd, fd_set *set)`: 从集合中**移除**一个 `fd`。将位图中代表 `fd` 的那一位设置为 0。
*   `FD_ISSET(int fd, fd_set *set)`: **检查**一个 `fd` 是否仍然在集合中（即对应的位是否为 1）。这个宏在 `select` 返回后，用来**找出**哪些 `fd` 准备好了。

#### **3. 核心函数：`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)`**

*   **作用**: 这是阻塞等待事件发生的核心函数。
*   **参数详解**:
    *   `nfds`: **最重要的参数之一**。它指定了被监视的所有 `fd` 中，**最大的那个 `fd` 的值加 1**。内核只需要扫描到这个位置即可，后面的就不用管了，这是一个小小的性能优化。
    *   `readfds`: 你**关心“可读”事件**的 `fd` 集合。当这个集合里的任何一个 `fd` 有数据可读时，`select` 就会返回。
    *   `writefds`: 你**关心“可写”事件**的 `fd` 集合。当这个集合里的任何一个 `fd` 的发送缓冲区不再满，可以写入数据时，`select` 就会返回。
    *   `exceptfds`: 你**关心“异常”事件**的 `fd` 集合。
    *   `timeout`: 设置**超时时间**。
        *   如果设为 `NULL`，`select` 会**永远阻塞**，直到有 `fd` 准备好。
        *   如果设为一个具体的时间值（比如 2.5 秒），`select` 最多阻塞这么久，超时后即使没有 `fd` 准备好，也会返回。
        *   如果设为 0，`select` 会立即返回，变成一个**非阻塞**的轮询。

---

### **二、`select` 的完整工作流程**

让我们用一个典型的服务器程序来走一遍 `select` 的完整流程。

**前提**: 服务器已经创建了一个 `listen_fd` 用于监听新的连接，并且已经接受了几个客户端连接，它们的 `fd` 分别是 `conn_fd1`, `conn_fd2`...

```c
// 伪代码
while (true) {

    // 1. 【用户空间】准备 fd_set (每次循环都必须重新准备)
    fd_set read_fds;
    FD_ZERO(&read_fds); // 清空集合
    
    FD_SET(listen_fd, &read_fds); // 把监听 socket 加入集合，关心新连接事件
    
    int max_fd = listen_fd;
    for (int i = 0; i < client_count; i++) {
        FD_SET(clients[i].fd, &read_fds); // 把所有已连接的客户端 socket 加入集合
        if (clients[i].fd > max_fd) {
            max_fd = clients[i].fd;
        }
    }

    // 2. 【系统调用】调用 select，程序阻塞
    //    此时，发生一次从用户空间到内核空间的上下文切换
    //    read_fds 从用户空间被【拷贝】到内核空间
    int ready_count = select(max_fd + 1, &read_fds, NULL, NULL, NULL);

    // ---- 程序被内核唤醒，从这里继续执行 ----
    
    if (ready_count < 0) {
        // 发生错误
        perror("select error");
        break;
    }
    
    // 3. 【用户空间】找出哪些 fd 准备好了
    
    // 检查监听 fd 是否有新连接
    if (FD_ISSET(listen_fd, &read_fds)) {
        // accept a new connection...
    }

    // 遍历所有客户端 fd，检查是否有数据可读
    for (int i = 0; i < client_count; i++) {
        if (FD_ISSET(clients[i].fd, &read_fds)) {
            // read data from this client...
        }
    }
}
```

#### **内核层面的工作**

当 `select` 被调用后，内核会做什么？
1.  **接收参数**: 内核接收到从用户空间**拷贝**过来的 `read_fds`。
2.  **线性扫描/轮询**: 内核会**从头到尾遍历**这个 `read_fds` 位图（从 0 到 `nfds-1`）。对于每一个被标记为 1 的 `fd`，内核都会去检查它对应的设备驱动程序，问：“你准备好读取了吗？”
3.  **休眠与唤醒**:
    *   如果一轮扫描下来，没有任何 `fd` 准备好，那么调用 `select` 的进程就会被置为**休眠**状态。
    *   当某个被监视的 `fd` 对应的硬件（如网卡）接收到数据时，会产生一个**硬件中断**。内核的中断处理程序会唤醒那个正在休眠的进程。
4.  **修改并拷贝返回**: 进程被唤醒后，内核会**再次遍历**一遍所有的 `fd`，更新 `read_fds` 的内容（只保留那些真正就绪的 `fd` 的标记），然后将这个修改后的 `fd_set` **拷贝回**用户空间。

---

### **三、`select` 的三大性能瓶颈分析**

通过上面的流程，我们可以清晰地看到 `select` 的三大性能瓶颈：

1.  **固定大小的连接数限制**: `FD_SETSIZE` (通常是 1024) 这个硬性上限，使得 `select` 无法用于需要处理上万并发连接的现代高性能服务器（C10K 问题）。

2.  **昂贵的内存拷贝**: `fd_set` 是一个**“输入输出参数”**。这意味着：
    *   **调用前**: 你需要把它从用户空间**拷贝**到内核空间，告诉内核你要监视谁。
    *   **调用后**: 内核需要把它**修改后**的副本，再从内核空间**拷贝**回用户空间，告诉你结果。
    *   当 `FD_SETSIZE` 是 1024 时，即使你只关心 2 个连接，每次调用也需要拷贝 `1024 / 8 = 128` 字节的数据。当连接数接近上限时，这个拷贝开销会非常显著。

3.  **低效的 `O(n)` 扫描**:
    *   **内核层面**: 无论有多少个连接是活跃的，内核每次都需要**线性扫描**所有你告诉它要监视的 `fd`。
    *   **用户层面**: `select` 返回后，它只告诉你“有 `ready_count` 个 `fd` 准备好了”，但**不告诉你具体是哪几个**。你必须自己**再次线性扫描**整个 `fd_set`（从 0 到 `max_fd`），用 `FD_ISSET` 挨个去检查，才能找出那些就绪的 `fd`。
    *   当总连接数很大，但同一时间只有少数连接活跃时，这两次扫描的开销是巨大的性能浪费。

好的，我们来深入地、详细地剖析 `epoll`。`epoll` 之所以被誉为 Linux 下高性能网络编程的“神器”，是因为它在设计思想上对 `select` 和 `poll` 进行了**根本性的颠覆**。

理解 `epoll` 的关键，在于理解它的**三大核心组件**、**两种工作模式**以及它如何通过**“事件驱动”**和**“共享内存”**的思想来解决性能瓶颈。

---

### **一、`epoll` 的三大核心 API 组件**

`epoll` 将 `select/poll` 的“一次性提交所有任务”的模式，拆分成了三个独立的、各司其职的步骤。

#### **1. `int epoll_create(int size)`**

*   **作用**: 在 Linux 内核中创建一个 `epoll` 实例，并返回一个指向它的文件描述符（我们称之为 `epfd`）。
*   **比喻**: 这就像在总机系统里为你**开辟一个专属的、高级的“监控室”**。这个 `epfd` 就是进入这个监控室的钥匙。
*   **内部结构**: 这个“监控室”里，内核为你维护了两个核心的数据结构：
    1.  **红黑树 (Red-Black Tree)**: 用来存放你所有**“感兴趣”**的 socket `fd`。红黑树保证了即使在高并发下，对这个列表进行增、删、改、查的操作，其时间复杂度都是高效的 `O(log n)`。这就是你的**“关注列表 (interest list)”**。
    2.  **双向链表 (Doubly Linked List)**: 用来存放所有已经**“准备就绪”**的 `fd`。这就是你的**“就绪列表 (ready list)”**。
*   **`size` 参数**: 在早期的内核版本中，这个参数用来告诉内核你大概要监控多少个 `fd`，以便内核分配初始内存。但在现代 Linux 内核中，这个参数**已经被忽略**，内核会自动动态调整大小。你只需要传入一个大于 0 的数即可。

#### **2. `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)`**

*   **作用**: 对 `epfd` 所代表的那个“监控室”里的“关注列表”进行**增、删、改**操作。
*   **比喻**: 这是你用来**管理“关注列表”**的对讲机。
*   **参数详解**:
    *   `epfd`: `epoll_create` 返回的那个钥匙。
    *   `op`: 你要执行的操作，主要有三种：
        *   `EPOLL_CTL_ADD`: **添加**一个新的 `fd` 到关注列表。
        *   `EPOLL_CTL_MOD`: **修改**一个已存在的 `fd` 所关注的事件。
        *   `EPOLL_CTL_DEL`: 从关注列表中**删除**一个 `fd`。
    *   `fd`: 你要操作的具体那个 socket `fd`。
    *   `event`: 一个 `epoll_event` 结构体，用来告诉内核你关心这个 `fd` 的什么事件（比如 `EPOLLIN` - 可读，`EPOLLOUT` - 可写）。

*   **核心优势**: 这个操作是**增量式**的。一旦你通过 `epoll_ctl` 把一个 `fd` 加入了内核的关注列表，除非你再次调用 `epoll_ctl` 去修改或删除它，否则它就一直存在于内核中。你**不需要**在每次等待事件时，都把整个列表重新提交一遍。这就**根除了 `select/poll` 的重复内存拷贝问题**。

#### **3. `int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`**

*   **作用**: 这是**阻塞等待**事件发生的主循环函数。
*   **比喻**: 这是你坐在监控室里，**等待警报响起**的动作。
*   **工作原理 (事件驱动)**:
    1.  你调用 `epoll_wait`，如果“就绪列表”是空的，你的程序就会**阻塞（睡觉）**。
    2.  在内核层面，当一个被你关注的 `fd` 对应的设备（比如网卡）接收到数据时，会产生一个**硬件中断**。
    3.  内核的中断处理程序被唤醒，它处理完数据后，会检查这个 `fd` 是否在某个 `epoll` 实例的“关注列表”中。
    4.  如果存在，内核就会把这个 `fd` **添加**到与 `epfd` 关联的那个**“就绪列表”**中。这个过程是**内核主动完成的，而不是 `epoll_wait` 去轮询发现的**。
    5.  一旦“就绪列表”不为空，`epoll_wait` 就会被**唤醒**。
    6.  `epoll_wait` 的工作非常简单：它只是把“就绪列表”中的 `fd` **拷贝**到你传入的 `events` 这个数组中，并返回实际就绪的 `fd` 数量。

*   **核心优势**:
    *   **没有轮询**: 内核不再需要线性扫描所有被监控的 `fd`。
    *   **高效返回**: `epoll_wait` 返回时，你拿到的 `events` 数组里，**全都是已经准备好的 `fd`**。你不需要再自己遍历一个巨大的集合去查找。
    *   **性能与总连接数无关**: `epoll` 的性能只取决于**当前活跃的连接数**，而不是你监控的总连接数。这使得它能够轻松应对 C10K 甚至 C100K 的挑战。

---

### **三、总结：`epoll` 为什么是革命性的？**

| 对比 `select/poll`   | `epoll` 的解决方案                                           |
| :------------------- | :----------------------------------------------------------- |
| **重复拷贝监控列表** | 通过 **`epoll_ctl` 增量管理**，内核与用户共享一份列表，避免了拷贝。 |
| **内核线性扫描**     | 通过**硬件中断回调**机制，内核不再轮询，而是事件主动上报。   |
| **用户线性扫描**     | `epoll_wait` **直接返回就绪列表**，用户无需自己查找。        |
| **核心思想**         | 轮询模型                                                     |
| **性能瓶颈**         | `O(总连接数)`                                                |



# 数据结构

### 栈的使用场景

* 括号匹配检查
* 函数累计调用里面的栈帧

### 堆

* 出堆：将堆顶元素弹出，然后将数组的最后一个元素（即最右侧的叶子节点）放到堆顶，尝试把他下沉
* 入堆：把元素放到数组最后一个位置，即最右侧的叶子节点，尝试上浮
* 一般用数组实现堆
* 出堆，入堆，删除都是logn的时间复杂度，然后建堆时间复杂度

### 堆的使用场景

* 优先级队列 :操作系统的进程调度、定时任务的执行、异步消息队列中需要优先处理的消息

### 堆求解topk问题：

- 先从n中取出前k个元素，组成一个小顶堆，此时堆顶元素是最小的，我们从K+1个元素开始，和堆顶元素进行比较，如果比堆顶元素小，那么不做处理，继续下一个；如果比堆顶元素大，那么把堆顶元素删除，并插入k+1这个元素；等到n个元素全部遍历完，那么小顶堆中的k个元素就是Top-k数据了；
- 对于外部不断添加进来的数据其实思路是一样的，把它当成数据源，不断地和堆顶元素比较，重复上面的步骤操作即可。

时间主要耗费在一开始k个元素的初始建堆O(logk)上，还有删除堆顶元素和插入新元素时的堆化O(logk)上；所以，总的时间复杂度应该为O(nlogk)，比使用排序来获取Top-k的O(nlogn)还是要高效的。

## 快排求解topk问题

### 红黑树

非常好的问题！红黑树要解决的核心问题，一言以蔽之，就是**防止二叉搜索树（BST）在最坏情况下退化成链表，从而保证其各项操作的时间复杂度维持在 O(log n) 级别**。

为了完全理解这一点，我们需要一个“英雄出场”的故事：

---

### 第一幕：美好的理想 —— 二叉搜索树 (Binary Search Tree)

二叉搜索树（BST）是一个很棒的数据结构，它的定义很简单：
*   任何节点的左子树上的所有值都小于该节点的值。
*   任何节点的右子树上的所有值都大于该节点的值。
*   左右子树也都是二叉搜索树。

**理想情况下的优势**：如果一棵BST是“平衡”的（即左右子树的高度差不多），那么它的查找、插入和删除操作都非常快，时间复杂度都是 **O(log n)**。这几乎和二分查找一样快。



### 第二幕：残酷的现实 —— BST的退化

BST有一个致命的弱点：它的性能完全依赖于数据的**插入顺序**。

**最坏情况**：如果我们插入一组已经排好序的数据，比如 `[10, 20, 30, 40, 50]`，会发生什么？

*   插入10，成为根节点。
*   插入20，比10大，成为10的右孩子。
*   插入30，比10大，比20大，成为20的右孩子。
*   ...以此类推

最后，这棵“树”会变成这个样子：

```
10
  \
   20
     \
      30
        \
         40
           \
            50
```

这棵树已经完全**退化成了一个链表**。

**问题暴露**：在这个退化的树上，如果我们想查找50，我们需要从10开始，一路遍历到底，总共需要比较5次。如果树中有n个节点，查找操作的时间复杂度就从理想的 **O(log n)** 急剧恶化为了 **O(n)**。插入和删除也是如此。这就完全失去了使用树形结构的意义。

---

### 第三幕：英雄登场 —— 红黑树的解决方案

红黑树就是为了解决BST退化问题而生的。它是一种**自平衡的二叉搜索树**。

它在普通BST的基础上，增加了两个东西：

1.  **一个额外的属性**：每个节点都有一个颜色，要么是**红色**，要么是**黑色**。
2.  **一套固定的规则（5条规则）**：
    1.  每个节点要么是红色，要么是黑色。
    2.  根节点是黑色的。
    3.  每个叶子节点（NIL节点，空节点）是黑色的。
    4.  如果一个节点是红色的，那么它的两个子节点必须是黑色的（**不能有两个连续的红色节点**）。
    5.  从任一节点到其每个叶子节点的所有路径都包含相同数目的黑色节点（**黑高一致**）。

#### 红黑树如何解决问题？

红黑树的核心在于，**当进行插入或删除操作，导致上述任何一条规则被破坏时，它会通过一系列的“修正动作”来重新满足这些规则，从而强制让树保持“大致的平衡”**。

这些修正动作主要有两种：

*   **变色 (Recoloring)**：改变节点的颜色。
*   **旋转 (Rotation)**：分为左旋和右旋，这会改变树的局部结构，但不会破坏BST的基本性质。

**核心保证**：正是由于这5条规则，尤其是第4条和第5条，共同保证了红黑树中最长路径（从根到最远的叶子）的长度，不会超过最短路径长度的两倍。这就从数学上杜绝了树退化成链表的可能性，保证了树的高度始终约等于 **log n**。

**最终结果**：因为树的高度被严格控制住了，所以红黑树的**查找、插入、删除操作在任何情况下（包括最坏情况）的时间复杂度都能稳定在 O(log n)**。

# 场景题

## 内存有限，排序一千万条数据，怎么排序

这是一个非常经典的面试题，它考察的是在资源受限的情况下处理大规模数据的能力。解决这个问题的标准方案是 **外部排序 (External Sorting)**。

核心思想非常简单，就是一种“分而治之”（Divide and Conquer）的策略：

1.  **分割 (Split)**：因为不能一次性将所有数据读入内存，所以我们将大文件分割成内存可以容纳的小块。
2.  **内部排序 (Sort)**：将每个小块依次读入内存，使用高效的内存排序算法（如快速排序）进行排序。
3.  **归并 (Merge)**：将这些已经排好序的小块文件，通过一种高效的方式合并成一个最终的有序大文件。

---

### 一个生动的比喻

想象一下，你有一副由1000万张卡片组成的巨大扑克牌堆需要排序，但你只有一张很小的桌子（有限的内存）。

1.  **分批整理（分割与内部排序）**：你从大牌堆里抓一小把你能拿得动的牌（比如100张）放到桌子上。在桌子上，你可以快速地把这100张牌理好顺序。理好后，你把这100张有序的牌堆成一小堆，放到地板上。你重复这个过程，直到把所有的大牌堆都变成了地板上的一堆堆有序的小牌堆。
2.  **合并小牌堆（归并）**：现在地板上有一堆堆已经排好序的小牌堆。你从每个小牌堆的顶上各拿一张牌放在手里。然后，你从手里的牌中选出最小的那一张，放到一个新的、最终的牌堆里。你再从刚刚被拿走牌的那个小牌堆顶上补充一张到手里。不断重复“从手里选最小的 -> 放入最终牌堆 -> 补充一张新牌”这个过程，直到所有小牌堆都被合并完。最终，你就得到了一个完全有序的巨大牌堆。

---

### 技术实现步骤详解

#### **阶段一：分割与内部排序 (Split and In-Memory Sort)**

1.  **确定块大小**：根据可用的内存大小，确定每个数据块（Chunk）的大小。例如，如果内存能容纳100万条数据，我们就以100万条为一块。
2.  **读取与排序**：
    *   从磁盘上的1000万条数据文件中，读取前100万条到内存中。
    *   在内存中，使用高效的排序算法（如 **快速排序 (QuickSort)**、**堆排序 (HeapSort)** 或 **Timsort**）对这100万条数据进行排序。
    *   将排序好的这100万条数据写入磁盘上的一个临时文件（例如 `temp_file_1.sorted`）。
3.  **重复操作**：重复步骤2，继续读取接下来的100万条数据，排序后写入第二个临时文件（`temp_file_2.sorted`），以此类推。
4.  **完成分割**：直到原始文件的所有数据都被处理完毕。在这个例子中，我们会得到10个临时的、内部已有序的文件。

**这个阶段结束后，我们得到的是 `k` 个已经排好序的临时文件。**

#### **阶段二：多路归并 (Multi-way Merge)**

这是外部排序最核心、最巧妙的部分。我们的目标是将这10个有序的临时文件，合并成一个最终的有序文件，并且这个过程同样不能消耗太多内存。

1.  **输入缓冲区**：为每个临时文件（共10个）在内存中开辟一个小的输入缓冲区。我们只需要从每个文件中读取**第一个**数据元素放入各自的缓冲区。
2.  **选择最小元素**：现在内存中有10个来自不同文件开头的元素。我们需要在这10个元素中找到**最小**的那一个。
    *   **高效实现**：为了高效地找到这`k`个元素中的最小值，我们通常使用一个 **最小堆（Min-Heap）** 数据结构。将这10个元素和它们来源的文件索引一同放入最小堆。堆顶的元素永远是当前所有缓冲区中最小的。
3.  **写入与补充**：
    *   从最小堆中取出堆顶的元素（全局最小值），将它写入到最终的输出文件。
    *   查看这个最小元素来自于哪个临时文件（比如来自 `temp_file_3.sorted`）。
    *   从 `temp_file_3.sorted` 中读取**下一个**元素，并将其放入最小堆中。
4.  **循环归并**：重复步骤3，不断地“从堆顶取最小 -> 写入输出文件 -> 从来源文件补充新元素入堆”，直到所有临时文件的数据都被读取完毕。

**这个阶段结束后，最终的输出文件就是一个包含1000万条数据的、完全有序的文件。**

时间复杂度，假如分成k份，每份 m 个数据，各自排序是k*mlog m = n logm

外部排序的复杂度是，nlogk。

nlogk + n log m = n（log k + log m） = nlogn
