---
title: "Swift 编译流程"
date: 2022-03-31 22:54:49
categories:
  - iOS
tags:
  - Swift
---

## 前言

本文从内存分区、内存对齐、内存模型与方法派发角度介绍了关于 Swift 中的一些知识点。

<!-- more -->

## 内存分区

iOS 应用内存分为 5 个区域，其中“全局区”、“常量区” 和 “代码区”的内存空间在编译时确定，“栈”、“堆”的内存空间在运行时确定。

![iOS应用内存分区](/images/blog/iOS应用内存分区.png)

* 栈区：存储值类型的局部变量，函数参数，其大小有限，连续分配，向低地址拓展；由在运行时系统自动分配和释放内存空间，每一个线程都有其对应的栈。
* 堆区：存储引用类型，不连续的内存区域，向高地址拓展，大小受限于系统中的虚拟内存；由程序员动态创建和释放对象，在运行时分配。
* 全局/静态区：存储未初始化的全局变量和静态变量，即 .bss ；和已初始化的全局变量和静态变量，即 .data。
* 常量区：存储字符串常量
* 代码区：存储程序运行时的代码

## 内存对齐

**为什么需要内存对齐：**

1. 某些硬件不能随意访问任意地址
2. 提高访问效率

**内存对齐原则：**

1. 结构体第一个成员的**偏移量（offset）**为0，以后每个成员相对于结构体首地址的 **offset** 都是**该成员大小与对齐系数中较小值**的整数倍，如有需要编译器会在成员之间加上填充字节。
2. **结构体的总大小**为对齐系数的**整数倍**，如有需要编译器会在最末一个成员之后加上填充字节。

注：结构体可代替为 **class**、**protocol**等任意类型，只是其他类型内存分布与结构体不同。

在 **Swift** 中：

```Swift
MemoryLayout<T>.size // 属性占用大小(与其属性匹配)
MemoryLayout<T>.stride // 实际占用大小
MemoryLayout<T>.alignment // 对齐系数
```

## 内存模型

### **struct**

例：

```swift
struct Foo {
    let a: Int8 = 2   // 1 byte
    let b: Int16 = 4  // 2 bytes
    let c: Int32 = 6  // 4 bytes
    let d: Int = 8    // 8 bytes
    
    func foo() {
    	print("Hello")
    }
}

let foo = Foo()

print(MemoryLayout<Foo>.size)
print("===end===")
```

<img src="/images/blog/image-20220311002137424-6929301.png" alt="image-20211226103015351" style="zoom: 67%;" />

可以看出，**struct** 的内存是连续分布的，但是由于**内存对齐原则**，属性 **a** 所占用内存空间为 **2 byte**，对于实例方法其内部并不会做存储，而是在编译后直接指向方法的地址。

### **class**

**Swift** 与 **Objective-C** 中的 **Class** 类型，为了兼容 **Objective-C** 且具有更多的 **Swift** 特性，在 **Swift** 源码中，其类型为 ```swift_class_t``` 的结构体，继承于 `objc_class`。其内存布局如下图所示：

![image-20220313210720322](/images/blog/image-20220313210720322-7176842.png)

**isa**：与 **Objective-C** 中 **isa** 一样，包含了这个类型的信息，如 父类的 **isa** 信息、是否存在关联对象、以及 **virtual table** 用以方法调用等内容。

**retain count**：记录其引用计数值

### **protocol**

看下面源码

```swift
protocol Drawable { func draw() }
struct Point : Drawable {
    var x: Int
    var y: Int
    func draw() { print("Point") }
}
struct Line : Drawable {
    var x1, y1, x2, y2: Int
    func draw() { print("Line") }
}
class Cricle: Drawable {
    var r: Int = 5
    func draw() { print("SharedLine") }
}
struct Simple {
    var drawable1: Drawable
    var drawable2: Drawable
    var drawable3: Drawable
    func printMemoryLayout() {
        MemoryLayout.size(ofValue: drawable1)
        MemoryLayout.size(ofValue: drawable2)
        MemoryLayout.size(ofValue: drawable3)
    }
}
```

首先在 **Build Setting** 中，将 **Reflection Metadata Level** 改为 **None**，调用 ```printMemoryLayout``` 再进行调试。

```
40
40
40
```



![image-20220313200904959](/images/blog/image-20220313200904959-7173347.png)

**Reflection Metadata Level** 改为 **None**，是防止编译器将反射元数据发送到二进制文件中，会对分析造成干扰。

对于协议类型来说，为了实现语义上的多态，且创建时其内存大小是不固定的，因此引入了新的内存结构进行处理。

可以看到输出内存占用大小全部为 40，且都拥有相同的数据结构。如下图所示：

![image-20220311002137424](/images/blog/image-20220311002137424-6929301.png)

**valueBuffer**：3位，对于空间小于或等于 **valueBuffer** 的值，直接存储在 **valueBuffer** 中；对于空间大于 valueBuffer的值，则会在堆中开辟内存空间，**valueBuffer** 则存储其引用地址。对应 ```drawable1``` 与 ```drawable2```。

**value witness table**：由于每个协议类型的初始化不尽相同，所以每一种类型(上上图的 **metadata** )都会有一个 **value witness table**，用于进行生命周期管理，有 ```alloc```、```copy```、```destruct```、```deallocate``` 等方法。

**protocol witness table**：类似于 **class** 的 **virtual table**，用以存储每个协议类型的方法。每种类型会创造 `PWT` 表，内部包含指针，指向方法具体实现。

## 方法派发

