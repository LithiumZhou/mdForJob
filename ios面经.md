# 面经

## ***load和initialize方法

### 1. `+ (void)load` 方法

`load` 方法是在类或分类（Category）被加载到内存时调用的，调用时机非常早。

| 特性         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| **调用时机** | **最早**。在 App 启动时，**程序的主函数 `main()` 执行之前**，只要类或分类被编译到可执行文件中，就会被调用。 |
| **调用次数** | **只调用一次**。由运行时环境自动调用。                       |
| **继承性**   | **不继承**。如果父类和子类都实现了 `load` 方法，那么两个方法都会被调用，**并且父类的 `load` 会先于子类的 `load` 调用**。对于分类，类及其所有分类的 `load` 都会被调用，顺序是：**类 → 分类**（分类的顺序由编译顺序决定） |
|              |                                                              |
| **主要用途** | 进行**方法交换（Method Swizzling）**或**注册（Register）**特定的子类。由于调用时机早且独立，它是执行不可依赖其他类状态的“硬核”初始化工作的理想场所。 |

### 总结：

`load` 方法是进行**运行时配置**（如方法交换）的最佳位置，因为它保证在一切开始之前执行，且不会受到继承的影响。

### 2. `+ (void)initialize` 方法

`initialize` 方法是在类第一次接收到**消息**时调用的，通常比 `load` 晚得多。

| 特性         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| **调用时机** | **懒加载（Lazy Loading）**。在类第一次接收到消息（如调用类方法或实例化对象）之前被调用。 |
| **调用次数** | **只调用一次**。由运行时系统自动调用。                       |
| **继承性**   | **遵循继承链**。如果子类没有实现 `initialize` 方法，会调用**父类**的 `initialize` 方法。**父类的 `initialize` 可能会被调用多次**（每次其子类第一次收到消息时，都会先确保父类被初始化）。为了避免重复初始化，通常需要在方法内部进行**安全检查**。 |
| **安全性**   | **线程安全**。运行时系统保证在调用时是线程安全的。           |
| **主要用途** | **进行类相关的配置**，**如设置静态变量**、初始化全局共享资源等。由于它是在运行时第一次使用时才调用，可以节省启动时间。 |

### `load` 和 `initialize` 的关键区别

| 特性         | `+load`                                    | `+initialize`                                           |
| ------------ | ------------------------------------------ | ------------------------------------------------------- |
| **调用时机** | **App 启动时，`main()` 之前**。            | **类第一次接收消息时**（懒加载）。                      |
| **继承特性** | **不继承**。父类、子类、分类都会独立执行。 | **遵循继承链**。子类会调用父类版本，需手动检查 `self`。 |
| **目的**     | **运行时配置**（如 Method Swizzling）。    | **类配置**、初始化静态/全局变量。                       |
| **适用场景** | 必须在应用启动前完成的**关键设置**。       | 依赖于运行时环境的**按需初始化**。                      |

## ***MRC介绍

### 1. 先明确一点

在 **MRC 下**：

- 用 `alloc/init` 创建的对象，引用计数 = **1**。
- 谁 `retain`，谁负责 `release`。
- 当引用计数 = **0** 时，对象会调用 `dealloc`，被销毁。

------

### 2. 最简单的例子

```objc
Person *p = [[Person alloc] init]; // retainCount = 1

[p release]; // retainCount = 0，对象销毁，调用 dealloc
```

👉 这里很直观：

- `alloc/init` 得到的对象，你要负责调用 `release`，不然就是内存泄漏。

------

### 3. retain 的意义

为什么要 `retain` 呢？
 因为 **一个对象可能会被多个地方使用**。每个使用方如果想“持有”它，就需要 `retain`。

例子：

```objc
Person *p = [[Person alloc] init]; // retainCount = 1

// a 对象想保存这个 person
[a setPerson:p]; // 在 setter 里通常会 [p retain] -> retainCount = 2

// b 对象也想保存这个 person
[b setPerson:p]; // retainCount = 3

// a 不要了
[a setPerson:nil]; // 在 setter 里 release -> retainCount = 2

// b 不要了
[b setPerson:nil]; // retainCount = 1

// main 方法最后自己也不用了
[p release]; // retainCount = 0 -> dealloc
```

👉 你可以理解为：

- `retain` 就像“借书登记”，谁拿了书就要登记。
- `release` 就像“还书”，登记 -1。
- 登记人数 = 0 时，书可以销毁。

------

### 4. setter 方法中的 retain/release

在 MRC 下，属性的管理几乎都写成这样：

```objc
- (void)setCar:(Car *)car {
    if (_car != car) {
        [_car release];   // 旧的 car 不要了，释放一次
        _car = [car retain]; // 保留新的 car
    }
}
```

这样保证：

- 每个持有 car 的对象都有责任 retain/release。
- 避免了内存泄漏和野指针。

------

### 5. 一个完整流程示例

```objc
int main() {
    Person *p = [[Person alloc] init]; // retainCount = 1
    Car *c = [[Car alloc] init];       // retainCount = 1

    [p setCar:c];  // setter: c retainCount = 2
    [c release];   // main 不再持有 -> retainCount = 1

    [p release];   // person dealloc
                   // 在 dealloc 里会 [_car release]; -> c retainCount = 0 -> dealloc
    return 0;
}
```

## ***block的使用场景，如何避免循环引用

这是一个关于 **Block** 非常好的问题。Block 是 Objective-C 中一个强大且常用的特性，它将一段代码及其执行时的上下文（捕获的变量）封装起来，可以在任何时候调用，非常灵活

Block 本质上是**代码块**和**闭包**（Closure），它们最常用于以下需要延迟执行或定义回调的场景：

### 1. 异步操作和多线程回调

在进行耗时的操作（如网络请求、文件I/O）时，为了不阻塞主线程（UI），我们会将这些操作放到子线程中执行。Block 是处理这些操作完成后**回调**和**结果返回**的最佳方式。

- **例子：** 网络请求库（如 `AFNetworking` 或原生 `NSURLSession`）通常使用 Block 来处理请求成功、失败的回调。

  Objective-C

  ```
  [NetworkManager fetchDataWithCompletion:^(NSData *data, NSError *error) {
      if (!error) {
          // 成功：在主线程更新 UI
      } else {
          // 失败：处理错误
      }
  }];
  ```

### 2. 动画和延迟执行

Block 允许您将一组相关的操作定义在一起，并在特定时间或条件满足时执行。

- **例子：** `UIView` 动画（在 iOS 开发中）：

  Objective-C

  ```
  [UIView animateWithDuration:0.3 animations:^{
      // 动画过程中要执行的代码，如修改视图的 frame 或 alpha
      self.myView.alpha = 0.5;
  } completion:^(BOOL finished) {
      // 动画完成后的回调
      [self showNextView];
  }];
  ```

### 3. 替代 Delegate（委托）模式

当一个类只需要一个或少数几个回调时，使用 Block 可以极大地简化代码，避免定义协议和大量可选方法，使代码更加集中和清晰。

- **例子：** 视图跳转传值、自定义控件的点击事件处理等。

### 什么时候用 Block

当您遇到以下情况时，通常是使用 Block 的最佳时机：

1. **需要将一段代码作为参数传递给另一个方法时。**（延迟执行）
2. **需要定义一个操作完成后的**回调（Callback）**机制时。**（如异步操作）
3. **当您要**简化 Delegate（委托）模式**，且回调方法少于三个时。**
4. **需要在代码执行时捕获并使用**当前上下文的变量**时。**

### 3. 避免循环引用（Retain Cycle）

**这是 Block 内存管理中最常见的问题。**

当一个对象（如 `self`）持有一个 Block，而这个 Block 又**强引用**了该对象（`self`）时，就会形成引用循环，导致两者都无法释放（内存泄漏）。

**解决办法：使用 `__weak` 或 `__unsafe_unretained`。**

在 Block 内部需要访问 `self` 或 `self` 的实例变量时，应该先声明一个 `__weak` 引用：

Objective-C

```objc
// 1. 声明一个弱引用的 self
__weak typeof(self) weakSelf = self;

self.myBlock = ^{
    // 2. 在 Block 内部使用弱引用
    [weakSelf doSomething]; 
};
```

## ***assign, weak, strong, copy,atomic,nonatomic

* **assign和weak:**给对象使用这个关键字，他们两个同样都会给指针赋值，不增加引用计数，但是核心区别在于，**weak指向的对象销毁后，weak会自动指向nil**，assign也不会增加引用计数，但是会导致**野指针，即对象被销毁，但是指针还指向那块区域**。并且，由于oc的msg_send机制，oc中不存在空指针异常，会先判断指针是不是为空。
* **strong和copy：** strong赋值时会直接复制指针地址，并且会引用计数加一，修改赋值和背赋值的指针里面的对象，都会互相影响。但是copy指针赋值时，相当于复制出一个新的对象在堆中，然后把被赋值的指针指向新对象，互不干扰。
* **atomic 和 nonatomic**:atomic使得setter和getter方法变成原子性（原理就是在方法上加上互斥锁），nonatomic则不是原子性的。
* **unsafe_unretained**:类似weak，不会增加引用计数，但是对象销毁不会给指针置为nil

## ***nsstring和nsarray用哪种修饰符来修饰，为什么

### 1\. `copy` 修饰符（首选）

对于像 `NSString` 和 `NSArray` 这样的**可变（Mutable）对应物**（如 `NSMutableString` 和 `NSMutableArray`）存在的**不可变（Immutable）对象**，通常建议使用 `copy` 修饰符。

**主要原因是为了防止可变性带来的副作用 (Side Effects)：**

假设您有一个属性声明为 `strong`：

```objective-c
@property (nonatomic, strong) NSString *name;
```

如果调用者传入的是一个 **可变的字符串对象** (`NSMutableString`)：

```objective-c
NSMutableString *mutableName = [NSMutableString stringWithString:@"Initial"];
self.name = mutableName; // name 强引用了 mutableName
// ...
[mutableName appendString:@" Change"]; // 外部修改了 mutableName
// ...
NSLog(@"%@", self.name); // self.name 的值也会跟着改变，变成了 "Initial Change"
```

这违反了您的属性是 **`NSString`** (不可变类型) 的初衷，并可能导致不可预期的行为。

-----

而使用 **`copy`** 修饰符时：

```objective-c
@property (nonatomic, copy) NSString *name;
```

当您给 `name` 赋值时：

```objective-c
NSMutableString *mutableName = [NSMutableString stringWithString:@"Initial"];
self.name = mutableName; // 触发 copy 操作，创建了一个新的 NSString 对象
// ...
[mutableName appendString:@" Change"]; // 外部修改了 mutableName，不会影响 self.name
// ...
NSLog(@"%@", self.name); // self.name 的值是 "Initial"，没有被外部修改
```

**`copy`** 会创建一个属性值（新对象）的**不可变副本**，从而确保了属性的内部值不会被外部修改，保证了数据的**封装性**和**不变性**。对于 `NSArray` 也是同样的道理。

-----

### 2\. `strong` 修饰符（备选）

在某些情况下，也可以使用 **`strong`**：

#### 场景 A：性能优化（不常用）

如果您**确定**传入的值始终是不可变的 `NSString` 或 `NSArray`（例如，您只在类内部创建并赋值），并且您想避免额外的 `copy` 操作带来的性能开销，可以使用 `strong`。

#### 场景 B：可变属性

如果您声明的属性本身就是 **可变类型** (例如 `NSMutableString` 或 `NSMutableArray`)，那么应该使用 **`strong`**，因为您需要的是对**同一个可变对象**的强引用，而不是创建一个不可变的副本。

```objective-c
@property (nonatomic, strong) NSMutableString *mutableName; // 使用 strong
```

## ***NSTimer造成的循环引用

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

### 2. 使用 Block 版本 API (推荐的现代方法)

在 iOS 10 / macOS 10.12 及更高版本中，苹果引入了基于 Block 的 `NSTimer` API，这允许我们使用 **弱引用 (Weak Reference)** 来打破循环。

**Objective-C 代码（使用 Block API）：**

Objective-C

```objc
- (void)startTimer {
    // 创建 MyClass 的弱引用
    __weak typeof(self) weakSelf = self; 

    // 使用 timerWithTimeInterval:repeats:block:
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 
                                                 repeats:YES 
                                                   block:^(NSTimer * _Nonnull timer) {
        // 在 Block 内部使用弱引用或临时强引用
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            [strongSelf timerFired:timer];
        } else {
            // 如果 strongSelf 为空，说明 MyClass 已经被释放了，此时需要停止定时器
            [timer invalidate];
        }
    }];
}
```

在这个方法中：

- `MyClass` 强引用 `NSTimer` (不变)。
- `NSTimer` 强引用它持有的 **Block**。
- **Block** 内部通过 `__weak typeof(self) weakSelf` 来弱引用 `MyClass`。

这样就切断了 `NSTimer` → `MyClass` 的强引用链，从而解决了循环引用问题。

## ***autolayout原理

AutoLayout的原理是基于Cassowary算法的一种实现。Cassowary是一种用于解决用户界面布局问题的算法，它将布局问题抽象成线性等式或不等式约束来进行求解。AutoLayout将视图的布局问题转换为线性等式，然后通过求解这些等式来确定视图的frame属性（origin和size。

只有在layoutSubviews中，这些视图的frame才真正被计算出来，这个时候可以拿到他们的frame。

## ***`ARC` 在运行时做了哪些工作？

* 自动插入retain，release，autorelease
* 自动维护了weak指针

## 如何扩大一个button的响应范围

### 1. 创建 `EnlargedHitTestButton` 类

创建 `EnlargedHitTestButton.h` 和 `EnlargedHitTestButton.m` 文件。

### 2. 在 `.h` 文件中声明属性

在子类的头文件 (`EnlargedHitTestButton.h`) 中直接声明一个属性来存储边距，无需使用 Runtime：

Objective-C

```objc
z#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface EnlargedHitTestButton : UIButton

// 直接声明属性来存储您希望扩大的边缘值
// 注意：负值表示向外扩大区域
@property (nonatomic, assign) UIEdgeInsets hitTestEdgeInsets;

@end

NS_ASSUME_NONNULL_END
```

### 3. 在 `.m` 文件中实现核心逻辑Z

```objc
#import "EnlargedHitTestButton.h"

@implementation EnlargedHitTestButton

// 核心方法：判断点击点是否在响应区域内
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    
    UIEdgeInsets insets = self.hitTestEdgeInsets;
    
    // 如果没有设置扩展（即所有值都为 0），则直接调用父类判断
    if (UIEdgeInsetsEqualToEdgeInsets(insets, UIEdgeInsetsZero)) {
        return [super pointInside:point withEvent:event];
    }
    
    // 计算新的响应区域 (CGRect)
    // UIEdgeInsetsInsetRect 会根据负的 insets 值，把原始 bounds 扩大
    // 注意：insets 应该是负值（例如：{-10, -10, -10, -10}）
    CGRect enlargedBounds = UIEdgeInsetsInsetRect(self.bounds, insets);
    
    // 判断点击点是否在新区域内
    return CGRectContainsPoint(enlargedBounds, point);
}

@end
```

### 4. 使用方法

在你的 `ViewController.m` 中，使用这个自定义的子类：

```objc
#import "EnlargedHitTestButton.h"

// ...

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 实例化自定义按钮
    EnlargedHitTestButton *myButton = [[EnlargedHitTestButton alloc] initWithFrame:CGRectMake(100, 100, 44, 44)];
    [myButton setTitle:@"Click" forState:UIControlStateNormal];
    [myButton setTitleColor:[UIColor systemBlueColor] forState:UIControlStateNormal];
    [self.view addSubview:myButton];

    // 只需要设置这个属性即可扩大响应区域
    myButton.hitTestEdgeInsets = UIEdgeInsetsMake(-20, -20, -20, -20);
    // 实际点击区域将是 84x84 (44 + 20 + 20)
}
```

## SDWebImage这个库的详细介绍

### 核心功能 / 为什么它如此受欢迎？

1.  **异步图片下载器**：它在后台下载图片，因此你的 App UI 能保持流畅和响应迅速。
2.  **先进的二级缓存**：正如我们之前讨论的，它拥有一个高速的**内存缓存**和一个持久化的**磁盘缓存**。这结合了两者的优点：最近看过的图片可以即时加载，而之前看过的图片即使在离线状态下也能访问。
3.  **`UIImageView` 分类/扩展**：它为 `UIImageView` 添加了极其易用的方法（例如在 Swift 中的 `sd_setImage(with: URL, ...)`）。你只需一行代码就能处理好下载、缓存和显示图片的所有逻辑。
4.  **自动化的缓存管理**：它会自动处理缓存的验证和清理（基于文件的存储时间和总大小），你无需担心缓存会无限增长，耗尽用户的磁盘空间。
9.  **预加载（Prefetching）**：就像在你之前的代码示例中看到的那样，它提供了一个预加载器，可以提前下载图片，这样当用户滚动到它们时，图片早已在缓存中待命了。

### 它的工作原理：内部工作流

当你调用 `imageView.sd_setImage(with: someURL)` 时，一个复杂而精密的流程就开始了：

1.  **设置占位图（可选）**：如果你提供了一张占位图，它会立刻被显示在 `UIImageView` 上。
2.  **检查内存缓存**：`SDWebImageManager`（库的中央协调器）会首先使用图片的 URL 作为键（Key）来检查内存缓存中是否存在这张图片。
    *   **如果找到**：立即从内存中取出图片并显示。流程结束，这是最快的情况。
3.  **检查磁盘缓存**：如果内存中没有，它会将检查操作转移到一个后台线程，并在磁盘缓存中寻找图片。
    *   **如果找到**：从磁盘加载图片数据，将其解码成一个 `UIImage` 对象，然后为了未来的快速访问，将这个对象存入**内存缓存**。最后，在主线程上显示图片。
4.  **启动网络下载**：如果两个缓存中都没有，`SDWebImageDownloader` 才会启动一个网络任务，在后台线程从指定的 URL 下载图片。
5.  **处理下载数据**：一旦下载完成：
    *   原始数据会被解码成一个 `UIImage` 对象（这个过程也在后台完成）。
    *   如果你指定了任何图片转换操作，会在这里应用。
6.  **存入缓存**：最终处理好的图片会被**同时存入内存缓存和磁盘缓存**。
7.  **显示图片**：最后，操作切回主线程，`UIImageView` 会被更新为最终的图片，通常还会伴有一个优雅的淡入动画。

在整个过程中，它为每个 `UIImageView` 维护了一个活动操作的字典。如果你在同一个视图上再次调用 `sd_setImage`（例如，因为 Cell 被复用显示新内容），它会先取消之前的操作，然后再开始一个新的操作。

**Objective-C:**

```objc
#import <SDWebImage/SDWebImage.h>

// ... 在视图控制器或 Cell 内部

UIImageView *imageView = [[UIImageView alloc] init];
NSURL *imageURL = [NSURL URLWithString:@"http://www.example.com/image.jpg"];

// 基本用法
[imageView sd_setImageWithURL:imageURL];

// 高级用法：带占位图、选项和完成回调
[imageView sd_setImageWithURL:imageURL
             placeholderImage:[UIImage imageNamed:@"placeholder"]
                      options:SDWebImageRetryFailed | SDWebImageProgressiveLoad
                    completed:^(UIImage * _Nullable image, NSError * _Nullable error, SDImageCacheType cacheType, NSURL * _Nullable imageURL) {
    if (error) {
        NSLog(@"图片下载失败: %@", error);
    } else {
        NSLog(@"图片加载成功");
    }
}];
```

### 核心入口：`UIImageView+WebCache` (最常用的API)

`SDWebImage`通过Category（分类）的方式，为`UIImageView`、`UIButton`等UIKit控件添加了简单易用的网络图片加载方法。这是你90%的时间都会用到的API。

#### 最基础的方法

```objectivec
#import <SDWebImage/UIImageView+WebCache.h>

// 假设有一个UIImageView实例
UIImageView *imageView = [[UIImageView alloc] init];

// 和一个图片URL
NSURL *imageURL = [NSURL URLWithString:@"https://example.com/image.jpg"];

// --- 开始加载 ---
[imageView sd_setImageWithURL:imageURL];
```
**作用**: 这是最简单的方法。它会异步地从给定的URL下载图片，下载过程中`imageView`显示空白。下载完成后，图片会自动显示出来，并被缓存到内存和磁盘中。

#### 带占位图 (Placeholder)

```objectivec
// 提供一张本地的占位图片
UIImage *placeholderImage = [UIImage imageNamed:@"placeholder"];

[imageView sd_setImageWithURL:imageURL 
             placeholderImage:placeholderImage];
```
**作用**: 在网络图片开始下载或下载失败时，`imageView`会显示你提供的`placeholderImage`。这是改善用户体验最基本的方式，避免了UI上的空白。

#### 带选项和进度回调 (Options & Progress)

```objectivec
// SDWebImageOptions 是一个位掩码枚举，可以组合使用
SDWebImageOptions options = SDWebImageRetryFailed | SDWebImageRefreshCached;

[imageView sd_setImageWithURL:imageURL
             placeholderImage:placeholderImage
                      options:options
                     progress:^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
                         // 下载进度回调
                         float progress = (float)receivedSize / expectedSize;
                         NSLog(@"下载进度: %.2f%%", progress * 100);
                     }
                    completed:^(UIImage * _Nullable image, NSError * _Nullable error, SDImageCacheType cacheType, NSURL * _Nullable imageURL) {
                        // 下载完成回调
                        if (error) {
                            NSLog(@"图片加载失败: %@", error);
                            return;
                        }
                        
                        if (image) {
                            // 图片加载成功
                            NSLog(@"图片加载成功！");
                            
                            // cacheType 会告诉你图片来自哪里:
                            // SDImageCacheTypeNone: 从网络下载
                            // SDImageCacheTypeDisk: 从磁盘缓存
                            // SDImageCacheTypeMemory: 从内存缓存
                            
                            // 在这里可以做一些额外的处理，比如开启动画等
                            imageView.alpha = 0;
                            [UIView animateWithDuration:0.3 animations:^{
                                imageView.alpha = 1.0;
                            }];
                        }
                    }];
```
**这是功能最全面的方法，我们来逐一解析它的参数：**

*   **`options: (SDWebImageOptions)`**: 控制下载和缓存行为的关键。
    *   `SDWebImageRetryFailed`: 默认情况下，如果一个URL下载失败，它会被加入黑名单，短时间内不再尝试。使用此选项可以无视黑名单，重新尝试下载。
    *   `SDWebImageLowPriority`: 使用低优先级的URL连接，当用户正在滚动列表时非常有用，可以避免阻塞UI。
    *   `SDWebImageRefreshCached`: 即使图片已经被缓存，也强制从网络刷新。服务器需要配合`ETag`或`Last-Modified`头，如果图片未改变，服务器会返回304，SDWebImage会从缓存中加载，但`completed` block依然会被调用。
    *   `SDWebImageAvoidAutoSetImage`: 下载完成后，不要自动把图片设置到`imageView`上。这给了你在`completed` block中自定义显示逻辑的机会（比如先处理再显示）。
    *   `SDWebImageScaleDownLargeImages`: 对大图进行缩放，可以有效减少内存占用。推荐默认开启。
*   **`progress: (SDImageLoaderProgressBlock)`**: 下载进度的回调。它在**后台线程**被调用，如果你要用它来更新UI（如进度条），必须切回主线程。

---

### `SDWebImagePrefetcher` (图片预取器)

#### 1. 核心目标
`SDWebImagePrefetcher` 的设计目标非常明确：**提前、批量地、悄无声息地将一组图片下载并缓存起来，以便在用户真正看到它们时，能够瞬间从缓存中加载，从而极大地提升用户体验。**

它解决了这样一个痛点：当用户快速滚动一个充满图片的列表时，如果每个将要出现的图片都临时去发起网络请求，会导致大量的延迟和占位图闪烁。而预取器则将这个加载过程“前置”了。

#### 3. 技术工作原理

`SDWebImagePrefetcher` 内部做了一系列非常智能的管理工作：

*   **URL管理**: 它接收一个URL数组作为预取任务。
*   **去重与过滤**: 它会自动检查每个URL对应的图片是否已经在缓存中（包括内存和磁盘）。如果已存在，它会直接跳过，避免重复下载。
*   **并发控制**: 它不会一次性对所有URL发起请求，而是维护一个并发下载队列（可以通过`maxConcurrentDownloads`属性设置），比如一次最多同时下载3-5张图片，避免瞬间网络拥塞。
*   **生命周期管理**: 你可以随时取消一个预取任务。

#### 4. 核心API与使用场景

`SDWebImagePrefetcher` 也是一个单例，使用起来非常简单。

**最核心的方法:**

```objectivec
#import <SDWebImage/SDWebImagePrefetcher.h>

// 假设我们从服务器获取到了一个数据模型数组，每个模型里都有一个图片URL
NSArray<MyDataModel *> *dataModels = ...; 

// 1. 提取出所有需要预加载的图片URL
NSMutableArray<NSURL *> *urlsToPrefetch = [NSMutableArray array];
for (MyDataModel *model in dataModels) {
    if (model.imageURLString) {
        [urlsToPrefetch addObject:[NSURL URLWithString:model.imageURLString]];
    }
}

// 2. 获取预取器单例，并开始预取任务
[[SDWebImagePrefetcher sharedImagePrefetcher] prefetchURLs:urlsToPrefetch];
```
这就完成了最基本的预加载！图片会被下载并存入磁盘缓存。

**带进度和完成回调的完整方法:**

```objectivec
[[SDWebImagePrefetcher sharedImagePrefetcher] prefetchURLs:urlsToPrefetch
                                                  progress:^(NSUInteger noOfFinishedUrls, NSUInteger noOfTotalUrls) {
                                                      // 预取进度回调
                                                      // 比如，你可以在后台任务中记录进度
                                                      NSLog(@"预取进度: %lu / %lu", (unsigned long)noOfFinishedUrls, (unsigned long)noOfTotalUrls);
                                                  }
                                                 completed:^(NSUInteger noOfFinishedUrls, NSUInteger noOfSkippedUrls) {
                                                     // 预取完成回调
                                                     // noOfFinishedUrls: 成功下载的图片数量
                                                     // noOfSkippedUrls: 因为已在缓存中而跳过的图片数量
                                                     NSLog(@"预取完成！下载了 %lu 张图片, 跳过了 %lu 张图片。", (unsigned long)noOfFinishedUrls, (unsigned long)noOfSkippedUrls);
                                                 }];
```

#### 5. 最佳实践：结合`UITableView` / `UICollectionView`

`SDWebImagePrefetcher` 与列表的结合是其最经典的用法。

**时机**:
通常在**获取到列表数据源之后，`reloadData`之前**，是发起预取任务的最佳时机。

