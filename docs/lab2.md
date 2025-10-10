# Lab 2：单核内核线程调度

!!! danger "DDL"

    本实验文档正在编写中，尚未正式发布。

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

在 Lab1 中，我们已经让内核能够启动并处理时钟中断。但是如果没有进一步的机制，操作系统仍然只能在 CPU 上执行单个任务（即 `start_kernel` 函数）。本次实验的目标是：**实现单核内核线程调度，让多个任务能够在一个 CPU 上通过时间复用的方式交替运行。**

要完成这一目标，我们需要逐步解决以下几个问题：

1. **如何在内核中管理物理内存？**（理论课第八章内容 - 主存）

    在后续实验里，我们要为进程、线程分配栈和控制块。这些数据结构必须存放在内存里。但是操作系统不能直接调用 `malloc` 等标准库函数，那么问题来了：

    - **操作系统如何在裸机环境下动态分配内存？**
    - **为什么我们不能直接线性地“挖一块”内存？碎片化如何解决？**

    实验框架提供了一个简化版 **Buddy System 物理内存分配器**。同学们需要理解它的基本原理，并学会通过接口分配和释放物理页，为后续任务管理打下基础。

    Buddy System 仅能实现物理页分配，**无法满足对象粒度的分配需求**。因此，我们还需要实现一个**空闲链表缓存（free list cache）**，用于高效地分配内核对象。

2. **如何管理进程？**（理论课第三章内容 - 进程）

    任务交替运行的核心在于**上下文切换（context switch）**。要让任务 A 运行完一段时间后交出 CPU，再切换到任务 B，必须保存 A 的上下文，并在之后恢复它。

    - **上下文具体包含哪些内容？**
    - **RISC-V 调用约定和中断机制如何影响上下文切换？**

    这一步，我们需要为每个 Task 设计一个数据结构（即理论课介绍的 PCB，Process Control Block），用于保存任务的上下文和其他属性。

    此外，我们还需要实现进程的创建和销毁机制，从而让内核能够动态地管理多个任务。

3. **调度器应该如何工作？**（理论课第五章内容 - 调度算法）

    有了内存和进程管理，接下来就能够实现调度器。调度器的任务是：

    1. 根据调度算法选择下一个要运行的任务
    2. 调用上下文切换代码，将 CPU 控制权交给它

    本次实验中，我们将先实现一个简单的**优先级调度器**。之后，同学们可以在此基础上扩展更复杂的调度算法（如多级反馈队列）。

## Part 0：合并代码

现在你位于 `lab1` 分支。你需要创建 `lab2` 分支，合并上游的代码：

```shell
git checkout -b lab2
git fetch upstream
git merge upstream/lab2
```

合并说明：

- 新增内存管理代码，见 `mm.h`
- 新增进程管理相关实验代码，见 `proc.h`

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

            在计科的硬件课程中，大家可能还没有系统学习过**总线 (Bus)** 的概念（图灵班的计算机系统贯通课已经在实验中加入了 AXI 总线）。在这里我们只需要掌握最基本的理解：

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

常用接口如下：

- **初始化**

    ```c
    void mm_init(void);
    ```

    在系统启动时调用，完成 buddy allocator 的初始化工作。

- **分配物理页**

    ```c
    void *alloc_page(void);          // 分配 1 页
    void *alloc_pages(size_t n);     // 分配 n 页（n 向上取整为 2 的幂）
    ```

    返回值是一个 **物理地址**，指向分配到的页首地址。

- **释放物理页**

    ```c
    void free_pages(void *pa);
    ```

    参数 `pa` 是一个物理地址，必须是 `alloc_page(s)` 的返回值，否则可能破坏 buddy 系统的状态。

在实验里，你只需要通过这些接口申请或释放物理页，而不必关心内部具体是如何实现的。

例如：

```c
void *a = alloc_page();      // 分配一页
void *b = alloc_pages(8);    // 分配 8 页
free_pages(a);               // 释放之前分配的一页
```

