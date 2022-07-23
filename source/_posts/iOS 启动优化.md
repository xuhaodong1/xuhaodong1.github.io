---
title: "iOS 启动优化"
date: 2022-03-07 22:54:49
categories:
  - iOS
tags:
  - App启动
---
## 前言

启动时间往往是一个用户对 APP 的第一影响，如何优化启动时间一直都是个老生常谈的问题。本文结合了 WWDC16、WWDC17与WWDC19的相关 Session 与借鉴老司机的博客，梳理了启动阶段、dyld变化、Mach-O、虚拟内存等内容，简要阐述了如何优化与检测启动时间。

<!-- more -->

## 启动流程

### 启动类型

- 冷启动

	- 重启后 / APP 很长时间为启动过
	- 将它从磁盘加载到内存
	- 启动支撑 APP 运行的系统侧服务(system-side service)
	- 创建 APP 进程

- 热启动

	- APP 最近启动过
	- 由于有APP部分与系统侧服务在内存中，所以会比冷启动快一些。

### 启动阶段

- main 函数之前

	- 加载可执行文件
	- dyld 加载 dylibs，进行 rebase 指针调整 与 bind 符号绑定。

		- rebase: 修正指向当前 Mach-O 文件的指针。所需要的指针信息已经被编码到 __LINKEDIT 里，然后就是不断重复的对 __DATA 的指针加上这个偏移量。
		- bind: 对调用外部符号进行绑定处理。

	- Runtime 初始化

- main 函数之后

	- 实例化 UIApplication 与 UIApplicationDelegate 
	- 开始事件处理与系统集成
	- 调用 UIApplicationDelegate 中生命周期方法

		- 普通：
	application:willFinishLaunchingWithOptions: 
	application:didFinishLaunchingWithOptions:
	applicationDidBecomeActive:
		- 针对 UISceneDelegate：
	application:willFinishLaunchingWithOptions: 
	application:didFinishLaunchingWithOptions:
	scene:willConnectToSession:options:
	sceneWillEnterForeground:
	sceneDidBecomeActive:

	- 首屏渲染

		- loadView
		- viewDidLoad
		- layoutSubViews

	- 异步获取数据并呈现给用户

## dyld

### dyld 1.x

- 1996 - 2004
- 包含在 NeXTStep 3.0 中，不能很好支持相同语义同时一些边界条件无法正常运作。
- 预绑定(PreBuilding)：为 dylib 和 App 找到合适的固定地址，每次启动都会有。

### dyld 2.x

- 2004 - 2017
- 特性

	- Codesign

		- 以页为单位，Mach-O 文件的每一页都要被 hash，在被载入时进行与  __LINKEDIT 中的信息进行校验。

	- ASLR

		- Mach-O 文件被加载到随机的地址上。

	- Bounds Checking

		- 避免恶意的二进制文件的加入。

	- Shared cache

		- 共享缓存是包含所有系统 dylib 的单个文件，代替了 PreBuilding。

- 流程

	- 解析 Mach-O 的 Header 与 Load Commands，递归加载其依赖的动态库。
	- 映射所有 Mach-O 文件到 App 的地址空间。
	- 符号查找
	- Rebase & Bind
	- 运行初始化程序

### dyld 3.x

- 2017 - 至今
- 可缓存阶段 “解析Mach-O文件、找到对应依赖库、执行符号查找”，除了软件更新或者更改了磁盘上的库，负责每次启动的App依赖库符号不变、库中符号始终处于相同偏移量。因此将可缓存挪到了最前面，并将执行结果以闭包形式写入到磁盘。
- dyld 被分为 3 个组件

	- 进程外的 Mach-O 解析器 / 编译器
	- 进程内的引擎，用来启动闭包缓存
	- 启动闭包缓存服务

- 流程

	- 解析 Mach-O文件
	- 找到所有依赖库
	- 符号查找
	- 读取启动闭包
	- 验证启动闭包
	- 映射 Mach-O 文件到 App的地址空间
	- Rebase & Bind
	- 运行初始化程序

## Mach-O

### Mach-O 是系统不同运行时期可执行文件的文件类型统称，主要分为 3 种

- Executable：可执行文件，是 APP 中的主要 2 进制文件
- Dylib：动态库，在其他平台也叫 DSO / DLL
- Bundle：苹果平台特有的类型，是无法被链接的 Dylib。是能在运行时通过 dlopen() 加载

### 文件格式

- Header：包含了 Mach-O 文件的基本信息。如：CPU 架构、文件类型、加载指令数量等
- Load Commands：跟在 Header 后面的加载命令区，包含文件的组织架构、在虚拟内存的布局方式，在调用时知道如何设置和加载二进制文件。
- Data：包含 Load Commands 中需要的各个  Segment 的数据。
- 绝大多数 Mach-O 包含以下三种类型的 Segment

	- __TEXT：代码段，包含头文件、代码、常量。只读不可修改
	- __DATA：数据段，包含全局变量、静态变量。可读可写
	- __LINKEDIT：如何加载程序，包含方法和变量的元数据(位置、偏移量)，以及代码签名等信息。只读不可修改

### Mach-O Universal  Files

- 支持多个架构的 Mach-O 文件通常被称为通用二进制文件，也称胖二进制文件。
- 胖二进制文件起始处有一个 Fat Header，包含所有的架构以及它们在文件中的偏移量。

## 虚拟内存

### 虚拟内存是建立在物理内存和进程之间的中间层。是一个连续的逻辑地址空间，而且逻辑地址可以没有对应的实际物理地址，也可以让多个逻辑地址对应到同一个物理地址上。

### 缺页中断(Page Fault)

- 当进程访问一个没有对应物理地址的逻辑地址时，会发生 Page Fault

### 懒加载(Lazy Reading)

- 某个想要读取的页没有存在于内存会触发 Page Fault，系统通过 mmap() 函数读取指定页，这个过程成为 Lazy Reading。mmap() 函数可以将文件的部分内容映射到地址范围，而不是整个文件。

### 写时复制(Copy On Write)

- 当进程需要对某一页的内容进行修改时，内核会把需要修改的部分先复制一份，然后再修改，那个重新把逻辑地址映射到物理地址上去。

### Dirty Page & Clean Page

- Dirty Page：包含特定进程信息的页
- Clean Page：不包含特定进程信息的页。如果需要，内核可以重新生成 Clean Page。

### 共享内存(Shared RAM)

- 当多个 Mach-O 依赖同一个 Dylib(eg. UIKit) 时，系统会让几个 Mach-O 的调用 Dylib 的逻辑地址指向同一块物理内存空间，从而实现内存共享。Dirty Page 为进程独有，不能被共享。

## 优化方式

减少动态库或将多个动态库合并成一个

减少静态初始化

+load 方法内容放到首屏渲染后再执行，或使用 +initialize() 方法替换。

在 didFinishLaunchingWithOptions 和首屏渲染中只保留必要的初始化任务。

## 如何检测

hook

Instrument - App Lunch / 本地测量

Organizer - Launch Time / 线上数据

启动参数- DYLD_PRINT_STATISTICS=ture

## 参考链接

[老司机技术周报 - 优化 APP 启动](https://xiaozhuanlan.com/topic/4690823715)

[WWDC19 / 423 - 优化 APP 启动](https://developer.apple.com/videos/play/wwdc2019/423/)

[WWDC17 / 413 - App 启动时间: 过去、现在和将来](https://developer.apple.com/videos/play/wwdc2017/413)

[WWDC16 / 406 - 优化 App 启动时间](https://huang-libo.github.io/posts/Optimizing-App-Startup-Time/#%E7%9B%AE%E5%BD%95)