```objectivec
- (void)fetchDataFromServer {
    [self.apiClient fetchDataWithCompletion:^(NSArray<MyDataModel *> *newData) {
        
        // --- 最佳实践点 ---
        // 1. 更新数据源
        self.dataModels = newData;
        
        // 2. 提取URL并启动预取任务
        NSMutableArray<NSURL *> *urls = [NSMutableArray array];
        for (MyDataModel *model in self.dataModels) {
            [urls addObject:[NSURL URLWithString:model.imageURLString]];
        }
        [[SDWebImagePrefetcher sharedImagePrefetcher] prefetchURLs:urls];
        
        // 3. 刷新UI
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.tableView reloadData];
        });
    }];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    MyCell *cell = [tableView dequeueReusableCellWithIdentifier:@"MyCell" forIndexPath:indexPath];
    MyDataModel *model = self.dataModels[indexPath.row];
    NSURL *imageURL = [NSURL URLWithString:model.imageURLString];
    
    // 此时调用sd_setImage，图片很大概率已经被Prefetcher下载到缓存中了
    // cacheType 会是 SDImageCacheTypeDisk 或 SDImageCacheTypeMemory
    // 加载会非常快，几乎是瞬时完成
    [cell.customImageView sd_setImageWithURL:imageURL placeholderImage:[UIImage imageNamed:@"placeholder"]];
    
    return cell;
}
```

## 想统计所有类中哪些方法耗时最高 怎么统计 oc

好的，这是一个非常典型的iOS性能监控和APM（Application Performance Management）领域的问题。要统计所有OC类中方法的耗时，并且找出最耗时的方法，我们不能手动去给每个方法加代码，那样不现实。

正确的方法是利用Objective-C语言强大的**动态特性**，特别是**方法交换 (Method Swizzling)** 和 **消息转发 (Message Forwarding)**。

核心思想是：**在运行时，用我们自己写的“监控”方法，去替换掉所有我们想监控的原始方法的实现。这样，当原始方法被调用时，会先执行我们的监控代码（记录时间），然后我们再手动调用原始的实现，等原始实现执行完毕后，再次记录时间，计算出差值，就得到了方法的耗时。**

下面我将详细介绍几种实现方案，从简单概念到工业级实现。

---

### **方案一：基于 Method Swizzling 的简单实现（核心概念）**

这是最基础、最核心的技术，面试时必须掌握。它利用 `method_exchangeImplementations` 函数来交换两个方法的IMP（Implementation，指向函数实现的指针）。

**核心步骤：**

1.  **创建一个分类 (Category):** 为 `NSObject` 创建一个分类，因为我们想监控所有继承自 `NSObject` 的类。

2.  **定义一个“钩子”方法:** 在分类中定义一个我们自己的方法，比如 `apm_hooked_method`。这个方法的签名（参数、返回值）必须与我们想要hook的方法完全一样，但这在监控所有方法时很难做到。所以，一个更通用的方法是hook `forwardInvocation:`。但我们先从简单的开始，假设我们只hook无参方法。

3.  **在 `+load` 方法中执行交换:** `+load` 方法在类被加载到内存时会自动调用，是执行方法交换最安全、最合适的时机。

**示例代码 (仅用于演示hook一个特定方法，比如 `viewDidLoad`):**

```objectivec
// UIViewController+APM.m
#import <objc/runtime.h>

@implementation UIViewController (APM)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 获取原始方法
        SEL originalSelector = @selector(viewDidLoad);
        Method originalMethod = class_getInstanceMethod(self, originalSelector);

        // 获取我们自己的钩子方法
        SEL swizzledSelector = @selector(apm_viewDidLoad);
        Method swizzledMethod = class_getInstanceMethod(self, swizzledSelector);

        // 交换两个方法的实现
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}

- (void)apm_viewDidLoad {
    // 1. 记录开始时间
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();

    // 2. 调用原始实现
    // 因为方法实现已经被交换了，所以这里调用 apm_viewDidLoad 实际上是在调用原始的 viewDidLoad
    [self apm_viewDidLoad]; 

    // 3. 记录结束时间并计算耗时
    CFAbsoluteTime endTime = CFAbsoluteTimeGetCurrent();
    NSTimeInterval cost = (endTime - startTime) * 1000.0; // 转换为毫秒

    // 4. 上报或打印耗时
    // 注意：这里不能打印 self，可能会导致死循环。打印 className 和方法名。
    NSLog(@"-[%@ %@] cost: %.2f ms", NSStringFromClass([self class]), @"viewDidLoad", cost);
}

@end
```
**这种方法的局限性：**
*   **无法通用：** 你不可能为成千上万个不同的方法签名都写一个对应的 `apm_` 方法。
*   **粒度太粗：** `+load` 里只能交换当前类的方法，无法遍历所有类。

---

### **方案二：基于消息转发 (Message Forwarding) 的通用方案（工业级思路）**

要实现**“统计所有类中所有方法”**这个宏伟目标，我们需要更强大的武器——**消息转发机制**。

当一个对象收到一个它无法响应的消息时（即找不到对应的方法实现），OC的运行时会启动一套消息转发流程，给我们三次机会来“拯救”这次调用。我们可以在这个流程中做手脚。

**核心思想：**
我们故意用一个**统一的、不存在的函数指针 `_objc_msgForward`** 去替换掉所有想监控的方法的IMP。这样，当这些方法被调用时，就会因为找不到实现而触发消息转发。然后我们再实现 `forwardInvocation:` 方法，在这个方法里“接住”所有调用，进行耗时统计。

**实现步骤：**

1.  **遍历所有类和方法:**
    *   在合适的时机（比如 `main` 函数执行前，或通过 `+load`），我们需要获取运行时中**所有已注册的类**。
    *   对于每一个类，遍历它的**方法列表** (`method list`)。

2.  **替换IMP为 `_objc_msgForward`:**
    *   对于遍历到的每一个方法，我们保存它原始的IMP。
    *   然后，使用 `method_setImplementation` 将它的IMP，替换为 `_objc_msgForward` 的IMP。这个特殊的IMP是OC消息转发机制的入口点。
    *   **注意：** 对于返回结构体的方法，需要使用 `_objc_msgForward_stret`。

3.  **实现 `forwardInvocation:`:**
    *   我们通过方法交换，将所有类的 `forwardInvocation:` 方法都替换为我们自己的 `apm_forwardInvocation:`。
    *   在 `apm_forwardInvocation:` 中，我们就能拿到一个 `NSInvocation` 对象，这个对象包含了这次方法调用的**所有信息**：目标对象 (`target`)、方法选择器 (`selector`)、所有参数。

**伪代码逻辑：**

```objectivec
// 在一个集中的APM管理类中

- (void)startMonitoring {
    // 1. 获取所有类
    int numClasses;
    Class *classes = objc_copyClassList(&numClasses);

    for (int i = 0; i < numClasses; i++) {
        Class currentClass = classes[i];

        // 过滤掉我们不关心的类（比如系统私有类）
        if (![self shouldMonitorClass:currentClass]) continue;

        // 2. 遍历这个类的所有方法
        unsigned int numMethods;
        Method *methods = class_copyMethodList(currentClass, &numMethods);

        for (int j = 0; j < numMethods; j++) {
            Method method = methods[j];
            SEL selector = method_getName(method);

            // 过滤掉我们不关心的系统方法（如 getter/setter）
            if (![self shouldMonitorSelector:selector]) continue;

            // 3. 保存原始IMP，并替换为 _objc_msgForward
            IMP originalIMP = method_getImplementation(method);
            // 将 (Class, SEL) -> IMP 的映射关系保存到一个全局字典里
            [self saveOriginalIMP:originalIMP forClass:currentClass selector:selector];

            // 获取 _objc_msgForward 的实现
            IMP msgForwardIMP = _objc_msgForward; 
            
            // 替换！
            class_replaceMethod(currentClass, selector, msgForwardIMP, method_getTypeEncoding(method));
        }
        free(methods);
        
        // 4. 交换 forwardInvocation:
        [self swizzleForwardInvocationForClass:currentClass];
    }
    free(classes);
}

// 替换后的 forwardInvocation: 实现
- (void)apm_forwardInvocation:(NSInvocation *)invocation {
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();

    // 从我们保存的全局字典里，找到这个方法原始的 IMP
    IMP originalIMP = [self getOriginalIMPForClass:[self class] selector:invocation.selector];
    
    if (originalIMP) {
        // 将 invocation 的 selector 临时改成一个别名，防止死循环
        SEL aliasSelector = [self aliasSelectorForSelector:invocation.selector];
        // 确保类能响应这个别名，并将实现指向原始 IMP
        class_addMethod([self class], aliasSelector, originalIMP, ...);
        invocation.selector = aliasSelector;
        
        // 调用原始实现
        [invocation invoke];
    }

    CFAbsoluteTime endTime = CFAbsoluteTimeGetCurrent();
    NSTimeInterval cost = (endTime - startTime) * 1000.0;
    
    // 上报耗时和方法信息
    if (cost > THRESHOLD) { // 只上报超过阈值的耗时
        [self reportCost:cost forClass:[self class] selector:invocation.selector];
    }
}
```

**这种方案的优点：**
*   **非常通用：** 可以监控任何类、任何方法的调用，无需关心其参数和返回值类型。
*   **信息全面：** `NSInvocation` 提供了关于方法调用的所有元信息。
*   **集中处理：** 所有的耗时统计逻辑都集中在 `apm_forwardInvocation:` 中，易于管理。

**缺点：**
*   **性能开销大：** 消息转发本身的开销就比直接的方法调用要大。对所有方法都进行hook，会显著影响App的整体性能，尤其是在频繁调用的方法上。
*   **实现复杂：** 需要处理很多边界情况，比如结构体返回值、与系统其他hook的冲突、多线程安全等。

## 聊聊property

一种语法糖

**`@property` 本质上是一个编译器指令，它会自动为你生成一个用于存储数据的私有实例变量（`ivar`），以及一套标准的、遵循你指定内存管理规则（如`strong`或`copy`）的 `getter` 和 `setter` 方法，从而让你能以“点语法”（`object.property`）安全地访问这个变量。**

## atomic是如何实现原子性的

### 1. 自动加锁的存取方法

当你将一个属性声明为 `atomic`（这也是默认行为），编译器会自动合成对应的getter和setter方法。在这些自动生成的方法内部，包含了一个锁，以保护对底层实例变量的访问。

*   **Setter方法**: 当一个线程调用setter方法来修改属性值时，该方法会先获取一个锁。在持有锁的期间，它会完成整个赋值操作（对于对象，这可能包括释放旧值并保留新值）。操作完成后，释放锁。 任何其他尝试访问该属性的线程都必须等待，直到该锁被释放。

*   **Getter方法**: 同样，当一个线程调用getter方法读取属性值时，它也会先获取同一个锁。在持有锁的情况下读取值，然后释放锁。 这确保了在读取期间，没有其他线程可以修改该值，从而防止读取到不完整或损坏的数据。

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

    Person *personCopy = [[[self class] allocWith-Zone:zone] init];
    
    personCopy.name = self.name; 
    
    // c. 返回创建好的副本对象
    return personCopy;
}

@end
```

## ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些？

* 对象类型`@property (strong, atomic, readwrite) NSString *name;`

* 非对象类型`@property (assign, atomic, readwrite) NSInteger score`

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

## ？category能不能添加property

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

### 1. 什么是 RunLoop Mode？

一个 `RunLoop Mode` 是一个**`Input Sources` (输入源)** 和 **`Timers` (定时器)** 的集合。

可以把它想象成一个**“工作场景”**或**“任务清单”**。

*   一个 `RunLoop` 对象，可以包含多个不同的 `Mode`（多个不同的任务清单）。
*   但是，在**任何一个特定的时间点**，`RunLoop` 只能选择**其中一个 `Mode`** 来运行。
*   当 `RunLoop` 在某个 `Mode` 下运行时，它就只会处理和监听那些被**注册到这个 `Mode` 下**的 `Sources` 和 `Timers`，而会**完全忽略**其他 `Mode` 下的所有事件源

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

### ✅ **注意点**

1. **RunLoop 不会无缘无故保持运行**，必须有事件源（Port、Timer、Source）。
2. **GCD** 线程池里的线程不能这么保活，因为它们是系统管理的。
3. 子线程保活后，要注意避免内存泄漏或死循环，确保退出机制。

## runtime

### 面试官：“同学，讲一下你对 Runtime 的理解吧”

**面试官，您好。我对 Runtime 的理解是，它更像是一套处于 Objective-C 底层的、用 C 语言实现的 API 库。它的核心作用，就是把 C 语言这种静态语言的很多决策，比如调用哪个函数，都从“编译时”推迟到了“运行时”来做，这就赋予了 Objective-C 极致的动态性。**

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

---

#### 第三步：举出实际应用【我能用来做什么？】

**正是因为 Runtime 的这套动态机制，我们才能在实际开发中实现很多强大的功能。我举几个最典型的例子：**

1.  **方法交换 (Method Swizzling)**：我们可以在运行时动态地交换两个方法的实现。最经典的应用就是做“埋点”，比如在不修改原有代码的情况下，我们可以 hook 掉所有 `UIViewController` 的 `viewWillAppear:` 方法，在里面加入我们自己的统计代码，实现对所有页面展示的全局监控。这是一种 AOP（面向切面编程）思想的体现。
2.  **动态添加属性 (Associated Objects)**：我们都知道 Category 不能直接添加实例变量，但借助 Runtime 的关联对象技术，我们就可以在运行时为一个已有的类“附加”上新的属性，极大地扩展了 Category 的能力。

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

### 2. 能做什么（常见功能）

- **消息机制**：调用方法实际上是给对象发送消息。
- **动态特性**：
  - 动态添加方法（`class_addMethod`）。
  - 方法交换（Method Swizzling）。
  - 动态解析未实现的方法。
- **KVC/KVO**：底层依赖 runtime 实现。

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

**KVC 原理**

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

好的，这是一个非常深入且核心的 Objective-C 问题。我们来详细拆解一下 Objective-C 对象、它的类（Class）以及它们在内存中的分布。

为了更好地理解，我们把它分为三个部分：
1.  **实例对象 (Instance) 在内存中是什么样的？** (我们平时 `alloc init` 出来的东西)
2.  **类对象 (Class) 里面有什么？** (实例对象的 `isa` 指向的东西)
3.  **元类对象 (Metaclass) 是什么？** (类对象的 `isa` 指向的东西)

---

### Part 1: 实例对象 (Instance Object) 的内存布局

当我们写下如下代码时：

```objc
// 定义一个 Person 类
@interface Person : NSObject
{
    NSString *_name; // 实例变量
    int _age;        // 实例变量
}
@end

// 创建一个实例
Person *person = [[Person alloc] init];
```

这个 `person` 对象在内存中的结构非常简单，它基本上就是：

1.  **一个 `isa` 指针**
2.  **所有成员变量（的值**

### Part 2: 类对象  的内部结构

`isa` 指针指向的是一个**类对象**。在 Objective-C 中，**类本身也是一个对象**。`Person` 这个类，在内存中也存在一个唯一的、类型为 `Class` 的对象。

这个类对象存储了描述一个类的所有“元信息”，也就是“蓝图”信息。它主要包含以下内容：

1.  **`isa` 指针**: 是的，类对象自己也有一个 `isa` 指针。它指向这个类的**元类（Metaclass）**。
2.  **`superclass` 指针**: 指向其父类的**类对象**。例如，`Person` 的 `superclass` 指针就指向 `NSObject` 的类对象。这个指针构成了继承链。
3.  **方法缓存 **: 一个为了性能优化的缓存。当一个方法被调用后，其指针和选择器（selector）会被缓存到这里，下次再调用同一个方法时，运行时可以直接从缓存中查找，速度极快。
4.  **类信息 **: 这是一个包含了几乎所有核心信息的复杂结构。通过它可以访问到一个 `class_ro_t` (Read-Only) 的结构体，里面存储了类的“只读”信息，包括：
    *   **实例变量列表 (`ivar_list_t`)**: 描述了这个类的所有成员变量的名称、类型、内存偏移量等。
    *   **实例方法列表 (`method_list_t`)**: **这是最重要的部分**，它存储了这个类的所有**实例方法**（减号 `-` 方法）的实现（IMP）、选择器（SEL）和类型编码。
    *   **属性列表 (`property_list_t`)**: `@property` 声明的属性信息。
    *   **协议列表 (`protocol_list_t`)**: 这个类遵守的所有协议。



### Part 3: 元类对象 (Metaclass Object)

我们已经知道，类对象也有一个 `isa` 指针，那它指向哪里呢？答案是**元类对象 (Metaclass)**。

**为什么需要元类？**

Objective-C 的设计哲学是“一切皆对象”，消息发送是其核心机制。我们调用实例方法是向实例对象发送消息，例如 `[person instanceMethod]`。

那么，当我们调用一个**类方法**（加号 `+` 方法）时，例如 `[Person aClassMethod]`，消息是发送给谁的呢？答案是发送给**类对象**。

既然类对象也要能响应消息（调用方法），那么它也必须符合对象的定义，即它也需要有一个“类”来描述它自己，这个“类”就是**元类**。

**元类的核心作用**：
*   **存储类方法**：元类对象的结构和类对象非常相似，但它的方法列表里存储的是**类方法**（`+` 方法）的定义。

所以，整个关系链是：

1.  **实例对象**的 `isa` 指向其**类对象**。
2.  **类对象**的 `isa` 指向其**元类对象**。
3.  **元类对象**的 `isa` 指向根元类（通常是 `NSObject` 的元类）。
4.  根元类的 `isa` 最终指向它自己，形成一个闭环。

*   **`isa` 链 (纵向)**: `实例` -> `类` -> `元类` -> `根元类`。这条链用于**消息发送**时查找方法的实现。调用实例方法时，从“类”开始找；调用类方法时，从“元类”开始找。
*   **`superclass` 链 (横向)**: `子类` -> `父类` -> `NSObject` -> `nil`。这条链用于**继承**。当在本类的方法列表中找不到方法时，会沿着这条链向上查找。

## oc内存区域

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

## gcd的死锁场景

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

## GCD和NSOperation的区别？

### 一、抽象层次与核心概念

| 特性         | GCD (Grand Central Dispatch)                                 | NSOperation / NSOperationQueue                               |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **核心概念** | **任务 (Task)** 和 **队列 (Queue)**                          | **操作 (Operation)** 和 **队列 (Queue)**                     |
| **抽象层次** | **更底层，更轻量**。它是基于 C 的 API，直接与系统内核的线程调度交互。你关注的是“把这个 Block 扔到这个队列里执行”。 | **更高层，面向对象**。它是对 GCD 的一层封装，提供了更多高级功能。你关注的是“创建一个操作对象，配置它，然后添加到队列里”。 |
| **本质**     | **First-In-First-Out (FIFO) 队列**。任务一旦提交，就按顺序等待调度。 | **更灵活的队列**。虽然默认也是 FIFO，但可以通过设置**优先级**和**依赖关系**来改变实际的执行顺序。 |

---

### 二、核心功能与特性对比

| 功能点              | GCD                                                          | NSOperation / NSOperationQueue                               |
| :------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **1. 依赖关系**     | **不支持**。你无法简单地声明“任务 C 必须在任务 A 和 B 完成后执行”。虽然可以通过 `dispatch_group` 或 `dispatch_barrier` 间接模拟，但比较繁琐。 | **原生支持，非常强大**。这是 `NSOperation` 的**杀手级特性**。<br> `[operationC addDependency:operationA];`<br> `[operationC addDependency:operationB];`<br> 这样可以轻松构建复杂的任务执行图。 |
| **2. 状态控制**     | **不支持**。任务一旦提交到队列，你就无法中途控制它。它会一直执行直到完成。你不能暂停、恢复或取消一个已经提交的 Block。 | **原生支持**。`NSOperation` 是一个状态机，有 `isReady`, `isExecuting`, `isFinished`, `isCancelled` 等状态。<br> - `[operation cancel]`：可以取消一个**尚未开始**或**正在执行中**的操作（需要你在操作内部检查 `isCancelled` 状态）。<br> - `[queue setSuspended:YES]`：可以**暂停**整个队列，所有未开始的任务都会被挂起。<br> - `[queue setSuspended:NO]`：可以**恢复**队列。 |
| **3. 最大并发数**   | **不支持直接设置**。并发队列的并发数由系统 GCD 动态管理。虽然可以通过 `DispatchSemaphore` (信号量) 来间接控制并发数量，但这是一种限制手段，而非队列属性。 | **原生支持**。`NSOperationQueue` 有一个 `maxConcurrentOperationCount` 属性，可以非常方便地控制同时执行的操作数量。<br> - `queue.maxConcurrentOperationCount = 1;` // 变成了**串行队列**。<br> - `queue.maxConcurrentOperationCount = 5;` // 最多同时执行 5 个操作。 |
| **4. 优先级 (KVO)** | **支持**。通过服务质量 (QoS) 来设置。`DISPATCH_QUEUE_PRIORITY_HIGH`, `QOS_CLASS_USER_INITIATED` 等。 | **原生支持**。`NSOperation` 有 `queuePriority` 属性 (`NSOperationQueuePriorityVeryHigh` 等)。更重要的是，你可以通过 **KVO (键值观察)** 来监听 `NSOperation` 的状态变化（如 `isFinished`），这在需要更新 UI 时非常有用。 |
| **5. 性能**         | **更轻量，性能更高**。由于是更底层的 C API，没有额外的面向对象开销，对于简单的、一次性的后台任务，GCD 是最快的。 | **有面向对象的开销**。创建 `NSOperation` 对象比创建一个 Block 要消耗更多的内存和时间。但对于需要复杂控制的场景，这点开销完全可以忽略不计。 |

---

### 三、使用场景与选择

**什么时候应该使用 GCD？**

1.  **简单的后台任务**：当你只需要“把这个任务扔到后台执行，执行完通知我”时，GCD 是最简单、最高效的选择。
    ```objc
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
        // Do heavy work...
        dispatch_async(dispatch_get_main_queue(), ^{
            // Update UI
        });
    });
    ```
2.  **对性能要求极高的场景**：当你需要管理成千上万个小任务时，GCD 的轻量级特性更有优势。
3.  **需要使用 GCD 特有功能时**：如 `dispatch_once` (单例模式), `dispatch_group` (等待一组任务完成), `dispatch_barrier` (栅栏函数，用于线程安全的读写分离), `dispatch_source` (监听系统事件)。

**什么时候应该使用 NSOperation / NSOperationQueue？**

1.  **需要复杂的任务依赖关系时**：这是**首选** `NSOperation` 的最重要理由。比如，一个多步骤流程：1. 下载数据 -> 2. 解析数据 -> 3. 更新数据库。这三步可以被完美地建模为三个有依赖关系的 `NSOperation`。
2.  **需要随时取消、暂停、恢复任务时**：比如一个图片下载队列，当用户离开当前页面时，你需要取消所有未完成的下载任务。
3.  **需要控制并发数量时**：比如，创建一个最大并发数为 3 的网络请求队列，以避免同时对服务器发起过多请求。
4.  **需要重用和封装任务逻辑时**：你可以创建 `NSOperation` 的子类，将复杂的任务逻辑（包括状态管理）封装在内部，使其成为一个可重用的组件。

## 通知中心的实现原理

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
                        userInfo:userInfo]; //一个字典
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

### SEL 的本质：一个 C 字符串（方法名）

正如上一个回答中提到的，`SEL` 的本质是一个在编译时被注册的、唯一的 C 字符串，它代表了一个方法的名字。

*   **类型**: `typedef struct objc_selector *SEL;`
*   **核心**: 它只是一个 **名字**。在整个程序中，所有同名的方法（例如，不同类中都有的 `viewDidLoad` 方法）都对应着同一个 `SEL`。编译器会维护一个全局的方法名表，`SEL` 就是指向这个表中某个名字的指针。

---

### IMP 的本质：一个函数指针

这是问题的关键。**`IMP` 的本质是一个指向方法具体实现的函数指针 **。

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

## nsurlsession

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

类似java的arraylist，动态数组底层，满了就扩容两倍，创建一个新的两倍容量的数组 把内容拷贝过去

## block和delegate如何使用

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

### 🔹 懒汉式单例（第一次使用时才创建实例）

### 特点

- 第一次调用时才创建实例，按需分配。
- 常用 `dispatch_once` 来保证线程安全。

### 实现示例

```objective-c
@interface MyManager : NSObject
+ (instancetype)sharedInstance;
@end

@implementation MyManager

static MyManager *instance = nil;

