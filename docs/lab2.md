# Lab 2：抢占式内核线程调度

!!! danger "DDL"

    - 代码、报告：2025-11-04 23:59
    - 验收：2025-11-11 实验课

## 实验简介

!!! tip "术语：进程、线程与 Task（任务）"

    在理论课上，我们明确定义了进程和线程。而在工程中，实现并不会那么死板。请你阅读下面的文段，了解在 Linux 和本实验的语境中这些术语的含义：

    > As you will see later, Linux has a unique implementation of threads: **It does not differentiate between threads and processes.**To Linux, a thread is just a special kind of process.
    >
    > Another name for **a process is a task**. The Linux kernel internally refers to processes as tasks. In this book, **I use the terms interchangeably**, although when I say task I am generally referring to a process from the kernel’s point of view.

    上面的文段来自 [Linux Kernel Development](https://www.oreilly.com/library/view/linux-kernel-development/9780768696974/)，一本经典的 Linux 内核开发入门书籍。

    具体来说：

    - Linux 系统调用 `clone()` 可以创建新的 Task，传入的参数将决定新 Task 具有怎样的性质：
        - `CLONE_FILES | CLONE_VM | CLONE_FS`：表示新 Task 与父 Task 共享文件系统、内存空间和文件描述符表，这样的 Task 符合我们对**线程（Thread）**的定义。
        - 不使用上面的参数：表示新 Task 拥有独立的文件系统、内存空间和文件描述符表，这样的 Task 符合我们对**进程（Process）**的定义。
    - 在调度器看来，只有 Task，没有进程和线程的区别。

    那为什么本实验叫做「内核线程」不叫「内核任务」呢？因为历史上一直把运行在内核态的 Task 叫做**内核线程（Kernel Thread）**。

    现在你已经理解在实验中这些术语的区分并没有那么严格，接下来阅读实验内容时就不用扣字眼了😉。

在 Lab1 中，我们已经让内核能够启动并处理时钟中断。但是如果没有进一步的机制，操作系统仍然只能在 CPU 上执行单个任务（即 `start_kernel` 函数）。本次实验的目标是：**实现抢占式内核线程调度，让多个任务能够在一个 CPU 上通过时间复用的方式交替运行。**

要完成这一目标，我们需要逐步解决以下几个问题：

1. **如何在内核中管理物理内存？**（理论课第八章内容 - 主存）

    在后续实验里，我们需要动态分配各类数据结构。但是操作系统无法使用 `malloc` 等标准库函数，那么问题来了：

    - **操作系统如何在裸机环境下动态分配内存？**
    - **为什么我们不能直接线性地“挖一块”内存？碎片化如何解决？**

    实验框架提供了一个简化版 **Buddy System 物理内存分配器**。同学们需要理解它的基本原理，并学会通过接口分配和释放物理页。

    Buddy System 仅能实现物理页分配，**无法满足对象粒度的分配需求**。因此，我们还需要实现一个**空闲链表缓存（free list cache）**，用于高效地分配内核对象。

2. **如何管理进程？**（理论课第三章内容 - 进程）

    任务交替运行的核心在于**上下文切换（context switch）**。要让任务 A 运行完一段时间后交出 CPU，再切换到任务 B，必须保存 A 的上下文，并在之后恢复它。

    - **上下文具体包含哪些内容？**
    - **RISC-V 调用约定和中断机制如何影响上下文切换？**
    - **如何实现抢占？**

    解决完上下文切换的基本问题后，我们需要为每个 Task 设计一个数据结构（即理论课介绍的 PCB，Process Control Block），用于保存上下文和其他属性。并且还需要实现进程的创建和销毁机制，从而让内核能够动态地管理多个任务。

3. **调度器应该如何工作？**（理论课第五章内容 - 调度算法）

    有了内存和进程管理，接下来就能够实现调度器。调度器的任务是：

    1. 根据调度算法选择下一个要运行的任务
    2. 调用上下文切换代码，将 CPU 控制权交给它

    本次实验中，我们将先实现一个简单的**时间片轮转（Round Robin）调度器**。

## Part 0：环境配置

### 更新代码

现在你位于 `lab1` 分支。你需要创建 `lab2` 分支，合并上游的代码：

```shell
git checkout -b lab2
git fetch upstream
git merge upstream/lab2
```

下面的合并说明供同学们解决合并冲突时参考：

- 新增实验相关：
    - 内存管理代码 `mm.h`，在 Part1.2 介绍
    - 链表代码 `list.h`，在 Part1.3 介绍
    - 空闲链表缓存代码 `cache.h`，在 Part1.4 介绍
    - 进程管理相关实验代码 `proc.h`，在 Part1.5 介绍
    - 内核线程代码 `kthread.h`，在 Part1.6 介绍
    - 调度器代码 `sched.h`，在 Part4 介绍
    - 几个测试样例 `test.h`、`test_*.c` 在对应的 Task 介绍
- 新增其他：
    - 日志系统，见 `log.h`
        - 自动识别并打印日志位置的**文件名和行号**
        - 支持**日志级别**（DEBUG、INFO、WARN、ERROR），可以通过修改 `log.h` 的 `LOG_LEVEL` 宏来控制输出的详细程度
        - **框架已经添加了部分调试日志**，同学们可以将 `LOG_LEVEL` 设置到 `LOG_LEVEL_DEBUG` 来查看
        - **建议同学们多用这一日志系统而不是 `printk`**
- 变更：
    - CSR 相关内容从 `sbi.h` 移动到 `csr.h`
    - 时钟中断的间隔与调度算法有关，从 1s 改为 `kernel/arch/riscv/include/sched.h` 中定义的 `TIMER_INTERVAL`
    - `arch/riscv/kernel/include` 中的所有头文件现在都添加了 Doxygen 风格的**详细的 API、数据结构、常量的注释，请同学们积极阅读和使用**，文档不会再赘述。

### 更新镜像

本次镜像更新在 `/etc/gitconfig` 添加了 `safe.directory` 的配置，以解决代码库挂载到容器后的权限问题。这不是一个必要的修复，同学们可以视情况选择更新镜像。

## Part 1：内存管理

### 物理内存空间

在学习如何管理物理内存之前，我们需要先弄清楚**物理内存空间**的概念。所谓内存空间，就是 **CPU 可以直接寻址的地址范围**，其大小通常由寄存器的位数决定。

- 例如，在 **RV32 架构**中，通用寄存器宽度为 32 位。当执行指令 `ld x1, 0(x2)` 时，`x2` 中存放的就是一个内存地址，它决定了 `x1` 要从哪里取数据。由于寄存器只有 32 位，最多能寻址 $2^{32}$ 个字节，因此 RV32 的地址空间大小为 $2^{32} \text{B} = 4 \text{GB}$。
- 而在 **RV64 架构**下，寄存器宽度扩展为 64 位，对应的寻址空间理论上可以达到 $2^{64} \text{B} = 2^{34} \text{GB} = 16 \text{EB}$，远超当前实际硬件所能提供的内存容量。

在真实机器上，CPU 不可能真的拥有 16 EB 的内存。**系统只会映射其中的一小部分地址作为物理内存，其余地址范围通常保留给外设、固件、I/O 总线等。**在 Lab0 中，我们已经使用过 `info mtree` 命令来查看 QEMU 模拟的内存映射，接下来基于该结果进行具体分析：

- **RV64 架构的地址空间**：`0000000000000000` 到 `ffffffffffffffff`，总共 16 EB。

    ```text title="(qemu) info mtree"
    address-space: cpu-memory-0
    address-space: memory
      0000000000000000-ffffffffffffffff (prio 0, i/o): system
        ...
    ```

- **地址空间的具体划分**：

    - 某些区域存放固件或测试寄存器（如 `mrom`, `sifive.test`）

        ```text
        0000000000001000-000000000000ffff (prio 0, rom): riscv_virt_board.mrom
        0000000000100000-0000000000100fff (prio 0, i/o): riscv.sifive.test
        ```

    - 某些区域对应定时器、PLIC（中断控制器）、串口、virtio 设备等外设

        ```text
        000000000c000000-000000000c5fffff (prio 0, i/o): riscv.sifive.plic
        0000000010000000-0000000010000007 (prio 0, i/o): serial
        0000000000101000-0000000000101023 (prio 0, i/o): goldfish_rtc
        ```

        !!! note "要点：总线与内存映射 I/O"

            在计科的硬件课程中，大家可能还没有系统学习过**总线 (Bus)** 的概念。在这里我们只需要掌握最基本的理解：

            - **总线是一种统一的通信通道**，CPU、内存、外设都通过总线交互数据。
            - 在现代计算机中，CPU 把一部分地址空间“划给”外设。当我们用普通的 `load/store` 指令访问这些地址时，看似在访问内存，实际上是通过总线把数据读写到某个外设寄存器上。
            - 举个例子：当我们向串口地址空间写入一个字节，硬件就会把它输出到终端；当我们读取 RTC（实时时钟）的寄存器，就能获得当前时间。

            这就是所谓的 **Memory-Mapped I/O（MMIO）**：通过“内存地址空间”来操纵外设。

    - 最关键的是，一部分地址范围被映射为 **RAM**，即实验中我们真正可以使用的物理内存。例如：

        ```text
        0000000080000000-0000000087ffffff (prio 0, ram): riscv_virt_board.ram
        ```

        这表示从 `0x80000000` 开始，有一段连续的物理内存区域被作为可用 RAM 提供给操作系统。

并且，在前述的实验中，我们也具体了解到 `0x80000000-0x87ffffff` 这 128MB 的物理内存的划分：

- `0x80000000-0x801fffff`：OpenSBI 占用
- `0x80200000-_ekernel`：内核占用
- `_ekernel-0x87ffffff`：可以自由使用的物理内存

!!! note "要点：物理内存空间"

    - RV64 的理论地址空间非常大，但实际可用的物理内存只是其中一小块。
    - QEMU 会在 `0x80000000` 起映射出一段 RAM，这是我们在实验中管理和使用的主要物理内存区域。
    - 其他区域虽然也是“地址”，但对应的是固件或外设。CPU 通过读写这些地址，与硬件进行交互。

### Buddy System

在理解了“我们有哪些物理内存”之后，下一步要思考的就是：**如何高效地管理这段有限的物理内存**。如果没有一套规则来分配和回收内存，那么很快就会出现“要么分配不出去，要么浪费大片空间”的问题。

最常见的一个困难是 **内存碎片**：

- 如果我们每次都按照请求大小直接分配，随着不断申请和释放，内存会被切得零零碎碎，即使总剩余空间很多，也可能无法满足一个大块内存的请求。
- 我们需要一种机制来让内存块尽量保持“规整”，方便回收和再次利用。

**Buddy System** 就是这样一种经典方案：

1. **内存划分规则**

    - 假设我们有一段连续的物理内存，总大小是 $2^n$ 页。
    - 系统按 **2 的幂次方大小**划分内存块：1 页、2 页、4 页……直到 $2^n$ 页。
    - 每次分配时，如果需要的大小不是 2 的幂，会自动向上取整。

2. **分配过程**

    - 找到能满足需求的最小块，如果没有，就从更大的块中“劈开一半”。
    - 每劈一次，就会得到一对“伙伴块”（buddy）。

3. **释放过程**

    - 当一个块被释放后，系统会检查它的伙伴是否空闲。
    - 如果伙伴也空闲，就将两者合并成更大的块，并继续向上合并，直到不能合并为止。
    - 这样可以减少碎片，使得大块空间能重新腾出来。

在实验框架里，我们已经为大家实现了一个简化版的 **Buddy System 物理内存分配器**（位于 `mm.c` / `mm.h`）。你无需从零开始写算法，只需要掌握它的使用方式。

请阅读 `kernel/arch/riscv/include/list.h`，了解 Buddy System 提供的接口。在实验里，你只需要通过这些接口申请或释放物理页，而不必关心内部具体是如何实现的。

例如：

```c title="Buddy System 接口使用示例"
void *a = alloc_page();      // 分配一页
void *b = alloc_pages(8);    // 分配 8 页
free_pages(a);               // 释放之前分配的一页
```

通过这一套接口，我们的内核就能在后续实验（如进程调度、内存管理、虚拟内存）中，稳定地使用物理页作为底层资源。

!!! note "要点：Buddy System"

    - Buddy System 按 2 的幂次方划分内存块，方便合并和回收，减少碎片。
    - 实验框架已经实现了简化版的 Buddy System，你只需通过 `alloc_page(s)` 和 `free_pages()` 接口来分配和释放物理页。

### 侵入式双向循环链表

在内核中我们经常需要组织各种动态对象，例如空闲页链表、就绪任务队列、等待队列等。Linux 内核实现了**侵入式双向循环链表（intrusive doubly-linked list）**来高效地管理这些对象。

- **侵入式与泛型接口**：

    普通链表实现**将数据嵌入在链表中**，而侵入式实现**将链表嵌入在数据中**。前者**每一种数据类型都需要单独定义链表节点结构体和相关的函数**，导致代码重复且不易维护。后者**不同类型可以复用同一套接口**，这就是泛型（generic）。

    <div class="grid cards" markdown>

    ```c title="侵入式链表"
    struct list_head {
        struct list_head *next, *prev;
    };
    #define LIST_HEAD_INIT(name) { &(name), &(name) }
    #define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)
    static inline void list_add(struct list_head *new, struct list_head *head);
    static inline void list_del(struct list_head *entry)

    // 使用例
    struct data_node_A {
        int data;
        struct list_head list;
    } a;
    struct data_node_B {
        char data;
        struct list_head list;
    } x;
    LIST_HEAD(list_A);
    LIST_HEAD(list_B);
    list_add(&a.list, &list_A);
    list_add(&x.list, &list_B);
    ```

    ```c title="普通链表"
    struct data_node_A {
        int data;
        struct data_node_A *next, *prev;
    } a, *list_A = NULL;
    struct data_node_B {
        char data;
        struct data_node_B *next, *prev;
    } x, *list_B = NULL;
    static inline void list_A_add(struct data_node_A *new, struct data_node_A *head);
    static inline void list_B_add(struct data_node_B *new, struct data_node_B *head);

    // 使用例
    list_A_add(&a, list_A);
    list_B_add(&x, list_B);
    ```

    </div>

- **访问数据**：

    理解了上面的代码，你会发现盲点：`list_head` 结构体并不包含数据字段。怎么通过 `list_A` 这样的链表头，访问到 `data_node_A` 结构体呢？

    这就需要使用编译器魔法了：`container_of` 宏。它的定义如下：

    ```c title="kernel/arch/riscv/include/list.h"
    #define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr); \
        (type *)( (char *)__mptr - offsetof(type,member) );})
    ```

    编译器知道类型的大小，提供了 `typeof()` 和 `offestof()`。我们可以利用这些信息**移动成员变量的指针，计算出包含该成员变量的结构体的起始地址**。例如：

    ```c title="container_of 使用示例"
    struct data_node_A a;
    struct list_head *p = &a.list;
    struct data_node_A *q = container_of(p, struct data_node_A, list);
    assert(q == &a); // q 恰好指向 a 的起始地址
    ```

<figure markdown="span">
    ![list_head.drawio](lab2.assets/list_head.drawio)
    <figcaption>
    侵入式双向循环链表示意图
    </figcaption>
</figure>

实现链表还给我们附带了一个惊喜：**实现了队列**。同学们在数据结构课上已经学过，**双向循环链表实现了双端队列（deque）**，它支持在头尾两端高效地插入和删除节点。于是我们有实现多级反馈队列调度器所需的工具了。

!!! note "要点：侵入式双向循环链表"

    - 侵入式链表将链表节点嵌入数据结构中，支持泛型接口，代码复用性强。
    - 通过 `container_of` 宏，可以从链表节点指针计算出包含它的结构体指针。

### Task 1：实现双向链表

你应该早在大一的 C 语言课上就能秒杀双向链表了 😉。请补全 `kernel/arch/riscv/include/list.h`：

- 提示：观察 `LIST_HEAD` 宏，你会发现这应该是一个**带哨兵**的双向链表。思考如果不带哨兵，它应当该如何实现？
- 补全 `list_empty()`。
- 补全 `list_add()`、`list_add_tail()` 和 `list_del()`。提示：一个优雅的做法是令 `list_add()` 和 `list_add_tail()` 都调用一个更底层的 `__list_add()`。
- 补全 `list_for_each` 宏。

!!! success "完成条件"

    - 通过评测框架的 `lab2 task1` 测试。

    `kernel/arch/riscv/kernel/test_list.c` 中的 `test_list()` 函数用于测试你的链表实现。

### 空闲链表缓存

本节利用链表实现一个内核中非常常见的机制——**对象缓存（object cache）**。

在内核中，许多内核对象（如 `task_struct`、`inode`、`vm_area_struct` 等）会被**频繁创建和销毁**。如果动态创建都调用通用的内存分配器（如 Buddy System），会对性能产生较大影响，且容易引起内存碎片。

为了解决这些问题，Linux 内核引入了 **Slab 分配器**：它为每种类型的对象维护一个缓存池，将整页内存切分为多个固定大小的对象块，用链表管理这些空闲对象。换句话说，Slab 缓存层（`kmem_cache`）相当于物理页分配器（`alloc_page()`）之上的对象粒度的再分配层。

不过，Linux 的 Slab 实现更加复杂，感兴趣的同学可以阅读 [Slab Layer - Linux Kernel Development Second Edition](https://litux.nl/mirror/kerneldevelopment/0672327201/ch11lev1sec6.html)。在我们的教学实验中，只保留了它的**核心思想：用空闲链表（free list）预分配缓存对象。**

`kernel/arch/riscv/include/cache.h` 和 `kernel/arch/riscv/include/cache.c` 实现了 Slab 分配器，要求同学们阅读代码并理解其工作原理。

!!! example "动手做：理解空闲链表"

    阅读 `cache.h` 和 `cache.c`，理解空闲链表缓存的实现。

    画一幅图，展示相关的数据结构及其布局。

## Part 2：进程切换与 Trap 处理

本节我们探究实现用于切换进程上下文的汇编函数 `__switch_to()`，以及进程切换与 Trap 处理的相互影响。

### 进程切换：非抢占情况

让我们从最简单的不考虑 Trap 的情况开始。这种情况下没有时钟中断，进程切换仅能依靠进程主动调用 `__switch_to()` 完成，内核是非抢占式的。

设想这样一个场景：CPU 上正在运行 Task1，Task1 想调用 `__switch_to()` 切换到 Task2，然后 Task2 运行一段时间后再切换回来继续执行 Task1。`__switch_to()` 应该怎么设计？

Lab 1 Trap 处理的场景与此类似：可以将 `_traps()` 看作 `__switch_to()`，将 `trap_handler()` 看作 Task2。`trap_handler()` 执行完毕后，恢复 Trap 上下文就相当于 Task2 切换回 Task1。**但进程切换与 Trap 有一个重要区别：Trap 可以发生在任何时候**，因此 Trap 上下文是所有寄存器的状态；而 `__switch_to()` 是一个函数，**它的调用者一定是遵守 RISC-V 调用约定的**，因此不需要保存所有寄存器。示意图如下：

<figure markdown="span">
    ![switch.drawio](lab2.assets/switch.drawio)
    <figcaption>
    `__switch_to()` 的调用和返回路径
    </figcaption>
</figure>

理解了这些，我们容易设计出 `__switch_to()` 需要保存的进程上下文：

- `ra`：函数返回地址，我们需要它来返回到调用 `__switch_to()` 的位置
- `sp`、`s0-s11`：调用约定中规定的 callee-saved 寄存器

于是我们写出了进程上下文的数据结构：

```c
struct thread_struct {
    uint64 ra;      // return address
    uint64 sp;      // stack pointer
    uint64 s[12];   // callee-saved registers
};
```

那么这个结构体应该存放在哪里呢？具体存放的位置暂且按下不表，留到设计进程数据结构时再说。先假设调用 `__switch_to()` 时 `a0` 和 `a1` 分别指向 Task1 和 Task2 的 `thread_struct` 结构体，那么 `__switch_to()` 的实现就很简单了：

```asm title="第一版 __switch_to"
    .globl __switch_to
__switch_to:
    sd ra, 0(a0)          // 保存 Task1 上下文
    sd sp, 8(t0)
    sd s0, 16(t0)
    ...

    ld ra, 0(a1)          // 恢复 Task2 上下文
    ld sp, 8(a1)
    ld s0, 16(a1)
    ...
    ret                    // 返回到 Task2
```

> 还能再换个角度：站在 Task1 的角度看，调用 `__switch_to()` **就像调用了一个空函数**。它遵守 RISC-V 调用约定，但什么都没做，原路返回了。

但切换的目标 Task 2 的上下文从哪里来？对于目前不考虑 Trap 的场景来说，只有两种可能：

1. Task 2 之前运行过，但主动调用 `__switch_to()` 切换到别的进程。这种情况**保存过进程上下文**，能够顺利恢复。
2. Task 2 是新创建的进程，尚未运行过。这种情况需要我们**设计一个初始的进程上下文**。

初始进程上下文的设计也很简单：令 `ra` 指向进程的第一条指令，`__switch_to()` 就会跳过去开始执行。如果要考虑向进程传递参数，可以利用 `s0-s11` 这些寄存器，然后设置一个蹦床函数（trampoline）来将其移动到 `a0-a7` 作为参数。事实上 Linux 就是这么实现 kthread 的初始上下文的，这个蹦床函数如下所示：

```asm title="(Linux)arch/riscv/kernel/entry.S"
SYM_CODE_START(ret_from_fork_kernel_asm)
    call schedule_tail
    move a0, s1 /* fn_arg */
    move a1, s0 /* fn */
    move a2, sp /* pt_regs */
    call ret_from_fork_kernel
    j ret_from_exception
SYM_CODE_END(ret_from_fork_kernel_asm)
```

```c title="(Linux)arch/riscv/kernel/process.c"
asmlinkage void ret_from_fork_kernel(void *fn_arg, int (*fn)(void *), struct pt_regs *regs)
{
    fn(fn_arg);

    syscall_exit_to_user_mode(regs);
}
```

从代码中可以看出，Linux 将 kthread 要运行的函数 `fn` 放在了初始进程上下文的 `s0` 寄存器中，将参数 `fn_arg` 放在了 `s1` 寄存器中。蹦床函数 `ret_from_fork_kernel_asm` 将它们分别移动到 `a1` 和 `a0`，然后调用 `ret_from_fork_kernel` 去具体执行。

### 进程切换：抢占与 Trap 处理

当我们希望实现[内核抢占（Kernel preemption）](https://kernelnewbies.org/FAQ/Preemption)时，情况就变得复杂了起来，需要综合考虑时钟中断、系统调用等 Trap 情况。

#### 切换、调度、Trap 与抢占的关系

先理清几个概念：

- **进程切换（context switch）**：

    由上文讨论的 `__switch_to()` 执行，完成进程上下文切换

- **调度（schedule）**：

    由将在 Part 4 介绍的 `schedule()` 执行，使用调度算法选择要切换到的进程，并调用 `__switch_to()` 完成切换

- **Trap**：

    CPU 异常和中断引发，可能发生在任何时候，由 `_traps()` 处理，负责保存和恢复 Trap 上下文

- **抢占**：

    在进程不主动执行切换或调度的情况下，能够打断当前进程的执行触发**调度**，从而实现时间复用

这几个概念的联系是：**抢占**是通过在 **Trap** 退出路径上视情况**调度**实现的。接下来的几节我们将分析这一关系。

#### 内核中不能被 Trap 的部分

首先，我们要明确内核中不能被 Trap 的部分。回忆 RISC-V 中断机制，进入 Trap 处理时，CPU 会自动执行下面的操作禁用中断：

- `sstatus.SPIE = sstatus.SIE`
- `sstatus.SIE = 0`

此外，我们设计 `_traps()` 时会确保不触发异常。**所以，Trap 处理过程是不会被 Trap 的。**

其他部分似乎都可以被 Trap，毕竟我们设计 `_traps()` 时就考虑到它随时可能发生，会将所有整数寄存器保存到 Trap 上下文中。

#### 抢占的时机

要实现抢占，只能寄希望于（时钟）中断打断 Task1 的执行，才有机会切换到 Task2。接下来我们用**排除法和反证法**寻找这个机会在哪里：

- **Trap 上下文恢复过程**显然不能切换，否则就破坏 Trap 上下文了。
- **Trap 上下文恢复完成**（离开 `_traps()`）后，就回到 Task 1 没机会切换进程了。
- **Trap 处理过程（`trap_handler()`）**不能 `__switch_to()`，因为中断还没处理完。

    Trap 的处理过程指的是从 `xIP.i` 置 1、中断 Pending 开始，到该 Pending 位被置 0 为止。以 Lab1 涉及的 Trap 情况为例：

    - 时钟中断的处理从 SEE 发现 `stimecmp > stime` 然后将 `STIP` 置 1 开始，到 `sbi_set_timer()` 将 `STIP` 置 0 结束
    - 软件中断的处理从 `SSIP` 置 1 开始，到 `clear_ssip()` 将 `SSIP` 置 0 结束。

    如果 Trap 处理未完成就切换：

    - 因为切换不涉及 `SIE`，新的进程继续运行在中断禁用的状态，无法响应新的中断。
    - 如果非要打开 `SIE`，那么 CPU 马上会再次触发该 Trap，再次进入 `_traps()`，重复循环下去导致栈 `sp` 溢出。

        > 你是否想起这一情况与 Lab1「动手做」探究的为什么能进入 `printk()` 类似。`printk()` 的情况比较幸运，`sp` 最初位于 OpenSBI 保护的地址，爆栈也没有破坏 OpenSBI 的代码和数据，最终走到了 `0x7???????` 之类的位置（并未探究这是什么地方，反正既不是内核也不是 OpenSBI）。但是当我们将 `sp` 设置为内核栈后，爆栈向下增长将直接破坏内核数据（data 段）和代码（text 段），导致系统崩溃。

    !!! tip

        没有理解上面论述的同学，请仔细阅读 RISC-V 特权级手册 3.1.9. Machine Interrupt (mip and mie) Registers 部分，彻底理解中断触发和清除的机制。

通过上面的讨论，我们只能将 `__switch_to()` 安排在 `trap_handler()` 的末尾。

#### 进程切换是否能被抢占

当我们这样安排 `__switch_to()` 的调用，它还能被抢占吗？请同学们思考这样的场景：

- Task 1 主动调用 `__switch_to()` 切换到其他任务，我们把这次调用称为 A
- 在这个 `__switch_to()` 过程中，时钟中断触发，进入 `_traps()` 处理
- Trap 处理完成后，因为我们安排了时钟中断触发进程切换，于是又开始调用 `__switch_to()` 切换到其他任务，我们把这次调用称为 B

请问 A 和 B 都能顺利完成吗？请你思考后再展开下面的答案。

??? note "答案"

    B 保存的上下文覆盖了 A 保存的上下文，A 再也没办法恢复了。示意图如下：

    <figure markdown="span">
        ![double_switch.drawio](lab2.assets/double_switch.drawio)
        <figcaption>
        抢占导致的双重进程切换
        </figcaption>
    </figure>

    因此 `__switch_to()` 不能被抢占，进入 `__switch_to()` 时必须禁用中断。

    > 如果整个 `_traps()` 不会触发 `__switch_to()`，那么 `__switch_to()` 应当是可以抢占的。但如果 `_traps()` 不触发进程切换，时钟中断还有意义吗？还能实现抢占吗？这一点留给有兴趣的同学思考，助教也没有深入思考过，欢迎讨论。

!!! success "恭喜你"

    至此，你已经彻底理解了**内核抢占的原理**（注意这与用户态抢占不同，仅用户态抢占的实现很容易）。Linux 在 2.6 版本才引入内核抢占。当其他班级还在 Linux 0.11 时，你已经领先了 12 年😉。

!!! info "更多资料"

    - [linux kernel - What happens to preempted interrupt handler? - Stack Overflow](https://stackoverflow.com/questions/11779397/what-happens-to-preempted-interrupt-handler)
    - [FAQ/Preemption - Linux Kernel Newbies](https://kernelnewbies.org/FAQ/Preemption)

#### 讨论所有切换情况

现在我们实现了抢占，再算上前文非抢占情况讨论的主动切换、初始上下文，一共有 $2 \times 3 = 6$ 种切换情况：

| 情况 | Task 1 状态 | Task 2 状态 | 用 `__switch_to()` 切换后 Task 2 会怎么样运行 |
| - | - | - | - |
| 1 | 抢占 | 抢占 | 通过 `_traps()` 返回，其中 `sret` 重新开启中断 |
| 2 | 抢占 | 主动切换 | 返回路径上没有其他地方开启中断 |
| 3 | 抢占 | 初始上下文 | 返回路径上没有其他地方开启中断 |
| 4 | 主动切换 | 抢占 | 通过 `_traps()` 返回，其中 `sret` 重新开启中断 |
| 5 | 主动切换 | 主动切换 | 返回路径上没有其他地方开启中断 |
| 6 | 主动切换 | 初始上下文 | 返回路径上没有其他地方开启中断 |

上一节确定了进入 `__switch_to()` 时必须禁用中断，但又会导致情况 2-3、5-6 中 Task 2 运行时中断被禁用，无法响应时钟中断，进而无法抢占。**这显然不是我们想要的结果**。

于是我们继续修补：

- 离开 `__switch_to()` 时总是开启中断，解决了情况 2、5。
- 初始上下文的蹦床函数中开启中断，解决了情况 3、6。

至此，我们彻底完成了内核抢占情况下进程切换的设计。这一部分思维量较大，建议同学们现在阅读代码理解：

- `kernel/arch/riscv/kernel/proc.c` 中的 `switch_to()` 实现
- `kernel/arch/riscv/kernel/entry.S` 中的 `__switch_to()` 和 `__kthread()` 实现

### 进程数据结构和内存布局

理解完进程切换机制，我们终于可以开始设计进程的数据结构了。我们需要存储的信息包括：

- `uint64_t pid` 进程 ID
- `struct thread_struct thread` 进程的上下文
- `void *stack` 进程的栈

    如果不给每个进程独立的栈，则会产生下图所示的局面：

    <figure markdown="span">
        ![no_stack.drawio](lab2.assets/no_stack.drawio)
        <figcaption>
        多个进程复用同一栈会导致冲突
        </figcaption>
    </figure>

    细心的同学会注意到，这样的设计也让不同进程的 Trap 上下文分离了，存储在各自的栈上。

- `struct sched_entity se` Part 4 实现的调度器使用
- `struct list_head list` 所有进程数据结构都组织成一个双向循环链表，方便调度器遍历

于是我们设计出了 `struct task_struct` 结构体来保存这些信息：

```c title="kernel/arch/riscv/include/proc.h"
struct task_struct {
    uint64_t pid;
    uint64_t state;

    struct sched_entity se;
    struct thread_struct thread;
    struct list_head list;
};
```

内存布局如下所示：

<figure markdown="span">
    ![task_struct.drawio](lab2.assets/task_struct.drawio)
    <figcaption>
    进程数据结构示意图
    </figcaption>
</figure>

此外，我们令 `tp`（RISC-V 调用约定称为 Thread Pointer）寄存器始终指向当前 CPU 上运行的进程的 `task_struct` 结构体，这样内核就能轻松地通过 `tp` 访问当前进程的信息。我们运用 Lab1 中内联汇编的知识，将该寄存器绑定到变量 `current` 方便在 C 代码中使用：

```c title="kernel/arch/riscv/include/proc.h"
register struct task_struct *current asm("tp");
```

### Task 2：设置初始进程

请补全 `kernel/arch/riscv/kernel/proc.c` 文件中的 `task_init()` 函数，实现初始化第一个内核进程（init 进程）的功能。具体要求如下：

- 使用 `kmem_cache_create()` 创建 `task_struct` 对象缓存池，并分配一个新的 `task_struct` 结构体作为初始进程。
- 初始化该进程的各项字段，包括 `pid`、`stack`、`state`、`se`（调度实体，见 Part4）等。
- 将该进程设置为当前运行进程，并将其加入进程链表（`task_list`）。

完成后，`start_kernel` 将作为第一个 Task，进而能够通过它切换到其他进程，为后续进程管理和调度打下基础。

!!! success "完成条件"

    - 通过评测框架的 `lab2 task2` 测试。本 Task 涉及调度相关字段，同学们可能需要阅读 Part4 后再回来做才能通过测试。

    本测试会在进入 `start_kernel()` 时检查 `tp` 及其指向的 `task_struct` 结构体是否正确初始化。

## Part 3：进程生命周期

本节我们实现 kthread 的创建、运行和退出，完成进程的生命周期管理。

### 进程复制与加载

Windows 使用 [`CreateProcess()`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa?redirectedfrom=MSDN) **创建进程**，这个接口庞大且僵化，还有好几个衍生版本：

```cpp title="processthreadsapi.h"
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

而 Unix 进程模型将其分为两步：

1. `fork()`：**复制**当前进程，创建一个新进程
2. `exec()`：在当前进程的上下文中**加载**并执行一个新的程序

在 `fork()` 后 `exec()` 前，可以自由配置新的进程。当内核引入新的功能时，也无需修改 `fork`/`exec` 的接口。这样**复制 + 加载**的设计比**直接创建**更灵活，体现了**组合优于复杂接口**的 Unix 哲学。

`fork()` 和 `exec()` 是提供给用户态程序的系统调用接口，将在 Lab4 进行封装。本次实验在内核态中实现相应的底层功能：

- `copy_process()`：**拷贝**当前 Task
- `kernel_thread()`：修改前者创建的 Task，在其中**运行新的函数**

### Task 3：实现进程复制与加载

请补全 `kernel/arch/riscv/kernel/proc.c` 文件中的 `copy_process()` 和 `kernel_thread()` 函数，实现内核线程的创建流程：

- `copy_process(struct task_struct *old)`：复制当前进程的数据结构，分配新的内核栈，分配唯一的进程号，并初始化调度相关字段（见 Part4），将新进程加入进程链表。
- `kernel_thread(int (*fn)(void *), void *arg)`：
    - 回忆 Part2 非抢占情况，我们讨论了如何设计初始进程上下文，通过 `struct thread_struct` 向蹦床函数传递要运行的函数和参数。
    - 和 Linux 一样，我们规定**内核线程执行的函数的接口**如下所示：

        ```c
        int fn(void *arg);
        ```

        基于此规定，框架已经实现了用于内核线程的蹦床函数 `__kthread()`，请你理解它的工作原理。

    - 请你在 `kernel_thread()` 中调用 `copy_process()` 创建新进程，并根据对 `__kthread()` 的理解设置新进程的初始上下文。

补全后，内核将能够通过 `kernel_thread()` 创建新的内核线程，并正确初始化其执行环境。

!!! success "完成条件"

    - 通过评测框架的 `lab2 task3` 测试。与 Task2 一样，本 Task 涉及调度相关字段，同学们可能需要阅读 Part4 后再回来做才能通过测试。

    本测试会检查 `start_kernel` 中调用 `kernel_thread(kthreadd, NULL)` 创建的进程上下文是否正确。

### 进程退出与销毁

当一个内核线程结束任务、退出执行，与其相关的资源需要被销毁，这包括 `struct task_struct` 结构体及其通过指针引用的成员（如内核栈）等。

**进程没有办法析构自己**，这一逻辑很容易理解。如果进程析构自身，那么析构工作也是进程运行的一部分。只要进程仍在运行状态，释放相关资源就可能导致悬空指针等问题。因此可以将进程的析构设计为退出和销毁两个阶段：

- `do_exit()`：进程完成所有任务后调用该函数，将自身状态置为 `TASK_DEAD`，然后主动发起调度，此后它再也不会被调度运行，可以安全释放
- `release_task()`：调度离开上一步的进程后，随时可以释放其资源

此外，我们还会实现一个辅助函数：

- `kthread()`：内核线程的 **Wrapper**，只是为了帮助在每个内核线程结束后调用 `do_exit()`，这样就不用让所有工作函数都显式调用 `do_exit()` 了。

    ```c
    noreturn int kthread(void *arg)
    {
        struct kthread_create_info *info = arg;
        int (*threadfn)(void *) = info->threadfn;
        void *data = info->data;
        int ret = threadfn(data);
        do_exit(ret);
    }
    ```

    一个例外是 `kthreadd` 线程（在刚刚的 Task 中见过），**它是第一个内核线程，负责创建其他内核线程**。**它永远不会退出**，因此不需要 `kthread()` 包装。

由于 `do_exit()` 依赖于调度器，这部分的任务留到 Part 4 一起完成。

## Part 4：调度器

到本节，我们终于可以把**进程切换**这个词升级为**调度**了。我们将研究调度算法，由它替选择要切换的进程。

### 时间片轮转调度

RR 调度算法已经在理论课上讲过了，这里把 PPT 内容再贴一遍：

> - 基本思路：通过时间片轮转，提高进程并发性和响应时间特性，从而提高资源利用率。
>
> - RR 算法：
>
>     - 将系统中所有的就绪进程按照 FCFS 原则，排成一个队列。
>     - 每次调度时将 CPU 分派给队首进程，让其执行一个时间片 (time slice) 。
>     - 在一个时间片结束时，暂停当前进程的执行，将其送到就绪队列的末尾，并通过上下文切换执行就绪队列的队首进程。
>     - 进程可以未使用完一个时间片，就出让 CPU（如阻塞）。

考虑 `struct sched_entity` 的设计。对于 RR 调度来说，需要保存的信息就是进程的时间片（剩余多少时间）。此外下文提及的 `test_sched.c` 测例需要使用累计时间来安排进程的创建顺序，额外新增一个字段，但它并非调度算法所必需。

```c title="kernel/arch/riscv/include/proc.h"
struct sched_entity {
    uint64_t runtime; /**< 时间片 */
    uint64_t sum_exec_runtime; /**< 已执行时间 */
};
```

### Task 4：实现时间片轮转调度和抢占

- 请补全 `kernel/arch/riscv/kernel/sched.c` 文件中的 `schedule()` 函数，实现时间片轮转调度算法。
- 请补全 `kernel/arch/riscv/kernel/proc.c` 文件中的 `do_exit()` 和 `release_task()` 函数，实现内核线程的退出与销毁。
- 在 `trap_handler()` 的末尾调用 `schedule()`，从而启用内核抢占，彻底完成本次实验。

!!! success "完成条件"

    - 通过评测框架的 `lab2 task4` 测试。
        - 该测试会检查进程退出和资源释放是否正确。
        - 该测试会在 `schedule()` 中打断点，检查最终选择的 `next_task` 是否符合上述时间片轮转调度算法计算的结果。

!!! example "动手做：计算响应比"

    除了基本的 RR 调度，实验框架还模仿了 Linux 的线程管理机制，引入了 **异步内核线程创建**。

    具体流程如下：

    - 设置一个队列，用于存放待创建的内核线程请求；
    - `kthreadd` 线程不断从该队列中取出请求，创建其他内核线程；
    - 每次创建完成后，`kthreadd` 会**立即让出 CPU**（用完时间片），因此它几乎不占用调度时间；
    - `kthread_create()` 用于提交创建请求，它会**立即返回**，不会等待线程真正启动（这点与 Linux 不同）。

    框架提供的 `test_sched.c` 模拟了理论课中的调度考题：

    | 进程 | 请求创建时间 | 运行时间 |
    | ---- | ------------ | -------- |
    | 1 | 0 | 10 |
    | 2 | 1 | 1 |
    | 3 | 4 | 2 |
    | 4 | 5 | 1 |
    | 5 | 8 | 5 |

    这里的“请求创建时间”指 `kthread_create()` 被调用的时刻，**并不代表线程立即进入就绪队列**。

    当你完成所有任务后，运行内核，观察该测例的输出结果，**计算 5 个进程的平均周转时间、平均等待时间和响应比**。

!!! example "动手做：解释时钟对调度结果的影响"

    需要说明的是，同学们应该在非调试情况下运行内核。如果开启调试，可能会导致调度结果不符合预期。这是因为 GDB 在单步执行或频繁检查寄存器、内存状态时，会显著拖慢模拟器运行，导致所有指令变慢，用体系结构的术语说就是 IPC 变低了。

    这些情况都可能导致涉及时间计算的调度算法的结果发生变化。**尝试解释为什么 RR 调度的结果可能发生变化**。

## 结语：Lab2 和 Linux 调度器的历史

其实，实验框架最开始实现的是优先级调度，但是优先级调度算法很容易造成饥饿（Starvation），与 `kthreadd` 的异步创建机制冲突。举例：

- 新的创建请求放入队列
- `kthreadd` 线程从队列中取出请求，创建新线程
- 默认情况下新线程与 `kthreadd` 线程**优先级相同**，但 PID 较大
- 在优先级相同的情况下，`schedule()` 的调度算法**始终选择 PID 较小的进程**（因为是从进程链表头开始遍历的）
- 新线程永远无法被调度运行

使用理论课上讲的老化（动态优先级）可以解决饥饿问题，但不公平（Fairness）的问题仍然十分严重，难以平衡各个进程的响应时间。比如 `kthreadd` 可能需要很长的累积等待时间才能被调度运行，严重影响内核进程异步创建的能力。最终实验框架选用了 RR 调度，**至少能平衡各个进程的响应时间**。但由于没有实现阻塞和唤醒机制，RR 调度的优势也无法体现出来。

总的来说，Lab2 的内核抢占和进程管理已经成功改革到 Linux 2.6 时代，但调度算法还没跟上，所以同学们做 Lab 时可能会觉得有些地方的实现不够合理，欢迎你来找助教讨论。

要想让 `kthreadd` 真正发挥作用，需要实现与 Linux 的**[完全公平调度器（Completely Fair Scheduler, CFS）](https://docs.kernel.org/scheduler/sched-design-CFS.html)**类似的调度算法。感兴趣的同学可以自行阅读相关资料，它最终应该会在 Bonus B 中实现。

最后简单介绍一下 [Linux 的调度算法演进史](https://codemia.io/blog/path/The-Evolution-of-Linux-CPU-Schedulers-From-O1-to-CFS-to-UserSpace-Scheduling)：

| 版本 | 调度算法 | 参考链接 |
| ---- | -------- | -------- |
| 2.6 前 | $O(n)$ 调度器（其他班级实现的） | [sched.c - kernel/sched.c - Linux source code 0.11 - Bootlin Elixir Cross Referencer](https://elixir.bootlin.com/linux/0.11/source/kernel/sched.c#L122) |
| 2.6.0-2.6.22 | $O(1)$ 调度器 | [The Linux Scheduling Algorithm - Linux Kernel Development Second Edition](https://litux.nl/mirror/kerneldevelopment/0672327201/ch04lev1sec2.html) |
| 2.6.23 - 6.6 | CFS 调度器 | [CFS Scheduler — The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-design-CFS.html) |
| 6.6 - 今 | EEVDF 调度器 | [EEVDF Scheduler — The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-eevdf.html) |
| 开发中 | `sched_ext` 通过 BPF 程序定制任意调度算法 | [Extensible Scheduler Class — The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-ext.html) |
