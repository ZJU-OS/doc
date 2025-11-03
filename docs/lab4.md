# Lab 4：用户态

!!! danger "DDL"

    本实验尚未发布。

## 实验简介

经过 Lab2（调度）和 Lab3（虚拟内存）的铺垫，我们的内核已经具备了**多任务**和**地址空间隔离**的基本能力。现在，是时候让我们的操作系统迈出关键一步 —— **运行真正的用户态程序**。

然而实现用户态并不容易，本次实验的内容量较大，请同学们合理安排时间。我们需要解决下列问题：

- **如何运行用户态程序？**

    在 Lab1 学习链接器脚本时，我们知道普通应用程序是 ELF 格式的可执行文件。所以我们要**写一个 ELF 加载器**，解析 ELF 文件，并将其加载到用户态进程的地址空间中。

    既然要加载到用户态进程的地址空间，我们还需要**设计用户态程序的内存布局**，比如代码、数据、堆栈等段的位置和权限。

- **用户态程序如何调用内核服务？**

    在理论课上我们学习了**系统调用**的概念。用户态程序不能直接进行和敏感资源（如硬件设备、内存管理单元等）相关的操作，而是通过系统调用请求内核代为执行。因此，我们需要**进一步增强 `trap_handler()`，实现系统调用机制**，处理来自用户态的 ecall 异常。

## Part 0：准备工作

### 更新代码

现在你位于 `lab3` 分支。你需要创建 `lab4` 分支，合并上游的代码：

```shell
git checkout -b lab4
git fetch upstream
git merge upstream/lab4
```

下面的合并说明供同学们解决合并冲突时参考：

TODO

## Part 1：内核对用户态的支持

作为本次实验的第一步，我们先不管用户态程序是什么样子，而是从内核的视角出发，设计和实现内核对用户态程序的支持。

- 首先我们要认识用户态的内存布局，修改 `task_struct` 以支持用户态进程的页表、分离的用户栈与内核栈。
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

在 Lab3 中我们将 `0x0000000000000000` ~ `0x0000003fffffffff` 划给用户空间，可以自由设计用户空间的布局。本实验采用非常简单的设计：

- 把整个程序（代码、数据等）放到这段空间的开头
- 把用户态栈放到这段空间的末尾，即设置 `sp` 初始值为 `0x0000004000000000`

`struct task_struct` 新增了 `pgd` 成员来保存进程的页表。

### 用户栈与内核栈

用户态 Trap 进入内核态时，`sp` 指向用户空间。内核的代码能直接使用这个栈吗？

答案是否定的，有以下原因：

- **隔离：**PTE 中的 `U` 位起到了隔离用户态和内核态数据的作用：

    > The U bit indicates whether the page is accessible to user mode. U-mode software may only access the page when U=1. If the SUM bit in the sstatus register is set, supervisor mode software may also access pages with U=1. However, supervisor code normally operates with the SUM bit clear, in which case, supervisor code will fault on accesses to user-mode pages. Irrespective of SUM, the supervisor may not execute code on pages with U=1.

    `sstatus.SUM` 默认为 0，因此内核态代码无法访问用户态数据，包括用户栈。

- **安全：**把 `sstatus.SUM` 置 1，内核代码就能使用用户栈了，但这并不安全。使用栈的前后，程序只会移动栈指针，而**不会清理栈上的数据**。如果内核代码使用用户栈，则有可能泄露内核敏感数据到用户态，被恶意程序利用。