方法派发是指告诉 **CPU** 如何去找到该函数地址并进行调用的过程，在 **Swift** 中分为 3 种派发机制，分别是静态派发、函数表派发与动态派发。那么什么时候会是什么样的方法派发呢？主要有两方面纬度的考量：

### 声明类型

对于不同的声明位置来说，其方法派发的是不同的，若下图所示：

|                   | 类中声明 | 拓展声明 |
| ----------------- | -------- | -------- |
| value type        | static   | static   |
| protocol          | table    | static   |
| class             | table    | static   |
| NSObject SubClass | table    | message  |

**规律：**

值类型都是静态派发

协议和类中的拓展都是静态派发

**NSObject** 拓展采用消息派发，类中声明采用函数表派发

协议中默认实现使用函数表派发

### 关键字

对于某些关键字来说，也能够改变其派发方式：

| 关键字           | 派发方式                |
| ---------------- | ----------------------- |
| final            | static                  |
| dynamic          | Message                 |
| @objc & @nonobjc | 修改 Objective-C 可见性 |
| @inline          | Static                  |

**规律：**

final - 静态派发

dynamic - 消息派发

@objc & @nonobjc - 声明函数能否被 objective-c runtime捕捉到

final @objc - 调用时静态派发，但会将函数注册到 objective-c runtime中

@inline - 直接派发，但如果是 dynamic @inline，则会采用消息派发

结合上两图总结如下：

|            | 直接派发          | 函数表派发 | 消息派发            |
| ---------- | ----------------- | ---------- | ------------------- |
| NSObject   | @nonobjc / final  | 类中声明   | 拓展申明 且 dynamic |
| class      | 拓展声明 且 final | 类中声明   | dynamic             |
| protocol   | 拓展声明          | 类中声明   | @objc               |
| value type | 所有方法          | 无         | 无                  |

### **静态派发**：

指编译时直接跳转到函数的地址，调用速度最快，同时可能经过编译器优化成 **inline**。

在 **Swift** 中，值类型的方法调用与 **final** 修饰的是静态派发。（值类型与 **final** 不支持继承与 **override**）

### **函数表派发：**

为在类中申明过的所有方法生成一个函数指针数组

**virtual table / protocol witness table**

<img src="/images/blog/virtual-dispatch.png" alt="A diagram showing the memory offsets for method1, method2, and method3 in ParentClass and ChildClass." style="zoom:80%;" />

相较于静态派发，速度更慢，需要两次读取地址与一次跳转，同时编译器无优化操作，将自身作为实例作为隐含参数传递到方法中。例如下面一段协议类型的方法调用的 **SIL** 代码如下：

```swift
// drawACopy(drawables:)
sil hidden @$s14ViewController9drawACopy9drawablesyAA8Drawable_p_tF : $@convention(thin) (@in_guaranteed Drawable) -> () {
// %0 "drawables"                                 // users: %2, %1
bb0(%0 : $*Drawable):
  debug_value_addr %0 : $*Drawable, let, name "drawables", argno 1 // id: %1
  %2 = open_existential_addr immutable_access %0 : $*Drawable to $*@opened("F57086CE-A07E-11EC-86EE-ACDE48001122") Drawable // users: %4, %4, %3
  %3 = witness_method $@opened("F57086CE-A07E-11EC-86EE-ACDE48001122") Drawable, #Drawable.draw : <Self where Self : Drawable> (Self) -> () -> (), %2 : $*@opened("F57086CE-A07E-11EC-86EE-ACDE48001122") Drawable : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@in_guaranteed τ_0_0) -> () // type-defs: %2; user: %4
  %4 = apply %3<@opened("F57086CE-A07E-11EC-86EE-ACDE48001122") Drawable>(%2) : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@in_guaranteed τ_0_0) -> () // type-defs: %2
  %5 = tuple ()                                   // user: %6
  return %5 : $()                                 // id: %6
} // end sil function '$s14ViewController9drawACopy9drawablesyAA8Drawable_p_tF'
```

可以看到对于协议类型，会通过 ```open_existential_addr``` 创建一个局部变量 ```%2```，在通过 ```%2``` 找到其对应```witness_method``` - ```%3```，最后再通过 ```%2``` 与作为入参执行方法 （```apply```） ```%3```。

### 动态派发：

与 **Objective-C** 一致，会被翻译成 **objc_send** 这样的代码，会经过 **cache** 查找、通过 **isa** 指针在当前类与父类的 **method_list** 查找、最后到消息转发流程。动态派发的速度最慢，但可功能性最强。

网上关于动态派发的文章很多，这里不再赘述，详情参考：[iOS 消息发送与转发详解](https://juejin.cn/post/6844903575437606920#heading-6)

## 🔗

[iOS-底层原理 24: 内存5大区](https://www.jianshu.com/p/e5a54813b93d)

[Size, Stride, Alignment](https://swiftunboxed.com/internals/size-stride-alignment/)

[Swift 方法调度与内存布局](https://github.com/LeoMobileDeveloper/Blogs/blob/master/Swift/Method%20Dispatch%20and%20Memory%20Layout.md#%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D)

[Swift 中的方法调用（Method Dispatch）（一） - 概述](https://zhuanlan.zhihu.com/p/35696161)

[[译] Swift 中的方法派发](https://juejin.cn/post/6968799729853399053#note3)

[switch-method-dispatch-table](https://www.rightpoint.com/rplabs/switch-method-dispatch-table)

[Exploring Swift Memory Layout](https://academy.realm.io/posts/goto-mike-ash-exploring-swift-memory-layout/)