通过这一套接口，我们的内核就能在后续实验（如进程调度、内存管理、虚拟内存）中，稳定地使用物理页作为底层资源。

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

    ```c
    #define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr); \
        (type *)( (char *)__mptr - offsetof(type,member) );})
    ```

    编译器知道类型的大小，提供了 `typeof()` 和 `offestof()`。我们可以利用这些信息**移动成员变量的指针，计算出包含该成员变量的结构体的起始地址**。例如：

    ```c
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

### Task 1：实现双向链表

你应该早在大一的 C 语言课上就能秒杀双向链表了😉。请完成下列任务：

- 补全 `list_empty()`。提示：观察 `LIST_HEAD` 宏，你会发现这是一个**带哨兵**的双向链表。
- 补全 `list_add()`、`list_add_tail()` 和 `list_del()`。提示：一个优雅的做法是令 `list_add()` 和 `list_add_tail()` 都调用一个更底层的 `__list_add()`。
- 补全 `list_for_each` 宏。

`main.c` 中的 `test_list()` 函数用于测试你的链表实现。

!!! success "完成条件"

    - 通过评测框架的 `lab2 task1` 测试。

### 空闲链表缓存

在上一节中，我们学习了如何实现**侵入式双向循环链表**。现在，我们将用它来实现一个内核中非常常见的机制——**对象缓存（object cache）**。

在内核中，许多内核对象（如 `task_struct`、`inode`、`vm_area_struct` 等）会被**频繁创建和销毁**。如果动态创建都调用通用的内存分配器（如 Buddy System），会对性能产生较大影响，且容易引起内存碎片。

为了解决这些问题，Linux 内核引入了 **Slab 分配器**：它为每种类型的对象维护一个缓存池，将整页内存切分为多个固定大小的对象块，用链表管理这些空闲对象。

不过，Linux 的 Slab 实现更加复杂，感兴趣的同学可以阅读 [Slab Layer - Linux Kernel Development Second Edition](https://litux.nl/mirror/kerneldevelopment/0672327201/ch11lev1sec6.html)。在我们的教学实验中，只保留了它的**核心思想：用空闲链表（free list）预分配缓存对象。**

- **`kmem_cache` 结构**：为每种类型的对象建立的缓存池

    它记录了对象大小、对齐方式以及一个空闲链表：

    ```c
    struct kmem_cache {
        const char *name;           // 缓存名称
        size_t obj_size;            // 对象大小
        size_t align;               // 对齐要求
        struct list_head list;      // 链入全局缓存链表
        struct list_head free_objects; // 空闲对象链表
    };
    ```

    所有空闲的 `kmem_cache` 结构本身也组织为空闲列表 `free_caches`，由 `cache_init()` 在启动时初始化。

- **创建缓存**：

    当内核需要为某种类型创建对象缓存时：

    ```c
    struct kmem_cache *task_cache =
        kmem_cache_create("task_struct", sizeof(struct task_struct), 8);
    ```

    系统会从 `free_caches` 中取出一个空闲条目，初始化为新的缓存描述符。

- **分配与释放对象**：

    当我们需要真正分配一个对象时：

    ```c
    struct task_struct *t = kmem_cache_alloc(task_cache);
    ```

    执行流程如下：

    1. 检查 `task_cache` 的空闲链表是否为空；
    2. 如果为空，从 Buddy System 申请一页新的物理内存；
        - 按对象大小（外加链表节点头）将该页切分为多个小对象；
        - 将这些对象逐个加入 `cachep->free_objects` 链表；
    3. 从链表头取出一个对象返回；

    `kmem_cache_free(t)` 释放对象时，只需将其重新挂回空闲链表。

换句话说，Slab 缓存层（`kmem_cache`）相当于物理页分配器（`alloc_page()`）之上的对象粒度的再分配层。

!!! example "动手做"

    阅读 `cache.h` 和 `cache.c`，理解空闲链表缓存的实现。

    画一幅图，展示相关的数据结构及其布局。

## Part 2：进程切换与 Trap 处理

本节我们探究实现用于切换进程上下文的汇编函数 `__switch_to()`，以及进程切换与 Trap 处理的相互影响。

### 进程切换：非抢占情况

让我们从最简单的不考虑 Trap 的情况开始。这种情况下没有时钟中断，进程切换仅能依靠进程主动调用 `__switch_to()` 完成。

设想这样一个场景：CPU 上正在运行 Task1，Task1 想调用 `__switch_to()` 切换到 Task2，然后 Task2 运行一段时间后再切换回来继续执行 Task1。`__switch_to()` 应该怎么设计？

Lab 1 Trap 处理的场景与此类似：可以将 `_traps()` 看作 `__switch_to()`，将 `trap_handler()` 看作 Task2。`trap_handler()` 执行完毕后，恢复 Trap 上下文就相当于 Task2 切换回 Task1。**但进程切换与 Trap 有一个重要区别：Trap 可以发生在任何时候**，因此 Trap 上下文是所有寄存器的状态；而 `__switch_to()` 是一个函数，**它的调用者一定是遵守 RISC-V 调用约定的**，因此不需要保存所有寄存器。示意图如下：

![switch.drawio](lab2.assets/switch.drawio)

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

那么这个结构体应该存放在哪里呢？暂且按下不表，留到设计进程数据结构时再说。反正不能像 Trap 上下文那样放在栈 `sp` 上，否则会出现下面的情况：

<figure markdown="span">
    ![no_stack.drawio](lab2.assets/no_stack.drawio)
    <figcaption>
    进程上下文不能放在栈上
    </figcaption>
</figure>

于是我们实现了 `__switch_to()`：

```asm title="第一版 __switch_to"
    .globl __switch_to
