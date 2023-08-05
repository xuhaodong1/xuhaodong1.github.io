---
title: "iOS中的内存管理｜weak"
date: 2023-08-06 09:49:00
categories:
  - iOS
tags:
  - 内存管理
---

## 循环引用原因

在对象图中经常会出现一种情况，就是几个对象都以某种方式相互引用，从而形成“环”（cycle），同时由于 iOS 中采用引用计数内存管理模型，所以这种情况通常会导致内存泄漏，因为最后没有别的东西会引用环中的对象。这样的话，环里的对象就无法为外界所访问，但对象之间尚有引用，这些引用使得他们都能继续存活下去，而不会为系统回收。例如：
<!-- more -->
```swift
class Teacher {
    var student: Student?
}

class Student {
    var teacher: Teacher?
}

let teacher = Teacher()
let student = Student()
teacher.student = student
student.teacher = teacher
```

从上面代码可以看出，`teacher` 与 `student` 相互持有，形成保留环，如果 `teacher` 和 `student` 对象无法被外界访问到，整个保留环就泄漏了。但如果还能被外界所访问，外界还能手动破除“环”以避免循环引用，比如 `student.teacher = nil`。

在垃圾回收机制中，若采用 Root Tracing 算法，就能检测到这些保留环，从而回收掉这些对象。它通过一系列名为 “GCRoots” 的对象作为起始点，从这个节点向下搜索，搜索走过的路径称为 `ReferenceChain`，当一个对象与 `GCRoots` 没有任何 `ReferenceChain` 相连时，就证明这个对象不可用，可以被回收。

在 iOS 中，提供了 `weak` 来帮助我们解决循环引用这种内存管理问题，使用 `weak` 修饰的对象不会持有对象，因此不会使对象的引用计数加 1，同时弱引用指向的对象被废弃时，弱引用指针会指向 `nil`。在上述例子中，例如将 `Student` 类中的 `teacher` 加上 `weak` 所有权修饰符，就可以避免强引用“环”的出现。

## 实现 weak 的源码

在类似这样使用 `__weak id weakObj = object;` 弱引用所有权修饰符时，编译器会将其转换成类似 `objc_initWeak(&weakObj, 0);`这样的代码。同样的，还有销毁 `weak` 指针，`objc_destroyWeak`。

以下参考的源码为 objc4-838。

### SideTable 的数据结构

`SideTable` 中存储了对象的引用计数以及所关联的弱引用指针，它是 `SideTables()` 这样一个全局哈希表的 `value`，其数据结构如下图所示：

![image-20230731154644415](/images/blog/image-20230731154644415-0789607.png)

关于 `SideTables()`，它是 `SideTablesMap` 的封装函数，其实际类型为 `StripedMap<SideTable>`，它是通过 `SideTablesMap.get()` 获取，而实际的 `SideTablesMap` 则又被 `ExplicitInit` 所封装；为什么需要 `ExplicitInit` 呢？在官方注释可以发现答案：由于 C++ 的静态初始化器在 Runtime 初始化之后，而在 Runtime 初始化时需要用到这个对象，因此需要显式初始化。`StripedMap` 是一个哈希表，通过对象的地址偏移与 `StripeCount` 大小映射到属于对象的 `SideTable` 的下标，这样可以将对象放在不同的 `SideTable` 存储，避免同时访问提升访问效率。

```c++
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;

static StripedMap<SideTable>& SideTables() {
    return SideTablesMap.get();
}

static unsigned int indexForPointer(const void *p) {
    uintptr_t addr = reinterpret_cast<uintptr_t>(p);
    return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```

`SideTable` 中存储了对象的引用计数和弱引用指针，分别是 `refcnts` 和 `weak_table`，`weak_table` 本身是一个简易的哈希表，它的 `weak_entries` 是存储具体对象弱引用指针的数组，`size_entries` 表明数组的大小，`mask` 用于哈希映射，`max_hash_displacement` 则表示数组中对象的最大哈希冲突次数，采用线性探测法解决哈希冲突。其具体 `key-value` 映射逻辑如下：

```c++
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```

