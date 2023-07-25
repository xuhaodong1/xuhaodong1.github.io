---
title: "iOS中的内存管理｜AutoreleasePool"
date: 2023-07-25 09:49:00
categories:
  - iOS
tags:
  - 内存管理
---

## 基本使用

在 ARC 中，使用 AutoreleasePool 非常简单，只需形如以下方式调用即可，编译器会将块中的对象插入类似如 `[obj autorelease];` 一样的代码，在超出 AutoreleasePool 块作用域后会自动调用对象的 `release` 方法，这能延迟对象的释放。但一般来说，并不需要显式使用 `@autoreleasepool{ }`，这是因为在主线程 RunLoop 的每个周期中都会自动进行自动释放池的创建和销毁。

<!-- more -->

```objc
@autoreleasepool {
		
}
```

## 什么对象会纳入到 AutoreleasePool 中？

除了显式加入 `__autoreleasing` 所有权修饰对象外，还有些对象会直接被隐式纳入 AutoReleasePool 管理。

* 非自己生成并持有的对象

编译器会检查方法名是否以 `alloc`、`new`、`copy`、`mutableCopy` 开始，如果不是则自动将其返回值注册到 AutoreleasePool 中。ARC 通过命名约定将内存管理标准化，本来 ARC 也可以直接舍弃 autorelease 这个概念，并且规定，所有从方法中返回的对象其引用计数比预期的多 1，但这样做就破坏了向后兼容性（backward compatibility），无法与不使用 ARC 的代码兼容。

不过利用 `clang attribute` ，如 `- (id)allocObject __attribute__((objc_method_family(none)))`，会将`allocObject` 这个方法当做普通方法返回对象看待。

在普通方法返回对象后，可能会将对象 `retain` 一次以进行强持有。例如以下的代码会被翻译为：

```objc
EOCPerson _myPerson = [EOCPerson personWithName: @"Bob Smith"]; // 会调用 `autorelease`

// 被翻译为
EOCPerson *tmp = [EOCPerson personWithName: @"Bob Smith"];
_myPerson = [tmp retain];
```

其中 `autorelease` 和 `retain` 结对出现，是多余的，为了提升性能可以将其删除。于是编译器在被调用方采用 `objc_retainAutoreleaseReturnValue` 方法取代 `autorelease` ，会检查即将执行的那段代码是否会执行 `retain` 操作，若有则会在线程局部存储（TLS，Thread Local Storage）中存储这个对象，不执行 `autorelease ` 操作；在调用方采用 `objc_retainAutoreleasedReturnValue` 方法取代 `retain` ，会检测 TLS 是否存了这个对象，若有则直接返回这个对象，不进行 `retain` 操作。

* id 的指针或对象的指针

id 的指针（`id **`）和对象的指针（`NSError **`），如果没有显式指定，会自动加上关键字 `__autoreleasing`，注册到 AutoreleasePool 中。

* 关于 `__weak1` 修饰的对象

在 LLVM 8.0 之前的编译器，关键字 `__weak` 修饰的对象，会自动注册到 AutoreleasePool 中；在 LLVM 8.0 以及之后的编译器，则会直接调用 `release` 方法。

## 什么时候显式使用 @autoreleasepool？

* CLI（Command-line interface）程序

在 Cocoa 框架中由于有 RunLoop 机制的原因，每个周期都会进行自动释放池的创建与释放，但在 CLI 中意味着不会定期清理内存，因此需要更多关注。

* 循环中使生成大量局部变量

再循环过程中产生了大量的局部变量，会导致内存峰值过高，因此手动加入 `@autoreleasepool` 可以降低内存使用峰值。

虽然只有 Autorelease 对象（也即上文提到的哪些对象会纳入 AutoreleasePool 管理）会纳入AutoreleasePool 管理，但这可以利用块机制，让编译器将在块末尾自动插入 `release` 代码。