!!! note "要点：隔离用户态和内核态数据"

    很多漏洞都是由于内核态数据泄露引起，例如：

    - 硬件漏洞：[幽灵・熔毁・预兆 | 孙耀珠的博客](https://blog.yzsun.me/spectre-meltdown-foreshadow/)
    - 软件漏洞：[How STACKLEAK improves Linux kernel security | Alexander Popov](https://a13xp0p0v.github.io/2018/11/04/stackleak.html)

    因此，隔离用户态和内核态数据是操作系统设计中的重要原则。

`struct task_struct` 新增了两个成员：`kernel_sp` 和 `user_sp` 用于保存用户栈和内核栈。

### 从用户态到内核态

本节讨论 Trap 处理程序如何完成用户栈与内核栈的切换。RISC-V 提供了一个专门的寄存器 `sscratch`，见 [12.1.6. Supervisor Scratch (sscratch) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#_supervisor_scratch_sscratch_register)，标准已经说明了它的用法：

> Typically, sscratch is used to hold a pointer to the hart-local supervisor context while the hart is executing user code. At the beginning of a trap handler, software normally uses a CSRRW instruction to swap sscratch with an integer register to obtain an initial working register.

- **CPU 运行 User 程序时**：`sscratch` 保存 supervisor context，指的就是存放在内核中的进程控制块（Process Control Block, PCB），即 `struct task_struct`。
- **Trap 发生时**：使用 CSRRW 指令交换 `sscratch` 和 `tp`，Trap 处理程序就能通过 `tp` 访问 `task_struct`，进而存取其中的 `kernel_sp` 和 `user_sp` 了。
- **切换栈**：把 `sp`（现在是用户栈）保存到 `user_sp`，再把 `kernel_sp` 加载到 `sp`，切换到内核栈。**然后在内核栈上保存 Trap 上下文，因为中断处理程序要用。**
- **Trap 处理结束后**：再把 `tp` 保存回 `sscratch`。

上面讨论了从用户态 Trap 的情况，但 Trap 处理程序仍需要处理来自内核态的 Trap。为了区分两种情况，我们约定：

- **Trap 上下文保存完成后**：应当把 `sscratch` 置 0。
- **Trap 处理程序开始时**：检查 `sscratch` 是否为 0。
    - 如果为 0，说明是内核态 Trap，继续使用当前的 `sp`。
    - 如果不为 0，说明是用户态 Trap，需要切换到内核栈。

### Task1：改进 Trap Handler

按照上面的说明，修改中断处理程序。

## Part 2：加载和运行用户态程序

本节我们先了解用户态程序是怎么编译、嵌入内核的，同时认识 ELF 格式和用户态内存布局。接下来我们会实现一个 ELF 加载器，把用户态程序加载到正确的内存位置并设置好权限。完成后就能运行用户态程序了。

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

    上面的修改将所有 `uapp` 段的内容放在内核的 `_sramdisk` 和 `_eramdisk` 符号之间，而上一步就是把 `uapp.elf` 这个文件放进了 `uapp` 段，现在这个就被嵌入到了内核镜像 `vmlinux` 中。

### ELF 规范阅读

[](https://refspecs.linuxfoundation.org/elf/elf.pdf)

请你阅读其中 Book I: Executable and Linking Format (ELF) 的：

- Chapter 1. Object Files 的 Introduction、File Format、ELF Header 部分
- Chapter 2. Program Loading and Dynamic Linking 的 Introduction、Program Header 部分

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

    Program Header（程序头）

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

    请同学们使用 `readelf -W -l uapp.elf` 命令查看 `uapp.elf` 的 Program Header 信息。请你描述一下根据这个信息，加载器需要将哪些内容加载到内存的哪些位置，并设置什么权限？

    其中有些段的 `MemSiz` 为 0，意思就是这些段不需要加载到内存中。

### Task 1：实现 ELF 加载器

你的任务是补全 `kernel/arch/riscv/kernel/binfmt_elf.c` 中的 `load_elf_binary()` 函数，其整体逻辑如下：

- `file` 指向 ELF 文件起始位置，也就是 ELF Header。
- 根据 ELF Header 中的 `e_phoff`、`e_phentsize` 和 `e_phnum` 字段，找到 Program Header Table。
- 遍历 Program Header Table 中的每个条目：

    - 如果 `p_type` 不是 `PT_LOAD`，跳过该条目。
    - 否则，根据 `p_offset`、`p_vaddr`、`p_filesz` 和 `p_memsz` 字段，将文件中的内容加载到用户态进程的地址空间中。

### 从内核态到用户态

只有一种方法能够进入用户态：当 `sstatus.SPP` 为 0 时，执行 `sret` 指令。因此我们需要构造一个合适的 Trap 上下文，然后从 `call trap_handler` 之后的部分返回即可。我们在 `entry.S` 中为该部分插入一个新的标签 `ret_from_trap`。

像 `struct thread_struct` 一样，我们把 Trap 上下文定义为一个结构体 `struct pt_regs`

```c title="kernel/arch/riscv/include/proc.h"
struct pt_regs {
    uint64_t ra;
    uint64_t sp;
    //...
};
```

### Task 2



### 进程页表与用户态内存布局

## Part 2：用户态与内核态的交互



### 改进 Trap Handler

经历过前几个实验，同学们应该对进入 Trap 时保存上下文很熟悉了。**系统调用正是利用 ecall 作为一种异常，通过 Trap 上下文与内核传递参数和返回值的**。现在，

### 系统调用

### 内核栈与 sscratch

### Task1：实现系统调用

你的任务是：

1. 改进 `_traps()` 的实现：上下文保存到内核栈栈顶，布局与 `struct pt_regs` 定义一致。


## Part 3：实现系统调用

## Part 1：运行用户态程序



### Task 2：加载并运行用户态程序