在具体的 `entry` 中，有两种方式存储弱引用指针，弱引用指针总个数 <= 4 采用数组，大于 4 采用哈希表。在采用哈希表的实现中，`out_of_line_ness` 表示是否超出使用数组的大小范围（4），`num_refs` 表示弱引用指针个数，`mask` 用于哈希映射，`max_hash_displacement` 表示数组中对象最大哈希冲突的个数。因此其 `key-value` 的映射逻辑也与 `weak_table` 的类似：

```c++
size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
size_t index = begin;
size_t hash_displacement = 0;
while (entry->referrers[index] != old_referrer) {
    index = (index+1) & entry->mask;
    if (index == begin) bad_weak_table(entry);
    hash_displacement++;
    if (hash_displacement > entry->max_hash_displacement) {
        objc_weak_error();
        return;
    }
}
```

### storeWeak

`objc_initWeak` 与 `objc_destroyWeak` 类似，最终都指向 `storeWeak` 方法，只是传递参数不同。

`objc_initWeak` 调用为 `storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating> (location, (objc_object*)newObj)`；

`objc_destroyWeak` 调用为 `storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating> (location, nil)`；

`HaveOld` 与 `HaveNew` 总是相反的，两者不会同时都有和同时都没有，`DontHaveOld, DoHaveNew` 表示初始化，`DoHaveOld, DontHaveNew` 表示销毁；`CrashIfDeallocating` 表示如果对象正处于销毁阶段是否产生 Crash，因此在对象的 `dealloc` 中不要试图使用 `weak` 修饰 `self`；最后的参数则是弱引用指针和对象地址。

```c++
// 简化后代码
template <HaveOld haveOld, HaveNew haveNew,
          enum CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;


 retry:
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    // 根据 table 地址，按大小进行加锁，避免死锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // 清除弱引用指针
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating ? CrashIfDeallocating : ReturnNilIfDeallocating);

        // 设置 isa 指针的 WeaklyReference 位域
        if (!_objc_isTaggedPointerOrNil(newObj)) {
            newObj->setWeaklyReferenced_nolock();
        }

        *location = (id)newObj;
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    // 如果对象实现了 _setWeaklyReferenced 方法，会调用通知
    callSetWeaklyReferenced((id)newObj);

    return (id)newObj;
}
```

在上述代码中，首先会获取到相应的 `SideTable`，在进行加锁时按 `SideTable` 的地址大小顺序进行枷锁，这可以避免死锁，之后进行 `SideTable` 的清理或者添加，最后设置 isa 指针的 `weakly_referenced` 位域。

在进行清理和添加时，会按一定逻辑进行压缩和扩容：

在清除时，`weak_table` 可能会压缩，大小大于 1024 且容量不满 1/16，压缩 `weak_table` 到原来的 1/8：

```c++
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // 大小大于 1024 且容量不满 1/16，压缩 weak_table
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
    }
}
```

在添加时，如果 `entry` 的大小超过 4，会转换成哈希表，如果容量占满 3/4，会进行扩容到原来的两倍：

```c++
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (! entry->out_of_line()) {
        // 尝试塞入到数组中
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        // 无法塞入数组，转换成哈希表
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[i];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```

如果 `weak_table` 容量占满 3/4，会进行扩容到原来的两倍：

```c++
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}
```

### weak_clear_no_lock

在对象被销毁时，会进行对象的所有弱引用指针的清理，它由 `dealloc` 调用：

```c++
// 简化后代码
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);

    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```

## NSTimer 的循环引用问题

在使用 `NSTimer` 时，可能会产生循环引用问题。例如，我们常常这样使用：

```objc
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(timerTriggered:) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
}

- (void)timerTriggered:(NSTimer *)timer {
    NSLog(@"timerTriggered");
}

- (void)dealloc {
    [self.timer invalidate];
    self.timer = nil;
    NSLog(@"%s", __func__);
}
@end
```

