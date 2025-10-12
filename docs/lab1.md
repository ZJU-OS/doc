# Lab 1：内核启动与时钟中断

!!! danger "DDL"

    - 代码、报告：2025-10-13 23:59
    - 验收：2025-10-21 实验课

## 实验简介

在 Lab 1 中我们将开始构建自己的内核。梳理一下目前的思路：

- 在实验导读和 Lab0 中，我们已经了解了启动过程：

    - OpenSBI 跳转到内核第一条指令处。
    - `arch/riscv/head.S` 是内核最开始执行的**汇编代码**。查看它，你会发现它最终执行了 `tail start_kernel` 跳转到位于 `arch/riscv/init/main.c` 的 **C 语言代码**。

    那么有如下问题：

    - 从 OpenSBI 跳到内核时，系统的状态（寄存器、内存等）是什么样的？这需要你动手调试，用 Lab0 学习的 QEMU Monitor 和 GDB 自行查看。
    - 从汇编代码跳到 C 函数，调用者、被调用者各需要做什么工作？回忆《计算机组成》课程，我们学习过汇编与 C 语言的关系。这需要你学习**RISC-V 调用约定**，然后补全 `arch/riscv/boot.S`。
    - 如何控制内核代码、数据放置的位置？这需要你学习**链接器脚本**，然后理解 `arch/riscv/kernel/vmlinux.lds`。
    - 没有了标准库，打印字符等基本功能如何实现？这需要你学习**SBI 规范**，然后补全 `arch/riscv/kernel/sbi.c`。

完成这些，你就能够启动自己的内核，并打印 `Hello, ZJU-OS!` 了。

- 我们的下一个目标是为 Lab2 **线程调度**做准备：

    我们在理论课上学习了操作系统是**事件驱动**的，意味着系统的执行流是由各种事件触发的，例如 I/O 完成、进程创建、用户输入等。时钟中断是其中最基础、最重要的事件源之一。**如果没有时钟中断，操作系统将无法感知时间的流逝，无法在合适的时机进行进程切换、调度、超时处理或实现延时等待。**通过周期性触发的时钟中断，**内核可以被动地转变为主动：不再依赖进程自愿交出 CPU，而是能在中断到来时打断当前执行流**，检查是否有更高优先级的任务或等待超时的事件需要处理，从而保证多任务系统的公平性和响应性。

    那么有如下问题：

    - RISC-V 是如何设计中断与异常的？怎么开启时钟中断？这需要你学习**RISC-V 特权级、中断与异常、CSR 寄存器**。
    - 中断、异常可能在任何时候发生。应该如何实现 Trap 处理程序，保存和恢复现场，从而保证内核和用户程序的正确执行？这需要你学习**RISC-V 中断异常委派、中断处理**，然后补全 `arch/riscv/kernel/entry.S`。

完成这些，你就能做 Lab2 了。

## 实验要求

