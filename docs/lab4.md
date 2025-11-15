# Lab 4：用户态

!!! danger "DDL"

    - 代码、报告：2025-12-02 23:59
    - 验收：2025-12-09 实验课

## 实验简介

经过 Lab2（调度）和 Lab3（虚拟内存）的铺垫，我们的内核已经具备了**多任务**和**地址空间隔离**的基本能力。现在，是时候让我们的操作系统迈出关键一步 —— **运行真正的用户态程序**。

然而实现用户态并不容易，本次实验的内容量较大，请同学们合理安排时间。我们需要解决下列问题：

- **如何运行用户态程序？**

    在 Lab1 学习链接器脚本时，我们知道普通应用程序是 ELF 格式的可执行文件。所以我们要**写一个 ELF 加载器**，解析 ELF 文件，并将其加载到用户态进程的地址空间中。

    既然要加载到用户态进程的地址空间，我们还需要**设计用户态程序的内存布局**，比如代码、数据、堆栈等段的位置和权限。

- **用户态程序如何调用内核服务？**

    在理论课上我们学习了**系统调用**的概念。用户态程序不能直接执行和敏感资源（如硬件设备、内存管理单元等）相关的操作，而是**通过系统调用请求内核代为执行**。因此，我们需要**进一步完善 Trap 处理机制，以实现系统调用**，即处理来自用户态的 ecall 异常。

温馨提示：

- 本次实验的栈、内存地址等内容略显复杂，建议同学们仔细阅读实验文档中的图示，也可以自己画图理一理思路。

## Part 0：准备工作

### 更新代码

现在你位于 `lab3` 分支。你需要创建 `lab4` 分支，合并上游的代码：

```shell
git checkout -b lab4
git fetch upstream
git merge upstream/lab4
```

下面的合并说明供同学们解决合并冲突时参考：

- **新增实验相关文件**

    本次 Lab4 引入 **用户态**、**ELF 加载器**、**系统调用** 三大模块，因此新增文件较多。

    1. 新增：ELF 加载器相关

        | 文件                                      | 描述                                             |
        | --------------------------------------- | ---------------------------------------------- |
        | `kernel/arch/riscv/include/binfmts.h`   | ELF 加载接口声明                                     |
        | `kernel/arch/riscv/kernel/binfmt_elf.c` | 主要的 ELF 加载器实现（load_elf_binary()）               |
        | `kernel/include/elf.h`                  | 完整 ELF 规范结构体定义（所有 ELF header / program header） |


    2. 新增：系统调用相关

        | 文件                                    | 描述                                    |
        | ------------------------------------- | ------------------------------------- |
        | `kernel/arch/riscv/include/syscall.h` | syscall 号、trap 编号定义                   |
        | `kernel/include/syscall_table.h`      | 系统调用分发表（用户态号 → 内核处理函数）                |
        | `kernel/arch/riscv/kernel/syscall.c`  | 系统调用总入口（do_syscall / syscall_handler） |

    3. 新增：用户态程序构建系统

        这些文件全部来自 `kernel/user/`：

        | 文件                       | 描述                                        |
        | ------------------------ | ----------------------------------------- |
        | `kernel/user/Makefile`   | 编译用户态程序 uapp.elf                          |
        | `kernel/user/src/*`      | 用户态 main.c、libc（简化版 printf）、syscalls stub |
        | `kernel/user/src/head.S` | 用户态入口 `_start`                            |
        | `kernel/user/uapp.S`     | incbin 用户态 ELF 到内核镜像 `.uapp` 段            |
        | `kernel/user/uapp.lds`   | 用户态 ELF 链接脚本                              |

        **合并注意：**

        - `kernel/Makefile` 修改：需要把 `user/uapp.o` 链接进 vmlinux.
        - `kernel/arch/riscv/kernel/vmlinux.lds` 修改：新增 ramdisk 区（uapp），详见 Part2 用户态程序嵌入内核。