```swift
func loadBigData() {
    if let path = NSBundle.mainBundle().pathForResource("big", ofType: "jpg") {
        for i in 1...10000 {
            autoreleasepool {
                let data = NSData.dataWithContentsOfFile(path, options: nil, error: nil)
                let person = Person("nihao") //也会释放
                NSThread.sleepForTimeInterval(0.5)
            }
        }
    }
}
```

* 常驻线程

主线程的 RunLoop 会在每个周期进行自动释放池的创建与释放，子线程则不会，同时子线程也不一定会有 RunLoop。但只要是 Autorelease 对象，就会自动纳入 AutoreleasePool 管理，每个线程都会自动创建并管理自己的自动释放池，等到线程销毁的时候释放。但常驻线程中的对象因线程无法销毁迟迟得不到释放，这就需要手动添加 AutoreleasePool：

```swift
class KeepAliveThreadManager {
    private init() {}
    static let shared = KeepAliveThreadManager()

    private(set) var thread: Thread?

    /// 开启常驻线程
    public func start() {
        if thread != nil, thread!.isExecuting {
            return
        }
        thread = Thread {
            autoreleasepool {
                let currentRunLoop = RunLoop.current
                // 如果想要加对该RunLoop的状态观察，需要在获取后添加，而不是等到启动之后再添加，
                currentRunLoop.add(Port(), forMode: .common)
                currentRunLoop.run()
            }
        }
        thread?.start()
    }

    /// 关闭常驻线程
    public func end() {
        thread?.cancel()
        thread = nil
    }
}

class Test: NSObject {
    func test() {
        if let thread = KeepAliveThreadManager.shared.thread {
            perform(#selector(task), on: thread, with: nil, waitUntilDone: false)
        }
    }

    @objc
    func task() {
        /// 在任务外加一层 autoreleasepool
        autoreleasepool {

        }
    }
}
```

## 与 RunLoop 的关系

主线程在 RunLoop 中注册了两个 Observer，其回调都是 `_wrapRunLoopWithAutoreleasePoolHandler`。

* 第一个 Observer 监测 `Entry` 事件（即将进入 RunLoop）

回调内部会调用 `_objc_autoreleasePoolPush` 创建自动释放池，其 order = -214748364，优先级最高，保证创建自动释放池在其他所有回调之前。

* 第二个 Observer 监测 `BeforeWaiting` 及 `Exit` 事件

`BeforeWaiting` 时调用 `_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的自动释放池并创建新的自动释放池。

`Exit` 时调用 `_objc_autoreleasePoolPop` 来释放自动释放池，其 order = 2147483647，优先级最低，保证其其它回调都在释放自动释放池之前。

## AutoReleasePool 源码

形如 `_objc_autoreleasePoolPush` 、 `_objc_autoreleasePoolPush` 和 `objc_autorelease` 其内部都是调用 `AutoreleasePoolPage` 的相关静态方法。因此其源码主要是对 `AutoreleasePoolPage` 的探索。

以下参考的源码为 objc4-838。

```c++
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
} 

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}

__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    ASSERT(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}
```

### AutoreleasePoolPage 的数据结构

`AutoreleasePoolPage` 继承自 `AutoreleasePoolPageData`，`AutoreleasePoolPageData` 存储了自动释放池实例对象的信息，而 `AutoreleasePoolPage` 里则存储了全局所有的自动释放池的所需信息，因此其属性类型也都是 `static const`。

```c++
struct AutoreleasePoolPageData
{
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
    // 用来优化同一对象多次加入 AutoreleasePoolPage，只需记录其地址与数量，无需重复递增，节省空间
    struct AutoreleasePoolEntry {
        uintptr_t ptr: 48;
        uintptr_t count: 16;

