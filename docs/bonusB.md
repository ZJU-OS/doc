# Bonus B：多核处理器与调度算法

## 实验简介

在此前的实验中，我们的处理器只有一个核心。这种情况下的并发不是真并发，每个时刻系统中仍只有一个进程在运行。

当处理器拥有多个核心，操作系统要考虑的事情就变得十分复杂。我们已经在《计算机体系结构》课程中体会过这种复杂性：缓存一致性和内存一致性都由多核系统导致。好在 QEMU 帮我们处理了这些内容，我们只需要考虑软件层面的事情：

1. 多个核心会如何启动？

    多核启动时，OpenSBI 会选择一个核心作为启动核心，其余核心则暂时不启动，可以通过 SBI 调用唤起。我们将思考：

    - 启动过程的哪些工作应当在唤起其他核心前完成？
    - 唤起其他核心时，它们应当执行什么代码？

2. 如何在多个核心上调度进程？

    将进程分配到不同核心上执行，才能实现真并发。我们将思考：

    - 如何设计调度算法，能够将进程自动分配到合适的核心上运行？
    - 进程、调度相关的数据结构需要如何修改？

3. 如何处理真并发？

    在单核系统中，我们通过控制中断阻止抢占的方式来保护关键区。在多核系统中，即使我们关闭了某个核心的中断，其他核心的进程仍然在运行，仍可能修改共享的数据结构。我们将思考：

    - 如何实现自旋锁？
    - 内核中的哪些地方是关键区？
    - 如何使用自旋锁保护关键区？

## Part 0：合并代码

### PerCPU 数据结构

在多核系统中，有些**全局和静态变量**需要为每个核心单独维护，例如 `current` 应当指向当前核心运行的进程的 PCB。为了方便这一用法，Linux 内核设计了 PerCPU 数据结构。用法如下：

```c
DEFINE_PER_CPU(struct percpu_struct, percpu);
get_cpu_var(percpu)++;
```

实现静态的 PerCPU 数据结构很简单，使用数组即可：

- 将宏 `DEFINE_PER_CPU` 展开为长度等于核心数的数组
- 获取值时，根据当前的核心编号，从数组中获取对应的 PerCPU 数据结构

!!! example "思考题"

    - 为什么自动变量（本地变量）不需要设计 PerCPU 变量？
        
        !!! tips "回忆 C 语言课程中学习变量生命周期、存储位置的知识。"
    
    - 为什么 `current` 没有定义为 PerCPU 变量？

### 自旋锁

本节讲解自旋锁（spin lock）的实现。自旋锁用于防止多个线程同时进入临界区，用法如下：

```c
spin_lock(&lock);
/* critical section */
spin_unlock(&lock);
```

实现自旋锁有多种方式，最简单的一种是原子操作。GCC 提供了相关的内建函数，见 [`__atomic` Builtins (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)。我们将使用 `bool __atomic_test_and_set (void *ptr, int memorder)` 来实现自旋锁，它原子化地对指针 `ptr` 指向的位置执行下面的操作：

- 如果 `ptr` 指向的值为 Set，则返回 False
- 如果 `ptr` 指向的值不为 Set，则将 `ptr` 置为 Set，返回 True

用它实现自旋锁的逻辑如下：

```c
while (__atomic_test_and_set(&lock->lock, __ATOMIC_ACQUIRE)) {
    // busy wait
}
```

在 RISC-V 架构下，该函数由 A 指令集的原子指令 `amoor.w.aq` 实现：

<iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:___c,selection:(endColumn:1,endLineNumber:6,positionColumn:1,positionLineNumber:6,selectionStartColumn:1,selectionStartLineNumber:6,startColumn:1,startLineNumber:6),source:'%23include+%3Cstdint.h%3E%0Avoid+test(uint64_t+*lock)%0A%7B%0A++++__atomic_test_and_set(lock,+__ATOMIC_ACQUIRE)%3B%0A%7D%0A'),l:'5',n:'0',o:'C+source+%231',t:'0')),k:40.502227622519236,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-cgcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'1',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:___c,libs:!((name:cs50,ver:'910')),options:'',overrides:!(),selection:(endColumn:19,endLineNumber:13,positionColumn:9,positionLineNumber:13,selectionStartColumn:19,selectionStartLineNumber:13,startColumn:9,startLineNumber:13),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:59.497772377480764,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

请阅读 [20 RISC-V指令精讲（五）：原子指令实现与调试](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E5%AE%9E%E6%88%98%E8%AF%BE/20%20RISC-V%E6%8C%87%E4%BB%A4%E7%B2%BE%E8%AE%B2%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E5%8E%9F%E5%AD%90%E6%8C%87%E4%BB%A4%E5%AE%9E%E7%8E%B0%E4%B8%8E%E8%B0%83%E8%AF%95.md)，理解 RISC-V Atomic Memory Operation。

## Part 1：多核启动过程

现在 QEMU 的启动参数添加了 `-smp 4`，此时将使用四个核启动 QEMU。观察 OpenSBI 的输出，可以看到：

```text
Domain0 Boot HART           : 2
Domain0 HARTs               : 0*,1*,2*,3*
```

不难推测，OpenSBI 会选择其中一个核作为启动核心，其余核心则暂时不启动。如果不做其他工作，我们的系统仍然能正常运行。你可以向合并前的代码添加 `-smp 4` 并运行 QEMU 来验证。

那么如何唤醒剩余的核心，并告诉它们应该执行什么代码呢？我们再次向 SBI 寻求帮助。请阅读 SBI 规范 Chapter 9. Hart State Management Extension (EID#0x48534D "HSM")，这一章介绍了 SBI 提供的管理 Hart（硬件线程）的接口。你需要在规范中找到下列问题的答案：

- Hart 有哪些状态？
- 如何使用 SBI 提供的接口改变 Hart 的状态？
- 如何使用 `sbi_hart_start()`？它接收什么参数？
- `sbi_hart_start()` 启动一个 Hart 时，它处于什么样的状态？
    - 它启用了中断吗？
    - 它使用物理地址还是虚拟地址？
    - 它的寄存器存放着什么内容？

!!! example "动手做：查看核心状态"

    请你在 `start_kernel()` 中使用 OpenSBI 提供的接口，打印 4 个核心的状态。

如果你正确找到了上述问题的答案，不难分析出剩余核心唤醒后需要做的工作：

- 启用虚拟内存
- 开启中断

### Task 1：唤醒其他核心

## Part 2：多核调度

本节实现的调度算法基于 Linux 2.6.0-2.6.22 的 $O(1)$ 调度器，请同学们阅读 [The Linux Scheduling Algorithm](https://litux.nl/mirror/kerneldevelopment/0672327201/ch04lev1sec2.html)。

## Part 3：多核线程同步
