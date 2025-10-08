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

    画一幅图，展示空闲链表缓存相关的数据结构及其布局。

## Part 2：进程生命周期管理

### 进程上下文与数据结构

设想这样一个场景：Task1 正在 CPU 上运行 ，现在想将 Task2 调度上去，并在一段时间后再切换回来继续执行 Task1。应该怎么做？

其实 Lab1 中断处理已经实现了切换的功能：

- 发生中断时，上下文被 `_traps()` 保存到栈上，然后 `trap_handler()` 上去执行
- 中断处理完毕后，从栈上恢复中断上下文

进程上下文与中断上下文类似，但有两点区别：

- **需要保存的寄存器更少**，这是由下面两个条件保证：

    - 稍后我们将实现 `__switch_to()`，**所有进程切换都将通过它实现**。这意味着再次调度时将从它原路返回。
    - `__switch_to()` 的调用者都遵守 RISC-V 调用约定。

    如下图所示：

    ![switch.drawio](lab2.assets/switch.drawio)

    于是进程上下文实现为下面的结构体：

    ```c
    struct thread_struct {
        uint64 ra;      // return address
        uint64 sp;      // stack pointer
        uint64 s[12];   // callee-saved registers
    };
    ```

- **存放在独立的位置**：

    中断处理时可以简单地复用当前栈 `sp` 来保存中断下文，但进程上下文之间不可以复用栈，否则会出现下面的情况：

    ![no_stack.drawio](lab2.assets/no_stack.drawio)

    我们使用 Buddy System 为每个进程分配一页内存作为栈，并将 `thread_struct` 放在栈的另一端。

进程除了上下文还有 ID、状态、优先级等信息，它们将存储在 `struct task_struct` 中，在理论课上称之为 PCB（Process Control Block）。最终，这些数据结构的布局如下图所示：

<figure markdown="span">
    ![task_struct.drawio](lab2.assets/task_struct.drawio)
    <figcaption>
    进程数据结构示意图
    </figcaption>
</figure>

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

## Part 3：调度器

### 优先级调度

### Task 4：实现优先级调度