        static const uintptr_t maxCount = 65535; // 2^16 - 1
    };
#endif
  // 对当前 AutoreleasePool 完整性校验
	magic_t const magic;
  // 指向下一个即将产生的 autorelease 对象的位置
	__unsafe_unretained id *next;
  // 关联的线程
	pthread_t const thread;
  // 指向父节点
	AutoreleasePoolPage * const parent;
  // 指向字节点
	AutoreleasePoolPage *child;
  // 链表的深度
	uint32_t const depth;
  // 水位线(DEBUG 使用，用作判断上次和这次的对象增加数量)
	uint32_t hiwat;
};
```

从 `next`、`parent`、`child` 的结构来看，构成了以栈作为节点的双向链表，每个 `AutorleasePoolPage` 的大小为 4096 个字节。

![image-20230724164816710](/images/blog/image-20230724164816710.png)

值的注意的是，引入了 `AutoreleasePoolEntry` 结构，用作将同一对象多次进行 `autorelease` 操作时的优化，这时不会将 `next` 递增，而是将 `AutoreleasePoolEntry` 中 `count` 递增，得以优化内存空间。这里将 `ptr` 和 `count` 指定存储大小，其总大小为 64 字节，与 `id` 类型指针大小相同，使得 `AutoreleasePoolEntry` 和 普通的 `id` 类型可以互操作。

```c++
class AutoreleasePoolPage : private AutoreleasePoolPageData
{
public:
  // 每个 Page 的大小，为 4096 字节(虚拟内存一页的大小)
	static size_t const SIZE = PAGE_MIN_SIZE;
    
private:
  // 关于 AutoreleasePool 的 Key，用来查找存储在 TLS 中线程的 HotPage
	static pthread_key_t const key = AUTORELEASE_POOL_KEY;
  // 释放对象后用 0xA3A3A3A3 占位
	static uint8_t const SCRIBBLE = 0xA3;
  // 存储对象个数
	static size_t const COUNT = SIZE / sizeof(id);
  // 最大错误数量(DEBUG 使用)
  static size_t const MAX_FAULTS = 2;
}
```

关于 `AutoreleasePoolPage` 的静态属性，其中比较重要的：

* `size` 固定为 `4096`，刚好为虚拟内存大小的一页。
* `key` 为 `43`，用作线程局部存储的 `Key`，存储的是线程所属的 `hotPage`，隔离区分其他线程的 `AutoreleasePoolPage`。
* `SCRIBBLE` 为 `0xA3A3A3A3`，在用作占位释放掉的 `next` 指针，标识为未初始化的地址。

### AutoreleasePoolPage::push()

`AutoreleasePoolPage::push() ` 创建一个自动释放池，实际上是插入一个 `POOL_BOUNDARY` （哨兵对象，指向 `nil`）用来表示不同的自动释放池，去除掉 `DEBUG` 调试和一些边界条件，其主要逻辑集中在 `autoreleaseFast` 方法中，根据 `hotPage` 的状态分为三种情况：

关于 `hotPage`，其存储在 TLS 中，表示当前正活跃的 `Page`；与之相对应是 `coldPage`，指向的是双向链表的头节点。

```c++
// 此处的 obj 为 POOL_BOUNDARY
static inline id *autoreleaseFast(id obj) 
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

* 存在 `hotPage` 并且 `hotPage` 未满

这是直接调用 `hotPage` 的 `add` 实例方法，根据宏定义，判断是否利用 `AutoreleasePoolEntry` 类型优化同一对象的多次 `autorelease`，否则直接加入，`next` 指向下一个将要加入 `AutoreleasePoolPage` 的地址。

* 存在 `hotPage` 并且 `hotPage` 已满

从 `hotPage` ，遍历找一个未满的子节点，若没有则创建一个 `AutoreleasePoolPage` ，随后将找到或生成的 `page` 置为 `hotPage` （利用 TLS 机制），并将对象 `add` 到 `hotPage` 中。

* 不存在 `hotPage`

创建一个 `AutoreleasePoolPage` ，将其设置为 `hotPage`，并将对象加入到 `hotPage` 中。

### AutoreleasePoolPage::pop(ctxt)

`pop` 方法需要传入参数，在 `_objc_autoreleasePoolPush` 中是传入 `push` 方法返回的参数，`push` 返回的是存储的哨兵对象的地址，因此传入的也是哨兵对象的地址。

不过该方法也可能在其他地方调用，如果是哨兵对象的地址会销毁整个以哨兵对象开始的单个自动释放池，还有可能销毁整个自动释放池，其方法主要逻辑如下：

