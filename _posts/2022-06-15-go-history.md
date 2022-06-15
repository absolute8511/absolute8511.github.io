---
layout: post
title: 一文纵览 Go 历史 - 解析 Go 历史版本的重要变化
date: 2022-06-15
categories: tech
tags: [go,tech]
comments: true
---

本文是一个公司内部分享整理的资料，梳理了 Go 这十几年来的发展和变化，由于内容和资料众多，这里只将一些重要的功能优化做了介绍，可以通过附加的链接深入了解感兴趣的内容，从这些资料可以看到 Go 团队的技术创新能力。

随着Go生态越来越丰富，和云原生的发展相辅相成，未来发展前景无限。

转载请注明来源：[Go 历史](http://absolute8511.github.io/tech/2022/06/15/go-history/）

## Go创始作者

- Ken Thompson (Unix, C语言, Plan9， Bell Lab)
- Robert Griesemer (Google's V8, the Java HotSpot JVM)
- Rob Pike (Unix, Plan9， Bell Lab)

## Go传统

- 从诞生开始, 完全开源化运作
- 半年一个版本
- 1.x 保障语法完全向下兼容
- 吉祥物: Gopher 地鼠

## Go历史里程碑

- 2007年 开始设计 (Google 20% project)
- 2009.11 对外宣布启动 Go project
- 2012.03 Go 1.0版本正式发布
- 2015.08 Go1.5版本开始完成Go自举
- 2018.04 新Logo启用
- 2019.11 十周年
- 2022.03 Go1.18加入泛型支持


## 值得一提的性能优化和功能优化

核心: 不要过早优化, 把优化当做需求去分析对待, 没有需求的优化就是过早优化

Golang前期的优化主要集中在 调度器, 内存分配器, GC, 定时器, 编译, cgo, 包管理等几个方面

### 史前阶段(before Go1)

- 最简单的调度循环, 所有goroutine的调度通过全局互斥锁进行全局级别的管理， G-M 模型组成
- 最简陋可用的全局STW串行GC


### 1.1

- 优化maps实现 (significant reduction in memory footprint and CPU time)
- 允许更大的heap size (from 1.1， 到1.10都是线性内存空间)
- 优化并行GC
- 更好的调度算法实现, 经典的work-stealing算法( <https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit> , <https://rakyll.org/scheduler/> )

### 1.2

- 基于协作抢占调度(Pre-emption in the scheduler): any loop that includes a (non-inlined) function call can be pre-empted.
- goroutine最小栈大小从 4KB 改到 8KB. (因为老的栈实现方式是不连续的分段链式栈, 随着栈增长性能下降)
- 网络性能改成使用系统集成的poller优化30% (on Windows and BSD, 类似Linux的epoll) 
- 支持测试覆盖率统计

### 1.3

- 内存模型变更: 确保channel符合新的内存模型, a buffered channel can be used as a simple semaphore, using a send into the channel to acquire and a receive from the channel to release.
- goroutine栈实现从分段模型(segment stack)改成连续模型(contiguous segment). 消除了“hot split”的问题 ( https://medium.com/a-journey-with-go/go-how-does-the-goroutine-stack-size-evolve-447fc02085e5 )
- 运行时基于只有指针类型的值包含指针的假设增加了对栈内存的精确扫描支持，实现了真正精确的垃圾收集
- GC: 使用并发sweep算法优化, better parallelization, and larger pages. (50-70% reduction in collector pause time)
- 优化regexp实现
- 默认开启TCP keep-alives
- 支持sync.pool (参考 https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277 )

### 1.4

- 使用 Go 重写大部分的runtime实现, 这样GC能更准确的扫描栈上活跃变量信息. (to be fully precise, meaning that it is aware of the location of all active pointers in the program. This means the heap will be smaller as there will be no false positives keeping non-pointers alive.) 其他优化也减少了部分heap size(10%-30%优化).
- 栈动态增长时, 使用拷贝并更新指向栈变量的指针, 默认栈大小从8KB 减少到2KB.
- 为并发GC做准备, 更新堆上指针会被包装成一个函数调用, 增加write barrier. 

### 1.5

- The compiler and runtime are now written entirely in Go 
- 大幅优化GC, 基于三色标记清扫的并发GC和其他goroutines并行优化, 大幅减少stw时间 (https://docs.google.com/document/d/1wmjrocXIWTr1JxU-3EQBI6BK6KgtiFArkG47XK73xIQ/edit#)
- GOMAXPROCS默认从1改成设置成cpu核数
- 包管理新增 vendoring 支持
- 一个新的go tool trace, 支持更细粒度的程序性能调优和跟踪
- 简化跨平台编译
- 动态共享库的测试支持(后继新版本移除了该支持)
- channel 优化(https://docs.google.com/document/d/1yIAYmbvL3JxOKOjuCyon7JhW4cSv1wy5hC0ApeGMV9s/pub)

### 1.6

- 针对和C共享Go指针新增几个强制规则. 可以共享的前提是, 指针指向的内存内容不包含任何指向Go分配的内存的指针, 并且在c函数返回后, C不再引用该指针. (Go and C may share memory allocated by Go when a pointer to that memory is passed to C as part of a cgo call, provided that the memory itself contains no pointers to Go-allocated memory, and provided that C does not retain the pointer after the call returns. https://github.com/golang/proposal/blob/master/design/12416-cgo-pointers.md)
- 库自动支持 HTTP/2
- sort优化
- 使用密集的位图替代空闲链表表示的堆内存，降低清除阶段的 CPU 占用，降低大内存使用时的GC latency (1.5 heap超过20G，latency将超过10ms, 新版会稳定， https://go.googlesource.com/proposal/+/master/design/12800-sweep-free-alloc.md)
- 去中心化GC, 任意 Goroutine 都能触发垃圾收集的状态迁移

### 1.7

- 通过并行栈收缩将垃圾收集的时间缩短至 2ms 以内 (for programs with large numbers of idle goroutines, substantial stack size fluctuation, or large package-level variables， https://github.com/golang/go/issues/12967#issuecomment-171466238)
- 64-bit x86新的编译后端基于SSA(static single assignment)
- 大幅优化编译时间, 提升一倍 (依然是1.4的2倍)
- Context包支持

### 1.8

- 优化GC暂停时间, 使用混合写屏障，消除栈重新扫描带来的stw (参考 https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md), 为了移除栈的重扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。
- 所有架构启用SSA (generates more compact, more efficient code and provides a better platform for optimizations such as bounds check elimination. The new back end reduces the CPU time required by our benchmark programs by 20-30% on 32-bit ARM systems.) 借助SSA IR，编译器则可以通过查找赋值了但是没有使用的变量，来识别冗余赋值。其他可做的优化包括常量折叠（constant folding）、常量传播（constant propagation）、强度削减（strength reduction）以及死代码删除（dead code elimination）等
```
x1=4*1024经过常量折叠后变为x1=4096
x1=4; y1=x1经过常量传播后变为x1=4;y1=4
y1=x1*3经过强度削减后变为y1=(x1<<1)+x1
if(2>1){y1=1;}else{y2=1;}经过死代码删除后变为y1=1
```
- The compiler and linker have been optimized and run faster in this release than in Go 1.7
- 支持 plugins
- GC不再认为参数一定会在函数内存活( no longer considers arguments live throughout the entirety of a function.) 可以使用 runtime.KeepAlive 确保变量存活, 常用于系统调用和cgo调用.
- cgo调用优化(减少不必要的defer调用, https://go-review.googlesource.com/c/go/+/30080/)
- http优雅退出和http2 push支持

### 1.9

- 大对象分配性能优化(using large (>50GB) heaps containing many large objects)
- 透明的单调时间支持(Transparent Monotonic Time support， https://go.googlesource.com/proposal/+/master/design/12914-monotonic.md)
- 增加Concurrent Map
- 包函数级别的并行编译


### 1.10

- 自动编译缓存, 提升编译速度
- permits passing string values directly between Go and C using cgo.
- 增加程序诊断工具, 包括pprof的webui等.
- 更新了GC Pacer的实现，分离软硬堆大小的目标（ Many applications should experience significantly lower allocation latency and overall performance overhead when the garbage collector is active https://go.googlesource.com/proposal/+/master/design/14951-soft-heap-limit.md ）
- 增加描述https://tip.golang.org/ref/spec#Representability
- 新增strings.Builder补充bytes.Buffer


### 1.11

- Go Modules引入
- adds experimental support for calling Go functions from within a debugger.
- 引入稀疏堆布局, 稀疏的堆内存空间替代了连续的堆区内存， 突破堆大小不能超过512GB的限制(The runtime now uses a sparse heap layout so there is no longer a limit to the size of the Go heap). 解决部分场景在C 和 Go 混合使用时会导致程序崩溃的问题。（https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/）
- 新增读写锁支持

### 1.12

- go mod继续改进
- 活跃变量分析改进. (finalizers will be executed sooner. consider the appropriate addition of a runtime.KeepAlive call.)
- 使用新的标记终止算法简化垃圾收集器的几个阶段（significantly improves the performance of sweeping when a large fraction of the heap remains live. This reduces allocation latency immediately following a garbage collection， https://go.googlesource.com/proposal/+/master/design/26903-simplify-mark-termination.md）
- runtime更加主动的归还内存给OS
- 定时器并发性能优化, 全局使用 64 个四叉堆维护全部的计时器, 网络超时精度会更准
- runtime使用MADV_FREE配置释放内存.可能会导致OS使用内存上涨

### 1.13

- 持续迭代go mod
- 更精准的逃逸分析, 减少了gc压力 (影响unsafe的不合法使用)
- defer优化, 当该关键字在函数体中最多执行一次时, 使用栈上分配
- 支持error wrapping

### 1.14

- 官方开始建议迁移到go mod, 标志着go mod的成熟度达到了预期
- defer最后的优化, 使用开放编码的优化, 可以认为大部分基本无开销. (少于或者等于 8 个, 不能在循环中,  return 语句与 defer 语句的乘积小于或者等于 15 个)中间代码生成时判断是否启用, 一旦确定使用，就会在编译期间初始化延迟比特和延迟记录(在栈上初始化大小为 8 个比特的 deferBits)
- Goroutines支持基于系统信号的异步抢占. 没有函数调用的循环体也不再死锁调度, 对GC暂停也不再有过大的影响.
- 全新page allocator内存分配器，在高并发下锁竞争更少, 性能更好, 高吞吐下的内存敏感型应用延迟毛刺会变少. （https://go.googlesource.com/proposal/+/master/design/35112-scaling-the-page-allocator.md） 
- 进一步优化定时器性能, 将timer分配到每个P上，降低锁竞争；去掉timer thread，减少上下文切换开销；使用netpoll的timeout实现timer机制。(https://github.com/golang/go/issues/6239#issuecomment-206361959)
- 支持 fuzzing test
- 在特定版本linux内核上会出现crash的问题，源于这些内核的一个已知bug,信号抢占提升了bug出现的几率(https://github.com/golang/go/issues/37436, https://bugzilla.kernel.org/show_bug.cgi?id=205663)

### 1.15

- 继续优化小对象在高并发下的内存分配
- better linker (https://docs.google.com/document/d/1D13QhciikbdLtaI67U6Ble5d_1nsI4befEd6_k1z91U/view, 后继可能会带来更好的plugin实现, 并且未来可能需要一个新的object文件格式了, 当前基本都是1980年代的设计.)

### 1.16

- 支持bin文件嵌入静态资源
- On Linux, 归还内存使用 MADV_DONTNEED之前是 MADV_FREE (rather than lazily when the operating system is under memory pressure). 
- deprecated io/ioutil

### 1.17

-  实现新的传参和传返回值规范, 优先使用寄存器 (show performance improvements of about 5%), 之前使用栈实现起来会更加简单、更容易维护. （https://go.googlesource.com/proposal/+/master/design/40724-register-calling.md）

### 1.18

- 支持部分泛型语法 (cannot handle type declarations inside generic functions or methods), 暂未支持泛型方法仅支持泛型函数和泛型类型.
- 增加fuzzing实现
- 支持 "Workspace" mode. 
- 动态GC运行时机优化 (includes non-heap sources of garbage collector work (e.g., stack scanning) when determining how frequently to run)
- for Go 1.19 to require Go 1.17 or later for bootstrap

### unreleased

- 减少gc goroutine在空闲时对cpu的占用, 参考 https://github.com/golang/go/commit/d8cf2243e0ed1c498ed405432c10f9596815a582
- goroutine初始栈会根据历史平均来确定, 总体会更少浪费了。
- 编译器使用jump table重新实现了针对大整型数和string类型的switch语句，平均性能提升20%左右。
- runtime.SetMemoryLimit限制内存上限, 从而更积极的归还内存
- 修订Go memory model描述, 做了更正式的整体描述，增加了对multiword竞态、runtime.SetFinalizer、更多sync类型、atomic操作以及编译器优化方面的描述(https://go.dev/ref/mem)

### 参考资料

- [https://www.luozhiyun.com/archives/458]
- [https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/history/]
- [https://draveness.me/golang/]
- [https://go.dev/doc/devel/release]
- [https://medium.com/a-journey-with-go/go-retrospective-b9723352e9b0]
- [https://go.googlesource.com/proposal/+/master/design/]

## Go代码常用优化手段

- slice初始化预分配优化
- pool对象池
- goroutine池
- gc cpu调优, 避免频繁gc， 设置合适的GCPercent（内存使用稳定的业务, 避免 100MB->200MB->100MB->200MB， https://eng.uber.com/how-we-saved-70k-cores-across-30-mission-critical-services/）
- string, []byte 转换使用unsafe避免
- epoll改造替换(参考开源库)
- pb序列化使用gogo优化
- timer粒度优化, 重用优化
- string concat优化
- 避免map内使用指针

## Go创始团队的总结

- Go is the answer to Development scale and Production scale. (云厂商让更多小公司也能实现更快更大规模的部署, 大部分公司依赖由大量开发者维护的开源基础设施)
- 包管理, import, export
- Go encourages composition of types, provides object-oriented polymorphism through its interface types
- Concurrency, goroutine, efficient multiplexing operation(epoll), channels(CSP)
- Security and Safety, 没有悬挂指针, 数值类型不会自动转换, 空指针引用和越界访问会触发异常
- Completeness, HTTPS, strings, hash maps, and dynamically sized arrays as built-in, easily used data types, provides integrated support for cross-compilation, testing, profiling, code coverage, fuzzing, and more. 
- Consistency, race detector, 一致的运行性能和启动性能表现, (没有其他语言JIT的慢启动), fully concurrent garbage collector with pauses taking less than a millisecond.  backward-compatible changes to the language and standard library
- Tool-Aided Development, go vet tool aims to identify common correctness problems with high precision. gofmt formats source code using consistently applied layout rules. profilers, debuggers, analyzers, build automators, test frameworks, and so on
- Libraries, semantic versioning, proactively identify and report vulnerable packages to users
- Pike and Thompson 已退休(被google尊养), Russ Cox为目前Go团队的新leader(MIT博士毕业加入go团队), Ian Lance Taylor

参考: [https://cacm.acm.org/magazines/2022/5/260357-the-go-programming-language-and-environment/fulltext][260357-the-go-programming-language-and-environment]

[260357-the-go-programming-language-and-environment]: https://cacm.acm.org/magazines/2022/5/260357-the-go-programming-language-and-environment/fulltext 
[https://www.luozhiyun.com/archives/458]: https://www.luozhiyun.com/archives/458
[https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/history/]: https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/history/
[https://draveness.me/golang/]: https://draveness.me/golang/
[https://go.dev/doc/devel/release]: https://go.dev/doc/devel/release
[https://medium.com/a-journey-with-go/go-retrospective-b9723352e9b0]: https://medium.com/a-journey-with-go/go-retrospective-b9723352e9b0
[https://go.googlesource.com/proposal/+/master/design/]: https://go.googlesource.com/proposal/+/master/design/