+ (instancetype)sharedInstance {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

@end
```

🔸 优点：需要时才创建，节省资源。
 🔸 缺点：实现比饿汉式稍微复杂一点。

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

### 第二部分：核心概念与工作原理 (How it Works)

要理解GCD，必须先理解几个核心概念：

1.  **任务 (Task)**：指的是我们希望执行的操作。在Objective-C和Swift中，这通常是一个**Block**或者一个函数。任务分为两种执行方式：
    *   **同步 (Sync)**：使用 `dispatch_sync` 提交。这个函数会阻塞当前线程，直到提交的任务在指定队列上执行完毕后，才会返回。
    *   **异步 (Async)**：使用 `dispatch_async` 提交。这个函数会立即返回，不会阻塞当前线程，任务会在后台的某个时间点被执行。这是实现并发和UI不卡顿的关键。

2.  **队列 (Dispatch Queue)**：这是一个遵循**FIFO（先进先出）**原则的数据结构，用于存放待执行的任务。GCD中的队列主要分为两种类型：
    *   **串行队列 (Serial Queue)**：同一时间内，队列中只有一个任务在执行。任务会一个接一个地、按顺序执行。虽然只有一个任务在跑，但执行任务的线程可能是变化的（由GCD决定）。主队列就是一个特殊的全局串行队列。
    *   **并发队列 (Concurrent Queue)**：同一时间内，队列中可以有多个任务并发执行（只要系统有足够的计算资源）。任务的启动顺序仍然是FIFO，但它们的完成顺序是不可预测的。

3.  **系统提供的队列**：
    *   **主队列 (Main Queue)**：这是一个全局可用的**串行队列**，所有提交到主队列的任务都会在应用程序的**主线程**上执行。因此，所有UI更新相关的操作都必须放在主队列中。可以通过 `DispatchQueue.main` (Swift) 或 `dispatch_get_main_queue()` (OC) 获取。
    *   **全局并发队列 (Global Concurrent Queues)**：这是系统为我们提供的并发队列，方便我们执行后台耗时操作。它们根据**服务质量（QoS）**进行了优先级划分，以告诉系统任务的重要程度，从而合理分配CPU、I/O等资源。常见的QoS级别有：
        *   `userInteractive`: 用户交互相关，需要立即完成，优先级最高。
        *   `userInitiated`: 用户主动发起的任务，需要快速响应。
        *   `default`: 默认级别。
        *   `utility`: 需要一些时间，用户不急切等待结果。
        *   `background`: 后台维护性任务，对时间不敏感，优先级最低。
        可以通过 `DispatchQueue.global(qos:)` 或 `dispatch_get_global_queue()` 获取。



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

### 特殊情况：Tagged Pointer

对于一些小对象，比如特定数值的 `NSNumber`、短的 `NSString` 等，苹果使用了 **Tagged Pointer** 技术。

*   **原理：** 不在堆上分配内存，而是将对象的数据**直接编码存储在指针自身的值里**。
*   **判断方法：** 对于 64 位架构，如果一个指针的最低有效位是 1 (macOS) 或最高有效位是 1 (iOS)，那么它就是一个 Tagged Pointer。
*   **引用计数：** **Tagged Pointer 对象没有引用计数**。它们不存储在堆上，其生命周期管理方式更像栈上的基本数据类型。对它进行 `retain` 和 `release` 操作实际上是空操作 (no-op)。当你把它赋值给另一个变量时，仅仅是指针值（也就是数据本身）的拷贝。

## 从32b到64b的指针，苹果做了什么

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
};**
```

*   **极速的引用计数：**
    *   **快速路径：** 对于绝大多数对象，其引用计数（`extra_rc`，19位足够表示50多万次引用）可以直接在 `isa` 内部通过原子操作进行增减。这比去查找并锁定一个全局的哈希表（Side Table）要快几个数量级。
    *   **慢速路径：** 只有当 `extra_rc` 溢出，或者对象有了弱引用时，才会启用 `has_sidetable_rc` 标志，并将完整的引用计数存入 Side Table。这保证了常见情况下的极致性能。
*   **对象状态信息聚合：** 无需额外查询，通过 `isa` 就能快速判断一个对象是否有弱引用、是否有关联对象等，加快了运行时的判断速度。

## nsstring生命周期的管理

### `NSString` 的三种主要类型及其生命周期

`NSString` 实际上是一个“类簇 (Class Cluster)”，它的背后有多种不同的私有子类来实现。我们可以将它们归为三大类：

#### 1. 编译期常量字符串 (`__NSCFConstantString`)

这是最常见也最特殊的一种。所有在代码中直接用 `@"..."` 语法创建的字符串字面量都属于这一类。

*   **创建方式**:
    ```objectivec
    NSString *str1 = @"Hello, World!";
    ```
*   **内存位置**: 这种字符串在**编译时**就已经被创建好了，并存储在应用程序二进制文件的**数据段 (`__TEXT` 或 `__DATA`)** 中。
*   **生命周期管理**:
    *   **它没有引用计数**，或者说它的引用计数是一个“无限大”的特殊值。
    *   它**与应用程序的生命周期相同**。只要 App 在运行，它就存在于内存中，App 退出时才会被回收。
    *   对它进行 `retain`、`release`、`copy` 操作都**不起任何作用**，它会直接返回自身。ARC 也不会对它进行任何内存管理操作。

*   **验证**:
    ```objectivec
    NSString *str = @"This is a constant string";
    NSLog(@"Class: %@", [str class]); // 输出: __NSCFConstantString
    NSLog(@"Retain count: %lu", (unsigned long)[str retainCount]); // 输出一个非常大的值，代表“无限”
    ```
    你不需要担心它的释放，它永远都在。

#### 2. 标记指针字符串 (`NSTaggedPointerString`)

这是苹果在 64 位环境下引入的一项重大性能优化。对于一些**非常短小、内容简单**（由 ASCII 字符组成）的字符串，系统不会在堆上为它分配内存。

*   **创建方式**: 通常在运行时动态创建，但内容很短。
    ```objectivec
    NSString *name = @"Tim";
    NSString *dynamicStr = [NSString stringWithFormat:@"Name: %@", name]; // "Name: Tim" 可能会是 Tagged Pointer
    ```
*   **内存位置**: **它没有独立的内存空间！** 字符串的数据直接存储在**指针变量自身**的 64 位空间里。指针的值不再是内存地址，而是一个包含了类型标记和数据的特殊编码。
*   **生命周期管理**:
    *   **它也不需要引用计数**。因为它不占用堆内存，所以也就不需要 ARC 来管理。
    *   它的生命周期类似于一个**栈上的局部变量**。当指针变量超出其作用域时，它所占用的栈空间被回收，数据也就消失了。
    *   创建和销毁的开销极小，几乎为零。

*   **验证**:
    ```objectivec
    NSString *shortStr = [NSString stringWithFormat:@"abc"];
    // 在调试器中打印 po shortStr，你会看到类似 0xa000000006362613 的值，这就是 Tagged Pointer
    NSLog(@"Class: %@", [shortStr class]); // 输出: NSTaggedPointerString
    ```
    这种字符串的性能极高，你同样不需要关心它的内存管理。

#### 3. 堆上分配的字符串 (`__NSCFString` 或其他)

当字符串是在运行时创建，并且内容较长或复杂，无法用 Tagged Pointer 表示时，系统就会在**堆 (Heap)** 上为它分配内存。

*   **创建方式**:
    *   从文件、网络数据创建。
    *   对长字符串进行拼接、格式化。
    *   `NSMutableString` 类型的对象。
    ```objectivec
    NSString *longStr = [NSString stringWithFormat:@"This is a very long string that cannot be a tagged pointer..."];
    NSString *fileContent = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
    NSMutableString *mutableStr = [NSMutableString stringWithString:@"Initial"];
    ```
*   **内存位置**: **堆 (Heap)**。
*   **生命周期管理**:
    *   **这是唯一一类真正由 ARC 管理的字符串**。
    *   它有正常的引用计数。当一个 `strong` 指针指向它时，引用计数加 1。当 `strong` 指针被置为 `nil` 或离开作用域时，引用计数减 1。
    *   当引用计数变为 0 时，对象被 `dealloc`，其占用的堆内存被系统回收。
    *   这就是我们通常理解的 Objective-C 对象的生命周期。

---

### 核心实践：为什么 `NSString` 属性要用 `copy`

这是 `NSString` 生命周期管理中最经典的一个实践问题。

**结论**：在声明 `NSString` 属性时，应始终使用 `copy` 关键字，而不是 `strong`。

```objectivec
@property (nonatomic, copy) NSString *name; // 正确的做法
@property (nonatomic, strong) NSString *wrongName; // 错误的做法
```

**原因**：为了防止将一个 `NSMutableString` 对象赋值给 `NSString` 属性后，外部的 `NSMutableString` 发生改变，导致 `NSString` 属性的值“意外”地被修改。

看下面的例子：

```objectivec
NSMutableString *mutableName = [NSMutableString stringWithString:@"Alice"];

// 假设 MyClass 的 name 属性是用 strong 声明的
MyClass *obj = [[MyClass alloc] init];
obj.wrongName = mutableName; // strong 只是增加引用计数，obj.wrongName 和 mutableName 指向同一个对象

NSLog(@"Initial name: %@", obj.wrongName); // 输出: Initial name: Alice

// 现在，我们在外部修改 mutableName
[mutableName appendString:@" Smith"];

// 看看 obj 的属性变成了什么
NSLog(@"Changed name: %@", obj.wrongName); // 输出: Changed name: Alice Smith
```
**问题出现了**：`obj.wrongName` 的值在我们不知情的情况下被改变了！这破坏了 `NSString` 的“不可变性”原则，可能导致非常隐蔽和难以调试的 Bug。

**`copy` 如何解决这个问题**：
当你使用 `copy` 关键字时，setter 方法的内部实现会是这样的：

```objectivec
- (void)setName:(NSString *)name {
    // _name = [name copy]; // 这是 copy 的核心
    
    // 如果 name 是 NSString, [name copy] 只是增加引用计数 (浅拷贝)，效率很高。
    // 如果 name 是 NSMutableString, [name copy] 会创建一个新的、不可变的 NSString 对象 (深拷贝)，
    // 并将新对象的地址赋值给 _name。
}
```

这样，无论你传进来的是 `NSString` 还是 `NSMutableString`，`name` 属性最终持有的都将是一个**不可变的、与外部对象无关**的副本，从而保证了数据的安全性和稳定性

## ***事件传递和响应链

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

5.  **最终**：如果事件一路传递到`UIApplication`都无人处理，那么这个事件就会被系统丢弃

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

## ***uiview怎么渲染到屏幕上

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

渲染服务器进程接收到数据后，接力棒就交给了 GPU。

### 第五步：显示在屏幕上 (Display)

1.  **帧缓冲区 (Framebuffer)**：GPU将合成好的最终图片，放入一块名为**帧缓冲区**的特殊内存区域。这块内存里的数据，直接对应着屏幕上即将显示的每一个像素的颜色。

2.  **双缓冲机制 (Double Buffering)**：为了防止用户看到正在绘制中的、不完整的画面（即“屏幕撕裂”），系统使用了双缓冲技术。
    *   屏幕当前正在显示的是**前台缓冲区（Front Buffer）**。
    *   GPU在后台默默地将新的一帧画面绘制到**后台缓冲区（Back Buffer）**。
    *   当GPU画完一整帧后，系统会等待一个时机。

3.  **垂直同步 (VSync)**：这个时机就是屏幕的**垂直同步信号**。你的iPhone屏幕以固定的频率刷新（比如60Hz或120Hz）。每当屏幕准备好刷新下一帧时，就会发出一个VSync信号。
    *   收到信号后，系统会**瞬间交换**前后台缓冲区的指针。后台缓冲区变成新的前台缓冲区，开始显示在屏幕上；而旧的前台缓冲区则变成后台缓冲区，等待GPU绘制下一帧。

## 离屏渲染是什么，怎么避免？

* 归根到底是因为画家算法，它要由近及远的去渲染每个图层，第一个图层是一个风景，第二图层是一个圆形的遮罩子，你渲染遮罩的时候竟然说我要把一开始的那个图片给裁切还要处理阴影，gpu没办法调过头去渲染刚才的第一个图层，所以只能开辟中间缓冲区中转一下

* **打开 Xcode**，在顶部菜单栏选择 **Product > Profile**，这会打开 Instruments 应用。

  在弹出的模板选择器中，选择 **`Core Animation`** 模板。

---

### 一、离屏渲染是什么？（一个“厨师做菜”的比喻）

**技术定义**：
离屏渲染（Off-Screen Rendering）指的是 GPU 在当前屏幕缓冲区之外，预先开辟一个新的缓冲区，对部分图层进行渲染，并将渲染结果保存起来。当需要时，再将这个已经渲染好的结果拷贝到屏幕缓冲区上，最终显示出来。

---

### 二、为什么会触发离屏渲染？

离屏渲染的本质原因是：**有些视觉效果无法通过一次渲染通道（one-pass rendering）就完成，需要依赖于图层之前的渲染结果。**

最常见的触发条件有以下几种：

1.  **圆角（`cornerRadius`）与 `masksToBounds` 同时使用**：
    *   **原因**：这是最普遍的场景。当只设置 `cornerRadius` 时，只是影响了背景色和边框，内容（比如子视图 `UIImageView`）还是方的。当同时设置 `masksToBounds = YES` 时，系统必须先将该图层及其所有子图层渲染到一个离屏缓冲区，然后根据 `cornerRadius` 对这个缓冲区进行裁剪，最后再将裁剪后的结果贴回到屏幕上。
    *   **例外**：如果一个图层仅仅是 `UIImageView`，并且只设置了它的 `image`，同时使用圆角和裁剪，iOS 系统有特殊的优化，**可能不会**触发离屏渲染。但只要上面有其他子视图，就一定会触发。

---

### 三、离屏渲染为什么会影响性能？

它主要的开销在于：

1.  **创建离屏缓冲区**：需要在内存中额外分配一块空间来存储中间渲染结果，增加了内存开销。
2.  **上下文切换（Context Switch）**：这是**最大的性能杀手**。渲染过程需要从当前的屏幕缓冲区（On-Screen）切换到离屏缓冲区（Off-Screen），完成渲染后，再切换回屏幕缓冲区，把结果拷贝回去。GPU 的上下文切换非常耗时。
3.  **GPU 额外工作负载**：整个渲染管线需要至少执行两次，一次离屏，一次上屏，增加了 GPU 的工作量。

这些开销在单个视图上可能不明显，但如果在 `UITableView` 或 `UICollectionView` 的大量 cell 中频繁触发，就会导致滑动时帧率下降，出现**卡顿**。

---

### 四、如何检测和避免离屏渲染？

#### 检测工具

1.  **Simulator (模拟器)**:
    *   在菜单栏选择 `Debug` -> `Color Off-screen Rendered`。
    *   开启后，所有触发了离屏渲染的区域都会被高亮成**黄色**。这是最直观、最快捷的检测方法。

2.  **Instruments**:
    *   使用 `Core Animation` 工具，勾选 `Color Off-screen Rendered Yellow` 选项，可以在真机上进行检测。

#### 避免策略（对症下药）

常规做法，也就是**会触发离屏渲染**的写法是这样的：
```objc
// 在 MyTableViewCell.m 中
- (void)awakeFromNib {
    [super awakeFromNib];
    
    // 这种写法简单直接，但性能开销大
    self.layer.cornerRadius = 10.0;
    self.layer.masksToBounds = YES; // 或者 self.clipsToBounds = YES;
}
```
---

### 方案一：使用带圆角的背景图 (UI/资源层面解决)

这是**最简单、最高效**的方案，因为它从根本上避免了运行时的裁剪计算。

*   **核心思想**：让视图的“圆角”效果仅仅是一种视觉呈现，而不是通过实时计算裁剪得到的。

*   **具体实现**：
    1.  **让 UI 设计师直接提供**一张中间内容透明、四个角是背景色的 PNG 图片。
    2.  在你的 Cell 中，将这张图片作为背景 `UIImageView`，并置于所有其他内容的**最下层**。
    3.  你的内容视图（比如 `contentView`）的 `backgroundColor` 应该设置为 `clearColor`。
    4.  **不需要**设置 `cornerRadius` 和 `masksToBounds`。

*   **示例代码**：
    ```objc
    // 在 MyTableViewCell.m 中
    - (void)awakeFromNib {
        [super awakeFromNib];
        
        // 假设 self.backgroundImageView 是 XIB 中拖好的
        // 或者代码创建并添加到最底层
        self.backgroundImageView.image = [UIImage imageNamed:@"rounded_corner_background"];
        
        // 确保内容视图是透明的，这样才能透出下面的圆角背景
        self.contentView.backgroundColor = [UIColor clearColor];
    }
    ```

*   **优点**：
    *   性能极高，零运行时开销。
    *   实现简单。
*   **缺点**：
    *   依赖 UI 资源，不够灵活。如果圆角大小、背景色需要动态改变，这个方案就不适用了。
    *   如果 Cell 的内容（比如一个 `UIImageView`）需要被裁剪，这个方案也无法解决。

---

### 方案二：遮罩法 (视觉欺骗)

如果你的 Cell 背景是纯色（比如白色），这个方案非常巧妙且高效。

*   **核心思想**：不裁剪内容，而是在内容的**上层**盖住四个角，模拟出圆角的效果。

*   **具体实现**：
    1.  在 Cell 的四个角上，分别添加四个小的 `UIView`。
    2.  这四个 `UIView` 的背景色**必须和 Cell 的父视图背景色完全相同**（比如 `tableView` 的背景色）。
    3.  通过 Auto Layout 或计算 Frame 将它们精确定位在四个角。
    4.  **不需要**设置 `cornerRadius` 和 `masksToBounds`。

*   **优点**：
    *   性能很高，只增加了四个简单的 `UIView` 渲染开销。
    *   非常灵活，可以动态改变。
*   **缺点**：
    *   只适用于**纯色背景**的场景。如果背景是渐变色或图片，这个方法就行不通了。

---

### 方案三：使用 `CAShapeLayer` 作为蒙版 (Mask)

这是**最通用、最推荐**的纯代码解决方案。它利用了 `CAShapeLayer` 的一个特性：GPU 可以非常高效地利用 `CAShapeLayer` 的路径来进行渲染和裁剪，这个过程通常被高度优化，不会触发离屏渲染。

*   **核心思想**：用一个“形状图层” (`CAShapeLayer`) 来告诉渲染引擎最终的视图应该是什么形状，而不是让渲染引擎自己去计算裁剪。

*   **具体实现**：
    1.  创建一个 `UIBezierPath` 来定义你想要的圆角矩形路径。
    2.  用这个路径创建一个 `CAShapeLayer`。
    3.  将这个 `CAShapeLayer` 设置为你的视图 `layer` 的 `mask`。

*   **示例代码**：
    ```objc
    // 在 MyTableViewCell.m 或自定义 UIView 中
    // 这个方法应该在视图 bounds 确定后调用，比如 layoutSubviews
    - (void)layoutSubviews {
        [super layoutSubviews];
        
        // 如果 mask 已经存在，并且大小没变，就无需重复创建
        if (self.layer.mask && CGRectEqualToRect(self.layer.mask.frame, self.bounds)) {
            return;
        }
    
        // 1. 创建圆角路径
        UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds
                                                       byRoundingCorners:UIRectCornerAllCorners // 可以指定任意角
                                                             cornerRadii:CGSizeMake(10.0, 10.0)];
        
        // 2. 创建 CAShapeLayer
        CAShapeLayer *maskLayer = [[CAShapeLayer alloc] init];
        maskLayer.frame = self.bounds;
        maskLayer.path = maskPath.CGPath;
        
        // 3. 设置为视图的 mask
        self.layer.mask = maskLayer;
    }
    ```

*   **优点**：
    *   性能远高于 `cornerRadius` + `masksToBounds`。
    *   极其灵活，可以创建任意复杂的形状（圆形、五角星等），并且可以动态改变。
    *   适用所有场景（纯色、渐变、图片背景等）。
*   **缺点**：
    *   代码量比 `cornerRadius` 稍多。
    *   需要注意在 `layoutSubviews` 中处理，确保 `bounds` 是正确的，并避免不必要的重复创建。

## ***app从点击屏幕到完成渲染的过程（相当于手势识别+事件传递+响应链）

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

## layoutIfNeed

好的，`layoutIfNeeded` 是 iOS Auto Layout 中一个非常重要且经常被误解的方法。简单来说，它的作用是**强制立即执行视图及其子视图的布局**。

为了彻底理解它，我们需要先了解 iOS 的标准布局流程。

### iOS 的标准布局流程 

当你修改一个视图的约束（例如，改变一个 `height` 约束的 `constant` 值）时，系统并不会马上重新计算并更新视图的 `frame`。相反，它会采取一种更高效的“延迟”策略：

1.  **标记为“需要布局”**：系统会调用 `setNeedsLayout` 方法，在视图上设置一个“脏标记 (dirty flag)”，表示“这个视图的布局已经过时了，需要在未来某个时刻更新”。
2.  **等待更新周期**：系统会等到当前的**运行循环 (Run Loop)** 结束时，才开始处理所有被标记为“需要布局”的视图。
3.  **执行布局**：在一个统一的布局传递 (Layout Pass) 中，系统会调用这些视图的 `layoutSubviews` 方法，根据约束重新计算它们的 `frame`，并最终更新到屏幕上。

这个延迟处理的机制非常高效，因为它可以将一个运行循环内发生的多次布局相关的修改（比如你连续修改了 5 个约束）合并成一次最终的布局计算，避免了不必要的性能开销。

---

### `layoutIfNeeded` 的作用：打破常规

`layoutIfNeeded` 的作用就是打破上述的延迟机制。当你调用一个视图的 `layoutIfNeeded` 方法时，它会告诉系统：**“不要再等了，如果这个视图的布局被标记为需要更新，就立刻、马上、现在就执行布局，更新它的 `frame`！”**

*   **同步执行**：`layoutIfNeeded` 是一个**同步**方法。也就是说，当这个方法执行完毕返回时，你可以保证视图的 `frame` 已经根据当前的约束更新到了最新状态。
*   **递归调用**：它会从调用该方法的视图开始，递归地对所有子视图进行布局。

---

### 最核心的应用场景：动画化约束变更

`layoutIfNeeded` 最常见也是最重要的用途，就是**在 `UIView` 动画块中实现约束变更的动画效果**。

让我们看一个例子：点击一个按钮，让一个视图的高度从 100 变为 200，并带有动画效果。

**错误的做法（没有动画效果）：**

```objective-c
- (IBAction)buttonTapped:(id)sender {
    // 1. 修改约束
    self.heightConstraint.constant = 200;

    [UIView animateWithDuration:0.3 animations:^{
        // 2. 此时视图的 frame 还没有真正改变
        // 动画块里没有什么可以变化的，所以没有动画效果
    }];
}
```
**为什么这样不行？**
在动画块开始时，`heightConstraint` 的值虽然变了，但视图的 `frame` 仍然是旧的。因为标准的布局流程是延迟的，`frame` 的更新要等到这个 run loop 的末尾才会发生。所以动画块捕获不到 `frame` 的变化，视图会直接“跳”到最终状态。

**正确的做法 (使用 `layoutIfNeeded`)：**

```objective-c
- (IBAction)buttonTapped:(id)sender {
    // 1. 在动画块外部，修改约束的 constant 值
    self.heightConstraint.constant = 200;

    // 2. 在动画块内部，调用 superview 的 layoutIfNeeded
    [UIView animateWithDuration:0.3 animations:^{
        [self.view layoutIfNeeded]; // 强制立即布局
    }];
}
```
**这套组合拳的工作原理（非常关键）：**
1.  **捕获初始状态**：`[UIView animateWithDuration:...]` 方法在执行时，会先记录下所有参与动画的视图的当前状态（包括 `frame`, `alpha`, `transform` 等）。
2.  **修改约束**：我们把 `heightConstraint` 的值从 100 改为 200。此时，视图的布局被标记为“需要更新”，但 `frame` 还是旧的（高度为 100）。
3.  **进入动画块**：动画块开始执行。
4.  **强制布局**：在动画块内部，我们调用了 `[self.view layoutIfNeeded]`。系统立即根据新的约束（高度为 200）重新计算并设置了视图的 `frame`。
5.  **产生动画**：动画块检测到视图的 `frame` 从初始状态（高度 100）变成了新的状态（高度 200）。于是，它会在 0.3 秒内，平滑地将 `frame` 从旧值过渡到新值，从而产生了我们想要的动画效果。

> **注意**：通常是在**父视图 (superview)** 上调用 `layoutIfNeeded`，因为约束的改变可能会影响到父视图以及其他兄弟视图的布局。调用 `self.view` (视图控制器的主视图) 通常是最保险的做法。

---

### 另一个应用场景：立即获取更新后的 `frame`

有时，你在代码中通过约束添加了一个新视图，并需要立即知道它布局完成后的 `frame` 大小，以便进行后续计算。

```objective-c
UIView *myView = [[UIView alloc] init];
[self.view addSubview:myView];

[myView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self.view);
    make.width.equalTo(self.view).multipliedBy(0.5);
    make.height.equalTo(@100);
}];

// 此时直接打印 myView.frame，很可能是 CGRectZero
NSLog(@"Frame before layout: %@", NSStringFromCGRect(myView.frame)); // -> {{0, 0}, {0, 0}}

// 强制立即布局
[self.view layoutIfNeeded];

// 现在可以获取到正确的 frame 了
NSLog(@"Frame after layout: %@", NSStringFromCGRect(myView.frame)); // -> {{93.75, 383.5}, {187.5, 100}} (示例值)
```

---

### `layoutIfNeeded` vs `setNeedsLayout`

这是一个常见的面试题，总结一下它们的区别：

| 特性         | `setNeedsLayout`                                             | `layoutIfNeeded`                                             |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **执行时机** | **异步 (Asynchronous)**                                      | **同步 (Synchronous)**                                       |
| **行为**     | 标记视图为“需要布局”，等待系统在下一个更新周期调用 `layoutSubviews`。 | 如果视图被标记为需要布局，则**立即**触发 `layoutSubviews`。  |
| **用途**     | 大多数情况下，当你希望系统自动、高效地处理布局更新时使用。   | 当你需要**立即**获取更新后的 `frame`，或在动画块中**强制布局**以产生动画效果时使用。 |
| **性能**     | 高效，会将多次变更合并为一次。                               | 相对耗性能，因为它会立即触发布局计算，应避免在循环或频繁调用的地方使用。 |

### 总结

*   `layoutIfNeeded` 是一个**强制立即布局**的命令。
*   它的核心用途是配合 `UIView` 动画块，实现**约束变化的动画**。
*   它也可以用来在修改约束后，**立即获取视图的准确 `frame`**。
*   理解它与 `setNeedsLayout` 的**同步/异步**区别是掌握 Auto Layout 动画的关键。

## masonry的约束应该写在哪里比较合适

在 iOS 开发中，使用 Masonry 设置约束的位置对于代码的清晰度、性能和可维护性至关重要。通常，约束的编写位置分为**初始设置**和**动态更新**两种场景。

### 结论先行：最佳实践总结

| 场景               | `UIView` 子类                               | `UIViewController`                               |
| :----------------- | :------------------------------------------ | :----------------------------------------------- |
| **首次创建约束**   | 在初始化方法 (`init` / `initWithFrame:`) 中 | 在 `viewDidLoad` 中                              |
| **更新约束**       | 覆盖 `updateConstraints` 方法               | 直接在事件响应方法（如按钮点击）或状态改变时更新 |
| **避免使用的位置** | `layoutSubviews`                            | `viewDidLayoutSubviews` / `viewWillAppear`       |

---

### 1. 首次创建和设置约束 (Initial Setup)

对于只设置一次、不会再改变的约束，应该在视图或视图控制器生命周期中尽可能早地添加，以确保高效。

#### 在 `UIView` 的子类中：初始化方法

在自定义 `UIView` 子类中，最适合设置初始约束的地方是其**初始化方法**（如 `initWithFrame:` 或自定义的 `init`）。

*   **原因**:
    *   **职责单一**：视图的布局是其内部实现的一部分，封装在视图内部可以提高复用性和独立性。
    *   **时机最早**：在视图被创建时就立即定义其内部子视图的布局关系，逻辑清晰。
    *   **性能**：约束只被添加一次，避免了在 `layoutSubviews` 等可能被多次调用的方法中重复添加。

*   **示例代码**:
    ```objective-c
    @implementation CustomView
    
    - (instancetype)initWithFrame:(CGRect)frame {
        self = [super initWithFrame:frame];
        if (self) {
            // 1. 创建并添加子视图
            UILabel *titleLabel = [[UILabel alloc] init];
            [self addSubview:titleLabel];
    
            // 2. 使用 mas_makeConstraints 添加约束 (只执行一次)
            [titleLabel mas_makeConstraints:^(MASConstraintMaker *make) {
                make.top.equalTo(self).offset(10);
                make.centerX.equalTo(self);
            }];
        }
        return self;
    }
    
    @end
    ```

#### 在 `UIViewController` 中：`viewDidLoad`

当你在视图控制器中直接管理多个视图的布局时，`viewDidLoad` 是设置初始约束的最佳位置。

*   **原因**:
    *   `viewDidLoad` 在视图控制器的生命周期中只被调用一次，保证了约束只会被创建一次。
    *   此时，视图控制器的 `view` 及其所有子视图都已经加载到内存中，可以安全地添加约束。

*   **示例代码**:
    ```objective-c
    @implementation MyViewController
    
    - (void)viewDidLoad {
        [super viewDidLoad];
    
        // 1. 创建并添加子视图
        UIView *redView = [[UIView alloc] init];
        redView.backgroundColor = [UIColor redColor];
        [self.view addSubview:redView];
    
        UIView *blueView = [[UIView alloc] init];
        blueView.backgroundColor = [UIColor blueColor];
        [self.view addSubview:blueView];
    
        // 2. 使用 mas_makeConstraints 添加约束
        [redView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.top.equalTo(self.view).offset(100);
            make.left.equalTo(self.view).offset(20);
            make.width.height.equalTo(@100);
        }];
    
        [blueView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.top.equalTo(redView.mas_bottom).offset(20);
            make.left.equalTo(redView);
            make.width.height.equalTo(redView);
        }];
    }
    
    @end
    ```



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
    #import <Foundation/Foundation.h>
    
    // 定义条件值
    #define HAS_DATA_CONDITION 1
    #define NO_DATA_CONDITION 0
    
    @interface MyBuffer : NSObject
    @property (strong, nonatomic) NSConditionLock *conditionLock;
    @property (strong, nonatomic) NSMutableArray *dataArray;
    @end
    
    @implementation MyBuffer
    
    - (instancetype)init {
        self = [super init];
        if (self) {
            // 创建一个初始条件值为 NO_DATA_CONDITION 的锁
            self.conditionLock = [[NSConditionLock alloc] initWithCondition:NO_DATA_CONDITION];
            self.dataArray = [NSMutableArray array];
        }
        return self;
    }
    
    // 生产者方法：向缓冲区添加数据
    - (void)produceData:(NSString *)data {
        // 生产者必须在 NO_DATA_CONDITION 的条件下才能加锁
        [self.conditionLock lockWhenCondition:NO_DATA_CONDITION];
    
        // 添加数据
        [self.dataArray addObject:data];
        NSLog(@"Produced data: %@. Current count: %lu", data, (unsigned long)self.dataArray.count);
        
        // 释放锁，并改变条件值为 HAS_DATA_CONDITION
        // 这会唤醒等待 HAS_DATA_CONDITION 的消费者线程
        [self.conditionLock unlockWithCondition:HAS_DATA_CONDITION];
    }
    
    // 消费者方法：从缓冲区取出数据
    - (NSString *)consumeData {
        // 消费者必须在 HAS_DATA_CONDITION 的条件下才能加锁
        [self.conditionLock lockWhenCondition:HAS_DATA_CONDITION];
    
        // 取出数据
        NSString *data = self.dataArray.firstObject;
        if (data) {
            [self.dataArray removeObjectAtIndex:0];
        }
        NSLog(@"Consumed data: %@. Current count: %lu", data, (unsigned long)self.dataArray.count);
    
        // 释放锁，并根据缓冲区是否为空改变条件值
        if (self.dataArray.count > 0) {
            [self.conditionLock unlockWithCondition:HAS_DATA_CONDITION];
        } else {
            [self.conditionLock unlockWithCondition:NO_DATA_CONDITION];
        }
        return data;
    }
    
    @end
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

## 如何使线程保活

## TableView优化，怎么减少卡顿

好的，我们继续深入。掌握了 Cell 的正确复用后，下一个对性能影响最大的因素，就是 **Cell 高度的计算**。

### 方案二：高效计算并缓存 Cell 高度

`UITableView` 在显示内容前，需要知道每个 Cell 的确切高度，以便计算出自己的 `contentSize`、滚动条的位置等信息。如果高度计算过于耗时，就会在滑动时严重拖慢主线程，导致卡顿。

#### 1. 问题的根源：`heightForRowAtIndexPath` 的调用时机

你可能会认为 `-tableView:heightForRowAtIndexPath:` 这个代理方法只会在 Cell 即将出现时才被调用。但事实并非如此：

*   **加载时**：`UITableView` 在第一次加载时，会调用**所有可见 Cell**的 `heightForRowAtIndexPath:`。
*   **滑动时**：每次滑动，它不仅会为**新出现的 Cell** 调用此方法，还可能为了计算滚动条的比例而调用**大量甚至全部** Cell 的 `heightForRowahoawIndexPath:`。

如果你的高度计算逻辑很复杂（比如基于文本内容动态计算），并且每次都实时计算，那么当列表数据量很大时，性能开销将是巨大的。

#### 2. 优化策略：计算与缓存分离

核心思想是：**不要在 `heightForRowAtIndexPath:` 中进行任何实时计算**。这个方法应该像一个高速的查询接口，只负责从一个已经准备好的地方**读取**高度值。真正的计算应该提前完成，并且结果要被**缓存**起来。

**第一步：将高度计算逻辑移到数据模型（Model）中**

让数据模型自己负责计算其对应内容所需的高度。这样，高度就和数据绑定在了一起，逻辑更清晰。

**`MyDataModel.m`**
```objc
// Objective-C
#import <UIKit/UIKit.h>

@implementation MyDataModel

- (CGFloat)cellHeight {
    // _cellHeight 是在 .h 中声明的属性，用于缓存
    // 如果已经计算过，直接返回缓存值
    if (_cellHeight > 0) {
        return _cellHeight;
    }
    
    // --- 在这里执行复杂的计算逻辑 ---
    CGFloat height = 0;
    
    // 假设有标题、内容和图片
    height += 10; // 顶部间距
    
    // 计算标题高度
    height += [self.title boundingRectWithSize:CGSizeMake(CELL_CONTENT_WIDTH, CGFLOAT_MAX) 
                                      options:NSStringDrawingUsesLineFragmentOrigin 
                                   attributes:@{NSFontAttributeName: [UIFont systemFontOfSize:17]} 
                                      context:nil].size.height;
    
    height += 8; // 标题和内容间距
    
    // 计算内容高度
    height += [self.content boundingRectWithSize:CGSizeMake(CELL_CONTENT_WIDTH, CGFLOAT_MAX) 
                                        options:NSStringDrawingUsesLineFragmentOrigin 
                                     attributes:@{NSFontAttributeName: [UIFont systemFontOfSize:14]} 
                                        context:nil].size.height;
    
    if (self.image) {
        height += 8; // 内容和图片间距
        height += 150; // 图片固定高度
    }
    
    height += 10; // 底部间距
    
    // 将计算结果缓存起来
    _cellHeight = height;
    
    return _cellHeight;
}
```
**注意**: `CELL_CONTENT_WIDTH` 是指 Cell 内容区域的宽度，你需要根据实际布局来确定。

**第二步：在获取数据时提前计算并缓存**

在网络请求返回、数据解析完成后，立即遍历所有数据模型，触发一次高度计算。

```objc
// Objective-C
- (void)processFetchedData:(NSArray *)newData {
    NSMutableArray *processedData = [NSMutableArray array];
    for (NSDictionary *dataDict in newData) {
        MyDataModel *model = [[MyDataModel alloc] initWithDictionary:dataDict];
        // 关键一步：在添加到数据源之前，就计算并缓存好高度
        [model cellHeight]; 
        [processedData addObject:model];
    }
    
    self.dataSource = processedData;
    [self.tableView reloadData];
}
```
这个过程可以在后台线程完成，避免阻塞主线程。

**第三步：`heightForRowAtIndexPath` 只负责读取**

现在，这个代理方法变得极其轻量和高效。

```objc
// Objective-C
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    MyDataModel *model = self.dataSource[indexPath.row];
    // 直接从模型中读取已缓存的高度
    return model.cellHeight;
}
```

好的，前两个方案解决了 Cell 的生命周期管理和高度计算效率，接下来我们关注 Cell 内容的绘制，特别是图片这类耗时资源的处理。

---

### 方案三：异步图片加载与预加载 / 异步绘制 (Offloading Work from Main Thread)

在 `UITableViewCell` 中显示图片，尤其是从网络加载的图片，是导致卡顿的常见原因。

#### 1. 问题的根源：主线程阻塞

当你在 `cellForRowAtIndexPath` 中同步地（在主线程）执行以下任何操作，都可能导致卡顿：

*   **加载网络图片**：发出网络请求，等待图片下载完成。
*   **处理大尺寸本地图片**：将磁盘上的大图加载到内存，进行解码。
*   **对图片进行操作**：裁剪、缩放、添加滤镜、合成等。
*   **大量文字排版计算**：虽然大部分文字渲染由系统优化，但极端复杂的自定义排版仍可能耗时。

这些操作通常耗时较长，如果发生在主线程，就会阻塞 UI 渲染，导致掉帧。

#### 2. 优化策略：将耗时操作移出主线程

核心思想是：**任何可能耗时的操作都应该放到后台线程执行**，而主线程只负责更新 UI。

**第一步：异步加载网络图片 (推荐使用第三方库)**

这是最常见也最简单有效的优化。不要自己手动写 `NSURLSession` 去下载图片，而是使用成熟的第三方库。

*   **例如：SDWebImage (或其他类似库如 Kingfisher, Nuke)**
    这些库自带了图片缓存（内存缓存和磁盘缓存）、异步下载、占位图、取消旧请求、渐进式加载等一系列高级功能。

    ```objc
    // Objective-C
    #import <SDWebImage/UIImageView+WebCache.h>
    
    - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
        MyTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"MyCellIdentifier" forIndexPath:indexPath];
        
        MyDataModel *data = self.dataSource[indexPath.row];
        
        // 关键：使用 SDWebImage 异步加载图片
        // 会自动处理占位图、缓存、下载、取消旧请求等
        [cell.customImageView sd_setImageWithURL:[NSURL URLWithString:data.imageUrl] 
                                placeholderImage:[UIImage imageNamed:@"placeholder"]];
        
        return cell;
    }
    ```
    *   **原理**：这些库在后台线程进行图片下载和解码，下载完成后在主线程更新 `UIImageView`。

好的，我们已经讨论了 Cell 的生命周期、高度计算和内容加载。现在，我们来关注一个更深层次的性能问题：**图层混合与离屏渲染**。

---

### 方案四：减少图层混合与避免离屏渲染

即使 Cell 的内容和高度都已优化，不恰当的视觉效果设置仍然可能给 GPU 带来沉重负担，导致性能下降。

#### 1. 问题的根源：GPU 的额外工作

**A. 图层混合 (Blending)**

*   **是什么？** 当多个半透明（alpha < 1.0）的图层叠加在一起时，GPU 需要计算它们混合后的最终颜色。这个计算过程不是简单的颜色覆盖，而是需要读取目标像素颜色，再结合源像素颜色和透明度，通过公式计算出新颜色。
*   **为什么耗性能？** 这个“读-改-写”的过程比简单地用新颜色覆盖旧颜色（不透明图层）要消耗更多的 GPU 处理能力。如果在一个 `UITableView` 中，每个 Cell 都有多层半透明视图叠加，滑动时 GPU 的负担会非常重。

**B. 离屏渲染 (Off-Screen Rendering)**

*   我们在之前的讨论中已经详细解释过。它会**创建额外的缓冲区**并导致**昂贵的上下文切换**。在 `UITableView` 中，最常见的触发源就是**圆角裁剪**和**阴影**。

#### 2. 优化策略：让 GPU “减负”

核心思想是：**尽量让视图不透明，并避免使用需要多趟渲染才能实现的视觉效果。**

#### **如何检测？**

*   **图层混合**：在模拟器 `Debug` 菜单中，选择 `Color Blended Layers`。所有发生混合的区域都会被高亮成**红色**。不透明的区域是绿色。你的目标是**尽可能减少红色区域**。
*   **离屏渲染**：同样在 `Debug` 菜单中，选择 `Color Off-screen Rendered`。触发离屏渲染的区域会高亮成**黄色**。你的目标是**消除不必要的黄色区域**。

#### **如何优化？**

**A. 减少和消除图层混合**

1.  **设置 `backgroundColor`**：
    *   **最常见的问题**：`UILabel`, `UIImageView` 等控件默认的 `backgroundColor` 是 `nil`（即透明）。如果它们被放在一个有背景色的父视图上，就会发生图层混合。
    *   **解决方案**：为 Cell 内的每个视图控件（特别是 `UILabel`）显式设置一个**不透明的背景色**。如果它的背景应该和 Cell 的背景一样，就设置成 Cell 的背景色。
        ```objc
        // 在 MyTableViewCell.m 中
        myLabel.backgroundColor = [UIColor whiteColor]; // 或者 self.contentView.backgroundColor
        myLabel.layer.masksToBounds = YES; // 对于 UILabel，设置这个可以进一步优化性能
        ```

2.  **设置 `opaque` 属性**：
    *   向系统表明这个视图是完全不透明的。这可以让渲染系统进行一些优化，比如在绘制时跳过其后面的任何内容。
    *   当你确定一个视图（及其子视图）完全不透明时，可以设置 `view.layer.opaque = YES;`。`UIImageView` 默认就是 `YES`。

3.  **避免使用 `alpha` < 1.0**：
    *   如果只是想让一个视图的颜色变浅，**不要**直接设置 `view.alpha = 0.8;`。
    *   **更好的方法**是计算出一个混合后的**不透明颜色**，并直接使用这个颜色。例如，如果背景是白色，你想要一个 80% 透明度的黑色文字，那么直接设置文字颜色为 20% 的灰色 (`[UIColor colorWithWhite:0.2 alpha:1.0]`) 效果是一样的，但完全不透明。

**B. 避免离屏渲染**

我们在之前的讨论中已经详细列出了策略，这里针对 `UITableView` 场景再强调一下：

1.  **圆角 (`cornerRadius` + `masksToBounds`)**：
    *   **最佳方案**：让 UI 设计师直接提供带圆角的图片。
    *   **次优方案**：使用 `CAShapeLayer` 作为 `mask`，或者在后台线程异步绘制圆角图片。
    *   **避免**：直接在 `cellForRowAtIndexPath` 中对 Cell 的 `layer` 设置 `cornerRadius` 和 `masksToBounds`。

2.  **阴影 (`shadow`)**：
    *   **必须**设置 `shadowPath` 来预先告诉 Core Animation 阴影的形状，避免离屏计算。
        ```objc
        // 在 MyTableViewCell.m 的 layoutSubviews 方法中设置
        - (void)layoutSubviews {
            [super layoutSubviews];
            self.layer.shadowPath = [UIBezierPath bezierPathWithRect:self.bounds].CGPath;
        }
        ```

3.  **光栅化 (`shouldRasterize`)**：
    *   再次强调，这是一个**缓存**机制，它本身会**触发一次离屏渲染**。
    *   **明智地使用它**：只对那些内容**静态不变**但视觉效果复杂（比如包含阴影、圆角、多种控件）的 Cell 开启，可以极大提升后续滚动的性能。对于内容频繁变化的 Cell，则要禁用。

#### 核心要点

*   **打开调试开关**：善用 `Color Blended Layers` (红色) 和 `Color Off-screen Rendered` (黄色) 来定位问题。
*   **消灭红色 (混合)**：给控件设置**不透明的背景色**，避免使用 `alpha` 透明度。
*   **消灭黄色 (离屏)**：高效处理**圆角**和**阴影**，这是 `UITableView` 中最常见的离屏渲染来源。

通过对 GPU 渲染层面的优化，你可以进一步压榨出列表的性能，使其在各种设备上都能如丝般顺滑。

接下来，我们可以讨论一些更零散但同样有效的“锦上添花”的优化技巧。

## oc的cell重用机制 重用的数量由什么决定

### 深入解析OC中的Cell重用机制

`UITableView`和`UICollectionView`之所以能够高效处理大量数据，其核心就在于**Cell的重用机制**。想象一下，如果一个列表有一万行数据，为每一行都创建一个`UITableViewCell`对象，将会消耗巨大的内存，导致应用卡顿甚至崩溃。重用机制就是为了解决这个问题而设计的。

其核心思想是：**只创建足够在屏幕上显示以及少量用于缓冲的Cell实例，当Cell滚出屏幕时，不销毁它，而是将其放入一个“重用池”（Reuse Queue）中。当需要显示新的Cell时，优先从这个重用池中取出一个已经存在的Cell实例，而不是重新创建一个。**

---

### 决定重用数量的核心因素：一个动态的平衡

重用池中Cell的数量以及系统总共创建的Cell实例数量，并不是一个固定的值，而是由以下几个关键因素动态决定的：

1.  **屏幕可见区域的高度 (Bounds Height):**
    *   这是最直接的决定因素。系统会计算在当前`UITableView`或`UICollectionView`的可见区域内，能够容纳多少个Cell。
    *   例如，如果一个`UITableView`的高度是600个点（points），每个Cell的高度是60个点，那么屏幕上最多可以完整显示10个Cell。系统就必须至少创建10个Cell实例来填满这个可见区域。
2.  **Cell的高度或尺寸 (Item Size):**
    *   Cell的尺寸直接影响了屏幕上可以同时显示的数量。如果Cell的高度变小，那么屏幕上需要显示的Cell数量就会增多，系统相应地就需要创建更多的实例来填满屏幕。反之亦然。
    *   对于高度不固定的Cell（例如使用`UITableViewAutomaticDimension`），系统会根据估算高度和实际内容计算来动态管理。
3.  **缓冲区域 (Buffer Zone):**
    *   为了确保滚动的流畅性，系统并不会只创建刚好填满屏幕的Cell。它会在可见区域的上方和下方额外创建并缓存一小部分Cell，形成一个缓冲区。
    *   这个缓冲区的作用是，当用户快速滚动时，即将进入屏幕的Cell已经准备好了，可以直接从缓冲区中取出并显示，从而避免了因实时创建Cell实例可能带来的延迟和卡顿。
    *   这个缓冲区的大小是系统内部管理的，通常不大，可能只有几个Cell的数量，但足以应对大多数滚动场景。

### Cell的生命周期与工作流程

让我们通过一个完整的流程来理解这一切是如何运作的：

1.  **首次加载 (Cold Start):**
    *   当`UITableView`首次加载数据时，它会调用`tableView:cellForRowAtIndexPath:`方法。
    *   此时，因为是第一次加载，对应`reuseIdentifier`的重用池是空的。
    *   `dequeueReusableCellWithIdentifier:`会返回`nil`（在旧的API中，或者在使用`registerClass:forCellReuseIdentifier:`或Storyboard注册后，它会为你自动创建一个新的实例）。
    *   你的代码会执行创建新Cell的逻辑（`[[UITableViewCell alloc] initWithStyle:reuseIdentifier:]`）。
    *   这个过程会不断重复，直到`UITableView`的可见区域被填满，并且上下缓冲区也准备好了相应的Cell实例。假设屏幕能显示10个，缓冲区需要2个，那么系统可能会创建大约12个Cell实例。

2.  **滚动与重用 (Scrolling & Reusing):**
    *   当用户向上滚动列表时，最顶部的Cell（比如第0行）开始移出屏幕。
    *   一旦这个Cell完全离开可见区域，它并不会被销毁，而是被系统放入对应其`reuseIdentifier`的重用池中，等待下一次被“召唤”。
    *   同时，列表的底部需要显示一个新的Cell（比如第11行）。
    *   此时再次调用`tableView:cellForRowAtIndexPath:`方法。
    *   `dequeueReusableCellWithIdentifier:`会检查重用池，发现里面有一个刚刚被放入的、可用的Cell（就是之前第0行的那个实例）。
    *   该方法会返回这个Cell的实例。**注意：返回的是一个已经存在的对象，它可能还保留着之前显示的数据和状态（比如文本、颜色等）。**

3.  **数据更新与状态重置:**
    *   在`tableView:cellForRowAtIndexPath:`中，拿到这个重用的Cell后，你的责任是**用新的数据覆盖旧的数据**。例如，`cell.textLabel.text = self.dataArray[indexPath.row];`。
    *   **一个常见的陷阱是：** 如果你只在某些条件下设置了Cell的某个属性（比如高亮颜色），而没有在其他条件下将其重置，那么当这个Cell被重用时，它可能会带着之前的高亮状态错误地显示出来。
    *   为了解决这个问题，最佳实践是在自定义Cell的`prepareForReuse`方法中，将所有可能变化的状态重置为默认值。这个方法在Cell被重用之前，由系统自动调用。

    ```objectivec
    // In MyCustomCell.m
    - (void)prepareForReuse {
        [super prepareForReuse];
        // Reset all custom properties to their default state
        self.customLabel.text = @"";
        self.customImageView.image = nil;
        self.contentView.backgroundColor = [UIColor whiteColor];
    }
    ```

### 总结

总而言之，重用Cell的数量不是一个固定的数值，而是一个由**“屏幕可见数量 + 少量缓冲区数量”**共同决定的动态值。这个值会受到设备屏幕尺寸、Cell自身尺寸以及滚动速度等多种因素的影响。作为开发者，我们不需要也无法直接控制这个数量，但我们需要深刻理解其工作原理，并遵循以下最佳实践，以确保列表的性能和正确性：

*   **始终使用`dequeueReusableCellWithIdentifier:`** 来获取Cell实例。
*   **为不同类型的Cell使用不同的`reuseIdentifier`**。
*   **在`tableView:cellForRowAtIndexPath:`中，总是为Cell的所有UI元素设置新的数据**，覆盖可能存在的旧数据。
*   **在自定义Cell的`prepareForReuse`方法中，将所有自定义的状态重置为初始状态**，以避免数据错乱。

### 优化清单总结

| 优先级   | 问题领域        | 优化策略                                            |
| -------- | --------------- | --------------------------------------------------- |
| **最高** | **Cell 高度**   | **预计算并缓存高度到数据模型**                      |
| **最高** | **图片处理**    | **异步加载 + 后台解码/缩放 (使用 SDWebImage)**      |
| **高**   | **Cell 创建**   | 保证 dequeueReusableCell，保持视图层级扁平          |
| **高**   | **图形渲染**    | 避免离屏渲染（圆角、阴影），设置视图为 opaque       |
| **中**   | **数据处理**    | 网络请求、JSON 解析等全部放到后台线程               |
| **中**   | **Auto Layout** | 如果使用自适应高度，**必须**提供 estimatedRowHeight |

## 有哪些crash的情况

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

## @dynamic和@synthesize

* `@dynamic` 的作用就是将生成存取方法以及与之相关的实例变量的任务，完全推迟到运行时。并且使用a.x 语法不会报错。
* @synthesize的作用就是自动生成实现，并且关联到_name变量

```objc
#import "MyClass.h"

@implementation MyClass
@synthesize name = _name; // 告诉编译器，生成name属性的存取方法，并关联到_name实例变量
@end
```



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
        if (self.loginStateDidChange) { 
          self.loginStateDidChange(self); 
        }
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
又或者用kvo更新view的状态

```objc
// LoginViewController.m

- (void)bindViewModel {
    // 注册观察者
    [self.viewModel addObserver:self forKeyPath:@"isLoginButtonEnabled" options:NSKeyValueObservingOptionNew context:nil];
    [self.viewModel addObserver:self forKeyPath:@"loginButtonColor" options:NSKeyValueObservingOptionNew context:nil];
    [self.viewModel addObserver:self forKeyPath:@"statusText" options:NSKeyValueObservingOptionNew context:nil];
    
    // ... View -> ViewModel 的绑定不变 ...
}

// 实现观察者方法
- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object 
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change 
                       context:(void *)context {
    if (object == self.viewModel) {
        // 在这里根据 keyPath 更新对应的 UI
        if ([keyPath isEqualToString:@"isLoginButtonEnabled"]) {
            self.loginButton.enabled = self.viewModel.isLoginButtonEnabled;
        } else if ([keyPath isEqualToString:@"loginButtonColor"]) {
            self.loginButton.backgroundColor = self.viewModel.loginButtonColor;
        } else if ([keyPath isEqualToString:@"statusText"]) {
            self.statusLabel.text = self.viewModel.statusText;
        }
    }
}

- (void)dealloc {
    // 必须移除观察者
    [self.viewModel removeObserver:self forKeyPath:@"isLoginButtonEnabled"];
    [self.viewModel removeObserver:self forKeyPath:@"loginButtonColor"];
    [self.viewModel removeObserver:self forKeyPath:@"statusText"];
}
```

#### 问题一：“请解释一下你理解的MVVM是什么？”

**优秀回答 (展现深度和思考)：**
“MVVM是一种旨在**分离用户界面（View）和业务逻辑/状态（ViewModel）**的架构模式。它的核心思想是解决苹果官方MVC中ViewController过于臃肿（Massive View Controller）的问题。”

“在我看来，MVVM可以这样理解：”
*   **Model:** 和MVC中的一样，是纯粹的数据模型，负责定义数据结构和提供数据。
*   **View/ViewController:** 它的职责被大大简化，只负责两件事：**UI的布局和渲染**，以及**将用户的操作（如点击）转发给ViewModel**。它本身不包含任何业务逻辑。在iOS中，`UIViewController` 和 `UIView` 都属于View层。
*   **ViewModel:** 这是MVVM的精髓。它是一个**专门为View服务的对象**。它从Model获取原始数据，然后进行加工处理，转换成View可以直接展示的**状态和数据**（比如把Date对象格式化成"2023-10-27"这样的字符串）。同时，它也提供命令（Command）来响应View转发过来的用户操作。

“最关键的一点是，**ViewModel不直接持有View的引用**。它们之间通过**数据绑定（Data Binding）**机制进行通信，实现了View和ViewModel的彻底解耦，这为单元测试带来了巨大的便利。”

> **面试官追问点：** 为什么ViewModel不应该持有View？（答案：为了解耦和可测试性。一旦持有View，ViewModel就依赖了UIKit，无法进行脱离UI的单元测试，也容易造成循环引用。）

---

#### 问题二：“MVVM和MVC相比，有什么具体的优缺点？”

**普通回答：**
“MVVM的优点是解耦、可测试性好。缺点是对于简单页面，会增加一些类和复杂度。”

**优秀回答 (结合具体痛点和解决方案)：**
“MVVM最大的优点是它精准地解决了苹果MVC架构下的**‘Massive View Controller’（重量级视图控制器）**问题。在MVC中，大量的业务逻辑、网络请求、数据加工代码都堆在ViewController里，导致其非常臃肿且难以测试。”

“具体来说，MVVM的优势体现在：”
1.  **高度解耦 (High Decoupling):** View和ViewModel通过数据绑定通信，彼此没有直接依赖。ViewModel也不知道展示它数据的是`UILabel`还是`UITextView`，它只负责输出“状态”。这使得更换UI变得非常容易。
2.  **极高的可测试性 (Excellent Testability):** 这是MVVM最核心的优势。因为ViewModel是一个**纯粹的Swift/Objective-C对象**，它不依赖任何UIKit/SwiftUI的组件。所以我们可以非常轻松地在单元测试环境中实例化一个ViewModel，给它输入，然后断言它的输出是否符合预期，测试覆盖率可以做得非常高。
3.  **职责分离 (Separation of Concerns):** UI设计师可以专注于View层，而业务逻辑开发者可以专注于ViewModel层，两者可以并行工作，互不干扰。

---

#### 问题三：“你通常如何实现View和ViewModel之间的数据绑定？”

**普通回答：**

1.  **KVO (Key-Value Observing):** 这是Foundation框架提供的传统机制。可以让一个对象观察另一个对象的属性变化。虽然能用，但API比较古老，语法不直观，且需要注意移除观察者，否则容易导致崩溃。
2.  **闭包/属性观察者 (简单场景):** 对于简单的单向数据流，我们也可以在ViewModel中定义一个闭包，比如`var onDataUpdate: (() -> Void)?`。当数据变化时，调用这个闭包。ViewController在初始化ViewModel时，将UI更新的代码赋给这个闭包。这种方式最轻量，但对于复杂交互就不够用了。

这是一个关于 iOS/macOS 架构模式中对象持有（引用）关系的经典问题。理解这些关系对于避免内存泄漏（尤其是循环引用）至关重要。

## MVC (Model-View-Controller) 中的持有关系

在 Apple 平台的传统 MVC 模式中，对象之间的关系通常是**单向强持有**和**双向弱引用**的结合。

| 对象                    | 持有谁？              | 被谁持有？                                                   | 关系类型 | 说明                                                         |
| ----------------------- | --------------------- | ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| **VC (ViewController)** | **View** (强持有)     | **Navigation Controller** (强持有), **Tab Bar Controller** (强持有), **父 VC** (强持有) | **强**   | VC 是 View 的“拥有者”，生命周期与 View 紧密关联。            |
| **View**                | **Subviews** (强持有) | **VC** (强持有)                                              | **强**   | View 通常不持有任何其他对象，其生命周期由其父 View 或 VC 控制。 |
| **Model**               | (不持有其他架构对象)  | **VC** (强持有) 或 **其他服务层** (强持有)                   | **强**   | Model 是数据对象，由 VC 或数据管理者持有。                   |
| **VC**                  | **Model** (强持有)    | **Model** (无，或弱引用 `delegate`)                          | **强**   | VC 通常强持有它需要展示的数据 Model。                        |
| **Model**               | **VC**                | VC (弱引用 `delegate`)                                       | **弱**   | 如果 Model 需要通知 VC 数据变化，会通过 `delegate` 模式，通常是**弱引用**，以避免循环引用。 |

### MVC 总结

- **VC ↔ View：** VC 强持有 View（UIKit/AppKit 自动设置），View 不持有 VC。
- **VC ↔ Model：** VC 强持有 Model。Model 如果需要反向通信，则对 VC 采用**弱引用 (`weak`) 的 Delegate**。

## MVVM (Model-View-ViewModel) 中的持有关系

### MVVM 总结

- **VC → ViewModel：** **单向强持有**。VC 强持有 ViewModel。
- **ViewModel → VC/View：** **无直接持有**。通过**数据绑定**（通知机制）进行通信。如果使用闭包或回调，必须使用 **`[weak self]`** 来避免循环引用。

**核心目标：** **ViewModel 必须独立于 View 的生命周期。** 当 View/VC 被销毁时，它对 ViewModel 的强引用解除，ViewModel 也应被销毁。



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

## `nil`、`NIL`、`NSNULL` 有什么区别？

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

## `PerformSelector` 的使用和实现原理

## 实例对象，类对象，元类对象的数据结构

太棒了！这个问题是理解 Objective-C 对象模型和消息发送机制的**核心**。实例对象、类对象、元类对象在内存中都是 C 结构体，但它们的结构和指向关系各有不同，共同构成了一个精妙的体系。

---

### 总结图

### 1. 实例对象 (Instance Object) - “具体的房子”

*   **用途**：存储每个对象独有的数据（实例变量）。
*   **数据结构**：`struct objc_object`

```c
struct objc_object {
    Class isa;  // 8字节 (在64位下)
    // --- 紧跟着是所有实例变量 ---
    // NSString *_name;
    // int _age;
    // ...
};
```

*   **结构详解**：
    *   **`isa` 指针**：
        *   这是**唯一必须有**的成员。
        *   它指向该实例所属的**类对象 (Class Object)**。
        *   当你调用一个**实例方法**时，例如 `[personInstance sayHello]`，运行时系统就是通过 `isa` 指针找到 `Person` 类对象，再去它的方法列表里查找 `sayHello` 的实现。
    *   **实例变量 (ivars)**：
        *   紧跟在 `isa` 之后。
        *   存储这个实例的特定状态，比如 `person1` 的 `_name` 是 "张三"，而 `person2` 的 `_name` 是 "李四"。
        *   实例对象本身**不存储任何方法代码**。

### 2. 类对象 (Class Object) - “房子的设计蓝图”

*   **用途**：描述一类对象是什么样的。它存储了**实例方法**、属性、协议等元数据。在内存中，**每个类只有一份类对象**。
*   **数据结构**：`struct objc_class` (它继承自 `objc_object`，所以它也有一个 `isa` 指针)

```c
struct objc_class : objc_object {
    Class superclass;         // 指向父类的类对象
    cache_t cache;            // 方法缓存，用于加速方法查找
    class_data_bits_t bits;   // 核心数据，包含了方法列表、属性列表等
    // ...
};
```

*   **结构详解**：
    *   **`isa` 指针**：
        *   类对象也有 `isa`！它的 `isa` 指向该类的**元类对象 (Meta-Class Object)**。
        *   **为什么？** 因为当你调用一个**类方法**时，例如 `[Person aClassMethod]`，系统也需要一个地方去查找方法的实现。这个地方就是元类。
    *   **`superclass` 指针**：
        *   指向其父类的**类对象**。例如，`Person` 的类对象的 `superclass` 指向 `NSObject` 的类对象。
        *   这是实现**继承**的关键。如果在 `Person` 类中找不到某个实例方法，就会顺着 `superclass` 链向上去父类里找。
    *   **`cache`**：
        *   一个用于缓存最近调用过的方法的哈希表。当一个方法被调用后，其实现会被缓存起来，下次再调用时就可以直接在 `cache` 中找到，极大地提高了性能。
    *   **`bits`**：
        *   这是一个包含了大量信息的结构体指针。我们可以把它理解为一个“工具箱”，里面装着：
            *   **方法列表 (Method List)**：存储所有**实例方法**的信息（方法名 SEL、实现 IMP、类型编码）。
            *   **属性列表 (Property List)**：存储所有 `@property` 声明的属性。
            *   **协议列表 (Protocol List)**：存储该类遵循的所有协议。
            *   **实例变量布局 (Ivar Layout)**：描述实例变量的内存布局。

### 3. 元类对象 (Meta-Class Object) - “蓝图的设计规范”

*   **用途**：元类的**唯一目的**就是存储**类方法**。
*   **数据结构**：与类对象**完全相同**，也是 `struct objc_class`。

*   **结构详解**：
    *   **`isa` 指针**：
        *   元类的 `isa` 指向**根元类对象**（Root Meta-Class）。在大多数情况下，所有类的元类最终都指向 `NSObject` 的元类。
        *   **特例**：根元类（`NSObject` 的元类）的 `isa` 指针**指向它自己**，形成一个闭环。
    *   **`superclass` 指针**：
        *   指向其父类的**元类对象**。例如，`Person` 的元类的 `superclass` 指向 `NSObject` 的元类。
        *   这保证了类方法的继承链。如果在 `Person` 的元类中找不到某个类方法，就会顺着 `superclass` 链去 `NSObject` 的元类里找。
    *   **`bits` (方法列表)**：
        *   元类的方法列表里存储的是所有的**类方法**。

### 总结与对比

| 对象类型     | 数据结构             | `isa` 指针指向                | `superclass` 指针指向 | 主要存储内容             |
| :----------- | :------------------- | :---------------------------- | :-------------------- | :----------------------- |
| **实例对象** | `struct objc_object` | **类对象**                    | (无)                  | 实例变量的值             |
| **类对象**   | `struct objc_class`  | **元类对象**                  | **父类的类对象**      | **实例方法**、属性、协议 |
| **元类对象** | `struct objc_class`  | **根元类对象** (最终指向自己) | **父类的元类对象**    | **类方法**               |

**消息发送流程回顾：**

1.  `[personInstance sayHello]` (实例方法)
    *   `personInstance` -> `isa` -> `Person(Class)` -> 在方法列表中查找 `sayHello`。
    *   找不到？ -> `superclass` -> `NSObject(Class)` -> 在方法列表中查找 `sayHello`。

2.  `[Person aClassMethod]` (类方法)
    *   `Person(Class)` -> `isa` -> `Person(Meta-Class)` -> 在方法列表中查找 `aClassMethod`。
    *   找不到？ -> `superclass` -> `NSObject(Meta-Class)` -> 在方法列表中查找 `aClassMethod`。

理解了这三者的数据结构和它们之间通过 `isa` 与 `superclass` 构成的双重关系链，你就掌握了 Objective-C 对象模型的精髓。

## UIButton 的父类是什么？UILabel 的父类又是什么

这是一个非常基础且重要的 iOS 开发问题！

### UIButton 的父类

`UIButton` 的直接父类是 **`UIControl`**。

*   **为什么是 `UIControl`？**
    *   `UIControl` 是所有**可交互**的 UI 组件的基类，比如按钮、滑块、开关等。
    *   它为子类提供了核心的交互能力，最主要的就是**目标-动作机制 (Target-Action)**，让你能够方便地处理用户的触摸事件（比如 `touchUpInside`）。
    *   它还负责管理控件的各种状态（正常 `normal`、高亮 `highlighted`、选中 `selected`、禁用 `disabled` 等）。
    *   因为按钮是一个需要响应用户点击并有多种状态的控件，所以它继承自 `UIControl` 是非常合理的。

*   **完整的继承链**：
    `NSObject` -> `UIResponder` -> `UIView` -> `UIControl` -> `UIButton`

---

### UILabel 的父类

`UILabel` 的直接父类是 **`UIView`**。

*   **为什么是 `UIView`？**
    *   `UILabel` 的核心职责是**展示文本内容**，它本身默认不是一个可交互的控件。
    *   它只需要 `UIView` 提供的基本能力，比如在屏幕上绘制自己、管理自身的尺寸和位置（frame/bounds）、处理层级关系等。
    *   因为它不需要 `UIControl` 提供的目标-动作和状态管理等复杂的交互功能，所以它直接继承自 `UIView` 即可。

*   **完整的继承链**：
    `NSObject` -> `UIResponder` -> `UIView` -> `UILabel`

## `PerformSelector:afterDelay:`这个方法在子线程中是否起作用？为什么？怎么解决？

这是一个非常经典且重要的多线程问题！它能很好地检验开发者对 `NSRunLoop` 的理解。

### 核心答案

**`performSelector:afterDelay:` 在子线程中**通常不会起作用**，或者说**不会像你预期的那样工作**。

---

### 为什么不起作用？(The "Why")

根本原因在于：**`performSelector:afterDelay:` 的实现依赖于 `NSRunLoop` 和 `NSTimer`。**

1.  **工作机制**：当你调用 `[self performSelector:@selector(doSomething) withObject:nil afterDelay:2.0];` 时，系统实际上会创建一个 `NSTimer`（定时器），并将它添加到**当前线程**的 `NSRunLoop` 中。

2.  **`NSRunLoop` 是什么？**
    *   你可以把 `NSRunLoop` 想象成一个线程的**“事件管家”**。它的工作就是一个循环：**“接收事件 -> 处理事件 -> 睡眠 -> 接收事件...”**。
    *   定时器 (`NSTimer`) 就是一种需要 `NSRunLoop` 来触发的事件。管家会在每次循环时检查他的待办列表，看看有没有到期的定时器需要触发。

3.  **主线程 vs. 子线程**：
    *   **主线程**：应用程序启动时，系统会自动为主线程创建一个 `NSRunLoop`，并且**让它一直处于运行状态**。这就是为什么主线程能响应用户点击、UI更新和定时器等各种事件。它的“管家”一直在岗工作。
    *   **子线程**：当你创建一个子线程（无论是通过 `NSThread` 还是 GCD 的 `DispatchQueue.global()`）时，系统**默认不会为它启动 `NSRunLoop`**。子线程的“管家”是默认**“休眠”**的。子线程通常被设计为执行一个特定的、有限的任务，**任务执行完后线程就会退出**。

4.  **问题所在**：
    *   当你在一个子线程中调用 `performSelector:afterDelay:` 时，定时器被成功地添加到了这个子线程的 `NSRunLoop` 的“待办列表”里。
    *   但是，因为这个子线程的“管家”（RunLoop）**没有被启动**，它根本就不会去检查这个列表。
    *   更糟糕的是，这个子线程在执行完它的同步代码后，通常就会**立即退出并被销毁**。那个被添加进去的定时器也就随之石沉大海，永远没有机会被触发。

---

### 怎么解决？(The "How")

解决方案的核心思想是：要么**让子线程的 RunLoop 跑起来**，要么**换一种不依赖当前线程 RunLoop 的方式**。

#### 方案一：使用 GCD 的 `asyncAfter` (最佳、最推荐的现代方案)

这是最简单、最安全、最符合现代并发编程思想的方式。它不依赖于任何线程的 RunLoop。

**逻辑**：让 GCD 这个强大的调度系统来负责“延迟”这件事，然后在指定的时间后，将你的任务派发到你想要的队列（线程）上执行。

**Objective-C:**

```objc
// 延迟2秒后，在后台线程执行
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"在后台线程延迟执行 - %@", [NSThread currentThread]);
    [self doSomething];
});

// 如果延迟后需要在主线程更新UI
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    NSLog(@"在主线程延迟执行 - %@", [NSThread currentThread]);
    // 更UI...
});
```

#### 方案二：手动开启子线程的 RunLoop (不推荐，除非你有特殊需求)

这种方法能直接解决 `performSelector:afterDelay:` 失效的问题，但通常会导致**线程无法正常退出**，需要你手动管理线程的生命周期，非常麻烦。

**只适用于你需要创建一个常驻后台线程来持续处理事件的场景。**

**Objective-C:**

```objc
- (void)startBackgroundThread {
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(runOnBackgroundThread) object:nil];
    [thread start];
}

- (void)runOnBackgroundThread {
    NSLog(@"子线程启动，准备执行延迟任务");
    
    // 关键：在这里调用，因为RunLoop已经准备好了
    [self performSelector:@selector(doSomething) withObject:nil afterDelay:2.0];
    
    // 关键：手动启动当前线程的RunLoop
    // 这行代码会阻塞线程，让它进入事件循环，直到你手动停止它
    [[NSRunLoop currentRunLoop] run];
    
    // 注意：这行代码将永远不会被执行，除非RunLoop被外部停止
    NSLog(@"子线程即将退出");
}

- (void)doSomething {
    NSLog(@"延迟任务在子线程执行了！- %@", [NSThread currentThread]);
    // 如果需要，可以在这里停止RunLoop以退出线程
    // CFRunLoopStop(CFRunLoopGetCurrent());
}
```

## 讲一下obeserver

好的，我们来深入讲解一下 `NSRunLoop` 中一个非常强大但也相对底层的组件：**Observer (观察者)**。

### 一、核心思想与比喻

在 `RunLoop` 的整个工作流程中，它会经历多个不同的阶段（比如“将要处理定时器”、“将要休眠”、“刚被唤醒”等）。

**RunLoop Observer 就像一个安插在 RunLoop 内部的“秘密观察员”或“事件记者”。**

这个观察员的职责**不是**处理具体的任务（不像 `Source` 或 `Timer`），而是**在 RunLoop 的状态发生变化时，能够得到通知**。它允许你在 `RunLoop` 生命周期的特定时刻，插入你自己的代码逻辑。

---

### 二、Observer 关注的“关键时刻” (Activities)

Observer 并不是时刻都在工作，它只对自己感兴趣的“关键时刻”（`CFRunLoopActivity`）进行响应。这些时刻主要包括：

| Activity 枚举 (Core Foundation) | 含义               | 通俗解释                                                     |
| :------------------------------ | :----------------- | :----------------------------------------------------------- |
| `kCFRunLoopEntry`               | 即将进入 RunLoop   | “管家”刚上班，准备开始一天的工作循环。                       |
| `kCFRunLoopBeforeTimers`        | 即将处理 Timers    | “管家”准备看看日程表上有没有到期的定时器任务。               |
| `kCFRunLoopBeforeSources`       | 即将处理 Sources   | “管家”准备处理一下收到的信件（Source0）或端口消息（Source1）。 |
| `kCFRunLoopBeforeWaiting`       | 即将进入休眠状态   | “管-家”忙完一圈了，没新任务，准备打个盹（`mach_msg` 内核调用），等新任务来唤醒他。 |
| `kCFRunLoopAfterWaiting`        | 刚从休眠状态中唤醒 | “管家”被新任务（比如用户触摸屏幕）叫醒了，准备开始新一轮工作。 |
| `kCFRunLoopExit`                | 即将退出 RunLoop   | “管家”收到明确的下班指令，准备结束工作循环。                 |
| `kCFRunLoopAllActivities`       | 监听所有上述状态   | “超级观察员”，对所有状态变化都感兴趣。                       |

---

### 三、如何创建和使用 Observer (Objective-C)

在 Objective-C 中，我们通常不直接使用 `NSRunLoop` 的 API 来创建 Observer，而是使用其底层的 **Core Foundation** 框架中的 `CFRunLoopObserverRef`。

整个过程分为三步：

1.  **创建一个 Observer 对象**：定义你要监听哪些状态，以及状态发生变化时要执行什么操作（通常是一个 block）。
2.  **将 Observer 添加到 RunLoop 中**：指定要将这个观察员安插到哪个 `RunLoop` 的哪个 `Mode` 下。
3.  **(可选) 移除 Observer**：在不需要时将其移除。

#### 代码示例：

下面的代码演示了如何创建一个 Observer 来监控主线程 RunLoop 的状态变化。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setupRunLoopObserver];
}