- **对现有内核的修改点**

    1. **页表 / 虚拟内存模块**扩展：详见 Part1

        | 文件                               | 变更                                           |
        | -------------------------------- | -------------------------------------------- |
        | `kernel/arch/riscv/include/mm.h` | 新增物理页引用计数函数 `get_page`/`put_page` 定义     |
        | `kernel/arch/riscv/include/vm.h` | 新增页表相关宏与用户态地址空间定义，映射接口扩展、PGD 拷贝函数声明  |
        | `kernel/arch/riscv/kernel/vm.c`  | 新增 `copy_pgd`；扩展 `create_mapping`；新增用户态映射逻辑 |
        | `kernel/arch/riscv/kernel/mm.c`  | 新增物理页引用计数函数 `get_page`/`put_page` 实现           |

    2. **proc（进程结构）**：详见 Part1 进程页表

        | 文件                                 | 变更                                                       |
        | ---------------------------------- | -------------------------------------------------------- |
        | `kernel/arch/riscv/include/proc.h` | 大量新增：用户态栈、`kernel_sp`、`user_sp`、`pt_regs`、`task_pt_regs` 宏      |
        | `kernel/arch/riscv/kernel/proc.c`  | `task_init` / `copy_process` / `release_task` 都新增 pgd / 栈指针维护 |

    3. **用户态、内核态切换与 trap 处理逻辑**：详见 Part1 用户栈与内核栈

        | 文件                                 | 变更                                  |
        | ---------------------------------- | ----------------------------------- |
        | `kernel/arch/riscv/kernel/head.S` | 内核运行时处于 S 态，初始化 `sscratch` 寄存器为 0 |
        | `kernel/arch/riscv/kernel/entry.S` | Trap 开头新增用户态/内核态判断，切换栈；结尾新增 SPP 逻辑 |
        | `kernel/arch/riscv/kernel/trap.c`  | `trap_handler` 修改为只接收 *regs         |

    4. **实验测试**：
        在 `main.c` 中按如下方式创建用户态和内核态进程：
    ```diff title="kernel/arch/riscv/kernel/main.c"
    --- a/kernel/arch/riscv/kernel/main.c
    +++ b/kernel/arch/riscv/kernel/main.c
	+  user_mode_thread(_sramdisk); // Lab4 Test
	   kernel_thread(kthreadd, NULL); // Lab2 Test3
	+  user_mode_thread(_sramdisk); // Lab4 Test
	   kthread_create(test_sched, NULL); // Lab2 Test4
    ```

## Part 1：内核对用户态的支持

作为本次实验的第一步，我们先不管用户态程序是什么样子，而是从内核的视角出发，设计和实现内核对用户态程序的支持。

- 首先我们要认识用户态的内存布局，修改 `task_struct` 以支持各进程页表独立，分离出用户栈与内核栈。
- 然后我们要改进 `trap_handler()`，以支持从用户态陷入内核态。

### 用户态内存布局

假设我们有两个用户态程序：微信和 QQ。它们的内存布局如下图所示：

<figure markdown="span">
    ![layout.drawio](lab4.assets/layout.drawio)
    <figcaption>
    用户态内存布局
    </figcaption>
</figure>

从图中可以看出虚拟内存起到的作用：

- **简化：**每个用户程序都认为自己独占整个内存空间，因此程序可以在固定的虚拟地址上加载与运行，而无需关心实际的物理内存位置或碎片情况。比如从微信和 QQ 的视角来看，它们都认为自己运行在从 `0x0` 开始的地址空间中，但实际上它们被映射到了不同的物理内存。
    - **对应用程序开发者：**不需要关心物理内存的布局、是否碎片化、是否与其他程序冲突。
    - **对编译器/链接器：**可以统一假设所有程序都从相同的虚拟基地址开始编译（例如 .text 段从 0x0 起），不需要为不同进程做地址重定位。
- **隔离：**为每个用户态进程提供一个独立的地址空间，防止它们互相干扰。例如图中微信和 QQ 进程的页表各自独立，微信的页表中不会有 QQ 相关内存的映射，微信没有任何办法访问 QQ 的数据，反之亦然。

在 Lab3 中我们已经知道 `0x0000000000000000~0x0000003fffffffff` 是用户空间的地址范围，我们可以自由设计这部分空间的布局。本实验采用非常简单的设计：

- 把整个程序（代码、数据等）放到这段空间的开头（细节见 Part 2 ELF 加载）。
- 把用户态栈放到这段空间的末尾，即设置 `sp` 初始值为 `0x0000004000000000`。

### 物理页引用计数

现在每个进程都有自己的页表，我们遇到了 C/C++ 语言中最麻烦的问题：共享指针。如下图所示：

<figure markdown="span">
    ![ref_count.drawio](lab4.assets/ref_count.drawio)
    <figcaption>
    不同页表共享物理页
    </figcaption>
</figure>

例如当 PID0 退出时，我们要释放它的页表，那页表指向的页要不要释放呢？这些页可能被其他进程映射，如果直接释放这些物理页，就会导致 PID1 访问非法内存，引发异常。

为此 Buddy System 提供了物理页的引用计数机制。API 如下：

- `alloc_page/alloc_pages()`：分配物理页时，引用计数初始化为 1。
- `get_page()`：引用计数加 1。
- `put_page()`：引用计数减 1，当引用计数变为 0 时，释放该物理页。

最后分析页表中存在哪些映射，如何做引用计数管理：

- **内核空间的映射：**在 Lab3 中我们将物理内存 `0x80000000~0x88000000` 直接映射到内核空间 `0xffffffd600000000~0xffffffd608000000`。所有用户态、内核态进程的页表中都包含这部分映射，这些页也永远不需要释放、不需要引用计数管理。

    进一步地，我们可以在三级页表中让这部分页表共享，即指向同一份子页表数据结构，**节省内存开销**。如下图所示：

    <figure markdown="span">
        ![shared_kernel_pgtable.drawio](lab4.assets/shared_kernel_pgtable.drawio)
        <figcaption>
        共享内核空间页表
        </figcaption>
    </figure>

