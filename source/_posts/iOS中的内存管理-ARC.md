---
title: "iOS中的事件以及事件传递机制"
date: 2023-07-20 09:49:00
categories:
  - iOS
tags:
  - 内存管理
---

## iOS 采用什么内存管理方式

在 iOS 中，采用自动引用计数（ARC，Automatic Reference Counting）机制来进行内存管理，让编译器来帮助内存管理，无需程序员手动键入 retain、release 等代码进行内存管理，取而代之的是由编译器来插入相关内存管理的代码。这一点的好处在于能够降低程序崩溃、内存泄漏等风险的同时，很大程度上也能够减少程序员的工作量。

<!-- more -->

与引用计数相对应的，是垃圾回收（GC，Garbage Collection）机制，JavaScript、Java、Golong 等语言都采用这种机制进行内存管理，它将所有的对象看成一个集合，然后在 GC 循环中定时监测活动对象和非活动对象，及时将这些用不到的非活动对象释放以避免内存泄漏。

相对于 GC 来说，引用计数是局部的，在运行时无需额外开销，同时其内存回收是平稳、时机明确的，没有被持有的对象会被立即释放，但同时也引入了循环引用导致的内存泄漏这种新的内存管理问题。

## 内存管理的相关操作

当生成新的对象时，其引用计数为 1，当有其他指针持有这个对象时，其引用计数加 1，当其他指针释放这个对象时，其引用计数减 1，当这个对象的引用计数变为 0 时，对象会被废弃。上文中出现的“生成”、“持有”、“释放”、“废弃”对应的 Objective-C 的方法如下表：

| 对象操作       | Objective-C 方法                      |
| :------------- | ------------------------------------- |
| 生成并持有对象 | alloc / new / copy / mutableCopy 方法 |
| 持有对象       | retain 方法                           |
| 释放对象       | release 方法                          |
| 废弃对象       | dealloc 方法                          |

在 MRC 机制下，由于需要程序员手动插入 retain、release 代码，无需考虑引用计数，按如下思考方式进行代码编写就可以管理好内存：

* 自己生成的对象，自己所持有。
* 非自己生成的对象，自己也能持有。
* 不再需要自己持有的对象时释放。
* 非自己持有的对象无法释放。

但 ARC 中，由于交给了编译器进行内存管理，每个对象都是相当于强引用，但这会产生循环引用的问题，由于引用计数不能达到 0，导致对象无法被释放，因此引入了所有权修饰符来解决这个问题：

* `__strong`：对象的默认所有权修饰符，它表示对对象的“强引用”，在超出其作用域时，强引用失效。

* `__weak`：使用 `__weak ` 修饰的对象不会持有对象，因此不会使对象的引用计数加 1，同时弱引用指向的对象被废弃时，弱引用会指向 nil，利用这一点可以来解决循环引用的场景。
* `__unsafe_unretained`：与 `__weak` 类似，`__unsafe_unretained` 修饰的对象不会持有对象，但在指向的对象被废弃时，不会指向 nil，会变成野指针。使用 `__unsafe_unretained` 时需要明确清楚它的生命周期小于或者等于被指向对象的生命周期，它与 Swift 中的 `unowned` 类似，同时它的效率也比 `__weak` 高。
* `__autoreleasing`：使用 `__autoreleaseing` 修饰的对象会被注册到 AutoReleasePool 中，会延迟到 AutoReleasePool 被销毁时才会调用对象的 release 方法，这会延长对象的生命周期。但由于编译器优化的原因，实际用到的地方是很少的。

> 循环引用：对象间的相互强引用产生有向环，导致有向环中的每一个节点都无法被释放（没有对象的引用计数为 0），进而会导致内存泄漏。
>
> ```objective-c
> // test 持有自身导致的循环引用
> id test = [[Test alloc] init];
> [test setObject: test];
> ```

## alloc & retain & release & dealloc 源码探究

### `alloc`

对象内存分配的主要逻辑集中在 `callAlloc` 与 `_class_createInstanceFromZone` 方法里。为了提升效率，其在 `callAlloc` 里判断了 `hasCustomAWZ`（自定义的 allocWithZone 方法），没有执行 `_objc_rootAllocWithZone`，有则进入消息派发 `allocWithZone`。NSZone 已被系统忽略，由于历史遗留原因才得以保留，因此虽然有许多跳转流程，但最终都会指向 `_class_createInstanceFromZone` 方法。

```objc
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif
  
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}
```

`_class_createInstanceFromZone` 主要做了三件事，获取对象内存占用大小并分配、设置 `isa` 指针、以及执行 C++ 构造方法。