- (void)setupRunLoopObserver {
    // 1. 定义 Observer
    // 使用 CFRunLoopObserverCreateWithHandler 可以方便地使用 block
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(
        kCFAllocatorDefault,      // 分配器，使用默认的即可
        kCFRunLoopAllActivities,  // 要监听的 Activity，这里我们监听所有状态
        YES,                      // 是否重复监听
        0,                        // 优先级，0 为最高
        ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
            // 3. 状态发生变化时，这个 block 就会被回调
            switch (activity) {
                case kCFRunLoopEntry:
                    NSLog(@"RunLoop 即将进入");
                    break;
                case kCFRunLoopBeforeTimers:
                    NSLog(@"RunLoop 即将处理 Timers");
                    break;
                case kCFRunLoopBeforeSources:
                    NSLog(@"RunLoop 即将处理 Sources");
                    break;
                case kCFRunLoopBeforeWaiting:
                    NSLog(@"RunLoop 即将休眠");
                    break;
                case kCFRunLoopAfterWaiting:
                    NSLog(@"RunLoop 刚被唤醒");
                    break;
                case kCFRunLoopExit:
                    NSLog(@"RunLoop 即将退出");
                    break;
                default:
                    break;
            }
        });

    // 2. 将 Observer 添加到当前的 RunLoop 中，监听其 Default 模式
    // 注意：主线程的 RunLoop 默认就是 kCFRunLoopDefaultMode
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
    
    // 因为 CF 对象 ARC 不会自动管理内存，所以需要手动释放
    // (如果 observer 是一个局部变量，在函数结束时就释放了，会导致观察失效，
    //  所以通常需要将其作为类的成员变量持有，或者不释放让其常驻)
    // CFRelease(observer); 
    // 在这个例子里，我们希望它一直有效，所以先不释放。
    // 更好的做法是将其保存为成员变量，在 dealloc 时释放。
}
```

当你运行这段代码，然后与界面进行交互（比如滚动 `UIScrollView`），你会在控制台看到 `RunLoop` 状态的实时打印，因为滚动会持续唤醒 `RunLoop`。

---

### 四、Observer 的经典应用场景

你可能会问，知道这些状态有什么用？Observer 在苹果的框架和一些第三方库中有非常重要的应用。

#### 1. 自动释放池 (`@autoreleasepool`)

这是 Observer **最经典、最重要**的应用！

`@autoreleasepool` 能够自动释放对象的原理，就完全依赖于 `RunLoop` 的 Observer。系统会在主线程的 `RunLoop` 中注册两个 Observer：

*   **第一个 Observer (优先级最高)**：
    *   **监听状态**：`kCFRunLoopEntry` (即将进入循环)。
    *   **执行操作**：调用 `objc_autoreleasePoolPush()`，**创建一个新的自动释放池**。

*   **第二个 Observer (优先级最低)**：
    *   **监听状态**：`kCFRunLoopBeforeWaiting` (即将休眠) 和 `kCFRunLoopExit` (即将退出)。
    *   **执行操作**：调用 `objc_autoreleasePoolPop()`，**释放旧的池子**，然后 `objc_autoreleasePoolPush()` **创建一个新的池子**。

**整个流程**：
一次事件循环开始 -> (Observer 1) 创建新池子 -> 处理各种事件（代码中产生的 autoreleased 对象会加入这个池子）-> 准备休眠 -> (Observer 2) 销毁池子（所有 autoreleased 对象被释放），并创建下一个循环用的新池子 -> 休眠。

#### 2. UI 刷新与布局

系统内部也利用 Observer 来处理 UI 的更新。当一个事件（如触摸）处理完成后，在 `RunLoop` 即将进入休眠（`BeforeWaiting`）之前，会集中处理所有被标记为“需要重绘”的视图 (`setNeedsDisplay`) 和“需要重新布局”的视图 (`setNeedsLayout`)。这样做可以把一轮事件循环中的所有 UI 变更合并到一次绘制中，提高了性能。

## 正常的字典有哪些方法

当然，我们来系统地梳理一下苹果官方 `NSDictionary` (以及 `NSMutableDictionary`) 类提供的、用于根据 `key` 来获取 `value` 的标准方法。

了解这些原生方法的行为，能让你更深刻地理解为什么项目中需要封装 `blp_...` 这样的安全方法。

---

### `NSDictionary` (不可变字典) 的核心取值方法

#### 1. `objectForKey:` (最常用)

这是最经典、最广为人知的取值方法。

*   **方法签名**: `- (nullable id)objectForKey:(id)aKey;`
*   **功能**: 返回与 `aKey` 相关联的值。
*   **行为**:
    *   如果找到了 `key`，返回对应的 `value` 对象。
    *   如果**没有找到** `key`，返回 `nil`。
    *   如果传入的 `aKey` 是 `nil`，会**直接抛出异常 (NSInvalidArgumentException)，导致程序崩溃**。
*   **示例**:
    ```objectivec
    NSDictionary *dict = @{@"name": @"John", @"age": @30};
    NSString *name = [dict objectForKey:@"name"]; // -> @"John"
    NSNumber *age = [dict objectForKey:@"age"];   // -> @30
    NSString *city = [dict objectForKey:@"city"]; // -> nil
    // [dict objectForKey:nil]; // -> CRASH!
    ```

#### 2. 下标语法 (Modern Objective-C Syntax)

这是现代 Objective-C 中更简洁、更推荐的写法，其底层实现就是调用 `objectForKey:`。

*   **语法**: `dictionary[aKey]`
*   **功能**: 与 `objectForKey:` 完全相同。
*   **行为**: 与 `objectForKey:` 完全相同，包括传入 `nil` 的 `key` 会导致**崩溃**。
*   **示例**:
    ```objectivec
    NSDictionary *dict = @{@"name": @"John"};
    NSString *name = dict[@"name"]; // -> @"John"
    NSString *city = dict[@"city"]; // -> nil
    // id key = nil;
    // NSString *value = dict[key]; // -> CRASH!
    ```

#### 3. `valueForKey:` (KVC - Key-Value Coding)

这个方法来自 `NSKeyValueCoding` 协议，`NSDictionary` 实现了它。它的功能比 `objectForKey:` 更强大，但也更复杂。

*   **方法签名**: `- (nullable id)valueForKey:(NSString *)key;`
*   **功能**: 它的行为和 `objectForKey:` 非常相似，也是根据 `key` 返回 `value`。
*   **主要区别**: `valueForKey:` 是 KVC 体系的一部分，它有一些特殊的行为（比如处理 `@avg`, `@sum` 等集合运算符），但在简单的字典取值场景下，可以认为它和 `objectForKey:` 效果一样。
*   **行为**:
    *   如果 `key` 是 `nil`，同样会**崩溃**。
*   **建议**: 在只是想从字典里取值时，**优先使用 `objectForKey:` 或下标语法**，因为它们更直接、性能也可能稍好。只有在需要利用 KVC 的高级特性时才使用 `valueForKey:`。
*   **示例**:
    ```objectivec
    NSDictionary *dict = @{@"name": @"John"};
    NSString *name = [dict valueForKey:@"name"]; // -> @"John"
    ```



## ios解析json的方法

当然，通过您之前发给我的多段代码，我可以非常肯定地指出您项目中解析 JSON 的**核心方法**。

这个方法并不是一个单一的函数调用，而是一套**组合拳**，体现了非常好的封装和安全设计。

---

### 核心方法：`NSJSONSerialization` + 自定义 `NSDictionary` Category

您项目中解析 JSON 的流程分为两步：

#### 第一步：将原始二进制数据 (NSData) 转换为字典 (NSDictionary)

这一步使用的是苹果的**原生框架 `Foundation`** 中的 `NSJSONSerialization` 类。在 `BLPCommonWebViewEventManager.m` 的 `handleLiveScheme:...` 方法中有明确的体现：

```objectivec
- (void)handleLiveScheme:(NSURL *)url ... {
    // ...
    if (params && params.length > 0) {
        // 1. 将 H5 传来的 params 字符串转换为 NSData
        NSData *data = [params dataUsingEncoding:NSUTF8StringEncoding];
        
        // 2. 【核心】使用 NSJSONSerialization 将 NSData 解析成 NSDictionary
        NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:data 
                                                            options:0 
                                                              error:nil];
        [paramsDic addEntriesFromDictionary:dic];
    }
    // ...
}
```*   **`[NSJSONSerialization JSONObjectWithData:options:error:]`** 就是最底层的、将符合 JSON 格式的 `NSData` 转换成 `NSDictionary` 或 `NSArray` 的标准方法。

#### 第二步：从字典 (NSDictionary) 中安全地取值并赋值给模型 (Model)

在将 JSON 数据转换成 `NSDictionary` 之后，您的项目并没有直接使用 `dictionary[@"key"]` 或 `[dictionary objectForKey:@"key"]` 这种原始的方式来取值。

而是使用了**一套封装在 `NSDictionary` 分类 (Category) 中的自定义安全取值方法**。这一点在您提供的所有业务插件代码中都得到了反复印证：

*   **在 `BDPLiveDataCenterPlugin.m` 中**：
    ```objectivec
    NSString *actType = [params blp_stringValueForKey:kLiveDataCenterActType];
    NSArray *actTypeArr = [params blp_arrayValueForKey:kLiveDataCenterActType];