- **用户空间的映射：**上一节描述的 `0x0000000000000000~0x0000003fffffffff` 的区域，将在 Part2 展开讲解。

    简单来说，我们会先 `alloc_page()` 分配物理页，然后再通过页表映射到用户空间。在释放页表时，对用户空间的页面调用 `put_page()`。

    对于本实验，所有用户态进程的用户态内存都是独立的一份拷贝，并不会出现上面描述的指向同一个页面的情况，引用计数看起来好像没什么用。但它为下一个实验的 Fork 和内存页面的写时复制（Copy-On-Write, COW）打下了基础。

### Task1：实现进程页表

`struct task_struct` 新增了 `pgd` 成员来保存进程的页表。

- 实现页表的工具函数，见 `kernel/arch/riscv/kernel/vm.c`：

    - **请你补全** `copy_pgd()`。该函数接收一个三级页表，对其进行**深拷贝**，返回新页表。它的实现和 Lab3 中的 `create_mapping()` 类似。

        特例：对于内核空间的页表项，按上面的描述直接共享即可。

    - **我们提供**了 `free_pgd()` 用于释放三级页表，并且用 Buddy System 的 `put_page()` 释放物理页。同样地，内核空间的页表项指向的物理页直接跳过。

- 相应地，进程的整个生命周期也需要维护页表，见 `kernel/arch/riscv/kernel/proc.c`：

    - **请你修改** `task_init()`，设置第一个进程的页表，即将 `pgd` 指向内核页表 `swapper_pg_dir`。
    - **请你修改** `release_task()`，添加进程页表的释放。
    - **请你修改** `copy_process()`，添加进程页表的拷贝。

- **我们修改了** `kernel/arch/riscv/kernel/entry.S` 的 `__switch_to`，在切换进程时切换页表。

!!! success "完成条件"

    仅完成该 Task 时，代码并不完善，无法运行。请继续完成下一个 Task。

### 用户栈与内核栈

用户态 Trap 进入内核态时，`sp` 指向用户空间。内核的代码能直接使用这个栈吗？

答案是否定的，有以下原因：

- **隔离：**PTE 中的 `U` 位起到了隔离用户态和内核态数据的作用：

    > The U bit indicates whether the page is accessible to user mode. U-mode software may only access the page when U=1. If the SUM bit in the sstatus register is set, supervisor mode software may also access pages with U=1. However, supervisor code normally operates with the SUM bit clear, in which case, supervisor code will fault on accesses to user-mode pages. Irrespective of SUM, the supervisor may not execute code on pages with U=1.

    `sstatus.SUM` 默认为 0，因此内核态代码无法访问用户态数据，包括用户栈。

- **安全：**把 `sstatus.SUM` 置 1，内核代码就能使用用户栈了，但这并不安全。使用栈的前后，程序只会移动栈指针，而**不会清理栈上的数据**。如果内核代码使用用户栈，则有可能泄露内核敏感数据到用户态，被恶意程序利用。

