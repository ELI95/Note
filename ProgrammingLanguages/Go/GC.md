#### 垃圾收集器

- Go 垃圾收集算法
    - 介绍
        - Go 的垃圾回收算法使用的是无无分代（对象没有代际之分）、不整理（回收过程中不对对象进行移动与整理）、并发（与用户代码并发执行）的标记清除（三色标记）算法
    - 选择原因
        - 对象整理的优势是解决内存碎片问题以及“允许”使用顺序内存分配器。但 Go 运行时的分配算法基于tcmalloc，基本上没有碎片问题。 并且顺序内存分配器在多线程的场景下并不适用。Go 使用的是基于tcmalloc的现代内存分配算法，对对象进行整理不会带来实质性的性能提升
        - 分代GC依赖分代假设，即GC将主要的回收目标放在新创建的对象上（存活时间短，更倾向于被回收），而非频繁检查所有对象。Go 的编译器会通过逃逸分析将大部分新生对象存储在栈上（栈直接被回收），只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。也就是说，分代GC回收的那些存活时间短的对象在 Go 中是直接被分配到栈上，当goroutine死亡后栈也会被直接回收，不需要GC的参与，进而分代假设并没有带来直接优势。Go 的垃圾回收器与用户代码并发执行，使得 STW 的时间与对象的代际、对象的 size 没有关系。Go 团队更关注于如何更好地让 GC 与用户代码并发执行（使用适当的 CPU 来执行垃圾回收），而非减少停顿时间这一单一目标上

- 根对象
    - 全局变量：程序在编译期就能确定的那些存在于程序整个生命周期的变量
    - 执行栈：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针
    - 寄存器：寄存器的值可能表示一个指针，参与计算的这些指针可能指向某些赋值器分配的堆内存区块
  
- 三色标记
    - 三色不变性
        - 强三色不变性：黑色对象不可以指向白色对象
        - 弱三色不变性：黑色对象可以指向白色对象，但白色对象上游必须存在灰色对象
    - 遵循三色不变性中的任意一个，都能保证垃圾收集算法的正确性

- 内存屏障
    - 原理
        - 保障了代码描述中对内存的操作顺序 既不会在编译期被编译器进行调整，也不会在运行时被 CPU 的乱序执行所打乱， 是一种语言与语言用户间的契约
        - 在SMP架构下，每个CPU与内存之间，都配有自己的高速缓存（Cache），以减少访问内存时的冲突
        - 其本质原理就是对系统总线加锁
    - 场景
        - 内存屏障是在并发标记过程中保证三色不变性的重要技术
    - Dijkstra插入写屏障
      ```sh
      writePointer(slot, ptr):
        shade(ptr)
        *slot = ptr
      ```
      - 在新增或修改对象的指针时，如果指针指向的对象是白色的，那么会将该对象设置为灰色
      - 满足强三色不变性
      - 优点
        - 性能优势：无需对指针读进行任何处理
        - 前进保障：对象只会按 白色->灰色->黑色 的流程转换颜色，因此总工作量受到堆大小的限制
      - 缺点
        - 由于 Dijkstra 插入屏障的保守，在一次回收过程中可能会产生一部分被染黑的垃圾对象，只有在下一个回收过程中才会被回收
        - Dijkstra 必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之间做出权衡
    - Yuasa删除写屏障
      ```sh
      writePointer(slot, ptr)
        shade(*slot)
        *slot = ptr
      ```
      - 在某对象的引用被删除时，如果该对象为白色，则设置为灰色
      - 满足弱三色不变性
      - the Yuasa barrier requires a STW at the beginning of marking to either scan or snapshot stacks, but does not require a re-scan at the end of marking
    - 混合写屏障
      ```sh
      writePointer(slot, ptr):
        shade(*slot)
        if current stack is grey:
          shade(ptr)
        *slot = ptr
      ```
      - 组合插入写屏障和删除写屏障
      - 满足弱三色不变性
      - 为了移除栈的重扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要将创建的所有新对象都标记成黑色
      - The write barrier shades the object whose reference is being overwritten, and, if the current goroutine's stack has not yet been scanned, also shades the reference being installed
      - The Yuasa barrier requires a STW at the beginning of marking to either scan or snapshot stacks, but does not require a re-scan at the end of marking
      - The Dijkstra barrier lets concurrent marking start right away, but requires a STW at the end of marking to re-scan stacks (though more sophisticated non-STW approaches are possible)
      - The hybrid barrier inherits the best properties of both, allowing stacks to be concurrently scanned at the beginning of the mark phase, while also keeping stacks black after this initial scan
    - 批量写屏障缓存
        - 将需要着色的指针统一写入一个缓存， 每当缓存满时统一对缓存中的所有 ptr 指针进行着色

