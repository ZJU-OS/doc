# Bonus B：多核处理器与调度算法

!!! danger "DDL"

    - 代码、报告：2026-01-10 23:59
    - 验收：无

!!! tip "本次实验供同学们自由发挥：没有自动评测，提供的代码不多，描述也比较简略。"

## 实验简介

在此前的实验中，我们的处理器只有一个核心。这种情况下的并发不是真并发，每个时刻系统中仍只有一个进程在运行。

当处理器拥有多个核心，软、硬件要考虑的事情就变得十分复杂。我们已经在《计算机体系结构》课程中体会过这种复杂性：

- 缓存一致性：使用侦听或目录协议保证多核系统的 Cache 与主存保持一致
- 内存一致性：不同模型下，核心之间的读写操作可见性不同

好在 QEMU 帮我们处理了这些硬件部分的内容，我们的操作系统在不同核心上看到的内存是一致的。

本次实验让我们来考虑软件部分：

1. 多个核心要如何启动？

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

### 更新代码

现在你位于 `lab5` 分支。你需要创建 `bonus-b` 分支，合并上游的代码：

```shell
git checkout -b bonus-b
git fetch upstream
git merge upstream/bonus-b
```

### PerCPU 数据结构

当我们的系统扩展到多核时，有些**全局和静态变量**需要为每个核心单独维护，例如 `current` 应当指向当前核心运行的进程的 PCB。为了方便这一用法，Linux 内核设计了 PerCPU 数据结构。用法如下：

```c
DEFINE_PER_CPU(struct percpu_struct, percpu);
get_cpu_var(percpu)++;
```

实现静态的 PerCPU 数据结构很简单，使用数组即可：

- 将宏 `DEFINE_PER_CPU` 展开为长度等于核心数的数组定义
- 获取值时，根据当前的核心编号，从数组中获取对应的 PerCPU 数据结构

请你阅读相关代码理解 PerCPU 数据结构。

!!! example "思考题"

    - 为什么自动变量（本地变量）不需要设计 PerCPU 变量？

        !!! tip "回忆 C 语言课程中学习变量生命周期、存储位置的知识。"

    - 为什么 `current` 没有定义为 PerCPU 变量？

## Part 1：多核启动过程

现在 QEMU 的启动参数添加了 `-smp 4`，此时将使用四个核启动 QEMU。观察 OpenSBI 的输出，可以看到：

```text
Domain0 Boot HART           : 2
Domain0 HARTs               : 0*,1*,2*,3*
```

不难推测，OpenSBI 会选择其中一个核作为 Boot Hart，其余 Hart 则暂时不启动。如果不做其他工作，我们的系统就这样只使用一个核心，之前实验的代码仍然能正常运行。你可以向合并前的代码添加 `-smp 4` 并运行 QEMU 来验证。