!!! note "要点：隔离用户态和内核态数据"

    很多漏洞都是由于内核态数据泄露引起。感兴趣的同学可以阅读以下资料：

    - 硬件漏洞：[幽灵・熔毁・预兆 | 孙耀珠的博客](https://blog.yzsun.me/spectre-meltdown-foreshadow/)，通过 CPU 乱序执行的漏洞获取内核数据。
    - 软件漏洞：[How STACKLEAK improves Linux kernel security | Alexander Popov](https://a13xp0p0v.github.io/2018/11/04/stackleak.html)，通过内核栈数据泄露的漏洞获取内核数据。

    因此，隔离用户态和内核态数据是操作系统设计中的重要原则。

`struct task_struct` 新增了两个成员：`kernel_sp` 和 `user_sp` 用于保存用户态 `sp` 和内核态 `sp`。`stack` 始终指向内核栈的低地址端。

<figure markdown="span">
    ![stacks.drawio](lab4.assets/stacks.drawio)
    <figcaption>
    用户栈与内核栈
    </figcaption>
</figure>

Part2 将介绍用户栈的用法，下一节分析 Linux 是怎么设计 Trap 处理和内核栈布局的。

### 从用户态到内核态

本节讨论 Trap 处理程序如何实现用户栈与内核栈的切换。RISC-V 提供了一个专门的寄存器 `sscratch`，见 [12.1.6. Supervisor Scratch (sscratch) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#_supervisor_scratch_sscratch_register)，标准已经说明了它的用法：

> Typically, sscratch is used to hold a pointer to the hart-local supervisor context while the hart is executing user code. At the beginning of a trap handler, software normally uses a CSRRW instruction to swap sscratch with an integer register to obtain an initial working register.

- **CPU 运行用户态程序时**：
    - `sscratch` 保存 supervisor context，指的就是存放在内核中的进程控制块（Process Control Block, PCB，对应的数据结构是 `struct task_struct`），也就是内核态的 `tp` 寄存器（`proc.h` 将其绑定到 C 语言变量 `current`）。
    - `tp` 应该是 `0`，不能让 User 程序知道自己的 PCB 在哪里。
- **Trap 发生时**：使用 CSRRW 指令交换 `sscratch` 和 `tp`，Trap 处理程序就能通过 `tp` 访问 `task_struct`，进而存取其中的 `kernel_sp` 和 `user_sp` 了。
- **切换栈**：
    - 把 `sp`（现在是用户栈）保存到 `user_sp`
    - 再把 `kernel_sp` 加载到 `sp`，切换到内核栈。
- **在内核栈上保存 Trap 上下文**
- **Trap 处理结束后**：再把 `tp` 保存回 `sscratch`。

通过上面的设计，很自然地能够在 `_traps()` 开头区分来自用户态的 Trap 和来自内核态的 Trap：

- 来自用户态的 Trap：`sscratch` 中保存的是 PCB 地址，不为 0。此时需要：

    - 切换用户栈和内核栈
    - 交换 `sscratch` 和 `tp`

- 来自内核态的 Trap：`sscratch` 中保存的是 0。此时：

    - 不需要操作栈，直接使用当前的 `sp`
    - 也不需要交换 `sscratch` 和 `tp`

最后，我们考虑嵌套 Trap 的情况。嵌套 Trap 一定是来自内核态的 Trap（先前的 Trap 要么从用户态进入内核态，要么已经在内核态了）。这时会**直接使用当前的 `sp`**，那么 Trap 上下文就依次保存到内核栈上了，不难想象会是下图的情况：

<figure markdown="span">
    ![nested_trap.drawio](lab4.assets/nested_trap.drawio)
    <figcaption>
    嵌套 Trap 时的栈布局
    </figcaption>
</figure>

因此 `_traps()` 的结尾也需要区分用户态 Trap 的返回和内核态 Trap 的返回。思路也很简单：

- `call trap_handler` 返回后，`sp` 仍然指向最新的 Trap 上下文。可以获取其中的 `sstatus.SPP` 位判断要返回到用户态还是内核态。
- 如果返回到用户态：
    - 需要切换用户栈和内核栈
    - 需要交换 `sscratch` 和 `tp`
- 如果返回到内核态：
    - 不需要切换栈
    - 不需要交换 `sscratch` 和 `tp`

!!! info "FAQ：为什么会有嵌套 Trap"

    有同学会问，一进 Trap `sstatus.SIE` 就被移到 `sstatus.SPIE` 了，内核态中断被禁止了，怎么还会有 Trap 呢？

    - `SIE` 只管 Interrupt，你的内核代码写得不好的话，可能会触发 **Exception**，比如访问非法内存地址、除零等。这就造成了嵌套 Trap。
    - 在现实的硬件中，有一种中断叫 NMIs（Non-Maskable Interrupts，非屏蔽中断），它们是不可屏蔽的中断。实验中不需要考虑，感兴趣的同学可以进一步阅读特权级手册的 [8. "Smrnmi" Extension for Resumable Non-Maskable Interrupts, Version 1.0](https://zju-os.github.io/doc/spec/riscv-privileged.html#rnmi) 章节。

### Task2：实现用户栈与内核栈

按照上文和下面的说明完成任务。

- `kernel/arch/riscv/include/proc.h`：
    - **我们提供** `struct pt_regs`，用于统一 Trap 上下文的保存格式。
    - **我们提供** `task_pt_regs(tsk)` 宏，用于获取最新的 Trap 上下文指针。
    - **我们提供** `user_mode(struct pt_regs *regs)` 函数用于判断 Trap 来源于用户态还是内核态。
    - 在进程的整个生命周期中，`user_sp` 和 `kernel_sp` 都需要正确维护。**请你修改** `task_init()`、`release_task()` 和 `copy_process()` 等函数，确保它们被正确设置、拷贝和释放。
- `kernel/arch/riscv/kernel/entry.S`：
    - **请检查并修改** `_traps()`
        - 确保保存和恢复的内容与 `struct pt_regs` 定义一致。

            !!! tip "建议在 `entry.S` 中使用 `proc.h` 中提供的一些偏移量宏，这些宏有编译器静态检查，不容易出错。"

        - 把 Trap 上下文传递给 `trap_handler()`
        - **我们提供** 在 `_traps()` 返回路径上，根据 `sstatus.SPP` 判断返回到用户态还是内核态，并执行相应的栈切换和 `sscratch` 与 `tp` 交换的逻辑。
        - **请你修改** `_traps()` 的开头，判断 Trap 来源于用户态还是内核态，并执行相应的栈切换和 `sscratch` 与 `tp` 交换逻辑。

- `kernel/arch/riscv/kernel/trap.c`：
    - **请你修改** `trap_handler()`：
        - 仅接受一个参数 `struct pt_regs *regs`，表示本次 Trap 的上下文。
        - 相关的参数如 `scause`、`sepc` 等均从 `regs` 中获取。

!!! success "完成条件"

    该 Task 没有单独的评测。内核应当和 Lab3 完成时一样正常运行。

!!! example "动手做：Trap 来源的判断"

    为什么在 Trap 开头**不得不**使用 `sscratch` 判断 Trap 来源，不能像返回时那样通过 `sstatus.SPP` 判断呢？为什么非得用 `csrrw` 指令呢？请你给出解释。

## Part 2：加载和运行用户态程序

本节我们先了解用户态程序是怎么编译、嵌入内核的，同时认识 ELF 格式和用户态内存布局。接下来实现一个 ELF 加载器，把用户态程序加载到正确的内存位置并设置好权限。完成后就能运行用户态程序了。

### 用户态程序嵌入内核

因为我们还没有实现文件系统，所以无法从磁盘加载用户态程序。用户态程序被编译为 ELF 格式的二进制文件，嵌入到内核镜像中。

请同学们打开 `kernel/user/Makefile`，了解用户态程序的编译步骤：

- 首先将 `../lib` 和 `src` 下的所有源文件（`.c`，对应变量 `C_SRC`）编译为目标文件（`.o`，对应变量 `OBJ`）。
- 然后根据链接脚本 `uapp.lds` 将这些目标文件链接为 ELF 格式的可执行文件 `uapp.elf`。

    ```shell
    $(LD) -T uapp.lds -o $@ $(OBJ)
    ```

    ```text title="kernel/user/uapp.lds"
    ENTRY(_start)
    SECTIONS
    {
        . = 0;

        .text : {
            *(.text.init)
            *(.text .text.*)
        }

        .rodata : {
            *(.rodata .rodata*)
        }

        .data : {
            *(.data .data.*)
            *(.sbss .sbss.*)
            *(.bss .bss.*)
        }
    }
    ```

    - `ENTRY(_start)` 指定程序入口点为 `_start` 符号，它定义在 `src/head.S` 中。

- 接着编译 `uapp.S`，这个汇编代码段使用 `incbin` 指令将 `uapp.elf` 嵌入到 `uapp` 段中，得到目标文件 `uapp.o`。

    ```shell
    $(GCC) $(CFLAGS) -c uapp.S
    ```

    ```assembly title="kernel/user/uapp.S"
    .section .uapp
    .incbin "uapp.elf"
    ```

- 合并时修改了 `vmlinux.lds` 和 `vmlinux` 的链接命令：

    ```diff
    --- a/kernel/arch/riscv/kernel/vmlinux.lds
    +++ b/kernel/arch/riscv/kernel/vmlinux.lds
            _edata = .;

    +        . = ALIGN(0x1000);
    +        _sramdisk = .;
    +        *(.uapp .uapp*)
    +        _eramdisk = .;
    +
            . = ALIGN(0x1000);
    --- a/kernel/Makefile
    +++ b/kernel/Makefile
                $(LD) -T arch/riscv/kernel/vmlinux.lds \
                    arch/riscv/kernel/*.o \
                    lib/*.o \
    +               user/uapp.o \
                    -o vmlinux
    ```

    上面的修改将所有 `uapp` 段的内容放在内核的 `_sramdisk` 和 `_eramdisk` 符号之间，而上一步就是把 `uapp.elf` 这个文件放进了 `uapp` 段。最终这个文件就被嵌入到了内核镜像中。

### ELF 规范阅读

这是 ELF 规范：[Tool Interface Standard (TIS) Executable and Linking Format (ELF) Specification](https://refspecs.linuxfoundation.org/elf/elf.pdf)。本节涉及其中 Book I: Executable and Linking Format (ELF) 的：

- Chapter 1. Object Files 的 Introduction、File Format、ELF Header 部分
- Chapter 2. Program Loading and Dynamic Linking 的 Introduction、Program Header 部分

建议先看看下面的要点，搞不清楚的地方再回头看规范原文。

??? note "要点：Object Files"

    **File Format（文件格式）**

    📘 核心概念

    - ELF（Executable and Linking Format）是一种**可重定位、可执行、可共享库的统一文件格式**。
    - 同一文件在**链接阶段（Linking View）**与**执行阶段（Execution View）**有不同的视图。

    📂 文件整体结构

    一个 ELF 文件的基本组成如下：

    ```
    ELF Header
    Program Header Table (optional)
    Section 1
    Section 2
    ...
    Section n
    Section Header Table (optional)
    ```

    **两种视图：**

    | 视图类型           | 用途                   | 主要结构        |
    | -------------- | -------------------- | ----------- |
    | Linking View   | 用于链接阶段（编译器/链接器）      | Sections（节） |
    | Execution View | 用于加载与执行阶段（OS/Loader） | Segments（段） |

    > ✅ 重点理解：
    >
    > * **Sections**：面向链接器（包含代码、数据、符号表、重定位信息等）。
    > * **Segments**：面向加载器（将文件映射到内存中形成进程映像）。
    > * **Program Header Table**：描述如何将文件内容映射到内存（仅可执行文件和共享库需要）。
    > * **Section Header Table**：描述每个 Section 的信息（仅链接器需要）。

    ⚠️ 注意事项

    * 只有 **ELF Header** 位置固定（文件起始处）。
    * **Program Header Table** 和 **Section Header Table** 的位置可变。
    * 并非所有 ELF 文件都包含两种 Header 表：

    * 可执行文件：必须有 Program Header Table。
    * 可重定位文件：必须有 Section Header Table。

    **ELF Header（ELF 文件头）**

    📘 作用

    ELF Header 位于文件开头，起到“地图”作用，指明文件中其他结构（段、节等）的位置与大小。

    📐 结构定义（简化）

    ```c
    typedef struct {
    unsigned char e_ident[16];  // ELF 标识信息
    Elf32_Half    e_type;       // 文件类型
    Elf32_Half    e_machine;    // 目标架构
    Elf32_Word    e_version;    // 文件格式版本
    Elf32_Addr    e_entry;      // 程序入口地址
    Elf32_Off     e_phoff;      // Program Header 表偏移
    Elf32_Off     e_shoff;      // Section Header 表偏移
    Elf32_Word    e_flags;      // 架构相关标志
    Elf32_Half    e_ehsize;     // ELF Header 大小
    Elf32_Half    e_phentsize;  // Program Header 表项大小
    Elf32_Half    e_phnum;      // Program Header 表项数量
    Elf32_Half    e_shentsize;  // Section Header 表项大小
    Elf32_Half    e_shnum;      // Section Header 表项数量
    Elf32_Half    e_shstrndx;   // 字符串表所在节索引
    } Elf32_Ehdr;
    ```

    🧭 各字段要点

    | 字段名                       | 含义                       | 示例值                                 |
    | ------------------------- | ------------------------ | ----------------------------------- |
    | **e_ident**               | 前 16 字节，识别 ELF 文件、字节序、ABI 等 | `0x7f 'E' 'L' 'F'`                  |
    | **e_type**                | 文件类型                     | `ET_REL=1`, `ET_EXEC=2`, `ET_DYN=3` |
    | **e_machine**             | 目标机器架构                   | `EM_386=3`, `EM_MIPS=8`             |
    | **e_version**             | ELF 版本号                  | `EV_CURRENT=1`                      |
    | **e_entry**               | 程序入口虚拟地址                 | 加载器跳转起点                             |
    | **e_phoff**               | Program Header 表在文件中的偏移  | 用于加载阶段                              |
    | **e_shoff**               | Section Header 表在文件中的偏移  | 用于链接阶段                              |
    | **e_flags**               | CPU 特定标志位                | 架构定义                                |
    | **e_ehsize**              | ELF Header 总字节数          | 通常为 52 (ELF32)                      |
    | **e_phentsize / e_phnum** | 每个程序头大小 & 数量             | 加载时遍历用                              |
    | **e_shentsize / e_shnum** | 每个节头大小 & 数量              | 链接器使用                               |
    | **e_shstrndx**            | 节名字符串表的索引                | 用于节名解析                              |

    > ✅ 理解要点：
    >
    > - **ELF Header 决定文件类型和结构布局**；
    > - **e_entry、e_phoff、e_shoff** 是最关键的三个指针；
    > - **可执行文件**关注 `Program Header`；
    > - **目标文件**关注 `Section Header`。

读完 Chapter 1，应该理解 Linking View 和 Execution View 的区别：

- Linking View 面向编译器和链接器，关注 Sections（节），用于符号解析和重定位。
- Execution View 面向操作系统和加载器，关注 Segments（段），用于将文件内容映射到内存中运行。

接下来将关注 Execution View 的相关内容，理解怎么加载、执行文件。

??? note "要点：Program Loading and Dynamic Linking"

    **Program Header（程序头）**

    📘 作用

    Program Header Table 描述了**文件中各段（segment）**在内存中的映射方式，是操作系统加载 ELF 的核心依据。

    仅对：

    * **Executable (ET_EXEC)**
    * **Shared Object (ET_DYN)**
    文件有效。

    📐 结构定义（简化）

    ```c
    typedef struct {
    Elf32_Word p_type;   // 段类型
    Elf32_Off  p_offset; // 段在文件中的偏移
    Elf32_Addr p_vaddr;  // 段的虚拟地址
    Elf32_Addr p_paddr;  // 段的物理地址（通常未用）
    Elf32_Word p_filesz; // 文件中占用的字节数
    Elf32_Word p_memsz;  // 内存中占用的字节数
    Elf32_Word p_flags;  // 段权限标志（R/W/X）
    Elf32_Word p_align;  // 对齐要求
    } Elf32_Phdr;
    ```

    🧩 各字段要点

    | 字段           | 含义          | 加载阶段作用           |
    | ------------ | ----------- | ---------------- |
    | **p_type**   | 段类型（见下表）    | 决定如何解释此项         |
    | **p_offset** | 段在文件中的起始偏移  | 从文件读入内存时使用       |
    | **p_vaddr**  | 段映射到内存的虚拟地址 | 建立进程映像时使用        |
    | **p_paddr**  | 物理地址（少用）    | 某些系统上用于硬件加载      |
    | **p_filesz** | 文件中该段长度     | 文件读取长度           |
    | **p_memsz**  | 内存中该段长度     | 若 > 文件长度，剩余部分清零  |
    | **p_flags**  | 访问权限        | R/W/X            |
    | **p_align**  | 文件与内存的对齐约束  | 通常为页大小（如 0x1000） |

    > ✅ 关键理解：
    >
    > * `p_filesz ≤ p_memsz`。多出的部分（如 `.bss` 段）在内存中初始化为 0。
    > * `p_vaddr % p_align == p_offset % p_align` 必须成立。
    > * 加载器根据这些信息将文件段映射到内存。

    🔖 段类型（p_type）

    | 名称                      | 值                     | 含义                              |
    | ----------------------- | --------------------- | ------------------------------- |
    | **PT_NULL**             | 0                     | 无效项（忽略）                         |
    | **PT_LOAD**             | 1                     | 可加载段（程序代码、数据等）                  |
    | **PT_DYNAMIC**          | 2                     | 动态链接信息段                         |
    | **PT_INTERP**           | 3                     | 指定解释器路径（如 `/lib/ld-linux.so.2`） |
    | **PT_NOTE**             | 4                     | 辅助信息（调试、core dump）              |
    | **PT_SHLIB**            | 5                     | 保留，未定义                          |
    | **PT_PHDR**             | 6                     | 程序头表本身                          |
    | **PT_LOPROC–PT_HIPROC** | 0x70000000–0x7fffffff | 处理器专用                           |

    > ✅ 通常 ELF 可执行文件中至少包含两个 `PT_LOAD` 段（代码段 + 数据段）。

读完 Chapter 2，应该理解接下来实现的 ELF 加载器就是要根据 Program Header 中的信息将文件内容加载到内存中。Program Header 描述了每个段（Segment）的文件偏移、需要加载到的内存地址、大小和权限等信息，加载器根据这些信息完成加载过程。

??? note "要点：ELF 文件总结"

    📘 总结思维导图式理解

    ```
    ELF File
    │
    ├── ELF Header —— 描述文件总体结构（文件类型、入口地址、各表偏移）
    │
    ├── Program Header Table —— [运行时用]
    │      └── 若为可执行或共享文件，则描述 Segment 映射关系
    │
    ├── Sections —— [编译/链接时用]
    │      └── 包含代码(.text)、数据(.data)、符号表(.symtab) 等
    │
    └── Section Header Table —— 供链接器解析
    ```

    🧭 建议同学理解重点

    | 主题                 | 应理解的核心问题                     |
    | ------------------ | ---------------------------- |
    | **ELF Header**     | 文件是什么类型？入口在哪？段表和节表在哪？        |
    | **Program Header** | 如何将文件映射到内存？每个段怎么装载？          |
    | **File Format**    | Section 与 Segment 的区别与联系是什么？ |

!!! example "动手做：查看 `uapp.elf` 的 Program Header"

    稍后我们实现的 ELF 加载器需要根据 Program Header 将用户态程序 `uapp.elf` 加载到内存中，所以先了解一下它的内容。

    请同学们使用 `readelf -W -l uapp.elf` 命令查看 `uapp.elf` 的 Program Header 信息。请你描述一下根据这个信息，加载器需要将哪些内容加载到内存的哪些位置，并设置什么权限？

    其中有些段的 `MemSiz` 为 0，意思就是这些段不需要加载到内存中。

    （提示：这类“查看 xxx”的动手做，没有什么需要思考的，不需要写一大堆）

### Task 3：实现 ELF 加载器

你的任务是补全 `kernel/arch/riscv/kernel/binfmt_elf.c` 中的 `load_elf_binary()` 函数。函数接口定义在相应头文件中，其整体逻辑如下：

- `file` 指向 ELF 文件起始位置，也就是 ELF Header。
- 根据 ELF Header 中信息，找到 Program Header Table。
- 遍历 Program Header Table 中的每个条目，也就是每个 Segment。如果该 Segment 需要加载，就根据 Segment 的信息完成内存映射和数据拷贝。具体来说，针对每个 Segment：

    - 根据该段的**大小**，从 Buddy System 中**分配**一些内存页。
    - 把该段数据**拷贝**到分配的内存页中。
    - 在该进程的页表中建立相应的**映射**，设置好**权限**。这里我们提供了 `flags_phdr_to_pte()` 函数用于把 ELF 段的权限转换为 PTE 权限位。

    这些步骤所需的所有信息都在 Program Header Table 中，上一节已经介绍过，这里就不具体点出了。

!!! success "完成条件"

    本 Task 没有单独的评测，和下一个 Task 一起评测。

### 从内核态到用户态

只有一种方法能够进入用户态：当 `sstatus.SPP` 为 0 时，执行 `sret` 指令。恰好 `entry.S` 中 `call trap_handler` 之后的部分就是执行 `sret` 指令返回，我们想**复用 Trap 返回的代码路径**来进入用户态。

为此我们在 `entry.S` 中为该部分插入一个新的标签 `ret_from_trap()`。然后需要**构造一个初始的 Trap 上下文**，即为 `struct pt_regs` 填充合适的值，然后跳转到 `ret_from_trap()` 即可。在上文“从用户态到内核态”中我们对 `ret_from_trap()` 分类讨论了返回到用户态和返回到内核态的情况，这里就按照返回到用户态的情况来构造 Trap 上下文。

上下文的构造不难想，无非就是设置好 PC、栈等内容，这里直接给到同学们，同学们需要理解为什么这么设置：

- `sepc`：应当填充用户态程序的入口地址，这一信息也在 ELF Header 中。
- `sstatus`：
    - `SPIE` 应当置 1，以便从用户态返回时重新启用 S 态中断。
    - `SPP` 应当置 0，以便从 S 态返回 U 态。
    - `UXL` 应当设置为 2，表示用户态程序运行在 64 位模式。
- `sp`：应当填充用户栈的初始值。

这些工作在 `kernel/arch/riscv/kernel/binfmt_elf.c` 的 `start_thread()` 函数中完成。`load_elf_binary()` 调用它来为新进程构造初始 Trap 上下文。

最后我们来梳理一遍用户态进程从创建到运行的整体流程：

- **创建进程**：调用 `copy_process()` 创建一个新进程，并加入到进程队列。
- **加载程序**：调用 `load_elf_binary()` 加载用户态程序到新进程的内存空间，并调用 `start_thread()` 构造初始 Trap 上下文。
- **切换进程**：调度器切换到该进程时，执行 `__switch_to` 切换页表，并跳转到 `ret_from_trap()`。
- **进入用户态**：`ret_from_trap()` 设置好寄存器后执行 `sret`，进入用户态运行用户态程序。

### Task 4：运行用户态程序

现在，我们有了 `load_elf_binary()` 用于加载用户态程序，还有上一个 Lab 实现的 `copy_process()` 用于创建进程。**请你补全** `kernel/arch/riscv/kernel/proc.c` 中的 `user_mode_thread()` 函数。它的逻辑应该是：

- 调用 `copy_process()` 从当前进程拷贝一个新进程
- 调用 `load_elf_binary()` 加载用户态程序到新进程的内存空间
    - 调用 `start_thread()` 为新进程构造一个 Trap 上下文，用于 `ret_from_trap` 返回到用户态
- 设置新进程的 `struct thread_struct` 中的几个寄存器。
    - `ra`：设置为 `ret_from_trap` 即可
    - `sp`：指向初始 Trap 上下文所在的位置，因为 `ret_from_trap` 会从这里恢复寄存器

!!! success "完成条件"

    通过评测。

    因为还未实现系统调用，用户态程序没法输出，所以没什么可以观测的现象。当然，你可以开启 DEBUG 级别的 LOG 输出，看看调度器有没有成功切换到用户态进程。

## Part 3：系统调用

### 系统调用

同学们在理论课上学习了系统调用，**它是用户态程序请求内核提供服务的接口**。我们将实现 Linux 风格的系统调用机制，相关资料如下：

- Linux 的系统调用规范见 [syscall(2) - Linux manual page](https://man7.org/linux/man-pages/man2/syscall.2.html)，可以在其中看到 RISC-V 架构的系统调用约定：

    ```text
        The first table lists the instruction used to transition to kernel
    mode (which might not be the fastest or best way to transition to
    the kernel, so you might have to refer to vdso(7)), the register
    used to indicate the system call number, the register(s) used to
    return the system call result, and the register used to signal an
    error.
    Arch/ABI    Instruction           System  Ret  Ret  Error    Notes
                                        call #  val  val2
    ───────────────────────────────────────────────────────────────────
    riscv       ecall                 a7      a0   a1   -
    ```

    ```text
        The second table shows the registers used to pass the system call
    arguments.
    Arch/ABI      arg1  arg2  arg3  arg4  arg5  arg6  arg7  Notes
    ──────────────────────────────────────────────────────────────
    riscv         a0    a1    a2    a3    a4    a5    -
    ```

    同学们应该想起 RISC-V SBI 规范也有类似的规定。

- Linux 提供的所有系统调用见 [syscalls(2) - Linux manual page](https://man7.org/linux/man-pages/man2/syscalls.2.html)。

    本次实验，我们需要实现 `getpid()`、`write()` 两个系统调用，请同学们简单看看这两个系统调用的文档描述：

    - [getpid(2) - Linux manual page](https://man7.org/linux/man-pages/man2/getpid.2.html)
    - [write(2) - Linux manual page](https://man7.org/linux/man-pages/man2/write.2.html)

请同学们阅读代码并理解：

- `kernel/user/src/syscalls.c`：这是系统调用在用户态侧的接口实现，你应当想起 Lab1 中 SBI 调用的实现，基本类似。
- `kernel/arch/riscv/kernel/syscall.c`：这是系统调用在内核态侧的处理实现。
- `kernel/arch/riscv/kernel/trap.c`：这里的 `do_ecall_u()` 调用系统调用处理函数。

!!! question "考点：系统调用的实现"

    系统调用的用户态、内核态接口以前要同学们自己写，但是没什么意思。现在直接给出了，验收时会对这些代码提问，你要能讲清楚它是怎么工作的。

### Task 5：实现 `write` 系统调用

在实验中，`write()` 系统调用使用 `sbi_debug_console_write()` 来输出字符。但 SBI 运行在 M 模式，不经过 S 模式的地址翻译，所以它只接受物理地址作为参数。而 `write()` 系统调用中的第二个参数（`char *buf`）从用户态传过来时，是用户态的虚拟地址。

**请你补全** `kernel/arch/riscv/kernel/syscall.c` 的 `UVA2PA()` 函数，把用户态的虚拟地址转换为物理地址。

!!! success "完成条件"

    通过评测。

    用户态程序见 `kernel/user/src/main.c`，请你阅读它的代码，理解它在干什么。你应该能在控制台看到用户态程序打印出五颜六色的文本。