```c++
static inline void
pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;
    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        page = hotPage();
        if (!page) {
            // 如果整个自动释放池为空，仅有占位符，以 nil 填充
            return setHotPage(nil);
        }
        // 从头节点开始，移除自动释放池中的所有内容
        page = coldPage();
        token = page->begin();
    } else {
        page = pageForPointer(token);
    }

    stop = (id *)token;

    return popPage<false>(token, page, stop);
}
```

如何通过到 `token`（也即传入的哨兵对象的地址） 查找到所对应的 `AutoreleasePoolPage` ，在 `pageForPointer` 方法中。

这里有个前置条件，`AutoreleasePoolPage` 会通过 `malloc_zone_memalign` 方式分配内存，因此每个 `AutoreleasePoolPage` 的地址都是 `SIZE`（4096）的倍数，也就是地址会进行对齐，在与 `SIZE` 进行取余操作后，得到相对于 `token` 所在的 `AutoreleasePoolPage` 的偏移，相减则就能得到其首地址。

```c++
static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    ASSERT(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}
```

在 `popPage` 方法中，从 `hotPage` 开始，一直进行出栈操作，也即会 `objc_release(obj);`，直到满足栈首的地址与 `stop` 的地址一致，之后会调用 `child` / `child-> child` 的 `kill` 方法，将所有的子节点销毁。

有意思的是，会根据子节点的状态（子节点中已存储大小小于总大小的一半）进行区分，更有意思的是，在进行 `releaseUntil` 方法时，会将每一个子节点清空，里面也会判断 `ASSERT(page->empty());`，因此只会调用 `page->child->kill();`。

不过这里提到一个概念，迟滞现象（hysteresis），wiki 是这样解释的：

> 一系统经过某一输入路径之运作后，即使换回最初的状态时同样的输入值，状态也不能回到其初始。

推测是虽然需要将所有子节点清空，但是系统不同以往了，可能后续需要重新创建子节点，这里先不清空，为后续使用提高效率。

```c++
template<bool allowDebug>
static void
popPage(void *token, AutoreleasePoolPage *page, id *stop)
{
    page->releaseUntil(stop);
    if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

### objc_autorelease

去除掉一些优化条件，如是否是 `taggedPointer` 指针，是否采用 TLS 优化 `autorelease` 步骤 （上文提到），是否是类对象等。

一般最终会指向 `AutoreleasePoolPage::autorelease((id)this);` ，这与 `AutoreleasePoolPage::push()` 的分析情况一致。

```c++
static inline id autorelease(id obj)
{
    ASSERT(!_objc_isTaggedPointerOrNil(obj));
    id *dest __unused = autoreleaseFast(obj);
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
    ASSERT(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  (id)((AutoreleasePoolEntry *)dest)->ptr == obj);
#else
    ASSERT(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
#endif
    return obj;
}
```

## 🔗

[1] [Why is @autoreleasepool still needed with arc](https://stackoverflow.com/questions/9086913/why-is-autoreleasepool-still-needed-with-arc#:~:text=(%2D1)%20%40autoreleasepool%20Forces%20process,footprint%20will%20be%20constantly%20increasing)

[2] [Objective-C 高级编程 iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)

[3] [AutoreleasePool](https://mp.weixin.qq.com/s/Z3MWUxR2SLtmzFZ3e5WzYQ)

[4] [Effective Objective-C 2.0 编写高质量iOS与OS X代码的52个有效方法](https://book.douban.com/subject/25829244/)

[5] [Why __weak object will be added to autorelease pool?](https://stackoverflow.com/questions/40993809/why-weak-object-will-be-added-to-autorelease-pool)

[6] [黑幕后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[7] [iOS AutoreleasePool](https://zhuanlan.zhihu.com/p/323200445)

[8] [nihao_objc4_838](https://github.com/xuhaodong1/objc4_838_source_code)

[9] [迟滞现象](https://zh.wikipedia.org/zh-sg/%E9%81%B2%E6%BB%AF%E7%8F%BE%E8%B1%A1)