那么如何唤醒剩余的核心，并告诉它们应该执行什么代码呢？我们再次向 SBI 寻求帮助。请阅读 SBI 规范 Chapter 9. Hart State Management Extension (EID#0x48534D "HSM")，这一章介绍了 SBI 提供的管理 Hart（硬件线程）的接口。你需要在规范中找到下列问题的答案：

- Hart 有哪些状态？
- 如何使用 SBI 提供的接口改变 Hart 的状态？
- 如何使用 `sbi_hart_start()`？它接收什么参数？
- `sbi_hart_start()` 启动一个 Hart 时，它处于什么样的状态？
    - 它启用了中断吗？
    - 它使用物理地址还是虚拟地址？
    - 它的寄存器存放着什么内容？

!!! example "动手做：查看核心初始状态"

    请你在 `start_kernel()` 中使用 OpenSBI 提供的接口，打印刚启动时 4 个核心的状态。

如果你正确找到了上述问题的答案，不难分析出剩余核心唤醒后需要做的工作：

- 设置 Trap 向量寄存器
- 启用虚拟内存
- 开启中断

完成这些工作后，我们的所有核心就都运行在虚拟地址空间上，并且能够由时钟中断驱动调度等机制的运行了。

### Task 1：唤醒其他核心

- **请你阅读** `start_kernel()` 中新增的 `smp_init()` 及其相关源码，理解 Boot Hart 启动其他核心的逻辑。
- **请你补全** `kernel/arch/riscv/kernel/head.S` 的 `secondary_start()`，完成其他核心启动时需要做的工作。

## Part 2：多核调度

本节实现的调度算法基于 Linux 2.6.0-2.6.22 的 $O(1)$ 调度器，建议你阅读 [The Linux Scheduling Algorithm](https://litux.nl/mirror/kerneldevelopment/0672327201/ch04lev1sec2.html)。

### Run Queue

将调度算法扩展到多核并不复杂，最简单的方式就是让每个核心运行各自的调度算法。于是我们将原先的全局链表 `task_list` 修改为 PerCPU 的运行队列，在每个核心各自的进程队列上运行 RR 调度算法。

请你阅读 `kernel/arch/riscv/kernel/sched.h` 中 `struct run_queue` 的定义和相关代码，理解 Run Queue 如何描述某个 CPU 核心上任务的运行情况。

### Load Balance

现在每个核心能够在自己的队列上运行调度算法。我们还需要解决核心间的问题：任务应当何时移动到其他核心？如何移动？

操作系统调度算法的普遍目标是提升公平性、响应性。要在多核系统中实现公平性，自然的想法是令所有核心的任务队列长度趋向相同，也就是进行**负载均衡（Load Balance）**。做法很简单：每个核心主动去寻找负载比自己高的核心，并尝试将任务移动到自己的队列。

### Task 2：实现多核负载均衡

- **请你自行修改**自己的调度器，实现上文描述的多核负载均衡的功能。我们对调度器的复杂度没有严格要求，它不一定要是 $O(1)$ 的，能实现合适的负载均衡策略即可。

## Part 3：多核同步方法

因为同学们的代码实现不同，所以实验指导无法涵盖所有的情况。本章将用两个例子介绍原子操作和锁两种多核同步方法，剩余的部分需要同学们自己阅读、调试代码，识别所有关键区。

### 使用原子操作保护简单变量

注意到多个核心可能同时调用 `create_pid()`，它的核心是一个静态变量。让我们看看它的汇编代码：

```text title="kernel/vmlinux.asm"
	return pm_nr_tasks++;
ffffffd600201ff8:	0004d797          	auipc	a5,0x4d
ffffffd600201ffc:	01078793          	addi	a5,a5,16 # ffffffd60024f008 <pm_nr_tasks.5>
ffffffd600202000:	0007a503          	lw	a0,0(a5)
ffffffd600202004:	0015071b          	addiw	a4,a0,1
ffffffd600202008:	00e7a023          	sw	a4,0(a5)
```

可以看到，C 语言中简单的一个递增操作需要三条指令来完成：`lw`、`addiw`、`sw`。

!!! example "思考题：识别临界区"

    同学们已经在理论课上学习过进程同步、临界区等知识。请你指出为什么这三条指令在多核系统中是一个**临界区**？

对于这样简单的临界区，我们不需要使用锁，只需要使用原子操作。请同学们阅读 [`__atomic` Builtins (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)，了解 GCC 内置了哪些原子操作函数。

通过阅读文档，我们找到了适合该操作的 `__atomic_fetch_add()` 函数：

```text title="kernel/vmlinux.asm"
	return __atomic_fetch_add(&pm_nr_tasks, 1, __ATOMIC_SEQ_CST);
ffffffd600202024:	0004d797          	auipc	a5,0x4d
ffffffd600202028:	fe478793          	addi	a5,a5,-28 # ffffffd60024f008 <pm_nr_tasks.5>
ffffffd60020202c:	00100713          	li	a4,1
ffffffd600202030:	06e7a52f          	amoadd.w.aqrl	a0,a4,(a5)
```

这样就实现了多线程安全的变量修改。

!!! example "思考题：`fetch` 与操作的先后顺序"

    注意到文档中还有 `__atomic_add_fetch()`，它和这里使用的 `__atomic_fetch_add()` 有什么区别？

### 使用锁保护复杂数据结构

**自旋锁（spin lock）**用于防止多个线程同时进入临界区，用法如下：

```c
spin_lock(&lock);
/* critical section */
spin_unlock(&lock);
```

有多种方法可以实现自旋锁，最简单的是原子操作。GCC 提供了相关的内建函数，见 [`__atomic` Builtins (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)。我们将使用 `bool __atomic_test_and_set (void *ptr, int memorder)` 来实现自旋锁，它原子化地对指针 `ptr` 指向的位置执行下面的操作：

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

请阅读 [20 RISC-V指令精讲（五）：原子指令实现与调试](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E5%AE%9E%E6%88%98%E8%AF%BE/20%20RISC-V%E6%8C%87%E4%BB%A4%E7%B2%BE%E8%AE%B2%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E5%8E%9F%E5%AD%90%E6%8C%87%E4%BB%A4%E5%AE%9E%E7%8E%B0%E4%B8%8E%E8%B0%83%E8%AF%95.md)，理解 RISC-V 指令集定义的原子操作。

### Task 3：识别并保护关键区

来到本次实验的最后一个任务，我们梳理至今为止的所有并发来源，帮助同学们更清晰地识别关键区并应用合适的保护措施：

1. **真并发**：同一时刻，有多个线程在临界区中运行。这是本实验引入的新的并发来源。

    1. **多核系统**：多核系统是真并发的唯一来源。只有多核系统中，同一时刻才可能存在多个线程在同一个临界区中执行。

        **保护措施**：如前文所述，使用原子操作或自旋锁。

1. **伪并发**：同一时刻，只有一个线程在运行；但多个线程交织运行，它们都位于临界区。这正是此前的实验处理的并发来源。

    1. **Trap**：Trap 可能在任意时刻发生，导致系统重复进入 Trap 处理程序 `trap_handler()`。如果它及其调用的函数存在临界区，就有可能因重复进入产生伪并发。

        **保护措施**：

        - 在 Trap 处理程序中，一般无法使用锁来保护临界区。考虑下面的死锁情况：

            - 旧 Trap 拿到锁进入了临界区。
            - 新 Trap 希望进入临界区，但锁被旧 Trap 占用。
            - 新 Trap 永远无法返回，旧 Trap 也永远无法释放锁。

        - 所以只能尽可能避免发生嵌套 Trap：

            1. RISC-V 指令集规定，Trap 时自动关闭全局**中断**。
            2. 只要代码写得好，一般不会发生**异常**。
            3. 对于**无法屏蔽的中断**，在处理代码中避免临界区，从而能够安全地重复进入；或者在中断控制器等硬件上为这类特殊的中断实现等待队列，上一个中断处理完成前不发送下一个中断。

    1. **抢占**：一些临界区不应当被抢占，特别是一些使用自旋锁的临界区。
    
        让我们考虑优先级调度的情况。设进程 A 和 B 都需要使用一把锁，其中 B 的优先级比 A 高。A 获取锁后被 B 抢占，B 忙等尝试获取锁。此时因为 B 优先级比 A 高，A 永远没有机会运行，无法释放锁，B 也无法获取锁，造成死锁。

        **保护措施**：进程结构体添加了 `preempt_count` 字段，初始值为 0。当进入抢占关键区时，该字段递增；离开时递减。当 `preempt_count` 不为 0 时，调度器不抢占进程。

!!! tip

    我们只需要关心内核代码的同步问题，不需要考虑用户态。因为用户态和内核态是完全隔离的，用户态的并发程序应当自己实现同步机制。

一个临界区可能受多种并发来源的影响，也有可能通过多种方式解决。比如上文讲解的 `create_pid()`，它在多核系统中受到真并发的影响，在单核系统中也会受到抢占的影响，但这两种并发来源都可以通过原子操作解决。

现在，请你自行分析内核中的代码、数据结构，并应用合适的保护措施。

## 预期实验结果

现在的 `start_kernel()` 结尾会每隔 1s 创建一个内核进程。预期行为是：

- 当进程不断增多时，你的调度器能够尽可能保证公平，即每个核心的任务队列长度差异不大。
- 系统内几乎没有同步问题（如死锁、临界区保护不当等），能够多次重复稳定运行，不受日志级别的影响。

本实验不要求做到完美无缺，毕竟并发问题始终是计算机领域的难点之一，助教们也花了非常多时间才完成。你在实验过程中一定会遇到不少困难，请在报告中详细记录你是如何解决的，这能够证明你确实是自己完成实验的。

## 调试建议

调试多线程程序时，如果使用 GDB 命令行将非常复杂。推荐使用 VSCode 的 GDB 调试插件，能够十分便捷地分析各个核心的运行情况。调试图例如下，可以在左侧看到每个核心的调用栈，在右侧看到对应的 C 和汇编代码等。

![vscode.png](bonusB.assets/vscode.png)