```
*   **在 `BDPLiveDragonBoatFestivalPlugin.m` 中**：
    ```objectivec
    double addPrice = [[params blp_numberValueForKey:@"price"] doubleValue] / 100;
    NSString *scoreString = [params blp_stringValueForKey:@"score_unit"] ?: @"安康值";
    ```
*   **在您写的 `BDPLiveCommonActivityAlertInfo.m` 中**：
    
    ```objectivec
    self.bgImg  = [dict blp_stringValueForKey:@"bgImg"];
    NSArray *tipsArray = [dict blp_arrayValueForKey:@"tips"];
    ```

### 1. `blp_stringValueForKey:` 的安全之旅

我们来看最常用的一个：当你需要从字典里取一个**字符串**时。

**调用示例**：
```objectivec
// JSON: {"code": 200, "message": "success", "userId": "user_123"}
NSDictionary *response = ...;
NSString *message = [response blp_stringValueForKey:@"message"]; // 期望得到 @"success"
NSString *codeStr = [response blp_stringValueForKey:@"code"];    // 期望得到 @"200"
NSString *userId = [response blp_stringValueForKey:@"userId"]; // 期望得到 @"user_123"
NSString *data = [response blp_stringValueForKey:@"data"];    // 期望得到 nil
```

**内部实现详解**：
```objectivec
- (NSString *)blp_stringValueForKey:(NSString *)key numberStringInsensitive:(BOOL)insensitive {
    // 第1关：基础安全检查
    // [self blp_objectForKey:key] 会先检查 key 是否为 nil 或空字符串。
    // 如果是，直接返回 nil，避免 dictionary 因为 key 为 nil 而崩溃。
    id objValue = [self blp_objectForKey:key];
    
    // 第2关：理想情况 - 类型完全匹配
    // BLPIsString(objValue) 检查 objValue 是不是一个 NSString。
    // 如果是（比如取 "message" 或 "userId"），就直接返回这个值。这是最高效的路径。
    if (BLPIsString(objValue)) {
        return objValue;
    } 
    // 第3关：容错处理 - 智能类型转换
    // 走到这里，说明类型不匹配。代码会进行一次“补救”。
    // BLPIsNumber(objValue) 检查 objValue 是不是一个 NSNumber。
    // insensitive 参数通常为 YES，表示允许这种转换。
    // 如果是（比如取 "code"，其值是 NSNumber 类型的 200），
    // 就会执行 [(NSNumber *)objValue stringValue]，将其强制转换为字符串 @"200" 并返回。
    // 这是为了兼容后端有时会将数字类型的数据返回的情况。
    else if (BLPIsNumber(objValue) && insensitive) {
        return [(NSNumber *)objValue stringValue];
    }
    
    // 第4关：最终防线 - 失败返回 nil
    // 如果代码走到了这里，说明 objValue 既不是 NSString，也不是 NSNumber
    // (比如取 "data"，它的值可能是个 NSDictionary，或者 key 根本不存在，objValue 是 nil)。
    // 在这种情况下，方法坚决返回 nil，确保调用方绝对不会收到一个错误类型的对象。
    return nil;
}
```

**安全保障总结** (`blp_stringValueForKey:`)：
*   **防崩溃**：防止因 `key` 为 `nil` 或值类型错误（比如你尝试对一个 `NSDictionary` 调用 `length` 方法）导致的崩溃。
*   **类型保证**：这个方法向你郑重承诺——“我返回的要么是 `NSString`，要么是 `nil`，绝不可能是别的类型”。
*   **兼容性**：能够自动处理服务器返回 `NSNumber` 的情况，增强了对不规范数据的适应能力。

---

### 2. `blp_numberValueForKey:` 的安全之旅

这个方法与上面那个是对称的，当你需要取一个**数字**时使用。

**调用示例**：
```objectivec
// JSON: {"code": 200, "price": "99.9", "status": "success"}
NSDictionary *response = ...;
NSNumber *code = [response blp_numberValueForKey:@"code"];       // 期望得到 NSNumber 类型的 200
NSNumber *price = [response blp_numberValueForKey:@"price"];     // 期望得到 NSNumber 类型的 99.9
NSNumber *status = [response blp_numberValueForKey:@"status"];   // 期望得到 nil
```

**内部实现详解**：
```objectivec
- (NSNumber *)blp_numberValueForKey:(NSString *)key numberStringInsensitive:(BOOL)insensitive {
    // 第1关：基础安全检查 (同上)
    id objValue = [self blp_objectForKey:key];
    
    // 第2关：理想情况 - 类型完全匹配
    // BLPIsNumber(objValue) 检查 objValue 是不是一个 NSNumber。
    // 如果是（比如取 "code"），直接返回。
    if (BLPIsNumber(objValue)) {
        return objValue;
    } 
    // 第3关：容错处理 - 智能类型转换
    // 走到这里，说明类型不匹配。
    // BLPIsString(objValue) 检查 objValue 是不是一个 NSString。
    // 如果是（比如取 "price"，其值是 NSString 类型的 @"99.9"），
    // 就会调用 [(NSString *)objValue blp_numberValue] 这个 NSString 的分类方法。
    // blp_numberValue 内部会使用 NSNumberFormatter 尝试将字符串转换为 NSNumber。
    // 这是为了兼容后端有时会将数字用字符串形式返回的情况。
    else if (BLPIsString(objValue) && insensitive) {
        return [(NSString *)objValue blp_numberValue];
    }
    
    // 第4关：最终防线 - 失败返回 nil (同上)
    // 如果 objValue 既不是 NSNumber，也不是 NSString（比如取 "status"，值是 @"success"），
    // 或者 key 不存在，最终返回 nil。
    return nil;
}
```

**安全保障总结** (`blp_numberValueForKey:`)：
*   **防崩溃**：同上。
*   **类型保证**：承诺返回的要么是 `NSNumber`，要么是 `nil`。
*   **兼容性**：能够自动处理服务器返回**数字字符串**的情况，非常实用。

---

### `blp_arrayValueForKey:` 和 `blp_dictionaryValueForKey:`

这两个方法的实现就简单直接得多了，因为 `Array` 和 `Dictionary` 之间通常不存在合理的“自动转换”场景。

```objectivec
- (NSArray *)blp_arrayValueForKey:(NSString *)key {
    // 第1关：基础安全检查
    id objValue = [self blp_objectForKey:key];
    
    // 第2关：严格的类型检查
    // BLPIsArray(objValue) 只检查 objValue 是不是一个 NSArray。
    // 如果是，就返回；如果不是（任何其他类型或 nil），都直接返回 nil。
    if (BLPIsArray(objValue)) {
        return objValue;
    }
    
    return nil;
}
```
它们的安全策略主要就是**类型守门**：**“是你要的类型，我就给你；不是，就返回 `nil`，绝不含糊。”**

## 为什么ios一定要在主线程操作ui

好的，我们完全抛开比喻，从计算机系统和框架设计的层面来精确解释这个问题。

将UI操作限制在主线程，其根本原因在于**UIKit框架的非线程安全设计**以及**主线程RunLoop在UI渲染和事件处理中所扮演的独一无二的、中心化的角色**。

这可以从以下三个技术层面来理解：

---

### 1. UIKit的非线程安全 (Not Thread-Safe)

这是最核心、最底层的约束。

*   **内部状态和数据结构：** 一个UI对象（如`UIView`或`UIViewController`）内部包含了大量的状态和数据结构，例如它的`frame`、`bounds`、`layer`树、子视图数组、约束等。这些内部数据被设计为只能由一个执行上下文（即一个线程）来访问和修改。
*   **并发修改的风险：** 如果允许两个线程同时操作同一个`UIView`，就会发生竞态条件（Race Condition）。例如：
    *   **线程A** 正在计算并设置视图的新位置：`view.frame = newFrame;`。这个操作可能涉及多个步骤，比如修改`origin`和`size`的内部变量。
    *   **线程B** 同时在修改视图的背景色：`view.backgroundColor = [UIColor redColor];`。
    *   如果操作系统在这两个操作之间切换线程，可能会导致`UIView`的内部数据处于一个不一致的、中间的状态。比如，`frame`的`origin`被更新了，但`size`还没有，此时线程B的修改可能会基于这个不完整的数据进行，最终导致视图损坏、渲染异常甚至程序崩溃。
*   **为什么不加锁？** 有人会问，为什么苹果不给所有UI对象的操作都加上锁（Lock）来让它变得线程安全？
    *   **性能开销巨大：** UI操作非常频繁。如果每次修改一个`UILabel`的文字或一个`UIView`的位置都需要进行加锁和解锁，这种同步开销会严重拖慢UI的响应速度，导致界面卡顿。
    *   **死锁风险：** 在一个复杂的视图层级中，加锁很容易引发死锁（Deadlock）。比如线程A锁住了父视图，想去操作子视图；而线程B同时锁住了子视图，想去操作父视图，两个线程就会互相等待对方释放锁，导致程序完全卡死。

**结论：** 出于性能和复杂度的考量，UIKit从设计之初就选择了“非线程安全”这条路。作为补偿，它提供了一个严格的规则：**所有对UIKit对象的访问都必须在同一个、指定的线程上进行串行化处理，这个线程就是主线程。**

---

### 2. 主线程RunLoop：UI更新的唯一调度者

主线程之所以被选中，是因为它运行着一个特殊的`NSRunLoop`（或底层的`CFRunLoop`），这个RunLoop是整个iOS应用UI机制的核心驱动力。

RunLoop的核心作用是管理和调度事件，并与系统的渲染时机进行同步。一个RunLoop周期内，它会执行一个精确的序列：

1.  **处理输入源 (Input Sources):** 首先，它处理来自触摸屏、网络端口等硬件或内核的异步事件。当用户触摸屏幕时，I/O Kit会捕捉到事件，通过Mach Port传递给应用的RunLoop，RunLoop被唤醒并处理这个触摸事件，最终分发到对应的`UIView`。

2.  **处理定时器 (Timers):** 接着，处理所有到期的`NSTimer`事件。

3.  **执行派发的任务:** 执行通过`dispatch_async(dispatch_get_main_queue(), ...)`或`performSelectorOnMainThread:`等方式从其他线程发送过来的代码块。

4.  **调用RunLoop观察者 (Observers):** 这是与UI更新最关键的一步。系统在主线程的RunLoop上注册了多个观察者，它们监听着RunLoop的不同状态。其中一个关键的观察者监听的是`kCFRunLoopBeforeWaiting`（即RunLoop即将进入休眠状态前）的时刻。

5.  **系统级的UI更新事务：**
    *   当这个观察者被触发时，它会启动一个UI更新的“事务”（Transaction）。
    *   这个事务会检查整个应用中所有被标记为“脏”（dirty）的视图。当你调用`setNeedsLayout`或`setNeedsDisplay`时，你并没有立即触发重绘，只是给视图打上了一个“脏”标记。
    *   在这个时间点，系统会依次执行：
        *   **布局 (Layout):** 调用所有被标记需要重新布局的视图的`layoutSubviews`方法。
        *   **绘制 (Display/Draw):** 为所有被标记需要重绘的视图调用`drawRect:`方法，将其内容绘制到一块位图（bitmap）上。
        *   **提交 (Commit):** 将所有视图层级（CALayer tree）的结构和内容变化，打包提交给一个独立的进程——**Render Server**。

6.  **Render Server渲染:** Render Server进程负责将所有应用提交的图层内容进行合成，并根据显示器的垂直同步信号（VSync），在下一次屏幕刷新时将最终的画面显示到屏幕上。

**结论：** 整个UI的布局、绘制、提交过程，都被紧密地集成在主线程RunLoop的特定阶段。如果一个后台线程直接修改UI（比如直接调用`drawRect:`或修改一个CALayer的属性），它就完全绕过了这个由RunLoop驱动的、与显示刷新同步的标准化流程。这会导致：

*   **时机错误：** 更新可能发生在两帧之间，导致画面撕裂。
*   **数据不同步：** Render Server可能正在读取图层树的数据，而后台线程同时在修改它，导致渲染出损坏或过时的内容。

### 总结

将UI操作限制在主线程，是iOS（及其他主流GUI系统）为了**从根本上解决并发带来的复杂性和不确定性**而做出的关键设计决策。它通过一个**串行化的RunLoop模型**，确保了UI状态的**一致性、可预测性和稳定性**，虽然对开发者提出了一定的规范要求，但极大地简化了UI编程的难度，并为流畅的用户体验提供了坚实的基础。

## 在init方法里面，设置背景颜色，会生效吗 会生效。为什么会？

是的，你的判断完全正确：**在 `init` 方法里设置背景颜色，会生效。**

这是一个非常好的问题，因为它触及了 `UIView` 对象生命周ogcyle和 iOS 渲染机制的核心。理解“为什么会生效”，能让你对整个视图系统有更深刻的认识。

我们可以把这个过程分解为三个独立的阶段：**1. 状态设定**，**2. 加入层级**，和 **3. 最终渲染**。

---

### 1. 状态设定 (State Configuration) - `init` 方法的职责

当你调用一个 `UIView` 的 `init`、`initWithFrame:` 或 `initWithCoder:` 方法时，你正在做的仅仅是**创建一个 Objective-C 对象并配置它的初始状态**。

*   **内存分配**：系统为这个 `UIView` 实例分配一块内存。
*   **属性赋值**：你在这块内存中为对象的各个属性（实例变量）赋值。

所以，当你在 `init` 方法里写下这行代码：
```objc
- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        // 关键：你正在做的事情是“状态设定”
        self.backgroundColor = [UIColor redColor];
    }
    return self;
}
```
你并没有在屏幕上画任何东西。你只是在这个 `UIView` 对象的内存中，将 `_backgroundColor` 这个实例变量的值设置为了 `[UIColor redColor]`。

可以把它想象成你正在填写一张“视图规格表”：
*   Frame: `{0, 0, 100, 100}`
*   **BackgroundColor: Red**
*   Alpha: 1.0
*   ...

此时，这个 `UIView` 对象只是一个存在于内存中的、配置好的数据结构。它知道自己“应该”是红色的，但它还没有机会展示自己。

---

### 2. 加入层级 (Joining the Hierarchy) - `addSubview:` 的职责

一个孤立的 `UIView` 对象是无法被渲染的。它必须被添加到一个可见的视图层级中，这个层级的根节点通常是 `UIWindow`。

这一步通常在 `UIViewController` 的 `viewDidLoad` 或其他生命周期方法中完成：
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    MyCustomView *customView = [[MyCustomView alloc] initWithFrame:CGRectMake(50, 50, 100, 100)];
    
    // 关键：将配置好的视图加入到视图层级中
    // 从这一刻起，渲染系统才知道有这么一个视图需要被绘制
    [self.view addSubview:customView]; 
}
```
当 `addSubview:` 被调用时，`customView` 就成为了 `self.view` 的一个子视图。现在，它正式成为了屏幕渲染管线的一部分。系统知道在下一次屏幕刷新时，需要考虑绘制这个 `customView` 了。