获取内存占用并分配：`cls->instanceSize(extraBytes);` 通过在 `cache` 或者 `ro()->instanceSize ` (编译时确定)获取占用内存，并进行内存对齐，最后调用 `calloc` 方法进行内存分配。

设置 `isa` 指针：设置如 has_cxx_dtor（是否有 C++ 析构函数）、shiftcls（类对象或者元类对象的地址）、extra_rc（引用计数） 等信息。

执行 C++ 构造方法：从基类开始向下递归执行 C++ 的构造函数。

```objc
// 简化后代码
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    // 获取实例变量内存占用大小
    size = cls->instanceSize(extraBytes);

    id obj;
    obj = (id)calloc(1, size);
 	  
    // 设置 isa 指针
    obj->initInstanceIsa(cls, hasCxxDtor);

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    // 执行 C++ 构造函数
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

> NSZone：它是为了防止内存碎片化而引入的结构，对内存分配的区域本身进行多重化管理，根据使用对象的目的、对象的大小分配内存，从而提高了内存管理的效率。但目前运行时系统中的内存管理本身已极具效率，使用 NSZone 来管理反而会引起内存使用效率低下以及源代码复杂化等问题，因此运行时只是简单地忽略了 NSZone 的概念。

#### retain

retain 会将对象的引用计数 + 1，其主要逻辑主要集中在 rootRetain 方法中，引用计数一般会存储在两个地方，首先是 isa 指针的 extra_rc 域中，若有溢出则会将一半的引用计数值存储到 SideTable 中。

```objective-c
// 简化后代码
ALWAYS_INLINE id
objc_object::rootRetain(bool tryRetain, objc_object::RRVariant variant)
{
    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    oldisa = LoadExclusive(&isa.bits); // 加载 isa 指针

    do {
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
	      
        // 若 newisa.extra_rc++ 溢出，再调用一次，将 variant 设置为 RRVariant::Full
        if (slowpath(carry)) {
            if (variant != RRVariant::Full) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }
            // 留下一半的引用计数值，并将另一半拷贝到 SideTable中
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, &oldisa.bits, newisa.bits)));

    if (variant == RRVariant::Full) {
        if (slowpath(transcribeToSideTable)) {
            // 拷贝一半的引用计数值到 SideTable 中
            sidetable_addExtraRC_nolock(RC_HALF);
        }

        if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    }

    return (id)this;
}
```

关于 SideTable，其本身是全局的 SideTables() 的 value 元素，key 则是通过对象指针地址的偏移映射，找到属于对象的 SideTable，再通过对象的地址，获得属于对象的引用计数表。当 SideTable 中对象的引用计数溢出时，会将标志位（SIDE_TABLE_RC_PINNED）置为 1。

```c++
// 通过对象的地址偏移与 StripeCount 大小映射到属于对象的 SideTable 的下标
static unsigned int indexForPointer(const void *p) {
    uintptr_t addr = reinterpret_cast<uintptr_t>(p);
    return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
// 加入额外的引用计数到 SideTable 中
bool 
objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}
```

#### release

与 retain 对应，release 方法的主要逻辑集中在 rootRelease 中，它会将对象的引用计数 - 1，如果发生下溢（underflow），则会从 SideTable 中借取一半引用计数值，若引用计数为 0 则销毁对象。

```c++
// 简化后代码
ALWAYS_INLINE bool
objc_object::rootRelease(bool performDealloc, objc_object::RRVariant variant)
{
    if (slowpath(isTaggedPointer())) return false;

    bool sideTableLocked = false;

    isa_t newisa, oldisa;

    oldisa = LoadExclusive(&isa.bits);
retry:
    do {
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) { // 发生下溢
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, &oldisa.bits, newisa.bits)));
    
    // 销毁对象
    if (slowpath(newisa.isDeallocating()))
        goto deallocate;

    return false;

 underflow:
    newisa = oldisa;

    if (slowpath(newisa.has_sidetable_rc)) {
        if (variant != RRVariant::Full) {
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc);
        }
      
        auto borrow = sidetable_subExtraRC_nolock(RC_HALF);
				bool emptySideTable = borrow.remaining == 0;
      
        if (borrow.borrowed > 0) {
            newisa.extra_rc = borrow.borrowed - 1;
            newisa.has_sidetable_rc = !emptySideTable;
        }
        if (emptySideTable)
                sidetable_clearExtraRC_nolock();
    }

