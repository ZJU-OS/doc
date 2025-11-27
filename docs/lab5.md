# Lab 5：缺页异常与 Fork 机制

!!! danger "DDL"

    - 代码、报告：2025-12-16 23:59
    - 验收：2025-12-23 实验课

## 实验简介

在 Lab2 的“进程生命周期”一节中，同学们已经了解了 Unix 进程模型将进程的创建分为 `fork()` 和 `exec()` 两个步骤。本实验将**实现其中的 `fork()` 机制，让进程能够复制自身**。

其实 `fork()` 已经基本做完了，就是前几个 Lab 实现的 `copy_process()`，现在考虑对其进行优化。Lab4 实现了 `copy_pgd()` 来对进程页表及其映射的内存执行深拷贝，但容易发现**父进程的内存往往并不需要被完全复制到子进程中**，一些例子如下：

- 子进程创建后立刻调用 `exec()` 加载一个新的程序，那么父进程的大部分内存空间都用不到（可能仅有少量的栈空间用于传递参数）。这些内容不需要复制；
- 子进程可能并不会修改父进程的内存内容，例如只读的代码段、共享库等。这些内容完全可以被父子进程共享。

于是我们为进程引入**写时复制（Copy-On-Write, COW）**机制：

- `fork()` 时，父子进程共享内存页并设为**只读**；如果多次 `fork()`，那么这些进程都将共享同一份内存页；
- 当某个进程尝试写入一个共享页时，将触发 Store/AMO page fault，此时再为该进程创建该页的私有副本。

注意到上面的操作在页表中将所有内存页都设为只读，丢失了进程原本的内存权限信息。于是在页表之上，我们引入**虚拟内存区域（Virtual Memory Area, VMA）**来描述进程的内存布局（映射的区域及其权限），将其加入到进程的结构体中。

同样的优化也可以应用在进程初次加载（即 `load_elf_binary()`）。不需要在此时拷贝、映射所有内存页，而是**只创建 VMA** 进行记录，等到进程访问某个页时再拷贝、映射。这称为**按需分页加载（Demand Paging）**。

## Part 0：准备工作

### 更新代码

现在你位于 `lab4` 分支。你需要创建 `lab5` 分支，合并上游的代码：

```shell
git checkout -b lab5
git fetch upstream
git merge upstream/lab5
```

下面的合并说明供同学们解决合并冲突时参考。

### Lab5 的测试

`kernel/user` 下均为我们提供的用于测试的用户态程序，同学们使用过程中不应自行更改。如果你修改过相关内容，则合并时可能会产生冲突。

Lab4 仅使用一个用户态程序进行测试，而 Lab5 需要测试多个不同的用户态程序，因此我们修改了 `kernel/user` 下的编译过程。

- `kernel/user/lib` 是用户态程序的公共库文件夹，包含了标准库函数（如 `printf()`）和系统调用的封装（如 `fork()`）。
- `kernel/user/src` 下的每一个文件都是一个独立的用户态程序。
- `kernel/user` 的 Makefile 会先将上面两个文件夹中的所有源文件编译成对应的 `.o` 目标文件，然后对 `src` 下的每一个用户态程序进行链接，生成对应的 `.elf` 文件。
- `kernel/user/uapp.sh` 会生成汇编文件 `uapp.S`，用于将上述 `.elf` 文件打包到 ELF 文件中的 `uapp` 段：

    ```assembly title="生成的 uapp.S 示例"
    .section .uapp
        .ascii "UAPP\0"
        # uapp table: offset of app1, offset of app2, ..., 0
    .Luapp_table:
        .quad .Llab4 - .Luapp_table
        .quad .Llab5_cow - .Luapp_table
        .quad 0
    .Llab4:
        .incbin "src/lab4.elf"
    .Llab5_cow:
        .incbin "src/lab5_cow.elf"
        # ...
    .Lend:
    ```

    这段汇编代码用汇编器指令填充 `.uapp` 段的内容，首先是一个标识符 `UAPP\0`，然后是一个应用程序表（`uapp table`），记录了每个用户态程序在该段中的偏移，最后是各个用户态程序的二进制内容。

- `uapp.S` 被编译为 `uapp.o`，然后加入内核的链接过程。这部分和 Lab4 一样。