---

### 3. 最终渲染 (The Actual Rendering) - Run Loop 和 Render Server 的职责

iOS 的界面并不会在你调用 `addSubview:` 的瞬间就立刻重绘。它会等待当前的**主运行循环（Run Loop）** 结束，然后在一个统一的时间点处理所有 UI 的更新。

这个过程大致是：
1.  **标记为“脏”**：当一个视图被添加到层级中，或者它的任何影响外观的属性（如 `frame`, `backgroundColor`）被改变时，它会被系统标记为“需要重绘”（dirty）。
2.  **提交图层树**：在 Run Loop 的末尾，你的 App 会将所有视图的图层（`CALayer`）信息打包成一个“图层树”，然后通过 IPC (进程间通信) 发送给一个独立的进程——**Render Server**。
3.  **Render Server 工作**：Render Server 接收到这个图层树后，开始工作。它会遍历树中的每一个图层，读取它们的**状态**（就是你在 `init` 中设置的那些属性）。
4.  **执行绘制**：当它处理到 `customView` 的图层时，它会读取规格表：
    *   “哦，这个图层的 `frame` 是 `{50, 50, 100, 100}`。”
    *   “它的 `backgroundColor` 是**红色**。”
    *   “...其他属性...”
5.  **调用 GPU**：Render Server 根据这些信息，组合成渲染指令，交给 GPU。GPU 最终在屏幕上对应的像素点上填充红色。

### 总结：为什么会生效？

**因为 `init` 方法成功地设置了对象的内部状态（State），而这个状态被后续的渲染流程所读取和使用。**

流程可以简化为：

1.  **`init` (对象内部)**：`self.backgroundColor = [UIColor redColor];`
    *   **结果**：`myView` 对象在内存中的 `_backgroundColor` 属性被设置为红色。**它记住了自己应该是红色的。**

2.  **`addSubview:` (加入层级)**：`[parentView addSubview:myView];`
    *   **结果**：`myView` 进入了“待渲染”的队列。

3.  **Run Loop Tick (系统渲染)**：Render Server 开始工作。
    *   **结果**：Render Server 查看 `myView` 的属性，发现了它“记住”的颜色是红色，于是命令 GPU 将其绘制成红色。

所以，`init` 方法是设置蓝图和规格的阶段，虽然没有立即产生视觉效果，但它为未来的视觉效果提供了必不可少的数据基础。

## NSThread，GCD，NSOperation相关的。开启一条线程的方法？线程可以取消吗

### 一、如何开启一条线程？

#### 1. `NSThread`

`NSThread` 是最轻量级的、面向对象的线程管理方式。它允许你直接创建和控制一个线程。

**方法一：创建并手动启动**
```objc
// 创建一个 NSThread 对象
NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(runTask:) object:@"Parameter"];
// 手动启动线程
[thread start];

...

- (void)runTask:(id)param {
    NSLog(@"Current Thread: %@, Parameter: %@", [NSThread currentThread], param);
    // 执行耗时操作...
}
```

**方法二：分离线程（创建后自动启动）**
这种方式创建的线程会在任务执行完毕后自动销毁，你无法再获取到这个线程对象。
```objc
[NSThread detachNewThreadSelector:@selector(runTask:) toTarget:self withObject:@"Parameter"];
```

**方法三：NSObject 的分类方法（隐式创建并启动）**
这是 `NSObject` 的一个分类，提供了一个非常便捷的方式来将任务放到后台线程执行。
```objc
[self performSelectorInBackground:@selector(runTask:) withObject:@"Parameter"];
```

#### 2. `GCD (Grand Central Dispatch)`

GCD 是基于任务和队列的，你**不直接创建线程**。你把任务（Block）扔给一个队列，GCD 会智能地从它管理的**线程池**中调配一个线程来执行你的任务。

**开启一个并发任务（最常用）**
```objc
// 获取一个全局并发队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

// 异步地将任务添加到队列中
dispatch_async(queue, ^{
    NSLog(@"Current Thread: %@", [NSThread currentThread]);
    // 执行耗时操作...
});
```

#### 3. `NSOperation` & `NSOperationQueue`

`NSOperation` 也是面向任务的，并且是对 GCD 的更高层封装。你同样不直接管理线程，而是将“操作”（`NSOperation`）添加到“操作队列”（`NSOperationQueue`）中。

**方法一：使用 `NSBlockOperation`**
```objc
// 创建一个操作队列（默认是并发的）
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// 创建一个 Block 操作
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"Current Thread: %@", [NSThread currentThread]);
    // 执行耗时操作...
}];

// 将操作添加到队列中，队列会自动调度线程执行
[queue addOperation:operation];
```

**方法三：使用 `addOperationWithBlock:` 快捷方式**
```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperationWithBlock:^{
    NSLog(@"Current Thread: %@", [NSThread currentThread]);
    // 执行耗时操作...
}];
```

---

### 二、线程（任务）可以取消吗？

这是一个非常关键的区别点。

#### 1. `NSThread` - **可以，但需要手动配合**

*   **如何取消**：你可以对 `NSThread` 对象调用 `[thread cancel];`。
*   **原理**：这个 `cancel` 方法并**不会**粗暴地终止线程。它只是在线程内部设置了一个 `isCancelled` 的标志位为 `YES`。
*   **你的责任**：你**必须**在你的耗时任务代码中（比如一个 `for` 循环或 `while` 循环内部）定期地、频繁地检查这个标志位。如果发现 `[NSThread currentThread].isCancelled` 为 `YES`，你就要**主动地、优雅地**退出任务，比如 `return;`。

```objc
- (void)runTask:(id)param {
    for (int i = 0; i < 10000; i++) {
        // 关键：定期检查取消状态
        if ([[NSThread currentThread] isCancelled]) {
            NSLog(@"Thread was cancelled. Exiting.");
            return; // 主动退出
        }
        
        // 执行一小部分工作...
        NSLog(@"Working... %d", i);
        [NSThread sleepForTimeInterval:0.1];
    }
}
```
**结论**：`NSThread` 的取消是一种**协作式**的机制，不是强制性的。

#### 2. `GCD` - **不可以**

*   **GCD 的设计哲学是“Fire and Forget”**（发后不理）。
*   一旦你把一个任务 Block `dispatch_async` 到一个队列中，你就**没有任何办法**从外部中途取消它。这个任务会一直执行，直到它自己完成。
*   **变通方法**：你可以模仿 `NSThread` 的方式，自己设置一个外部的 `BOOL` 标志位，然后在 Block 内部去检查这个标志位。但这并不是 GCD 原生支持的功能，需要自己管理状态同步问题（比如使用原子属性 `atomic`）。

```objc
// 这是一个不完美的模拟，需要注意线程安全
@property (atomic, assign) BOOL isTaskCancelled; 

- (void)startGcdTask {
    self.isTaskCancelled = NO;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for (int i = 0; i < 10000; i++) {
            if (self.isTaskCancelled) {
                NSLog(@"GCD task was cancelled. Exiting.");
                return;
            }
            // ...
        }
    });
}

- (void)cancelGcdTask {
    self.isTaskCancelled = YES;
}
```
**结论**：GCD **原生不支持**取消。

#### 3. `NSOperation` & `NSOperationQueue` - **可以，并且是原生支持的强大特性**

*   `NSOperation` 是这三者中对取消操作支持得**最好、最完善**的。
*   **如何取消**：
    *   取消单个操作：`[operation cancel];`
    *   取消队列中所有操作：`[queue cancelAllOperations];`
*   **原理**：
    *   和 `NSThread` 类似，调用 `cancel` 也是设置内部的 `isCancelled` 标志位为 `YES`。

```objc
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    for (int i = 0; i < 10000; i++) {
        // 关键：在 NSOperation 内部检查 isCancelled
        // 注意：这里没有 self，因为 operation 是局部变量，需要弱引用
        // 如果是在 NSOperation 子类中，直接用 self.isCancelled
        // 这里用 operation 本身来检查，但要注意 block 的循环引用
        // 更安全的做法是：
        // __weak NSBlockOperation *weakOp = operation;
        // if (weakOp.isCancelled) { ... }
        if (operation.isCancelled) {
            NSLog(@"Operation was cancelled. Exiting.");
            return;
        }
        // ...
    }
}];

[queue addOperation:operation];

// ... 在未来的某个时刻
[operation cancel];
```
**结论**：`NSOperation` **原生支持**强大的、协作式的取消机制，是其相对于 GCD 的核心优势之一。

### 总结

| 技术              | 如何开启线程/任务                           | 是否支持取消                              |
| :---------------- | :------------------------------------------ | :---------------------------------------- |
| **`NSThread`**    | 直接创建和启动线程对象。                    | **是** (协作式，需手动检查 `isCancelled`) |
| **`GCD`**         | 将任务 Block 添加到队列，由系统线程池管理。 | **否** (原生不支持)                       |
| **`NSOperation`** | 将操作对象添加到操作队列。                  | **是** (原生支持，协作式，功能强大)       |

## 知道iOS里面有哪些数据存储方法？什么时候该用哪些方法存储？



### 方案一：`NSUserDefaults` (或 Swift 中的 `UserDefaults`)

#### 1. 它是什么？

`NSUserDefaults` 是一个**轻量级的持久化键值存储**系统。你可以把它想象成 App 的一个专属“**便签本**”或“**设置面板**”。

*   **键值存储 (Key-Value)**：就像 `NSDictionary` 一样，你用一个唯一的字符串（Key）来存取一个值（Value）。
*   **持久化 (Persistent)**：你存入的数据，即使用户关闭甚至重启 App，下次打开时依然存在。
*   **轻量级 (Lightweight)**：它被设计用来存储少量、简单的数据。它的底层实现通常是一个小型的 Plist (属性列表) 文件，存放在 App 的沙盒目录里。

#### 2. 适合存储什么？(The Right Job)

`NSUserDefaults` 的设计目标非常明确：**存储用户的偏好设置和应用的简单状态**。

**典型用例**:

*   **用户设置**:
    *   `"isNightModeEnabled": true` (夜间模式是否开启 - `Bool`)
    *   `"fontSize": 16` (用户选择的字体大小 - `Int`)
    *   `"username": "john_doe"` (用户上次登录的账号名 - `String`)
    *   `"themeColor": "blue"` (用户选择的主题颜色 - `String`)

*   **应用状态**:
    *   `"hasShownOnboarding": true` (是否已经向用户展示过引导页 - `Bool`)
    *   `"appLaunchCount": 25` (应用启动了多少次 - `Int`)

**支持的数据类型**:
它只能存储基础的 Plist 类型，包括 `NSNumber` (包含 `Int`, `Float`, `Bool`), `NSString`, `NSDate`, `NSData`, `NSArray`, 和 `NSDictionary`。

#### 3. 如何使用？(The Code)

它的 API 非常简单直观。

```objc
// 获取单例对象
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

// --- 写/存数据 ---

// 存储一个 Bool 值
[defaults setBool:YES forKey:@"isNightModeEnabled"];

// 存储一个整型
[defaults setInteger:16 forKey:@"fontSize"];

// 存储一个字符串
[defaults setObject:@"john_doe" forKey:@"username"];


// --- 读/取数据 ---

// 读取 Bool 值 (如果 key 不存在，会返回默认值 NO)
BOOL isNightMode = [defaults boolForKey:@"isNightModeEnabled"];

// 读取整型 (如果 key 不存在，会返回默认值 0)
NSInteger fontSize = [defaults integerForKey:@"fontSize"];

// 读取对象 (如果 key 不存在，会返回 nil)
NSString *username = [defaults stringForKey:@"username"]; // 推荐使用 specific getter
// 或者通用 getter
// NSString *username = [defaults objectForKey:@"username"];


// --- 删除数据 ---
[defaults removeObjectForKey:@"username"];


// --- 立即同步 ---
// 通常你不需要手动调用它。系统会在合适的时机自动将数据写入磁盘。
// 只有在极少数情况下，比如你担心 App 即将崩溃需要立即保存时，才可能用到。
// [defaults synchronize]; // 在现代 iOS 中已不推荐主动调用
```

#### 4. 关键的“不适合”做什么？(The Wrong Job)

理解它的局限性至关重要，滥用 `NSUserDefaults` 是新手常见的错误。

*   **绝对不能存储敏感信息**：密码、API Token、密钥等。因为它是**明文存储**在文件里的，任何能接触到设备文件系统的人（比如越狱用户）都能轻易看到。
*   **不适合存储大量数据**：不要把从服务器请求来的成百上千条新闻列表、聊天记录等存进去。因为 `NSUserDefaults` 会在某个时间点将整个文件加载到内存，大数据量会严重影响性能和内存占用。
*   **它不是数据库**：它不支持查询、排序或条件过滤。如果你有“查找所有年龄大于 18 岁的用户”这类需求，`NSUserDefaults` 完全无能为力。

