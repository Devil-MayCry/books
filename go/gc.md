---
title: golang垃圾回收
date: "2019-08-02"
description: ""
url: /blog/go/
image: "/blog/go/gc.jpeg"
---
golang垃圾回收机制

<!--more-->

## 三色标记算法

### 何时触发GC
在堆上分配大于 32K byte 对象的时候进行检测此时是否满足垃圾回收条件，如果满足则进行垃圾回收。
``` go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ...
    shouldhelpgc := false
    // 分配的对象小于 32K byte
    if size <= maxSmallSize {
        ...
    } else {
        shouldhelpgc = true
        ...
    }
    ...
    // gcShouldStart() 函数进行触发条件检测
    if shouldhelpgc && gcShouldStart(false) {
        // gcStart() 函数进行垃圾回收
        gcStart(gcBackgroundMode, false)
    }
}
```
上面是自动垃圾回收，还有一种是主动垃圾回收，通过调用 runtime.GC()，这是阻塞式的
``` go
// GC runs a garbage collection and blocks the caller until the
// garbage collection is complete. It may also block the entire
// program.
func GC() {
    gcStart(gcForceBlockMode, false)
}
```

###  GC 触发条件

触发条件主要关注下面代码中的中间部分：forceTrigger || memstats.heap_live >= memstats.gc_trigger 。forceTrigger 是 forceGC 的标志；后面半句的意思是当前堆上的活跃对象大于我们初始化时候设置的 GC 触发阈值。在 malloc 以及 free 的时候 heap_live 会一直进行更新，这里就不再展开了。

``` go
// gcShouldStart returns true if the exit condition for the _GCoff
// phase has been met. The exit condition should be tested when
// allocating.
//
// If forceTrigger is true, it ignores the current heap size, but
// checks all other conditions. In general this should be false.
func gcShouldStart(forceTrigger bool) bool {
    return gcphase == _GCoff && (forceTrigger || memstats.heap_live >= memstats.gc_trigger) && memstats.enablegc && panicking == 0 && gcpercent >= 0
}

//初始化的时候设置 GC 的触发阈值
func gcinit() {
    _ = setGCPercent(readgogc())
    memstats.gc_trigger = heapminimum
    ...
}
// 启动的时候通过 GOGC 传递百分比 x
// 触发阈值等于 x * defaultHeapMinimum (defaultHeapMinimum 默认是 4M)
func readgogc() int32 {
    p := gogetenv("GOGC")
    if p == "off" {
        return -1
    }
    if n, ok := atoi32(p); ok {
        return n
    }
    return 100
}
```

### 垃圾回收的主要流程
三色标记算法是对标记阶段的改进，原理如下：
- 起初所有对象都是白色。
- 从根出发扫描所有可达对象，标记为灰色，放入待处理队列。
- 从队列取出灰色对象，将其引用对象标记为灰色放入队列，自身标记为黑色。
- 重复 3，直到灰色对象队列为空。此时白色对象即为垃圾，进行回收。
  
![](gc.png)

关于上图有几点需要说明的是。

- 首先从 root 开始遍历，root 包括全局指针和 goroutine 栈上的指针。
- mark 有两个过程。
  - 从 root 开始遍历，标记为灰色。遍历灰色队列。
  - re-scan 全局指针和栈。因为 mark 和用户程序是并行的，所以在过程 1 的时候可能会有新的对象分配，这个时候就需要通过写屏障（write barrier）记录下来。re-scan 再完成检查一下。
- Stop The World 有两个过程。
  - 第一个是 GC 将要开始的时候，这个时候主要是一些准备工作，比如 enable write barrier。
  - 第二个过程就是上面提到的 re-scan 过程。如果这个时候没有 stw，那么 mark 将无休止。

另外针对上图各个阶段对应 GCPhase 如下：
- Off: _GCoff
- Stack scan ~ Mark: _GCmark
- Mark termination: _GCmarktermination

#### 写屏障 (write barrier)
垃圾回收中的 write barrier 可以理解为编译器在写操作时特意插入的一段代码，对应的还有 read barrier。

为什么需要 write barrier，很简单，对于和用户程序并发运行的垃圾回收算法，用户程序会一直修改内存，所以需要记录下来。

### STW phase
#### phase1
GC 开始之前的准备工作
#### Mark
Mark 阶段是并行的运行，通过在后台一直运行 mark worker 来实现。
#### Mark termination (STW phase 2)
mark termination 阶段会 stop the world。函数实现在 gcMarkTermination()。1.8 版本已经不会再对 goroutine stack 进行 re-scan 了
#### 清扫
分为阻塞式和并行式。

对于并行式清扫，在 GC 初始化的时候就会启动 bgsweep()，然后在后台一直循环。

不管是阻塞式还是并行式，都是通过 sweepone()函数来做清扫工作的。内存管理都是基于 span 的，mheap_ 是一个全局的变量，所有分配的对象都会记录在 mheap_ 中。在标记的时候，我们只要找到对对象对应的 span 进行标记，清扫的时候扫描 span，没有标记的 span 就可以回收了。


(参考：https://www.cnblogs.com/hezhixiong/p/9577199.html)