见 [首页#要求和评分标准](index.md#要求和评分标准)

## Part 0：环境配置

### Linux 内核源码和课程仓库结构

详见 [Linux source code layout — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html#linux-source-code-layout)。你可以打开左侧的 Linux 源码链接，边读边看。

课程仓库目录的组织结构与 Linux 源码保持一致。

- `arch`：特定架构的代码，实现启动、异常处理等核心功能，都受到指令集架构的约束
- `arch/riscv/kernel/head.S`：内核最开始执行的代码
- `lib`：内核无法使用标准库，`printk` 等辅助函数实现在这里

其中 [`arch/riscv`](https://github.com/torvalds/linux/tree/master/arch/riscv) 目录最为重要，课程实验大部分的代码工作都将发生在这里。

### 更新代码

现在你位于 `lab0` 分支。你需要创建 `lab1` 分支，合并上游的代码：

```shell
git checkout -b lab1
git fetch upstream
git merge upstream/lab1
```

合并说明：

- 新增 `kernel` 目录下的实验代码
- 完善了环境支持（compose 文件、devcontainer 配置等）

### 更新镜像

助教对镜像进行了一些重要更新。最新的镜像 Digest 可以在 [Container registry · ZJU CS Lab / Tool  · GitLab](https://git.zju.edu.cn/zju-cs-lab/tool/container_registry/17) 查看。

请运行 `docker images --digests` 命令，确认你拉取的镜像 Digest 与 ZJU Git Registry 上的 `latest` **或**对应架构的 Digest 一致。

如果不正确，则运行 `make update` 更新镜像。

## Part 1：启动工作

### RISC-V 汇编与调用约定

同学们上次熟悉 RISC-V 汇编与 C 语言的关系，应该是在《计算机组成》课程中。上《计算机体系结构》的时候，同学们只烧二进制 Bit 文件，应该已经把相关内容忘得差不多了。本节我们来回顾一下。

<figure markdown="span">
    ![co.webp](lab1.assets/co.webp)
    <figcaption>
    《计算机组成》课程 PPT
    </figcaption>
</figure>

[Compiler Explorer](https://godbolt.org/) 可以方便地探索不同编译器、不同版本、不同优化选项下的汇编代码。我们用它来看看，C 与 RISC-V 汇编是如何对应的：

!!! tip "下面是一个可以交互的子页面，请点击[这个链接](<https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:6,positionColumn:2,positionLineNumber:6,selectionStartColumn:2,selectionStartLineNumber:6,startColumn:2,startLineNumber:6),source:'int+func(int+a)%7Breturn+a%3B%7D%0A%0Aint+main()+%7B%0A++int+r+%3D+func(10)%3B%0A++return+r+%2B+1%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-gcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'',overrides:!(),selection:(endColumn:27,endLineNumber:7,positionColumn:27,positionLineNumber:7,selectionStartColumn:27,selectionStartLineNumber:7,startColumn:27,startLineNumber:7),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4>)放大查看"

<iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:6,positionColumn:2,positionLineNumber:6,selectionStartColumn:2,selectionStartLineNumber:6,startColumn:2,startLineNumber:6),source:'int+func(int+a)%7Breturn+a%3B%7D%0A%0Aint+main()+%7B%0A++int+r+%3D+func(10)%3B%0A++return+r+%2B+1%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-gcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'',overrides:!(),selection:(endColumn:27,endLineNumber:7,positionColumn:27,positionLineNumber:7,selectionStartColumn:27,selectionStartLineNumber:7,startColumn:27,startLineNumber:7),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

请回忆你在《计算机组成》课程中学习的相关内容，回答下面的问题。如果有想不起来的地方，请阅读 [RISC-V ELF psABI Document](spec/riscv-abi.pdf) 的以下章节：

- 1.1. Integer Register Convention
- 2.1. Integer Calling Convention

!!! question "考点"

    - 每个函数的开头都操作了 `sp`，这是在干什么？
    - 尝试修改 C 语言代码，你会发现 `sp` 的差值总是 16 的倍数，这是为什么？
    - 调用函数前后做了什么？

??? note "要点：RISC-V 调用约定"

    - **参数寄存器与返回值**

        - RISC-V 提供 **8 个参数寄存器 a0–a7**：
            - **a0、a1**：用于返回值
            - **a2–a7**：用于传递参数，超出部分放在栈上
        - 超过寄存器数量的参数会通过**栈**传递

    - **参数传递规则**

        **标量（Scalars）**

        - **≤ XLEN 位**：放在一个寄存器或栈上传递
            - 小于 XLEN 的整数会符号扩展（sign-extend）到 XLEN 位
            - 小于 XLEN 的浮点数会零扩展到 XLEN 位，高位不定义
        - **2 × XLEN 位**：用连续两个寄存器传递，高位在编号大的寄存器中
        - **> 2 × XLEN 位**：通过引用（地址）传递

        **结构体和联合体（Aggregates）**

        - **≤ XLEN 位**：单寄存器传递
        - **≤ 2 × XLEN 位**：双寄存器传递，若寄存器不足，剩余部分放栈上
        - **> 2 × XLEN 位**：按引用传递（传地址）

    - **栈对齐和栈传参**

        - 栈指针（SP）必须在**函数入口时 128-bit 对齐**
        - 参数在栈上的布局按 **≥ type alignment 和 XLEN 对齐**，但不会超过栈本身的对齐要求
        - 传栈的第一个参数在 SP + 0 位置，之后依次递增

    - **变参函数（Variadic Functions）**

        - 变参与命名参数传递方式相同
        - 特殊规则：
            - **2 × XLEN 位对齐**的参数必须使用**偶数号寄存器对**
            - 一旦有参数被传到栈上，之后的参数都必须放栈上

    - **返回值**

        - 返回值的传递方式 = 该类型第一个命名参数的传递方式
        - 如果是按引用传递的，调用者分配空间，传地址，callee 写结果

    - **结构体对齐与位域规则**

        - 小于一个 XLEN 的结构体直接放在寄存器里，按内存布局排列字段
        - 位域按小端序（little-endian）打包：
            - 如果跨越对齐边界，跳到下一个对齐边界
            - 多余位按 padding 处理

    - **寄存器保存规则**

        - **s0–s11** 必须在函数调用间保持（callee-save）
        - **临时寄存器 t0–t6、a0–a7** 调用不保证保存（caller-save）

此外，除了基本的 I 指令集，我们在汇编时还经常使用一些**伪指令（pseudo-instruction）**。它们并不是 RISC-V ISA 中真实的指令，而是汇编语言的语法糖。伪指令会被汇编器翻译成一个或多个真实的指令。

请阅读 [RISC-V 汇编手册](spec/riscv-asm.pdf) 的以下章节，学习常用的伪指令：

- Chapter 29. A listing of standard RISC-V pseudoinstructions

!!! question "考点"

    - 下列伪指令分别对应什么真实指令？

        ```text
        la nop li mv j ret call tail
        ```

    - `call` 伪指令做了什么工作？它与 `tail` 指令有什么区别？

### 汇编器指令

你已经熟悉 RISC-V 汇编指令。但是为了写汇编代码，还要了解一些汇编器指令（assembler directive），这就像 C 语言中的预处理指令一样。

阅读 `arch/riscv/head.S`，尝试理解它的内容。如果有不懂的地方，可以查阅 `as` 汇编器手册的下列章节：

- [5 Symbols](https://sourceware.org/binutils/docs/as.html#Symbols-2)：阅读 5.1-5.4 即可
- [7.3 .align [abs-expr[, abs-expr[, abs-expr]]]](https://sourceware.org/binutils/docs/as.html#Align)
- [7.43 .global symbol, .globl symbol](https://sourceware.org/binutils/docs/as.html#Global)
- [7.87 .section name - ELF Version](https://sourceware.org/binutils/docs/as.html#ELF-Version)
- [7.65 .macro](https://sourceware.org/binutils/docs/as.html#g_t_002emacro)
- [7.94 .space size [,fill]](https://sourceware.org/binutils/docs/as.html#g_t_002espace-size-_005b_002cfill_005d)

??? note "要点：汇编器指令"

    - **Symbols 基础概念**

        - 程序员用符号命名；链接器用符号做链接；调试器用符号调试。
        - 符号用于命名程序中的地址或数据位置。
        - 可以代表位置计数器、常量值或程序中定义的标签。
        - 支持多种类型：全局符号、本地符号、局部**标签（Label）**等。

    - **Labels**

        - **标签的定义**
            - `symbol:` 定义标签，表示当前位置计数器的值。
            - 相同符号多次定义时，以第一次为准，后续会收到警告。

        - **本地标签（Local Labels）**
            - 形式：`N:` 定义标签，`Nb` 引用最近的前一个，`Nf` 引用最近的下一个。
            - 编译器会将本地标签转换成唯一的符号名，避免冲突。

        - **美元标签（Dollar Labels）**
            - 形式如 `55$:`，作用范围更小，遇到非本地标签就失效。

    - **符号赋值**
        - 形式：`symbol = expression` 等价于 `.set`，`symbol == expression` 等价于 `.eqv`。
        - 可以把符号赋值为任意表达式结果。

    - **符号命名规则**
        - 以字母或 `._` 开头，支持 `$`，大小写敏感。
        - 不以数字开头，除非是本地标签。
        - 支持多字节字符，但可能受编译器或系统限制。

    - **`.` 符号**
        - 表示当前位置计数器。
        - `melvin: .long .` → 定义 `melvin` 为其自身地址。
        - 赋值 `.=.+4` 等价于 `.org` 指令，跳过 4 字节。

    - **常见汇编器指令**

        - **`.align n`**：将位置计数器对齐到 n 字节边界。
        - **`.global symbol` / `.globl symbol`**：导出符号给链接器 `ld` 使用。
        - **`.section name`**：切换到指定节，支持 ELF 特定参数。
        - **`.macro` / `.endm`**：宏定义，可生成重复代码。
        - **`.space size[,fill]`**：分配 size 字节空间，填充值为 fill（默认为 0）。

### 链接器脚本与内核内存布局

在 C 语言编程课上，我们了解过编译器的基本流程：预处理、编译、汇编、链接，`gcc` 默认帮你完成了所有流程。假设程序由两个源文件 `main.c` 和 `func.c` 组成，拆分后的流程示例如下：

```shell
riscv64-linux-gnu-cpp main.c -o main.i
riscv64-linux-gnu-gcc -S main.i -o main.S
riscv64-linux-gnu-as main.S -o main.o
riscv64-linux-gnu-cpp func.c -o func.i
riscv64-linux-gnu-gcc -S func.i -o func.S
riscv64-linux-gnu-as func.S -o func.o
riscv64-linux-gnu-ld main.o func.o -o main
```

链接过程合并多个目标文件，并**重定位**它们的数据，**链接**各类符号（函数名、变量名等）。链接器 `ld` 通过**链接脚本**来控制最终的生成的文件，默认使用内置的脚本生成 **ELF 格式的可执行文件**，你可以使用下面的命令查看这个脚本：

```shell
riscv64-linux-gnu-ld --verbose
```

请阅读 [3 Linker Scripts - LD](https://sourceware.org/binutils/docs/ld.html#Scripts) 的以下章节，学习基本的链接器脚本语法：

- 3.1 [Basic Linker Script Concepts](https://sourceware.org/binutils/docs/ld.html#Basic-Linker-Script-Concepts)
- 3.3 [Simple Linker Script Example](https://sourceware.org/binutils/docs/ld.html#Simple-Linker-Script-Example)
- 3.5 [Assigning Values to Symbols](https://sourceware.org/binutils/docs/ld.html#Assigning-Values-to-Symbols)
- 3.6.1 [Output Section Description](https://sourceware.org/binutils/docs/ld.html#Output-Section-Description-1)

??? note "要点：链接器脚本"

    下面的要点中，**源码中的符号引用**非常重要，在后续写 C 代码时会用到。

    - **Linker Script 基础概念**

        1. **目标文件（Object File）**

            - GCC 或其他编译器生成的 `.o` 文件。
            - 每个目标文件包含若干 **输入节（Input Section）**，例如 `.text`, `.data`, `.bss`。
            - 每节有名字、大小和数据内容。

        2. **输出文件（Output File）**

            - 链接器将多个输入文件合并生成单一输出文件。
            - 输出文件也有若干 **输出节（Output Section）**。
            - 输出节是由输入节组成的，它们可以被加载到内存中或只分配空间。

        3. **节的属性**

            - **Loadable（可加载）**：节内容需要加载到内存，例如代码和初始化数据。
            - **Allocatable（可分配）**：节不包含数据，但需要在内存中保留空间，例如 `.bss`。
            - **非加载非分配**：节通常用于调试信息，例如 `.debug_*`。

        4. **符号（Symbol）**

            - 每个目标文件中定义或引用的函数、全局变量都是符号。
            - 链接器会解析符号的地址，未定义符号需要在链接时找到定义。

        5. **VMA 和 LMA**

            - **VMA（Virtual Memory Address）**：程序运行时的虚拟地址。
            - **LMA（Load Memory Address）**：程序加载到内存时的地址。
            - 例如，ROM 加载到 RAM 时 LMA ≠ VMA。

        6. **位置计数器 `.`**

            - 链接器使用 `.` 来表示当前内存位置。
            - 每定义一个输出节，`.` 会自动增加节大小。

    - **Linker Script 文件格式**

        - LD Script 是文本文件。
        - 支持 **注释**：

            ```ld
            /* 这是注释 */
            ```

        - 语法结构：

            - 关键字 + 参数，例如 `OUTPUT_ARCH("riscv")`
            - 符号赋值，例如 `BASE_ADDR = 0x80000000`
            - 使用 `SECTIONS` 定义输出节布局。

    - **SECTIONS 指令**

        `SECTIONS` 是最核心的命令，用于描述输出文件的内存布局。示例：

        ```ld
        SECTIONS
        {
            . = 0x10000;        /* 设置位置计数器 */
            .text : { *(.text) } /* 将所有输入文件的 .text 节放入输出节 .text */
            . = 0x8000000;
            .data : { *(.data) }
            .bss : { *(.bss) }
        }
        ```

        解释：

        - `. = 0x10000`：将位置计数器设置为 0x10000。
        - `.text : { *(.text) }`：输出节 `.text` 包含所有输入节 `.text`。
        - `.data` 和 `.bss` 同理。
        - 链接器会保证输出节按需对齐，如果地址不符合要求，会在节之间插入间隙。

    - **符号赋值**

        - **符号赋值的基本概念**

            - 在链接脚本（linker script）中可以给符号（symbol）赋值，定义它并把它加入符号表，作用域为全局。
            - 赋值语句形式：

                ```ld
                symbol = expression;      // 定义符号值
                symbol += expression;     // 在原值基础上加
                symbol -= expression;     // 在原值基础上减
                ...
                symbol |= expression;     // 按位或
                ```

            - 第一次赋值会定义符号，后续的 +=, -= 等操作要求符号已存在。
            - `.` 表示位置计数器（location counter），只能在 `SECTIONS` 中使用。

        - **三种赋值位置**

            符号赋值可以出现在：

            1. 单独作为命令
            2. `SECTIONS` 命令内部
            3. 输出段（output section）描述内部

            示例：

            ```ld
            floating_point = 0;
            SECTIONS
            {
              .text :
                {
                  *(.text)
                  _etext = .;
                }
              _bdata = (. + 3) & ~ 3;
              .data : { *(.data) }
            }
            ```

            - `floating_point = 0` → 定义为 0
            - `_etext = .` → 定义为 `.text` 末尾地址
            - `_bdata = (. + 3) & ~3` → 定义为 4 字节对齐的 `.text` 末尾地址


        - **PROVIDE**

            - `PROVIDE(symbol = expression)`
            - 仅在**符号被引用且未被定义**时才定义，避免与用户代码冲突。
            - 常用于传统符号如 `etext`，用户可覆盖定义。

            示例：

            ```ld
            SECTIONS {
              .text : {
                PROVIDE(etext = .);
              }
            }
            ```

        - **源码中的符号引用**

            - 链接脚本符号 ≠ C 语言变量，**没有实际内存分配**，只是一个地址。
            - 在汇编代码中直接作为地址使用：

                ```asm
                la a0, symbol_name  # 加载符号地址到 a0
                ```

            - 在 C 代码中使用时：

                ```c
                extern char start_of_ROM, end_of_ROM, start_of_FLASH;
                memcpy(&start_of_FLASH, &start_of_ROM, &end_of_ROM - &start_of_ROM);
                ```

            - 也可以直接声明为数组：

                ```c
                extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];
                memcpy(start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
                ```

            - **必须取地址，不能直接当作数值使用**。

现在，我们来简单对比一下 Linux 内核和 `ld` 内置（普通应用程序）的链接脚本的区别：

<div class="grid cards" markdown>

```text title="linux/arch/riscv/kernel/vmlinux.lds"
ENTRY(_start)
SECTIONS
{
 . = ((((-1))) - 0x80000000 + 1);
 _start = .;
 . = ALIGN((1 << 12));
 . = ALIGN((1 << 21));
 . = ALIGN(4);
```

```text title="riscv64-linux-gnu-ld --verbose"
ENTRY(_start)
SECTIONS
{
  . = SEGMENT_START("text-segment", 0x10000) + SIZEOF_HEADERS;
  .dynsym         : { *(.dynsym) }
  .rela.dyn       :
    {
      *(.rela.init)
      *(.rela.text .rela.text.* .rela.gnu.linkonce.t.*)
```

</div>

- **起始位置**：应用程序为 `0x10000 + SIZEOF_HEADERS`，而内核是 `0xffffffff80000000`。学习虚拟内存后你会理解这些地址受[操作系统内存布局](https://docs.kernel.org/arch/riscv/vm-layout.html)影响。
- **对齐：**内核对齐要求十分严格，通常使用页面对齐（4KB 或 2MB 等）、缓存行对齐（64B）等，需要考虑内存分页、NUMA 架构 Cache 一致性等问题（回忆你在《计算机体系结构》课程中学习的相关知识）。
- **节的选择**：内核不使用[动态链接](https://lwn.net/Articles/961117/)、[重定位](https://dram.page/p/relative-relocs-explained/)，这些技术是为了应用程序设计的。因此没有 `.dynsym`、`.rela.dyn` 等节。
- **`_start` 符号**：两者都定义了 `_start` 作为程序入口，但内核的 `_start` 定义在脚本中，而应用程序的 `_start` 由[标准库定义](https://www.monperrus.net/martin/compiling-where-is-_start)（标准库需要做一些初始化工作），并不在应用程序代码中。

    你可以尝试编译一个简单的程序而不带上标准库，`ld` 就会找不到 `_start` 符号：

    ```console
    $ riscv64-linux-gnu-gcc -nostdlib test.c
    /usr/lib/gcc-cross/riscv64-linux-gnu/15/../../../../riscv64-linux-gnu/bin/ld: warning: cannot find entry symbol _start; defaulting to 000000000000030a
    ```

!!! note "要点：内核链接脚本"

    `vmlinux` 与普通应用程序的异同：

    - 相同点：都是 **ELF 格式**的可执行文件。
    - 不同点：与普通的应用程序相比，操作系统的链接需要精细控制内存布局，包括数据段的放置和对齐等。

    链接器的输入和输出都是二进制文件，受到 **ABI 规范** 的约束。ELF 格式就是 ABI 规范的一部分。

链接器脚本的具体语法并不重要，能大致看懂就行。接下来，请阅读并理解实验代码仓中的 `arch/riscv/kernel/vmlinux.lds`，回答下面的问题：

!!! question "考点"

    - 这个链接脚本描述的就是整个内核的内存布局。它从哪里开始，有多大？
    - 其中的各个段（`.text`、`.rodata`、`.data`、`.bss`）分别存放什么数据？
    - `_skernel` 这些符号是什么？你要如何在汇编和 C 代码中使用它们？

!!! info "ABI 规范及其历史"

    RISC-V ABI 手册中没有 `.text`、`.rodata` 等的定义，因为它是历史传承下来的。RISC-V ABI 手册的参考文献指向了 [System V Application Binary Interface - DRAFT](https://www.sco.com/developers/gabi/latest/contents.html)，你可以在这里找到相关定义。

    操作系统有比较久远的历史沉淀，因此 ABI 层的规范不像 SBI 那样唯一。感兴趣的同学可以阅读 [The UNIX System -- History and Timeline -- UNIX History](https://unix.org/what_is_unix/history_timeline.html)。

`vmlinux.lds` 用于生成 `vmlinux`，但 `Image` 又是怎么生成的呢？请你：

- 阅读 `Makefile`，找到相关命令
- 阅读 [objcopy (GNU Binary Utilities)](https://sourceware.org/binutils/docs/binutils/objcopy.html) 的简介部分，理解它生成 binary 时做了什么工作

!!! note "要点：固件加载内存镜像"

    `objcopy -O binary` 从 ELF 格式的可执行文件中直接把程序要运行的那部分指令和数据，原封不动拷贝出来，得到一个内存布局和运行时完全一致的二进制文件，称为内存镜像（memory dump）。

    固件（如 OpenSBI）非常简单，在启动时只负责把这段二进制文件搬到一个固定的内存地址，然后直接跳过去执行，并不支持解析 ELF 等格式。

    你将在 Lab4 中完成 ELF 文件的解析和加载，以运行用户态程序。

此外，构建过程还会生成几个文件用于辅助调试，遇到问题时好好利用它们：

- `vmlinux.asm`：反汇编结果，包含源代码注释
- `System.map`：符号表，包含所有符号的地址

!!! example "动手做"

    运行 `make` 构建内核。

    用 VSCode 打开 `kernel/vmlinux`（已默认绑定到 ElfPreview 插件），查看它的内容。

    这些信息也可以通过 `readelf` 工具查看，请你自行尝试。

### OpenSBI 调试

本节让我们动手探究 OpenSBI 跳转到内核时，系统的状态（寄存器、内存等）是什么样的。

注意到 OpenSBI 有如下输出。这是 OpenSBI 在通过物理内存保护（Physical Memory Protection, PMP）机制设置物理内存的权限：

```text
Domain0 Region00            : 0x0000000000100000-0x0000000000100fff M: (I,R,W) S/U: (R,W)
Domain0 Region01            : 0x0000000010000000-0x0000000010000fff M: (I,R,W) S/U: (R,W)
Domain0 Region02            : 0x0000000002000000-0x000000000200ffff M: (I,R,W) S/U: ()
Domain0 Region03            : 0x0000000080040000-0x000000008005ffff M: (R,W) S/U: ()
Domain0 Region04            : 0x0000000080000000-0x000000008003ffff M: (R,X) S/U: ()
Domain0 Region05            : 0x000000000c400000-0x000000000c5fffff M: (I,R,W) S/U: (R,W)
Domain0 Region06            : 0x000000000c000000-0x000000000c3fffff M: (I,R,W) S/U: (R,W)
Domain0 Region07            : 0x0000000000000000-0xffffffffffffffff M: () S/U: (R,W,X)
```

!!! example "动手做"

    按 Lab0 中的步骤启动 QEMU 和 GDB 调试。

    1.  GDB 在 OpenSBI 跳转到的地址（Next Addr）处设置断点，查看此时：

        - `sp` 寄存器的值是多少？
        - 这属于哪个区域，该区域各个特权级的权限是什么？

    2. 用 VSCode 打开 `kernel/arch/riscv/boot/Image`（已默认绑定到 Hex Editor 插件），然后拉到最底下，观察 Hex Editor 左侧显示的文件偏移地址，它的大小是多少？接下来，请你对照 `vmlinux.lds` 和 `System.map`，查看起始和末尾符号（`_skernel` 和 `_ebss`）对应的地址之差是多少？（在这里，我们暂时不考虑 `_ekernel`，因此存在内存对齐的因素）

        请你思考：为什么 `Image` 文件的大小会**小于**链接脚本中定义的内核大小？在上面，我们已经知道 `Image` 文件的内存布局和运行时是一致的，理论上它应当符合链接脚本和符号表中的定义。（提示：你需要理解链接脚本中各个段存放的数据类型）

        事实上，由于 `.data` 段存放有初始值的变量，这些初始值本身必须被保存在镜像中，而 `.bss` 段存放未初始化的变量，我们只需要在镜像中记录这个段的大小，而不需要存储其实际内容。内核启动时，加载器会为它分配空间并**直接清零**内存。因此，`.bss` 段的大小不会反映在 `Image` 文件中。

        接下来，请你寻找 `vmlinux.lds` 中栈空间在哪里被定义，并指出具体的代码。我们为什么选择在 `.bss` 段开辟栈空间，而不是 `.data` 段呢？

!!! info "更多资料"

    - Physical Memory Protection (PMP) 机制：RISC-V 特权级手册 [3.7. Physical Memory Protection](spec/riscv-privileged.html#pmp)
    - OpenSBI 相关源码相关源码位于 [`lib/sbi/sbi_domain.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/sbi/sbi_domain.c) 中的 `sbi_domain_init()`
    - 感兴趣的同学可以使用 QEMU Monitor 打印内存树（`info mtree`），对比了解 OpenSBI 为什么要设置这些 PMP 区域

### Task 1：为 `start_kernel()` 准备运行环境

现在你已经知道：

- OpenSBI 跳转到内核时，`sp` 指向 OpenSBI 保护的区域，无法使用
- 我们在内核中开辟了一段栈空间
- 汇编代码中可以引用链接脚本定义的符号

你的任务是修改 `arch/riscv/head.S`，使得能够成功进入 `start_kernel()` 函数，并进入到 `printk()` 函数。

!!! success "完成条件"

    - GDB 断点打在 `printk()` 处，能{==成功==}进入 `printk()` 函数。这里{==成功==}的条件是：到达 `printk()` 断点时，`scause` 寄存器的值为 0，表明没有异常。

        ```text
        (gdb) b printk
        Breakpoint 1 at ...
        (gdb) c
        Continuing.
        Breakpoint 1, printk ...
        (gdb) i r scause
        scause        0x0    0
        ```

    - 可以通过评测框架的 `lab1-task1` 测试。

### C 内联汇编

C++ 标准支持内联汇编，但 C 标准并不支持。GCC 提供了内联汇编的扩展语法，允许在 C 代码中嵌入汇编指令。

请阅读 GCC 编译器手册中的下列章节，学习内联汇编的语法：

- [Extended Asm (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)
- [Local Register Variables (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html)

??? note "要点：C 内联汇编"

    - **Extended Asm**

        - **基本概念**

            - **Extended asm** 允许在 C 代码中嵌入汇编指令，并可读写 C 变量、使用 C 标签跳转。
            - 语法格式：

                ```c
                asm asm-qualifiers (
                    AssemblerTemplate
                    : OutputOperands
                    [ : InputOperands
                    [ : Clobbers
                    [ : GotoLabels ]]]
                )
                ```

            - `asm` 是 GNU 扩展。若需兼容 `-ansi` 和 `-std`，应使用 `__asm__`。

        - **关键限定符**

            - **volatile**
                - 防止编译器优化掉 asm 语句或移动其位置。
                - 必须用于有副作用、或结果不可预测的指令（如 `rdtsc`）。
            - **inline**
                - 告诉编译器 asm 语句可被内联。
            - **goto**
                - 允许从 asm 跳转到指定的 C 标签。

        - **参数类型**

            1. **AssemblerTemplate**
                - 字符串模板，混合汇编指令和参数占位符（如 `%0`, `%1` 或符号名 `%[name]`）。

            2. **OutputOperands**
                - C 变量，被汇编代码修改。格式：`[symbol] "constraint" (variable)`
                - 约束符必须以 `=`（只写）或 `+`（读写）开头。
                - 可用 `&` 防止输出与输入重叠。

            3. **InputOperands**
                - C 表达式，被汇编代码读取。格式：`[symbol] "constraint" (expression)`
                - 约束不能以 `=` 或 `+` 开头。可用数字/符号名指定与输出共享寄存器。

            4. **Clobbers**
                - 汇编会修改的额外寄存器或状态，如 `"cc"`（标志寄存器）、`"memory"`（内存屏障）、`"redzone"`。
                - 不可与输入输出寄存器重叠，不应包含栈指针。

            5. **GotoLabels**
                - 汇编中可能跳转的 C 标签。禁止跨越 asm 语句跳转。

        - **常见用法示例**

            - **基本用法**：

                ```c
                int src = 1, dst;
                asm ("mov %1, %0\n\t"
                    "add $1, %0"
                    : "=r"(dst)      // 输出
                    : "r"(src));     // 输入
                ```

            - **带 clobber**：

                ```c
                asm volatile ("movc3 %0, %1, %2"
                    : /* no outputs */
                    : "g"(from), "g"(to), "g"(count)
                    : "r0","r1","r2","r3","r4","r5","memory");
                ```

            - **符号名输入输出**：

                ```c
                uint32_t Mask = 1234, Index;
                asm ("bsfl %[aMask], %[aIndex]"
                    : [aIndex] "=r"(Index)
                    : [aMask] "r"(Mask)
                    : "cc");
                ```

        - **编译器与优化注意事项**

            - 若输出未使用，编译器可能删除 asm，需加 `volatile`。
            - 输入寄存器不可在 asm 内修改，除非与输出绑定。
            - `memory` clobber 形成编译器级别内存屏障，但不能阻止 CPU 推测执行。
            - 寄存器分配避免 clobber 中的寄存器。
            - 输出约束 `+` 同时算输入和输出，影响 30 个操作数上限。

        - **性能与安全**

            - 使用早期 clobber (`&`) 或绑定输出-输入避免寄存器冲突。
            - 谨慎使用 `memory`，会导致寄存器刷新，影响性能。
            - 避免直接修改栈指针寄存器。

    - **Specifying Registers for Local Variables**

        - **基本概念**

            - 可以在函数内定义局部寄存器变量，并将其绑定到特定寄存器：

                ```c
                register int *foo asm("r12");
                ```

            - `register` 关键字必需，不能与 `static` 一起使用。
            - 寄存器名称必须是目标平台上有效的寄存器名。

        - **限制与注意事项**

            - **禁止使用 `const` 和 `volatile`**：
                - `const` 可能导致编译器用初始值替换变量，使操作数分配到不同寄存器。
                - `volatile` 行为可能与预期不符。
            - 仅在调用 **Extended asm** 指定输入/输出寄存器时才支持此功能。
            - 建议选择在函数调用时 **自动保存和恢复** 的寄存器，以避免库函数调用破坏其值。

        - **用法示例**

            ```c
            register int *p1 asm("r0") = ...;
            register int *p2 asm("r1") = ...;
            register int *result asm("r0");
            asm("sysint" : "=r"(result) : "0"(p1), "r"(p2));
            ```

            - 上例中，`p1` 和 `p2` 的值在 `asm` 调用中绑定到 `r0` 和 `r1`。

        - **常见问题与解决**

            - **寄存器可能被后续代码或库函数调用破坏**，包括算术运算时的临时调用。
            - 解决方法：对中间表达式使用临时变量，避免直接在寄存器变量初始化中做计算：

                ```c
                int t1 = ...;
                register int *p1 asm("r0") = ...;
                register int *p2 asm("r1") = t1;
                register int *result asm("r0");
                asm("sysint" : "=r"(result) : "0"(p1), "r"(p2));
                ```

!!! example "动手做"

    下面这段 [C 内联汇编](<https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:___c,selection:(endColumn:1,endLineNumber:19,positionColumn:1,positionLineNumber:19,selectionStartColumn:1,selectionStartLineNumber:19,startColumn:1,startLineNumber:19),source:'%23include+%3Cstdint.h%3E%0A%0Along+wrong_function(long+a,+long+b,+long+c,+long+d)+%7B%0A++++long+out1,+out2%3B%0A++++__asm__+volatile(%0A++++++++%22mv+a0,+%25%5Ba%5D%5Cn%22%0A++++++++%22mv+a1,+%25%5Bb%5D%5Cn%22%0A++++++++%22mv+a2,+%25%5Bc%5D%5Cn%22%0A++++++++%22mv+a3,+%25%5Bd%5D%5Cn%22%0A++++++++%22ecall%5Cn%22%0A++++++++%22mv+%25%5Bout1%5D,+a0%5Cn%22%0A++++++++%22mv+%25%5Bout2%5D,+a1%5Cn%22%0A++++++++:+%5Bout1%5D+%22%3Dr%22(out1),+%5Bout2%5D+%22%3Dr%22(out2)%0A++++++++:+%5Ba%5D+%22r%22(a),+%5Bb%5D+%22r%22(b),+%5Bc%5D+%22r%22(c),+%5Bd%5D+%22r%22(d)%0A++++++++:+%22memory%22%0A++++)%3B%0A++++return+out1+%2B+out2%3B%0A%7D%0A'),l:'5',n:'0',o:'C+source+%231',t:'0')),k:35.29657581185066,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-cgcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:2,lang:___c,libs:!(),options:'-O0',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:64.70342418814936,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4>)存在问题：

    <iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:___c,selection:(endColumn:1,endLineNumber:19,positionColumn:1,positionLineNumber:19,selectionStartColumn:1,selectionStartLineNumber:19,startColumn:1,startLineNumber:19),source:'%23include+%3Cstdint.h%3E%0A%0Along+wrong_function(long+a,+long+b,+long+c,+long+d)+%7B%0A++++long+out1,+out2%3B%0A++++__asm__+volatile(%0A++++++++%22mv+a0,+%25%5Ba%5D%5Cn%22%0A++++++++%22mv+a1,+%25%5Bb%5D%5Cn%22%0A++++++++%22mv+a2,+%25%5Bc%5D%5Cn%22%0A++++++++%22mv+a3,+%25%5Bd%5D%5Cn%22%0A++++++++%22ecall%5Cn%22%0A++++++++%22mv+%25%5Bout1%5D,+a0%5Cn%22%0A++++++++%22mv+%25%5Bout2%5D,+a1%5Cn%22%0A++++++++:+%5Bout1%5D+%22%3Dr%22(out1),+%5Bout2%5D+%22%3Dr%22(out2)%0A++++++++:+%5Ba%5D+%22r%22(a),+%5Bb%5D+%22r%22(b),+%5Bc%5D+%22r%22(c),+%5Bd%5D+%22r%22(d)%0A++++++++:+%22memory%22%0A++++)%3B%0A++++return+out1+%2B+out2%3B%0A%7D%0A'),l:'5',n:'0',o:'C+source+%231',t:'0')),k:35.29657581185066,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-cgcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:2,lang:___c,libs:!(),options:'-O0',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:64.70342418814936,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

    1. 请你观察右侧汇编结果，指出问题所在，并分析产生问题的原因。
    2. 给出你修正后的代码。

### SBI 与 ECALL

请阅读以下材料：

- 非特权级手册 [2.8. Environment Call and Breakpoints](spec/riscv-unprivileged.html#ecall-ebreak)

    !!! question "考点"

        - ECALL 指令的作用是什么？
        - 对于我们实现的操作系统来说，服务请求的参数传递由谁定义？

- [SBI 手册](spec/riscv-sbi.pdf)

    - Chapter 1. Introduction

        !!! question "考点"

            - 什么是 SBI？它为谁提供服务？

    - Chapter 3. Binary Encoding 的章节导言

        !!! question "考点"

            - 在本课程中，谁是 Supervisor？谁是 SEE？
            - 如何标识一个特定的 SBI 调用？
            - SBI 调用的参数和返回值是如何传递的？
            - SBI 调用时，哪些寄存器的值不会被保存？
            - 如何判断 SBI 调用是否成功？

    - Chapter 12. Debug Console Extension (EID #0x4442434E "DBCN")

        !!! question "考点"

            - Debug Console Extension 提供了什么功能？
            - `sbi_debug_console_write` 函数的参数和返回值分别是什么？

### Task 2：使用 SBI 实现 `printk()`

现在你已经知道：

- 怎么写 C 内联汇编
- 怎么使用 ECALL 指令发起 SBI 调用
- `sbi_debug_console_write` 函数的参数和返回值

你的任务是：

- 在 `arch/riscv/sbi.c` 中使用内联汇编实现 `sbi_ecall()`。
- 在 `arch/riscv/sbi.c` 中补全 `sbi_debug_console_*()` 几个函数，也就是向 `sbi_ecall()` 传递正确的参数。

!!! success "完成条件"

    - 成功看到 `Hello, ZJU OS 2025!`。
    - 通过评测框架的 `lab1-task2` 测试。

!!! info "更多资料"

    Linux 内核的 `printk()` 与我们的简单输出不同，它实际上是打印到内核日志缓冲区（ring buffer）中，然后由专门的内核线程 `klogd` 负责将日志输出到控制台或保存到文件中。

    感兴趣的同学可以阅读 [Linux kernel printk new implementation - huang weiliang](https://huangweiliang.github.io/2024/06/05/printk/)。

## Part 2：时钟中断及其处理

### 特权级、中断与异常

首先，让我们了解特权级、中断与异常的概念：

- 特权级手册 [1.2. Privilege Levels](spec/riscv-privileged.html#_privilege_levels)：

    !!! question "考点"

        - 特权级是用来干什么的？
        - 执行当前特权级不允许的操作会发生什么？
        - M、U、S 模式分别是为了什么设计的？

- 非特权级手册 [1.6. Exceptions, Traps, and Interrupts](spec/riscv-unprivileged.html#trap-defn) 的第一段（直接贴在下面了）：

    > We use the term exception to refer to an unusual condition occurring at run time associated with an instruction in the current RISC-V hart. We use the term interrupt to refer to an external asynchronous event that may cause a RISC-V hart to experience an unexpected transfer of control. We use the term trap to refer to the transfer of control to a trap handler caused by either an exception or an interrupt.

    !!! question "考点"

        - RISC-V 中 exception、interrupt 有何异同？
        - Trap 是什么意思？举两个 Trap 的例子

### CSR 寄存器

接下来让我们了解 CSR 寄存器。它们是 RISC-V CPU 中的一系列特殊寄存器，能够反映和控制 CPU 当前的状态和执行机制。我们先了解 M 模式的 CSR 寄存器：

- [Chapter 2. Control and Status Registers (CSRs)](spec/riscv-privileged.html#priv-csrs) 的章节导言

    !!! question "考点"

        - 读取、修改、写入 CSR 的指令定义在哪个扩展？
        - S 模式的 CSR 能被 M 模式访问吗？反之呢？

- [3.1.6.1. Privilege and Global Interrupt-Enable Stack in **mstatus** register](https://zju-os.github.io/doc/spec/riscv-privileged.html#privstack)

    !!! question "考点"

        - mstatus 寄存器的作用是什么？
        - xIE bit 的作用是什么？
        - 运行在 S 模式且 mstatus.SIE=0，mstatus.MIE=0 时，发生中断会进入哪个模式？
        - 从特权级 y 陷入到更高的特权级 x 时，xPIE、xIE 和 xPP 会如何变化？
        - xRET 指令返回时，特权级和中断使能位会如何恢复？xPP 会设置为什么？

    ??? note "要点：mstatus"

        1. **全局中断使能位 (Global Interrupt-Enable Bits)**

            - `MIE` → 控制 M-mode 中断的全局使能。
            - `SIE` → 控制 S-mode 中断的全局使能（如果 S-mode 未实现，则为只读 0）。
            - 当在特权级 x 下执行时：
                - `xIE = 1` → 该特权级的中断**全局使能**。
                - `xIE = 0` → 该特权级的中断**全局关闭**。
            - 更高特权级的中断始终使能；更低特权级的中断始终关闭。

        2. **原子性保障**

            - `xIE` 位位于 `mstatus` 低位，可用单条 CSR 指令原子地开/关中断，确保中断处理的原子性。

        3. **两级中断与特权栈 (Two-Level Stack)**

            - **xPIE**：保存陷入前 `xIE` 的值。
            - **xPP**：保存陷入前的特权级，M-mode 是 2 位，S-mode 是 1 位。
            - 陷入时：
                - `xPIE ← xIE`
                - `xIE ← 0`（进入陷入时中断会被关闭）
                - `xPP ← y`（保存之前的特权级 y）

        4. **返回指令 xRET 的行为**

            - 从陷入返回时：
                - `xIE ← xPIE`
                - 特权级 ← `xPP`
                - `xPIE ← 1`
                - `xPP ← 最低支持的特权级`（帮助检测软件管理错误）
            - 如果返回的特权级 ≠ M，`MPRV` 被清零。

        5. **陷入处理的安全性要求**

            - 陷入处理必须避免在**关键状态保存阶段**开启中断或引发异常，避免：
                - 覆盖关键状态，导致恢复失败。
                - 陷入处理过程中出现无限递归陷入。
            - 需要小心设计，确保异常在安全的阶段被正确处理。

- [3.1.9. Machine Interrupt **(mip and mie)** Registers](spec/riscv-privileged.html#_machine_interrupt_mip_and_mie_registers)

    !!! question "考点"

        - mip 和 mie 寄存器的作用分别是什么？
        - 在什么条件下，中断会陷入 M 模式？
        - 为什么软件中断的优先级高于定时器中断？

    ??? note "要点：mip 和 mie"

        - **基本功能**

            - **mip (Machine Interrupt Pending)**：保存挂起中断的信息
            - **mie (Machine Interrupt Enable)**：控制各类中断是否被使能
            - **位 i** 对应中断原因编号 i（与 `mcause` 寄存器中的中断原因编号对应）
            - **标准中断位**：只占用 **bits 15:0**，bit 16 及以上为平台自定义

        - **中断触发条件**

            一个中断 i 会导致陷入 M 模式，需满足以下条件：

            1. 当前特权模式是 M 且 **mstatus.MIE = 1**，或当前特权模式 < M
            2. **mip[i] = 1** 且 **mie[i] = 1**
            3. 如果存在 **mideleg**，则 **mideleg[i] = 0**

            这些条件在以下情况下必须及时重新评估：

            - 中断挂起状态变化
            - 执行 `xRET` 指令
            - 显式写入 mip/mie/mstatus/mideleg

        - **中断优先级与可写性**

            - M 模式中断优先级最高
            - 每个 mip 位可能是 **只读** 或 **可写**：
                - 若可写，中断可通过写 0 清除挂起状态
                - 若只读，必须通过其他机制清除挂起状态
            - mie 中可写位：只有能挂起的中断才能在 mie 中被设置；不可写的必须为 0

        - **标准中断位分配（bits 15:0）**

            | 中断类型        | Pending 位  | Enable 位   | 可写性 / 来源                 |
            | ----------- | ---------- | ---------- | ------------------------ |
            | 外部中断 (M 级)  | mip.MEIP   | mie.MEIE   | MEIP 只读，平台中断控制器管理        |
            | 定时器中断 (M 级) | mip.MTIP   | mie.MTIE   | MTIP 只读，通过写 mtimecmp 清除  |
            | 软件中断 (M 级)  | mip.MSIP   | mie.MSIE   | MSIP 只读，通过内存映射寄存器设置      |
            | 外部中断 (S 级)  | mip.SEIP   | mie.SEIE   | SEIP 可写，用于 S 模拟外部中断      |
            | 定时器中断 (S 级) | mip.STIP   | mie.STIE   | STIP 可写，用于 S 模拟定时器中断     |
            | 软件中断 (S 级)  | mip.SSIP   | mie.SSIE   | SSIP 可写，可由平台中断控制器设置      |
            | 本地计数溢出中断    | mip.LCOFIP | mie.LCOFIE | Sscofpmf 扩展支持，R/W；否则恒为 0 |

        - **中断优先级顺序（高 → 低）**

            1. MEI (Machine External Interrupt)
            2. MSI (Machine Software Interrupt)
            3. MTI (Machine Timer Interrupt)
            4. SEI (Supervisor External Interrupt)
            5. SSI (Supervisor Software Interrupt)
            6. STI (Supervisor Timer Interrupt)
            7. LCOFI (Local Counter Overflow Interrupt)

            设计原则：

            - 高特权级中断优先于低特权级中断（支持抢占）
            - 外部中断优先于内部中断（设备响应要求高）
            - 软件中断优先于定时器中断（用于多核消息传递）

        - **S 模式相关**（在下一节学习）

            - S 模式通过 **sip / sie** 寄存器看到委派过来的中断
            - 如果 **mideleg[i] = 1**，中断 i 被委派到 S 模式，否则 sip/sie 中对应位为 0

- [3.1.7. Machine Trap-Vector Base-Address (**mtvec**) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#_machine_trap_vector_base_address_mtvec_register)

    !!! question "考点"

        - mtvec 寄存器的作用是什么？
        - 为什么这个寄存器的低两位可以分配给 MODE 字段？
        - 向量中断模式下，发生中断后 PC 会被设置成多少？

- [3.1.8. Machine Trap Delegation (**medeleg** and **mideleg**) Registers](https://zju-os.github.io/doc/spec/riscv-privileged.html#_machine_trap_delegation_medeleg_and_mideleg_registers)

    !!! question "考点"

        - 为什么需要委派机制？
        - medeleg 和 mideleg 寄存器的作用分别是什么？
        - 当一个 trap 被委派到 S 模式后，下面这些地方的值会如何变化？
            - scause
            - stval
            - sepc
            - mstatus.SPP
            - mstatus.SPIE
            - mstatus.SIE
        - 如果一个 trap 是在 M 模式下发生的，但它在 medeleg 中已被设置委派给 S 模式，会在哪里处理？
        - mideleg 中的某一位被设置后，这个中断在 M 模式下会被触发吗？

- [3.1.14. Machine Exception Program Counter (**mepc**) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#_machine_exception_program_counter_mepc_register) 和 [3.1.15. Machine Cause (**mcause**) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#mcause) 和 [3.1.16. Machine Trap Value (**mtval**) Register](https://zju-os.github.io/doc/spec/riscv-privileged.html#_machine_trap_value_mtval_register)

    !!! question "考点"

        - mepc、mcause、mtval 寄存器的作用是什么？
        - mcause 寄存器中，中断和异常的区别是什么？

总结一下，我们学习了 M 模式的几个关键 CSR 寄存器：

```text
mstatus mip mie mtvec medeleg mideleg mepc mcause mtval
```

S 模式也有几个对应的 CSR 寄存器：

```text
sstatus sip sie stvec scause sepc stval
```

它们的作用和 M 模式类似，请同学们在后续实验用到时自行阅读 [12.1. Supervisor CSRs](https://zju-os.github.io/doc/spec/riscv-privileged.html#_supervisor_csrs)。

!!! example "动手做"

    断点打在内核第一条指令处，使用 QEMU Monitor 查看此时 CSR 寄存器的状态。解释**本节学习的**所有 M、S 模式 CSR 寄存器的值的含义。你会发现有些 S 模式寄存器没有在 QEMU 中展示，这并不是一个 Bug，请查看 [RISC-V sstatus register is missing in qemu console / gdb (#1260) · Issue · qemu-project/qemu](https://gitlab.com/qemu-project/qemu/-/issues/1260) 了解原因。

!!! example "动手做"

    移除你在 Task1 做的工作，然后：

    - 在 `_start()` 打断点，然后单步调试，进入 `start_kernel()` 后程序能顺利跳转到 `printk()` 吗？如果不顺利，检查合适的 CSR 寄存器，看看发生了什么异常？
    - 删掉在 `_start()` 打的断点，在 `printk()` 入口处打断点，然后继续运行（continue），你会发现最终能够到达 `printk()` 停下来，这是为什么？

    （探究结束后记得 Task1 的工作还原回去）

### 特权指令

然后，特权级有几个特权指令，现在我们只关注 xRET。请阅读 [3.3.2. Trap-Return Instructions](spec/riscv-privileged.html#otherpriv)。

!!! question "考点"

    - xRET 指令的作用是什么？
    - xRET 指令执行后，CSR 寄存器会如何变化？PC 的值会如何变化？

### Zicsr 扩展

最后，我们将阅读 Zicsr 扩展，了解如何操控 CSR 寄存器。

- 非特权级手册 [6. "Zicsr", Extension for Control and Status Register (CSR) Instructions, Version 2.0](spec/riscv-unprivileged.html#csrinsts)

    ??? note "要点"

        - 掌握四个标准 CSR 指令的用法：
            - RW: Read/Write
            - RS: Read and Set bits
            - RC: Read and Clear bits
            - RWI/RSI/RCI: Immediate versions
        - 掌握四个 CSR 伪指令的用法：
            - R: Read
            - W: Write
            - S: Set bits
            - C: Clear bits
            - WI/SI/CI: Immediate versions

### Task3：Trap Handler

现在你已经全面了解了 RISC-V 的中断与异常机制以及 CSR 在其中扮演的重要角色。让我们用简单的 S 模式软件中断来实践这些知识。

你的任务是：

- 补全 `arch/riscv/include/sbi.h` 中的 `csr_*()` **宏函数**。

    宏函数的语法应该在大一的 C 语言课程中学习过，如果你忘记了，可以阅读 [Replacing text macros - cppreference.com](https://en.cppreference.com/w/c/preprocessor/replace.html)。

- 在 `start_kernel()` 中：

    - 打印 `sstatus`、`sie` 和 `sip` 的值
    - 将 `sie` 设置为合适的值，使得只有软件中断被使能
    - 将 `sstatus` 设置为合适的值，使能 S 模式中断
    - 将 `sip` 设置为合适的值，立刻触发一个软件中断

- 在 `head.S` 中，写入合适的 CSR 寄存器，将中断处理程序设置为 `_traps`
- 在 `entry.S` 中，补全 `_traps`
- 在 `trap.c` 中，完成软件中断的处理

    注意，我们总是把具体的中断处理包装为一个函数，然后在 `trap_handler()` 的 `switch` 语句中调用。这是因为 C 语言不太好直接在 `switch` 语句里声明变量等，会引起定义域问题，所以总是用一个函数进行包装。

    软件中断的处理函数是 `clear_ssip()`。

请你思考以下问题，再补全 `_traps`：

- Trap 随时可能发生。因此，Trap Handler 的首要工作是保存现场，并在完成 Trap 处理后恢复现场，保证程序的执行不受影响。为此，有哪些内容需要保存？保存到哪里？

    提示：可以参考一下 RISC-V 调用约定，它为了保存 caller 的上下文，是怎么做的？

- `_traps` 的第二个作用是跳转到 `trap.c` 中的 `trap_handler()` 函数，因为 C 语言编程更方便，我们当然想用 C 语言完成具体的 Trap 处理工作。你需要向 `trap_handler()` 传递哪些参数？
- `_traps` 非得用汇编写吗？直接指向 `trap_handler()` 可以吗？

!!! success "完成条件"

    - 成功进入中断处理程序，并且正确识别到软件中断。
    - 成功从中断程序退出，返回 `start_kernel()` 并继续执行后续的输出。
    - 通过评测框架的 `lab1-task3` 测试。

### 时钟中断

阅读 [3.2.1. Machine Timer **(mtime and mtimecmp)** Registers](spec/riscv-privileged.html#_machine_timer_mtime_and_mtimecmp_registers)，了解 RISC-V 的时钟中断机制：

!!! question "考点"

    - mtime 和 mtimecmp 寄存器的作用分别是什么？
    - mtime 记录的数值是 CPU 时钟周期吗？
    - M 模式时钟中断挂起的条件是什么？
    - 要让 CPU 响应 M 模式时钟中断，需要如何设置 CSR 寄存器？

??? note "要点：mtime 和 mtimecmp"

    - **mtime 寄存器**

        - 平台提供一个**内存映射**的机器模式读写寄存器 `mtime`，作为实时时钟
        - `mtime` 以**固定频率**递增，提供机制确定其周期
        - `mtime` 溢出后会回绕
        - 所有 RV32 和 RV64 系统中，`mtime` 都是 **64 位精度**

    - **mtimecmp 寄存器与中断**

        - `mtimecmp` 也是 64 位内存映射机器模式定时器比较寄存器
        - 当 `mtime ≥ mtimecmp` 时，机器定时器中断挂起
        - 中断会一直挂起，直到 `mtimecmp > mtime`（通常通过写入新的 mtimecmp 值实现）
        - 中断需 **启用全局中断** 且 **mie 寄存器中 MTIE 位置 1** 才会被响应

    - **设计原因与硬件特性**

        - 定时器基于**挂钟时间（wall-clock time）**，支持动态电压和频率调整的处理器
        - 实时时钟（RTC）成本高，通常系统中只有一个，且与 CPU 不在同一电压/频率域
        - 通过内存映射而非 CSR 暴露 mtime，便于多个 hart 共享 RTC

    - **低特权级与虚拟定时器**

        - 低特权级没有自己的 timecmp 寄存器
        - 机器模式软件可通过复用 `mtimecmp` 实现多个虚拟定时器

    - **比较与中断行为**

        - `mtime` 与 `mtimecmp` 比较结果的变化会最终反映到 **MTIP**，但可能有延迟
        - 可能出现**伪中断**：处理程序刚写入 `mtimecmp` 后立即返回时，中断可能还未清除

    - **RV32 和 RV64 的区别**

        - **RV32**：写入 `mtimecmp` 时需分两次写 32 位值，避免因中间值过小而触发伪中断
        - **RV64**：支持自然对齐的 64 位原子访问，简化写入操作

    - **time 与 timeh CSR**

        - `time` 是 `mtime` 的只读影子寄存器
        - RV32 下，`timeh` 影射高 32 位，`time` 影射低 32 位
        - `mtime` 变化会最终反映在 `time`/`timeh`，但可能有延迟

很可惜 `mtime` 和 `mtimecmp` 仅供 M 模式使用，让 OpenSBI 等 M 模式软件能够感知时间流逝。那我们的 S 模式内核呢？首先想到 SBI 是否有提供相关服务。请你阅读 [SBI 手册](spec/riscv-sbi.pdf) 中的 Chapter 6. Timer Extension (EID #0x54494D45 "TIME")，了解如何使用 SBI 提供的定时器服务。

!!! note "要点：SBI Timer Extension"

    **核心函数：`sbi_set_timer` (FID #0)**

    - **功能：**设置下一次定时器事件的触发时间，参数为 **绝对时间（`stime_value`）**。
    - **清除定时器中断的两种方法：**

        1. 设置 `stime_value = (uint64_t)-1` → 定时器中断触发在“无限未来”。
        2. 通过清除 `sie.STIE` 位 → 屏蔽定时器中断。

    - **自动清中断：**

        无论是否屏蔽定时器中断，只要设置了未来的时间，函数都会清除当前挂起的定时器中断。

我们从标准中了解到 SBI 的定时器服务是 M 模式软件（如 OpenSBI）通过复用 `mtimecmp` 模拟实现的，效率太低了。因此 RISC-V 定义了 SSTC 扩展解决这一问题。请你阅读 [20. "Sstc" Extension for Supervisor-mode Timer Interrupts, Version 1.0](spec/riscv-privileged.html#Sstc) 的第一节 [20.1. Machine and Supervisor Level Additions](https://zju-os.github.io/doc/spec/riscv-privileged.html#_machine_and_supervisor_level_additions)，具体了解新增 SSTC 扩展后的时钟中断机制。

!!! note "要点：SSTC 扩展"

    该扩展提高了 S 模式定时器中断的性能。未提供该扩展时，S/HS 模式的时钟中断机制是由 M 模式软件（如 OpenSBI）通过复用 `mtimecmp` 模拟实现的。

    该扩展主要内容包括：

    - **新增 CSR 寄存器**

        - `smtimecmp`：S 模式定时器比较寄存器
        - S 模式时钟中断挂起条件：`time` ≥ `smtimecmp`（还记得 `time` 对应哪个 M 模式寄存器吗？）

    - **修改相关寄存器描述**

        - `mip/mie`：定义 S 级定时器中断挂起与使能位（STIP/STIE）。
        - `sip/sie`：反映 S 级定时器中断状态，直接由 `stimecmp` 控制。
        - `mcounteren`：允许或禁止 S 模式访问 `stimecmp` / `vstimecmp`。

QEMU 已经支持 SSTC 扩展，因此你可以通过直接写 `csrw smtimecmp, a0` 来设置 S 模式的定时器中断了。但是，为了兼容性考虑，我们仍然应当使用 SBI 提供的定时器服务。

!!! example "动手做"

    阅读 OpenSBI 源码，了解 OpenSBI 是如何实现 `sbi_set_timer()` 函数的：

    - [`lib/sbi/sbi_ecall_time.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/sbi/sbi_ecall_time.c) 的 `sbi_ecall_set_timer()` 函数
    - [`lib/sbi/sbi_timer.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/sbi/sbi_timer.c) 的 `sbi_timer_event_start()` 函数

    请你指出：

    - 当平台支持 SSTC 扩展时，OpenSBI 会如何设置定时器？
    - 当平台不支持 SSTC 扩展时，OpenSBI 如何进行定时器多路复用？

!!! info "更多资料"

    - [riscv timer的基本逻辑 | Sherlock's blog](https://wangzhou.github.io/riscv-timer%E7%9A%84%E5%9F%BA%E6%9C%AC%E9%80%BB%E8%BE%91/)：对 RISC-V 规范、QEMU 和 Linux 内核的实现进行了整体分析。

### Task4：开启并处理 S 模式时钟中断

关于时间：

- 对于 QEMU RISC-V virt 机器，`time` 的频率是 10MHz，OpenSBI 帮你通过设备接口查询了：

    ```text
    Platform Timer Device       : aclint-mtimer @ 10000000Hz
    ```

- [POSIX 标准](https://pubs.opengroup.org/onlinepubs/009695099/functions/clock.html) 定义了：
    - `CLOCKS_PER_SEC` 常量：表示每秒的时钟滴答数（Clocks），在 POSIX 标准中要求为 1000000（1M）
    - `clock()` 函数：返回自进程启动以来，过去的时钟滴答数
    - 也就是说，要将 `clock()` 返回的值转换为秒，需要用返回值除以 `CLOCKS_PER_SEC` 宏的值
- 思考：经过多久时间 `time` 会回绕？

你的任务：

- 在 `start_kernel()` 中：

    - 将 `sie` 设置为合适的值，使得时钟中断也被使能
    - 使用 `sbi_set_timer()` 将定时器设置为 `0`，这样会立刻触发一次时钟中断

    因为没有重新设置 `stimecmp`，所以时钟中断会一直触发。你会看到控制台不停地打印。

- 在 `trap_handler()` 中，添加时钟中断的处理

    在 `clock_set_next_event()` 中用 `sbi_set_timer()` 设置下一次时钟中断到一秒后，现在你会看到控制台每秒打印一次时钟中断。

- 在 `clock.c` 中实现 POSIX 标准的 `clock()` 函数

    现在 `start_kernel()` 应该能正确对循环进行计时了

!!! success "完成条件"

    - 通过评测框架的 `lab1-task4` 测试。

!!! info "内核禁用浮点数"

    也许你会问，上面的 `start_kernel()` 中为什么不用 `(double)(time_end - time_start) / CLOCKS_PER_SEC` 打印秒数呢？

    事实上我们的 `printk()` 实现并不支持浮点数的输出（Linux 的也是）。如果要支持浮点数打印，就需要使用浮点数及其运算，然而这在内核是禁用的。原因有二：

    - **没必要：**我们学到现在都没接触过 RISC-V 的整数乘除法（M 指令集），更别说浮点数了（F、D 指令集），可见浮点数的复杂性。内核完全可以摒弃这一复杂性，使用整数完成所有工作。
    - **中断与异常：**现在我们的 Trap Handler 仅保存了 I 指令集的寄存器。如果使用浮点数，就需要保存 F 指令集的寄存器，增加了开销。此外，浮点数运算可能引发浮点异常（如无效操作、除零等），这些异常通过新的浮点 CSR（FCSR）来记录，并（在目前的 RISC-V 规范下）需要软件在每次浮点计算后进行处理。

    对浮点数感兴趣的同学可以进一步阅读：

    - [Floating-point API — The Linux Kernel documentation](https://docs.kernel.org/core-api/floating-point.html)
    - [Use of floating point in the Linux kernel - Stack Overflow](https://stackoverflow.com/questions/13886338/use-of-floating-point-in-the-linux-kernel)
    - [RISC-V 指令集架構介紹 - F Standard Extension | Jim's Dev Blog](https://tclin914.github.io/3d45634e/)