上述代码会产生循环引用，这是因为 `timer` 会保留其目标对象，等到自身“失效”时再释放此对象。调用 `invalidate` 方法可令计时器失效；执行完相关任务后，一次性定时器也会失效。如果是重复性定时器，那必须自己调用 `invalidate` 方法，才能令其停止。在 `vc` 中强持有一份 `timer`，同时由于这是一个重复性定时器，`NSTimer` 始终不会失效，也会一直强持有 `vc`，这产生了循环引用。当页面退出时，`vc` 和 `timer` 没有被外界对象引用 ，这会导致内存泄漏。

它的对象关系图如下：

![image-20230805234705294](/images/blog/image-20230805234705294-1250426.png)

下面给出一些常见的解决方案：

* 提前调用 `invalidate` 方法

如果能提前知道什么时候 `timer` 不需要了，可以提前调用 `invalidate` 方法，例如上例中可以在返回按钮被点击时调用 `invalidate`，这就使得 `RunLoop` 不会继续持有 `timer`，`timer` 因此失效，也不会强持有目标对象（`vc`），使得“环”被破除。但这种方式存在很大局限性，需要明确知道什么时候一定可以调用 `invalidate` 方法。

* 使用 `block` 方式使用 `timer`

在 iOS10 以上，新增了采用 `block` 方式使用 `timer`，这避免了以 target-action 方式强持有目标对象，只需处理对 `block` 的循环引用即可：

```objc
__weak typeof(self) weakSelf = self;
self.timer = [NSTimer timerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    NSLog(@"timerTriggered");
    // ...
}];
```

* 采用 `NSProxy` 或者中间对象进行消息转发

使目标对象从 `vc` 转换成其他对象，如 `NSProxy`，在 `NSProxy` 内部将消息转发到 `vc`，也可使得“环”被打破，例如：

```objc
@implementation LBWeakProxy

+ (instancetype)proxyWithTarget:(id)target {
    LBWeakProxy *proxy = [LBWeakProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    if ([self.target respondsToSelector:sel]) {
        NSMethodSignature *signature = [self.target methodSignatureForSelector:sel];
        return signature;
    }
    return [super methodSignatureForSelector:sel];
}

-(void)forwardInvocation:(NSInvocation *)invocation {
    SEL aSelector = invocation.selector;
    if ([self.target respondsToSelector:aSelector]) {
        invocation.target = self.target;
        [invocation invoke];
    } else {
        [super forwardInvocation:invocation];
    }
}

@end
  
LBWeakProxy *proxy = [LBWeakProxy proxyWithTarget:self];
self.timer = [NSTimer timerWithTimeInterval:1 target:proxy selector:@selector(timerTriggered:) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

它的对象关系图如下：

![image-20230805234413206](/images/blog/image-20230805234413206-1250254.png)

* 使用 `GCD` 代替 `NSTimer`

可以利用 `GCD` 实现一个 `timer`，例如开源库 [MSWeakTimer](https://github.com/mindsnacks/MSWeakTimer)，它不会保留目标对象，这样只需要在 `dealloc` 中释放掉 timer 就好。同时 GCD 的 timer 也不会有 RunLoop 的 Mode 切换、子线程创建 timer 的相关问题。

> NSTimer Special Considerations

> You must send this message from the thread on which the timer was installed. If you send this message from another thread, the input source associated with the timer may not be removed from its run loop, which could prevent the thread from exiting properly.

## 🔗

[1] [Effective Objective-C 2.0 编写高质量iOS与OS X代码的52个有效方法](https://book.douban.com/subject/25829244/)

[2] [Purpose of class ExplicitInit in objc runtime source code](https://stackoverflow.com/questions/64770338/purpose-of-class-explicitinit-in-objc-runtime-source-code)

[3] [如何在iOS中解决循环引用问题](https://draveness.me/retain-cycle1/)

[4] [警惕 NSTimer 引起的循环引用](https://huang-libo.github.io/posts/NSTimer-circular-reference/)

[5] [NSTimer循环引用分析与解决](https://nihao201.cn/2021/09/29/2021-09-29-NSTimer%20%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8%E5%88%86%E6%9E%90%E4%B8%8E%E8%A7%A3%E5%86%B3/)
