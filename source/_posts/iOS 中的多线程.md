---
title: "iOS中的多线程与线程安全"
date: 2022-04-02 22:45:00
categories:
  - iOS
tags:
  - iOS
  - 多线程
  - GCD
  - 锁
  - NSThread
---

## 前言

在 **iOS** 中有 3 种线程管理安全，分别是 **NSThread**、**GCD** 与 **NSOperation**，不包含几乎不直接使用的 **pthread** 。本文就其使用特点以及重要的 **API** ，以及线程安全等方面进行总结。
<!-- more -->
## NSThread

轻量级线程操作，面向对象，但需手动管理线程声明周期，同时控制不同线程之间执行顺序不是很友好。

### 创建

```swift
init(target: Any, selector: Selector, object argument: Any?) // 需要手动 start()
init(block: @escaping () -> Void) // 需要手动 start()
detachNewThread(_ block: @escaping () -> Void) // 自启动
detachNewThreadSelector(_ selector: Selector, toTarget target: Any, with argument: Any?) // 自启动
performSelector(inBackground aSelector: Selector, with arg: Any?) // 隐式创建
```

### 线程状态控制

```swift
start() // 启动
sleep(until date: Date) // 休眠到指定时间
sleep(forTimeInterval ti: TimeInterval) // 休眠指定时间
exit() // 强制立即退出，不管是否执行完毕，可能会导致异常
cancel() // 不会立即退出，待到处理完线程上下文后退出，可用 isCancel 监听其是否退出
```

### 线程间通信

```swift
performSelector(onMainThread aSelector: Selector, with arg: Any?, waitUntilDone wait: Bool, modes array: [String]?) // 到主线程中执行
performSelector(onMainThread aSelector: Selector, with arg: Any?, waitUntilDone wait: Bool) // 到主线程中执行
perform(_ aSelector: Selector, on thr: Thread, with arg: Any?, waitUntilDone wait: Bool, modes array: [String]?) // 到指定线程中执行
perform(_ aSelector: Selector, on thr: Thread, with arg: Any?, waitUntilDone wait: Bool) // 到指定线程中执行
```

### 线程保活

```swift
let thread = CusThread.init(target: self, selector: #selector(threadAction), object: nil)
@objc func threadAction() {
    let runLoop = RunLoop.current
    runLoop.add(.init(), forMode: .default)
    while !Thread.current.isCancelled {
        runLoop.run(mode: .default, before: Date.init(timeInterval: 2, since: .now)) // 两秒执行一次
        otherAction()
    }	
}
```

**runLoop** 需添加 **port** / **timer** 等内容，否则 **runLoop** 会立即退出
在需退出时手动调用 **cancel**() 方法，防止 **RunLoop** 持有 **Thread** 导致内存泄露问题

## GCD

### 简介

> **Grand Central Dispatch（GCD）** 是 Apple 开发的一个多核编程的较新的解决方法。它主要用于优化应用程序以支持多核处理器以及其他对称多处理系统。它是一个在线程池模式的基础上执行的并发任务。在 Mac OS X 10.6 雪豹中首次推出，也可在 iOS 4 及以上版本使用。