相应地，在 `start_kernel()` 中，按照类似加载 ELF 文件那样的步骤找到每个 ELF 文件的起始位置，然后调用 `user_mode_thread()` 创建用户态进程。

```c title="kernel/arch/riscv/kernel/main.c"
while ((app_offset = app_table[app_index++])) {
    void *app_addr = (void *)app_table + app_offset;
    struct task_struct *app_proc = user_mode_thread(app_addr);
    app_proc->wait_comp = &common_comp;
    wait_for_completion(&common_comp);
}
```

Lab5 一共有四个测试需要通过：

- Lab4 的测试程序，它会复制多份然后同时运行，并且同时还有一个内核线程在运行，用于测试用户、内核态混合调度下的正确性
- Lab5 Copy-On-Write 测试程序
- Lab5 斐波那契数组测试程序
- Lab5 多次 Fork 测试程序

实验过程中，你可以自己选择调整 `main.c` 中创建的进程，只运行部分测试程序以节省调试时间。最后提交时，autograder 要求按上面的流程完成所有测试。

### Completion 机制

我们希望 Lab5 的测试 app 逐个顺序运行，而不是像 Lab4 那样同时运行多个进程。因此我们引入了一个简单的 **Completion 机制**，用于等待某个事件完成。我们的简单实现和理论课介绍的**信号量（Semaphore）**类似，而在 Linux 内核中的实现则复杂得多。

相关源代码位于 `kernel/arch/riscv/include/completion.h` 和 `kernel/arch/riscv/kernel/completion.c`，请同学们**务必自行查看理解**。简单来说：

- `wait_for_completion()` 会将当前进程状态置为 `TASK_SLEEPING`，调度器将忽略它
- `complete()` 会找到等待该事件的进程，将其状态置为 `TASK_RUNNING`，从而能够再次被调度器调度

这里引入了新的进程状态 `TASK_SLEEPING`，请同学们回顾自己的调度器实现，可能需要做一些修改以支持该机制。

### 分页虚拟内存的问题

Lab4 的 `write()` 系统调用其实存在严重的 Bug。同学们应当能够理解，虚拟内存中连续的虚拟地址不一定映射到连续的物理地址上。如果 `write()` 调用中传入的缓冲区跨页了，那么 Lab4 的实现会怎么做呢？

<figure markdown="span">
    ![lab4_write_bug.drawio](lab5.assets/lab4_write_bug.drawio)
    <figcaption>
    Lab4 中的 write 跨页问题
    </figcaption>
</figure>

OpenSBI 会输出一些正确的内容，随后的行为无法预知，取决于遇到什么数据。

为了解决该问题，我们引入了 `copy_from_user()`，用于从用户空间复制数据到内核空间。Lab4 实现的 `UVA2PA()` 直接删去。

### 引入 `PANIC()` 宏

引入了 `PANIC()` 宏，它的功能很简单：当内核遇到无法处理的错误时，调用该宏输出错误信息并进入无限循环，便于调试查看情况。

## Part 1：虚拟内存区域 (VMA)

### 引入 VMA

如实验简介所说，VMA 本质上是对页表的一种补充和抽象。

- 与**树形结构、逐页记录**的页表相比，VMA 是**链表结构、按区域记录**的，抽象程度更高。
- 当页表被用于支持 CoW、按需加载等功能时，VMA 作为补充，能够描述内存是否能被映射、应该加载什么内容、权限是什么。

<figure markdown="span">
    ![vma.drawio](lab5.assets/vma.drawio){ width="100" }
    <figcaption>
    VMA 与页表的对比
    </figcaption>
</figure>

考虑“应该加载什么内容”这一点，在课程实验中可能有两种来源：

- **文件映射**：意思是这段内存空间的内容来自某个文件，比如进程的可执行文件、共享库文件等
- **匿名映射**：内核分配的一段内存，没有所谓“来源”，比如进程的栈空间、堆空间等

### VMA 的设计

VMA 的数据结构定义如下：