#### 总结

| 特性         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| **核心定位** | 应用的“偏好设置面板”                                         |
| **优点**     | **API 极其简单**，轻量，自动持久化                           |
| **缺点**     | **不安全**，只适合**少量、非敏感**数据                       |
| **黄金法则** | 当你问自己“这个数据是不是一个‘设置’项？”时，如果答案是肯定的，那么 `NSUserDefaults` 很可能就是正确的选择。 |

好的，我们来看下一个。在处理比用户偏好设置更复杂、特别是涉及**敏感信息**时，`NSUserDefaults` 显然力不从心。这时，我们的主角就该登场了。

---

### 方案二：`Keychain` (钥匙串)

#### 1. 它是什么？

`Keychain` (钥匙串) 是 iOS 系统提供的一个**高度安全的、加密的**数据存储服务。你可以把它想象成一个由操作系统管理的、极其坚固的**保险箱**。

*   **安全加密 (Secure & Encrypted)**：存入钥匙串的数据会被系统使用行业标准的加密算法（如 AES）进行加密。即使设备被越狱，文件系统被访问，黑客也无法轻易解密其中的内容。
*   **独立于应用 (App-Independent)**：这个“保险箱”不完全属于你的 App。它是整个操作系统的一部分。这意味着它的生命周期和你的 App 是**分离**的。
*   **集中管理 (Centralized)**：系统统一管理所有 App 的钥匙串数据，并实施严格的访问控制。

#### 2. 适合存储什么？(The Right Job)

`Keychain` 的唯一目的就是：**存储少量、极其敏感的数据**。

**典型用例**:

*   **用户凭证**:
    *   用户密码
    *   登录后服务器返回的 `access_token` 或 `refresh_token`
*   **私密信息**:
    *   银行卡信息（虽然通常由更专业的支付 SDK 处理）
    *   加密密钥、私钥
    *   证书

**一句话原则：任何“泄露出去会导致严重后果”的数据，都应该存储在 `Keychain` 中。**

#### 3. 如何使用？(The Code)

直接使用 Apple 提供的原生 `Security.framework` API 是一件非常痛苦和复杂的事情。它是纯 C 语言风格的，需要和 `CFDictionary` 等 Core Foundation 类型打交道，代码冗长且易错。

因此，在实际项目中，**几乎所有开发者都会使用一个成熟的第三方开源库**来封装 `Keychain` 的操作。

*   **推荐的封装库**：
    *   `SAMKeychain` (Objective-C, 老牌、稳定)
    *   `KeychainAccess` (Swift, 非常流行)

下面以经典的 `SAMKeychain` 为例，展示其用法是多么简单：

```objc
#import "SAMKeychain.h"

// --- 定义一个 Service Name 和 Account Name ---
// 这两个字符串组合起来，作为你存储数据的唯一标识
NSString * const kKeychainService = @"com.myapp.service";
NSString * const kUserAccount = @"currentUser"; // 可以用 userID 或一个固定字符串

// --- 写/存数据 ---
// 假设 loginToken 是从服务器获取的敏感 Token 字符串
NSError *error = nil;
BOOL success = [SAMKeychain setPassword:loginToken 
                              forService:kKeychainService 
                                 account:kUserAccount 
                                   error:&error];
if (success) {
    NSLog(@"Token 成功存入钥匙串！");
} else {
    NSLog(@"存储失败: %@", error);
}


// --- 读/取数据 ---
NSError *readError = nil;
NSString *storedToken = [SAMKeychain passwordForService:kKeychainService 
                                                 account:kUserAccount 
                                                   error:&readError];
if (storedToken) {
    NSLog(@"从钥匙串中读取到 Token: %@", storedToken);
} else {
    NSLog(@"读取失败或 Token 不存在: %@", readError);
}


// --- 删除数据 ---
NSError *deleteError = nil;
BOOL deleteSuccess = [SAMKeychain deletePasswordForService:kKeychainService 
                                                    account:kUserAccount 
                                                      error:&deleteError];
if (deleteSuccess) {
    NSLog(@"成功从钥匙串删除 Token！");
}
```

#### 4. `Keychain` 的独特优势与特性

*   **App 卸载后数据不丢失**：
    这是 `Keychain` 最神奇的特性之一。默认情况下，用户**删除了你的 App，你存储在钥匙串里的数据依然会保留下来**。当用户下一次重新安装你的 App 时，你仍然可以读取到之前存储的 Token。
    *   **应用场景**：可以实现“静默登录”或“恢复登录状态”的绝佳用户体验。

*   **跨 App 共享数据**：
    如果你在同一个开发者账号下有多个 App，可以通过配置 `App Groups` 和 `Keychain Sharing` 权限，让这些 App 访问同一个钥匙串项目。
    *   **应用场景**：实现单点登录 (Single Sign-On, SSO)，用户在一个 App 中登录后，打开你的另一个 App 就自动是登录状态。

#### 5. 局限性

*   **不适合大数据**：`Keychain` 被设计用来存储少量数据，比如几十到几百字节的字符串。不要用它来存储图片、JSON 大段文本等。
*   **读写性能较低**：由于涉及加密和解密操作，`Keychain` 的读写速度远慢于 `NSUserDefaults` 或文件系统。不应在需要高性能、频繁读写的场景（如循环中）使用。

#### 总结

| 特性         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| **核心定位** | 系统的“加密保险箱”                                           |
| **优点**     | **高度安全**，**App 卸载后数据不丢失**，可跨 App 共享        |
| **缺点**     | **API 复杂 (需封装)**，不适合大数据，读写慢                  |
| **黄金法则** | 当你处理的是**密码、Token** 或任何需要加密保护的凭证时，`Keychain` 是**唯一正确**的选择。 |

## Objective-C vs. Java: 深入剖析两大编程语言的核心差异

Objective-C 和 Java 是两种强大的面向对象编程语言，它们在软件开发领域都曾扮演过举足轻重的角色。尽管它们都源于 C 语言的血统并支持面向对象，但在设计哲学、核心机制和应用生态上却存在着根本性的区别。Objective-C 曾是苹果生态系统（iOS 和 macOS）的基石，而 Java 则以其“一次编写，到处运行”的跨平台能力闻名于世，尤其在安卓和企业级应用开发中占据主导地位。

以下是两种语言在几个关键方面的详细对比：

### 核心机制：消息传递 vs. 方法调用

两者最本质的区别在于对象间的通信方式。

*   **Objective-C：动态的消息传递 (Message Passing)**
    Objective-C 继承自 Smalltalk，其核心是消息传递机制。当你执行 `[myObject doSomething]` 时，你实际上是在向 `myObject` 发送一个名为 `doSomething` 的消息。 系统会在运行时才去检查 `myObject` 是否能响应这个消息，并找到对应的方法来执行。 这种动态派发的特性赋予了 Objective-C 极大的灵活性，允许在运行时添加方法、替换方法实现（称为 "swizzling"）等高级操作。

*   **Java：静态的方法调用 (Method Invocation)**
    Java 的语法更接近 C++，采用的是方法调用。 当你写下 `myObject.doSomething()` 时，编译器在编译阶段就会检查 `myObject` 的类型，并确认它是否拥有一个名为 `doSomething()` 的方法。 如果没有，编译就会失败。虽然 Java 通过多态也实现了运行时的动态绑定，但其本质仍然是基于编译时确定的方法签名，相比 Objective-C 更为严格和静态。

### 内存管理：ARC vs. 垃圾回收

内存管理是区分两者的另一个重要方面。

*   **Objective-C：自动引用计数 (ARC)**
    现代 Objective-C 使用 ARC (Automatic Reference Counting) 来管理内存。 ARC 是一种编译器特性，它会在编译时自动为代码插入 `retain` (增加引用计数) 和 `release` (减少引用计数) 操作。 当一个对象的引用计数变为零时，其内存就会被立即回收。 这种方式的优点是内存回收的时机非常明确和及时，但缺点是需要开发者手动解决“循环引用”的问题，即两个或多个对象相互强引用，导致它们的引用计数永远不会为零。

*   **Java：垃圾回收 (Garbage Collection - GC)**
    Java 运行在 Java 虚拟机 (JVM) 之上，由 JVM 内置的垃圾回收器来自动管理内存。 垃圾回收器会定期扫描内存，找出不再被引用的对象并释放它们的空间。 这种方式的优点是完全自动化，开发者无需关心内存的释放，并且能够自动处理循环引用的问题。 缺点是垃圾回收的时机不确定，可能会在某个时间点导致应用短暂的停顿（Stop-The-World）。

## oc编译过程

### 🔹 2. 编译过程（以 `.m` 为例）

### （1）预处理（Preprocessing）

- 处理 `#import`、`#define`、宏替换
- 生成纯 C/Objective-C 代码，扩展后的临时文件（通常是 `.i`）

命令示例：

```bash
clang -E main.m -o main.i
```

------

### （2）编译（Compiling）

- 把 `.i` 转换成 **LLVM IR**（中间语言，类似汇编）
- 处理语法检查、类型检查
- 最终生成汇编代码（`.s` 文件）

命令示例：

```bash
clang -S main.i -o main.s
```

------

### （3）汇编（Assembling）

- 汇编器把 `.s` 文件转成 **目标文件** `.o`（二进制机器码，但未链接）

命令示例：

```bash
clang -c main.s -o main.o
```

------

### （4）链接（Linking）

- 链接所有 `.o` 文件和依赖的 **静态库（.a）**、**动态库（.dylib / framework）**
- 生成最终可执行文件（Mach-O 格式）

命令示例：

```bash
clang main.o -framework Foundation -o mai
```

------

### 🔹 4. 程序运行时

编译完成只是生成可执行文件，真正运行还要依赖：

- **dyld**：动态链接器，加载可执行文件和依赖库
- **Objective-C runtime**：注册类、方法、Category，支持消息发送、KVO、isa-swizzling 等动态特性

------

### 🔹 总结

Objective-C 的编译流程（和 C 类似）：

```
源代码 (.m/.h)
   ↓ 预处理
.i 文件
   ↓ 编译 (Clang → LLVM IR → 汇编)
.s 文件
   ↓ 汇编
.o 文件
   ↓ 链接 (lib + framework)
Mach-O 可执行文件
   ↓ (运行时：dyld + ObjC runtime)
程序启动
```

`dyld` 主要做以下几件大事：

#### 1. 装载依赖库 (Loading Libraries)
*   **比喻：** 把所有装家具零件的箱子搬进房间。
*   **技术解释：** 你的 App 不仅仅包含你写的代码，还依赖很多苹果官方的“零件库”（如 `UIKit`, `Foundation`）和第三方的“零件库”（如 `AFNetworking`）。`dyld` 的第一件事就是找到所有这些必需的库（动态库 `.dylib`），然后把它们全部加载到内存里。

#### 2. Rebase 和 Bind (最重要的环节，也是最耗时的)
*   **比喻：** 仔细阅读说明书，把螺丝拧到正确的孔里，把 A 板和 B 板连接起来。
*   **技术解释：** 由于安全机制 **ASLR** (地址空间布局随机化)，你的 App 每次启动时，它和它依赖的库在内存中的地址都是**随机**的。这就导致代码里写死的地址引用全部失效了。
    *   **Rebase (内部修正):** `dyld` 需要修正 App **内部**的指针。比如，你自己的一个函数要调用你自己的另一个函数，`dyld` 就要计算出它们今天在内存中的新地址，然后把这个调用关系“接上”。
    *   **Bind (外部绑定):** `dyld` 需要把你 App 的代码和**外部**库的代码“绑定”在一起。这是我们讨论的**核心**。比如，你的代码里调用了 `NSLog()`，`dyld` 必须去 `Foundation` 库里找到 `NSLog` 函数的真实内存地址，然后把你的调用代码指向这个地址。**这个查找和绑定的过程，就是性能瓶颈所在。**

#### 3. 运行初始化代码 (Run Initializers)
*   **比喻：** 家具组装好后，给里面的电器通上电，让它们自检。
*   **技术解释：** `dyld` 会执行所有 Objective-C 的 `+load` 方法和 C++ 的静态构造函数。这些代码被要求在 `main` 函数之前必须执行完毕。

当 `dyld` 做完所有这些工作后，它才会调用你熟悉的 `main` 函数，App 的启动流程才进入到你自己可控的部分。

---

### 第二部分：这个哈希表的作用是什么？—— “智能眼镜”

现在我们聚焦于 `dyld` 最耗时的环节：**Bind (外部绑定)**。

*   **没有优化前 (视力不好的老花眼师傅):**
    *   `dyld` 需要绑定 `NSLog`。
    *   它拿到的是一个**字符串** `"NSLog"`。
    *   它会在 `Foundation` 库暴露出来的成千上万个符号（函数名、变量名）列表里，进行**字符串比较**，一个一个地找，直到找到匹配 `"NSLog"` 的那一项。
    *   这个过程在一个大型 App 里要重复**几十万次**。你可以想象一个老花眼师傅，拿着一张有几十万个零件名字的清单，挨个对比，这速度有多慢。

*   **优化后 (戴上智能眼镜的师傅):**
    这个**完美哈希表**，就是那副为 `dyld` 量身定做的“智能眼镜”。

    **哈希表的作用，就是提供一个“跳过字符串比较”的超高速查找通道。**

    现在，`dyld` 的工作流程变成了：
    1.  需要绑定 `NSLog`。
    2.  它不再去进行字符串比较了。它把字符串 `"NSLog"` 喂给“智能眼镜”（完美哈希函数）。
    3.  “智能眼镜”内部进行了一次超快的数学计算，瞬间告诉师傅：“你要找的东西在**仓库的第 8888 号货架上**！”
    4.  `dyld` 直接跑到第 8888 号货架，拿到了 `NSLog` 的真实内存地址。

**这个哈希表在这里的核心作用就是：**

**将【极其耗时的字符串查找操作】，替换为【一次恒定时间的、超快的数学计算和数组访问】。**

## oc中单例怎么实现，原理是什么

### 单例的实现

实现单例最常见、最安全的方式是使用 **GCD (Grand Central Dispatch)** 的 `dispatch_once` 函数。

Objective-C

```objc
#import <Foundation/Foundation.h>

@interface MySingleton : NSObject

+ (instancetype)sharedInstance;

@end

// MySingleton.m
#import "MySingleton.h"

@implementation MySingleton

+ (instancetype)sharedInstance {
    static MySingleton *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[super alloc] init];
    });
    return sharedInstance;
}

@end
```

这个实现方式的核心是 `dispatch_once` 函数，它能保证在整个应用的生命周期中，由 `onceToken` 标记的代码块只会被执行一次，而且是线程安全的。

### `dispatch_once` 的原理

`dispatch_once` 的底层实现非常巧妙，它利用了原子操作来确保线程安全和只执行一次。

- **原子操作（Atomic Operation）**：在底层，`dispatch_once` 使用原子操作来检查 `onceToken` 的状态。`onceToken` 本质上是一个标记，它的初始状态是“未执行”。
- **锁定机制**：当第一个线程进入 `dispatch_once` 时，它会通过原子操作将 `onceToken` 标记为“正在执行”，这会阻止其他线程进入这个代码块。
- **执行代码块**：第一个线程执行 `sharedInstance = [[super alloc] init];` 这行代码，完成实例的创建。
- **标记完成**：代码块执行完毕后，`dispatch_once` 会通过原子操作将 `onceToken` 标记为“已执行”。
- **后续调用**：此后，当任何其他线程再次调用 `dispatch_once` 时，它会检查 `onceToken` 的状态。由于状态已经是“已执行”，它会直接跳过代码块，立即返回已经创建好的 `sharedInstance`。

## 代理模式和通知区别

### 代理模式（Delegate Pattern）

代理模式是一种**一对一**的、**紧密**的通信方式。一个对象（**被代理者**）将自己要处理的某些任务或事件，**委托**给另一个对象（**代理者**）来完成。

- **工作机制**：
  1. 被代理者（例如 `UITableView`）会持有一个对代理者（例如你的 `ViewController`）的弱引用。
  2. 被代理者定义一个**协议（Protocol）**，其中包含了代理者需要实现的方法（例如 `tableView:didSelectRowAtIndexPath:`）。
  3. 代理者遵循并实现这个协议中的方法。
  4. 当特定事件发生时，被代理者会直接**调用**代理者的方法来处理事件。
- **优点**：
  - **强类型**：编译器会在编译时检查协议方法是否被正确实现，有助于及早发现错误。
  - **明确**：通信路径清晰，一个对象直接调用另一个对象的方法，易于理解和调试。
  - **可控**：被代理者可以决定何时调用代理方法，并可以接收代理方法的返回值。
- **缺点**：
  - **一对一**：一个被代理者只能有一个代理者，不适合需要多方响应的场景。
  - **耦合**：被代理者需要知道代理者的存在，虽然通过协议解耦，但两者之间仍是紧密相连的。

### 通知（Notification）

通知是一种**一对多**的、**松散**的通信方式。一个对象（**发布者**）在发生特定事件时，向一个**通知中心（NotificationCenter）**广播一条消息。所有对该消息感兴趣的对象（**观察者**）都可以接收到这条消息，而发布者对此一无所知。

- **工作机制**：
  1. **观察者**（例如多个 `ViewController`）向通知中心**注册**，声明自己对某个通知感兴趣。
  2. **发布者**在事件发生时，向通知中心**发送**一个通知，通知中通常包含一个名称和一些可选的数据。
  3. 通知中心将该通知**广播**给所有已注册的观察者。
  4. 观察者收到通知后，执行相应的操作。
- **优点**：
  - **解耦**：发布者和观察者之间完全解耦，它们互相不知道对方的存在。
  - **一对多**：一个事件可以同时被多个对象监听和响应，非常适合全局性的事件广播。
  - **灵活**：发送和接收通知的过程非常灵活，可以传递任何数据。
- **缺点**：
  - **无类型**：通知名称是字符串，编译器无法在编译时进行类型检查。
  - **难以追踪**：因为发布者和观察者之间没有直接联系，在代码中追溯消息流向会比较困难，容易造成维护问题。

## dyld的过程

### **宏观视角：从点击图标到`main`**

1.  **用户点击App图标**: SpringBoard（桌面App）收到点击事件。
2.  **启动进程**: SpringBoard通过`launchd`服务来启动新App的进程。
3.  **内核（XNU）介入**:
    *   `launchd`会发起一个`execve`系统调用，请求内核创建一个新的进程。
    *   内核（XNU）在内核空间创建进程，分配资源，然后开始加载App的可执行文件（一个**Mach-O格式**文件）。
4.  **识别`dyld`**: 内核在解析Mach-O文件时，会检查一个特殊的加载命令（Load Command）——`LC_LOAD_DYLINKER`。这个命令告诉内核：“我不是一个完整的、独立的可执行文件，请先加载指定的动态链接器来帮助我运行”。在iOS上，这个链接器就是`/usr/lib/dyld`。
5.  **加载`dyld`并移交控制权**: 内核加载`dyld`到App的地址空间，然后将**程序的控制权（PC寄存器）**交给`dyld`的入口点。

**从这里开始，`dyld`正式接管了整个启动过程。**

---

### **`dyld`的详细工作流程**

`dyld`的核心任务可以概括为以下几个主要步骤：

#### **第1步：加载所有依赖的动态库 (Loading Libraries)**

`dyld`的首要任务是构建一个完整的、可运行的程序镜像。

1.  **分析主可执行文件**: `dyld`首先分析App自己的Mach-O文件，读取其中的加载命令，特别是`LC_LOAD_DYLIB`。这些命令列出了App依赖的所有动态库（例如 `UIKit.framework`, `Foundation.framework` 等）。

2.  **递归解析依赖**: 这是一个递归的过程。`dyld`会：
    *   为App依赖的每个库找到对应的文件。
    *   加载该库的Mach-O文件。
    *   **再分析这个库的依赖**（比如`UIKit`依赖于`Foundation`）。
    *   将所有依赖项构建成一个依赖关系图，确保没有循环依赖，并确定加载顺序。

3.  **利用共享缓存 (dyld Shared Cache)**:
    *   **这是iOS上最重要的启动优化**。iOS上几乎所有的系统动态库（`UIKit`, `Foundation`等）都被预先处理并合并成一个巨大的缓存文件，存放在`/System/Library/Caches/com.apple.dyld/`。
    *   这个缓存文件中的库已经被“预链接”（pre-linked），它们的符号地址和依赖关系都已计算好。
    *   当`dyld`需要加载一个系统库时，它不需要去文件系统中单独寻找，而是直接从这个已经映射到内存的共享缓存中**直接获取**。这极大地减少了I/O操作和计算量，是iOS App启动速度快的关键原因之一。

4.  **映射到内存**: `dyld`将主可执行文件和所有依赖库的Mach-O文件映射到App的地址空间中。此时，它们只是被加载进来了，但还不能正确运行，因为里面的地址都是“错”的。

#### **第2步：修正地址（Rebasing 和 Binding）**

这是`dyld`最复杂、最核心的工作。由于**ASLR (Address Space Layout Randomization)**安全机制，每次App启动时，主程序和动态库都会被加载到一个**随机的内存地址**。然而，编译器在生成二进制文件时，假设它会被加载到一个固定的基地址（通常是0x0）。因此，所有内部的指针和函数引用地址都需要在运行时进行修正。

1.  **Rebasing (重定位/基地址复位)**:
    *   **目的**: 修正**二进制文件内部**的指针。
    *   **过程**: `dyld`计算出每个镜像（主程序或某个库）的实际加载地址与编译时假设的基地址之间的**差值（slide）**。然后，它会遍历二进制文件的 `__LINKEDIT` 段，这里记录了所有需要修正的内部指针位置。`dyld`会给每个这样的位置加上这个差值，使其指向正确的内存地址。
    *   **例子**: App内的一个全局变量指针，原本指向地址`0x1000`，如果App被加载到了`0x5A000`，差值就是`0x5A000`。`dyld`就会把这个指针的值更新为`0x5A000 + 0x1000`。
2.  **Binding (绑定)**:
    *   **目的**: 修正**指向外部动态库**的函数或符号的指针。
    *   **过程**: `dyld`需要解析所有未定义的“桩”符号（stub）。例如，当你的代码调用`NSLog`时，Mach-O文件里只有一个指向`NSLog`的占位符。`dyld`会在所有加载的动态库的符号表中查找名为`_NSLog`的符号，找到它在内存中的**真实地址**，然后将这个真实地址**回填**到App的占位符位置。

## oc中对一个nil调用方法 会报错吗 那假如这个方法返回值是bool 会返回什么

在 Objective-C (OC) 中，对一个 **`nil` 对象发送消息（调用方法）** 是完全有效的，并且不会导致程序崩溃或抛出异常。这是一个非常重要的特性，也是 OC 和其他一些语言（如 Swift 或 Java）的主要区别之一。



### 对 `nil` 调用方法会怎样？

当你对 `nil` 对象调用方法时，运行时（runtime）会默默地忽略这个调用。

- **对于没有返回值的方法**（返回 `void`），什么也不会发生，调用被安全地忽略。
- **对于有返回值的方法**，返回值会是 `0`、`NULL`、`nil` 或 `NO`。具体取决于返回类型。

### `nil` 方法调用的返回值

根据方法的返回类型，对 `nil` 调用方法会返回一个“零值”（zero value）。

1. **对象类型（id 或自定义类）：** 返回 `nil`。
   - 比如 `NSString *myString = nil;`
   - `NSString *result = [myString uppercaseString];`
   - 这里的 `result` 将是 `nil`。
2. **基本数据类型：** 返回 `0`。
   - 比如 `int result = [myObject intValue];` 如果 `myObject` 是 `nil`，`result` 会是 `0`。
   - **特别注意：** `NSInteger`、`CGFloat`、`float`、`double` 等类型也会返回 `0`。
3. **布尔类型（`BOOL`）：** 返回 `NO`。
   - 比如 `BOOL isValid = [myObject isValid];`
   - 如果 `myObject` 是 `nil`，`isValid` 会被赋值为 `NO`。

下面是一个关于布尔类型返回值的具体例子：

Objective-c

```objective-c
@interface MyClass : NSObject
- (BOOL)isReady;
@end

@implementation MyClass
- (BOOL)isReady {
    // 假设这个方法做一些检查
    return YES;
}
@end

int main() {
    MyClass *obj = [[MyClass alloc] init];
    BOOL result1 = [obj isReady]; // result1 将是 YES

    MyClass *nilObj = nil;
    BOOL result2 = [nilObj isReady]; // result2 将是 NO

    if (result2 == NO) {
        NSLog(@"对 nil 调用 BOOL 方法，返回了 NO。");
    }

    return 0;
}
```

# java

## java基础

### ***单例手写

### ***==和equals

### ***如何理解 Java 的跨平台性

### ***三大特性

对于 `String` 类型，`==` 运算符比较的是两个 `String` 对象的引用地址是否相同，也就是判断它们是否是内存中的同一个对象实例。而 `equals()` 方法（`String` 类重写了 `Object` 类的 `equals()` 方法）比较的是两个 `String` 对象的字符序列是否完全相同，即它们的内容是否相等。

### ***重写equals为什么要重写hashcode？

JDK的约定就是如果 `equals` 方法返回 `true`，则 `hashCode` 必须相等。不重写会出现hashset存入异常，两个equals的类，会被存到两个位置，没有去重。本身就不合逻辑

### throw和throws的区别

* **throw:**用在**方法体内部**，用来**显式地抛出一个异常对象**。
* **throws :**声明一个方法可能抛出的异常类型，告知调用者需要处理这些异常,调用这个方法的时候idea会强制要求你try catch包住 。

### Object类有哪些方法？

`equals(),hashcode(),toString,getClass(),clone(),wait(),notify()`

### ***Java为什么要有Integer？

### *说一下 integer的缓存

### *？泛型为什么只能存对象

类型擦除是 Java 编译器在**编译阶段**将泛型参数替换为 **原始类型（Raw Type）** 并移除泛型信息的过程。

在编译后，Java **会擦除泛型 `T`，并用 `Object` 代替**：

```java
class Box {
    private Object value;  // 泛型 T 被擦除为 Object

    public void setValue(Object value) {  // 泛型方法也被擦除
        this.value = value;
    }

    public Object getValue() {  // 泛型返回类型被擦除
        return value;
    }
}

```

### **怎么理解面向对象？简单说说封装继承多态

**继承** 

* **代码复用**
* **可维护性 :** 当需要修改或修复父类中的某个通用功能时，只需修改父类代码即可。
* **可扩展性 (Extensibility):** 可以基于现有的父类轻松创建新的子类，添加或修改特定的功能。

**多态:**多态允许你使用父类或接口类型的引用来指向子类或实现类的对象

* 接口规范
* 多态使得代码不依赖于具体的子类实现，而是依赖于抽象的父类或接口。