**GCD** 是 **iOS** 推出的多线程解决方案，其更强调「任务块」的概念，忽略了对线程的管理；**GCD** 是由 **C** 语言编写的轻量级线程处理方式，其源码在[这里](https://github.com/apple/swift-corelibs-libdispatch)，在其内部维护了 **pthread** 生成的线程池的概念。

### 任务 & 队列

**GCD** 中的核心概念 「队列」 与 「任务」：

队列即执行任务的等待队列，以先进先出为原则执行队列中的任务，主要分为 **串行队列** 与 **并行队列**

* 串行队列（**Serial Dispatch Queue**）：每次只有一个任务执行。
* 并行队列（**Concurrent Dispatch Queue**）： 可以让多个任务并发执行。

```swift
let queue = DispatchQueue(label: "name") // 串行队列创建 / 默认
let queue = DispatchQueue(label: "name", attributes: .concurrent) // 并行队列创建
```

任务即你放入 **GCD** 中的代码块，分为同步和异步两种

* 同步任务（**sync**）：同步任务会等待队列中前面的任务执行完再执行，在当前线程执行任务，不具备开启线程能力。
* 异步任务（**async**）：异步任务无需等待前面的任务执行完，即可继续执行任务，在新线程中执行任务，具备开启线程能力。

```swift
queue.sync { // 追加同步任务
    print(Thread.current)
} 
queue.async { // 追加异步任务
    print(Thread.current)
} 
```

**主队列**：即主线程所在队列，是一个串行队列，可通过``` DispatchQueue.main ``` 获取

**全局队列**：并行队列，可通过```DispatchQueue.global()``` 获取，同时还可根据任务优先级不同获取不同的全局队列：```DispatchQueue.global(qos: .background)```

由于任务是追加到队列中，因此有 4 种组合方式：

|               | 并发队列                      | 串行队列                      | 主队列                        |
| :------------ | :---------------------------- | :---------------------------- | ----------------------------- |
| 同步（sync）  | 没有开启新线程 / 串行执行任务 | 没有开启新线程 / 串行执行任务 | 没有开启新线程 / 串行执行任务 |
| 异步（async） | 开启新线程 / 并发执行任务     | 开启新线程 / 串行执行任务     | 没有开启新线程 / 串行执行任务 |

### GCD 中的死锁

> 死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

死锁有四个必要条件：互斥 & 请求保持 & 不可剥夺 & 环路条件。

在 **GCD** 中，由于不当使用 **API** 则可能会造成死锁，这个死锁的概念不像上述表示那样，主要是任务间的相互等待导致无法执行任务造成，较为常见的如下：

在主线程中执行

```swift
DispatchQueue.main.sync {
    print(Thread.current) // 同步任务
}
// 当前任务
```

```swift
let queue = DispatchQueue.init(label: "name")
    queue.sync {
        queue.sync {
	          print(Thread.current) // 同步任务
        }  
    } // 当前任务
    print(Thread.current)  
}
```

原因：同步的任务无法开始，需等待当前队列中的任务执行完，而当前任务又因同步的任务导致无法完成。

### 其他重要的**API**

#### DiapatchGroup

在追加多个异步任务后统一进行任务执行，可以采用 **DispatchGroup**，同样的基于手动将任务加入到 **DispatchGroup** 中(**enter** / **leave**)也可在多个网络请求后做同步操作。

```swift
// 将任务加入到DispatchGroup
let group = DispatchGroup()
let queue = DispatchQueue.init(label: "name", attributes: .concurrent)
for i in 1...5 {
    queue.async(group: group, execute: {
      print("------ \(i)")
    })
}
group.notify(queue: queue) {
    print("end")
}

// 手动将任务加入到DispatchGroup
let group = DispatchGroup()
let queue = DispatchQueue.init(label: "name", attributes: .concurrent)
for i in 1...5 {
    group.enter()
    netwrk.api {
        group.leave()
    }
}
group.notify(queue: queue) {
    print("end")
}
```

#### 栅栏函数(barrier)

栅栏任务会在前面任务都执行完后执行，在栅栏任务执行完后才会执行后面追加的任务，在具体场景中，可以用于“读者-写者问题”，即同一时刻可以有多个读者，但同一时刻只能有一个写者，如数据库的读写操作。

<img src="/images/blog/image-20220328172310546.png" alt="image-20220328172310546" style="zoom: 67%;" />

```swift
queue.async(group: nil, qos: .default, flags: .barrier) {
    print("隔离")
}
```

#### 延迟执行(asyncAfter)

延迟指定时间后将延迟任务加入到队列中，需要注意的是可以传递 **DispatchTime** 和 **DispatchWallTime** 这两个时间，前者是基于系统时间，不可被改变，后者为系统时钟，即锁屏界面的时间。

```swift
queue.asyncAfter(deadline: .now() + 1) {
    print("执行")
}
```

#### 信号量(semaphore)

**DispatchSemaphore** 与 操作系统中的信号量一样，都是用来避免数据竞争这一类问题的，在 **iOS** 中常用来控制并发任务执行的最大数量。

* **singal()**：将 信号量 + 1
* **wait()**：若此时信号量 >= 1时，将信号量减 1，然后返回；若信号量 <= 0时，则阻塞线程进行等待。

```swift
let semaphore = DispatchSemaphore(value: 3) // 将并发任务执行数量控制为3
let queue = DispatchQueue.init(label: "name", attributes: .concurrent)
for i in 1...5 {
    semaphore.wait()
    queue.async {
        print(i)
        semaphore.signal()
    }
}
```

#### 调度源(DispatchSource)

**DispatchSource** 用于监听系统的底层对象，比如文件描述符、**Mach** 端口、信号量、内存警告等。主要处理时间如下表：

| 宏定义                         | 说明         |
| :----------------------------- | :----------- |
| DispatchSourceUserDataAdd      | 数据增加     |
| DispatchSourceUserDataOr       | 数据OR       |
| DispatchSourceMachSend         | Mach端口发送 |
| DispatchSourceMachReceive      | Mach端口接收 |
| DispatchSourceMemoryPressure   | 内存情况     |
| DispatchSourceProcess          | 进程事件     |
| DispatchSourceRead             | 读数据       |
| DispatchSourceSignal           | 信号         |
| DispatchSourceTimer            | 定时器       |
| DispatchSourceFileSystemObject | 文件系统变化 |
| DispatchSourceWrite            | 文件写入     |

例如：

**监听内存情况**

```swift
var source = DispatchSource.makeMemoryPressureSource(eventMask: .all, queue: .main)
source.setEventHandler {
    print(source.data)
    // data为枚举值的rawValue, 主要有 normal、warning、critical、all
}
source.activate()
```

**定时器**

```swift
var source: DispatchSourceTimer?
source = DispatchSource.makeTimerSource(flags: .strict, queue: .main)
source?.schedule(deadline: .now() + 1, repeating: 1)
source?.setEventHandler {
    print("定时器触发\(Date.now)")
}
source?.activate()
```

值得注意的是，在使用 **NSTimer** 时，若在滑动页面时，此 **NSTimer** 会失效，需给 **timer** 加入的 **RunLoop** 添加 **commonMode** 模式，若采用  **DispatchSourceTimer**，则不会出现这种情况。

需注意以上代码 ```source``` 不要以局部变量进行测试，否则超出作用域就被释放。

#### DispatchIO

**DispatchIO** 提供一个操作文件描述符的通道，可以利用多线程高效的读取文件。以下是主要流程：

1. 创建 **DispatchIO** 对象，创建通道
2. 进行 **read** / **write** 操作
3. 调用 **close** 关闭通道
4. 进行 **cleanupHandler** 回调处理

```swift
var ioWrite: DispatchIO?
var ioRead: DispatchIO?
let queue = DispatchQueue(label: "com.nihao.serialQueue")
let filePath: NSString = (NSTemporaryDirectory() + "text.txt") as NSString
let fileDescriptor = open(filePath.utf8String!, (O_RDWR | O_CREAT | O_APPEND), (S_IRWXU | S_IRWXG))
let cleanupHandler: (Int32) -> Void = { errorNumber in
    print("最后的回调")
}
ioWrite = DispatchIO(type: .stream, fileDescriptor: fileDescriptor, queue: queue, cleanupHandler: cleanupHandler)
ioRead = DispatchIO(type: .stream, fileDescriptor: fileDescriptor, queue: queue, cleanupHandler: cleanupHandler)
let formattedString = "nihao!!!!!"
let data = Array(formattedString.utf8).withUnsafeBytes {
    DispatchData(bytes: $0)
}
ioWrite?.write(offset: 0, data: data, queue: queue) { done, data, error in }
ioRead?.read(offset: 0, length: Int.max, queue: queue) { done, data, error in }
```

## NSOperation

**NSOperation** 是基于 **GCD** 的面向对象的封装，因此也有「任务 **NSOperation**」和「队列 **NSOperationQueue**」两个概念，同时也增加了 **NSOperation** 之间相互依赖、通过 **KVO** 监听 **NSOperation** 状态、取消任务等特性。

**NSOperation** 是一个形式上的抽象类，系统提供了 **NSInvocationOperation** 和 **NSBlockInvocation** 两个子类，但由于 **NSInvocation** 在 **Swift** 中不可使用，所以在 **Swift** 中 **NSInvocationOperation** 也不可用。同时也可以自定义 **NSOperation**，若仅使用 **NSOperation** 则任务只会在主线程运行，因此需和 **NSOperationQueue** 搭配使用。

**NSOperationQueue** 初始化后就是一个并发队列，它会根据优先级与准备情况调用任务，可通过类属性 ```main``` 获取主队列，主要是通过给队列添加 **operation** 进行操作。值得注意的是，当任务已经被执行或执行已结束后就不能被再次添加进队列，否则会产生 **crash**。下面是一些使用范例：

```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2 //设置最大并发数
let operationA = BlockOperation { () -> Void in
    print("A - \(Thread.current)")
}
let operationB = BlockOperation { () -> Void in
    print("B - \(Thread.current)")
}
operationA.addDependency(operationB) // A 依赖于 B, 当 B 执行后 A 才会执行
queue.addOperation(operationA)
queue.addOperation(operationB)
queue.addBarrierBlock {
    print("我是屏障")
}
queue.addOperation { () -> Void in
    print("2 - \(Thread.current)")
}
```

### 自定义 NSOperation

通常来说非并发 **NSOperation**，自定义 **NSOperation** 不是一件困难的事，只需要重写 ```start()``` 方法，将需要执行的操作写入即可。但如果想要自定义并发 **NSOperation**，需要至少实现以下方法和属性：

* ```start()```
* ```isAsynchronous```
* ```isExecuting```
* ```isFinish```

总的来说需要在执行函数的去维护 **NSOperation** 的一些状态，如果还进行了 **KVO** 监听，还需要去发出 **KVO** 通知以反映值的改变。具体可以参考 **Apple** 的文档 [自定义NSOperation 对象](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW16)

## 线程安全

> 在拥有共享数据的多条线程并行执行的程序中，线程安全的代码会通过同步机制保证各个线程都可以正常且正确的执行，不会出现数据污染等意外情况。

再体会到了多线程的好处之后，需要对数据的安全情况进行保证。线程安全就是为了保证被多线程执行的过程中能够得到正确的结果，即数据不被污染。保证线程安全有「同步」和「非同步」两种方案。「同步」是指在多线程并发访问数据的过程中，保证共享数据在同一时刻只被一个线程使用，例如加锁。「非同步」是指在某些情况下不需要「同步」操作，例如函数本身就不涉及到共享代码，自然也就不需要「同步」去保证数据安全。

### 解决方案

#### 无同步

**可重入代码**（**ReentrantCode**）：可以在这个函数执行的任何时刻中断它，转入 **OS **调度下去执行另外一段代码，而返回控制时不会出现什么错误，这意味着它除了使用自己栈上的变量外不依赖于其他任何变量。这种情况每次执行结果都一样，且不会依赖共享变量，在无同步情况下保证了线程安全。

**线程本地存储**：若线程中需要的数据必须与其他线程共享，尝试判断这个共享的数据能否保证只在同一线程执行，如果可以，则可以对该线程创建一份共享变量的副本，这样也可以实现无同步保证线程安全。例如：

```swift
class Person: NSCopying {
    func copy(with zone: NSZone? = nil) -> Any {
        return Person(-1, name: "")
    }

    var age: Int
    var name: String
    init(_ age: Int, name: String) {
        self.age = age
        self.name = name
    }
}

var person = Person.init(2, name: "") // 全局变量

let queue = DispatchQueue.init(label: "name")
for i in 0...100 {
    queue.async {
        var currPerson = self.person.copy()
    }
}
```

若线程中每次都需要访问 **person**，且在后面不需要同步回原始 **person**，仅在当前线程中操作，也可保证线程安全。

#### 同步

**互斥同步**：也称非阻塞同步，是指调用返回结果前，当前线程会被挂起进入阻塞状态，只有在得到结果后才继续，是一种悲观的同步策略。在 **iOS** 中以主要以互斥锁方式体现，在获取互斥锁失败后，会进入阻塞状态，等待锁被释放以被唤醒。

**非阻塞同步**：是指在不能得到结果前，当前线程不会进入阻塞状态，是一种乐观的同步策略。在 **iOS** 中以主要以自旋锁的方式体现，在获取锁失败后不会进入阻塞状态，而是不断尝试获取锁，锁被释放，因为不涉及线程状态切换，所以效率高于互斥锁。

除了锁之外还有的同步工具，如 **Atomic Operations**、**Memory Barries**、**Volatile Variables**，下面进行简要介绍：

* **Atomic Operations**：原子操作是一种简单的同步形式，适用于简单的数据结构，它不会阻塞竞争线程；**OS X** 和 **iOS** 包含了许多对 32 位和 64 位值执行基本数学和逻辑运算操作。如```atomic_fetch_add```、```atomic_exchange```等，具体可[参考](https://en.cppreference.com/w/c/atomic)。
* **Memory Barries**：在单线程中，由于硬件会执行必要的记录，以确保程序的内存操作的执行顺序就像代码顺序一样；但是在多线程中，由于编译器为了优化经常对汇编级指令进行重排，就可能会导致产生潜在的错误。内存屏障是一种非阻塞同步工具，用于确保内存操作以正确顺序发生。例如：

```swift
// thread1:
while f == 0 {}
print(x)

// thread2:
x = 5
f = 1
```

并非每次都打印数字 **5**，如果 **thread2** 乱序执行，先执行 ```f = 1```，后执行 ```x = 5```，则可能会出现意外的值，因此引入了内存屏障，在需要确保执行顺序中插入 ```OSMemoryBarrier```，具体可[参考](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/OSMemoryBarrier.3.html#//apple_ref/doc/man/3/OSMemoryBarrier)。

* **Volatile Variables**：声明为 **volatile** 的变量不会被优化。例如变量被编译器优化，被放置于寄存器中并读取，这就有潜在读取风险，而 **volatile** 阻止了这种优化，每次都会从内存中读取变量的当前值。

内存屏障 和 **volatile **变量都会减少编译器的优化次数，因此只需要在确保正确的地方使用它们。在 **GCD**、 **NSOperation** 中有着许多内存屏障代码，让我们能遇上这种情况微乎其微。

## iOS 中的锁

在 **iOS** 中，实现线程安全一般以锁来进行线程同步，下面对主要的几种锁进行简要介绍以及性能对比。

### @synchronized

```objective-c
@synchronized (obj) {
    // 需要同步的代码
  	NSLog(@"nihao")!
}
```

**@synchronized** 是一个递归锁，其实现采用了 **recursive_mutex_t** ，递归锁即在同一个线程中获取锁多次，只能传递一个 **NSObject** 对象，在 **Swift** 中将此语法移除，将其还原为 **C++** 后可以发现其源码类似如下：

```objective-c
objc_sync_enter(_sync_obj);
// 需同步的代码
NSLog(@"nihao")!
objc_sync_exit(_sync_obj);
```

因此在 **Swift** 中可以利用这两个函数模拟出 **@synchronized**

```swift
func synchronized(lock: AnyObject, closure: () -> Void) {
    objc_sync_enter(lock)
    closure()
    objc_sync_exit(lock)
}
```

### NSLock & NSRecursiveLock

```swift
let lock = NSLock()
lock.lock()
// 需同步的代码
lock.unlock()

let recursiveLock = NSRecursiveLock()
recursiveLock.lock()
// 需同步的代码
recursiveLock.unlock()
```

**NSLock** 和 **NSRecursiveLock** 在使用上一致，**NSLock** 是一个互斥锁，而 **NSRecursiveLock** 是一个递归锁，都对 **pthread_mutex** 的封装，在使用场景中需考虑是否在同一线程中多次加锁。

### NSCondition & NSConditionLock

**NSCondition** 基于 ```pthread_mutex``` 实现，是一个条件锁，其内部维护了一个锁以及一个线程检查器：锁主要是为了同步临界区；线程检查器主要是根据条件决定是否继续运行。

* ```wait()```：让当前线程处于等待中
* ```singal()```：通知某一个线程从阻塞状态恢复到就绪状态
* ```broadcast()```：通知其他所有线程从阻塞状态恢复到就绪状态

比较常见的例子如生产者消费者模型：

```swift
var condition = NSCondition() // 锁
var money = 5 // 共享变量
// thread1
func consume() {
    condition.lock()
    while money == 0 {
        condition.wait()
    }
    money -= 1
    condition.unlock()
}
// thread2
func product() {
    condition.lock()
    money += 1
    condition.wait()
    condition.unlock()
}
```

**NSConditionLock** 定义了一个互斥锁，可以用于特定的值锁定与解决。其与 **NSCondition** 的行为有些类似，上面的代码可以转换为：

```swift
var condition = NSConditionLock(condition: 0)
var money = 0
// 0表示无数据 1表示有数据
func consume() {
    condition.lock(whenCondition: 1)
    money -= 1
    condition.unlock(withCondition: money == 0 ? 0 : 1) 
}

func product() {
    condition.lock()
    money += 1
    condition.unlock(withCondition: 1)
}
```

### NSDistrubutedLock

**NSDistrubutedLock** 是一个分布式锁，通常在多个主机上的多个应用程序用来限制对某些共享资源的访问，比如文件，它由文件系统项(例如文件或目录)实现，不过由于 **iOS** 应用的沙盒机制，并未有相应 **API**，在 **OS X** 中可以使用。

### DispatchSemaphore

上文有对 **DispatchSemaphore** 做介绍，这里就不赘述。

### OSSpinLock

**OSSpinLock** 是自选锁，但由于 **iOS** 系统中线程可以拥有不同的优先级，可能会产生优先级反转问题。具体来说，在低优先级的线程获得锁并访问共享资源，此时高优先级线程也尝试获取，由于 **OSSpinLock** 是自选锁，它会进入忙等状态并占用大量 **CPU**，进而导致低优先级线程无法与高优先级线程抢占 **CPU**，进而导致任务迟迟无法完成、无法释放 **lock**，因此 **OSSpinLock** 已经被弃用了，使用 **os_unfair_lock** 代替，这也是个互斥锁。

```objective-c
OSSpinLockLock(&spinlock) // 获取锁，线程一直忙等待。阻塞当前线程执行。
OSSpinLockTry(&spinlock) // 尝试获取锁，返回bool。当前线程锁失败，也可以继续其它任务，不阻塞当前线程。
OSSpinLockUnlock(&spinlock) // 解锁，参数是OSSpinLock地址。
```

### atomic

**Objective-C** 中属性中的关键字，会对属性的存值与取值进行加锁处理。它是基于```os_unfair_lock```进行实现，上文提到过，这是一个互斥锁，它不能保证线程安全，只能保证存取值的安全性。

### 各种锁的性能比较

图片截取自 [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/#more-41952)，做一个小的推测：

先讨论锁，然后再分析信号量。

**OSSpinLock** 由于自选锁的特性不会线程状态切换因此排在第一；

之后的锁都是基于 **POSIX thread** 的相关线程 **API**进行封装，性能根据封装强度不同而有所不同，比如互斥锁性能好于递归锁，同时也好于条件锁；

信号量其实与锁类似，但```pthread_mutex```支持多种类型，因此会有额外的判断，就造成了效率略低原因。

但这些锁虽性能有所差异，但都差距不大，在编码过程中还因考虑具体的场景和代码健壮性等方面。

<img src="/images/blog/lock_benchmark.png" alt="lock_benchmark" style="zoom:75%;" />

## 总结

本文就 **iOS** 中多线程这一点，概述了 **NSThread**、**GCD**、**NSOperation** 这三种多线程使用说明和它们的特点，以及关于线程安全介绍了一些同步手段和关于锁的一些使用。就本人而言，在平时工作中使用 **GCD** 较多，它是多线程编程的核心，应该更多的关注它，同时关于它们的源码，笔者由于水平有限，不能做出较为有理解的看法，就不做过多探讨，希望在后续过程中能够加强这方面的学习。

## 🔗

* 《**Objective-C** 高级编程 - **iOS** 与 **OS X** 多线程和内存管理》
* [细说GCD(Grand Central Dispatch)](https://ming1016.github.io/2016/01/13/how-to-use-gcd/)
* [关于 @synchronized，这儿比你想知道的还多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)
* [Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1)

* [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/#more-41952)