- 触发时机
    - 申请内存
        - runtime.mallocgc -> gcTriggerHeap -> gcStart
        - 运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发新的垃圾收集循环
    - 后台触发
        - runtime.sysmon -> runtime.forcegchelper -> gcTriggerTime -> gcStart
    - 手动触发
        - runtime.GC -> gcTriggerCycle -> gcStart

- 后台标记模式
    - 在垃圾收集启动期间，运行时会调用 runtime.gcBgMarkStartWorkers 为全局每个处理器创建用于执行后台标记任务的 Goroutine
    - 垃圾收集控制器会在 runtime.gcControllerState.findRunnabledGCWorker 方法中设置处理器的 gcMarkWorkerMode
    - gcMarkWorkerMode
        - gcMarkWorkerDedicatedMode
        - gcMarkWorkerFractionalMode
        - gcMarkWorkerIdleMode
    - 三种不同模式的工作协程会相互协同保证垃圾收集的 CPU 利用率达到期望的阈值，在到达目标堆大小前完成标记任务

- 并发扫描与标记辅助
    - runtime.gcBgMarkWorker
        - runtime.gcBgMarkWorker 是后台的标记任务执行的函数，该函数的循环中执行了对内存中对象图的扫描和标记
        - 原理
            - 获取当前处理器以及 Goroutine 打包成runtime.gcBgMarkWorkerNode 类型的结构并主动陷入休眠等待唤醒
            - 根据处理器上的 gcMarkWorkerMode 模式决定扫描任务的策略
            - 所有标记任务都完成后，调用 runtime.gcMarkDone 方法完成标记阶段
    - runtime.gcDrain
        - 是用于扫描和标记堆内存中对象的核心方法
    - 工作池
        - 写屏障、根对象扫描和栈扫描都会向工作池中增加额外的灰色对象等待处理，而对象的扫描过程会将灰色对象标记成黑色，同时也可能发现新的灰色对象，当工作队列中不包含灰色对象时，整个扫描过程就会结束
        - runtime.gcWork 为垃圾收集器提供了生产和消费任务的抽象
        - runtime.gcWork.balance 会将处理器本地一部分工作放回全局队列中，让其他的处理器处理，保证不同处理器负载的平衡
        - 运行时会调用 runtime.gcDrain 扫描工作缓冲区 runtime.gcWork 中的灰色对象
    - 写屏障
        - 当 Go 语言进入垃圾收集阶段时，全局变量 runtime.writeBarrier 中的 enabled 字段会被置成开启，所有的写操作都会调用 runtime.gcWriteBarrier
    - 标记辅助
        - 用户程序辅助标记的核心目的是避免用户程序分配内存影响垃圾收集器完成标记工作的期望时间，它通过维护账户体系保证用户程序不会对垃圾收集造成过多的负担，一旦用户程序分配了大量的内存，该用户程序就会通过辅助标记的方式平衡账本，这个过程会在最后达到相对平衡，保证标记任务在到达期望堆大小时完成

- 并发垃圾收集
    - 状态
        - _GCoff
        - _GCmark
        - _GCmarktermination
    - 流程
        - 验证垃圾收集的触发条件，并循环调用 runtime.sweepone 清理已经被标记的内存单元，完成上一个垃圾收集循环的收尾工作

        - 调用 runtime.gcBgMarkStartWorkers 启动后台标记任务
        - Stop the world
        - 将状态切换至 _GCmark，开启混合写屏障，将根对象入队
        - Start the world
        - 并发标记
            - 开始扫描根对象，优先扫描所有 Goroutine 的栈空间，将可达对象均标记为黑，扫描 Goroutine 的栈空间期间会暂停当前处理器，然后扫描全局对象以及不在堆中的运行时数据结构
            - 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色
            - 写屏障会将被覆盖的指针和新指针都标记成灰色，而所有新创建的对象都会被直接标记成黑色
            - 堆空间启用混合写屏障，栈空间不启用混合写屏障
        - 标记完成

        - Stop the world
        - 将状态切换至 _GCmarktermination
        - 清理处理器上的线程缓存，进行关于垃圾收集的数据统计
        - 将状态切换至 _GCoff
        - 关闭混合写屏障
        - Start the world
          
        - Sweep