* **方法重写（Override）**：子类重写父类的方法，实现不同的功能。

* **方法重载（Overload）**：同一类中的方法名称相同，但参数不同。

**为什么需要多态？**

- **提高代码的扩展性**：可以在不修改原代码的情况下，添加新的行为。
- **让代码更加灵活**：父类引用可以指向不同的子类对象，代码复用性强。

###  ***?面向对象的设计原则

**依赖倒置原则**:简单来说，就是 要依赖抽象（接口），而不是具体的实现类
**开放封闭原则**：**可以扩展功能（Open for extension）** ，✅**但不能修改已有代码（Closed for modification）** ❌。

**单一职责原则**：即一个类应该只负责一项职责

比如说某个类的方法依赖其他类的方法，最好在这个类中只定义类的接口就行，实际用的时候再传，而不是定义的时候就写死了某个实例。

### ***Java抽象类和接口的区别

* 单继承和多继承
* 可以有普通变量和 `static` 变量，接口只能有 `public static final` 变量
* 可以有**普通方法**和**抽象方法**，Java 8+ 可以有 `default` 方法和 `static` 方法，其他方法都是**抽象的**，default方法子类可以直接用或者重写
* 适用于**有相同属性或部分实现**的类，适用于**定义行为规范**

### **抽象类和普通类区别？

* 抽象类可以有抽象方法和普通方法，接口只能有默认方法，抽象方法，静态方法
* 抽象类是单继承，接口多继承，抽象类可以用来代码复用，接口主要是定义接口规范

### *什么是泛型？

### **java创建对象有哪些方式

* 反射创建对象

```java
    Class<?> clazz = MyClass.class;
    MyClass obj = (MyClass) clazz.newInstance();
```



### **反射在你平时写代码应用场景有哪些?

###  Java 8 你知道有什么新特性？

* **Lambda 表达式**：`(a, b) -> a + b` 代替匿名类实现接口

* 函数式接口是只包含一个抽象方法的接口。

* 接口可定义**默认实现和静态方法**，增强扩展性

* **Stream API**:

* ```java
  List<String> originalList = Arrays.asList("apple", "fig", "banana", "kiwi");
  List<String> filteredList = originalList.stream()
                          .filter(s -> s.length() > 3)
                          .collect(Collectors.toList());
  ```

### ？？？？深拷贝和浅拷贝

* 浅拷贝拷贝一个对象，会复制出一个新的对象来，但是这个对象里面的引用类型依然指向原来的那个对象里面的引用。

* 深拷贝对象的引用类型中则指向一个新的对象。

### ？？？try里面有return的话，执行过程是怎么样的

* 先把return的值保存起来，再去执行finally，假如finally里也有return 就会return finally的值。

## java集合

### ***常⻅的集合有哪些？

**ArrayList**

* 底层是动态数组

* `ArrayList` **可以使用任意单列（非 Map 类型）集合初始化**
* **扩容逻辑**：当数组满了，扩容**1.5倍**，初始容量是**0**，第一次添加时就**扩容到10.**
* Array.asList(数组)可以**转成List**，**List转数组**：list.toArray（new int[list.size()]）

**LinkedList和ArrayList的区别**

* linkedlist是双向链表实现的，ArrayList底层是动态数组
* 都不是线程安全的

**CopyOnWriteArrayList**、**TreeSet**（自动排序的集合）、**LinkedHashMap**（保留了元素插入的顺序）、**PriorityQueue**（根据元素的优先级来获取元素）

###  ***Arraylist和LinkedList的区别？

### ***HashMap的原理

### 为什么ArrayList不是线程安全的？

### 为什么HashMap要用红黑树而不是平衡二叉树？

### HashMap的扩容机制

### 列举HashMap在多线程出现的问题？

###  如何对map进行快速遍历？

### Hashmap和Hashtable有什么不一样的？

### CopyOnWriteArrList 是怎么实现安全的

当需要修改 CopyOnWriteArrayList 中的元素时，它不会直接在原数组上进行修改，而是会进行以下步骤：

1. **复制数组 (Copy):** 创建一个原数组的副本（一个新的数组）。
2. **修改副本 (Write):** 在这个新的副本数组上执行修改操作（例如，添加、删除、更新元素）。
3. **替换引用 (Replace):** 将内部指向旧数组的引用更新为指向这个新的副本数组。

由于所有修改操作都是在副本上进行的，并且在修改完成之前，读操作仍然是在旧数组上进行的，因此**读操作和写操作不会互相干扰

## JUC

### ***java有哪些锁

### ***AQS的原理

**一个原子变量 `state`** 来表示锁的状态（0 代表空闲，大于 0 代表锁被占用）。

**CAS（Compare-And-Swap）操作** 来保证 `state` 的原子性修改。

**一个 FIFO 线程等待队列** 来管理被阻塞的线程。

基本思想就是维护一个原子state和一个owner，一个线程获取锁的逻辑就是尝试cas的修改这个state为1，如果修改成功，就把owner改成当前线程。如果没获取到锁，把线程放到队列尾部，加进去之后自旋一次获取锁（这是为了防止加入之后锁已经被释放了），没获取到锁就阻塞等待唤醒。解锁逻辑就是，把state修改为0，唤醒队列里的的线程。

### ***什么情况会产生死锁问题？

**互斥**（Mutual Exclusion）：

- 资源一次只能被一个线程占用。

**占有且等待**（Hold and Wait）：

- 线程1持有A资源的同时，还在等待B资源。线程2持有B资源，等待A资源。

**不可剥夺**（No Preemption）：

- 资源不能被强制回收，必须由持有者释放。

**循环等待**（Circular Wait）：

- 线程之间形成资源**环形依赖**（A 等 B，B 等 A）

### ***voliatle关键字有什么作用？

### ***synchronized和reentrantlock及其应用场景有什么区别？

* **可中断性**:可以使用 `tryLock() 和 lockInterruptibly() 方法能够响应中`
* **设置超时时间**: `tryLock(long time, TimeUnit unit)` 方法来设置超时时间
* **公平锁和非公平锁: `new ReentrantLock(true)` 来创建**
* **实现不同：**synchronized 是 JVM 层面通过监视器实现的，而 ReentrantLock 是基于 AQS 实现的。

### ***synchronized原理？

它依赖于 JVM 的监视器锁（Monitor）机制，当使用syn（obj）加锁时，锁对象的对象头里面的mark word会标识锁的不同状态，锁有一个升级的过程。

* 偏向锁是为了优化单线程访问的场景。当对象没有被任何线程锁定时，JVM 会自动将锁偏向第一个获取锁的线程。这样，后续同一线程重复获取锁时不需要进行任何竞争或加锁操作。**Mark Word** 中存储的是持有锁的线程 ID，表示该对象的锁被该线程偏向。
* CAS的逻辑就是，尝试把线程id替换为自己的线程id，如果失败了说明有线程在竞争，锁升级
* 当有多个线程竞争锁时，**偏向锁**会被升级为**轻量级锁**。轻量级锁会在对象头的 **Mark Word** 中存储一个指向**锁记录的指针（存储了线程的信息）**，其他线程会自旋尝试获取锁即**CAS**的修改Mark word中的**标志位（要从无锁01改成有锁00），成功了就把当前线程的锁记录放进去**，直到自旋到获取到这个锁。
* 当锁竞争激烈时，轻量级锁会升级为 **重量级锁**，此时 `Mark Word` 会记录指向 **Monitor** 对象的地址。**Monitor** 中包含一个 **owner**，它表示当前持有锁的线程，同时还有一个 **阻塞队列**，用于存储那些尝试获取锁但被阻塞的线程。解锁时，持有锁的线程会释放锁并唤醒阻塞队列中的线程，允许它们去竞争锁。

### ***sleep 和 wait的区别是什么？

### StringBuffer怎么实现线程安全的

### Java线程的状态有哪些

### ***导致并发安全的根本问题，怎么保证多线程安全？

* 根本问题，可见性，有序性，原子性

### 不同的线程之间如何通信？

* wait()、notify()、notifyAll()
* `RecurrentLock` 和 `newCondition` 方法。这个比wait，notify更加精细。可以显式的唤醒某个线程
* `CountDownLatch`

### synchronized锁静态方法和普通方法区别

* 普通方法锁的是当前实例对象，静态方法锁的是类对象

### synchronized能够修饰什么，各有什么区别

* 普通方法：锁住调用类的实例对象
* 静态方法：锁住class对象
* 同步代码块

### JMM?

### CAS 有什么缺点？

### CAS原理

而是直接通过 JVM 调用了操作系统底层的**CPU 提供的原子指令**，一次实现比较和交换两个动作。`Unsafe` 类提供了一个桥梁

### 悲观锁和乐观锁的区别？

### 指令重排序的原理是什么？

### ***ConcurrentHashMap？

volatile + CAS 或者 synchronized 实现

* `Node.val` 被 **`volatile` 修饰**，保证保证写入后对其他线程可见。计算hashcode之后，如果该索引位置是空的，使用 CAS 插入，失败的线程会重试。
* 如果桶位置不为空，就利用synchronized锁住该位置链表的头节点，去链表找那个key，**不会锁整个 `ConcurrentHashMap`**，并发度较高。

### Runnable和Callable有什么区别

✅ **使用 `Runnable`**

- 不需要返回值
- 任务是 I/O 操作、日志记录、数据处理等

✅ **使用 `Callable`**

- 需要返回值（如计算任务）
- 需要处理受检异常（如数据库操作、文件读取）
- 需要与 `Future` 结合，异步获取结果

### ***？为什么要用线程池

* 复用线程，避免频繁创建和销毁线程的开销
* 限制最大线程数，避免系统过载

### *** 线程池的参数是哪些，怎么选择参数，线程池原理是什么？

* **corePoolSize**：核心线程数。**CPU密集型：CPU核数 + 1**（电商秒杀是短暂的 CPU 消耗，大量的io等待（数据库缓存读写）），**IO密集型（）:CPU核数 x 2**
* **maximumPoolSize：**最大线程数 = 核心 + 救急线程数 
* **keepAliveTime**：救济线程的生存实时间
* **workQueue**：线程阻塞队列，没有空闲核心线程时，任务进入队列等待
* **threadFactory**：定制线程的名字
* **handler**：当所有线程都忙，阻塞队列也满了，考虑拒绝
* 任务来了，线程池优先分配空闲核心线程；若无空闲核心线程，则尝试加入任务队列；队列满时，若未达最大线程数则创建非核心线程；超出最大线程数和队列容量则执行拒绝策略。

### ***有哪些拒绝策略

* CallerRunsPolicy，使用线程池的调用者所在的线程去执行被拒绝的任务，除非线程池被停止或者线程池的任务队列已有空缺。
* AbortPolicy，直接抛出一个任务被线程池拒绝的异常。
* DiscardPolicy，不做任何处理，静默拒绝提交的任务。
* DiscardOldestPolicy，抛弃最老的任务，然后执行该任务。

### **线程池种类有哪些？

* **FixedThreadPool：**拥有固定数量的核心线程，没有救急线程，阻塞队列无限大，容易导致oom。
* **CachedThreadPool：** 核心线程数为 0，最大线程数无限制，没有阻塞队列，空闲线程会被回收。容易创建很多线程导致oom。
* **SingleThreadExecutor：** 只有一个核心线程，保证所有任务按顺序执行，阻塞队列无限大。适用于需要串行执行任务的场景。容易导致oom
* **ScheduledThreadPool**: 支持定时或周期性执行任务。适用于需要调度任务的场景。

### **线程池有哪些阻塞队列

* **ArrayBlockingQueue**（有界，FIFO）➡ **适合任务量可预测**，防止 OOM。一把锁锁住整个数组，入队和出队操作是互斥的。
* **LinkedBlockingQueue**（默认无界（可以设置有界），FIFO）➡ **吞吐量高，但可能导致 OOM**。两把锁 锁住链表头和尾，出队入队操作不互斥。
* **SynchronousQueue**（不存储任务，直接交给线程执行）➡ **适用于高并发、任务执行迅速**。
* **DelayQueue**（任务延迟执行）➡ **适用于定时任务、缓存超时清理**。

### 线程池中的线程执行遇到异常和处理？任务丢失怎么解决？

* 任务内部遇到异常，用try-catch捕捉打印异常就可以了
* 如何程序崩溃可能就会丢失任务，可以考虑用mq来存储消息，或者直接持久化到数据库

## JVM

### ***四大引用

### ***jdk 、jre、 jvm 关系及用途 

### ***类载入过程 JVM 会做什么？

### ***JVM的内存模型

线程共享的的：

**元空间（方法区）**：

* **运行时常量池**：类的常量 + 符号引用 
* **类信息、常量、静态变量等数据。**

**Java 堆**：存放对象实例。

----

线程私有的：

**虚拟机栈**：局部变量、方法返回地址

**程序计数器**：存储当前线程正在执行的 Java 方法的 JVM 指令地址

**本地方法栈**：与 Java 虚拟机栈类似，主要为虚拟机使用到的 Native 方法服务。

----

**本地内存**

### ***JVM了解吗？

它的主要作用包括：

* **类加载器（ClassLoader）：** 负责加载 `.class` 成字节码
* **运行时数据区**：JVM 在执行程序时使用的内存区域
* **执行引擎：** 负责执行字节码
* **垃圾回收器**

###  ***双亲委派模型的过程和作用

* 保护 **Java 核心 API**，防止篡改
* 防止类的重复加载，确保**类的唯一性**

### ***内存泄漏和内存溢出的理解

**内存泄漏**

* 静态集合类如 `HashMap`、`ArrayList`，往里面不断添加对象，导致垃圾回收无法回收
* 线程持有外部引用

### ***Gc Root是哪些，有什么用

* 调用方法中的局部变量、类中的static变量引用某些堆中的变量、常量引用的对象

### 程序计数器的作用

### 对象创建的过程了解吗

* **类加载检查**：JVM 会先检查对应的类是否已经被加载、解析和初始化过
* **内存的分配**：指针碰撞： 适用于规整的堆内存，通过移动指针划分已用和空闲区域。空闲列表：适用于碎片化的堆内存，维护可用内存块列表并从中分配。
* **零值初始化**：JVM 会对分配的内存空间进行初始化，将其所有字段设置为零值
* **设置对象头**：Mark Word、Klass Pointer和数组长度
* **执行构造方法**

### ***OOM类型和原因-OutOfMemoryError

**堆内存溢出** **-Java Heap Space**

* **对象过多**
* **内存泄漏**（对象长期持有，无法被 GC 释放）
* **创建过大的对象**

**Metaspace 溢出** **-Metaspace OOM**

* **过多动态生成的类**（如**大量代理类、反射**，这是因为**动态生成的类都存在元空间**）
* **应用依赖的第三方库过多**

### StackOverflowError 

* **递归深度过深**，导致 **栈帧爆满**

### ***Error和exception

**Error**

* OOM
* StackOverflowError
* NoClassDefFoundError。 JVM在运行时找不到某个类定义。这通常是因为在编译时类存在，但在运行时类路径（classpath）中缺失。

**Checked Exception**和**RuntimeException**

**Checked Exception**

* FileNotFoundException

* IOException，读写文件失败
* InterruptedException,线程被中断

**RuntimeException**

* NullPointerException
* ArrayIndexOutOfBoundsException
* StringIndexOutOfBoundsException
* ArithmeticException 除0异常
* ClassCastException 类型转换异

### 运行时常量池，字符串常量池，常量池之间的关系

**运行时常量池**在**元空间（方法区）**中，其中包含字符串常量池（StringTable），而**常量池是一张表**。

### 类加载器有哪些

### 类加载过程

* **加载**：通过 **类的全类名** 找到 `.class` 文件，并将字节码转换为 **方法区中 **类的运行时的数据结构（c++实现的instanceKlass 描述 java 类），在堆中生成`Class` 对象。比如说可以通过对.Class对象访问到这个Klass，实例对象则访问这个Klass则是通过类对象中转访问。

* **验证**：确保字节码文件的正确性，检查字节码文件格式，字节码内容，符号引用，元数据

* **准备**： 为**类的静态变量** 分配内存，并赋初始值，它通常是存在类对象中的。

* **解析**：**符号引用**（类、方法）替换为 **直接引用**（地址）

* **初始化**：执行类的静态代码块，给静态变量赋值，只有在**真正使用**该类时才触发，如：
  - `new` 关键字创建对象
  - 访问静态变量/调用静态方法
  - 反射调用 `Class.forName("com.example.MyClass")`

* 

### 如果有个大对象一般是在哪个区域？

### 四大引用

### 字符串变量的拼接和字符串常量的拼接

s1 + s2的本质是StringBuilder的append然后toString(),会new一个新的字符串对象。而“a” + “b” = “ab”，”ab“对象本质上是存在字符串常量池中

###  ***垃圾回收算法有？

### 垃圾回收器有哪些？

* **ParNew GC** 和**CMS**
* **Serial GC 和 Serial Old GC**:它在进行垃圾回收时，会暂停所有用户线程。
* **G1**

### ？引用类型有哪些？有什么区别

### **复制算法和标记整理算法的区别

复制算法主要是用于新生代，**新生代对象生命周期短**，大部分对象会被回收，只有少量对象需要复制，成本较低。

 **标记-整理算法****适用于老年代（Old Generation）**，因为老年代对象存活时间长，**复制算法的成本太高**。

### **为什么要分新生代和老年代

### ***什么时候会发生STW，圾回收过程中为什么要有stop the world?

* 根对象的可达性分析
* 对象移动和内存整理

### ***minorGC、majorGC、fullGC？

### ***垃圾回收器 CMS 和 G1

**CMS收集老年代**（并发）-标记清除 ，最小化停顿时间为目标的垃圾收集器

* **初始标记**：先将Gc root关联的**第一个对象标记**，**STW**
* **并发标记**：不会STW，**顺着GC root继续标记整个堆内存**。
* **重新标记**：因为用户线程在修改， 前面的**标记可能有错误**（为了解决浮动垃圾和对象消失（一开始是垃圾，后来也被引用了，相当于正在用的对象被回收）），所以SWT重新标记
* **并发清理**：不会STW
* **Full GC**：年轻代晋升失败老年代没有连续内存，只能full gc整理老年代 ；直接在老年代分配对象（尤其是大对象）时，由于内存碎片导致空间不足发生Full GC

**G1收集老年代和新生代**

**分区Region**，将整个Java推分成大小相同的独立Region块， 一个Region有可能属于Eden、survivor或者Old区。

* **MaxGCPauseMillis**，最大停顿时间目标
* **新生代收集**：当Eden区满时触发，扫描整个年轻代，把**Eden或者survivor**的幸存对象**拷贝**到幸存区或者 Old Region，会发生**STW**
* **并发标记周期**： 当整个堆内存使用率达到一定阈值（45%），识别整个堆中（主要是 Old Region）的存活对象，为后续的 Mixed GC 提供哪些 Old Region 最值得回收（即垃圾最多）的信息。
  * **初始标记**，标记 GC Roots 直接可达的对象，**会STW**
  * **整个堆中进行并发标记**，遍历整个堆的对象，**不会STW**
  * **重新标记**：修正错误，**STW**
  * **清理**，清理**没有存活对象**的old region，统计每个 Old Region 中存活对象的比例，为混合收集做准备。**会STW**
* **混合收集（新生代，幸存区，老年代一起收集）**：并发标记周期成功完成后，G1 就知道哪些 Old Region 含有最多的垃圾，可能会触发一次混合收集。同时回收**全部年轻代**和部分**回收价值高的老年代**。会**STW**
* **Full GC**

**区别**

* 标记-清除和复制
* CMS容易产生内存碎片，依赖 Full GC 整理内存碎片，容易产生浮动垃圾
* 老年代，整个堆内存

### *跨代引用的问题

remember set

### G1 gemin 2.5pro 回答

G1 是一款面向服务端应用、主要针对大内存（通常指 6GB 以上）和需要可预测暂停时间场景的垃圾回收器。它与之前的分代回收器（如 CMS、Parallel Scavenge/Old）最大的不同在于其**堆内存布局**和**回收策略**。

**1. G1 的堆内存结构：基于 Region**

- **分区 (Region):** G1 不再将堆物理上划分为连续的年轻代和老年代。它将整个 Java 堆划分为多个大小相等（通常为 1MB 到 32MB 之间，启动时计算确定）的独立区域 (Region)。
- **角色动态分配:** 每个 Region 在某个时刻可能扮演 Eden、Survivor 或 Old Generation 的角色。还有一种特殊的 Humongous Region，用于存储大对象（通常超过 Region 大小一半的对象会直接分配在 Humongous Region，可能跨越多个连续 Region）。
- **Remembered Set (RSet):** 为了解决跨 Region 引用的问题（例如，一个 Old Region 中的对象引用了 Eden Region 中的对象），每个 Region 都有一个关联的 Remembered Set。RSet 记录了 *其他* Region 指向 *当前* Region 的引用信息（具体来说，是记录了引用来源的 Card）。这使得 G1 在回收某个 Region 时，不需要扫描整个堆来查找外部引用，只需查看其 RSet 即可，大大减少了 Young GC 和 Mixed GC 的扫描范围。

**2. G1 的垃圾回收流程**

G1 的回收过程主要涉及以下几种类型的 GC 活动，它们穿插进行：

- **Young GC (年轻代回收 / Evacuation Pause):**

  - **触发时机:** 当 Eden 区满时触发。

  - 过程:

    这是一个 Stop-The-World (STW) 的暂停。G1 会选定所有当前的 Eden Region 和 Survivor Region 组成 

    Collection Set (CSet)

    。然后，扫描 CSet 中的对象：

    - 存活的对象会被复制（Evacuate）到新的 Survivor Region 或者 Old Region（如果对象年龄达到阈值或 Survivor 区空间不足）。
    - 死亡的对象占用的空间被回收。
    - 在这个过程中，会更新相关 Region 的 RSet（因为对象移动了位置，引用关系可能改变）。

  - **特点:** 只回收年轻代区域，速度较快，暂停时间相对容易控制。G1 会根据用户设定的目标暂停时间 ( `-XX:MaxGCPauseMillis` ) 和历史数据来动态调整年轻代 Region 的数量，以尽量满足暂停时间目标。

- **Concurrent Marking Cycle (并发标记周期):**

  - **目标:** 识别整个堆中（主要是 Old Region）的存活对象，为后续的 Mixed GC 提供哪些 Old Region 最值得回收（即垃圾最多）的信息。
  - **触发时机:** 当整个堆内存使用率达到一定阈值时（由 `-XX:InitiatingHeapOccupancyPercent` 控制，默认 45%）会启动。它不是一次性完成的，而是穿插在 Young GC 之间并发执行。
  - 主要阶段:
    1. **Initial Mark (初始标记):** 标记 GC Roots 直接可达的对象。这个阶段需要 STW，但它通常** piggyback（搭便车）** 在一次 Young GC 上完成，所以增加的暂停时间很短。
    2. **Root Region Scanning (根区域扫描):** 扫描在 Initial Mark 暂停中标记的 Survivor Region 对 Old Region 的引用。此阶段与应用程序并发运行，不需要 STW。
    3. **Concurrent Marking (并发标记):** 从 GC Roots 开始，遍历整个堆的对象图，查找所有存活对象。这是并发标记的核心阶段，与应用程序并发执行，耗时可能较长，但不会暂停应用。G1 使用 **SATB (Snapshot-At-The-Beginning)** 算法来处理并发标记期间对象引用关系的变化。
    4. **Remark (重新标记):** 处理在并发标记阶段结束后，因应用程序运行而产生的少量引用变化（主要是处理 SATB 记录的日志）。这是一个 STW 阶段，但通常比 CMS 的 Remark 要快。
    5. Cleanup (清理):
       - 识别出完全没有存活对象的 Old Region，并立即回收这些 Region 的空间（这个过程部分 STW，部分并发）。
       - 统计每个 Old Region 中存活对象的比例（Liveness），用于后续 Mixed GC 筛选 CSet。
       - 重置 RSet 相关信息。

- **Mixed GC (混合回收 / Evacuation Pause):**

  - **触发时机:** 在并发标记周期成功完成后，G1 就知道哪些 Old Region 含有最多的垃圾。当后续触发 Young GC 时，G1 *可能* 会决定执行一次 Mixed GC。
  - **过程:** 这也是一个 STW 的暂停。它的 CSet 不仅包含**所有**的 Young Region (Eden + Survivor)，还会根据用户设定的暂停时间目标和各个 Old Region 的回收价值（垃圾比例），选择**部分**回收收益高的 Old Region 加入 CSet。然后，像 Young GC 一样，将 CSet 中所有 Region 的存活对象复制到新的 Region（可能是 Survivor 或 Old）。
  - **特点:** 同时回收年轻代和部分老年代。这是 G1 "Garbage-First" 名字的由来，因为它优先回收垃圾最多的区域。G1 会执行多次 Mixed GC 来逐步回收老年代的垃圾，而不是一次性回收所有老年代（避免长时间暂停）。Mixed GC 的次数由 `-XX:G1MixedGCCountTarget` 等参数影响。

- **Full GC (完全回收):**

  - 触发时机:

    G1 会尽量避免 Full GC。但在某些极端情况下，例如：

    - 并发模式失败（如并发标记过程中 OOM）。
    - 提升失败（Evacuation Failure）：在 Young GC 或 Mixed GC 期间，需要将对象复制到 Survivor 或 Old 区，但找不到足够的连续空间。
    - 分配 Humongous 对象时空间不足。
    - 显式调用 `System.gc()` (除非配置了 `-XX:+ExplicitGCInvokesConcurrent`)。

  - **过程:** G1 的 Full GC 是单线程的、STW 的，并且会对整个堆进行整理（Compaction）。这会导致非常长的暂停时间，是 G1 性能优化的主要规避目标。如果频繁发生 Full GC，通常意味着需要调整 G1 的配置或增加堆内存。

**总结 G1 流程特点:**

1. **基于 Region:** 内存管理更灵活，方便进行局部回收。
2. **可预测的暂停时间:** 通过选择回收部分 Region (CSet) 和动态调整 Young Region 数量，尝试满足用户设定的 `-XX:MaxGCPauseMillis` 目标。
3. **并发标记:** 使用并发标记周期来识别老年代垃圾，减少 STW 时间。
4. **Garbage-First 策略:** 在 Mixed GC 中优先回收垃圾最多的 Old Region，提高回收效率。
5. **空间整理:** 回收过程是基于复制（Evacuation）的，天然地避免了内存碎片问题（除了 Full GC）。

理解 G1 的关键在于理解 Region 的概念、RSet 的作用以及 Young GC、Concurrent Marking Cycle 和 Mixed GC 这几个核心阶段是如何协同工作的。

### ？？？java的内存分配和回收过程

### ThreadLocal原理和内存泄漏原因

每个Thread实例维护一个map，map中的key是threadlocal实例，value是你往对应threadlocal存的值，而thread对threadmap是强引用，map对value是强引用，而tomcat的线程是不会被销毁的，那么value永远得不到回收。

### ThreadLocal应用场景