```c title="kernel/arch/riscv/include/vma.h"
struct vm_area_struct {
    struct list_head list;
    uint64_t vm_start; /**< 虚拟内存起始地址 */
    uint64_t vm_end;   /**< 虚拟内存结束地址 */
    uint64_t vm_flags; /**< 权限标志 */
    /* 文件映射相关信息 */
    void *vm_file; /**< 文件指针 */
    uint64_t vm_filesz; /**< 文件大小 */
    uint64_t vm_pgoff; /**< 文件内偏移 */
};
```

我们把 VMA 和页表这些用于内存管理的数据结构组合为 `mm_struct` 结构体嵌入到 `task_struct` 中：

```c title="kernel/arch/riscv/include/proc.h"
struct mm_struct {
    pgd_t *pgd; /**< 根页表 */
    struct list_head mmap; /**< VMA 链表 */
};
```

关于 VMA 的使用，这里做一些补充说明：

- 我们已经知道所有进程的页表中都映射了内核空间，因此不需要为内核空间创建 VMA。VMA 只描述用户空间的内存布局。自然地，内核线程（如 `kthreadd`）不需要 VMA。
- `vm_file` 等三个字段用于描述文件映射的 VMA 的信息。如果 `vm_file` 为 NULL，则说明该 VMA 不是文件映射的；如果是文件映射，操作系统可以根据三个字段知道要加载的内容。
- VMA 的权限位（`vm_flags`）和 PTE 中的的 flags 并不完全相同，需要进行转换。