deallocate:
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
    }
    return true;
}
```

#### dealloc

对象内存释放的主要逻辑集中在 `rootDealloc` 与 `objc_destructInstance` 方法里。其中，taggedPointer 对象无需释放(其在栈上存储)、若同时满足以下条件则直接 `free`，否则进入 `objc_destructInstance`。

`dealloc` 在执行最终释放操作（release）的那个线程中被执行，而不是主线程；

在 `dealloc` 也不要使用 `__weak __typeof(self)weak_self = self` 这样的代码，这是因为在 weak 注册时会判断其是否处于 `deallocating` 状态，会产生崩溃。

1. `isa.nonpointer` 为 1，即存在 `ISA_BITFIELD` 位域数据
2. 此对象不是其他对象的弱引用对象
3. 此对象没有关联对象
4. 没有 C++ 的析构函数
5. 不存在 SideTable 记录引用计数

```objc
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?
    if (fastpath(isa.nonpointer                     &&
                 !isa.weakly_referenced             &&
                 !isa.has_assoc                     &&
#if ISA_HAS_CXX_DTOR_BIT
                 !isa.has_cxx_dtor                  &&
#else
                 !isa.getClass(false)->hasCxxDtor() &&
#endif
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

`objc_destructInstance` 则是清理对象关联的资源，C++ 的析构函数、关联对象、SideTable 中的弱引用指针和引用计数表，之后再 `free`。在 C++ 析构函数中，会遍历其所有的实例变量，形如 `objc_storeStrong(&ivar, null)` 调用，则会对所有的实例变量进行 release，并将其置为 nil。同时经由编译器插入类似 `[super dealloc]`，则会实现了由子类开始遍历到基类的 `dealloc`。

```objc
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj, /*deallocating*/true);
        obj->clearDeallocating();
    }
    return obj;
}
```

## 不要在 init 和 dealloc 中调用 accessor 方法

在 `init`、和 `dealloc` 中，这个阶段处于未完全初始化成功或者正在废弃阶段，同时由于继承、多态特性，本来目的到调用父类的方法调用到了子类，就可能会出现错误，例如在 init 中：

```objc
@interface BaseClass : NSObject
@property(nonatomic) NSString* info;
@end

@implementation BaseClass
- (instancetype)init {  
    if ([super init]) {
        self.info = @"baseInfo"; 
    } 
    return self;
}
@end

@interface SubClass : BaseClass
@end
@interface SubClass ()
@property (nonatomic) NSString* subInfo;
@end

@implementation SubClass
- (instancetype)init {
     if (self = [super init]) {
         self.subInfo = @"subInfo"; 
    } 
    return self;
}

- (void)setInfo:(NSString *)info {
    [super setInfo:info]; 
    NSString* copyString = [NSString stringWithString:self.subInfo]; NSLog(@"%@",copyString);
}
@end
```

这时候创建一个 SubClass 实例变量，由于继承、多态特性会调用到子类的 `setInfo`，子类的 accessor 实现的代码完全以子类已完全初始化的前提编写的，此时的 `subInfo` 还并未完全初始化，进而会造成崩溃。

这一点与 Swift 中，构造器在第一阶段构造完成之前，不能调用任何实例方法，不能读取任何实例属性的值，不能引用 `self` 作为一个值类似。

第一阶段：类中的每个存储型属性赋一个初始值。

第二阶段：给每个类一次机会，在新实例准备使用之前进一步自定义它们的存储型属性。

两段式构造过程的使用让构造过程更安全，同时在整个类层级结构中给予了每个类完全的灵活性。两段式构造过程可以防止属性值在初始化之前被访问，也可以防止属性被另外一个构造器意外地赋予不同的值。

> 在《Effective Objective-C 2.0 编写高质量iOS与OS X代码的有效52个有效方法》中也指出：
>
> 在dealloc里不要调用属性的存取方法，因为有人可能会覆写这些方法，并于其中做一些无法再回收阶段安全执行的操作。此外，属性可能正处于“键值观察”(Key-Value Observation，KVO)机制的监控之下，该属性的观察者(Observer)可能会在属性值改变时“保留”或使用这个即将回收的对象。这种做法会令运行期系统的状态完全失调，从而导致一些莫名其妙的错误。

## 🔗

[1] [Objective-C 高级编程 iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)

[2] [ARC下dealloc过程及.cxxdestruct的探究](http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)

[3] [黑箱中的 retain 和 release](https://draveness.me/rr/)

[4] [为什么不能在init和dealloc函数中使用accessor方法](https://cloud.tencent.com/developer/article/1143323)

[5] [Swift构造过程](https://swift.bootcss.com/02_language_guide/14_Initialization#memberwise-initializers-for-structure-types)