__switch_to:
    la t0, /* 存放 Task1 上下文的地址 */
    sd ra, 0(t0)          // 保存 Task1 上下文
    sd sp, 8(t0)
    sd s0, 16(t0)
    ...

    la t0, /* 存放 Task2 上下文的地址 */
    ld ra, 0(t0)          // 恢复 Task2 上下文
    ld sp, 8(t0)
    ld s0, 16(t0)
    ...
    ret                    // 返回到 Task2
```

> 我们还能再换个角度：站在 Task1 的角度看，调用 `__switch_to()` **就像调用了一个空函数**。它遵守 RISC-V 调用约定，但什么都没做，原路返回了。

但嗅觉敏锐的同学会发现问题：切换的目标 Task 2 的上下文从哪里来？在目前不考虑 Trap 的场景，只有两种可能：

1. Task 2 之前运行过，但主动调用 `__switch_to()` 切换到别的进程。这种情况**保存过进程上下文**，能够顺利恢复。
2. Task 2 是新创建的进程，尚未运行过。这种情况需要我们**设计一个初始的进程上下文**。

初始进程上下文的设计想想也很简单：`ra` 指向进程的第一条指令就行了。如果要考虑向进程传递参数，可以利用 `s0-s11` 这些寄存器，然后设置一个蹦床函数（trampoline）来将其移动到 `a0-a7` 作为参数。事实上 Linux 就是这么实现 kthread 的初始上下文的，这个蹦床函数如下所示：

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

从代码中可以看出，Linux 将 kthread 要运行的函数 `fn` 放在了初始进程上下文的 `s0` 寄存器中，将参数 `fn_arg` 放在了 `s1` 寄存器中。蹦床函数 `ret_from_fork_kernel_asm` 将它们分别移动到 `a1` 和 `a0`，然后调用 `ret_from_fork_kernel` 去具体执行。简化一下就是这样：

```asm
move a0, s1  // fn_arg
move a1, s0  // fn
call a1      // 调用 fn(fn_arg)
```

### 进程切换：抢占与 Trap 处理

当我们希望实现[内核抢占（Kernel preemption）](https://kernelnewbies.org/FAQ/Preemption)，综合考虑时钟中断、系统调用等 Trap 情况时，进程切换就变得复杂起来。

#### 内核中不能被抢占的部分

首先，我们要明确内核中不能被抢占的部分。抢占一定是通过（时钟）中断触发的，此外没有其他途径，因此**不能抢占 = 禁用中断**。回忆 RISC-V 中断机制，进入 Trap 处理时，CPU 会自动执行下面的操作禁用中断：

- `sstatus.SPIE = sstatus.SIE`
- `sstatus.SIE = 0`

除了 Trap 处理，暂时我们没有想到其他不能被抢占的部分。

#### `_trap()` 中可以抢占的部分

接下来思考：整个 `_traps()` 都不能被抢占吗？一定要等到它 `sret` 恢复 `SIE` 吗？

- 不是的，Trap 的处理过程本质上是从 `xIP.i` 置 1、中断 Pending 开始，到该 Pending 位被置 0 为止。

    以 Lab1 涉及的 Trap 情况为例：时钟中断的处理从 SEE 发现 `stimecmp > stime` 将 `STIP` 置 1 开始，到 `sbi_set_timer()` 将 `STIP` 置 0 结束。软件中断的处理从 `SSIP` 置 1 开始，到 `clear_ssip()` 将 `SSIP` 置 0 结束。

- 一旦 Pending 位被置 0（Trap 处理完成），就可以打开 `sstatus.SIE` 处理新的 Trap，并不需要等到 `sret` 恢复 `SIE`。

    如果 Trap 处理未完成，能打开 `sstatus.SIE` 吗？一旦打开，CPU 马上会再次触发该 Trap，再次进入 `_traps()`，重复循环下去导致栈 `sp` 溢出。

    > 你是否想起这一情况与 Lab1 「动手做」探究的为什么能进入 `printk()` 类似。`printk()` 的情况比较幸运，`sp` 最初位于 OpenSBI 保护的地址，爆栈也没有破坏 OpenSBI 的代码和数据，最终走到了 `0x7???????` 之类的位置（并未探究这是什么地方，反正既不是内核也不是 OpenSBI）。但是当我们将 `sp` 设置为内核栈后，爆栈向下增长将直接破坏内核数据（data 段）和代码（text 段），导致系统崩溃。

    在 Trap 处理完成后立刻打开 `SIE`，会导致 `_traps()` 的嵌套调用，最终它们能正确返回吗？

    - 答案是肯定的，就如上一节所讨论的，设计 `_traps()` 时我们考虑到它会在任何时候被调用，因此 Trap 上下文包括了所有整数寄存器和 sepc。设刚刚处理完的中断为 A，新的中断为 B，B 在 A 处理完到 `sret` 的过程中触发，A 此时的状态将被保存到栈上，B 处理完毕后从栈上恢复 A 的状态，继续执行 A 的 `sret`，一切正常。
    - 唯一的问题是 Trap 处理程序不能太长，否则开启 `SIE` 时总是又触发时钟中断 Trap，又会导致栈溢出。在 Linux 中，为了保证 Trap 的快速响应，仅会在 Trap 处理程序中做必要的工作，其余工作放入任务队列延后处理。

    > 细心的同学可能会追问 `scause`、`stval` 等其他 CSR 寄存器为什么不视为 Trap 上下文。它们的作用就是辅助中断处理，而中断处理过程是不能抢占的，它们不会丢失。处理完了它们的值就不重要了，仅剩 `sepc` 用于恢复到中断前的位置。

没有理解上面两段论述的同学，请仔细阅读 RISC-V 特权级手册 3.1.9. Machine Interrupt (mip and mie) Registers 部分，彻底理解中断触发和清除的机制。

#### 时钟中断与进程切换时机

最后，我们考虑时钟中断触发的进程切换（`__switch_to()`）应该在什么时候发生。显然它应该在 `SIE` 开启之后，Trap 上下文恢复之前。这一步可以用排除法来思考：

- Trap 处理过程不能被打断，当然也不能 `__switch_to()`。
- 如果在 `SIE` 开启之前切换，进程上下文的切换并不会修改 `sstatus`，新的进程就继续运行在中断禁用的状态，无法响应新的中断。
- 如果在 Trap 上下文恢复过程中切换进程，就破坏 Trap 上下文了。
- Trap 上下文恢复完成后，就没机会切换进程了。

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

    因此 `__switch_to()` 不能被抢占，进入 `__switch_to()` 时应当禁用中断。

    > 如果整个 `_traps()` 不会触发 `__switch_to()`，那么 `__switch_to()` 应当是可以抢占的。但如果 `_traps()` 不触发进程切换，时钟中断还有意义吗？还能实现抢占吗？这一点留给有兴趣的同学思考，助教也没有深入思考过，欢迎讨论。

!!! success "恭喜你"

    至此，你已经彻底理解了**内核抢占的原理**（注意这与用户态抢占不同，仅用户态抢占的实现很容易）。Linux 在 2.6 版本才引入内核抢占。当其他班级还在 Linux 0.11 时，你已经领先了 12 年😉。

!!! info "更多资料"

    - [linux kernel - What happens to preempted interrupt handler? - Stack Overflow](https://stackoverflow.com/questions/11779397/what-happens-to-preempted-interrupt-handler)

### 进程数据结构和内存布局

理解完进程切换机制，我们终于可以开始设计进程的数据结构了。我们需要存储的信息包括：

- 进程的上下文 `struct thread_struct`
- 进程的栈 `void *stack`

    如果不给每个进程独立的栈，则会产生下图所示的局面：

    <figure markdown="span">
        ![no_stack.drawio](lab2.assets/no_stack.drawio)
        <figcaption>
        多个进程复用同一栈会导致冲突
        </figcaption>
    </figure>

    细心的同学会注意到，这样的设计也让不同进程的 Trap 上下文分离了，存储在各自的栈上。预告一下，在后续的 Lab 中我们会继续完善 Trap 处理，让它不再复用 `sp` 保存上下文。

- 进程 ID、状态、优先级等调度相关的信息，供 Part 4 实现的调度器使用
- 所有进程数据结构都组织成一个双向循环链表，方便调度器遍历

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

此外，我们令 `tp`（RISC-V 调用约定称为 Thread Pointer）寄存器始终指向当前 CPU 上运行的进程的 `task_struct` 结构体，这样内核就能轻松地通过 `tp` 访问当前进程的信息。

### Task 2：设置初始进程

- 在 `head.S` 将 `init_task` 的地址加载到 `tp` 寄存器中
- 在 `proc.c` 设置 `init_task` 中设置合适字段（读到这里好像只能设置 `stack`）

### Task 3：重构 Trap Handler

请同学们根据本 Part 学习到的知识，重构 Lab1 的 Trap 处理代码，使其在第一个时钟中断处理完成后从自己切换到自己。要点如下：

- 在中断处理完成后允许抢占
- 在 `trap_handler()`

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

在 `fork()` 后 `exec()` 前，可以自由配置新的进程。当内核引入新的功能时，也无需修改 `fork`/`exec` 的接口。这样**复制+加载**的设计比**直接创建**更灵活，体现了**组合优于复杂接口**的 Unix 哲学。

`fork()` 和 `exec()` 是提供给用户态程序的系统调用接口，将在 Lab4 进行封装。本次实验在内核态中实现相应的底层功能：

- `task_init()`：为第一个 Task（即 `start_kernel()`）**初始化**相关资源
- `copy_process()`：**拷贝**当前 Task
- `kernel_thread()`：修改前者创建的 Task，在其中**运行新的函数**
- `switch_to()`：**切换**到另一个 Task
- `do_exit()`：在 Task 结束时调用，**释放**相关资源

### Task 2：实现内核线程创建

### 进程切换

### Task 3：进程退出

## Part 4：调度器

到本节，我们终于可以把「进程切换（switch）」这个词升级为「调度（schedule）」了，因为我们有了调度算法，由它替我们选择要切换的进程。

### 优先级调度

### Task 4：实现优先级调度