!!! info "扩展阅读：Linux 中的 VMA"

    同学们可以在 Linux 下运行 `cat /proc/1/maps` 命令，查询 PID 为 1 的进程的内存映射信息（也就是 VMA）。输出的内容的格式见 [proc_pid_maps(5)](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html)。

    今天的应用程序体量可能很庞大，Linux 内核采用 Maple Tree（一种 B-Tree）来高效地索引和查找 VMA。感兴趣的同学可以阅读下面的材料进一步了解。

    - Linux 中 VMA 的定义与设计：[linux/mm_types.h](https://elixir.bootlin.com/linux/v6.17/source/include/linux/mm_types.h#L815), [Process Addresses](https://docs.kernel.org/mm/process_addrs.html)
    - Linux 中管理 VMA 的实际挑战：[The Maple Tree, A Modern Data Structure for a Complex Problem](https://blogs.oracle.com/linux/the-maple-tree-a-modern-data-structure-for-a-complex-problem)

### Task1：实现 VMA

- 实现 VMA 的创建与管理，见 `kernel/arch/riscv/kernel/vma.c`：

    - **我们提供** `copy_vma()` 和 `free_vma()` 两个函数，分别用于拷贝和释放进程的 VMA 链表。
    - **请你补全** `do_mmap()`。该函数接收一个进程的 `mm_struct` 以及一段虚拟地址范围和权限等信息，创建一个新的 VMA 并将其添加到进程的 VMA 链表中。
    - **请你补全** `find_vma()`。该函数接收一个进程的 `mm_struct` 以及一个虚拟地址，查找并返回包含该地址的 VMA。如果没有找到则返回 NULL。

- 相应地，你需要修改进程的生命周期管理，实现对 VMA 的维护，见 `kernel/arch/riscv/kernel/proc.c`：

    - **请你修改** `task_init()`, `copy_process()` 和 `release_task()` 函数，实现对 VMA 的初始化、拷贝和释放。
    - 另外，由于我们移动了 `pgd` 到 `mm_struct` 中，你还需要修改相关代码以适应这一变化。

!!! success "完成条件"

    完成该 Task 后，由于我们还没有实现按需分页加载，VMA 中其实不会有任何内容。Lab4 的测试程序应该正常运行。

## Part 2：按需分页加载（Demand Paging）

### 按需加载的动机

有同学可能会问：按需加载只是将加载时机从创建时推迟到了访问时，而且这种方式引入了工程上额外的复杂度，它真的是必要的吗？下面我们来分析一下按需加载的必要性。同学们可以思考一下下面两个场景：

- 今天我们的操作系统中可能同时运行着成百上千个进程，我们可以自由地在进程间切换，仿佛它们都在运行。而如果我们计算一下这些程序的总大小（可以想象一下如果我们需要把整个 OS 实验环境的镜像（它有 11GB 之大）加载入内存，还要同时运行浏览器访问实验文档、打开 VSCODE 编写代码），它已经远远超过了我们机器的物理内存大小了。如果我们仍然采用一次性加载的方式，那么我们根本无法同时运行这么多程序。
- 一个程序在运行过程中，**创建的内存空间往往远大于它实际使用的内存空间**。例如，一个程序可能会申请一个 1GB 的缓冲区用于处理数据，但在实际运行过程中，它可能只会使用其中的几 MB。采用一次性加载的方式会浪费大量的内存资源。

针对**第一个场景**，同学们可能已经在课上学习了虚拟内存中的**换页** (page swapping) 机制。如果所有的进程都是必要的，那么当物理内存容量不够时，操作系统就会把若干物理页的内容写到类似于磁盘这种更低内存层级的存储介质中，然后回收这些物理页以供其他进程使用。由于我们目前的实验完全是 in memory 的，因此我们暂时不考虑换页机制。而针对**第二个场景**，**按需分页加载**就能很好地解决这个问题。

但无论上上述的哪一个机制，都离不开**缺页异常** (page fault) 的触发与处理，下面我们进行详细介绍。

### 缺页异常（Page Fault）

现在我们已经完成了按需分页加载的准备工作———VMA 虚拟内存区域的设计与实现。那么，**按需加载**究竟是如何实现的呢？

同学们在调试之前实验的过程中一定遇到过类似下面的报错：

```text title="Trap Handler 捕获 Store Page Fault"
[ERR, PID = 1] trap.c:149:trap_handler: Unhandled trap: scause = Store/AMO page fault (0xf), sepc = 0xffffffd60020187c, stval = 0xffffffffffffffc0
```

这是由于在一个启用了虚拟内存的系统上，当正在运行的程序访问一个页表中没有映射的虚拟地址时，MMU 尝试通过页表进行映射，发现无法找到对应的物理地址，或者找到了物理地址，但是访问权限不够（例如对只读页进行写操作），就会触发一个**缺页异常（Page Fault）**。VMA 的引入让一个虚拟地址多了一种新的状态——**已分配但未映射**，可以想象的是，当我们实现用户态程序的按需加载，它在第一次访问某个地址时，由于页表项未映射，肯定也会触发缺页异常。

在 RISC-V 架构下，我们有如下三种缺页异常：

<div style="text-align: center" markdown="1">

| Interrupt | Exception Code | Description |
| :-: | :-: | --- |
| 0 | 12 | Instruction Page Fault |
| 0 | 13 | Load Page Fault |
| 0 | 15 | Store/AMO Page Fault |

</div>

在本次实验中，我们将在 `trap.c` 中实现 `do_page_fault()` 函数，它会捕获所有的缺页异常，并通过一些方式区分按需加载和非法访问的情况，并进行相应的处理。而区分方法也非常简单，那就是**检查缺页异常发生时的地址是否在某个 VMA 范围内**。如果在，就说明这是一个合法的按需加载请求，我们就根据 VMA 的信息来加载对应的页面；如果不在，就说明这是一个非法访问，我们就进行相应的错误处理。

### Task 2：实现缺页异常处理

<figure markdown="span">
    ![page_fault.drawio](lab5.assets/page_fault.drawio){ width="100" }
    <figcaption>
    缺页异常处理流程图
    </figcaption>
</figure>

- 实现缺页异常的处理逻辑，见 `kernel/arch/riscv/kernel/trap.c`：
    - **请你补全** `do_page_fault()` 函数，实现上述的缺页异常处理逻辑。
- 实现按需加载，见 `kernel/arch/riscv/kernel/binfmt_elf.c`：
    - **请你修改** `load_elf_binary()` 函数，使其**不再直接为 ELF 段分配物理内存和创建映射**，而是**为每个段创建对应的 VMA 并通过** `do_mmap()` **添加到进程的 VMA 链表中**。

!!! tip "缺页异常处理流程"

    如果觉得缺页异常处理流程比较抽象，可以用下面的几个组件对话来帮助理解：

    1. 用户程序：我想要访问 `0x0` 这个地址。
    2. MMU：让我看看**页表**……咦？没有映射这个地址，**触发缺页异常**！
    3. OS 异常处理程序：让我看看 **VMA**……该进程的 VMA 中包含 `0x0`。让我为你创建对应的映射。
    4. OS 异常处理程序：好了，**刷新 TLB 和 Cache，返回 `sepc`**，继续执行程序。
    5. 用户程序：（并不知道刚才发生了什么）我想要访问 `0x0` 这个地址。
    6. MMU：让我看看**页表**……哦，有映射了！对应的物理内存是……

!!! tip "Tip：修改页表后记得使用 `sfence.vma`！"

!!! success "完成条件"

    完成该 Task 后，我们已经实现了按需分页加载的基本功能。程序应该和 Lab4 完成时一样正常运行。

## Part 3：Fork 机制

### 用户态进程拷贝

自 Lab2 [进程复制与加载](lab2.md#进程复制与加载) 一节后，我们在内核态已经有了进程复制的能力（`copy_process()`）。现在只需要将其包装一下，提供给用户态程序使用即可。

`fork()` 是一个库函数，是给用户态提供的一个接口，其对应的系统调用是 `clone()`。请同学们阅读 [fork(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html) 了解 fork 的接口定义与规范。

当用户态进程调用 `fork()` 时，内核会为该进程创建一个几乎一模一样的新进程，调用 fork 的进程一般被称为父进程，而新创建的进程被称为子进程。当 fork 完成，两个进程的**内存、寄存器、PC** 等状态都是一样的；但它们具有**不同的 PID 和虚拟内存映射**，在 fork 完成后会各自独立运行，互不干扰。

接下来，我们在上一节的基础上实现 fork 机制，允许用户态程序通过系统调用创建新的进程。我们先来思考一下 fork 调用之前和之后会发生什么。

- **调用之前**：fork 总是由父进程通过**系统调用**的方式发起，我们已经在 Lab4 中实现了系统调用的相关机制。
- **调用之后**：父进程沿着正常的系统调用路径结束 `sys_clone()`，并返回子进程的 PID；子进程继承父进程的状态，被加入调度队列，就像其他进程一样等待被调度运行，并在被调度时开始执行并返回 0。

那么，`sys_clone()` 具体应该做些什么呢？关键的有如下几点：

1. **创建新进程：**调用 `copy_process()` 创建一个新的 `task_struct`，它应该继承父进程的所有状态，包括寄存器、内存空间等。
2. **伪装成普通进程**：新进程的 `sp`, `ra` 等关键寄存器应该被设置为合适的值，使得它在被调度时能够像普通进程一样运行，而不是从 `sys_clone()` 返回。
3. **返回值的设置**：父进程和子进程在 `fork()` 返回时应该有不同的值，父进程返回子进程的 PID，而子进程返回 0。

!!! info "感受 fork 的威力，但是小心！"

    下面这一段 Shell 代码非常经典，就是著名的 Fork 炸弹，它会无限制地创建子进程，最终耗尽系统资源，导致系统崩溃。请**不要尝试在你的宿主机中或其他重要环境运行它**。你能想到它是如何不断创建子进程的吗？

    ```shell title="Fork Bomb 💣"
    :(){ :|:& };:
    ```

### Task 3：实现 Fork 机制

我们已经在系统调用表 `sys_call_table` 中添加了对应的 `sys_clone()`，`syscall_handler()` 会在接收到对应的系统调用号时执行该函数。

**请你完成** `kernel/arch/riscv/kernel/syscall.c` 中的 `sys_clone()` 函数，实现 fork 机制。下面是对上述三点的具体说明：

对于**第一点**，如果你之前的实现正确，那么 `copy_process()` 已经帮助你完成了进程拷贝的全部工作。你只需要在 `sys_clone()` 中调用它即可。为了帮助同学们确认这一点，下面是一些需要注意的点：

- 你需要**深拷贝**父进程的整个 `task_struct`，包括它的内核栈 `stack`、页表 `mm.pgd`、虚拟内存区域 `mm.mmap`。
- 你需要为新进程分配一个新的 PID，并将新进程添加到全局的进程列表 `task_list` 中。

对于**第二点**，情况略显复杂，需要同学们自己思考完成。以下是一些提示：

- 你需要**正确设置新进程的 `thread->sp, thread->ra`**。这很容易理解，因为新进程的第一次运行时刻是被调度器调度时，调度器通过 `switch_to()` 切换到新进程的 `ra` 和 `sp`，然后 `ret` 到 `ra` 指向的位置运行。

    !!! note "提示"

        **父子进程状态一致**，在 `sys_clone()` 中，父进程**处于内核态**，且即将结束 `sys_clone()` **从 trap 返回路径返回**。

- 你需要**正确设置新进程的 `sepc`**。这也很容易想到，因为**父子进程状态一致**，从 Lab4 我们已经知道系统调用返回时 `sepc` 应当指向 `ecall` 的下一条指令地址。

对于**第三点**，父进程返回子进程的 PID，我们直接通过设置 `sys_clone()` 的返回值即可；而子进程返回 0，同学们需要找到合适的位置进行设置。
当我们按照顺序完成上述三点后，fork 的实现就完成了。

!!! success "完成条件"

    完成该 Task 后，内核并不能正常运行。如果你运行 Lab 5 的测试代码，你会观察到一些灵车漂移的现象，你可以尝试分析其中的原因。

### 写时复制（Copy-On-Write, COW）

写时复制的核心思想是进一步延后分配页面、拷贝内容的时机，**在 fork 时不立即复制父进程的内存页，而是让父子进程共享这些内存页，直到其中一个进程尝试修改某个内存页时，才真正地为该页创建一个副本**。它在提升 fork 性能的同时也减少了内存资源的消耗，是操作系统中一个很重要的优化。

那么，在实现上，我们怎么知道一个页需要写时复制，以及在什么时候进行拷贝呢？

- 根据上面的描述，我们知道：创建页的时机是**当某个进程尝试写入一个共享页时**，现在我们创建页都是在缺页异常处理时进行的，因此写时复制也应该在**缺页异常**中进行判断和处理。
- 而我们怎么样让进程尝试写入一个共享页时触发缺页异常呢？联想缺页异常发生的原因之一（想写入的地址存在，但是权限不够），我们可以**将进程的相关页表项设置为只读**，这样当进程尝试写入该页时，就会触发一个**Store Page Fault**，从而进入缺页异常处理流程。

<figure markdown="span">
    ![cow.drawio](lab5.assets/cow.drawio)
    <figcaption>
    写时复制机制示意图
    </figcaption>
</figure>

### Task 4：实现写时复制

**请你修改**下面两个函数，实现写时复制机制：

- `kernel/arch/riscv/kernel/vm.c` 中的 `copy_pgd()` 函数；
- `kernel/arch/riscv/kernel/trap.c` 中的 `do_page_fault()` 函数。

下面是对这两个函数实现的具体说明：

- **页表项设置**：我们在 `sys_clone()` 中通过 `copy_process()` 的 `copy_pgd()` 函数来拷贝父进程的页表。现在，在拷贝页表项之前，我们需要**将父进程的相关页表项设置为只读**（清除 `PTE_W` 标志位），**然后再进行拷贝**。这样，我们将实现：
    - 父子进程的页表项都是只读的，且它们共享同一个物理页面；
    - 父子进程都可以正常读取该页面的内容，但当它们尝试写入该页面时，就会触发缺页异常。
- **缺页异常处理**：在 `do_page_fault()` 中，我们需要增加对写时复制的处理逻辑。当捕获到一个**合法的** Store Page Fault 时，我们需要**检查该地址对应的页表项是否是只读的**。如果是，我们就知道这是一个写时复制的请求，并可以**根据页面的引用计数**来决定是否需要创建一个新的物理页面：
    - 如果引用计数大于 1，说明还有其他进程也在使用该页面，我们需要**分配一个新的物理页面**，将原页面的内容复制到新页面中，然后**更新当前进程的页表项**，将该虚拟地址映射到新页面，并设置为可写。
    - 如果引用计数等于 1，说明只有当前进程在使用该页面，我们可以直接**将该页表项设置为可写**，而不需要进行页面复制。

!!! tip "Tip：修改页表后记得使用 `sfence.vma`！"

!!! success "完成条件"

    **通过评测。**


**恭喜你完成本学期操作系统实验正片的全部内容！**完结撒花 🎉

## 扩展阅读：Fork 是一个好的实现吗？

在 [进程复制与加载](lab2.md#进程复制与加载) 一节中，我们比较了 Windows 中的 `CreateProcess()` 和 Unix 中的 `fork()` 两种进程创建方式。Unix 哲学中简洁、通用的设计不仅实现更加简单，而且将进程创建的过程**进一步解耦**：`fork()` 负责创建新进程的骨架，而 `exec()` 负责加载新进程的实体，我们可以在 `fork()` 和 `exec()` 之间插入任意的操作（如修改内存空间、文件描述符等），从而实现更加灵活的进程创建方式。

另外，`fork()` 还强调了**进程之间的关系**。在 Windows 中，`CreateProcess()` 创建的进程和原进程之间联系较弱，而 Unix 中的 `fork()` 创建的进程之间存在**天然的父子关系**，为进程管理提供了便利，例如：

- 父进程可以通过 `wait()` 等待子进程的结束，从而实现进程间的同步；
- 父子进程可以通过继承来共享一些状态（如文件描述符）。

由于在 Linux 当中，进程都是通过 `fork()` 来创建的，因此操作系统会以 fork 为中心来设计进程管理的相关机制，例如**进程树**、**僵尸进程**等（同学们可以在 Linux 机器上运行 [`pstree`](https://man7.org/linux/man-pages/man1/pstree.1.html) 命令来查看进程树）。

然而，由于在调用 `fork()` 后，父子进程之间的状态存在大量共享，也带来了一些问题和挑战。在 POSIX 标准 [fork(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html) 中，关于调用 `fork()` 时的特殊情况处理就有二十余项之多，例如：

- 在多线程程序中，`fork()` 只会复制调用该函数的那个线程，其它线程的状态（包括锁）不会被继承，可能导致死锁或不一致；
- 子进程会继承父进程所有已打开的文件描述符，因此父子对同一文件的读写会相互影响；
- `fork()` 后的子进程在调用 `exec()` 前只能使用 async-signal-safe 的函数，否则可能因为锁状态不完整而导致崩溃或死锁。


!!! info "fork 共享外部状态的🌰"

    - 如上述所说，`fork()` 后父子进程将会获得相同的**文件描述符**（如 Linux 的 PCB 中的 [`files`](https://elixir.bootlin.com/linux/v6.17/source/include/linux/sched.h#L1181)），它们将会拥有相同的文件抽象和文件偏移量（file offset）。
    - 如果父子进程同时对同一个文件进行读写操作，可能会导致数据混乱和不一致。感兴趣的同学可以写一个简单的程序来验证这一点。

    <figure markdown="span">
        ![fork_files.drawio](lab5.assets/fork_files.drawio)
        <figcaption>
        父子进程共享文件描述符示意图
        </figcaption>
    </figure>


为了尝试解决 fork 中的一系列问题，现在的 Linux 也引入了一些替代方案：

| API | 描述 |
| :-: | :-: |
| [`posix_spawn`](https://man7.org/linux/man-pages/man3/posix_spawn.3.html) | 类似于 `fork` + `exec` 的组合，先获得一份进程的拷贝，然后调用 `exec` 执行；虽然灵活度不如 `fork`，但性能要明显更优，且执行时间与原进程的内存无关。 |
| [`vfork`](https://man7.org/linux/man-pages/man2/vfork.2.html) | 接口与 `fork` 一致，它会从父进程中创建出子进程，但不会为子进程单独创建地址空间，因此两者共享地址空间；为了保证正确性，`vfork` 在调用结束后会阻塞父进程，直到子进程 `exec` 或退出。  |
| [`clone`](https://man7.org/linux/man-pages/man2/clone.2.html) | 比 `fork` 更加精细，不同与 `fork` 的完全复制，它可以指定子进程不需要复制的部分，以及进程栈的位置等，具有更强的通用性。 |

!!! info "小彩蛋🥚：fork 在路上"

    感兴趣的同学可以阅读了解关于 fork 的更多内容：

    - 对 fork 局限性的详尽分析：[A fork() in the road](https://dl.acm.org/doi/pdf/10.1145/3317550.3321435)
    - fork 在云服务场景下的应用：[Fork in the Road: Reflections and Optimizations for Cold Start Latency in Production Serverless Systems](https://www.usenix.org/system/files/osdi25-chai-xiaohu.pdf)

